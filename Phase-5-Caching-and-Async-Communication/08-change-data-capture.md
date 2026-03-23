# Change Data Capture (CDC)

**Phase 5 — Caching + Async Communication | Topic 8 of 8**

---

## What is Change Data Capture?

Change Data Capture (CDC) is a technique for tracking and propagating changes made to a database — inserts, updates, and deletes — in near-real-time, so that downstream systems can react to those changes without polling the database.

Instead of asking "what's in the database?", CDC asks: **"what just changed?"**

```
Traditional Polling:                   CDC Approach:

Every 5 minutes:                        On every DB write:
  SELECT * FROM orders                    INSERT → event streamed
  WHERE updated_at > last_checked         UPDATE → event streamed
  → High latency (up to 5 min)            DELETE → event streamed
  → DB load from repeated queries         → Sub-second latency
  → Misses rapid changes                  → Zero polling overhead
```

---

## Why CDC Matters

### The Core Problem: Keeping Systems in Sync

Modern architectures often require the same data to exist in multiple places:

- PostgreSQL (primary source of truth)
- Elasticsearch (for full-text search)
- Redis (for caching)
- Data warehouse (for analytics)
- Other microservices (for downstream processing)

The naive approach — dual-writes — is dangerous:

```
❌ Dual-Write Problem:

  Service writes to PostgreSQL → ✅ success
  Service writes to Elasticsearch → ❌ failure

  → Data is now inconsistent between systems
  → No recovery without manual intervention
  → No atomicity guarantee
```

CDC solves this by making the database the single source of truth and streaming all changes downstream reliably.

---

## How CDC Works: Reading the WAL

Most production CDC systems read from the database's **Write-Ahead Log (WAL)** rather than querying tables.

### Write-Ahead Log (WAL)

Every write operation to a database first gets recorded in the WAL before the actual data pages are modified:

```
Client sends: UPDATE orders SET status='SHIPPED' WHERE id=123

Database process:
  1. Write to WAL:    "UPDATE orders, id=123, status: PENDING → SHIPPED"
  2. Flush WAL to disk (durability)
  3. Apply to data pages in memory
  4. Return success to client

On crash recovery: replay WAL → database reconstructed correctly
```

The WAL already exists for durability — CDC simply reads it as a stream of change events. **Zero additional overhead on the write path.**

### CDC Event Flow

```
[PostgreSQL]
     │
     │  WAL (Write-Ahead Log)
     ▼
[Debezium / CDC Connector]
     │  reads WAL, produces structured events
     ▼
[Kafka Topic: db.orders.changes]
     │
     ├──────────────────────────────────────────────┐
     │                                              │
     ▼                                              ▼
[Elasticsearch Consumer]               [Cache Invalidation Consumer]
  Index updated in <1 second            Redis key evicted immediately
     │
     ▼
[Analytics Consumer]
  Data warehouse updated
```

---

## Debezium: The Standard CDC Tool

**Debezium** is the most widely-used open-source CDC platform. It supports PostgreSQL, MySQL, MongoDB, SQL Server, and more.

### How Debezium Works with PostgreSQL

```
PostgreSQL WAL → Debezium Connector → Kafka

Configuration:
  connector.class = io.debezium.connector.postgresql.PostgresConnector
  database.hostname = postgres-host
  database.port = 5432
  database.dbname = orders_db
  table.include.list = public.orders, public.payments

  slot.name = debezium_slot    ← PostgreSQL replication slot
  plugin.name = pgoutput        ← logical decoding plugin
```

### Sample CDC Event

When a row changes in the `orders` table:

```json
{
  "op": "u",
  "before": {
    "id": 123,
    "status": "PENDING",
    "updated_at": "2024-01-15T10:00:00Z"
  },
  "after": {
    "id": 123,
    "status": "SHIPPED",
    "updated_at": "2024-01-15T10:05:00Z"
  },
  "source": {
    "db": "orders_db",
    "table": "orders",
    "lsn": 12345678
  },
  "ts_ms": 1705312345000
}
```

