# Database Architectures

**Phase 4 — Database Fundamentals | Topic 9 of 9**

---

## What are Database Architectures?

Database architecture refers to how a database system is structured internally and deployed — how it stores data, how it handles concurrency, how it distributes across machines, and how it balances reads vs writes.

---

## 1. Single-Node Architecture

The simplest architecture. One machine handles everything.

```
[Client] → [Database Server]
             ├── Query Processor
             ├── Transaction Manager
             ├── Storage Engine
             └── Disk Storage
```

```
Pros: Simple, ACID trivial, no distributed complexity
Cons: SPOF, vertical scale only, limited capacity

When appropriate:
  Small to medium applications
  Development and testing
  When dataset fits on one machine with room to grow
  When complexity of distribution isn't worth it
```

---

## 2. Primary-Replica Architecture

One primary handles writes, multiple replicas handle reads.

```
         [Primary]
         /    |    \
      [R1]  [R2]  [R3]
    reads  reads  reads

Primary:
  All writes go here
  Replicates changes to replicas asynchronously
  Also handles some reads if needed

Replicas:
  Read-only copies
  Serve majority of read traffic
  Can be promoted to primary on failure
```

Most common production architecture for SQL databases. Covered in depth in Topic 5 (Data Replication).

---

## 3. Active-Active Architecture

Multiple nodes each accept both reads and writes simultaneously.

```
         [Node A] ←——→ [Node B]
              ↑              ↑
          Reads+         Reads+
          Writes         Writes

Both nodes fully operational
Each replicates changes to the other (bidirectional)
Load balancer distributes traffic to both

Used by:
  Multi-datacenter deployments
  Geographic distribution
  Zero-downtime requirement
```

### The Hard Problem: Conflict Resolution

```
User updates profile on Node A (datacenter US):
  name = "Hari Kumar" at t=1000ms

Same user updates profile on Node B (datacenter EU):
  name = "H. Kumar" at t=1001ms

Both updates replicate to each other.
Which value wins?
```

**Conflict resolution strategies:**

```
Last Write Wins (LWW):
  Higher timestamp wins → "H. Kumar"
  Simple, but clock skew can cause issues
  Used by: Cassandra, DynamoDB

Custom merge logic:
  Application defines how to merge conflicts
  Complex but most correct

CRDT (Conflict-free Replicated Data Types):
  Data structures that automatically merge correctly
  Counters: both increments apply (additive)
  Sets: union of both sets
  Register: last-write-wins per field
  Used by: distributed collaborative editing (Google Docs)
```

---

## 4. Shared-Nothing Architecture

Each node is independent — its own CPU, memory, and storage. Nodes communicate only via network.

```
[Node 1]     [Node 2]     [Node 3]
CPU/RAM/Disk CPU/RAM/Disk CPU/RAM/Disk
  (subset      (subset      (subset
   of data)     of data)     of data)

No shared disk
No shared memory
Each node completely autonomous
Scale by adding nodes
```

This is the foundation of modern distributed databases:

```
Cassandra: Shared-nothing, consistent hashing ring
           Each node owns range of hash ring
           Data automatically distributed

MongoDB sharding: Each shard is shared-nothing
                  Mongos router directs to correct shard

HBase: Shared-nothing RegionServers
       Each RegionServer owns set of regions (tablet = row range)

Kafka: Shared-nothing brokers
       Each broker owns set of partition leaders
```

```
Pros:
✅ Linear scalability — add node = add proportional capacity
✅ No shared resource bottleneck (no contention)
✅ Fault isolation — node failure affects only its data
✅ Commodity hardware — no special shared storage needed

Cons:
❌ Cross-node operations expensive (JOIN, transactions)
❌ Network is the bottleneck
❌ Rebalancing when nodes added/removed
```

---

## 5. Shared-Disk Architecture

Multiple processing nodes share the same storage layer.

```
[Node 1]  [Node 2]  [Node 3]  ← compute nodes
   \          |          /
    [Shared Storage Layer]     ← SAN or distributed storage
         (single source of truth)
```

```
Oracle RAC (Real Application Clusters):
  Multiple Oracle instances share same storage
  Each instance reads/writes same data files
  Distributed lock manager coordinates access

AWS Aurora:
  Multiple read replicas share same distributed storage
  Writes go to primary → immediately visible to all replicas
  Replicas don't need to replay WAL (data already there)
  → Much faster replica promotion and failover

Pros:
  ✅ Any node can access any data
  ✅ Easy to add read nodes
  ✅ Simpler replication (storage is already shared)
  ✅ Faster failover (no data to copy to new primary)

Cons:
  ❌ Storage layer is potential bottleneck
  ❌ Storage must be highly available
  ❌ More expensive (specialized storage)
```

