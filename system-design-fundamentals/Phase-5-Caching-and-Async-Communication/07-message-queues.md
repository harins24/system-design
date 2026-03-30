# Message Queues

**Phase 5 — Caching + Async Communication | Topic 7**

---

## What is a Message Queue?

A message queue is a buffer that holds messages between a sender (producer) and receiver (consumer), allowing them to communicate asynchronously without being connected at the same time.

```
Without message queue (synchronous):
  User clicks "Place Order"
  → Order Service calls Payment Service (wait 200ms)
  → Order Service calls Inventory Service (wait 100ms)
  → Order Service calls Email Service (wait 500ms)
  → User waits 800ms total
  → If Email Service is down → entire order fails

With message queue (asynchronous):
  User clicks "Place Order"
  → Order Service saves order to DB
  → Order Service puts message in queue
  → Returns "Order Received" to user in 50ms

  Queue delivers message to:
  → Payment Service (processes when ready)
  → Inventory Service (processes when ready)
  → Email Service (processes when it comes back up)

  User gets fast response
  Downstream services process at their own pace
  Email Service down? Message waits in queue
```

---

## Core Message Queue Concepts

```
Producer:   Sends messages to queue
Consumer:   Reads and processes messages from queue
Queue:      Buffer holding messages until consumed
Message:    Unit of data being communicated
Broker:     Software managing the queue
            (RabbitMQ, SQS, ActiveMQ)

Message lifecycle:
  1. Producer sends message → broker stores it
  2. Consumer reads message → broker marks "in-flight"
  3. Consumer processes message
  4. Consumer ACKs → broker deletes message
  OR
  4. Consumer NACKs / timeout → broker re-queues
```

---

## Message Queue vs Pub/Sub

```
Message Queue:              Pub/Sub:
  One message               One message
  One consumer              Multiple consumers
  Message deleted after     Message retained/replayed
  consumption
  Work distribution         Event notification

  "Do this task"            "This thing happened"

  Example:                  Example:
  Image resize job          OrderCreated event
  → one worker resizes      → payment, inventory, email
    each image                all process independently
```

---

## RabbitMQ — The Traditional Message Broker

RabbitMQ implements the AMQP protocol and is the most widely deployed traditional message broker.

### Core Concepts

```
Exchange:  Receives messages from producers
           Routes messages to queues based on rules

Queue:     Stores messages for consumers

Binding:   Rule connecting exchange to queue
           Based on routing key or pattern

Routing:
  Producer → Exchange → (routing rules) → Queue → Consumer

Types of exchanges:
  Direct:   Route to queue with exact matching routing key
  Fanout:   Route to ALL bound queues (broadcast)
  Topic:    Route by pattern matching (wildcards)
  Headers:  Route by message header attributes
```

### Exchange Types in Detail

```
Direct Exchange:
  Routing key "payment" → payment-queue only
  Routing key "email"   → email-queue only

  Producer sends: {routing_key: "payment", message: ...}
  → Goes to queue bound with "payment" binding

  Use: task routing to specific workers

Fanout Exchange:
  ALL bound queues receive copy of every message
  Routing key ignored

  Producer sends order event
  → payment-queue receives copy
  → inventory-queue receives copy
  → email-queue receives copy

  Use: broadcast (equivalent to pub/sub)

Topic Exchange:
  Pattern matching with wildcards:
  * = exactly one word
  # = zero or more words

  Queue bindings:
  "order.*"      matches: order.created, order.shipped
                 no match: order.payment.processed
  "payment.#"    matches: payment.succeeded, payment.card.declined
  "#"            matches everything

  Use: flexible routing by event hierarchy
```

### RabbitMQ — Spring Boot

```java
@Configuration
public class RabbitMQConfig {

    // Declare queues
    @Bean
    public Queue paymentQueue() {
        return QueueBuilder.durable("payment-queue")
            .withArgument("x-dead-letter-exchange", "dlx")
            .withArgument("x-dead-letter-routing-key", "payment.failed")
            .withArgument("x-message-ttl", 300_000) // 5 min TTL
            .build();
    }

    // Declare exchange
    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order-events");
    }

    // Bind queue to exchange with routing pattern
    @Bean
    public Binding paymentBinding(Queue paymentQueue,
                                   TopicExchange orderExchange) {
        return BindingBuilder
            .bind(paymentQueue)
            .to(orderExchange)
            .with("order.created");  // routing key pattern
    }
}

// Producer
@Service
public class OrderEventPublisher {

    private final RabbitTemplate rabbitTemplate;

    public void publishOrderCreated(Order order) {
        OrderCreatedMessage message = OrderCreatedMessage.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .total(order.getTotal())
            .build();

        rabbitTemplate.convertAndSend(
            "order-events",    // exchange
            "order.created",   // routing key
            message            // payload
        );
    }
}

// Consumer
@Component
public class PaymentConsumer {

    @RabbitListener(queues = "payment-queue")
    public void processPayment(OrderCreatedMessage message,
                                Channel channel,
                                @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        try {
            paymentService.processPayment(message);
            channel.basicAck(tag, false); // acknowledge success
        } catch (PaymentException e) {
            channel.basicNack(tag, false, false); // reject → DLQ
        } catch (TransientException e) {
            channel.basicNack(tag, false, true);  // reject → requeue
        }
    }
}
```

