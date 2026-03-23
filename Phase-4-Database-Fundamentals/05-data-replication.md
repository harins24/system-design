# Data Replication

**Phase 4 — Database Fundamentals | Topic 5 of 9**

---

## What is Data Replication?

Data replication is the process of copying data from one database node to one or more other nodes and keeping those copies synchronized over time.

```
Without replication:
  [Primary DB] ← single node
  → Goes down = complete outage
  → Single machine handles all reads AND writes
  → No geographic distribution
  → Backup? Hope the tape works.

With replication:
  [Primary DB] → [Replica 1]
               → [Replica 2]
               → [Replica 3]

  → Primary fails → replica promoted → system continues
  → Reads distributed across replicas → primary handles writes only
  → Replicas in different regions → low latency globally
  → Any replica can serve as backup
```

Replication serves three purposes: **availability, scalability, and durability**.

---

## Replication Topologies

### Single-Leader (Master-Slave / Primary-Replica)

One node accepts all writes. Other nodes replicate from it and serve reads.

```
Writes:
  Client → [Primary] → data written here
                    → changes streamed to replicas

Reads:
  Client → [Replica 1] → serves read queries
  Client → [Replica 2] → serves read queries
  Client → [Replica 3] → serves read queries

                    [Primary]
                    /    |    \
             [R1]  [R2]  [R3]
```

Most common topology. Used by PostgreSQL, MySQL, MongoDB.

```
Write scalability:  ❌ Single primary = single write bottleneck
Read scalability:   ✅ Add replicas = more read capacity
Failover:           ✅ Promote replica if primary fails
Consistency:        ✅ Strong on primary, eventual on replicas
```

---

### Multi-Leader (Master-Master)

Multiple nodes accept writes. Each leader replicates to the others.

```
         [Leader 1] ←——→ [Leader 2]
              ↑                ↑
          Writes           Writes

Both leaders accept writes
Each replicates to the other
Conflict resolution required when same data written to both

Use cases:
  Multi-datacenter: each datacenter has own leader
  Offline-capable clients: each device is a leader
  Collaborative editing: each user's session is a leader

Write scalability:  ✅ Multiple write points
Availability:       ✅ Other leaders work if one fails
Conflict handling:  ❌ Hard — same row updated on two leaders
Complexity:         ❌ Significantly higher
```

---

### Leaderless (Dynamo-Style)

Any node can accept writes. Reads and writes go to multiple nodes. No single coordinator.

```
Write: send to N nodes, wait for W acknowledgments
Read:  send to N nodes, wait for R responses

N=3, W=2, R=2:
  Write to 3 nodes, 2 must confirm → write succeeds
  Read from 3 nodes, return when 2 respond → use latest version

Quorum condition: W + R > N
  2 + 2 > 3 ✅ → guaranteed to see latest write

Used by: Cassandra, DynamoDB, Riak

Write scalability:  ✅ Any node accepts writes
Availability:       ✅ Tolerates node failures (quorum based)
Consistency:        ❌ Eventual by default
Conflict handling:  ❌ Last-write-wins or application resolves
```

---

## Synchronous vs Asynchronous Replication

### Synchronous Replication

Primary waits for replica to confirm write before acknowledging to client.

```
Client → Primary: "Write X=5"
Primary → Replica: "Write X=5"
Replica: writes X=5 → "Confirmed"
Primary: "OK, committed" → Client: "Success"

Timeline:
t=0ms:  Client sends write
t=5ms:  Primary writes
t=10ms: Replica writes (network round trip)
t=10ms: Client gets success response

✅ Zero replication lag — replica always up to date
✅ No data loss if primary fails — replica has everything
✅ Can immediately read from replica after write

❌ Write latency = primary write time + replica round trip
❌ If replica is slow or down → primary blocks
   → One slow replica degrades ALL writes
❌ Impractical beyond 1-2 synchronous replicas
```

---

### Asynchronous Replication

Primary acknowledges write immediately. Replication happens in background.

```
Client → Primary: "Write X=5"
Primary: writes X=5 → Client: "Success" ← immediately
Primary → Replica: "Write X=5" (background, async)
Replica: writes X=5 (some time later)

✅ Write latency = primary write time only (fast)
✅ Replica being slow/down doesn't affect writes
✅ Can have many replicas without performance impact

❌ Replication lag: replica may be seconds/minutes behind
❌ Data loss on primary failure:
   writes after last replication not yet on replica
❌ Reading from replica may return stale data
```

