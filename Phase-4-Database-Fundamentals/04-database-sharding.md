# Database Sharding

**Phase 4 — Database Fundamentals | Topic 4 of 9**

---

## What is Sharding?

Sharding is the practice of splitting a large database into smaller, independent pieces called **shards**, each holding a subset of the data, distributed across multiple machines.

```
Without sharding:
All 500 million users on one database server
→ Single machine storage limit hit
→ Single machine CPU/RAM bottleneck
→ Every query competes for same resources
→ SPOF

With sharding:
Shard 0: users 0-124M    → Server A
Shard 1: users 125-249M  → Server B
Shard 2: users 250-374M  → Server C
Shard 3: users 375-499M  → Server D

Each server handles 25% of data and traffic
Linear scalability — add shard = add capacity
```

Sharding is **horizontal partitioning** — you're splitting rows across machines. This is different from vertical partitioning (splitting columns) or replication (copying data).

---

## When Do You Need Sharding?

Sharding introduces significant complexity. Don't reach for it early.

**Exhaust these options first (in order):**
```
1. Query optimization + indexes
   → Often fixes performance without schema changes

2. Read replicas
   → Offload reads to replicas, writes to primary
   → Handles read-heavy workloads easily

3. Caching (Redis)
   → Reduce DB load dramatically for repeated reads

4. Vertical scaling
   → Bigger machine: more RAM, faster CPU, faster NVMe
   → AWS db.r6g.16xlarge: 512GB RAM, 64 vCPUs
   → Gets you very far before needing sharding

5. Sharding
   → When single machine genuinely can't handle the load
   → When dataset exceeds single machine storage

Signal you need sharding:
  → Dataset > 1-5TB (single machine struggling)
  → Write throughput exceeds single master capacity
  → Single table has billions of rows and is bottleneck
  → Even with optimizations, P99 latency unacceptable
```

---

## Sharding Strategies

### 1. Range-Based Sharding

Partition data by ranges of the sharding key.

```
Shard by user_id ranges:

Shard 0: user_id 0        – 9,999,999
Shard 1: user_id 10M      – 19,999,999
Shard 2: user_id 20M      – 29,999,999
Shard 3: user_id 30M      – 39,999,999

Routing:
  shard = user_id / 10,000,000
  user_id 15,234,567 → shard 1
```

```
Pros:
✅ Simple routing logic
✅ Range scans efficient (all data for a range on one shard)
✅ Easy to split shards when they grow too large
✅ Good for time-series data (shard by date range)

Cons:
❌ Hotspot problem: new users always go to latest shard
❌ If most queries are for recent users: one shard handles 90% of traffic
```

Range sharding works well for time-series data:
```
Shard Jan 2026: all events from January
Shard Feb 2026: all events from February
Query last 7 days → hits 1-2 shards max
Archive old shards → move to cold storage
```

---

### 2. Hash-Based Sharding

Apply a hash function to the sharding key to determine the shard.

```
shard = hash(user_id) % num_shards

user_id "user_123" → hash = 4829374 → 4829374 % 4 = 2 → Shard 2
user_id "user_456" → hash = 9182736 → 9182736 % 4 = 1 → Shard 1
user_id "user_789" → hash = 2736481 → 2736481 % 4 = 3 → Shard 3
```

```
Pros:
✅ Even distribution — hash distributes uniformly
✅ No hotspots — traffic spread across all shards
✅ Simple routing formula

Cons:
❌ Adding/removing shards remaps almost all keys
   (This is why consistent hashing exists — solves this problem)
❌ Range queries span multiple shards:
   SELECT * FROM orders WHERE created_at > '2026-01-01'
   → Must query ALL shards and merge results
❌ No data locality — related records may be on different shards
```

---

### 3. Consistent Hashing (Hash Ring)

Shards and keys placed on a hash ring. Key routed to first shard clockwise from its position.

```
Adding shard: only keys between new shard and previous shard move
Removing shard: only that shard's keys move to next shard

Used by: Cassandra, DynamoDB, Redis Cluster
```

---

### 4. Directory-Based Sharding

A lookup service (directory) maps each key to its shard.

