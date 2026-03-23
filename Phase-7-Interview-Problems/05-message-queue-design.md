# Designing a Message Queue

**Phase 7 — Tradeoffs + Interview Problems | Topic 5 of 8**

---

## Overview

Designing a message queue from scratch is a staff-level question that tests your understanding of durability, ordering, exactly-once delivery, consumer group semantics, and storage engine design. Think Kafka internals — but you're building it.

---

## Phase 1: Requirements

### Functional Requirements

```
Core operations:
  produce(topic, message)      → publish a message to a topic
  consume(topic, groupId)      → read next unread message for group
  acknowledge(messageId)       → mark message as processed

Questions to ask:
  "Should consumers be able to replay messages?"
  "Are messages ordered? Globally or per-partition?"
  "Do we need pub/sub (all groups get all messages) or
   point-to-point (one consumer gets each message)?"
  "Push or pull delivery?"
  "How long should messages be retained?"

For this walkthrough:
  Pub/sub: multiple consumer groups each get all messages
  Per-partition ordering (not global)
  Pull-based delivery (consumers poll)
  Durable: messages persisted to disk
  Configurable retention: default 7 days
  Replay: consumers can seek to any offset
```

### Non-Functional Requirements

```
Scale:
  1M messages/sec write throughput
  10M messages/sec read throughput (10x writes, multiple consumers)

Message size:
  Up to 1MB per message (average 1KB)

Latency:
  End-to-end publish → available to consumer: <100ms P99

Durability:
  No data loss (messages replicated before acknowledging producer)

Availability:
  99.99% — must survive broker failures

Ordering:
  Messages within a partition are strictly ordered
  No global ordering across partitions

Delivery:
  At-least-once delivery (exactly-once via idempotent consumer)
```

---

## Phase 2: Capacity Estimation

```
Write throughput:
  1M messages/sec × 1KB avg = 1 GB/sec inbound

Read throughput:
  10M messages/sec × 1KB = 10 GB/sec outbound

Storage (7-day retention):
  1 GB/sec × 86,400 sec/day × 7 days = 604 TB raw
  Replication factor 3 → 1.8 PB total

Per broker node (assume 10TB SSD, 10 Gbps NIC):
  Storage: 1.8 PB / 10 TB per broker = ~200 broker nodes
  Sequential disk write at 5K × 1KB = 5 MB/sec → easily handled
  Design for disk throughput, not RAM
```

---

## Phase 3: High-Level Design

### APIs

```
Producer API:
  POST /topics/{topic}/messages
  Body: {key?: string, value: bytes, headers?: map}
  Response: {partitionId, offset}

Consumer API:
  GET /topics/{topic}/partitions/{partition}/messages?offset=N&limit=100
  Response: {messages: [{offset, key, value, timestamp}], nextOffset}

Consumer Group API:
  POST /consumer-groups/{groupId}/commit
  Body: {topic, partition, offset}

  GET /consumer-groups/{groupId}/offsets
  Response: {partitions: [{partition, committedOffset, logEndOffset, lag}]}

Admin API:
  POST /topics → create topic with N partitions, replication factor R
```

### System Diagram

```
Producers
   ↓
[Load Balancer]
   ↓
[Broker Cluster]
  ├── Broker 1 (leads Partitions: 0, 3, 6)
  │     Partition 0: [offset 0, 1, 2, ..., N]
  ├── Broker 2 (leads Partitions: 1, 4, 7)
  └── Broker 3 (leads Partitions: 2, 5, 8)
   ↓
[ZooKeeper / Internal Raft] ← cluster coordination, leader election

[Consumer Groups]
  Consumer Group A (payment-service): 3 instances, reads subset of partitions
  Consumer Group B (analytics-service): reads same messages independently
```

---

## Phase 4: Deep Dives

### Deep Dive 1: Partition and Storage Design

```
Topic divided into partitions:
  Topic "orders": 12 partitions
  Each partition = ordered, immutable, append-only log

  Offset: position of message in partition (0-indexed)
  Immutable: messages never updated or deleted
  Append-only: new messages always added to end

Benefits of append-only log:
  Sequential disk writes = fastest possible IO (~3 GB/sec SSD)
  No random seeks = predictable performance
  Offset = cursor = replay any point in history
```