Operation types: `"c"` (create/insert), `"u"` (update), `"d"` (delete), `"r"` (read/snapshot)

---

## The Transactional Outbox Pattern

CDC is most powerfully used with the **Transactional Outbox Pattern** to guarantee exactly-once event publishing from microservices.

### The Problem Without Outbox

```
❌ Typical dual-write failure scenario:

@Transactional
public void createOrder(Order order) {
    orderRepo.save(order);           // Step 1: Save to DB ✅
    kafkaProducer.send("OrderCreated", order);  // Step 2: Publish ❌
    // If Kafka is down here → order saved but event NEVER published
    // → Inventory never decremented
    // → Notification never sent
    // → Inconsistent system state
}
```

### The Outbox Pattern Solution

```
✅ Atomic write + guaranteed eventual publication:

@Transactional
public void createOrder(Order order) {
    // BOTH in the same DB transaction — atomic
    orderRepo.save(order);
    outboxRepo.save(OutboxEvent.builder()
        .eventType("OrderCreated")
        .aggregateId(order.getId())
        .payload(serialize(order))
        .status("PENDING")
        .build());
    // Either BOTH committed or NEITHER — guaranteed by DB ACID
}
```

The outbox table acts as a reliable staging area:

```sql
CREATE TABLE outbox (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type  VARCHAR(100) NOT NULL,
    aggregate_id VARCHAR(100) NOT NULL,
    payload     JSONB NOT NULL,
    status      VARCHAR(20) DEFAULT 'PENDING',
    created_at  TIMESTAMP DEFAULT NOW(),
    published_at TIMESTAMP
);
```

### Outbox Relay: Two Approaches

**Approach 1 — Polling Publisher (Simple)**
```
Background process every 500ms:
  SELECT * FROM outbox WHERE status = 'PENDING' LIMIT 100;
  → Publish each event to Kafka
  → UPDATE outbox SET status='PUBLISHED', published_at=NOW()

Pros:  Simple, no external tooling
Cons:  ~500ms latency, DB polling overhead
Use when: low volume, simplicity preferred
```

**Approach 2 — Debezium CDC (Production)**
```
Debezium reads outbox table changes from PostgreSQL WAL:
  Row inserted → WAL entry → Debezium reads → publishes to Kafka

  Sub-second latency from DB write to Kafka event
  No polling, no DB load, no code changes

Pros:  <1 second latency, zero polling overhead, automatic
Cons:  Requires Debezium infrastructure
Use when: high volume, low latency required
```

### Outbox with Debezium Flow

```
Order Service:
  BEGIN TRANSACTION
    INSERT INTO orders (id, status) VALUES (123, 'PENDING')
    INSERT INTO outbox (event_type, payload) VALUES ('OrderCreated', '...')
  COMMIT

PostgreSQL WAL:
  "Row inserted into outbox table"

Debezium:
  Reads WAL entry → parses outbox row
  Publishes to Kafka topic: order-events

Kafka Consumers:
  Inventory Service: decrements stock
  Notification Service: sends email
  Analytics Service: updates dashboards
  All receive the event reliably
```

---

## CDC Use Cases

### 1. Search Index Synchronization

Keep Elasticsearch in sync with PostgreSQL without dual-writes:

```
PostgreSQL (products table)
  → CDC → Kafka: db.products.changes
  → Elasticsearch Consumer
      on INSERT: index new document
      on UPDATE: update document
      on DELETE: remove document

Result:
  PostgreSQL is source of truth
  Elasticsearch always eventually consistent
  No dual-write code in application
  Sub-second search index updates
```

### 2. Cache Invalidation

Invalidate Redis cache entries when database changes:

```
PostgreSQL (users table)
  → CDC → Kafka: db.users.changes
  → Cache Invalidation Service:
      on UPDATE user: DELETE "user:{id}" from Redis
      on DELETE user: DELETE "user:{id}" from Redis

Result:
  Stale cache entries auto-evicted on DB change
  No cache invalidation code in business logic
```

### 3. Data Warehouse / Analytics Sync