---

## 6. OLTP vs OLAP Architecture

Two fundamentally different workload types require different architectures.

### OLTP — Online Transaction Processing

```
Characteristics:
  Many small, fast transactions
  Read/write individual rows
  Low latency required (< 10ms)
  High concurrency (thousands of users simultaneously)
  Normalized schema (minimize redundancy)

Workload:
  INSERT one order
  UPDATE user's balance
  SELECT user's recent 10 orders
  DELETE expired session

Database design:
  Normalized (3NF) → minimize redundancy
  Many indexes for fast point lookups
  ACID transactions critical
  Row-oriented storage

Examples: PostgreSQL, MySQL, Oracle OLTP, SQL Server
Use cases: web applications, mobile apps, e-commerce
```

### OLAP — Online Analytical Processing

```
Characteristics:
  Few large, complex queries
  Aggregate millions/billions of rows
  Higher latency acceptable (seconds to minutes)
  Low concurrency (few analysts, not thousands)
  Denormalized schema (optimize for read speed)

Workload:
  "What was total revenue by product category last quarter?"
  "Show me customer retention cohorts by signup month"
  "Which zip codes have highest order cancellation rate?"

Database design:
  Denormalized (star/snowflake schema)
  Columnar storage (read only needed columns)
  Fewer indexes, more materialized views
  Read-optimized, less write concern

Examples: Snowflake, BigQuery, Redshift, ClickHouse
Use cases: business intelligence, data warehousing, analytics
```

### Columnar Storage — Why OLAP is Different

```
Row-oriented storage (OLTP):
  Each row stored together on disk

  [id=1, name="Hari", city="Omaha", revenue=99.99]
  [id=2, name="John", city="Denver", revenue=49.99]
  [id=3, name="Jane", city="Omaha", revenue=199.99]

  "SELECT revenue FROM orders WHERE city = 'Omaha'"
  → Must read ALL columns for each row, discard non-needed
  → Reads 4 columns, uses only 2

Column-oriented storage (OLAP):
  Each column stored together on disk

  id:      [1, 2, 3]
  name:    ["Hari", "John", "Jane"]
  city:    ["Omaha", "Denver", "Omaha"]
  revenue: [99.99, 49.99, 199.99]

  "SELECT revenue FROM orders WHERE city = 'Omaha'"
  → Read city column → find matching row positions
  → Read revenue column at those positions only
  → Never reads id or name columns

  Also: columns compress extremely well (same data type, repeated values)
  city: "Omaha","Denver","Omaha" → run-length encoded → tiny
```

---

## 7. Lambda Architecture

Handles both real-time and historical analytics simultaneously.

```
                    [All Data]
                   /           \
          [Batch Layer]    [Speed Layer]
          (historical)     (real-time)
          Hadoop/Spark     Kafka + Flink
          hours/days lag   milliseconds lag
                   \           /
                  [Serving Layer]
                  (query results)
                  Merges batch + real-time

Example: Analytics for an e-commerce platform

Batch layer:
  Processes last month's orders
  Computes: revenue by category, customer cohorts
  Runs every few hours on Spark
  Results stored in serving layer

Speed layer:
  Processes orders as they happen (Kafka + Flink)
  Updates today's real-time metrics
  Low latency, approximate results

Serving layer:
  Query: "Revenue this month by category"
  = historical (batch) + today's (speed)
  Merges and returns complete answer
```

**Problem with Lambda:** Maintaining two code paths (batch + streaming) is expensive. Same logic implemented twice → bugs diverge. Complex to operate and maintain.

---

## 8. Kappa Architecture

Simplification of Lambda — one code path for everything.

```
[All Data] → [Kafka] → [Stream Processor (Flink/Kafka Streams)]
                                    ↓
                              [Serving Layer]

Historical reprocessing:
  Replay Kafka topic from beginning
  Recompute everything with current logic
  Replace serving layer when done

Advantages over Lambda:
  Single code path (no duplicate logic)
  Simpler to operate
  Kafka retention makes replay possible
```

Kappa is the preferred architecture in most modern systems:
- Kafka as the source of truth log
- Stream processing handles both real-time and historical
- Replay for reprocessing historical data
- No separate batch layer needed

---

## 9. HTAP — Hybrid Transactional/Analytical Processing

Combines OLTP and OLAP in a single system. Eliminates ETL lag.

```
Traditional approach:
  OLTP DB → [ETL pipeline, hours/days delay] → Data Warehouse → Analytics
  Problem: analytics data always stale by hours
           two systems to operate and sync

HTAP:
  Single database handles both transactional and analytical queries
  Real-time analytics on fresh data
  No ETL pipeline needed
```