**Segment Files:**
```
Each partition is split into segment files on disk:
  Partition 0 directory:
    00000000000000000000.log  ← messages offset 0 to 999,999
    00000000000001000000.log  ← messages offset 1,000,000 to 1,999,999
    00000000000002000000.log  ← messages offset 2,000,000+ (active)

  Active segment: new messages appended here
  Old segments: immutable, candidates for cleanup

Why segment files (not one giant file)?
  Easier to delete old data (just delete old segment files)
  Each segment has sparse index file for fast offset lookup

Segment index:
  [relative_offset=0 → file_position=0]
  [relative_offset=100 → file_position=14823]
  Binary search to find nearest offset → scan forward
```

```java
public class PartitionLog {

    private final List<Segment> segments;
    private Segment activeSegment;
    private long logEndOffset = 0;

    public long append(byte[] key, byte[] value, Map<String, String> headers) {
        long offset = logEndOffset++;

        LogRecord record = LogRecord.builder()
            .offset(offset).timestamp(Instant.now().toEpochMilli())
            .key(key).value(value).headers(headers).build();

        activeSegment.append(record);

        // Roll segment if too large (1GB)
        if (activeSegment.sizeBytes() > 1_073_741_824) {
            activeSegment.close();
            activeSegment = new Segment(directory, logEndOffset);
            segments.add(activeSegment);
        }

        return offset;
    }

    public List<LogRecord> read(long startOffset, int maxMessages) {
        Segment segment = findSegment(startOffset);
        return segment.read(startOffset, maxMessages);
    }
}
```

---

### Deep Dive 2: Replication

```
Replication factor = 3:
  Each partition has 1 leader + 2 followers
  All writes go to leader
  Followers pull from leader (replicate asynchronously)

ISR (In-Sync Replicas):
  Set of replicas caught up with leader
  Follower falls behind (30s lag) → removed from ISR

  acks=all: wait for ALL replicas in ISR to confirm
  acks=1:   wait for leader only
  acks=0:   fire and forget (fastest, data loss risk)
```

```java
// Producer configuration for zero data loss:
acks = "all"                    // wait for all ISR replicas
min.insync.replicas = 2         // ISR must have ≥ 2 replicas
retries = Integer.MAX_VALUE     // retry until success
enable.idempotence = true       // prevent duplicate messages on retry

// Combined guarantee:
// acks=all + min.insync.replicas=2:
// Write must be confirmed by at least 2 replicas
// Even if leader dies immediately: data on follower → zero data loss
```

**Leader Failover:**
```
Broker 1 (leader for partition 0) crashes:
  ZooKeeper/Raft detects: Broker 1 unreachable
  Controller elects new leader from ISR → Broker 2
  Recovery time: typically 15-30 seconds
  (Detection: 5s + leader election: 5s + propagation: 5s)

  Producer retry handles this transparently
  Consumer re-fetches metadata and routes to Broker 2
```

---

### Deep Dive 3: Consumer Groups

```
Consumer group = logical subscriber
Multiple instances share consumption of a topic's partitions

Topic "orders": 12 partitions
Consumer group "payment-service": 3 consumer instances

Partition assignment:
  Consumer 1 → Partitions 0, 1, 2, 3
  Consumer 2 → Partitions 4, 5, 6, 7
  Consumer 3 → Partitions 8, 9, 10, 11

  Each partition → exactly ONE consumer in the group
  Max parallelism = number of partitions
  4th consumer: idle (no partition to assign)

Offset Management:
  Each group maintains its own offset per partition
  Stored in internal topic: __consumer_offsets

Consumer Rebalance:
  Consumer dies → group coordinator detects (no heartbeat)
  Redistributes its partitions to remaining consumers
  Incremental cooperative rebalance (Kafka 2.4+): gradual, not stop-the-world
```

```java
@KafkaListener(
    topics = "orders",
    groupId = "payment-service",
    concurrency = "4"  // 4 consumer threads
)
public void consume(ConsumerRecord<String, OrderEvent> record,
                     Acknowledgment ack) {
    try {
        paymentService.processPayment(record.value());
        ack.acknowledge(); // commit offset after successful processing
    } catch (RetryableException e) {
        throw e; // Don't ack: message will be re-delivered
    } catch (PermanentException e) {
        deadLetterPublisher.send(record);
        ack.acknowledge(); // ack to move forward, message in DLQ
    }
}
```

