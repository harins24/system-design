# Database Scaling

**Phase 4 — Database Fundamentals | Topic 6 of 9**

---

## The Database Scaling Problem

Everything in your system can scale horizontally with relative ease — add more app servers, add more Kafka brokers. The database is always the hardest part to scale because it holds state, and state has to be consistent.

```
Easy to scale:
  [App Server] → add 10 more, put behind LB → done
  [Kafka]      → add brokers, rebalance partitions → done

Hard to scale:
  [Database]   → single node with all your data
                 can't just add more databases
                 data must be consistent across all of them
                 writes can't go everywhere simultaneously
```

---

## The Database Scaling Ladder

Always climb this ladder in order. Each step handles 10-100x more load than the previous.

```
Step 1: Optimize queries and add indexes
Step 2: Connection pooling
Step 3: Caching (Redis)
Step 4: Read replicas
Step 5: Vertical scaling
Step 6: Table partitioning
Step 7: Sharding
Step 8: Functional decomposition
```

---

## Step 1: Query Optimization and Indexes

The highest ROI step. A single missing index can make a query 10,000x faster.

```
Slow query log (PostgreSQL):
log_min_duration_statement = 100  # log queries > 100ms

Identify top 5 slowest queries
EXPLAIN ANALYZE each one
Look for Seq Scan on large tables
Add appropriate indexes

Before optimization:  SELECT * FROM orders WHERE user_id = 'x'
                     → Seq Scan: 3,241ms (10M rows)

After adding index:  CREATE INDEX ON orders(user_id)
                     → Index Scan: 0.091ms

Cost: zero hardware, zero infrastructure change
Gain: 35,000x faster for that query
```

---

## Step 2: Connection Pooling

Database connections are expensive — each consumes ~10MB memory and takes 20-50ms to establish.

```
Without connection pooling:
  100 concurrent requests
  → 100 new DB connections opened
  → 100 × 10MB = 1GB RAM on DB server just for connections
  → 100 × 50ms = wasted time establishing connections
  → DB overwhelmed by connection overhead

With connection pooling (HikariCP):
  Pool maintains 20 persistent connections
  100 requests share those 20 connections
  Requests queue briefly if pool busy
  → 80% reduction in DB connection overhead
```

```java
// Spring Boot application.properties
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000    // 30s wait for connection
spring.datasource.hikari.idle-timeout=600000         // 10min idle before close
spring.datasource.hikari.max-lifetime=1800000        // 30min max connection age

// Pool sizing formula:
// pool_size = (core_count * 2) + effective_spindle_count
// For 4 CPU cores, SSD: (4 * 2) + 1 = 9 connections
// Don't blindly set to 100 — more connections ≠ better performance

// PgBouncer for PostgreSQL (connection pooler as separate service):
// App → PgBouncer (maintains small pool) → PostgreSQL
// Handles thousands of app connections, maps to small DB pool
// Critical for apps with many microservice instances
```

---

## Step 3: Read Replicas

Typical production workload: 80% reads, 20% writes.

```
Without replicas:
  Primary handles all 100% → bottleneck

With 3 read replicas:
  Primary handles 20% (writes only)
  Each replica handles ~27% of reads
  → Primary load reduced 5x on reads

AWS RDS read replicas:
  Up to 15 read replicas
  Each can have its own replicas (cascading)
  Different instance sizes per replica
  → Analytics replica can be large (memory-optimized)
  → App replicas can be smaller (compute-optimized)
```

---

## Step 4: Caching

The biggest multiplier. Cache hits never touch the database.

```
Cache hit ratio = cache hits / total requests

90% cache hit ratio:
  10M requests/day total
  9M served from Redis → zero DB load for these
  1M hit DB → database sees only 10% of actual traffic

Cache the right things:
  ✅ Frequently read, rarely written (product catalog, user profiles)
  ✅ Expensive to compute (aggregations, complex joins)
  ✅ Tolerable staleness (showing 5-minute-old data is OK)

  ❌ Highly personalized per-user (cache hit rate low)
  ❌ Frequently changing (cache invalidation cost > benefit)
  ❌ Financial data requiring exact accuracy
```

---

## Step 5: Vertical Scaling

Before horizontal complexity, buy a bigger machine.