### RabbitMQ Delivery Guarantees

```
At-Most-Once (autoAck=true):
  Message deleted from queue when delivered (not when processed)
  Fast, but if consumer crashes mid-processing → message lost

At-Least-Once (manual ACK):
  Message stays "in-flight" until consumer ACKs
  Consumer crashes → message re-queued after timeout
  Consumer must be idempotent (may process twice)

Exactly-Once:
  Not natively supported by RabbitMQ
  Approximate with: idempotency key in message + DB dedup table
```

---

## Amazon SQS — Managed Queue

AWS's fully managed message queue service. No servers to operate.

```
Two queue types:

Standard Queue:
  Unlimited throughput
  At-least-once delivery
  Best-effort ordering (may be out of order)
  Use: most use cases, high throughput needed

FIFO Queue:
  300 msg/sec (or 3,000 with batching)
  Exactly-once delivery
  Strict ordering preserved
  Use: financial transactions, order processing
       anything where order and deduplication matter
```

### SQS Key Features

```
Visibility Timeout:
  When consumer reads message → message hidden from others
  Default: 30 seconds
  Consumer must ACK (delete) before timeout
  Timeout expires → message visible again → another consumer picks up

  Use: prevents two consumers processing same message
       Set to 6x your expected processing time

Long Polling:
  Consumer waits up to 20 seconds for messages
  Reduces empty responses and API costs
  vs Short Polling: returns immediately even if no messages

Message Retention:
  Default: 4 days
  Max: 14 days
  Messages automatically deleted after retention period

Dead Letter Queue (DLQ):
  After N failed processing attempts → moved to DLQ
  maxReceiveCount: how many times to try before DLQ
  Alert on DLQ depth for monitoring

Large Messages:
  SQS limit: 256KB per message
  For larger: use S3 Extended Client Library
  Stores message body in S3, sends S3 pointer in SQS
```

### SQS — Spring Boot

```java
@Configuration
public class SQSConfig {

    @Bean
    public SqsAsyncClient sqsAsyncClient() {
        return SqsAsyncClient.builder()
            .region(Region.US_EAST_1)
            .build();
    }
}

// Producer
@Service
public class OrderQueueService {

    private final SqsAsyncClient sqsClient;

    @Value("${sqs.order-queue-url}")
    private String queueUrl;

    public void sendOrderMessage(Order order) {
        String messageBody = objectMapper.writeValueAsString(order);

        SendMessageRequest request = SendMessageRequest.builder()
            .queueUrl(queueUrl)
            .messageBody(messageBody)
            .messageGroupId(order.getUserId())      // FIFO: group by user
            .messageDeduplicationId(order.getId())  // FIFO: dedup by order
            .build();

        sqsClient.sendMessage(request);
    }
}

// Consumer (Spring Cloud AWS)
@SqsListener(value = "${sqs.order-queue-url}",
             deletionPolicy = SqsMessageDeletionPolicy.ON_SUCCESS)
public void processOrder(Order order,
                          @Headers MessageHeaders headers) {
    try {
        orderService.processOrder(order);
        // Message automatically deleted on method return (ON_SUCCESS policy)
    } catch (RetryableException e) {
        throw e; // Message returns to queue for retry
    } catch (PermanentException e) {
        log.error("Permanent failure, message will go to DLQ");
        // Don't throw — let it exhaust retries and move to DLQ
    }
}
```

---

## Queue Patterns

### Work Queue (Task Distribution)

```
Multiple workers share a single queue:
  Producer → [Queue] → Worker 1
                     → Worker 2
                     → Worker 3

Each message processed by exactly ONE worker
Workers compete for messages
Automatically load-balanced — busy workers get fewer messages

Use: CPU-intensive tasks (image processing, video encoding)
     Any work that can be parallelized

Example: Image resizing service
  Upload Service: PUBLISH {imageId, s3Key, targetSizes}
  Worker 1: picks up message → resizes → ACKs
  Worker 2: picks up next message → resizes → ACKs
  Workers scale independently based on queue depth
```

### Priority Queue

```
Higher priority messages processed first:

RabbitMQ priority queue:
  @Bean
  public Queue priorityQueue() {
      return QueueBuilder.durable("priority-queue")
          .withArgument("x-max-priority", 10) // 0-10 priority levels
          .build();
  }

  // Producer sets priority
  MessageProperties props = new MessageProperties();
  props.setPriority(8); // high priority
  rabbitTemplate.send("priority-queue", new Message(body, props));

Use cases:
  Premium customers → higher priority order processing
  Critical alerts → processed before informational alerts
  Emergency jobs → jump queue ahead of batch work
```

### Request-Reply Pattern

```
Synchronous RPC over async queue:

  Client:
    1. Creates reply-to queue (temporary, unique per request)
    2. Sends request: {correlationId, replyTo: "reply-queue-xyz"}
    3. Waits on reply queue (with timeout)

  Server:
    1. Processes request
    2. Sends response to replyTo queue with same correlationId

  Client:
    3. Receives response matching correlationId

Use: when you need response but want async/decoupled infrastructure
     gRPC/REST is usually better — use this only when messaging is required
```