---

### Deep Dive 4: Message Routing

```
Option 1: Round robin (no key)
  Even distribution, no ordering guarantee across messages

Option 2: Key-based (consistent hashing) — Standard
  partition = hash(key) % num_partitions

  Same key → always same partition → ordering preserved for that key

  Order events keyed by orderId:
  orderId="123" → Partition 4 (always)
  → OrderCreated, OrderShipped, OrderDelivered all in Partition 4, in order

  CRITICAL: same key → same partition → same consumer instance
  → One consumer processes all events for order 123 → no race conditions
```

---

### Deep Dive 5: Retention and Cleanup

```
Time-based retention:
  Delete segments older than 7 days
  log.retention.hours = 168

Size-based retention:
  Delete oldest segments when total size exceeds limit
  log.retention.bytes = 1099511627776  (1 TB per partition)

Log compaction (different from deletion):
  For changelog topics — keep only LATEST value per key

  Before: [key=A,v=1], [key=B,v=5], [key=A,v=2], [key=A,v=7]
  After:  [key=B,v=5], [key=A,v=7]

  Use case: user profile updates, product price changes
  Consumer can read full topic to reconstruct current state
  Infinite retention possible (log stays bounded)
```

---

### Deep Dive 6: Exactly-Once Delivery

```
At-least-once (default):
  Producer retry → possible duplicate if network issue after broker received
  Consumer retry → possible reprocessing if crash before offset commit

Exactly-once requires:
  1. Idempotent producer (no duplicate writes)
  2. Transactional producer (atomic write across multiple partitions)
  3. Consumer: read_committed isolation (only see committed data)

Idempotent producer:
  Producer gets unique PID, each message tagged with {PID, sequenceNumber}
  Broker deduplicates if sequenceNumber already seen
  enable.idempotence=true

Transactional producer (cross-partition exactly-once):
  producer.beginTransaction();
  producer.send("topic-A", "msg1");
  producer.send("topic-B", "msg2");
  producer.sendOffsetsToTransaction(offsets, consumerGroupId);
  producer.commitTransaction(); // atomic: all or nothing

  Consumer: isolation.level=read_committed
  → Only sees messages from committed transactions
```

---

## Phase 5: Tradeoffs

```
1. Pull vs push delivery:
   Pull: consumer controls rate → prevents overwhelm
   Push: lower latency, server controls flow (can overwhelm slow consumers)
   Choice: pull (Kafka model) for consumer autonomy

2. Sequential disk writes over RAM:
   Sequential writes bypass seek latency → ~500MB/sec sustained
   OS page cache + sendfile() syscall: zero-copy read path
   Counter-intuitive: disk can be faster than random RAM access

3. Partition count:
   More partitions → more parallelism → higher throughput
   More partitions → more overhead (files, replication, rebalance)
   Rule: 2-3x expected consumer parallelism

4. Replication factor:
   RF=2: survive 1 failure, cheaper
   RF=3: survive 2 failures, standard choice
   RF=5: for critical data

5. ZooKeeper vs KRaft:
   KRaft: built-in consensus, no external dependency, Kafka 3.0+
```

---

## Key Takeaways

```
Message queue design covers the full distributed storage stack:

Storage:
  Append-only log per partition (sequential writes = fast)
  Segment files (easy retention management)
  Sparse index per segment (fast offset lookup)
  OS page cache = free read caching

Replication:
  ISR (In-Sync Replicas) concept
  acks=all + min.insync.replicas=2 for zero data loss
  Automatic leader failover on crash

Consumer Groups:
  Each group gets all messages independently
  Within group: each partition → exactly one consumer
  Max parallelism = partition count
  Offsets stored in __consumer_offsets topic

Routing:
  Key-based: same key → same partition → ordered per entity
  Critical for maintaining ordering for same entity

Retention:
  Time-based or size-based deletion
  Log compaction for changelog topics

Delivery guarantees:
  At-most-once: fast, possible loss
  At-least-once: default, need idempotent consumer
  Exactly-once: idempotent producer + transactions + read_committed

Performance patterns:
  Sequential disk writes (not random)
  Zero-copy sendfile() for reads
  Batch producer → amortize network + disk overhead

Real system this maps to: Apache Kafka
```
