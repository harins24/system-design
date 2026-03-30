# Reliability

**Phase 1 — Core Concepts | Topic 3 of 9**

---

## What is Reliability?

Reliability is the probability that a system performs its intended function correctly over a given period of time, under stated conditions.

If availability asks "is the system up?", reliability asks "does the system do the right thing, every time?"

A system is reliable if:
- ✅ It produces correct results
- ✅ It handles failures gracefully without data corruption
- ✅ It behaves consistently under normal AND abnormal conditions

---

## Why Reliability is Harder Than Availability

You can make a system "available" by just returning HTTP 200 to every request — even if the response is garbage. That's available but completely unreliable.

Reliability is harder because failures in distributed systems are subtle:
- A message gets delivered twice (duplicate processing)
- A message gets lost silently (no error, just gone)
- Data gets partially written (half a transaction committed)
- A service returns stale data without telling you it's stale
- A downstream failure causes silent data corruption upstream

These are far more dangerous than an outright crash, because you don't even know something went wrong.

---

## The Core Properties of a Reliable System

### 1. Correctness
The system produces the right answer. Sounds obvious, but under load, race conditions, network partitions, and retries can all cause incorrect results.

### 2. Durability
Once the system says "I saved your data," it actually saved it — even if the server crashes one second later. This is what the **D in ACID** (Durability) guarantees in databases.

### 3. Fault Tolerance
The system continues operating correctly even when components fail. Not just "stays up" — but "stays correct."

### 4. Consistency (in the data sense)
Data doesn't contradict itself across the system. If you transfer $100 from Account A to Account B, the total money in the system must remain the same — no matter what fails during the transaction.

---

## How Reliability is Achieved

### Replication with Consistency Guarantees
Data is copied to multiple nodes, but with rules ensuring all copies agree. Simply copying data isn't enough — you need consensus on what the "correct" value is.

### Checksums
When data is written or transmitted, a checksum (a mathematical fingerprint) is calculated and stored alongside it. When data is read, the checksum is recalculated and compared. If they don't match, the data was corrupted and the system rejects it. Kafka uses this for every message.

### Idempotency
Operations can be safely retried without causing duplicate effects. Critical for reliability because networks fail and retries are inevitable.

```
Non-idempotent:  POST /payment → charges $100 each time called
Idempotent:      POST /payment?requestId=abc123 → charges $100 only once,
                 subsequent calls with same ID return cached result
```

### Exactly-Once Semantics
The holy grail of messaging systems. Every message is processed exactly once — not lost, not duplicated. Kafka achieves this with idempotent producers and transactional APIs.

### Timeouts, Retries, and Dead Letter Queues
- Always set timeouts — never wait forever
- Retry transient failures with exponential backoff
- After N retries, send to a Dead Letter Queue (DLQ) for manual inspection rather than silently dropping

### Data Validation
Validate inputs at every boundary — don't trust data from other services, queues, or users. A corrupted message that passes through your system undetected is a reliability failure.

---

## Reliability Patterns

- **Circuit Breaker** — stops calling a failing downstream service, prevents cascading failures. Resilience4j in Spring Boot.
- **Saga Pattern** — for distributed transactions across microservices. Instead of one big ACID transaction, a series of local transactions with compensating actions if something fails.
- **Outbox Pattern** — write to your DB and a Kafka event in the same local transaction. Guarantees the event is always published if the DB write succeeds — no "DB saved but Kafka message lost" scenarios.

---

## Reliability Metrics

| Metric | What it measures |
|--------|-----------------|
| **MTBF** (Mean Time Between Failures) | Average time the system runs before a failure |
| **MTTR** (Mean Time To Recovery) | Average time to restore service after failure |
| **Error Rate** | Percentage of requests that result in errors |
| **P99 Latency** | Latency at the 99th percentile — what your worst 1% of users experience |

A reliable system maximizes MTBF and minimizes MTTR.

> The key insight: you can't prevent all failures in distributed systems. Reliability is about detecting failures fast and recovering correctly.

---

## Reliability vs Availability

| | Available | Reliable |
|--|-----------|---------|
| System is up but returns wrong data | ✅ | ❌ |
| System is down for 2 min, restarts cleanly | ❌ | ✅ |
| System processes every message exactly once | ✅ | ✅ |
| System silently drops 0.1% of messages | ✅ | ❌ |

The last row is the most dangerous in practice — and exactly why Kafka's acknowledgment mechanism, replication, and offset tracking exist.

---

## Key Takeaway for Interviews

> "Reliability means the system does the right thing consistently, not just that it stays up. I achieve it through idempotent operations, exactly-once message processing, data validation at every boundary, checksums for data integrity, and proper retry logic with dead letter queues for unrecoverable failures."

That answer covers correctness, durability, and fault tolerance in one breath.
