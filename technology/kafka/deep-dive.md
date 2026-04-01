---
title: Kafka Deep Dive
layout: default
---

# # Kafka Deep Dive

---
title: Kafka Deep Dive
layout: default
---

Technical and scenario-based interview questions covering Kafka architecture, exactly-once semantics, and partitioning strategy.

---

## Q1. Kafka Architecture: Brokers, Partitions, Consumer Groups, and Replication

### Cluster Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Kafka Cluster                         │
│                                                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐             │
│  │ Broker 1 │  │ Broker 2 │  │ Broker 3 │             │
│  └──────────┘  └──────────┘  └──────────┘             │
│                                                          │
│  Topic: order-events (3 partitions, replication=2)      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │ Partition 0 │  │ Partition 1 │  │ Partition 2 │    │
│  │  Leader: B1 │  │  Leader: B2 │  │  Leader: B3 │    │
│  │ Replica: B2 │  │ Replica: B3 │  │ Replica: B1 │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘

Consumer Group: inventory-service
  Consumer 1 → Partition 0
  Consumer 2 → Partition 1
  Consumer 3 → Partition 2
```

### 1. Brokers

Kafka servers that store data and serve clients. Each broker is identified by a unique ID, contains topic partitions (some as leaders, some as replicas), and automatically handles replication and data distribution.

```java
@Configuration
public class KafkaConfig {

    @Bean
    public ProducerFactory<String, OrderEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();

        // Multiple brokers for high availability
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
                  "broker1:9092,broker2:9092,broker3:9092");

        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                  StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                  JsonSerializer.class);

        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

### 2. Topics and Partitions

A **topic** is a logical channel/category for messages. A **partition** is a physical division of a topic for parallelism. Messages within a partition are ordered, and each partition has one leader and N-1 replicas.

```bash
# Create topic with 3 partitions and replication factor 2
kafka-topics.sh --create \
  --topic order-events \
  --partitions 3 \
  --replication-factor 2 \
  --bootstrap-server localhost:9092

# Topic structure:
# Topic: order-events
# Partition 0: Leader: Broker 1, Replicas: [1, 2], ISR: [1, 2]
# Partition 1: Leader: Broker 2, Replicas: [2, 3], ISR: [2, 3]
# Partition 2: Leader: Broker 3, Replicas: [3, 1], ISR: [3, 1]
```

Messages are routed to partitions using the key:

```java
@Service
public class OrderProducer {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrder(OrderEvent event) {
        // Using order ID as key ensures all events for same order
        // go to same partition (ordering guaranteed).
        // Kafka uses: hash(key) % numPartitions
        String key = event.getOrderId();
        kafkaTemplate.send("order-events", key, event);

        // Without key (null) → round-robin distribution
    }
}
```

### 3. Consumer Groups

Each partition is consumed by exactly **one** consumer in a group, enabling horizontal scaling and load balancing. Different groups can independently consume the same topic.

```java
@Service
public class OrderConsumerService {

    // Consumer in "inventory-service" group
    @KafkaListener(
        topics = "order-events",
        groupId = "inventory-service",
        concurrency = "3" // 3 consumers in this group
    )
    public void consumeOrderEvent(OrderEvent event) {
        inventoryService.reserveInventory(event);
    }
}

// Another service consuming same topic independently
@Service
public class NotificationService {

    @KafkaListener(
        topics = "order-events",
        groupId = "notification-service"
    )
    public void sendNotification(OrderEvent event) {
        emailService.sendOrderConfirmation(event);
    }
}

// Consumer group assignment example:
// Topic: order-events (3 partitions)
//
// Group: inventory-service (3 consumers)
//   Consumer-1 → Partition 0
//   Consumer-2 → Partition 1
//   Consumer-3 → Partition 2
//
// Group: notification-service (2 consumers)
//   Consumer-1 → Partitions 0, 1
//   Consumer-2 → Partition 2
//
// Both groups independently consume ALL messages
```

**Consumer Group Rebalancing** — triggered when a new consumer joins, an existing consumer leaves or crashes, partition count changes, or a consumer is too slow (`max.poll.interval.ms` exceeded):

```java
@Bean
public ConsumerAwareRebalanceListener rebalanceListener() {
    return new ConsumerAwareRebalanceListener() {

        @Override
        public void onPartitionsRevokedBeforeCommit(
                Consumer<?, ?> consumer,
                Collection<TopicPartition> partitions) {
            // Called before rebalance — commit offsets
            logger.info("Partitions revoked: {}", partitions);
            consumer.commitSync();
        }

        @Override
        public void onPartitionsAssigned(
                Consumer<?, ?> consumer,
                Collection<TopicPartition> partitions) {
            // Called after rebalance — resume from committed offset
            logger.info("Partitions assigned: {}", partitions);
        }
    };
}
```

### 4. Replication and Fault Tolerance

```
Replication Factor = 3 (1 Leader + 2 Followers)

┌──────────────────────────────────────────────────────┐
│ Partition 0                                          │
│                                                      │
│ Leader: Broker 1        ISR: [1, 2, 3]              │
│ ┌─────────────────┐                                 │
│ │ Message 1       │ ─┐                               │
│ │ Message 2       │  │ Replicate                     │
│ │ Message 3       │  │                               │
│ └─────────────────┘  │                               │
│                       │                               │
│ Follower: Broker 2    ↓     Follower: Broker 3      │
│ ┌─────────────────┐       ┌─────────────────┐      │
│ │ Message 1       │       │ Message 1       │      │
│ │ Message 2       │       │ Message 2       │      │
│ │ Message 3       │       │ Message 3       │      │
│ └─────────────────┘       └─────────────────┘      │
└──────────────────────────────────────────────────────┘

If Leader (Broker 1) fails:
  → One of the ISR followers (Broker 2 or 3) becomes new leader
  → No data loss (all replicas are in-sync)
  → Clients automatically reconnect to new leader
```

**Producer Configuration for Durability:**

```java
@Configuration
public class KafkaProducerConfig {

    @Bean
    public ProducerFactory<String, OrderEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();

        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG,
                  "broker1:9092,broker2:9092,broker3:9092");

        // ACKS Configuration:
        // acks=0: No acknowledgment (fastest, data loss possible)
        // acks=1: Leader acknowledges (fast, loss if leader fails before replication)
        // acks=all: All ISR replicas acknowledge (slowest, no data loss)
        config.put(ProducerConfig.ACKS_CONFIG, "all");

        // Topic setting: min.insync.replicas=2
        // → At least 2 replicas must acknowledge
        // → If only 1 available → Producer gets NOT_ENOUGH_REPLICAS error

        config.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);
        config.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "snappy");

        return new DefaultKafkaProducerFactory<>(config);
    }
}
```

**ISR (In-Sync Replica) Management:**

```java
// Scenario 1: All replicas in sync
// Leader: Broker 1, ISR: [1, 2, 3] ✓ Healthy

// Scenario 2: One replica falls behind
// Leader: Broker 1, ISR: [1, 2] (Broker 3 removed from ISR)
// If acks=all, producer only waits for Broker 1 and 2

// Scenario 3: Leader fails
// New leader elected from ISR (Broker 2 becomes leader)
// Leader: Broker 2, ISR: [2, 3]
// Broker 1 will rejoin ISR after catching up
```

**Consumer Offset Management:**

```java
@Bean
public ConsumerFactory<String, OrderEvent> consumerFactory() {
    Map<String, Object> config = new HashMap<>();

    config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG,
              "broker1:9092,broker2:9092,broker3:9092");
    config.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processor");

    config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
    config.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 5000);

    // earliest: Read from beginning if no offset exists
    // latest:   Read only new messages
    // none:     Throw exception if no offset
    config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

    return new DefaultKafkaConsumerFactory<>(config);
}

// Offsets stored in __consumer_offsets topic:
// [order-processor, order-events, 0] → 12345
// [order-processor, order-events, 1] → 67890
// [order-processor, order-events, 2] → 11111
```

### Key Takeaways

- **Brokers** — Storage and serving layer; handle replication automatically
- **Partitions** — Enable parallelism; maintain message order within a partition
- **Consumer Groups** — Enable horizontal scaling and load balancing
- **Replication** — Provides fault tolerance; prevents data loss
- **ISR** — Ensures durability when combined with `acks=all`
- **Offsets** — Track consumer progress; enable resume-on-restart capability

---

## Q2. Exactly-Once Semantics: Idempotent Producers and Transactional Messaging

### Delivery Semantics

| Guarantee | Description |
|---|---|
| At-most-once | May lose messages — not acceptable for most use cases |
| At-least-once | No message loss, but duplicates possible |
| Exactly-once | No loss, no duplicates — hardest to achieve |

### Problem Statement

Without exactly-once semantics, duplicates arise on both sides:

```java
// Producer side:
producer.send(message); // Network timeout after Kafka receives but before ack
producer.send(message); // Retry → DUPLICATE in Kafka

// Consumer side:
consume(message);
processOrder(message);
// CRASH before committing offset
// On restart: same message consumed again → DUPLICATE processing
```

