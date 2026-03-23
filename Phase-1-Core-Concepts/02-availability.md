# Availability

**Phase 1 — Core Concepts | Topic 2 of 9**

---

## What is Availability?

Availability is the percentage of time a system is operational and accessible when a user tries to use it.

```
Availability = Uptime / (Uptime + Downtime)
```

---

## The Nines of Availability

| Availability | Downtime per year | Downtime per month | Downtime per day |
|-------------|------------------|--------------------|-----------------|
| 90% (1 nine) | 36.5 days | 73 hours | 2.4 hours |
| 99% (2 nines) | 3.65 days | 7.3 hours | 14.4 minutes |
| 99.9% (3 nines) | 8.76 hours | 43.8 minutes | 1.4 minutes |
| 99.99% (4 nines) | 52.6 minutes | 4.4 minutes | 8.6 seconds |
| 99.999% (5 nines) | 5.26 minutes | 26 seconds | 0.9 seconds |

Most production systems target 99.9% to 99.99%. Five nines is extremely hard and expensive to achieve — that's what companies like AWS and Google aim for on their core infrastructure.

When an interviewer asks "what's your SLA?", this table is what they're referencing.

---

## How Availability is Achieved

### 1. Eliminate Single Points of Failure (SPOF)
If one component going down takes the whole system down, that's a SPOF. Fix it by adding redundancy — multiple instances, multiple data centers, multiple everything critical.

### 2. Redundancy
Run multiple copies of every critical component so if one fails, others take over.

```
Without redundancy:   [App] → [DB]          ← DB goes down = full outage
With redundancy:      [App] → [Primary DB]
                              [Replica DB]   ← Primary fails, replica takes over
```

### 3. Failover
The automatic process of switching to a backup when the primary fails. Can be:
- **Active-Passive**: backup sits idle, takes over on failure (some downtime during switchover)
- **Active-Active**: both run simultaneously, load is shared (zero downtime, but complex)

### 4. Health Checks & Load Balancers
Load balancers continuously ping services. If a service doesn't respond, traffic is automatically routed away from it. Your healthy instances absorb the load.

### 5. Geographic Distribution
Deploy across multiple availability zones or regions. A data center fire in Virginia doesn't take down your users in California.

---

## Availability in Series vs Parallel

### Components in Series — if any one fails, the whole chain fails:
```
Availability = A × B × C
Example: 99.9% × 99.9% × 99.9% = 99.7%
```
Every microservice you add to a synchronous call chain reduces overall availability. This is why long chains of synchronous service calls are dangerous.

### Components in Parallel — if one fails, others handle it:
```
Availability = 1 - (1-A) × (1-B)
Example: 1 - (0.001 × 0.001) = 99.9999%
```
Redundancy dramatically increases availability. This is the mathematical justification for running multiple instances.

> **Real implication:** If your Spring Boot service calls 5 downstream services synchronously and each is 99.9% available, your overall availability is only ~99.5%. This is why async communication via Kafka improves system availability — you break the synchronous chain.

---

## Availability vs Reliability

| | Available | Reliable |
|--|-----------|---------|
| System is up but returns wrong data | ✅ | ❌ |
| System is down for 2 min, restarts cleanly | ❌ | ✅ |
| System processes every message exactly once | ✅ | ✅ |
| System silently drops 0.1% of messages | ✅ | ❌ |

- **Availability** = is the system accessible right now?
- **Reliability** = does the system produce correct results consistently over time?

---

## Availability vs Consistency (CAP Preview)

During a network partition, do you want your system to:
- **Stay available** (respond, possibly with stale data), or
- **Stay consistent** (refuse to respond until it's sure the data is correct)?

Different systems make different choices — Kafka favors availability, traditional RDBMS favors consistency.

---

## 🎯 Interview Q&A

**Q1. Your system has three microservices called sequentially: Auth (99.9%), Order (99.9%), Payment (99.9%). What is the overall availability? What would you do to improve it?**

`99.9% × 99.9% × 99.9% = 99.7%` — that means ~26 hours of downtime per year just from chaining three services.

How to fix it:
- **Break the synchronous chain** — make Order and Payment calls async via Kafka. Auth still needs to be synchronous (verify identity first), but after that, fire an event and respond immediately.
- **Add caching to Auth** — cache validated tokens so Auth service downtime doesn't block every request.
- **Run multiple instances** of each service behind a load balancer.

**Q2. A stakeholder says "we need five nines availability for our internal admin tool used 9-5 Monday to Friday." How do you respond?**

This is a pushback question. Five nines = 5 minutes downtime per year, requiring active-active multi-region deployment, automated failover under 30 seconds, chaos engineering, and 24/7 on-call teams.

The right answer: *"99.9% gives us 8.7 hours downtime per year — most of which we can schedule during nights and weekends when nobody is using it. The cost of five nines isn't justified here. Let's target 99.9% with a good deployment process."*

Availability requirements should match business impact. Five nines is for payment processing and hospital systems, not internal dashboards.

**Q3. How does Kafka's design contribute to availability?**

- **Replication** — every partition has a leader and N replicas on different brokers. If the leader broker dies, one replica is automatically elected as the new leader.
- **No single broker dependency** — a Kafka cluster with 3 brokers can lose 1 broker and keep running. With replication factor 3, it can lose 2 brokers.
- **Consumer Groups** — if one consumer instance dies, Kafka triggers a rebalance and redistributes partitions to surviving consumers.
- **Persistence** — messages are written to disk. Even if all consumers go down, messages wait safely in Kafka. This is fundamentally different from a synchronous HTTP call which just fails.