---

### Semi-Synchronous (Hybrid)

Wait for at least one replica, rest are async.

```
Primary → Replica 1 (sync): wait for confirmation
        → Replica 2 (async): fire and forget
        → Replica 3 (async): fire and forget

Client gets success when Primary + Replica 1 both written
At least 2 copies before acknowledging = durability guarantee
Replica 1 failure → fall back to async until fixed

Used by: MySQL semi-sync replication
```

This is the practical sweet spot — durability guarantee without sacrificing much latency.

### PostgreSQL Synchronous Commit Options

```sql
-- Synchronous: wait for WAL flush on primary only (default)
synchronous_commit = local

-- Wait for at least one standby to receive WAL
synchronous_commit = remote_write

-- Wait for at least one standby to flush WAL to disk
synchronous_commit = on

-- Wait for at least one standby to apply WAL (fully caught up)
synchronous_commit = remote_apply

-- Async: return before any WAL flush (fastest, risk of loss)
synchronous_commit = off
```

---

## Replication Lag — The Practical Problem

In async replication, replicas lag behind the primary. This causes subtle bugs.

### Read-Your-Own-Writes Problem

```
t=0: User updates profile photo
     → writes to primary

t=1: User refreshes page
     → reads from replica (load balanced)
     → replica hasn't received update yet (10ms lag)
     → user sees old photo
     → "Did my update fail?"

Solution: Read-your-own-writes consistency
  After a write, read from primary for that user for 30 seconds
  Or: route that user's reads to primary for a short window
  Or: track write timestamp, redirect to primary if replica is behind
```

### Monotonic Read Problem

```
User makes two reads in sequence:
  Read 1 → Replica A (200ms behind) → sees posts 1,2,3,4,5
  Read 2 → Replica B (5 seconds behind) → sees posts 1,2,3

Data appears to go BACKWARD — post 4 and 5 vanished!

Solution: Monotonic reads
  Sticky user → same replica for all reads in a session
  Or: include read timestamp, replica only serves if caught up
```

### Causal Consistency Problem

```
User A posts a question
User B sees the question, posts an answer
User C loads the page:
  Reads from Replica 1: doesn't see question yet (lag)
  Reads answer from Replica 2 (less lag)
  → Sees answer without question — causality violated

Solution: Causal consistency
  Track causal dependencies between writes
  Don't return answer until question is also visible
  Complex to implement — Lamport clocks, vector clocks
```

---

## Handling Primary Failure — Failover

### Automatic Failover

```
Pattern — Sentinel/Monitor:
  Sentinel nodes monitor primary health (heartbeat)
  Primary misses N heartbeats → declared dead
  Sentinels vote on which replica to promote (quorum)
  Winning replica promoted to primary
  Other replicas reconfigure to follow new primary
  Application DNS updated to point to new primary

PostgreSQL: Patroni
  → Consensus via etcd/ZooKeeper/Consul
  → Automatic promotion
  → Used by large PostgreSQL installations

Redis: Redis Sentinel
  → Monitors Redis primary
  → Automatic failover
  → Client-transparent via Sentinel API

AWS RDS Multi-AZ:
  → Synchronous standby in different AZ
  → Automatic failover in 60-120 seconds
  → DNS name stays same → application reconnects automatically
```

### The Failover Risks

```
Split-brain during failover:
  Primary pauses (network issue, not dead)
  Sentinel declares primary dead → promotes replica
  Network recovers → old primary comes back up
  Now TWO primaries accepting writes → data divergence

  Solution: STONITH (Shoot The Other Node In The Head)
  Before promoting replica, send shutdown command to old primary
  Fence the old primary from storage
  Only then promote new primary

Data loss window:
  Async replication: writes after last replication lost
  Semi-sync: at most 1 replica's worth of lag lost
  Sync replication: zero data loss

  RTO (how long until system recovers):
    Manual:    minutes to hours
    Automatic: 30-120 seconds typical

  RPO (how much data lost):
    Async:    seconds to minutes of writes
    Semi-sync: near zero
    Sync:     zero
```

---

## Replication in Kafka