Replicate operational data to a data warehouse in near-real-time:

```
PostgreSQL (orders, payments, users)
  → CDC → Kafka
  → Spark Streaming / Flink
  → Snowflake / BigQuery / Redshift

Result:
  Analytics queries run on warehouse (no prod DB load)
  Data freshness: seconds, not hours
  Historical change log preserved (full audit trail)
```

### 4. Microservices Event Sourcing

Decouple microservices via database-driven events:

```
Inventory Service listens to order-events Kafka topic
  on OrderCreated: decrement stock
  on OrderCancelled: restore stock

Notification Service listens to same topic
  on OrderCreated: send confirmation email
  on OrderShipped: send tracking info

Result:
  Order Service never directly calls Inventory or Notification
  Fully decoupled via CDC + Kafka
  Each service independently deployable
```

---

## CDC vs Polling vs Dual-Write Comparison

```
Approach       Latency     Consistency    Complexity    DB Load
─────────────────────────────────────────────────────────────────
Dual-Write     Instant     ❌ At-risk     Low           None
Polling        5-60s       ✅ Eventually  Low           Repeated queries
Outbox+Poll    0.5-5s      ✅ Guaranteed  Medium        Some
Outbox+CDC     <1s         ✅ Guaranteed  Medium-High   Minimal (WAL)
CDC Direct     <1s         ✅ Guaranteed  High          Minimal (WAL)
```

**Use CDC when:**
- Sub-second propagation required
- High write volume (polling not practical)
- Cannot modify application code to add dual-writes
- Full audit trail of all changes needed

**Use Polling when:**
- Low write volume
- Simplicity is paramount
- A few seconds of lag is acceptable

---

## Ordering and Exactly-Once Guarantees

CDC preserves the exact order of database changes (using WAL sequence numbers / LSNs):

```
WAL sequence:
  LSN 1001: INSERT order id=100
  LSN 1002: UPDATE order id=100, status=SHIPPED
  LSN 1003: DELETE order id=100

Kafka partition key = table primary key
  All events for order id=100 → same Kafka partition
  → Consumers see events in exact DB commit order
```

For exactly-once processing, consumers must be idempotent:

```java
// Idempotent consumer — safe to replay events
@KafkaListener(topics = "db.orders.changes")
public void handleOrderChange(OrderChangeEvent event) {
    if (event.getOp().equals("u")) {
        // Check if we've already processed this WAL sequence number
        if (processedEventRepo.exists(event.getSource().getLsn())) {
            return; // Already processed — skip
        }

        // Process the change
        searchIndex.updateOrder(event.getAfter());

        // Mark as processed
        processedEventRepo.save(event.getSource().getLsn());
    }
}
```

---

## Key Takeaways

```
Change Data Capture: stream all DB changes downstream in near-real-time

Why CDC:
  Dual-writes are dangerous (no atomicity)
  Polling is wasteful and high-latency
  CDC reads WAL — zero overhead, sub-second latency

WAL (Write-Ahead Log):
  Database records all changes before applying them
  CDC reads the WAL as a reliable event stream
  Order of changes is guaranteed

Transactional Outbox Pattern:
  Write DB record + outbox event in same ACID transaction
  Outbox relay (or Debezium) publishes to Kafka
  Guarantees: event published ↔ DB write committed

Debezium:
  Industry-standard CDC tool
  Reads PostgreSQL/MySQL WAL → publishes to Kafka
  Sub-second latency, no polling, no application code changes

Use Cases:
  Search index sync (PostgreSQL → Elasticsearch)
  Cache invalidation (DB change → Redis eviction)
  Analytics replication (operational DB → warehouse)
  Microservices decoupling (outbox → event bus)

Ordering:
  CDC preserves exact commit order via WAL LSN
  Partition by primary key → ordered consumer processing

Your Company context:
  Kafka already in use → Debezium is a natural fit
  Outbox pattern ensures reliable event publishing from Spring Boot services
  CDC enables real-time replication to analytics pipelines
  without dual-write risk or polling overhead
```