### 1. Idempotent Producer (Exactly-Once for a Single Partition)

Kafka assigns each producer a unique Producer ID (PID). The producer includes a sequence number with each message, and the broker deduplicates based on `[PID, Sequence Number]`. This works **only within a single partition**.

```java
@Configuration
public class IdempotentProducerConfig {

    @Bean
    public ProducerFactory<String, OrderEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();

        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        // Enable idempotence — automatically sets:
        //   acks=all
        //   retries=Integer.MAX_VALUE
        //   max.in.flight.requests.per.connection <= 5
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);

        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                  StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                  JsonSerializer.class);

        return new DefaultKafkaProducerFactory<>(config);
    }
}

@Service
public class OrderProducerService {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrder(OrderEvent event) {
        // First attempt:  PID=123, SeqNum=0, Message="Order-1"
        // Retry (timeout): PID=123, SeqNum=0, Message="Order-1"
        // Broker detects duplicate (same PID + SeqNum) → ignores retry

        kafkaTemplate.send("order-events", event.getOrderId(), event);
    }
}

// How Kafka detects duplicates:
// Broker maintains: Map<ProducerID, Map<Partition, SequenceNumber>>
//
// Incoming: PID=123, Partition=0, SeqNum=5
// Last seen SeqNum=4 → Accept (5 is next)
// Last seen SeqNum=5 → Reject (duplicate)
// Last seen SeqNum=6 → Error  (out of sequence, possible data loss)
```

**Limitations:**
- Only works within a single partition
- Only works within a single producer session (restart = new PID)
- Does not help with transactional writes across multiple topics/partitions

### 2. Transactional Messaging (Exactly-Once Across Multiple Partitions/Topics)

The producer uses a stable **Transactional ID** (preserved across restarts). All messages in a transaction are atomic — all succeed or all fail. Consumers only see messages after the transaction commits.

```java
@Configuration
public class TransactionalProducerConfig {

    @Bean
    public ProducerFactory<String, OrderEvent> producerFactory() {
        Map<String, Object> config = new HashMap<>();

        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        // Unique across application restarts
        config.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-producer-1");

        // Idempotence is automatically enabled with transactions
        config.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        config.put(ProducerConfig.ACKS_CONFIG, "all");

        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
                  StringSerializer.class);
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                  JsonSerializer.class);

        return new DefaultKafkaProducerFactory<>(config);
    }

    @Bean
    public KafkaTemplate<String, OrderEvent> kafkaTemplate() {
        KafkaTemplate<String, OrderEvent> template =
            new KafkaTemplate<>(producerFactory());
        template.setAllowNonTransactional(false);
        return template;
    }
}

@Service
public class OrderProcessingService {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    @Autowired
    private KafkaTemplate<String, InventoryEvent> inventoryTemplate;

    @Transactional("kafkaTransactionManager")
    public void processOrder(Order order) {
        // Both messages are sent atomically
        kafkaTemplate.send("order-events", order.getId(),
                          new OrderCreatedEvent(order));

        inventoryTemplate.send("inventory-events", order.getId(),
                               new InventoryReserveEvent(order.getItems()));

        if (order.getAmount().compareTo(BigDecimal.ZERO) < 0) {
            throw new InvalidOrderException("Negative amount");
            // Transaction aborted — neither message is sent
        }
        // Transaction committed — both messages become visible
    }

    @Bean
    public KafkaTransactionManager kafkaTransactionManager() {
        return new KafkaTransactionManager(producerFactory());
    }
}

// Transaction Protocol:
// 1. Producer sends BeginTransaction to Transaction Coordinator
// 2. Producer writes messages to brokers (marked as transactional)
// 3. Producer sends CommitTransaction or AbortTransaction
// 4. Coordinator writes transaction marker to topic
// 5. read_committed consumers see messages only after commit
```

### 3. Read-Committed Consumer

```java
@Bean
public ConsumerFactory<String, OrderEvent> consumerFactory() {
    Map<String, Object> config = new HashMap<>();

    config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    config.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processor");

    // Only read messages from committed transactions
    // Default is "read_uncommitted" (sees all messages including open transactions)
    config.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");

    config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
              StringDeserializer.class);
    config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
              JsonDeserializer.class);

    return new DefaultKafkaConsumerFactory<>(config);
}
```

### 4. Exactly-Once: Read-Process-Write Pattern

For the pattern "consume from topic A → process → produce to topic B", all three steps can be made exactly-once by combining a transactional producer with a transactional consumer:

```java
@Service
public class ExactlyOnceProcessor {

    @Autowired
    private KafkaTemplate<String, ProcessedEvent> kafkaTemplate;

    @Autowired
    private OrderRepository orderRepository;

    @Transactional("kafkaTransactionManager")
    @KafkaListener(
        topics = "input-topic",
        groupId = "exactly-once-processor",
        containerFactory = "kafkaListenerContainerFactory"
    )
    public void processMessage(
            @Payload OrderEvent inputEvent,
            @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
            @Header(KafkaHeaders.OFFSET) long offset,
            Acknowledgment acknowledgment) {

        // 1. Process message
        ProcessedEvent outputEvent = processOrder(inputEvent);

        // 2. Write to database (optional)
        orderRepository.save(toEntity(inputEvent));

        // 3. Send to output topic
        kafkaTemplate.send("output-topic", outputEvent.getId(), outputEvent);

        // 4. Spring Kafka automatically commits offset when transaction succeeds

        // All three operations are atomic:
        //   database write + kafka send + consumer offset commit
    }
}

@Configuration
public class ExactlyOnceConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent>
            kafkaListenerContainerFactory() {

        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties()
               .setAckMode(ContainerProperties.AckMode.MANUAL);
        return factory;
    }

    @Bean
    public ChainedKafkaTransactionManager<Object, Object> chainedTransactionManager(
            KafkaTransactionManager kafkaTransactionManager,
            PlatformTransactionManager dbTransactionManager) {

        // Chain Kafka and DB transactions for full atomicity
        return new ChainedKafkaTransactionManager<>(
            kafkaTransactionManager,
            dbTransactionManager
        );
    }
}
```

### 5. Application-Level Idempotency (Defence-in-Depth)

Even with Kafka exactly-once, implementing application-level idempotency provides an additional safety layer:

```java
@Entity
@Table(name = "processed_messages")
public class ProcessedMessage {
    @Id
    private String messageId;  // Unique identifier
    private String topic;
    private Integer partition;
    private Long offset;
    private Instant processedAt;
}

@Service
public class IdempotentOrderProcessor {

    @Autowired
    private ProcessedMessageRepository processedRepo;

    @Autowired
    private OrderRepository orderRepository;

    @KafkaListener(topics = "order-events", groupId = "order-processor")
    @Transactional
    public void processOrder(OrderEvent event,
                            @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition,
                            @Header(KafkaHeaders.OFFSET) long offset) {

        String messageId = event.getOrderId() + "-" + partition + "-" + offset;

        if (processedRepo.existsById(messageId)) {
            logger.info("Message already processed: {}", messageId);
            return; // Skip duplicate
        }

        // Process and save atomically
        Order order = new Order(event.getOrderId(), event.getAmount());
        orderRepository.save(order);

        ProcessedMessage processed = new ProcessedMessage();
        processed.setMessageId(messageId);
        processed.setTopic("order-events");
        processed.setPartition(partition);
        processed.setOffset(offset);
        processed.setProcessedAt(Instant.now());
        processedRepo.save(processed);
        // If crash occurs, transaction rolls back → replay is safe
    }
}

// Retention policy: clean up old deduplication records
@Scheduled(cron = "0 0 2 * * ?") // Daily at 2 AM
public void cleanupOldMessages() {
    Instant cutoff = Instant.now().minus(Duration.ofDays(7));
    processedRepo.deleteByProcessedAtBefore(cutoff);
}
```

### Comparison

| Approach | Scope | Survives Producer Restart | Multi-Partition | Multi-Topic | DB Support |
|---|---|---|---|---|---|
| Idempotent Producer | Single partition | ✗ New PID on restart | ✗ | ✗ | ✗ |
| Transactional Messaging | Multiple | ✓ Same Transactional ID | ✓ | ✓ | ✗ |
| Application Idempotency | Multiple | ✓ | ✓ | ✓ | ✓ |

### Best Practices

- Use **Idempotent Producer** by default — minimal overhead
- Use **Transactions** when atomicity across topics/partitions is needed
- Implement **Application Idempotency** for the most critical operations
- Use unique message IDs (UUID or composite keys)
- Set appropriate retention for deduplication tables
- Monitor transaction latency — transactions add overhead
- Test failure scenarios extensively

### Common Pitfalls