```
Topic: train-events, 3 partitions, replication factor 3

Partition 0:
  Leader: Broker 1
  Followers: Broker 2, Broker 3

Producer sends message to Partition 0:
  → goes to Broker 1 (leader)
  → Broker 1 writes to local log
  → Broker 2 fetches from Broker 1 → writes local copy
  → Broker 3 fetches from Broker 1 → writes local copy

acks=all (most durable):
  Broker 1 waits until Broker 2 AND Broker 3 both have message
  → Returns success to producer
  → Zero data loss even if Broker 1 immediately dies

ISR (In-Sync Replicas):
  Set of replicas caught up to leader within replica.lag.time.max.ms
  acks=all only waits for ISR members
  If Broker 3 falls behind → removed from ISR
  Broker 1 only waits for Broker 2 (still in ISR)

min.insync.replicas = 2:
  At least 2 replicas must be in ISR for writes to succeed
  If only 1 replica in ISR → producer gets error
  Protects against silent data loss
```

---

## Read Replicas for Scale

```
Typical OLTP read/write ratio: 80% reads, 20% writes

Without replicas:
  Primary handles all 100% → bottleneck

With 3 read replicas:
  Primary handles 20% (writes only)
  Each replica handles ~27% of reads
  → Primary load reduced 5x on reads
```

```java
// Spring Boot — route reads to replicas:
@Configuration
public class DataSourceConfig {

    @Bean
    @Primary
    public DataSource routingDataSource(
            DataSource primary,
            List<DataSource> replicas) {

        AbstractRoutingDataSource routing = new AbstractRoutingDataSource() {
            @Override
            protected Object determineCurrentLookupKey() {
                return TransactionSynchronizationManager
                    .isCurrentTransactionReadOnly()
                    ? "replica"
                    : "primary";
            }
        };

        Map<Object, Object> sources = new HashMap<>();
        sources.put("primary", primary);
        sources.put("replica", loadBalancedReplica(replicas));
        routing.setTargetDataSources(sources);
        return routing;
    }
}

// Service layer — annotate read-only transactions
@Transactional(readOnly = true)  // ← routes to replica
public List<Order> getOrderHistory(String userId) {
    return orderRepository.findByUserId(userId);
}

@Transactional  // ← routes to primary
public Order createOrder(CreateOrderRequest request) {
    return orderRepository.save(new Order(request));
}
```

---

## Global Replication — Multi-Region

```
Users in different continents → route to nearest region:

  US-East (Primary writes)
  ↓ async replication (cross-region)
  EU-West (Read replica + local writes for EU users)
  ↓ async replication
  AP-Southeast (Read replica + local writes for Asian users)

Challenges:
  Cross-region latency: US→EU ~100ms, US→Asia ~150ms
  Conflict resolution: if same record updated in both regions

Solutions:
  Route writes for each user to their "home" region
  Read from local replica (low latency)
  Async replicate to other regions (eventual consistency)
  Conflict resolution: last-write-wins, CRDTs, or application logic

Used by: CockroachDB, Google Spanner (TrueTime), DynamoDB Global Tables
```

---

## Key Takeaways

```
Replication: Copy data to multiple nodes for
             availability, scalability, durability

Topologies:
  Single-leader: one writer, many readers — most common
  Multi-leader:  multiple writers, conflict resolution needed
  Leaderless:    any node writes, quorum consensus (Cassandra)

Sync vs Async:
  Synchronous:  zero lag, zero data loss, high write latency
  Asynchronous: some lag, small data loss risk, fast writes
  Semi-sync:    wait for 1 replica, rest async — sweet spot

Replication lag problems:
  Read-your-writes: user sees stale data after own write
  Monotonic reads:  data appears to go backward
  Causal violation: effect visible before cause
  Solution: sticky reads, primary reads after writes

Failover:
  Manual: slow, error-prone
  Automatic: Patroni (PostgreSQL), Sentinel (Redis), RDS Multi-AZ
  Risk: split-brain → STONITH solves it
  RTO: 30-120s automatic; RPO: 0 (sync) to seconds (async)

Kafka replication:
  ISR (In-Sync Replicas) set
  acks=all: wait for all ISR → zero data loss
  min.insync.replicas: minimum replicas required for write

Read replicas for scale:
  Route @Transactional(readOnly=true) to replica
  Route writes to primary
  Linear read scalability by adding replicas
```
