# Stream Processing

**Phase 6 — Distributed Systems + Architecture | Topic 12**

---

## What is Stream Processing?

Stream processing means continuously processing data as it arrives, record by record or in small micro-batches, rather than accumulating data and processing it all at once.

```
Batch processing:
  Collect data for 24 hours → process everything → results
  Latency: hours
  "Yesterday's revenue was $4.2M"

Stream processing:
  Event arrives → process immediately → results updated
  Latency: milliseconds to seconds
  "Current revenue today: $1.3M (updated 200ms ago)"

The fundamental difference:
  Batch: bounded dataset (has a beginning and end)
  Stream: unbounded dataset (keeps arriving forever)
```

---

## Why Stream Processing Matters

Real-time use cases that batch can't serve:

```
Fraud detection:
  Transaction happens → 200ms to decide fraud or not
  If fraud: block transaction
  If batch: detect fraud tomorrow → money already gone

Real-time dashboards:
  Operations team watching order volume during Black Friday
  Need to see spikes immediately
  Batch: "here's yesterday's data" → useless for live ops

Event-driven workflows:
  Order placed → inventory reserved → payment charged →
  warehouse notified → driver assigned
  Each step triggers the next immediately
  Hours of batch latency is unacceptable

Anomaly detection:
  Server CPU spikes → alert ops team in 30 seconds
  Batch at end of day → ops team finds out 8 hours later
```

---

## Stream Processing Concepts

### Events and Streams

```
Event: an immutable record of something that happened
  {orderId: "123", userId: "456", total: 99.99,
   timestamp: "2026-02-14T10:00:00Z"}

Stream: an unbounded, ordered sequence of events
  Order events: [e1, e2, e3, e4, e5, ...]
                continuously growing, never "done"

Stream processing: applying computation to this infinite sequence
```

### Time in Stream Processing

Time is the hardest concept in streaming. Two types matter:

```
Event time:
  When the event actually occurred (in the real world)
  Embedded in the event payload
  {timestamp: "2026-02-14T10:00:00Z"}

  Reliable indicator of when something happened
  BUT: events arrive late (network delay, mobile offline)

Processing time:
  When the event arrives at the stream processor
  Current system clock when Kafka consumer reads it

  Unreliable for ordering:
  Mobile device offline → queues events → comes online →
  Events from 2 hours ago arrive NOW
  Processing time = now, Event time = 2 hours ago

Example of the problem:
  10:00 AM: User places order (event time)
  10:02 AM: Server receives event (processing time)

  → 2-minute delay between event time and processing time
  → If computing "orders per minute", must use event time
    (assign event to 10:00 window, not 10:02 window)
```

### Watermarks — Handling Late Data

```
Watermark: declaration of "I believe all events up to time T
            have now arrived"

            Tells stream processor: safe to close window at T
            Accept late arrivals up to watermark

Example:
  Watermark = current_time - 5_minutes

  If current processing time = 10:10:00
  Watermark = 10:05:00

  Window 10:00-10:05 can be finalized (watermark has passed it)
  Events with timestamp < 10:05 that arrive now → late data

  Late data handling options:
  1. Drop late events (simplest)
  2. Update result (requires downstream to handle updates)
  3. Side output (send to separate late-data stream)
```

```java
// Flink watermark
WatermarkStrategy.<OrderEvent>forBoundedOutOfOrderness(
    Duration.ofMinutes(5))  // expect events up to 5 min late
.withTimestampAssigner(
    (event, timestamp) -> event.getTimestamp().toEpochMilli())
```

---

## Windows — Aggregating Over Time

Since streams are infinite, you aggregate over time windows:

### Tumbling Window

Fixed-size, non-overlapping windows:

```
|-- Window 1 --|-- Window 2 --|-- Window 3 --|
0s            60s           120s           180s

"Count orders per minute"
  Window 1 (0-60s):   47 orders
  Window 2 (60-120s): 63 orders
  Window 3 (120-180s): 41 orders

Each event belongs to exactly ONE window
Use: per-minute/hour/day aggregations
     Revenue reports, rate calculations
```

### Sliding Window

Fixed-size windows that overlap:

```
|-- Window 1 --|
        |-- Window 2 --|
                |-- Window 3 --|
0s     30s     60s     90s     120s

"Orders in last 60 seconds, updated every 30 seconds"
  Window 1 (0-60s):   47 orders
  Window 2 (30-90s):  52 orders
  Window 3 (60-120s): 58 orders

Each event may belong to MULTIPLE windows
Use: moving averages, rolling metrics
     "P99 latency over last 5 minutes"
```

### Session Window

Event-based windows that close on inactivity:

```
User activity:
  click, click, click, [30s gap], click, click, [30s gap], click

|-- Session 1 --|  [gap]  |-- Session 2 --|  [gap]  |-- Session 3 --|

Windows sized by user behavior, not clock
Session closes after 30 seconds of inactivity

Use: user session analytics
     "Average session length"
     "Events per session"
```

---

## Apache Flink — The Production Standard

Flink is the most powerful stream processing framework. True event-time processing, stateful computations, exactly-once guarantees.

### Flink Architecture

```
JobManager:   Coordinates execution plan, manages checkpoints
TaskManagers: Execute actual computation (stateful operators)
State backends: Where state is stored
  Memory: fastest, limited size, lost on failure
  RocksDB: disk-backed, handles large state, persisted

Checkpoint: consistent snapshot of all operator state
  Written to durable storage (S3, HDFS)
  On failure: restore from checkpoint → reprocess from that point
  Enables exactly-once semantics
```

### Flink — Revenue Per Minute

```java
StreamExecutionEnvironment env =
    StreamExecutionEnvironment.getExecutionEnvironment();

// Enable checkpointing for exactly-once
env.enableCheckpointing(60_000); // checkpoint every 60s
env.getCheckpointConfig()
    .setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// Source: read from Kafka
DataStream<OrderEvent> orders = env.addSource(
    KafkaSource.<OrderEvent>builder()
        .setBootstrapServers("kafka:9092")
        .setTopics("order-events")
        .setGroupId("flink-revenue-processor")
        .setStartingOffsets(OffsetsInitializer.latest())
        .setValueOnlyDeserializer(new OrderEventDeserializer())
        .build(),
    WatermarkStrategy
        .<OrderEvent>forBoundedOutOfOrderness(Duration.ofSeconds(10))
        .withTimestampAssigner(
            (event, ts) -> event.getTimestamp().toEpochMilli())
);

// Compute revenue per minute using tumbling window
DataStream<RevenueResult> revenue = orders
    .filter(order -> order.getStatus().equals("COMPLETED"))
    .keyBy(OrderEvent::getCategory)            // partition by category
    .window(TumblingEventTimeWindows.of(Time.minutes(1)))
    .aggregate(new RevenueAggregator());        // sum revenue

// Sink: write to Kafka (downstream consumers)
revenue.addSink(
    KafkaSink.<RevenueResult>builder()
        .setBootstrapServers("kafka:9092")
        .setRecordSerializer(
            KafkaRecordSerializationSchema.builder()
                .setTopic("revenue-per-minute")
                .setValueSerializationSchema(new RevenueResultSerializer())
                .build())
        .build()
);

env.execute("Revenue Per Minute");

// The aggregator function
public class RevenueAggregator
    implements AggregateFunction<OrderEvent, RevenueAccumulator, RevenueResult> {

    @Override
    public RevenueAccumulator createAccumulator() {
        return new RevenueAccumulator(0, 0.0);
    }

    @Override
    public RevenueAccumulator add(OrderEvent event, RevenueAccumulator acc) {
        return new RevenueAccumulator(
            acc.count() + 1,
            acc.revenue() + event.getTotal()
        );
    }

    @Override
    public RevenueResult getResult(RevenueAccumulator acc) {
        return new RevenueResult(acc.count(), acc.revenue());
    }

    @Override
    public RevenueAccumulator merge(RevenueAccumulator a, RevenueAccumulator b) {
        return new RevenueAccumulator(a.count() + b.count(),
                                      a.revenue() + b.revenue());
    }
}
```

### Flink Stateful Processing — Fraud Detection