```java
// PITFALL 1: Consumer not set to read_committed
// Producer uses transactions, consumer uses default read_uncommitted
// → Consumer sees uncommitted messages → not exactly-once!

// PITFALL 2: Long-running transaction exceeds transaction.timeout.ms
// → Transaction automatically aborted → messages lost

// PITFALL 3: Multiple producers sharing the same transactional.id
// → Causes fencing, unpredictable behavior
// → Each producer instance needs a unique transactional.id

// PITFALL 4: Assuming Kafka guarantees alone are sufficient
// Network issues and consumer crashes can still cause duplicates
// → Always pair with application-level idempotency for critical flows
```

---

## Q3. Kafka Topics, Partitions, and Consumer Groups — Partitioning Strategy

Partitioning is critical for parallelism (scale consumers horizontally), ordering (messages within a partition are ordered), and load distribution (balance across consumers).

### 1. Key-Based Partitioning (Default)

```java
// Kafka's default: partition = hash(key) % number_of_partitions
// Same key → Same partition → Ordering guaranteed

@Service
public class OrderEventProducer {

    @Autowired
    private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderEvent(OrderEvent event) {
        // With 3 partitions:
        // Order-123 → hash → 12345 % 3 = 0 → Partition 0
        // Order-456 → hash → 67890 % 3 = 0 → Partition 0
        // Order-789 → hash → 11111 % 3 = 1 → Partition 1
        kafkaTemplate.send("order-events", event.getOrderId(), event);
    }
}

// Real-world example: Product Catalog Updates
@Service
public class ProductUpdateProducer {

    public void publishProductUpdate(ProductEvent event) {
        // All updates for the same product go to the same partition
        // Consumer reads in order: $10 → $12 → $11 ✓ Correct sequence
        kafkaTemplate.send("product-updates", event.getSku(), event);
    }
}
```

### 2. Round-Robin Partitioning (No Key)

```java
@Service
public class LogEventProducer {

    public void publishLogEvent(LogEvent event) {
        // No key → round-robin distribution
        // Message 1 → P0, Message 2 → P1, Message 3 → P2, Message 4 → P0 ...
        // Use when: order doesn't matter, even load distribution is the goal
        kafkaTemplate.send("log-events", event);
    }
}
```

### 3. Custom Partitioner

```java
// Route stores by geographic region
public class StorePartitioner implements Partitioner {

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes,
                        Cluster cluster) {

        List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
        int numPartitions = partitions.size();

        if (key == null) {
            return ThreadLocalRandom.current().nextInt(numPartitions);
        }

        int storeNumber = Integer.parseInt((String) key);

        // Stores 001-100 → Partition 0 (West)
        // Stores 101-200 → Partition 1 (East)
        // Stores 201-300 → Partition 2 (Central)
        if (storeNumber <= 100) return 0;
        if (storeNumber <= 200) return 1;
        return 2;
    }

    @Override public void close() {}
    @Override public void configure(Map<String, ?> configs) {}
}

// Wire up the custom partitioner
@Bean
public ProducerFactory<String, InventoryEvent> producerFactory() {
    Map<String, Object> config = new HashMap<>();
    config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    config.put(ProducerConfig.PARTITIONER_CLASS_CONFIG,
              StorePartitioner.class.getName());
    config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG,
              StringSerializer.class);
    config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
              JsonSerializer.class);
    return new DefaultKafkaProducerFactory<>(config);
}

// Usage:
// Store-050 → Partition 0 (West)
// Store-150 → Partition 1 (East)
// Store-250 → Partition 2 (Central)
```

### 4. Sticky Partitioning (Batching Optimisation, Kafka 2.4+)

For null-key messages, sticky partitioning batches messages to the same partition until the batch is full or a timeout is reached, then switches to the next partition. This reduces network requests and improves throughput.

```java
@Bean
public ProducerFactory<String, LogEvent> producerFactory() {
    Map<String, Object> config = new HashMap<>();
    config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    config.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384); // 16 KB
    config.put(ProducerConfig.LINGER_MS_CONFIG, 10);     // Wait 10ms to fill batch
    return new DefaultKafkaProducerFactory<>(config);
}

// Without sticky: Msg1→P0, Msg2→P1, Msg3→P2, Msg4→P0 ... (small batches, more requests)
// With sticky:    Msg1–Msg10→P0 (batch full), Msg11–Msg20→P1, Msg21–Msg30→P2
```

### Scenario: Product Catalog Update Partitioning

**Setup:** E-commerce platform with 10,000 products, multiple updates per product per day, topic `product-catalog-updates` (10 partitions).

```java
@Service
public class ProductCatalogProducer {

    @Autowired
    private KafkaTemplate<String, ProductUpdateEvent> kafkaTemplate;

    // Strategy 1: Partition by SKU (BEST for ordering)
    public void publishProductUpdate(ProductUpdateEvent event) {
        kafkaTemplate.send("product-catalog-updates", event.getSku(), event);
        // All updates for the same product go to the same partition
        // SKU-123: $10 → $12 → Stock 100 (all on Partition 3)
        // Different products processed in parallel across partitions
    }

    // Strategy 2: Partition by Category (for category-level aggregations)
    public void publishByCategoryProductUpdate(ProductUpdateEvent event) {
        kafkaTemplate.send("product-catalog-updates", event.getCategoryId(), event);
        // Trade-off: less parallelism if there are few categories
    }

    // Strategy 3: Composite key (category + subcategory)
    public void publishCompositeKeyUpdate(ProductUpdateEvent event) {
        String key = event.getCategoryId() + "-" + event.getSubcategoryId();
        kafkaTemplate.send("product-catalog-updates", key, event);
        // Electronics-Phones → Partition X
        // Electronics-Laptops → Partition Y
        // Clothing-Shirts → Partition Z
    }
}

// Consumer — 5 consumers across 10 partitions
@KafkaListener(
    topics = "product-catalog-updates",
    groupId = "catalog-indexer",
    concurrency = "5" // Consumer 1 → P0,P1 | Consumer 2 → P2,P3 | etc.
)
public void updateCatalog(
        ProductUpdateEvent event,
        @Header(KafkaHeaders.RECEIVED_PARTITION_ID) int partition) {
    searchIndexService.updateProduct(event);
    // Ordering guaranteed within partition — same SKU always processed sequentially
}
```

### Choosing Partition Count

```java
// Formula: Partitions = max(Tp, Tc)
// Tp = Target throughput / Producer throughput per partition
// Tc = Target throughput / Consumer throughput per partition

// Example:
// Target: 1,000,000 messages/hour = ~277 msg/sec
// Producer throughput per partition: 50 msg/sec
// Consumer throughput per partition: 30 msg/sec
//
// Tp = 277 / 50 = 5.5 → 6 partitions
// Tc = 277 / 30 = 9.2 → 10 partitions
// Choose: 10 (higher of the two — consumer-bound)
```

```bash
kafka-topics.sh --create \
  --topic product-catalog-updates \
  --partitions 10 \
  --replication-factor 3 \
  --bootstrap-server localhost:9092
```

Considerations: more partitions means more parallelism but also more overhead. You can increase partitions later but cannot decrease them. Kafka recommends keeping partitions below 4,000 per broker.

### Partition Key Design Patterns

```java
// Pattern 1: Entity ID (most common)
key = order.getId();           // Order processing
key = user.getUserId();        // User activity
key = sku;                     // Product updates

// Pattern 2: Composite Key (grouped entities)
key = customerId + "-" + orderId;  // All orders for a customer together
key = storeId + "-" + sku;         // Store-specific inventory

// Pattern 3: Tenant ID (multi-tenancy)
key = tenantId;                // Isolate tenants to partitions

// Pattern 4: Geographic Region
key = regionCode;              // Region-specific processing

// Pattern 5: Time-based (use with caution — hot partitions)
key = dateHour;                // Groups by hour, may cause skew

// Anti-patterns:
// ❌ key = timestamp     — all recent messages go to same partition (hot spot)
// ❌ key = random()      — loses all ordering benefits
// ❌ key = constantValue — all messages to one partition (no parallelism)
```

### Handling Hot Partitions

```java
// Problem: popular products get far more updates than others
// → one partition becomes overloaded

@Service
public class LoadBalancedProducer {

    public void publishProductUpdate(ProductUpdateEvent event) {
        String sku = event.getSku();

        if (hotProductCache.contains(sku)) {
            // Add random suffix to spread the hot product across 10 partitions
            String key = sku + "-" + (System.currentTimeMillis() % 10);
            // Trade-off: loses strict ordering for hot products
            kafkaTemplate.send("product-updates", key, event);
        } else {
            // Normal products: maintain ordering with standard key
            kafkaTemplate.send("product-updates", sku, event);
        }
    }
}
```

### Best Practices

- Use **entity ID as key** for most use cases — balances ordering with parallelism
- **Calculate partition count** based on throughput requirements before creating the topic
- **Monitor partition lag** — adjust consumers if load is uneven across partitions
- **Avoid hot partitions** — use composite keys or random suffixes for high-volume keys
- **Don't change partitioning logic** after going to production — requires data migration
- Keep partition count reasonable — more is not always better
- **Consider rebalance impact** when scaling consumers up or down

---

## Q4. Handling Kafka Consumer Lag in High-Volume Scenarios

### What is Consumer Lag?