```
Typical progression:
  db.t3.medium   → 2 vCPU, 4GB RAM   → dev/test
  db.t3.large    → 2 vCPU, 8GB RAM   → small prod
  db.m6g.xlarge  → 4 vCPU, 16GB RAM  → growing prod
  db.m6g.4xlarge → 16 vCPU, 64GB RAM → serious prod
  db.r6g.8xlarge → 32 vCPU, 256GB RAM → large prod
  db.r6g.16xlarge→ 64 vCPU, 512GB RAM → very large

RAM is the key resource:
  If entire working set fits in RAM → disk IO eliminated
  512GB RAM instance often eliminates need for sharding
  PostgreSQL shared_buffers = 25% of RAM
  effective_cache_size = 75% of RAM

This step is underutilized. Engineers jump to sharding
when a $2,000/month larger RDS instance would solve it.
```

---

## Step 6: Table Partitioning

Split a large table into smaller physical partitions within the SAME database instance. The database handles routing transparently.

```sql
-- Orders table growing to 10 billion rows
-- Partition by month automatically

CREATE TABLE orders (
    id          UUID,
    user_id     UUID,
    total       DECIMAL,
    created_at  TIMESTAMP,
    status      VARCHAR(20)
) PARTITION BY RANGE (created_at);

-- Create partitions (automate with cron job)
CREATE TABLE orders_2026_01
    PARTITION OF orders
    FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

CREATE TABLE orders_2026_02
    PARTITION OF orders
    FOR VALUES FROM ('2026-02-01') TO ('2026-03-01');

-- Application queries unchanged:
SELECT * FROM orders WHERE created_at > '2026-01-15';
-- → PostgreSQL routes to orders_2026_01 only
-- → Scans 1/12th the data instead of entire table

-- Old partitions easy to archive:
ALTER TABLE orders DETACH PARTITION orders_2024_01;
-- Detach instantly, move to cold storage or drop
```

**Partition pruning** — when a query includes the partition key in its WHERE clause, the database scans only relevant partitions:

```
10 billion rows across 24 monthly partitions
= ~417M rows per partition

Query: WHERE created_at BETWEEN '2026-01-01' AND '2026-01-31'
→ Scans only orders_2026_01
→ 417M rows instead of 10 billion
→ 24x faster, same hardware
```

### Partition Strategies

```
Range partitioning (most common for time-series):
  PARTITION BY RANGE (created_at)
  → Monthly, quarterly, yearly partitions
  → Old partitions naturally go cold

List partitioning (by discrete values):
  PARTITION BY LIST (status)
  → PARTITION FOR VALUES IN ('PENDING', 'PROCESSING')
  → PARTITION FOR VALUES IN ('SHIPPED', 'DELIVERED')
  → Queries filtered by status scan one partition

Hash partitioning (for even distribution):
  PARTITION BY HASH (user_id)
  → 8 partitions, each gets 1/8 of users
  → Even write distribution across partitions
  → Good when no natural range key
```

---

## Step 7: Sharding

When you've exhausted steps 1-6 and still need more:

```
Read throughput: add more read replicas (nearly unlimited)
Write throughput: this is where sharding becomes necessary

Signs you need write sharding:
  → Primary CPU > 70% consistently despite optimization
  → WAL write rate saturating disk I/O
  → Lock contention on hot rows despite query optimization
  → Replication lag growing (primary generating WAL faster than replicas consume)

Sharding doubles your write capacity:
  Before: 1 primary → 50K writes/sec
  After:  2 primaries → 100K writes/sec
          (each handles half the data)
```

---

## Step 8: Functional Decomposition

Split one large database into multiple smaller databases by business domain.

```
Before:
  [Monolith DB]
  tables: users, orders, payments, inventory,
          notifications, analytics, sessions,
          products, shipping, reviews...

After (separate database per domain):
  [User DB]        → users, authentication, profiles
  [Order DB]       → orders, order_items, order_history
  [Payment DB]     → payments, refunds, invoices
  [Inventory DB]   → products, stock, warehouses
  [Analytics DB]   → read-optimized, denormalized

Benefits:
  Each DB sized for its domain's actual load
  Each DB optimized for its access patterns
  Failures isolated (payment DB down ≠ orders down)
  Teams own their databases independently

This is what microservices mandate:
  Each service owns its database
  No shared database between services
  Services communicate via APIs or events (Kafka)
```

---

## CQRS — Command Query Responsibility Segregation

A scaling pattern that separates the write model from the read model:

```
Traditional (same model for reads and writes):
  Write: INSERT/UPDATE normalized tables
  Read:  JOIN multiple tables → complex, slow for reporting

CQRS:
  Command side (writes):
    Normalized, ACID, PostgreSQL
    Optimized for correctness and transactional integrity

  Query side (reads):
    Denormalized, pre-joined, read-optimized
    Elasticsearch, Redis, read replicas, materialized views
    Optimized for query speed and flexibility

How they stay in sync:
  Write to Command DB → publish event to Kafka
  Event consumer updates Query DB

  Trade-off: eventual consistency between command and query sides
             (query model may be slightly behind)
```

```
Example: E-commerce order search

Command side (PostgreSQL, normalized):
  orders → order_items → products → users

Query side (Elasticsearch, denormalized):
{
  "orderId": "456",
  "userId": "123",
  "userName": "Hari Kumar",       ← denormalized
  "products": ["Laptop Stand"],   ← denormalized
  "total": 99.99,
  "status": "SHIPPED"
}

Search: "all orders for 'Hari' containing 'laptop' under $200"
→ Elasticsearch: instant full-text, filtered search
→ PostgreSQL: would require complex JOINs + full-text scan
```

---

## Materialized Views — Pre-Computed Results

Store query results as a physical table, refreshed periodically:

```sql
-- Expensive query computed every page load:
SELECT u.id, u.name,
       COUNT(o.id) as order_count,
       SUM(o.total) as lifetime_value,
       MAX(o.created_at) as last_order_date
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

-- Takes 30 seconds on 100M rows

-- Materialized view: compute once, query instantly
CREATE MATERIALIZED VIEW user_order_stats AS
SELECT u.id, u.name,
       COUNT(o.id) as order_count,
       SUM(o.total) as lifetime_value,
       MAX(o.created_at) as last_order_date
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;

CREATE INDEX ON user_order_stats(id);

-- Query: instant (reading pre-computed table)
SELECT * FROM user_order_stats WHERE id = 'user_123';

-- Refresh periodically:
REFRESH MATERIALIZED VIEW CONCURRENTLY user_order_stats;
-- CONCURRENTLY: allows reads during refresh (no lock)
-- Refresh every hour via pg_cron
```

---

## The Complete Scaling Architecture

```
                    [Users: millions]
                          ↓
                    [CDN / Cache]         ← static assets, edge cache
                          ↓
                    [Load Balancer]
                          ↓
              [App Servers: auto-scaled]
                /         |         \
        [Redis Cache]  [Primary DB]  [Elasticsearch]
        hit ratio:90%  writes only   full-text search
                          ↓
                  [Read Replicas × 3]    ← 80% of reads
                  [Analytics Replica]    ← reporting queries
                          ↓
               [Table Partitioning]      ← within each DB
               (time-based partitions)
                          ↓
          If still not enough:
               [Shard 1] [Shard 2] [Shard 3]
               each with own primary + replicas

Event-driven sync:
  Writes → Kafka → update Redis, Elasticsearch, analytics
```

---

## Database Scaling in Interview Answers

When asked "how would you scale this database?":

> "I'd follow the scaling ladder. First, I'd check for missing indexes and slow queries — this often solves 80% of performance problems. Then I'd add connection pooling if not present. Next, I'd add Redis caching for frequently-read data. For read scaling, I'd add read replicas and route read-only transactions there. If write volume is the bottleneck, I'd look at vertical scaling first — a 512GB RAM instance eliminates a lot of problems cheaply. Then table partitioning for large time-series tables. Only if all that's exhausted would I look at sharding — and I'd do functional decomposition first, splitting into domain databases before sharding within a domain."

---

## Key Takeaways

```
Database Scaling Ladder (in order):
  1. Query optimization + indexes   → zero cost, massive gain
  2. Connection pooling             → HikariCP, PgBouncer
  3. Caching                        → Redis, 90% cache hit rate
  4. Read replicas                  → multiply read capacity
  5. Vertical scaling               → bigger machine, often enough
  6. Table partitioning             → split large tables in same DB
  7. Sharding                       → split across multiple DBs
  8. Functional decomposition       → separate DBs per domain

CQRS:
  Separate write model (normalized, ACID)
  from read model (denormalized, fast)
  Sync via Kafka → eventual consistency

Materialized views:
  Pre-compute expensive queries
  Refresh periodically
  Instant query performance on pre-computed results

Connection pooling:
  HikariCP in app → pool_size = (cores × 2) + spindles
  PgBouncer between app and DB → thousands of app connections
  mapped to small DB connection pool

Interview: climb the ladder bottom-up
           never jump to sharding as first answer
```
