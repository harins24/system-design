# Consistency Patterns

**Phase 6 — Distributed Systems + Architecture | Topic 5**

---

## What is Consistency?

Consistency defines what value a system returns when you read data, especially after a write, and especially across distributed nodes.

```
Simple case (single node):
  Write: x = 5
  Read:  x → always returns 5
  Trivially consistent

Distributed case (3 nodes):
  Write x = 5 → goes to Node 1
  Immediately read x → hits Node 2
  Node 2 hasn't received update yet
  → Returns x = 3 (old value)
  Is this correct? Depends on your consistency model.
```

Distributed systems offer a spectrum of consistency guarantees. Understanding this spectrum is essential for senior engineering interviews.

---

## The Consistency Spectrum

```
Strongest                                              Weakest
──────────────────────────────────────────────────────────────────
Linearizability → Sequential → Causal → Eventual → No guarantee
      ↑                                                    ↑
   Safest                                             Fastest
   Most expensive                               Most available
```

---

## 1. Linearizability (Strict Consistency)

The strongest consistency model. Every operation appears to take effect instantaneously at some point between its invocation and completion. The system behaves as if there is a single copy of the data.

```
Timeline:
  t=1: Client A writes x = 5
  t=2: Write completes
  t=3: Client B reads x → MUST return 5
  t=4: Client C reads x → MUST return 5

  Any read that starts AFTER a write completes
  MUST see the new value.

  Even if clients are in different datacenters.
  Even if they hit different nodes.
```

Real example — distributed counter:

```
3 clients increment counter simultaneously
Counter was 10

Linearizable: exactly one of these orderings is valid
  A increments → B increments → C increments → final: 13
  or any other valid sequential ordering
  Final value is always 13

Non-linearizable:
  All three read 10, all increment → final: 11
  Lost updates
```

Implementation:

```
Requires: all reads and writes go through single leader
          OR distributed consensus (Paxos, Raft)

Cost: every write must be confirmed by majority
      high latency (network round trips)
      reduced availability (leader must be reachable)

Used by: etcd, ZooKeeper (cluster configuration, leader election)
         Google Spanner (external consistency)
         Single-master databases (leader reads only)

When to use:
  Distributed locking (two nodes must not acquire same lock)
  Leader election (only one node can be leader)
  Unique ID generation (no duplicates across nodes)
  Financial account balances (one source of truth)
```

---

## 2. Sequential Consistency

All operations appear to execute in some sequential order. All nodes see the same order, but the order doesn't have to match real-time.

```
Client A writes: x=1, then x=2
Client B writes: y=1, then y=2

Sequential consistency guarantees:
  All nodes see x=1 before x=2 (A's order preserved)
  All nodes see y=1 before y=2 (B's order preserved)

BUT: which comes first globally, A's writes or B's writes?
  Could be: x=1, x=2, y=1, y=2
  OR:       y=1, y=2, x=1, x=2
  OR:       x=1, y=1, x=2, y=2

  ALL are valid — no real-time constraint

NOT guaranteed:
  Client C writes x=5 at t=10
  Client D reads x at t=11 (after the write in real time)
  D might still see old value — that's OK under sequential consistency
```

```
Weaker than linearizability (no real-time constraint)
Stronger than causal (all nodes agree on global order)

Used by: some distributed databases
         Multi-processor memory models (CPU instruction reordering)

Rarely the primary model in distributed systems
(either need linearizability or can accept something weaker)
```

---

## 3. Causal Consistency

Operations that are causally related are seen in the same order by all nodes. Concurrent operations (no causal relationship) may be seen in different orders by different nodes.

```
Causal relationship:
  A writes x = 5
  B reads x (sees 5)
  B writes y = x + 1 (=6)

  B's write of y causally depends on A's write of x
  Any node that sees y=6 MUST also see x=5

  C reads y=6 → MUST be able to read x=5 (not an older value)

No causal relationship (concurrent):
  A writes x = 5
  B writes y = 10
  (neither knows about the other)

  Node 1 might see: x=5 first, then y=10
  Node 2 might see: y=10 first, then x=5
  Both valid under causal consistency
```

Implementation — Vector Clocks:

```
Each event tagged with vector clock:
  {nodeA: 3, nodeB: 2, nodeC: 1}

  Captures causal history:
  "This event happened after nodeA's 3rd event,
   nodeB's 2nd event, nodeC's 1st event"

  Causally related: one vector clock dominates another
  Concurrent: neither dominates → any order valid

Used by:
  Amazon DynamoDB (eventually consistent with causal metadata)
  Apache Cassandra (lightweight transactions)
  MongoDB (causal sessions)
  Git (DAG of commits = causal history)
```