```
Shard directory (stored in cache):
user_123 → Shard 2
user_456 → Shard 0
user_789 → Shard 3

Routing:
  Client asks directory: "Where is user_123?"
  Directory: "Shard 2"
  Client queries Shard 2
```

```
Pros:
✅ Flexible — any mapping logic, easily changed
✅ Easy to move individual keys between shards
✅ Can handle skewed data (put heavy users on dedicated shard)

Cons:
❌ Directory is a SPOF (must be highly available)
❌ Every query hits directory first → latency + bottleneck
❌ Directory must be kept consistent → complex
```

---

### 5. Geographic Sharding

Shard by user location — keep user's data close to them.

```
Shard NA:  users from North America → AWS us-east-1
Shard EU:  users from Europe        → AWS eu-west-1
Shard AP:  users from Asia Pacific  → AWS ap-southeast-1

Benefits:
  ✅ Data residency compliance (GDPR: EU data stays in EU)
  ✅ Low latency (user's data in nearby region)
  ✅ Natural traffic distribution

Challenges:
  ❌ User travels between regions
  ❌ Cross-region queries (user interacts with foreign user)
```

---

## Choosing a Sharding Key — The Most Important Decision

The sharding key determines everything. A bad choice cannot easily be undone.

```
Good sharding key properties:
  ✅ High cardinality — many distinct values
  ✅ Even distribution — no hotspots
  ✅ Aligned with query patterns — most queries include it
  ✅ Immutable — value doesn't change after creation

Common sharding key examples:

User-centric app:
  Shard by user_id
  → All user's data on same shard
  → Most queries are "get data for user X" → single shard
  → Even distribution (millions of users)

E-commerce:
  Shard orders by user_id (not order_id)
  → "Get orders for user X" → single shard hit
  → If sharded by order_id, user's orders scatter across all shards

Multi-tenant SaaS:
  Shard by tenant_id
  → All data for a tenant on one shard
  → Tenant-scoped queries never cross shards
  → Large tenants may need dedicated shards

Time-series:
  Shard by time range (month/year)
  → Recent data on hot shard
  → Old data naturally cold
  → Easy archival (drop old shards)

What to AVOID as sharding key:
  ❌ Auto-increment ID — all inserts go to same shard initially
  ❌ Low cardinality (status, country) — hotspots guaranteed
  ❌ Frequently updated fields — key change = row must move shards
  ❌ Column not in most queries — cross-shard queries everywhere
```

---

## The Problems Sharding Creates

### Cross-Shard Queries

```
"Find all orders placed in the last 24 hours"

Unsharded: SELECT * FROM orders WHERE created_at > now()-24h
→ One query, one database, fast

Sharded by user_id:
→ Must query ALL shards in parallel
→ Merge and sort results in application
→ Expensive, slow, complex

Solutions:
  Denormalization: maintain a separate unsharded reporting DB
                   fed by Kafka → all events → analytics DB

  Scatter-gather: query all shards in parallel, merge in app layer
                  acceptable if shards are few and fast

  CQRS: separate read model optimized for cross-shard queries
        write to sharded DB, replicate to analytics store
```

### Cross-Shard Transactions

```
Transfer $100 from user_123 (Shard 2) to user_456 (Shard 0):

Both users on different shards → can't use single ACID transaction

Options:
  2PC (Two-Phase Commit): complex, blocking, performance hit
  Saga pattern: compensating transactions, eventual consistency
  Design around it: keep related data on same shard
                   (best solution — avoid cross-shard transactions)
```

### Rebalancing

```
Start with 4 shards, need to add 5th:

Hash sharding: hash % 4 → hash % 5
               Almost every key remaps to different shard
               Massive data migration needed

Consistent hashing: only ~20% of keys move (1/N fraction)
                    Much better

Directory-based: update directory entries
                 Migrate data gradually
                 Flexible but needs careful coordination

Range sharding: split one large shard into two
                Only that shard's data moves
                Simpler migration
```

### Hot Shards (Shard Skew)

