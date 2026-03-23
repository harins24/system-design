# Scalability

**Phase 1 — Core Concepts | Topic 1 of 9**

---

## What is Scalability?

Scalability is a system's ability to handle increasing load without degrading performance or requiring a complete redesign.

"Load" can mean different things depending on the system — requests per second, concurrent users, data volume, or message throughput.

---

## The Two Types of Scaling

### Vertical Scaling (Scale Up)
You make a single machine more powerful — more CPU, more RAM, faster disk.

```
Before:  [Server: 4 CPU, 16GB RAM]
After:   [Server: 32 CPU, 256GB RAM]
```

Simple, but has a hard ceiling. The biggest machine AWS sells is still a single point of failure. Also, it usually requires downtime to upgrade.

### Horizontal Scaling (Scale Out)
You add more machines and distribute the load across them.

```
Before:  [Server A]
After:   [Server A] [Server B] [Server C]  ← behind a Load Balancer
```

This is how companies like Netflix and Google scale. No hard ceiling, no single point of failure — but it introduces complexity: how do you keep data consistent across machines? How does a client know which server to talk to?

---

## The Dimensions of Scalability

A well-designed system scales across three axes (from the "Scale Cube"):

| Axis | What it means | Example |
|------|--------------|---------|
| X-axis | Clone/replicate the whole service | Multiple instances of your Spring Boot app |
| Y-axis | Split by function (microservices) | Separate Order, Payment, Inventory services |
| Z-axis | Split by data partition | Each shard handles users A–M or N–Z |

Your Kafka work at Your Company is a perfect real-world example — you horizontally scale consumers by adding more consumer instances in a consumer group, and you scale throughput by adding more partitions (Z-axis data partitioning).

---

## What Makes a System Hard to Scale?

- **Shared mutable state** — if all servers write to the same database row, you have a bottleneck
- **Sticky sessions** — if a user must always hit the same server, you can't freely add/remove servers
- **Synchronous dependencies** — if Service A waits for Service B, B becomes your ceiling
- **Stateful services** — state stored in memory means you can't just spin up a new instance

This is why stateless services (each request carries all needed context, nothing stored in server memory) scale far more easily.

---

## Scalability vs Performance

These are often confused in interviews:

- **Performance** = how fast the system responds for one user
- **Scalability** = how the system behaves as more users arrive

A system can be fast but not scalable (handles 100 users great, falls over at 10,000). A system can be scalable but slow (always takes 2 seconds, but stays at 2 seconds even with 1 million users).

---

## How to Describe Scalability in an Interview

When asked "how would you scale X?", always follow this structure:

1. **Identify the bottleneck** — is it CPU, memory, I/O, network, or the database?
2. **Quantify the load** — how many requests/sec? How much data?
3. **Apply the right scaling strategy** — horizontal scaling, caching, sharding, async processing
4. **Acknowledge the tradeoffs** — consistency, complexity, cost

---

## 🎯 Interview Q&A

**Q1. Your Spring Boot microservice handles 500 requests/sec fine. At 5,000 req/sec it starts timing out. What are the first 3 things you investigate?**

When a service times out under 10x load, investigate:
- **Thread pool exhaustion** — Spring Boot's embedded Tomcat has a default of 200 threads. At 5,000 req/sec they're all busy waiting, new requests queue up and timeout. Fix: tune thread pool, or switch to reactive (WebFlux).
- **Database connection pool** — HikariCP default is 10 connections. Every request waiting for a DB connection = bottleneck. Fix: tune pool size or add read replicas.
- **Downstream service** — if your service calls another service synchronously, that service becomes your ceiling. Fix: async/Kafka, circuit breaker.

Then scale: add more instances behind a load balancer, add caching to reduce DB hits, make calls async.

**Q2. A teammate says "let's just scale vertically — it's simpler." What's your response?**

Vertical scaling has a hard ceiling on how powerful a single machine can be. More importantly, vertical scaling means downtime during the upgrade, which is unacceptable for production systems. And it's a single point of failure — one big machine goes down, everything goes down. Horizontal scaling gives you fault tolerance too, not just capacity.

**Q3. In your Kafka setup, when you add more consumer instances — which axis of the Scale Cube is that?**

Adding more consumer instances is the **X-axis** (cloning/replication — you're running multiple copies of the same consumer). Kafka partitions are the **Z-axis** (data partitioning — each partition is a slice of the data). It only works up to a certain point: you can't have more active consumers than partitions. If you have 10 partitions and spin up 15 consumers, 5 sit idle. So to scale consumers, you must first scale partitions.

---

## Quick Summary

| Scenario | Right answer |
|----------|-------------|
| Service timing out under load | Check threads, DB pool, downstream deps first |
| Vertical vs Horizontal | Vertical = simpler but has ceiling + SPOF; Horizontal = complex but unlimited + fault tolerant |
| Kafka consumers | X-axis scaling, capped by partition count |