```java
// Stateful stream processing: detect 3 failed payments in 5 minutes

public class FraudDetectionFunction
    extends KeyedProcessFunction<String, PaymentEvent, FraudAlert> {

    // State: list of recent failed payment timestamps per user
    private ListState<Long> recentFailures;

    @Override
    public void open(Configuration config) {
        ListStateDescriptor<Long> descriptor = new ListStateDescriptor<>(
            "recent-failures", Long.class);
        recentFailures = getRuntimeContext().getListState(descriptor);
    }

    @Override
    public void processElement(PaymentEvent event, Context ctx,
                                Collector<FraudAlert> out) throws Exception {

        if (event.getStatus().equals("FAILED")) {
            long eventTime = event.getTimestamp().toEpochMilli();
            long fiveMinutesAgo = eventTime - Duration.ofMinutes(5).toMillis();

            // Add this failure
            recentFailures.add(eventTime);

            // Count failures in last 5 minutes
            List<Long> recent = new ArrayList<>();
            for (Long ts : recentFailures.get()) {
                if (ts > fiveMinutesAgo) {
                    recent.add(ts);
                }
            }
            recentFailures.update(recent); // clean up old entries

            if (recent.size() >= 3) {
                out.collect(new FraudAlert(
                    event.getUserId(),
                    "3+ payment failures in 5 minutes",
                    eventTime
                ));
                recentFailures.clear(); // reset after alert
            }

            // Set timer to clean state after 5 minutes
            ctx.timerService().registerEventTimeTimer(
                eventTime + Duration.ofMinutes(5).toMillis());
        }
    }

    @Override
    public void onTimer(long timestamp, OnTimerContext ctx,
                         Collector<FraudAlert> out) throws Exception {
        recentFailures.clear(); // clean up state
    }
}

// Usage
DataStream<FraudAlert> alerts = payments
    .keyBy(PaymentEvent::getUserId)  // partition by user
    .process(new FraudDetectionFunction());
```

---

## Kafka Streams — Embedded Stream Processing

Kafka Streams is a library (not a cluster) that runs inside your Spring Boot service. No separate cluster needed.

```
Flink: separate cluster, handles petabytes, complex topologies
Kafka Streams: embedded library, microservice-scale, simpler ops

When to use Kafka Streams:
  Simple stream transformations in a microservice
  Don't want to operate a Flink cluster
  Already using Kafka
  Per-key aggregations (stateful per entity)
```

```java
@Configuration
public class OrderStreamProcessor {

    @Bean
    public KStream<String, OrderEvent> orderStream(
            StreamsBuilder builder) {

        // Consume from Kafka topic
        KStream<String, OrderEvent> orders = builder.stream(
            "order-events",
            Consumed.with(Serdes.String(), orderEventSerde()));

        // Real-time order status updates
        orders
            .filter((key, order) -> order.getStatus().equals("COMPLETED"))
            .mapValues(order -> new CompletedOrder(
                order.getOrderId(),
                order.getTotal(),
                order.getTimestamp()))
            .to("completed-orders");

        // Running total per user (stateful)
        KTable<String, Double> userRevenue = orders
            .filter((key, order) -> order.getStatus().equals("COMPLETED"))
            .groupBy((key, order) -> order.getUserId())
            .aggregate(
                () -> 0.0,                              // initializer
                (userId, order, total) ->               // aggregator
                    total + order.getTotal(),
                Materialized.<String, Double, KeyValueStore<Bytes, byte[]>>as(
                    "user-revenue-store")
                    .withValueSerde(Serdes.Double())
            );

        // Interactive query: what's user X's total revenue?
        ReadOnlyKeyValueStore<String, Double> store =
            streams.store(StoreQueryParameters.fromNameAndType(
                "user-revenue-store",
                QueryableStoreTypes.keyValueStore()));

        Double totalRevenue = store.get("user-123");

        return orders;
    }
}
```

---

## Exactly-Once Semantics

Processing guarantees in streaming systems:

```
At-most-once:
  Process event, then commit offset
  If processor crashes between processing and commit:
  → Event processed, offset not committed
  → On restart: offset not advanced → event skipped
  Result: event processed 0 or 1 times
  Use: metrics, logs where loss acceptable

At-least-once:
  Commit offset only after processing confirmed
  If processor crashes after processing but before commit:
  → Event processed, offset not committed
  → On restart: event reprocessed
  Result: event processed 1 or more times (duplicates possible)
  Kafka consumer default
  Use: most use cases (idempotent processing handles duplicates)

Exactly-once:
  Event processed exactly once, no matter what
  Requires: idempotent producers + transactions + checkpointing

  Kafka exactly-once:
  Producer: enable.idempotence=true
            transactional.id=unique-producer-id
  Consumer: isolation.level=read_committed
            (only see committed transactions)

  Flink exactly-once:
  Checkpoint captures operator state + Kafka offsets atomically
  On failure: restore state + replay from checkpointed offset
  No event lost, no event duplicated
```

