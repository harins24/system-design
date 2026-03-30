# System Design Learning Roadmap

A complete, structured curriculum covering system design from core concepts through distributed systems and interview-ready problem walkthroughs — 62 topics across 7 phases.

---

## 📚 Phases Overview

| Phase | Topic | Files |
|-------|-------|-------|
| [Phase 1](#phase-1--core-concepts) | Core Concepts | 9 topics |
| [Phase 2](#phase-2--networking-fundamentals) | Networking Fundamentals | 8 topics |
| [Phase 3](#phase-3--api-fundamentals) | API Fundamentals | 8 topics |
| [Phase 4](#phase-4--database-fundamentals) | Database Fundamentals | 9 topics |
| [Phase 5](#phase-5--caching--async-communication) | Caching + Async Communication | 8 topics |
| [Phase 6](#phase-6--distributed-systems--architecture) | Distributed Systems & Architecture | 12 topics |
| [Phase 7](#phase-7--interview-problems) | Interview Problems | 8 topics |

---

## Phase 1 — Core Concepts

> Foundations every system design interview builds on. Understand these cold before moving on.

| # | Topic | File |
|---|-------|------|
| 1 | Scalability | [01-scalability.md](system-design-fundamentals/Phase-1-Core-Concepts/01-scalability.md) |
| 2 | Availability | [02-availability.md](system-design-fundamentals/Phase-1-Core-Concepts/02-availability.md) |
| 3 | Reliability | [03-reliability.md](system-design-fundamentals/Phase-1-Core-Concepts/03-reliability.md) |
| 4 | Single Point of Failure (SPOF) | [04-single-point-of-failure.md](system-design-fundamentals/Phase-1-Core-Concepts/04-single-point-of-failure.md) |
| 5 | Latency vs Throughput vs Bandwidth | [05-latency-throughput-bandwidth.md](system-design-fundamentals/Phase-1-Core-Concepts/05-latency-throughput-bandwidth.md) |
| 6 | Consistent Hashing | [06-consistent-hashing.md](system-design-fundamentals/Phase-1-Core-Concepts/06-consistent-hashing.md) |
| 7 | CAP Theorem | [07-cap-theorem.md](system-design-fundamentals/Phase-1-Core-Concepts/07-cap-theorem.md) |
| 8 | Failover | [08-failover.md](system-design-fundamentals/Phase-1-Core-Concepts/08-failover.md) |
| 9 | Fault Tolerance | [09-fault-tolerance.md](system-design-fundamentals/Phase-1-Core-Concepts/09-fault-tolerance.md) |

---

## Phase 2 — Networking Fundamentals

> How data moves across the internet. Essential context for designing any networked system.

| # | Topic | File |
|---|-------|------|
| 1 | OSI Model | [01-osi-model.md](system-design-fundamentals/Phase-2-Networking-Fundamentals/01-osi-model.md) |
| 2 | IP Addresses | [02-ip-addresses.md](system-design-fundamentals/Phase-2-Networking-Fundamentals/02-ip-addresses.md) |
| 3 | DNS | [03-dns.md](system-design-fundamentals/Phase-2-Networking-Fundamentals/03-dns.md) |
| 4 | Proxy vs Reverse Proxy | [04-proxy-vs-reverse-proxy.md](system-design-fundamentals/Phase-2-Networking-Fundamentals/04-proxy-vs-reverse-proxy.md) |
| 5 | HTTP / HTTPS | [05-http-https.md](system-design-fundamentals/Phase-2-Networking-Fundamentals/05-http-https.md) |
| 6 | TCP vs UDP | [06-tcp-vs-udp.md](system-design-fundamentals/Phase-2-Networking-Fundamentals/06-tcp-vs-udp.md) |
| 7 | Load Balancing | [07-load-balancing.md](system-design-fundamentals/Phase-2-Networking-Fundamentals/07-load-balancing.md) |
| 8 | Checksums | [08-checksums.md](system-design-fundamentals/Phase-2-Networking-Fundamentals/08-checksums.md) |

---

## Phase 3 — API Fundamentals

> How services communicate. Directly relevant to Spring Boot microservices work.

| # | Topic | File |
|---|-------|------|
| 1 | APIs | [01-apis.md](system-design-fundamentals/Phase-3-API-Fundamentals/01-apis.md) |
| 2 | API Gateway | [02-api-gateway.md](system-design-fundamentals/Phase-3-API-Fundamentals/02-api-gateway.md) |
| 3 | REST vs GraphQL | [03-rest-vs-graphql.md](system-design-fundamentals/Phase-3-API-Fundamentals/03-rest-vs-graphql.md) |
| 4 | WebSockets | [04-websockets.md](system-design-fundamentals/Phase-3-API-Fundamentals/04-websockets.md) |
| 5 | Webhooks | [05-webhooks.md](system-design-fundamentals/Phase-3-API-Fundamentals/05-webhooks.md) |
| 6 | Idempotency | [06-idempotency.md](system-design-fundamentals/Phase-3-API-Fundamentals/06-idempotency.md) |
| 7 | Rate Limiting | [07-rate-limiting.md](system-design-fundamentals/Phase-3-API-Fundamentals/07-rate-limiting.md) |
| 8 | API Design | [08-api-design.md](system-design-fundamentals/Phase-3-API-Fundamentals/08-api-design.md) |

---

## Phase 4 — Database Fundamentals

> How data is stored, indexed, replicated, and scaled. The most interview-tested area.

| # | Topic | File |
|---|-------|------|
| 1 | ACID Transactions | [01-acid-transactions.md](system-design-fundamentals/Phase-4-Database-Fundamentals/01-acid-transactions.md) |
| 2 | SQL vs NoSQL | [02-sql-vs-nosql.md](system-design-fundamentals/Phase-4-Database-Fundamentals/02-sql-vs-nosql.md) |
| 3 | Database Indexes | [03-database-indexes.md](system-design-fundamentals/Phase-4-Database-Fundamentals/03-database-indexes.md) |
| 4 | Database Sharding | [04-database-sharding.md](system-design-fundamentals/Phase-4-Database-Fundamentals/04-database-sharding.md) |
| 5 | Data Replication | [05-data-replication.md](system-design-fundamentals/Phase-4-Database-Fundamentals/05-data-replication.md) |
| 6 | Database Scaling | [06-database-scaling.md](system-design-fundamentals/Phase-4-Database-Fundamentals/06-database-scaling.md) |
| 7 | Types of Databases | [07-database-types.md](system-design-fundamentals/Phase-4-Database-Fundamentals/07-database-types.md) |
| 8 | Bloom Filters | [08-bloom-filters.md](system-design-fundamentals/Phase-4-Database-Fundamentals/08-bloom-filters.md) |
| 9 | Database Architectures | [09-database-architectures.md](system-design-fundamentals/Phase-4-Database-Fundamentals/09-database-architectures.md) |

---

## Phase 5 — Caching + Async Communication

> How to reduce latency, offload databases, and decouple services asynchronously.

| # | Topic | File |
|---|-------|------|
| 1 | Caching 101 | [01-caching-101.md](system-design-fundamentals/Phase-5-Caching-and-Async-Communication/01-caching-101.md) |
| 2 | Caching Strategies | [02-caching-strategies.md](system-design-fundamentals/Phase-5-Caching-and-Async-Communication/02-caching-strategies.md) |
| 3 | Cache Eviction Policies | [03-cache-eviction-policies.md](system-design-fundamentals/Phase-5-Caching-and-Async-Communication/03-cache-eviction-policies.md) |
| 4 | Distributed Caching | [04-distributed-caching.md](system-design-fundamentals/Phase-5-Caching-and-Async-Communication/04-distributed-caching.md) |
| 5 | Content Delivery Network (CDN) | [05-cdn.md](system-design-fundamentals/Phase-5-Caching-and-Async-Communication/05-cdn.md) |
| 6 | Pub/Sub | [06-pub-sub.md](system-design-fundamentals/Phase-5-Caching-and-Async-Communication/06-pub-sub.md) |
| 7 | Message Queues | [07-message-queues.md](system-design-fundamentals/Phase-5-Caching-and-Async-Communication/07-message-queues.md) |
| 8 | Change Data Capture (CDC) | [08-change-data-capture.md](system-design-fundamentals/Phase-5-Caching-and-Async-Communication/08-change-data-capture.md) |

---

## Phase 6 — Distributed Systems & Architecture

> Where everything comes together. The topics that separate senior engineers from lead engineers.

| # | Topic | File |
|---|-------|------|
| 1 | Microservices Architecture | [01-microservices-architecture.md](system-design-fundamentals/Phase-6-Distributed-Systems-and-Architecture/01-microservices-architecture.md) |
| 2 | CQRS and Event Sourcing | [02-cqrs-event-sourcing.md](system-design-fundamentals/Phase-6-Distributed-Systems-and-Architecture/02-cqrs-event-sourcing.md) |
| 3 | Service Discovery and Load Balancing | [03-service-discovery-load-balancing.md](system-design-fundamentals/Phase-6-Distributed-Systems-and-Architecture/03-service-discovery-load-balancing.md) |
| 4 | Distributed Transactions | [04-distributed-transactions.md](system-design-fundamentals/Phase-6-Distributed-Systems-and-Architecture/04-distributed-transactions.md) |
| 5 | Consistency Patterns | [05-consistency-patterns.md](system-design-fundamentals/Phase-6-Distributed-Systems-and-Architecture/05-consistency-patterns.md) |
| 6 | Fault Tolerance and Resilience Patterns | [06-fault-tolerance-resilience.md](system-design-fundamentals/Phase-6-Distributed-Systems-and-Architecture/06-fault-tolerance-resilience.md) |
| 7 | Distributed Tracing and Observability | [07-distributed-tracing-observability.md](system-design-fundamentals/Phase-6-Distributed-Systems-and-Architecture/07-distributed-tracing-observability.md) |
| 8 | Deployment Strategies | [08-deployment-strategies.md](system-design-fundamentals/Phase-6-Distributed-Systems-and-Architecture/08-deployment-strategies.md) |
| 9 | Scalability Patterns | [09-scalability-patterns.md](system-design-fundamentals/Phase-6-Distributed-Systems-and-Architecture/09-scalability-patterns.md) |
| 10 | Consensus Algorithms | [10-consensus-algorithms.md](system-design-fundamentals/Phase-6-Distributed-Systems-and-Architecture/10-consensus-algorithms.md) |
| 11 | MapReduce and Batch Processing | [11-mapreduce-batch-processing.md](system-design-fundamentals/Phase-6-Distributed-Systems-and-Architecture/11-mapreduce-batch-processing.md) |
| 12 | Stream Processing | [12-stream-processing.md](system-design-fundamentals/Phase-6-Distributed-Systems-and-Architecture/12-stream-processing.md) |

---

## Phase 7 — Interview Problems

> Full system design walkthroughs using the five-phase interview framework. Easy → Hard.

| # | Topic | File |
|---|-------|------|
| 1 | How to Approach System Design Interviews | [01-system-design-interview-approach.md](system-design-fundamentals/Phase-7-Interview-Problems/01-system-design-interview-approach.md) |
| 2 | Design a URL Shortener | [02-url-shortener.md](system-design-fundamentals/Phase-7-Interview-Problems/02-url-shortener.md) |
| 3 | Design a Rate Limiter | [03-rate-limiter-design.md](system-design-fundamentals/Phase-7-Interview-Problems/03-rate-limiter-design.md) |
| 4 | Design a Key-Value Store | [04-key-value-store-design.md](system-design-fundamentals/Phase-7-Interview-Problems/04-key-value-store-design.md) |
| 5 | Design a Message Queue | [05-message-queue-design.md](system-design-fundamentals/Phase-7-Interview-Problems/05-message-queue-design.md) |
| 6 | Design a Social Media Feed | [06-social-media-feed-design.md](system-design-fundamentals/Phase-7-Interview-Problems/06-social-media-feed-design.md) |
| 7 | Design a Notification System | [07-notification-system-design.md](system-design-fundamentals/Phase-7-Interview-Problems/07-notification-system-design.md) |
| 8 | Design a Ride-Sharing System (Uber/Lyft) | [08-ride-sharing-design.md](system-design-fundamentals/Phase-7-Interview-Problems/08-ride-sharing-design.md) |

---

*62 topics · 7 phases · Full curriculum*