### Competing Consumers

```
Scale processing by adding consumers:

Queue: [msg1, msg2, msg3, msg4, msg5, msg6]

1 consumer:    processes 1 msg/sec → backlog builds
Add consumer:  2 msgs/sec → backlog drains
Add more:      scales linearly up to message arrival rate

Auto-scaling example (AWS):
  SQS queue depth > 1000 messages → scale out consumer group
  SQS queue depth < 100 messages  → scale in

  CloudWatch alarm → Auto Scaling Group → ECS tasks
```

---

## Queue Reliability Patterns

### Idempotent Consumer

```
Messages may be delivered more than once (at-least-once).
Consumer must handle duplicates:
```

```java
@Service
public class IdempotentPaymentConsumer {

    @Transactional
    public void processPayment(PaymentMessage message) {
        String idempotencyKey = message.getIdempotencyKey();

        // Check if already processed
        if (processedPayments.existsByIdempotencyKey(idempotencyKey)) {
            log.info("Duplicate message ignored: {}", idempotencyKey);
            return; // Already processed, skip
        }

        // Process payment
        paymentService.charge(message.getAmount(), message.getCardToken());

        // Mark as processed
        processedPayments.save(new ProcessedPayment(idempotencyKey));
    }
}
```

### Poison Message Handling

```
Some messages can never be processed successfully:
  Malformed data
  Logical error (product doesn't exist)
  Bug in consumer code

Without handling:
  Consumer fails → message requeued → fails again
  Infinite retry loop → blocks entire queue

Solution: Max retry count → DLQ

RabbitMQ:
  x-delivery-count header tracks attempts
  After N attempts → route to dead-letter exchange

SQS:
  maxReceiveCount = 3 → after 3 failures → DLQ

DLQ monitoring:
  CloudWatch alarm: DLQ depth > 0 → alert on-call
  Human reviews malformed messages
  Fix code → replay from DLQ → reprocess
```

---

## Kafka vs RabbitMQ vs SQS

```
Property          Kafka              RabbitMQ           SQS
──────────────────────────────────────────────────────────────────
Model             Event log          Message broker     Managed queue
Retention         Configurable       Until consumed     Up to 14 days
                  (default 7 days)
Replay            Yes (from offset)  No                 No
Throughput        Very high          High               High
Ordering          Per partition      Per queue          FIFO queue only
Consumer groups   Yes (powerful)     Manual setup       Not native
Managed service   No (self-host)     No (self-host)     Yes (AWS)
                  (MSK = managed)    (CloudAMQP)
Pub/sub           Yes (groups)       Yes (exchanges)    No
Routing           Partition key      Exchange patterns  Basic filters
Exactly-once      Yes (transactions) No                 FIFO queues
Use when          Event sourcing,    Complex routing,   AWS ecosystem,
                  high throughput,   RPC patterns,      serverless,
                  replay needed      legacy systems     managed ops
```

---

## When to Use Message Queues

```
Use message queues when:

Decoupling required:
  Services should not know about each other
  Allows independent deployment and scaling

Async processing acceptable:
  User doesn't need immediate result
  "We'll process your order shortly"

Load leveling:
  Traffic spikes → queue absorbs burst
  Consumers process at steady rate
  Black Friday spike → queue grows → consumers drain it

Resilience:
  Downstream service down? Messages wait in queue
  Not lost, processed when service recovers

Expensive operations:
  Video encoding, PDF generation, ML inference
  Don't make user wait → queue it → notify when done

Retry logic:
  Failed operations automatically retried
  Configurable backoff, DLQ for permanent failures
```

---

## Key Takeaways

```
Message Queue: Async buffer between producer and consumer
               Decouples timing, scale, and failure

vs Pub/Sub:
  Queue:   one consumer gets message (work distribution)
  Pub/Sub: all groups get message (event notification)

RabbitMQ:
  Exchange → Queue → Consumer
  Exchanges: Direct (exact key), Fanout (all), Topic (pattern)
  Manual ACK for at-least-once
  Rich routing, AMQP protocol
  Use: complex routing, RPC, traditional message brokering

Amazon SQS:
  Standard: high throughput, at-least-once, best-effort order
  FIFO: 300/sec, exactly-once, strict order
  Visibility timeout prevents double processing
  DLQ after maxReceiveCount failures
  Use: AWS ecosystem, serverless, no ops overhead

Patterns:
  Work Queue:           competing consumers share load
  Priority Queue:       high priority first
  Request-Reply:        sync RPC over async queue
  Idempotent Consumer:  handle duplicate delivery
  DLQ:                  poison messages don't block queue

Kafka vs RabbitMQ vs SQS:
  Kafka:    replay, throughput, consumer groups, event sourcing
  RabbitMQ: routing flexibility, RPC, AMQP ecosystem
  SQS:      managed, AWS native, serverless-friendly

Message queues provide:
  Decoupling, async processing, load leveling,
  resilience to downstream failures, retry logic
```