```java
// MongoDB causal consistency session
try (ClientSession session = client.startSession()) {
    session.startTransaction();

    // All reads and writes in session respect causal order
    // Even if reads hit replicas, they see writes from this session

    usersCollection.insertOne(session, newUser);

    // This read will see the user just inserted
    // even if it hits a different node
    User user = usersCollection.find(session,
        Filters.eq("_id", newUser.getId())).first();

    session.commitTransaction();
}
```

**When to use:**

```
✅ Social media: user sees their own posts immediately
✅ Collaborative editing: your changes visible to you before others
✅ E-commerce: user sees their cart updates immediately
✅ Any user-facing app where "read-your-writes" is critical
   without needing full linearizability
```

---

## 4. Read-Your-Writes Consistency

A client always sees its own writes. After a write, all subsequent reads by the SAME client return the updated value.

```
Single user guarantee:
  Client writes: profile.photo = new_photo.jpg
  Client immediately reads profile
  → MUST see new photo (their own write)

  Other clients: may still see old photo (not guaranteed)

Implementation:
  Option 1: Route user's reads to primary (for X seconds after write)
  Option 2: Track user's latest write timestamp
             Read replica must be caught up to that timestamp
             If not → route to primary
  Option 3: Read from primary always (simple, expensive)
  Option 4: Session tokens with vector clocks
```

```java
@Service
public class ReadYourWritesUserService {

    private final UserRepository primaryRepo;
    private final UserRepository replicaRepo;
    private final RedisTemplate<String, Long> recentWriteTracker;

    public User updateProfile(String userId, ProfileUpdate update) {
        User user = primaryRepo.findById(userId);
        user.applyUpdate(update);
        primaryRepo.save(user);

        // Track when this user last wrote
        recentWriteTracker.opsForValue().set(
            "user-write:" + userId,
            System.currentTimeMillis(),
            Duration.ofSeconds(10)
        );

        return user;
    }

    public User getProfile(String userId) {
        // Did this user write recently?
        Long lastWrite = recentWriteTracker.opsForValue()
            .get("user-write:" + userId);

        if (lastWrite != null) {
            // Route to primary — ensure they see their write
            return primaryRepo.findById(userId);
        }

        // Safe to read from replica
        return replicaRepo.findById(userId);
    }
}
```

---

## 5. Monotonic Reads

A client will never read older data after reading newer data. Once you see a value, you'll never see an older value for the same data.

```
Without monotonic reads:
  Client reads user count: 1,000,000  (from Replica A, mostly up to date)
  Client reads user count again: 999,987  (from Replica B, more lagged)

  Count went backward! This breaks user trust.

With monotonic reads:
  Once you've seen user count = 1,000,000
  Future reads must return ≥ 1,000,000 (or the same data/newer)

Implementation:
  Sticky sessions: same client → same replica always
  Version tokens: replica must have version ≥ last seen version
```

---

## 6. Monotonic Writes

Writes from a single client are applied in the order they were issued.

```
Client issues writes:
  Write 1: counter = 1
  Write 2: counter = 2
  Write 3: counter = 3

  All nodes must see these in order 1, 2, 3
  Never: counter = 3, then counter = 2 (reversal)

Usually guaranteed by:
  Session-scoped write ordering
  Versioned writes (optimistic locking)
  Single-leader per client session
```

---

## 7. Eventual Consistency

If no new writes are made, all replicas will eventually converge to the same value. No guarantee on when, but it will happen.

```
t=0:  Primary receives write: x = 5
t=1:  Client reads from Replica 1: x = 3 (old, not replicated yet)
t=2:  Replication propagates: Replica 1 gets x = 5
t=3:  Client reads from Replica 1: x = 5 (now consistent)

No bound on "eventually" — could be milliseconds or hours
In practice: usually milliseconds to seconds

Strong eventual consistency (SEC):
  All nodes that receive same set of updates
  compute same final value
  (even if they receive updates in different orders)
  CRDTs achieve this
```

**Conflict resolution in eventual consistency:**

```
Two clients simultaneously update same record on different nodes:
  Node A: user.name = "Hari"    at t=100
  Node B: user.name = "H. Kumar" at t=101

  Which wins?

Last Write Wins (LWW):
  Compare timestamps → t=101 wins → "H. Kumar"
  Simple but: clock skew can cause wrong winner

Version vectors:
  Track which node made which version
  Detect conflict when vectors diverge
  Application or user resolves conflict
  Used by: Dynamo-style databases, Riak

CRDTs (Conflict-free Replicated Data Types):
  Data structures that merge automatically without conflict

  Counter CRDT: each node has own increment counter
  Merge: sum all node counters
  Result: always correct total, never conflicts

  Used by: Redis CRDT, Riak, Apple Notes sync
```

