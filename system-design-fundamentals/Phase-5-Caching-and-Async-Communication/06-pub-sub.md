# Pub/Sub (Publish/Subscribe)

**Phase 5 — Caching + Async Communication | Topic 6**

---

## What is Pub/Sub?

Pub/Sub (Publish/Subscribe) is a messaging pattern where publishers send messages to named channels (topics), and subscribers receive messages from those channels — without publishers knowing who the subscribers are, or vice versa.

```
Without Pub/Sub (direct coupling):
  Order Service directly calls:
    → Payment Service.processPayment()
    → Inventory Service.decrementStock()
    → Notification Service.sendEmail()
    → Analytics Service.recordOrder()

  Problems:
  → Order Service knows about 4 other services
  → If Notification Service is down, order fails
  → Adding new service requires modifying Order Service
  → Synchronous chain = latency adds up

With Pub/Sub:
  Order Service publishes: OrderCreated event

  Payment Service   subscribes to OrderCreated → charges card
  Inventory Service subscribes to OrderCreated → decrements stock
  Notification Service subscribes to OrderCreated → sends email
  Analytics Service subscribes to OrderCreated → records event

  Benefits:
  → Order Service knows nothing about subscribers
  → Subscriber down? Event waits or is retried independently
  → Add new service? Just subscribe — no Order Service changes
  → Fully asynchronous — Order Service returns immediately
```

---

## Core Pub/Sub Concepts

```
Publisher:   Sends messages to a topic
             Doesn't know who reads them

Topic:       Named channel for messages
             Logical grouping of related events
             Examples: "order.created", "payment.processed"

Subscriber:  Receives messages from a topic
             Doesn't know who sent them
             Can have many subscribers per topic

Message:     The data being communicated
             Usually: event type + payload + metadata

Broker:      Middleware that routes messages
             Decouples publisher from subscriber
             Examples: Kafka, RabbitMQ, Redis, Google Pub/Sub
```

---

## Pub/Sub vs Message Queue — Key Difference

```
Message Queue:
  Message sent to queue
  ONE consumer receives and processes it
  Message removed after processing
  Point-to-point: 1 sender → 1 receiver

  Use: task distribution, work queues
  "Process this order" → one worker handles it

Pub/Sub:
  Message sent to topic
  ALL subscribers receive a copy
  Message retained (depending on system)
  Broadcast: 1 sender → N receivers

  Use: event notification, broadcast
  "Order created" → payment AND inventory AND notifications all get it
```

```
Kafka blurs this distinction:
  Consumer groups = Pub/Sub (each group gets all messages)
  Within a group = Message Queue (one consumer per partition)
  Best of both: broadcast to groups + load balance within group
```

---

## Redis Pub/Sub

Simplest pub/sub implementation. In-memory, fire-and-forget.

```
Publish:
  PUBLISH order-events '{"type":"ORDER_CREATED","orderId":"123"}'

Subscribe:
  SUBSCRIBE order-events

  → Receives all messages published to order-events
```

```java
// Spring Boot Redis Pub/Sub

// Publisher
@Service
public class OrderEventPublisher {

    private final RedisTemplate<String, String> redis;

    public void publish(OrderCreatedEvent event) {
        String message = objectMapper.writeValueAsString(event);
        redis.convertAndSend("order-events", message);
    }
}

// Subscriber
@Component
public class PaymentEventListener {

    @Bean
    public MessageListenerAdapter listenerAdapter() {
        return new MessageListenerAdapter(this, "handleMessage");
    }

    @Bean
    public RedisMessageListenerContainer container(
            RedisConnectionFactory factory,
            MessageListenerAdapter listenerAdapter) {

        RedisMessageListenerContainer container =
            new RedisMessageListenerContainer();
        container.setConnectionFactory(factory);
        container.addMessageListener(
            listenerAdapter,
            new PatternTopic("order-events")); // subscribe
        return container;
    }

    public void handleMessage(String message) {
        OrderCreatedEvent event = objectMapper.readValue(
            message, OrderCreatedEvent.class);
        paymentService.processPayment(event.getOrderId());
    }
}
```

**Redis Pub/Sub limitations:**

```
❌ No message persistence — if subscriber is down, messages lost
❌ No consumer groups — all subscribers get every message
❌ No replay — can't re-read old messages
❌ No backpressure — publisher overwhelms subscriber
❌ No delivery guarantees — at-most-once

Use Redis Pub/Sub for:
  ✅ Cache invalidation broadcast (covered earlier)
  ✅ Real-time notifications (presence, typing indicators)
  ✅ Short-lived signals where loss is acceptable
  ✅ WebSocket broadcast to multiple server instances

NOT for: reliable event processing, audit trails, order-critical events
```

---

## Kafka Pub/Sub — Production Standard

Kafka is a **distributed event streaming platform** that implements pub/sub with durability, replay, and consumer group semantics.