Consumer lag occurs when the producer writes faster than the consumer can process:

```
Producer writes at 10,000 msg/sec
Consumer processes at  6,000 msg/sec
Lag grows at           4,000 msg/sec

Topic: order-events
  Partition 0:
    Latest Offset:    50,000  (Producer position)
    Consumer Offset:  30,000  (Consumer position)
    LAG:              20,000  ← Problem!
```

### Measuring Consumer Lag

```bash
# Check lag using Kafka CLI
kafka-consumer-groups.sh \
  --bootstrap-server localhost:9092 \
  --group inventory-service \
  --describe

# Output:
# GROUP             TOPIC         PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# inventory-service order-events  0          30000           50000           20000
# inventory-service order-events  1          28000           48000           20000
# inventory-service order-events  2          25000           45000           20000
```

**Programmatic Lag Monitoring:**

```java
@Service
public class KafkaLagMonitor {

    @Autowired
    private AdminClient adminClient;

    @Autowired
    private MeterRegistry meterRegistry;

    @Scheduled(fixedDelay = 30000) // Every 30 seconds
    public void monitorLag() {
        Map<TopicPartition, Long> endOffsets = getEndOffsets("order-events");
        Map<TopicPartition, OffsetAndMetadata> committedOffsets =
            getCommittedOffsets("inventory-service", "order-events");

        for (Map.Entry<TopicPartition, Long> entry : endOffsets.entrySet()) {
            TopicPartition partition = entry.getKey();
            long endOffset = entry.getValue();
            long committedOffset = committedOffsets
                .getOrDefault(partition, new OffsetAndMetadata(0))
                .offset();

            long lag = endOffset - committedOffset;

            meterRegistry.gauge(
                "kafka.consumer.lag",
                Tags.of(
                    "topic", partition.topic(),
                    "partition", String.valueOf(partition.partition()),
                    "group", "inventory-service"
                ),
                lag
            );

            if (lag > 100000) {
                alertService.sendAlert(
                    "High Kafka lag detected",
                    "Partition " + partition + " lag: " + lag
                );
            }
        }
    }
}
```

### Strategies to Reduce Consumer Lag

#### Strategy 1: Increase Consumer Parallelism

```java
@Service
public class InventoryConsumer {

    // Increase concurrency to match partition count
    @KafkaListener(
        topics = "order-events",
        groupId = "inventory-service",
        concurrency = "6" // One thread per partition
    )
    public void consumeOrderEvent(OrderEvent event) {
        inventoryService.processOrder(event);
    }
}
```

```yaml
# application.yml
spring:
  kafka:
    listener:
      concurrency: 6     # Match partition count
      type: single       # or batch for bulk processing
      poll-timeout: 3000
# Rule: concurrency <= number of partitions
# If concurrency > partitions, extra consumers sit idle
```

#### Strategy 2: Batch Processing

```java
@Configuration
public class BatchConsumerConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent>
            batchKafkaListenerContainerFactory() {

        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();

        factory.setConsumerFactory(consumerFactory());
        factory.setBatchListener(true);
        factory.setConcurrency(6);

        return factory;
    }

    @Bean
    public ConsumerFactory<String, OrderEvent> consumerFactory() {
        Map<String, Object> config = new HashMap<>();

        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.GROUP_ID_CONFIG, "inventory-service");

        // Fetch more messages per poll
        config.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 500);
        config.put(ConsumerConfig.FETCH_MIN_BYTES_CONFIG, 1024 * 1024); // 1MB minimum
        config.put(ConsumerConfig.FETCH_MAX_WAIT_MS_CONFIG, 500);       // Wait up to 500ms

        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG,
                  StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  JsonDeserializer.class);

        return new DefaultKafkaConsumerFactory<>(config);
    }
}

@Service
public class BatchInventoryConsumer {

    @Autowired
    private InventoryRepository inventoryRepository;

    @KafkaListener(
        topics = "order-events",
        groupId = "inventory-service",
        containerFactory = "batchKafkaListenerContainerFactory"
    )
    public void consumeBatch(List<OrderEvent> events) {
        logger.info("Processing batch of {} events", events.size());

        // Group by SKU for bulk database update
        Map<String, List<OrderEvent>> eventsBySku = events.stream()
            .collect(Collectors.groupingBy(OrderEvent::getSku));

        List<InventoryUpdate> updates = eventsBySku.entrySet().stream()
            .map(entry -> {
                String sku = entry.getKey();
                int totalQuantity = entry.getValue().stream()
                    .mapToInt(OrderEvent::getQuantity)
                    .sum();
                return new InventoryUpdate(sku, totalQuantity);
            })
            .collect(Collectors.toList());

        // Single bulk database call instead of N calls
        inventoryRepository.bulkUpdate(updates);
    }
}
```

#### Strategy 3: Async Processing

```java
@Service
public class AsyncInventoryConsumer {

    @Autowired
    private TaskExecutor taskExecutor;

    @KafkaListener(
        topics = "order-events",
        groupId = "inventory-service",
        concurrency = "6"
    )
    public void consume(OrderEvent event, Acknowledgment ack) {
        // Submit to thread pool — consumer thread is freed immediately
        CompletableFuture
            .runAsync(() -> inventoryService.processOrder(event), taskExecutor)
            .thenRun(() -> ack.acknowledge())
            .exceptionally(ex -> {
                logger.error("Failed to process event", ex);
                return null;
            });
    }

    @Bean
    public ThreadPoolTaskExecutor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(20);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(500);
        executor.setThreadNamePrefix("inventory-processor-");
        return executor;
    }
}
```

#### Strategy 4: Increase Partition Count

```bash
# Step 1: Increase partitions (one-time operation — cannot decrease)
kafka-topics.sh --alter \
  --topic order-events \
  --partitions 12 \
  --bootstrap-server localhost:9092
```

```java
// Step 2: Update consumer concurrency to match
@KafkaListener(
    topics = "order-events",
    groupId = "inventory-service",
    concurrency = "12"
)
public void consume(OrderEvent event) {
    inventoryService.processOrder(event);
}

// Warning:
// - Partition count can only increase, never decrease
// - Increasing partitions changes message routing
// - Key-based ordering may be affected
```

#### Strategy 5: Optimise Consumer Processing

```java
@Service
public class OptimizedInventoryConsumer {

    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void consume(List<OrderEvent> events) {

        // Optimization 1: Deduplicate within batch (keep latest per SKU)
        Map<String, OrderEvent> deduplicated = events.stream()
            .collect(Collectors.toMap(
                OrderEvent::getSku,
                e -> e,
                (existing, replacement) -> replacement
            ));

        List<String> skus = new ArrayList<>(deduplicated.keySet());

        // Optimization 2: Fetch from Redis cache (faster than DB)
        Map<String, Inventory> cachedInventories =
            redisTemplate.opsForValue().multiGet(skus).stream()
                .filter(Objects::nonNull)
                .collect(Collectors.toMap(Inventory::getSku, i -> i));

        // Optimization 3: Only DB lookup for cache misses
        List<String> cacheMisses = skus.stream()
            .filter(sku -> !cachedInventories.containsKey(sku))
            .collect(Collectors.toList());

        Map<String, Inventory> dbInventories =
            inventoryRepository.findAllBySkuIn(cacheMisses).stream()
                .collect(Collectors.toMap(Inventory::getSku, i -> i));

        // Optimization 4: Bulk update DB
        List<Inventory> toUpdate = new ArrayList<>();
        for (Map.Entry<String, OrderEvent> entry : deduplicated.entrySet()) {
            Inventory inv = cachedInventories.getOrDefault(
                entry.getKey(), dbInventories.get(entry.getKey()));
            if (inv != null) {
                inv.decrementQuantity(entry.getValue().getQuantity());
                toUpdate.add(inv);
            }
        }
        inventoryRepository.saveAll(toUpdate); // Single bulk operation

        // Optimization 5: Update cache
        toUpdate.forEach(inv ->
            redisTemplate.opsForValue().set(inv.getSku(), inv, Duration.ofMinutes(10)));
    }
}
```

#### Strategy 6: Temporary Catch-Up Consumer

```java
// Deploy an additional consumer group at higher concurrency for catch-up.
// Normal: inventory-service (6 consumers)
// Catch-up: inventory-service-catchup (12 consumers)
// Once lag is cleared, decommission the catch-up group.
// Requires idempotent processing (both groups may process same messages).

@Service
@ConditionalOnProperty(name = "kafka.catchup.enabled", havingValue = "true")
public class CatchUpConsumer {

    @KafkaListener(
        topics = "order-events",
        groupId = "inventory-service-catchup",
        concurrency = "12"
    )
    public void consumeCatchUp(OrderEvent event) {
        if (!processedMessageRepository.exists(event.getId())) {
            inventoryService.processOrder(event);
            processedMessageRepository.markProcessed(event.getId());
        }
    }
}
```

#### Strategy 7: Consumer Lag Dashboard