```
Products:
  TiDB:        MySQL-compatible, separate row + columnar storage
  CockroachDB: growing HTAP capabilities
  SingleStore: Fast HTAP, row + columnar storage
  SQL Server:  In-Memory OLTP + columnstore indexes

  Google Spanner + BigQuery integration:
  Spanner → real-time sync → BigQuery (near-zero lag)

Trade-off:
  Neither OLTP nor OLAP is quite as fast as a dedicated system
  But "good enough" for both in a single system
  Eliminates the operational burden of two systems + ETL
```

---

## 10. Distributed SQL Architecture (NewSQL Internals)

How CockroachDB and similar databases work:

```
[SQL Layer]          Parse, plan, optimize queries
     ↓
[Transaction Layer]  Distributed ACID via 2PC + Paxos
     ↓
[Distribution Layer] Range-based partitioning + routing
     ↓
[Replication Layer]  Raft consensus per range
     ↓
[Storage Layer]      RocksDB (LSM-tree) on local disk
```

```
Key concepts:

Ranges:
  Data divided into 64MB ranges
  Each range replicated 3x across nodes (Raft group)
  Leaseholder (like primary) handles reads/writes for range

Raft per range:
  Each range has independent Raft group
  Changes require majority of Raft group to agree
  Leader elected per range independently
  → Node failure only affects its leased ranges
  → Other ranges unaffected → minimal blast radius

Distributed transactions:
  Multi-range transactions use distributed timestamp
  2PC across range leaders
  Parallel commits optimization → O(1) instead of 2 rounds
```

---

## 11. LSM-Tree Architecture

Used by LevelDB, RocksDB (Cassandra, HBase), many NoSQL databases. Optimized for write-heavy workloads.

```
Problem with B-Tree for writes:
  Update requires random disk I/O (seek to page, modify in place)
  Random I/O is slow on spinning disks
  Even SSDs: random write < sequential write

LSM-Tree (Log-Structured Merge Tree):
  All writes go to in-memory memtable first (fast, sequential)
  When full → flush to disk as immutable SSTable (sequential write!)
  SSTables periodically merged (compaction) in background
```

### Write Path

```
1. Write to WAL (crash recovery)
2. Write to memtable (in-memory sorted structure)
3. When memtable full → flush to L0 SSTable on disk
4. Background: merge SSTables into larger L1, L2 files
```

### Read Path (slower than B-Tree)

```
Check memtable → check L0 SSTables → check L1 → L2...
Bloom filter per SSTable → skip files that definitely lack key
```

```
LSM pros:
  ✅ Very fast writes (all writes sequential)
  ✅ High write throughput
  ✅ Good compression

LSM cons:
  ❌ Slower reads (multiple levels to check)
  ❌ Write amplification during compaction
  ❌ Space amplification (duplicate data during merge)

Used by: Cassandra, HBase, RocksDB, LevelDB
Good for: Write-heavy workloads, append-heavy (logs, events)
Bad for:  Read-heavy workloads with random access patterns
```

---

## Architecture Selection Framework

```
Single workload, moderate scale:
  → Single-node PostgreSQL + read replicas

High read:write ratio:
  → Primary-replica, Redis cache in front

Multi-datacenter, global users:
  → Active-active (CockroachDB, Spanner, DynamoDB Global)

Massive write throughput, time-series:
  → LSM-tree based (Cassandra, RocksDB)

Analytical queries on large datasets:
  → Columnar OLAP (Snowflake, BigQuery, ClickHouse)

Both OLTP and analytics:
  → HTAP (TiDB) or separate systems with Kafka sync

Real-time streaming analytics:
  → Kappa architecture (Kafka + Flink + serving layer)

Historical + real-time (but team is small):
  → Kappa over Lambda (single codebase)
```

---

## Key Takeaways

```
Database architectures — know the tradeoffs:

Single-node:      Simple, SPOF, limited scale
Primary-replica:  Most common, read scale, async lag
Active-active:    Write scale, conflict resolution needed
Shared-nothing:   Linear scale, distributed complexity
Shared-disk:      Shared truth, storage bottleneck risk

OLTP:   Many small fast transactions, row storage, normalized
OLAP:   Few large slow queries, columnar storage, denormalized
        Columnar = read only needed columns + great compression
HTAP:   Both in one, eliminates ETL lag, growing adoption

Lambda: Batch + stream, two code paths, complex
Kappa:  Stream only, Kafka replay for historical,
        simpler → modern preference

LSM-Tree: Write-optimized storage
  Sequential writes → fast throughput
  Multiple levels → slower reads
  Bloom filters compensate for read performance
  Used by Cassandra, RocksDB, HBase

Distributed SQL (CockroachDB):
  Raft per range, distributed transactions
  SQL semantics at horizontal scale
```