```
Celebrity problem:
  Twitter shards by user_id
  @elonmusk has 150M followers
  Every @elonmusk tweet → massive traffic to one shard

Solutions:
  Shard popular users differently (dedicated shard)
  Cache heavily queried data from hot shards in Redis
  Read replicas specifically for hot shards
  Application-level caching to absorb hot key traffic
```

---

## Sharding in Practice

### Application-Level Sharding

Most common. Your application code decides which shard.

```java
@Service
public class ShardRouter {

    private final List<DataSource> shards;
    private final int SHARD_COUNT = 4;

    public DataSource getShardForUser(String userId) {
        // Consistent hashing or simple modulo
        int shardIndex = Math.abs(userId.hashCode()) % SHARD_COUNT;
        return shards.get(shardIndex);
    }

    public DataSource getShardForOrder(String orderId, String userId) {
        // Route by user_id to keep user's orders together
        return getShardForUser(userId);
    }
}

@Repository
public class OrderRepository {

    private final ShardRouter shardRouter;

    public Order findByIdAndUserId(String orderId, String userId) {
        DataSource shard = shardRouter.getShardForUser(userId);
        return jdbcTemplate(shard).queryForObject(
            "SELECT * FROM orders WHERE id = ? AND user_id = ?",
            ORDER_MAPPER, orderId, userId);
    }

    // Cross-shard query — scatter-gather pattern
    public List<Order> findRecentOrders(Instant since) {
        return shards.parallelStream()
            .flatMap(shard ->
                jdbcTemplate(shard)
                    .query("SELECT * FROM orders WHERE created_at > ?",
                           ORDER_MAPPER, since)
                    .stream())
            .sorted(Comparator.comparing(Order::getCreatedAt).reversed())
            .collect(toList());
    }
}
```

### Middleware Sharding (Proxy)

A sharding proxy sits between application and databases. Application talks to proxy like a single database.

```
Application → [Vitess / ProxySQL / Citus] → [Shard 0] [Shard 1] [Shard 2]

Vitess (YouTube's sharding layer, now open source):
  → MySQL sharding at massive scale
  → Query routing, connection pooling, rebalancing
  → Application doesn't know about sharding
  → Used by YouTube, Slack, GitHub

Citus (PostgreSQL sharding):
  → Distributed PostgreSQL
  → SQL queries automatically distributed
  → Transparent to application
```

---

## Sharding vs Partitioning

Often confused:

```
Partitioning: Split data within a single database instance
              Different partitions on same machine (or local storage)
              Database manages it internally

              PostgreSQL table partitioning:
              CREATE TABLE orders_2026_01 PARTITION OF orders
              FOR VALUES FROM ('2026-01-01') TO ('2026-02-01');

Sharding:     Split data across multiple database instances
              Different shards on different machines
              Application or proxy manages routing

Partitioning is a stepping stone before sharding.
Start with partitioning, shard when you outgrow one machine.
```

---

## Key Takeaways

```
Sharding: Split large database horizontally across machines
          Each shard holds subset of rows
          Linear scalability at the cost of complexity

Do sharding last:
  Indexes → replicas → caching → vertical scale → THEN shard

Sharding strategies:
  Range:       simple, hotspot risk, good for time-series
  Hash:        even distribution, bad for ranges, rebalancing hard
  Consistent:  like hash but minimal rebalancing (Cassandra, Redis)
  Directory:   flexible, SPOF risk, lookup overhead
  Geographic:  compliance, latency, data residency

Sharding key rules:
  High cardinality + even distribution + aligned with queries + immutable
  Bad key = hotspots + cross-shard queries = worse than before

Problems sharding creates:
  Cross-shard queries  → scatter-gather or separate analytics DB
  Cross-shard txns     → Saga pattern or design to avoid
  Rebalancing          → consistent hashing minimizes movement
  Hot shards           → caching, dedicated shards, read replicas

Implementation:
  Application-level: code routes to correct shard
  Proxy-based: Vitess, Citus — transparent to application

Interview answer when asked about scaling DB:
  1. Add indexes
  2. Add read replicas
  3. Add caching
  4. Vertical scale
  5. Partition within single DB
  6. Shard across multiple DBs
  Never jump to sharding first
```