```java
@RestController
@RequestMapping("/kafka/metrics")
public class KafkaMetricsController {

    @GetMapping("/lag")
    public ResponseEntity<Map<String, Object>> getLagMetrics() {
        Map<String, Long> lagByPartition = lagMonitor.getLagByPartition(
            "inventory-service", "order-events");

        long totalLag = lagByPartition.values().stream()
            .mapToLong(Long::longValue).sum();

        Map<String, Object> metrics = new HashMap<>();
        metrics.put("totalLag", totalLag);
        metrics.put("lagByPartition", lagByPartition);
        metrics.put("estimatedCatchUpTime",
            calculateCatchUpTime(totalLag, 6000)); // 6000 msg/sec

        return ResponseEntity.ok(metrics);
    }

    private String calculateCatchUpTime(long lag, long processingRate) {
        long seconds = lag / processingRate;
        return seconds + " seconds (" + (seconds / 60) + " minutes)";
    }
}
```

### Summary

| Strategy | Lag Reduction | Complexity | Risk | When to Use |
|---|---|---|---|---|
| Increase concurrency | High | Low | Low | First option always |
| Batch processing | High | Medium | Low | High-volume topics |
| Async processing | Medium | High | Medium | Slow per-message processing |
| More partitions | High | Low | Medium | Permanent scaling |
| Optimise processing | High | High | Low | Long-term fix |
| Catch-up consumer | High | Medium | Low | One-time catch-up |

---

## Q5. Kafka Offset Management: Auto-Commit vs Manual Commit

### What is an Offset?

```
Topic: order-events, Partition 0:

Offset:    0      1      2      3      4      5 ...
Message: [Msg1] [Msg2] [Msg3] [Msg4] [Msg5] [Msg6]

Consumer Position:
  Committed Offset = 3  (Msgs 0,1,2 processed and committed)
  Current Position = 4  (Currently processing Msg4)

If consumer crashes:
  Auto-commit:   May have committed offset 4–5 already → LOST MESSAGES
  Manual-commit: Offset = 3, restarts from Msg3 → AT-LEAST-ONCE delivery
```

### Auto-Commit

```java
@Bean
public ConsumerFactory<String, OrderEvent> consumerFactory() {
    Map<String, Object> config = new HashMap<>();

    config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    config.put(ConsumerConfig.GROUP_ID_CONFIG, "order-processor");

    config.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
    config.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 5000); // every 5s

    // earliest: Start from beginning if no committed offset
    // latest:   Start from newest messages only
    config.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");

    return new DefaultKafkaConsumerFactory<>(config);
}

@KafkaListener(topics = "order-events", groupId = "order-processor")
public void consume(OrderEvent event) {
    processOrder(event);
    // Offset committed automatically every 5 seconds — no explicit commit needed
}

// Auto-commit pitfall:
// T=0:    Messages 1–100 polled, offset auto-committed to 100
// T=100ms: Still processing message 50
// T=200ms: Consumer crashes
// T=300ms: Consumer restarts from offset 101
// Result:  Messages 51–100 NEVER processed! DATA LOSS!
```

### Manual Commit Strategies

#### 1. Synchronous Commit (After Each Message)

```java
@Configuration
public class ManualCommitConfig {

    @Bean
    public ConcurrentKafkaListenerContainerFactory<String, OrderEvent>
            kafkaListenerContainerFactory() {

        ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
            new ConcurrentKafkaListenerContainerFactory<>();
        factory.setConsumerFactory(consumerFactory());
        factory.getContainerProperties()
               .setAckMode(ContainerProperties.AckMode.MANUAL_IMMEDIATE);
        return factory;
    }
}

@KafkaListener(topics = "order-events", groupId = "order-processor",
               containerFactory = "kafkaListenerContainerFactory")
public void consume(OrderEvent event, Acknowledgment ack) {
    try {
        processOrder(event);
        ack.acknowledge(); // Commit offset after successful processing
        // Pro: No message loss
        // Con: One commit per message — high overhead for large volumes
    } catch (Exception e) {
        logger.error("Failed to process order: {}", event.getOrderId(), e);
        // Don't acknowledge → message will be reprocessed
        // Ensure idempotent processing to handle retries
    }
}
```

#### 2. Batch Manual Commit (After Each Batch)

```java
@Bean
public ConcurrentKafkaListenerContainerFactory<String, OrderEvent>
        batchKafkaListenerContainerFactory() {

    ConcurrentKafkaListenerContainerFactory<String, OrderEvent> factory =
        new ConcurrentKafkaListenerContainerFactory<>();
    factory.setConsumerFactory(consumerFactory());
    factory.setBatchListener(true);
    factory.getContainerProperties().setAckMode(ContainerProperties.AckMode.BATCH);
    return factory;
}

@KafkaListener(topics = "order-events", groupId = "order-processor",
               containerFactory = "batchKafkaListenerContainerFactory")
public void consumeBatch(List<OrderEvent> events, Acknowledgment ack) {
    try {
        for (OrderEvent event : events) {
            processOrder(event);
        }
        ack.acknowledge(); // Commit after entire batch
        // Pro: Fewer commits (better performance)
        // Con: If crash mid-batch, entire batch is reprocessed
    } catch (Exception e) {
        logger.error("Batch processing failed", e);
        // Don't acknowledge → entire batch reprocessed
    }
}
```

#### 3. Selective Commit (At Specific Offsets)

```java
@KafkaListener(topics = "order-events", groupId = "order-processor")
public void consume(
        ConsumerRecord<String, OrderEvent> record,
        Consumer<String, OrderEvent> consumer) {

    try {
        processOrder(record.value());

        Map<TopicPartition, OffsetAndMetadata> offsets = new HashMap<>();
        offsets.put(
            new TopicPartition(record.topic(), record.partition()),
            new OffsetAndMetadata(record.offset() + 1) // Always commit current + 1
        );

        consumer.commitSync(offsets); // Blocks until committed
    } catch (Exception e) {
        logger.error("Failed to process message", e);
        // Don't commit — message will be reprocessed
    }
}
```

#### 4. Async Commit (Higher Throughput)

```java
@KafkaListener(topics = "order-events", groupId = "order-processor")
public void consume(
        ConsumerRecord<String, OrderEvent> record,
        Consumer<String, OrderEvent> consumer) {

    processOrder(record.value());

    Map<TopicPartition, OffsetAndMetadata> offsets = Collections.singletonMap(
        new TopicPartition(record.topic(), record.partition()),
        new OffsetAndMetadata(record.offset() + 1)
    );

    consumer.commitAsync(offsets, (committedOffsets, exception) -> {
        if (exception != null) {
            logger.error("Async commit failed", exception);
        }
    });
    // Pro: Non-blocking, higher throughput
    // Con: Commit may fail without automatic retry
    // Best practice: async normally, sync on shutdown
}
```

#### 5. Hybrid Commit Strategy (Production Best Practice)

```java
@Service
public class HybridCommitConsumer {

    private volatile boolean running = true;

    @KafkaListener(topics = "order-events", groupId = "order-processor")
    public void consume(
            ConsumerRecord<String, OrderEvent> record,
            Consumer<String, OrderEvent> consumer) {

        processOrder(record.value());

        Map<TopicPartition, OffsetAndMetadata> offsets = Collections.singletonMap(
            new TopicPartition(record.topic(), record.partition()),
            new OffsetAndMetadata(record.offset() + 1)
        );

        if (running) {
            // Normal operation: async for performance
            consumer.commitAsync(offsets, (co, ex) -> {
                if (ex != null) {
                    consumer.commitSync(offsets); // Retry with sync on failure
                }
            });
        } else {
            // Shutdown: sync to ensure no data loss
            consumer.commitSync(offsets);
        }
    }

    @PreDestroy
    public void shutdown() {
        running = false;
    }
}
```

#### 6. Manual Partition Assignment with Offset Control

```java
@Service
public class ManualPartitionConsumer {

    public void reprocessFromOffset(String topic, int partition, long startOffset) {
        Consumer<String, OrderEvent> consumer = consumerFactory.createConsumer();
        TopicPartition topicPartition = new TopicPartition(topic, partition);
        consumer.assign(Collections.singletonList(topicPartition));

        consumer.seek(topicPartition, startOffset); // Seek to specific offset
        // consumer.seekToBeginning(...) — replay from start
        // consumer.seekToEnd(...)       — skip to latest

        try {
            while (true) {
                ConsumerRecords<String, OrderEvent> records =
                    consumer.poll(Duration.ofMillis(1000));
                for (ConsumerRecord<String, OrderEvent> record : records) {
                    processOrder(record.value());
                }
                consumer.commitSync();
            }
        } finally {
            consumer.close();
        }
    }
}
```

### AckMode Reference

