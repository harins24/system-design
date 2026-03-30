# Fault Tolerance

**Phase 1 — Core Concepts | Topic 9 of 9**

---

## What is Fault Tolerance?

Fault tolerance is the ability of a system to **continue operating correctly** even when some of its components fail.

The key word is **correctly** — not just "stays up" (that's availability), but continues doing the right thing despite failures.

```
Fault Tolerant System:
Component fails → System detects it → System routes around it →
Users experience nothing (or minimal degradation)

Non-Fault-Tolerant System:
Component fails → System crashes → Users experience full outage
```

---

## Faults vs Failures — Important Distinction

- **Fault** = a defect in a component (a bug, hardware error, network drop)
- **Failure** = when a fault causes the system to deviate from its expected behavior

```
Fault:   A hard drive develops bad sectors
Failure: Data cannot be read → application crashes

Fault:   A network packet is dropped
Failure: A request times out → user sees error
```

Fault tolerance means tolerating faults **before they become failures**. The goal is to break that chain.

---

## The Three Types of Faults

### Transient Faults
Temporary, self-healing. The component works fine after a brief period.

- Network packet loss (retry succeeds)
- Momentary CPU spike causing timeout
- Brief GC pause in JVM

**Solution:** Retry with exponential backoff

### Intermittent Faults
Come and go unpredictably. Hardest to diagnose.

- Flaky network connection
- Memory leak that causes occasional OOM
- Race condition that triggers under specific load

**Solution:** Circuit breaker, extensive logging, chaos engineering

### Permanent Faults
Component is broken and stays broken until replaced.

- Hard drive failure
- Dead server
- Corrupted database

**Solution:** Redundancy, automatic failover, data replication

---

## Fault Tolerance Techniques

### 1. Redundancy
Run multiple copies so one failing doesn't matter.

### 2. Retry with Exponential Backoff
For transient faults, simply retrying often succeeds. But naive retries can make things worse — if 1000 clients all retry every second, you hammer an already struggling service.

Exponential backoff spreads retries out:

```
Attempt 1: wait 1s
Attempt 2: wait 2s
Attempt 3: wait 4s
Attempt 4: wait 8s  + random jitter (±20%)
Attempt 5: give up → send to DLQ
```

**Jitter is critical** — without it, all clients back off to the same interval and retry simultaneously (thundering herd).

```java
// Spring Boot with Resilience4j
@Retry(name = "paymentService", fallbackMethod = "paymentFallback")
public PaymentResponse processPayment(PaymentRequest request) {
    return paymentClient.process(request);
}
```

### 3. Circuit Breaker

Prevents a failing downstream service from cascading failures upstream.

```
CLOSED state (normal):
Requests flow through → monitor failure rate

OPEN state (tripped):
Failure rate exceeds threshold
→ Stop sending requests immediately
→ Return fallback response
→ Wait for reset timeout

HALF-OPEN state (testing):
Send a few test requests
→ If succeed: back to CLOSED
→ If fail: back to OPEN
```

Without circuit breakers, a slow downstream service causes your threads to pile up waiting, exhausting your thread pool, making YOUR service slow, causing YOUR callers to pile up — **cascading failure** across the entire system.

```java
// Resilience4j Circuit Breaker in Spring Boot
@CircuitBreaker(name = "inventoryService", fallbackMethod = "getDefaultInventory")
public Inventory getInventory(String productId) {
    return inventoryClient.get(productId);
}

public Inventory getDefaultInventory(String productId, Exception e) {
    return Inventory.empty(); // graceful degradation
}
```

### 4. Bulkhead Pattern

Isolate components so a failure in one doesn't drain resources from others. Named after the watertight compartments in ship hulls.

```
Without Bulkhead:
[Thread Pool: 200 threads]
All services share the same pool
→ Slow service eats all 200 threads
→ Every other service starved

With Bulkhead:
[Payment: 50 threads] [Inventory: 50 threads] [Orders: 50 threads]
→ Payment service hangs → only its 50 threads affected
→ Inventory and Orders keep working normally
```

### 5. Timeout

Never wait forever. Every external call — HTTP, DB query, Kafka poll — must have a timeout.

```java
// RestTemplate with timeout
RestTemplate restTemplate = new RestTemplateBuilder()
    .connectTimeout(Duration.ofSeconds(2))
    .readTimeout(Duration.ofSeconds(5))
    .build();
```

Without timeouts, one slow dependency holds a thread forever. With enough slow dependencies, your thread pool exhausts and your service dies.

### 6. Graceful Degradation

When a component fails, the system continues with **reduced functionality** rather than failing completely.

```
Full functionality:    Show personalized recommendations
Degraded (cache down): Show generic popular items
Degraded (DB slow):    Show cached results from 5 minutes ago
Degraded (severe):     Show static "our most popular" list

Never: Show error page and refuse to serve anything
```

Amazon's product pages are famous for this — each widget (recommendations, reviews, pricing) loads independently. If recommendations fail, everything else still works.

### 7. Dead Letter Queue (DLQ)

For message processing systems — if a message fails after N retries, move it to a DLQ instead of dropping it or blocking the queue.

```
Normal flow:
[Kafka Topic] → [Consumer] → [Process] → ✅

Failure flow:
[Kafka Topic] → [Consumer] → [Process] → ❌
                                  ↓ retry 3x
                            [Dead Letter Topic]
                                  ↓
                         [Alert + Manual Review]
```

A malformed message shouldn't block all subsequent messages. The DLQ captures it for investigation without stopping the pipeline.

---

## Fault Tolerance vs High Availability vs Reliability

| Concept | Question it answers | Mechanism |
|---------|-------------------|-----------|
| **Fault Tolerance** | Can the system survive component failures? | Redundancy, circuit breakers, retries |
| **High Availability** | What % of time is the system accessible? | Eliminate SPOFs, fast failover |
| **Reliability** | Does the system always do the right thing? | Idempotency, consistency, checksums |

```
A fault tolerant system is usually highly available.
A highly available system is not always fault tolerant.
(It might stay "up" but return wrong results during failures)

All three together = a production-grade system.
```

---

## Chaos Engineering — Proving Fault Tolerance

You can't know if your system is truly fault tolerant until you deliberately break things in production. This is **Chaos Engineering** — pioneered by Netflix with their **Chaos Monkey** tool.

```
Chaos Monkey randomly kills servers in production
→ Forces engineers to build systems that survive it
→ Discovers fault tolerance gaps before real failures do

More advanced:
Chaos Gorilla  → kills entire availability zones
Chaos Kong     → kills entire AWS regions
Latency Monkey → introduces artificial latency
```

The philosophy: **if you're not testing failure, you don't know how your system fails.**

---

## Fault Tolerance in Your Stack

| Technology | Built-in Fault Tolerance |
|-----------|--------------------------|
| **Kafka** | Broker replication, consumer rebalancing, DLQ support |
| **Spring Boot + Resilience4j** | Circuit breaker, retry, bulkhead, rate limiter |
| **Kubernetes** | Auto-restarts failed pods, reschedules on failed nodes |
| **PostgreSQL** | WAL (Write-Ahead Log) ensures durability, streaming replication |
| **Redis Sentinel** | Monitors Redis, promotes replica on primary failure |

---

## Key Takeaways

```
Fault Tolerance: Continue operating correctly despite component failures

Fault types:
  Transient   → retry with exponential backoff + jitter
  Intermittent → circuit breaker + logging
  Permanent   → redundancy + failover

Core techniques:
  Retry + backoff    → handle transient faults
  Circuit Breaker    → prevent cascading failures
  Bulkhead           → isolate failure domains
  Timeout            → never wait forever
  Graceful Degrade   → reduced function > no function
  DLQ                → capture unprocessable messages

Chaos Engineering   → prove fault tolerance by breaking things deliberately
```