---

## 8. CRDT — Conflict-free Replicated Data Types

Data structures that automatically merge concurrent updates without conflicts:

```
G-Counter (grow-only counter):
  Each node has slot in array:
  Node1: [5, 0, 0]
  Node2: [5, 3, 0]
  Node3: [5, 0, 2]

  Merge: take max per slot
  [max(5,5,5), max(0,3,0), max(0,0,2)] = [5, 3, 2]
  Total = 5 + 3 + 2 = 10

  Works regardless of replication order
  All nodes converge to same total

PN-Counter (increment and decrement):
  Two G-Counters: positive (increments) + negative (decrements)
  Value = P - N

OR-Set (add/remove set):
  Each add operation tagged with unique ID
  Remove operation removes specific tag
  If same element added and removed concurrently:
  → "Add wins" semantics (most common)

  Used by: shopping carts (concurrent add/remove)
           collaborative document editing
```

---

## Consistency vs Latency: The Practical Tradeoff

```
Stronger consistency → more coordination → higher latency

Linearizable read (Raft/Paxos):
  Must contact majority of nodes
  Wait for responses
  Typical: 5-10ms additional latency per read

Causal consistency (session token):
  Must wait for replica to catch up to version
  Typical: 1-5ms additional latency

Eventual consistency:
  Just read local replica
  Typical: <1ms additional latency

At scale, even 1ms per request matters:
  1 million requests/sec × 5ms = 5,000 CPU-seconds/sec of wait time
  vs
  1 million requests/sec × 1ms = 1,000 CPU-seconds/sec
```

---

## Choosing the Right Consistency Model

```
Linearizability:
  Distributed locks, leader election, counters
  Financial balances (but banks often use weaker models!)
  Any "there can only be one" scenarios

Causal:
  Social media (see your own posts)
  Collaborative tools (see causally related updates)
  Most user-facing applications where read-your-writes matters

Read-your-writes:
  Profile updates
  Any operation where user expects immediate feedback
  E-commerce (see item added to cart)

Eventual:
  Product catalog (stale by seconds is fine)
  Social feed (approximate counts are acceptable)
  Recommendations (eventually updated is fine)
  DNS (intentionally designed as eventual)
  CDN cache (TTL-based)

In practice — layered approach:
  User's own data:      read-your-writes (they expect to see changes)
  Shared counters:      causal or eventual with compensation
  Inventory stock:      linearizable for final purchase, eventual for display
  Product info:         eventual (minutes stale is fine)
  Distributed config:   linearizable (all nodes see same config)
```

---

## Consistency in Your Stack

```
PostgreSQL:
  Default: read committed (not linearizable)
  Serializable isolation: closest to linearizable for single DB
  Logical replication → replicas are eventually consistent
  Synchronous replication → replica can be linearizable for reads

Kafka:
  Within partition: linearizable (strict ordering)
  Across partitions: no ordering guarantee
  Consumer group: each message processed once (sequential per partition)

Redis:
  Single instance: linearizable (single-threaded, no races)
  Redis Cluster: eventual consistency across shards
  Redis Sentinel: brief inconsistency during failover

Cassandra:
  Default: eventual consistency
  QUORUM reads/writes: strong consistency (similar to linearizable)
  LOCAL_QUORUM: consistency within datacenter

  Consistency level = how many nodes must respond:
  ONE:    fastest, eventual
  QUORUM: (n/2)+1 nodes, strong
  ALL:    all nodes, strongest (rarely used)
```

---

## Key Takeaways

```
Consistency spectrum (strongest to weakest):
  Linearizability → Sequential → Causal → Eventual

Linearizability:
  Single global order, real-time constraint
  Every read sees latest write
  Most expensive, highest latency
  Use: locks, leader election, unique constraints

Causal consistency:
  Causally related ops seen in order everywhere
  Concurrent ops: any order valid
  Vector clocks track causality
  Use: social media, collaborative editing

Read-your-writes:
  Client always sees its own writes
  Other clients may lag
  Implementation: route writes to primary,
                  track write timestamps per user
  Use: profile updates, any user-visible write

Monotonic reads:
  Never read older data after reading newer
  Sticky sessions or version tokens
  Use: any metric/count that shouldn't go backward

Eventual consistency:
  All nodes converge eventually
  No real-time guarantee
  Fastest, most available
  Use: product catalog, social counts, CDN

CRDTs:
  Conflict-free merge without coordination
  Counters, sets, maps that auto-resolve
  Use: distributed counters, shopping carts, offline sync

Practical rule:
  Identify consistency requirement per data type
  Don't over-engineer with linearizability everywhere
  Read-your-writes + monotonic reads covers most UX needs
  Reserve linearizability for correctness-critical operations
```