```java
// RECORD          — Commit after each record (slowest, safest)
factory.getContainerProperties().setAckMode(AckMode.RECORD);

// BATCH           — Commit after each batch poll (good balance)
factory.getContainerProperties().setAckMode(AckMode.BATCH);

// TIME            — Commit every N milliseconds (similar to auto-commit)
factory.getContainerProperties().setAckMode(AckMode.TIME);
factory.getContainerProperties().setAckTime(5000);

// COUNT           — Commit every N records (good for high throughput)
factory.getContainerProperties().setAckMode(AckMode.COUNT);
factory.getContainerProperties().setAckCount(100);

// COUNT_TIME      — Commit on count OR time, whichever comes first
factory.getContainerProperties().setAckMode(AckMode.COUNT_TIME);

// MANUAL          — Explicit ack.acknowledge() required, batches until next poll
factory.getContainerProperties().setAckMode(AckMode.MANUAL);

// MANUAL_IMMEDIATE — Explicit ack.acknowledge() required, commits immediately
factory.getContainerProperties().setAckMode(AckMode.MANUAL_IMMEDIATE);
```

### Commit Strategy Decision

| Scenario | Strategy | Why |
|---|---|---|
| Critical data, no loss | MANUAL_IMMEDIATE | Full control |
| High-throughput logs | AUTO or BATCH | Performance priority |
| Batch processing | BATCH + manual | Atomic batch commit |
| Financial transactions | MANUAL_IMMEDIATE | Zero tolerance for loss |
| Analytics pipeline | AUTO | Duplicates acceptable |

---

## Q6. Event Sourcing for Inventory Tracking Using Kafka

### Pattern Overview

```
Traditional: Store only current state
  inventory table: {sku: "ABC", quantity: 85}

Event Sourcing: Store every event that led to current state
  Event 1: {type: RECEIVED, sku: "ABC", qty: 100, timestamp: T1}
  Event 2: {type: SOLD,     sku: "ABC", qty: 10,  timestamp: T2}
  Event 3: {type: DAMAGED,  sku: "ABC", qty: 5,   timestamp: T3}

  Current state = replay all events: 100 - 10 - 5 = 85
```

### Event Definitions

```java
@JsonTypeInfo(use = JsonTypeInfo.Id.NAME)
@JsonSubTypes({
    @JsonSubTypes.Type(value = InventoryReceivedEvent.class, name = "RECEIVED"),
    @JsonSubTypes.Type(value = InventoryReservedEvent.class, name = "RESERVED"),
    @JsonSubTypes.Type(value = InventorySoldEvent.class,    name = "SOLD"),
    @JsonSubTypes.Type(value = InventoryAdjustedEvent.class, name = "ADJUSTED"),
    @JsonSubTypes.Type(value = InventoryDamagedEvent.class,  name = "DAMAGED"),
    @JsonSubTypes.Type(value = InventoryReturnedEvent.class, name = "RETURNED")
})
public abstract class InventoryEvent {
    private String eventId;
    private String sku;
    private String storeId;
    private Instant timestamp;
    private String correlationId;
    private Long version;

    public abstract int getQuantityChange();
}

public class InventoryReceivedEvent extends InventoryEvent {
    private int quantity;
    private String supplierId;
    private String purchaseOrderId;

    @Override public int getQuantityChange() { return quantity; }
}

public class InventorySoldEvent extends InventoryEvent {
    private int quantity;
    private String orderId;

    @Override public int getQuantityChange() { return -quantity; }
}

public class InventoryAdjustedEvent extends InventoryEvent {
    private int adjustmentAmount; // Can be positive or negative
    private String reason;
    private String adjustedBy;

    @Override public int getQuantityChange() { return adjustmentAmount; }
}
```

### Aggregate Root

```java
public class InventoryAggregate {
    private String sku;
    private String storeId;
    private int currentQuantity;
    private int reservedQuantity;
    private Long version;
    private List<InventoryEvent> uncommittedEvents = new ArrayList<>();

    // Reconstitute from event history
    public static InventoryAggregate reconstitute(List<InventoryEvent> events) {
        InventoryAggregate aggregate = new InventoryAggregate();
        for (InventoryEvent event : events) {
            aggregate.apply(event);
        }
        return aggregate;
    }

    // Command: Receive inventory
    public void receiveInventory(String supplierId, int quantity, String poId) {
        if (quantity <= 0) throw new InvalidQuantityException("Quantity must be positive");

        InventoryReceivedEvent event = new InventoryReceivedEvent();
        event.setEventId(UUID.randomUUID().toString());
        event.setSku(this.sku);
        event.setQuantity(quantity);
        event.setSupplierId(supplierId);
        event.setPurchaseOrderId(poId);
        event.setTimestamp(Instant.now());
        event.setVersion(this.version + 1);

        apply(event);
        uncommittedEvents.add(event);
    }

    // Command: Reserve inventory for an order
    public void reserveInventory(String orderId, int quantity) {
        int available = currentQuantity - reservedQuantity;
        if (available < quantity) {
            throw new InsufficientInventoryException(
                "Available: " + available + ", Requested: " + quantity);
        }

        InventoryReservedEvent event = new InventoryReservedEvent();
        event.setEventId(UUID.randomUUID().toString());
        event.setSku(this.sku);
        event.setQuantity(quantity);
        event.setOrderId(orderId);
        event.setTimestamp(Instant.now());
        event.setVersion(this.version + 1);

        apply(event);
        uncommittedEvents.add(event);
    }

    // Apply events to state — used for both new events and replay
    private void apply(InventoryEvent event) {
        if (event instanceof InventoryReceivedEvent) {
            this.currentQuantity += ((InventoryReceivedEvent) event).getQuantity();
        } else if (event instanceof InventoryReservedEvent) {
            this.reservedQuantity += ((InventoryReservedEvent) event).getQuantity();
        } else if (event instanceof InventorySoldEvent) {
            InventorySoldEvent e = (InventorySoldEvent) event;
            this.currentQuantity -= e.getQuantity();
            this.reservedQuantity -= e.getQuantity();
        } else if (event instanceof InventoryAdjustedEvent) {
            this.currentQuantity += ((InventoryAdjustedEvent) event).getAdjustmentAmount();
        } else if (event instanceof InventoryDamagedEvent) {
            this.currentQuantity -= ((InventoryDamagedEvent) event).getQuantity();
        } else if (event instanceof InventoryReturnedEvent) {
            this.currentQuantity += ((InventoryReturnedEvent) event).getQuantity();
        }
        this.version = event.getVersion();
    }

    public List<InventoryEvent> getUncommittedEvents() {
        return Collections.unmodifiableList(uncommittedEvents);
    }

    public void clearUncommittedEvents() { uncommittedEvents.clear(); }
}
```

### Command Service

```java
@Service
public class InventoryCommandService {

    @Autowired private EventStore eventStore;
    @Autowired private SnapshotStore snapshotStore;
    @Autowired private KafkaTemplate<String, InventoryEvent> kafkaTemplate;

    @Transactional
    public void receiveInventory(ReceiveInventoryCommand command) {
        InventoryAggregate aggregate = loadAggregate(
            command.getSku(), command.getStoreId());

        aggregate.receiveInventory(
            command.getSupplierId(), command.getQuantity(), command.getPurchaseOrderId());

        saveAndPublishEvents(aggregate, command.getSku());
    }

    @Transactional
    public void reserveInventory(ReserveInventoryCommand command) {
        InventoryAggregate aggregate = loadAggregate(
            command.getSku(), command.getStoreId());
        aggregate.reserveInventory(command.getOrderId(), command.getQuantity());
        saveAndPublishEvents(aggregate, command.getSku());
    }

    private InventoryAggregate loadAggregate(String sku, String storeId) {
        String aggregateId = sku + "-" + storeId;

        Optional<InventorySnapshot> snapshot = snapshotStore.getLatestSnapshot(aggregateId);
        InventoryAggregate aggregate;
        List<InventoryEvent> events;

        if (snapshot.isPresent()) {
            // Start from snapshot, replay only newer events
            aggregate = snapshot.get().toAggregate();
            events = eventStore.getEventsAfterVersion(aggregateId, snapshot.get().getVersion());
        } else {
            // Replay from the very beginning
            aggregate = new InventoryAggregate();
            aggregate.setSku(sku);
            aggregate.setStoreId(storeId);
            events = eventStore.getAllEvents(aggregateId);
        }

        for (InventoryEvent event : events) aggregate.apply(event);
        return aggregate;
    }

    private void saveAndPublishEvents(InventoryAggregate aggregate, String sku) {
        List<InventoryEvent> newEvents = aggregate.getUncommittedEvents();

        eventStore.saveEvents(sku, newEvents, aggregate.getVersion());

        for (InventoryEvent event : newEvents) {
            kafkaTemplate.send("inventory-events", sku, event);
        }

        aggregate.clearUncommittedEvents();

        // Create snapshot every 100 events to speed up future replays
        if (eventStore.getEventCount(sku) % 100 == 0) {
            snapshotStore.saveSnapshot(InventorySnapshot.from(aggregate));
        }
    }
}
```

### Event Store Implementation