```
Kafka key concepts:
  Topic:          Named log (like a database table for events)
  Partition:      Ordered, immutable sequence of records within a topic
  Offset:         Position of a record within a partition
  Producer:       Writes records to topics
  Consumer:       Reads records from topics
  Consumer Group: Logical subscriber — group of consumers sharing load

Topic: order-events (3 partitions)
  Partition 0: [msg0, msg1, msg4, msg7]
  Partition 1: [msg2, msg5, msg8]
  Partition 2: [msg3, msg6, msg9]
```

### Consumer Groups — The Killer Feature

```
Topic: order-events

Consumer Group "payment-service" (3 instances):
  Consumer A → reads Partition 0
  Consumer B → reads Partition 1
  Consumer C → reads Partition 2
  → Every message processed by exactly one payment consumer
  → Load balanced across instances

Consumer Group "notification-service" (2 instances):
  Consumer D → reads Partitions 0, 1
  Consumer E → reads Partition 2
  → Every message also processed by notification service
  → Payment and notification groups are independent

Both groups receive ALL messages
Within each group, messages are load balanced
Different groups process at different speeds
Different groups maintain independent offsets
```

### Kafka Pub/Sub in Spring Boot

```java
// Producer (Publisher)
@Service
public class OrderEventProducer {

    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .total(order.getTotal())
            .timestamp(Instant.now())
            .build();

        // Key = orderId → same order always goes to same partition
        // Guarantees ordering for same order
        kafkaTemplate.send("order-events", order.getId(), event);

        log.info("Published OrderCreated for order {}", order.getId());
    }
}

// Consumer (Subscriber) — Payment Service
@Service
public class PaymentEventConsumer {

    @KafkaListener(
        topics = "order-events",
        groupId = "payment-service",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void handleOrderCreated(
            @Payload OrderCreatedEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {

        log.info("Processing order {} from partition {} offset {}",
            event.getOrderId(), partition, offset);

        paymentService.processPayment(event);
    }
}

// Consumer (Subscriber) — Notification Service
@Service
public class NotificationEventConsumer {

    @KafkaListener(
        topics = "order-events",
        groupId = "notification-service"  // different group = gets all messages
    )
    public void handleOrderCreated(OrderCreatedEvent event) {
        notificationService.sendOrderConfirmation(event);
    }
}
```

---

## Pub/Sub Patterns

### Fan-Out Pattern

One event → multiple independent workflows:

```
OrderCreated event
    ├── payment-service consumer group     → charge card
    ├── inventory-service consumer group   → decrement stock
    ├── notification-service consumer group → send email
    ├── analytics-service consumer group   → record in data warehouse
    ├── fraud-service consumer group       → check for fraud
    └── loyalty-service consumer group     → award points

Add new service? Subscribe to topic — no other changes needed.
Order Service is completely unaware of downstream processing.
```

### Event Sourcing Pattern

Kafka topic as immutable event log = source of truth:

```
All state changes recorded as events:
  UserRegistered { userId, email, timestamp }
  OrderCreated   { orderId, userId, items, timestamp }
  PaymentCharged { orderId, amount, timestamp }
  OrderShipped   { orderId, trackingId, timestamp }

Benefits:
  Complete audit trail — replay entire history
  Rebuild any state by replaying events from offset 0
  Time travel — state at any point in time
  Multiple views — build different read models from same events

Your state = result of all events applied in order
```

### Dead Letter Queue (DLQ) Pattern

```
Consumer fails to process message after retries:
  → Move to Dead Letter Topic instead of blocking

order-events (main topic)
    → Payment Consumer processes successfully ✅

    → Notification Consumer fails (email service down)
       Retry 1: fails
       Retry 2: fails
       Retry 3: fails
       → Message sent to notification-service-dlq topic

notification-service-dlq:
  Alert/log the failure
  Human reviews
  Fix notification service
  Replay from DLQ when ready

Main topic processing continues unblocked
```

```java
@Bean
public DefaultErrorHandler errorHandler(KafkaTemplate<?, ?> template) {
    DeadLetterPublishingRecoverer recoverer =
        new DeadLetterPublishingRecoverer(template,
            (record, ex) -> new TopicPartition(
                record.topic() + ".dlq",  // send to topic-name.dlq
                record.partition()
            ));

    ExponentialBackOffWithMaxRetries backOff =
        new ExponentialBackOffWithMaxRetries(3);
    backOff.setInitialInterval(1_000);  // 1 second
    backOff.setMultiplier(2);           // 2s, 4s, 8s
    backOff.setMaxInterval(10_000);     // max 10 seconds

    return new DefaultErrorHandler(recoverer, backOff);
}
```

### Saga Pattern via Pub/Sub

Distributed transactions using events:

```
CreateOrder Saga:

Step 1: Order Service
  → Creates order (status: PENDING)
  → Publishes OrderCreated to Kafka

Step 2: Payment Service (subscribes to OrderCreated)
  → Processes payment
  → If SUCCESS: publishes PaymentSucceeded
  → If FAILURE: publishes PaymentFailed

Step 3: Inventory Service (subscribes to PaymentSucceeded)
  → Decrements stock
  → Publishes StockDecremented

Step 4: Order Service (subscribes to StockDecremented)
  → Updates order status: CONFIRMED

Compensation flow (if PaymentFailed):
  Order Service subscribes to PaymentFailed
  → Updates order status: CANCELLED
  → Publishes OrderCancelled
```