---

## Stream Processing Architecture Patterns

### Enrichment Pattern

```
Raw event (minimal data) → Enrich with reference data → Rich event

Payment event: {orderId: "123", amount: 99.99}

Enrichment:
  Look up order details from DB or KTable
  Look up user profile from Redis

Enriched event: {
  orderId: "123",
  amount: 99.99,
  userId: "456",
  userName: "Hari Kumar",
  userTier: "PREMIUM",
  productCategory: "Electronics"
}

Kafka Streams: join stream with KTable (reference data)
Flink: async I/O for DB lookups without blocking
```

```java
// Kafka Streams enrichment with KTable
KTable<String, User> users = builder.table("user-events");

KStream<String, EnrichedPayment> enriched = payments
    .join(users,
        (payment, user) -> new EnrichedPayment(payment, user),
        Joined.with(Serdes.String(), paymentSerde(), userSerde()));
```

### CQRS with Streaming

```
Stream processing builds read models from write events.

Order events (Kafka)
    → Flink/Kafka Streams
        → Update Elasticsearch index (search read model)
        → Update Redis cache (fast lookup)
        → Update analytics DB (reporting read model)
        → Update user notification service

All happen in real-time as events arrive
Each downstream gets exactly-once delivery via transactions
```

### Event-Driven Microservices

```
Chain of stream processors:
  Order Service → order-events (Kafka)
      → Payment Processor (Flink/Kafka Streams)
            → payment-events (Kafka)
                → Inventory Processor
                      → inventory-events (Kafka)
                          → Notification Processor
                                → notification-events

Each processor:
  Reads from input topic
  Processes
  Writes to output topic

Fully decoupled, each scales independently
Backpressure: slow processor → Kafka lag grows → alert + scale
```

---

## Flink vs Kafka Streams vs Spark Streaming

```
Property          Flink              Kafka Streams      Spark Streaming
──────────────────────────────────────────────────────────────────────────
Architecture      Separate cluster   Library (embedded) Separate cluster
Latency           ms (true stream)   ms (true stream)   100ms-1s (micro-batch)
State management  Excellent          Good               Limited
Exactly-once      Yes (checkpoints)  Yes (transactions) Yes (WAL)
Throughput        Very high          High               Very high
Windowing         Rich event-time    Good               Good
SQL support       Flink SQL          KSQL (separate)    Spark SQL
Operational cost  High (cluster)     Low (library)      High (cluster)
Use when          Complex topologies Simple enrichment  Large-scale analytics
                  Petabyte scale     Microservice-scale Batch + streaming
                  ML inference       Per-entity state   Same team as batch
```

---

## Key Takeaways

```
Stream processing: continuously process events as they arrive
vs Batch: bounded data, hours of latency
vs Stream: unbounded data, milliseconds of latency

Time:
  Event time: when it happened (use this for correctness)
  Processing time: when it arrived (unreliable for ordering)
  Watermark: "all events up to T have arrived" → close window

Windows:
  Tumbling: fixed, non-overlapping (per-minute aggregations)
  Sliding:  overlapping (rolling averages, moving windows)
  Session:  inactivity-based (user session analytics)

Apache Flink:
  True event-time stream processing
  Rich stateful operators (KeyedProcessFunction)
  Exactly-once via checkpoints to S3
  Best for: complex topologies, large scale, ML inference

Kafka Streams:
  Embedded library, runs in your Spring Boot app
  KStream (events) + KTable (state/reference data)
  Interactive queries (read state directly)
  Best for: microservice-scale, simple enrichment/aggregation

Exactly-once:
  At-most-once: fast, may lose events
  At-least-once: safe, may duplicate (need idempotent consumer)
  Exactly-once: Kafka transactions + Flink checkpoints

Patterns:
  Enrichment: join stream with reference data (KTable)
  CQRS: events build multiple read models in real-time
  Event-driven: chain of stream processors via Kafka
```