```java
@Entity
@Table(name = "inventory_events")
public class InventoryEventEntity {
    @Id
    private String eventId;
    private String aggregateId; // sku + storeId
    private String eventType;

    @Column(columnDefinition = "TEXT")
    private String eventData; // JSON-serialised event

    private Long version;
    private Instant timestamp;
}

@Service
public class EventStore {

    @Transactional
    public void saveEvents(String aggregateId,
                          List<InventoryEvent> events,
                          Long expectedVersion) {

        // Optimistic concurrency check
        Long currentVersion = repository
            .findTopByAggregateIdOrderByVersionDesc(aggregateId)
            .map(InventoryEventEntity::getVersion)
            .orElse(0L);

        if (!currentVersion.equals(expectedVersion - events.size())) {
            throw new ConcurrentModificationException(
                "Version mismatch. Expected: " + expectedVersion +
                ", Current: " + currentVersion);
        }

        List<InventoryEventEntity> entities = events.stream()
            .map(event -> {
                InventoryEventEntity entity = new InventoryEventEntity();
                entity.setEventId(event.getEventId());
                entity.setAggregateId(aggregateId);
                entity.setEventType(event.getClass().getSimpleName());
                entity.setEventData(serialize(event));
                entity.setVersion(event.getVersion());
                entity.setTimestamp(event.getTimestamp());
                return entity;
            })
            .collect(Collectors.toList());

        repository.saveAll(entities);
    }
}
```

### Query Side (CQRS Read Model)

```java
@Service
public class InventoryQueryHandler {

    // Consume events to update the denormalised read model
    @KafkaListener(topics = "inventory-events", groupId = "inventory-projector")
    @Transactional
    public void projectEvent(InventoryEvent event) {
        String aggregateId = event.getSku() + "-" + event.getStoreId();

        InventoryReadModel readModel = readModelRepository
            .findByAggregateId(aggregateId)
            .orElseGet(() -> new InventoryReadModel(aggregateId, event.getSku()));

        if (event instanceof InventoryReceivedEvent) {
            InventoryReceivedEvent e = (InventoryReceivedEvent) event;
            readModel.setCurrentQuantity(readModel.getCurrentQuantity() + e.getQuantity());
            readModel.setLastReceivedAt(e.getTimestamp());
        } else if (event instanceof InventorySoldEvent) {
            InventorySoldEvent e = (InventorySoldEvent) event;
            readModel.setCurrentQuantity(readModel.getCurrentQuantity() - e.getQuantity());
            readModel.setTotalSold(readModel.getTotalSold() + e.getQuantity());
        } else if (event instanceof InventoryAdjustedEvent) {
            InventoryAdjustedEvent e = (InventoryAdjustedEvent) event;
            readModel.setCurrentQuantity(readModel.getCurrentQuantity() + e.getAdjustmentAmount());
        }

        readModel.setVersion(event.getVersion());
        readModel.setLastUpdated(Instant.now());
        readModelRepository.save(readModel);
    }

    // Fast query from denormalised read model
    public InventoryDTO getInventory(String sku, String storeId) {
        return readModelRepository
            .findByAggregateId(sku + "-" + storeId)
            .map(this::toDTO)
            .orElseThrow(() -> new InventoryNotFoundException(sku));
    }

    // Time-travel query: reconstruct state at any point in history
    public InventoryDTO getInventoryAtTime(String sku, String storeId, Instant pointInTime) {
        List<InventoryEvent> events = eventStore.getEventsUpTo(
            sku + "-" + storeId, pointInTime);
        InventoryAggregate aggregate = InventoryAggregate.reconstitute(events);
        return new InventoryDTO(aggregate.getSku(), aggregate.getCurrentQuantity(),
                                aggregate.getReservedQuantity());
    }
}
```

### Benefits of Event Sourcing with Kafka

- **Complete audit trail** — every inventory change is recorded with full context
- **Time-travel queries** — reconstruct the exact state at any point in history
- **Event replay** — rebuild read models from scratch if projections need to change
- **Decoupled consumers** — multiple services consume the same event stream independently
- **Debuggability** — understand exactly what happened and why for any given moment

---

## Q7. Kafka Streams vs Traditional Consumer Processing

### Traditional Consumer Processing

```java
// Simple: Poll → Process → Commit
@Service
public class TraditionalOrderProcessor {

    @KafkaListener(topics = "order-events", groupId = "order-processor")
    public void process(OrderEvent event) {
        Order order = createOrder(event);
        orderRepository.save(order);
        // No stream operations (filter, map, aggregate, join)
    }
}
```

### Kafka Streams Processing

```java
@Configuration
public class KafkaStreamsConfig {

    @Bean(name = KafkaStreamsDefaultConfiguration.DEFAULT_STREAMS_CONFIG_BEAN_NAME)
    public KafkaStreamsConfiguration kafkaStreamsConfig() {
        Map<String, Object> config = new HashMap<>();

        config.put(StreamsConfig.APPLICATION_ID_CONFIG, "merchandising-streams");
        config.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG,
                  Serdes.String().getClass());
        config.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, JsonSerde.class);
        config.put(StreamsConfig.STATE_DIR_CONFIG, "/tmp/kafka-streams");

        return new KafkaStreamsConfiguration(config);
    }
}

@Component
public class InventoryStreamsProcessor {

    @Autowired
    private StreamsBuilder streamsBuilder;

    @PostConstruct
    public void buildTopology() {
        KStream<String, OrderEvent> orders = streamsBuilder.stream("order-events");

        // 1. Filter and transform
        KStream<String, InventoryReservation> reservations = orders
            .filter((key, event) -> event.getStatus() == OrderStatus.CONFIRMED)
            .map((key, event) -> KeyValue.pair(
                event.getSku(),
                new InventoryReservation(event.getSku(), event.getQuantity())
            ));
        reservations.to("inventory-reservations");

        // 2. Aggregation — count orders per SKU per hour
        TimeWindows hourlyWindow = TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1));

        orders
            .groupBy((key, event) -> event.getSku())
            .windowedBy(hourlyWindow)
            .count(Materialized.as("order-counts-store"))
            .toStream()
            .map((windowedKey, count) -> KeyValue.pair(
                windowedKey.key(),
                new SkuOrderCount(windowedKey.key(), count,
                    windowedKey.window().startTime(),
                    windowedKey.window().endTime())
            ))
            .to("sku-order-counts");

        // 3. Join — enrich orders with product details
        KTable<String, Product> products = streamsBuilder.table("product-catalog");

        orders
            .join(products,
                (order, product) -> new EnrichedOrder(order, product),
                Joined.with(Serdes.String(), JsonSerde.class, JsonSerde.class))
            .to("enriched-orders");

        // 4. Real-time low stock detection
        KTable<String, InventoryLevel> inventoryLevels =
            streamsBuilder.table("inventory-levels");

        reservations
            .join(inventoryLevels, (reservation, inventory) -> {
                int remaining = inventory.getQuantity() - reservation.getQuantity();
                if (remaining < inventory.getReorderPoint()) {
                    return new LowStockAlert(reservation.getSku(), remaining,
                                             inventory.getReorderPoint());
                }
                return null;
            })
            .filter((key, alert) -> alert != null)
            .to("low-stock-alerts");

        // 5. Session windows — detect abandoned carts
        SessionWindows sessionWindow =
            SessionWindows.ofInactivityGapWithNoGrace(Duration.ofMinutes(30));

        streamsBuilder.stream("user-activity-events")
            .groupBy((key, event) -> event.getUserId())
            .windowedBy(sessionWindow)
            .count();

        // 6. Stateful processing with a local state store
        StoreBuilder<KeyValueStore<String, Integer>> storeBuilder =
            Stores.keyValueStoreBuilder(
                Stores.persistentKeyValueStore("inventory-store"),
                Serdes.String(), Serdes.Integer()
            );
        streamsBuilder.addStateStore(storeBuilder);
        orders.process(() -> new InventoryDeductionProcessor(), "inventory-store");
    }
}

// Custom stateful processor
public class InventoryDeductionProcessor implements Processor<String, OrderEvent> {

    private KeyValueStore<String, Integer> inventoryStore;

    @Override
    public void init(ProcessorContext context) {
        inventoryStore = context.getStateStore("inventory-store");
    }

    @Override
    public void process(String key, OrderEvent event) {
        String sku = event.getSku();
        Integer current = inventoryStore.get(sku);
        if (current == null) current = 0;

        int newQuantity = current - event.getQuantity();
        inventoryStore.put(sku, newQuantity);

        if (newQuantity < 10) {
            context.forward(sku, new LowStockEvent(sku, newQuantity));
        }
    }

    @Override public void close() {}
}
```

### Interactive Queries (Query State Stores via REST)