---

## Pub/Sub Delivery Guarantees

```
At-Most-Once:
  Message delivered 0 or 1 times
  Fast, no overhead
  May lose messages
  Use: metrics, logs where loss acceptable

At-Least-Once:
  Message delivered 1 or more times (retries on failure)
  Consumer must be idempotent (handle duplicates)
  Kafka default with acks=all + consumer retries
  Use: most production use cases

Exactly-Once:
  Message delivered exactly once
  Requires: idempotent producers + transactions
  Higher overhead
  Kafka: enable.idempotence=true + transactional API
  Use: financial transactions, inventory changes
```

---

## Backpressure Handling

What happens when publishers produce faster than consumers can process?

```
Without backpressure handling:
  Producer: 100,000 msg/sec
  Consumer: 10,000 msg/sec

  Kafka: messages pile up in topic
  Lag = produced - consumed grows continuously
  Consumer never catches up
  Alert: consumer group lag > threshold

Solutions:

1. Scale consumers:
   Add more consumer instances
   Up to max = number of partitions

2. Increase partitions:
   More partitions → more parallelism → more consumers

3. Batch processing:
   Consumer reads 500 messages at once instead of 1
   Process as batch → more efficient

4. Separate slow consumers:
   Slow analytics processing shouldn't block fast payment processing
   Different consumer groups = independent scaling

5. Consumer lag monitoring:
   Alert when lag exceeds threshold
   Auto-scale consumer instances

   // Micrometer metric
   @Scheduled(fixedDelay = 30_000)
   public void reportConsumerLag() {
       Map<TopicPartition, Long> lags = adminClient
           .listConsumerGroupOffsets("payment-service")
           ...
       meterRegistry.gauge("kafka.consumer.lag", lags.values().stream()
           .mapToLong(Long::longValue).sum());
   }
```

---

## Google Cloud Pub/Sub — Managed Alternative

```
Google's fully managed pub/sub service:

Features:
  Global message delivery
  Automatic scaling
  At-least-once delivery
  Ordering keys (like Kafka partition keys)
  Dead letter topics
  Push (HTTP webhook) or Pull delivery

Push delivery:
  Pub/Sub calls your HTTP endpoint when message arrives
  No consumer process needed — serverless friendly

  Message → Pub/Sub → POST https://yourservice.com/webhook

  Useful for: Cloud Run, Cloud Functions (serverless)

Pull delivery:
  Your consumer polls for messages
  Like Kafka consumer

  Useful for: long-running services, batch processing

vs Kafka:
  Fully managed (no cluster to operate)
  Less control over partitioning
  Higher latency than Kafka
  Easier to get started
  Use when: on GCP, want managed service, don't need Kafka features
```

---

## Pub/Sub in System Design Interviews

When designing systems with Pub/Sub, structure your answer:

**Event definition:**
> "The Order Service publishes an `OrderCreated` event with fields: orderId, userId, items, totalAmount, timestamp. This event is published to the `order-events` Kafka topic."

**Consumer groups:**
> "Each downstream service has its own consumer group — payment-service, notification-service, analytics-service, fraud-service. Each group independently receives all messages and can scale its consumers up to the number of partitions."

**Ordering guarantee:**
> "Messages are keyed by orderId, ensuring all events for the same order go to the same partition — guaranteeing per-order ordering."

**Failure handling:**
> "Failed messages are retried with exponential backoff. After 3 retries, messages go to a dead letter topic for manual review and replay."

---

## Key Takeaways

```
Pub/Sub: Decouple publishers from subscribers via broker
         Publisher → topic → subscriber(s)
         Publisher doesn't know subscribers exist
         Subscribers don't know publishers exist

vs Message Queue:
  Queue:   1 sender → 1 receiver (work distribution)
  Pub/Sub: 1 sender → N receivers (event broadcast)
  Kafka:   Both — groups get all messages, within group load balanced

Redis Pub/Sub:
  Simple, in-memory, fire-and-forget
  No persistence, no consumer groups, no replay
  Use: cache invalidation, real-time signals
  Not for: reliable event processing

Kafka Pub/Sub:
  Durable, replayable, consumer groups
  At-least-once or exactly-once delivery
  Consumer groups: each group gets all messages
  Within group: load balanced across consumers
  Use: everything important

Key patterns:
  Fan-Out:       one event → many independent workflows
  Event Sourcing: Kafka as immutable audit log + source of truth
  DLQ:           failed messages → dlq topic → manual review
  Saga:          distributed transactions via event choreography

Delivery guarantees:
  At-most-once:  fast, possible loss
  At-least-once: safe, possible duplicates → consumer must be idempotent
  Exactly-once:  Kafka transactions, highest guarantee

Backpressure:
  Kafka lag = producer - consumer pace difference
  Solution: scale consumers up to partition count
            scale partitions → more consumer parallelism
```
