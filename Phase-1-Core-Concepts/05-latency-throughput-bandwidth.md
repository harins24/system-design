# Latency, Throughput, and Bandwidth

**Phase 1 — Core Concepts | Topic 5 of 9**

---

## The Water Pipe Analogy

Imagine water flowing through a pipe:

```
Bandwidth  = the WIDTH of the pipe (maximum capacity)
Throughput = the actual amount of water flowing right now
Latency    = how long it takes one drop of water to travel
             from one end to the other
```

A wide pipe doesn't mean water flows fast. A narrow pipe doesn't mean water flows slowly. These are independent dimensions.

---

## Latency

Latency is the time it takes for one unit of work to complete — from the moment a request is sent to the moment a response is received.

**Measured in:** milliseconds (ms) or microseconds (µs)

```
User clicks "Submit"
    ↓  t=0ms
Request leaves browser
    ↓  t=5ms    ← network travel time
Hits your Load Balancer
    ↓  t=6ms
Hits your Spring Boot service
    ↓  t=8ms
DB query executes
    ↓  t=28ms   ← DB was slow
Response travels back
    ↓  t=35ms
User sees result      ← Total latency = 35ms
```

### Sources of Latency

| Source | Typical magnitude |
|--------|-----------------|
| In-memory operation | nanoseconds |
| CPU cache read | ~1ns |
| RAM read | ~100ns |
| SSD read | ~100µs |
| Network (same datacenter) | ~1ms |
| Network (cross-region) | ~100ms |
| HDD seek | ~10ms |
| DB query (with index) | ~1ms |
| DB query (full table scan) | seconds |

### Latency Percentiles — P50, P95, P99

Average latency is misleading. Use percentiles:

```
P50 = 20ms  → half of requests complete in under 20ms
P95 = 80ms  → 95% of requests complete under 80ms
P99 = 500ms → 99% complete under 500ms
              (your worst 1% of users wait half a second)
```

**P99 is what senior engineers care about.** In a system with millions of users, 1% is still tens of thousands of people having a terrible experience. Also, in microservice chains, one slow P99 request poisons the whole chain.

---

## Throughput

Throughput is the number of units of work completed per unit of time.

**Measured in:** requests per second (RPS), transactions per second (TPS), messages per second, MB/s

```
Your Kafka cluster processes 1,000,000 messages/day
= ~11.6 messages/second throughput

Your Spring Boot API handles 500 requests/second
= 500 RPS throughput
```

### What Limits Throughput?

Always one bottleneck at a time (Little's Law territory):

- **CPU bound** — computation is the bottleneck (encryption, compression, complex logic)
- **I/O bound** — waiting for disk or network (most web services are here)
- **Memory bound** — running out of RAM, causing swapping
- **Connection bound** — DB connection pool exhausted, threads waiting

The bottleneck shifts as you fix each one. You can't improve throughput without identifying and fixing the current bottleneck.

---

## Bandwidth

Bandwidth is the maximum theoretical data transfer capacity of a link.

**Measured in:** Mbps, Gbps

```
Your server has a 1 Gbps network card
→ Maximum 1 Gbps of data can flow in or out
→ This is the ceiling — throughput can never exceed it
```

Bandwidth is the physical constraint. Throughput is what you actually achieve within that constraint.

In practice, bandwidth is rarely the bottleneck in modern cloud systems — you're more likely to be bottlenecked by CPU, DB, or application logic long before saturating network bandwidth.

---

## How They Relate — The Critical Insight

```
High bandwidth + High latency = Fast highway with a distant destination
                                (video streaming: lots of data, but buffering at start)

Low bandwidth + Low latency  = Narrow but nearby pipe
                                (real-time trading: tiny messages, must be instant)
```

### High throughput ≠ Low latency

You often have to choose between optimizing latency OR throughput — they pull in opposite directions:

| Optimization | Effect on Latency | Effect on Throughput |
|-------------|------------------|---------------------|
| Batching requests | ❌ Increases (wait to fill batch) | ✅ Improves (fewer round trips) |
| Async processing | ❌ Increases (no immediate response) | ✅ Improves (no blocking) |
| Caching | ✅ Reduces | ✅ Improves |
| Compression | ❌ Slight increase (CPU cost) | ✅ Improves (less data over wire) |
| Connection pooling | ✅ Reduces | ✅ Improves |

Kafka is the perfect example of choosing **throughput over latency**. Kafka batches messages, compresses them, and writes sequentially to disk — this introduces some latency per message but achieves extraordinary throughput. That's the right tradeoff for a log processing system. It would be the wrong tradeoff for a real-time trading system.

---

## In System Design Interviews

When you discuss performance, always clarify which dimension matters:

- **Chat app** → latency matters most (messages must feel instant)
- **Video streaming** → throughput and bandwidth matter most (smooth playback)
- **Batch ETL pipeline** → throughput matters most (process all records fast)
- **Financial transactions** → latency AND correctness matter (fast and accurate)
- **Search autocomplete** → latency matters most (must respond in <100ms or feels broken)

Always ask the interviewer: *"Are we optimizing for latency or throughput here? That changes the design significantly."*

---

## Key Numbers to Memorize

```
Human perception threshold:    ~100ms  (feels "instant" below this)
Acceptable web response:       <200ms
"Slow" threshold:              >1 second (user notices)
"Broken" threshold:            >3 seconds (user leaves)

Same datacenter round trip:    ~1ms
Cross-region (US East→West):   ~60ms
US to Europe:                  ~100ms
US to Asia:                    ~150-200ms
```