```java
@RestController
@RequestMapping("/inventory/streams")
public class InventoryStreamsQueryController {

    @Autowired
    private KafkaStreams kafkaStreams;

    @GetMapping("/{sku}/count")
    public ResponseEntity<Long> getOrderCount(@PathVariable String sku) {
        ReadOnlyKeyValueStore<String, Long> store = kafkaStreams
            .store(StoreQueryParameters.fromNameAndType(
                "order-counts-store", QueryableStoreTypes.keyValueStore()));

        Long count = store.get(sku);
        return ResponseEntity.ok(count != null ? count : 0L);
    }

    @GetMapping("/{sku}/hourly-orders")
    public ResponseEntity<List<Map<String, Object>>> getHourlyOrders(
            @PathVariable String sku) {

        ReadOnlyWindowStore<String, Long> windowStore = kafkaStreams
            .store(StoreQueryParameters.fromNameAndType(
                "order-counts-store", QueryableStoreTypes.windowStore()));

        Instant now = Instant.now();
        Instant oneDayAgo = now.minus(Duration.ofDays(1));
        List<Map<String, Object>> results = new ArrayList<>();

        WindowStoreIterator<Long> iterator = windowStore.fetch(sku, oneDayAgo, now);
        while (iterator.hasNext()) {
            KeyValue<Long, Long> kv = iterator.next();
            results.add(Map.of(
                "windowStart", Instant.ofEpochMilli(kv.key).toString(),
                "count", kv.value
            ));
        }
        iterator.close();
        return ResponseEntity.ok(results);
    }
}
```

### When to Use Each

| Aspect | Traditional Consumer | Kafka Streams |
|---|---|---|
| Use case | Simple processing | Complex stream processing |
| Stateful ops | Requires external DB | Built-in state stores |
| Aggregations | Manual, complex | Built-in windowed ops |
| Joins | Not natively supported | Supported natively |
| Deployment | Any JVM app | Embedded library |
| Scaling | Manual | Automatic (partition-based) |
| Fault tolerance | Manual offset management | Automatic |
| Learning curve | Low | High |

**Use Traditional Consumer when:** simple message processing (one-in, one-out), persisting to a database, calling external APIs, no aggregations or joins needed, simple ETL pipelines.

**Use Kafka Streams when:** real-time aggregations (count, sum, average), stream-stream or stream-table joins, stateful processing with time windows, complex event processing (CEP), enrichment pipelines, real-time dashboards.

---

## Q8. Schema Evolution in Event-Driven Systems — Schema Registry

### The Problem

```java
// Version 1: Original event schema
public class OrderCreatedEvent {
    private String orderId;
    private String customerId;
    private double amount;
}

// Version 2: Updated schema (breaking changes)
public class OrderCreatedEvent {
    private String orderId;
    private String customerId;
    private BigDecimal amount;  // Changed: double → BigDecimal ← BREAKING
    private String storeId;     // Added: new required field ← BREAKING
    // paymentMethod removed   ← BREAKING
}

// Problem: old producers send V1, new consumers expect V2 → deserialization fails!
```

### Solution: Schema Registry + Avro

```json
// order-created-v1.avsc
{
  "type": "record",
  "name": "OrderCreated",
  "namespace": "com.retail.events",
  "fields": [
    {"name": "orderId",    "type": "string"},
    {"name": "customerId", "type": "string"},
    {"name": "amount",     "type": "double"}
  ]
}

// order-created-v2.avsc (backward-compatible evolution)
{
  "type": "record",
  "name": "OrderCreated",
  "namespace": "com.retail.events",
  "fields": [
    {"name": "orderId",    "type": "string"},
    {"name": "customerId", "type": "string"},
    {"name": "amount",     "type": "double"},
    {"name": "storeId",    "type": ["null", "string"], "default": null},
    {"name": "currency",   "type": "string",           "default": "USD"}
  ]
}
```

### Schema Registry Configuration

```java
@Configuration
public class SchemaRegistryKafkaConfig {

    // Producer with Avro serializer
    @Bean
    public ProducerFactory<String, OrderCreated> producerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);

        // Avro serializer — registers/validates schema automatically
        config.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG,
                  KafkaAvroSerializer.class);
        config.put("schema.registry.url", "http://schema-registry:8081");
        config.put("auto.register.schemas", true);

        return new DefaultKafkaProducerFactory<>(config);
    }

    // Consumer with Avro deserializer
    @Bean
    public ConsumerFactory<String, OrderCreated> consumerFactory() {
        Map<String, Object> config = new HashMap<>();
        config.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        config.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        config.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG,
                  KafkaAvroDeserializer.class);
        config.put("schema.registry.url", "http://schema-registry:8081");
        config.put("specific.avro.reader", true);

        return new DefaultKafkaConsumerFactory<>(config);
    }
}
```

### Schema Evolution Compatibility Rules

```java
// BACKWARD COMPATIBLE — new consumers can read old messages
// ✓ Add optional field with a default value
// ✓ Remove a field (old data for that field is simply lost)
// ✗ Add a required field without a default
// ✗ Change a field type

// FORWARD COMPATIBLE — old consumers can read new messages
// ✓ Remove an optional field that had a default
// ✗ Add any new field
// ✗ Change a field type

// FULL COMPATIBLE — both backward and forward
// ✓ Add optional field with default
// ✓ Remove optional field with default

// NONE — no compatibility check (use with extreme caution)
```

```bash
# Set compatibility mode for a topic's value schema
curl -X PUT \
  "http://schema-registry:8081/config/order-events-value" \
  -H "Content-Type: application/json" \
  -d '{"compatibility": "BACKWARD"}'

# Test compatibility before deploying a new producer version
curl -X POST \
  "http://schema-registry:8081/compatibility/subjects/order-events-value/versions/latest" \
  -H "Content-Type: application/json" \
  -d @new-schema.json
# Returns: {"is_compatible": true}
```

### Producer with Schema Evolution

```java
@Service
public class OrderEventProducer {

    @Autowired
    private KafkaTemplate<String, OrderCreated> kafkaTemplate;

    public void publishOrderV2(Order order) {
        OrderCreated event = OrderCreated.newBuilder()
            .setOrderId(order.getId())
            .setCustomerId(order.getCustomerId())
            .setAmount(order.getAmount().doubleValue())
            .setStoreId(order.getStoreId()) // V2 field
            .setCurrency(order.getCurrency()) // V2 field
            .build();

        // Schema Registry flow:
        // 1. Checks if schema exists
        // 2. Validates compatibility if new schema
        // 3. Registers schema, gets schema ID
        // 4. Prefixes message with schema ID
        //
        // Wire format: [0x00][4-byte schema_id][avro bytes]

        kafkaTemplate.send("order-events", order.getId(), event);
    }
}
```

### Consumer Handling Multiple Schema Versions

```java
// V1 consumer — still works with V2 messages (backward compatible)
@KafkaListener(topics = "order-events", groupId = "order-processor-v1")
public void consumeOrderV1(OrderCreated event) {
    // V2 message received — storeId/currency set to defaults, core fields work fine
    String orderId = event.getOrderId().toString();
    String customerId = event.getCustomerId().toString();
    double amount = event.getAmount();
    processOrder(orderId, customerId, amount);
}

// V2 consumer — uses new fields
@KafkaListener(topics = "order-events", groupId = "order-processor-v2")
public void consumeOrderV2(OrderCreated event) {
    String orderId = event.getOrderId().toString();
    String storeId = event.getStoreId() != null ? event.getStoreId().toString() : "DEFAULT";
    String currency = event.getCurrency().toString();
    processOrderV2(orderId, storeId, currency);
}
```

### JSON Alternative (Without Avro)

```java
@KafkaListener(topics = "order-events", groupId = "order-processor")
public void consume(String rawMessage) {
    try {
        JsonNode jsonNode = objectMapper.readTree(rawMessage);
        int version = jsonNode.path("version").asInt(1);

        if (version == 1) {
            processV1(objectMapper.treeToValue(jsonNode, OrderEventV1.class));
        } else if (version == 2) {
            processV2(objectMapper.treeToValue(jsonNode, OrderEventV2.class));
        }
    } catch (JsonProcessingException e) {
        logger.error("Failed to deserialize message", e);
        sendToDLQ(rawMessage, e);
    }
}

public class OrderEventV1 {
    private String orderId;
    private String customerId;
    private double amount;
}

public class OrderEventV2 extends OrderEventV1 {
    private String storeId;
    private String currency = "USD"; // Default for old messages
}
```

### Schema Registry Architecture

```
Producer App          Schema Registry           Consumer App
     │                      │                       │
     │ 1. Check schema      │                       │
     │─────────────────────>│                       │
     │                      │                       │
     │ 2. Get/Register ID   │                       │
     │<─────────────────────│                       │
     │                      │                       │
     │ 3. Send [ID + bytes] ────────────────────────>│
     │                      │                       │
     │                      │ 4. Get schema by ID   │
     │                      │<──────────────────────│
     │                      │                       │
     │                      │ 5. Return schema ─────>│
     │                      │                       │
     │                      │ 6. Deserialize bytes  │
```

### Best Practices

- **Use BACKWARD compatibility** by default — it is the safest option
- **Never change field types** — add new fields instead
- **Always provide defaults** for new fields
- **Version your events** explicitly in the payload
- **Test compatibility** before deploying new producers
- **Use Schema Registry** for Avro or Protobuf schemas
- **Keep schemas simple** — avoid deeply nested structures
- **Document breaking changes** and coordinate rollouts across teams

