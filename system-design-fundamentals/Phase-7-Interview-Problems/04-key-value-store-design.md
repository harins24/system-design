# Designing a Key-Value Store

**Phase 7 — Tradeoffs + Interview Problems | Topic 4 of 8**

---

## Overview

Designing a key-value store tests your understanding of distributed systems fundamentals at the deepest level — storage engines, consistent hashing, replication, consistency tradeoffs, and fault tolerance. This is a senior/staff-level question.

---

## Phase 1: Requirements

### Functional Requirements

```
Core operations:
  put(key, value)   → store a key-value pair
  get(key)          → retrieve value for key
  delete(key)       → remove a key-value pair

Questions to ask:
  "What's the expected key and value size?"
  "Do we need TTL (expiration)?"
  "Do we need range queries or just point lookups?"
  "Is this for internal use or external API?"
  "Do we need transactions across multiple keys?"

For this walkthrough:
  Key: up to 1KB, Value: up to 10MB
  TTL: yes, keys can expire
  Point lookups only (no range queries → simpler design)
  Highly available distributed system
  No multi-key transactions
```

### Non-Functional Requirements

```
Scale:
  10M keys total, 100K reads/sec, 10K writes/sec

Latency:
  get() < 10ms P99, put() < 50ms P99

Availability:
  99.99% — available for reads even during node failures

Consistency:
  Tunable: strong or eventual depending on use case
  Default: eventual consistency (availability prioritized)

Durability:
  No data loss on node failure
  Replication factor: 3

Partition tolerance:
  System continues during network partitions (AP in CAP)
```

---

## Phase 2: Capacity Estimation

```
Storage:
  10M keys × (1KB key + 1MB value avg) = 10TB data
  Replication factor 3 → 30TB total raw storage
  → Needs multiple nodes, can't fit on one machine

Node sizing (assume 1TB RAM nodes):
  10TB data / 1TB per node = 10 nodes minimum
  With replication: 10 primary + 20 replicas = 30 nodes
  Headroom (use 60%): 17 primary nodes → ~27 nodes cluster

QPS per node:
  100K reads / 17 nodes = ~6,000 reads/node → very manageable

Bandwidth:
  100K reads × 1MB avg = 100 GB/sec outbound
  → Need CDN or limit value size to keep bandwidth reasonable
```

---

## Phase 3: High-Level Design

```
Client API:
  PUT  /keys/{key}     body: {value, ttlSeconds?}
  GET  /keys/{key}     response: {value, expiresAt?}
  DELETE /keys/{key}

System components:

[Client]
   ↓
[Coordinator Node]           ← routes requests, doesn't store data
   ↓ consistent hashing
[Storage Nodes]              ← actual key-value storage
   [Node 1] [Node 2] ... [Node N]

Each storage node:
   [Request Handler]         ← receives requests
   [Replication Manager]     ← replicates to other nodes
   [Storage Engine]          ← writes to disk
   [Membership Service]      ← gossip protocol for node discovery

Data flow:
  Put(key, value):
    Coordinator hashes key → determines primary + N-1 replica nodes
    Sends write to all responsible nodes
    Returns success when W nodes confirm (W = write quorum)

  Get(key):
    Coordinator hashes key → determines responsible nodes
    Sends read to R nodes (R = read quorum)
    Returns value when R nodes respond
    Resolves conflicts with version vector
```

---

## Phase 4: Deep Dives

### Deep Dive 1: Consistent Hashing

```
Naive modulo hashing:
  node = hash(key) % num_nodes
  Problem: add one node → almost all keys remap (90%)
  Massive data migration, thundering herd

Consistent hashing:
  Hash space: 0 to 2^32 - 1 (arranged as a ring)
  Nodes placed at positions on ring
  Key routes to first node clockwise from hash(key)

  Add node: only keys between new node and its predecessor move
  Remove node: only that node's keys move to next node
  ~1/N keys affected (not all)

Virtual nodes (vnodes):
  Each physical node has 150+ virtual positions on ring
  Better distribution (avoids hot spots from uneven placement)

  Node A (powerful): 200 vnodes → handles ~44%
  Node B (normal):   150 vnodes → handles ~33%
  Node C (small):    100 vnodes → handles ~22%
```

```java
public class ConsistentHashRing {

    private final TreeMap<Long, String> ring = new TreeMap<>();
    private final int vnodes;

    public void addNode(String nodeId) {
        for (int i = 0; i < vnodes; i++) {
            long hash = hash(nodeId + "-vnode-" + i);
            ring.put(hash, nodeId);
        }
    }

    public String getNode(String key) {
        if (ring.isEmpty()) return null;
        long hash = hash(key);
        Map.Entry<Long, String> entry = ring.ceilingEntry(hash);
        if (entry == null) entry = ring.firstEntry(); // wrap around
        return entry.getValue();
    }

    // Get N nodes for replication (next N clockwise)
    public List<String> getReplicaNodes(String key, int n) {
        List<String> nodes = new ArrayList<>();
        Set<String> seen = new HashSet<>();
        long hash = hash(key);

        NavigableMap<Long, String> tailMap = ring.tailMap(hash);
        for (String node : Iterables.concat(tailMap.values(), ring.values())) {
            if (seen.add(node)) {
                nodes.add(node);
                if (nodes.size() == n) break;
            }
        }
        return nodes;
    }
}
```

---

### Deep Dive 2: Replication

```
Replication factor N = 3 (common choice)

Quorum replication (best for distributed KV):
  W = write quorum (how many nodes must confirm write)
  R = read quorum (how many nodes must respond to read)
  N = total replicas

  Consistency guarantee: if W + R > N, reads always see latest write

  Common configurations:
  N=3, W=2, R=2 → W+R=4>3 → strong consistency
  N=3, W=1, R=1 → W+R=2<3 → eventual consistency (fast)
  N=3, W=3, R=1 → strong consistency, slow writes
  N=3, W=1, R=3 → strong consistency, slow reads

  DynamoDB/Cassandra default: W=1, R=1 (eventual, fast)
  Tunable: application chooses per request

Quorum overlap:
  write quorum    read quorum
  [N1, N2]  ∩   [N2, N3]  = {N2} (overlap!)
  N2 has the latest write → read always sees it
```

---

### Deep Dive 3: Conflict Resolution

```
With eventual consistency and network partitions,
two nodes can receive conflicting writes for same key.

Scenario:
  t=1: Client A writes key="x", value="hello" → Node 1
  Network partition occurs
  t=2: Client B writes key="x", value="world" → Node 2
  Partition heals → conflict!

Option 1: Last Write Wins (LWW)
  Higher timestamp wins: "world" (t=2) > "hello" (t=1)
  ✅ Simple
  ❌ Clock skew: different servers have different times
  ❌ Silently discards writes

Option 2: Version Vectors (Vector Clocks)
  Each value tagged with vector clock {node → version}

  write("x", "hello") from Node 1: version = {N1:1}
  write("x", "world") from Node 2: version = {N2:1}

  On merge: {N1:1} and {N2:1} are concurrent → CONFLICT detected

  Resolution options:
  a) Return both to client, client resolves (Amazon Cart approach)
  b) Server-side: LWW, merge strategies
  c) CRDT: automatic conflict-free merge

Option 3: CRDTs
  Data types that merge without conflict
  Counter: merge = sum of all increments
  Set: merge = union
  Only works for specific data types
```

---

### Deep Dive 4: Storage Engine

```
Option 1: B-Tree (PostgreSQL, MySQL)
  O(log N) read and write, good for range queries
  Problem: random writes → random disk seeks (slow on HDD)

Option 2: LSM-Tree (Cassandra, LevelDB, RocksDB) — Recommended
  Write-optimized: all writes go to in-memory Memtable
  Periodically flushed to disk as sorted SSTable files
  Background compaction merges SSTables

  Excellent write throughput (sequential disk writes)
  Reads: may need to check multiple SSTables + Bloom filter

LSM-Tree write path:
  1. Write to WAL (Write-Ahead Log) → durability
  2. Write to Memtable (in-memory sorted map)
  3. When Memtable full → flush to SSTable on disk

LSM-Tree read path:
  1. Check Memtable (most recent writes)
  2. Check Bloom filter (is key possibly in this SSTable?)
  3. Check SSTables from newest to oldest
  4. Return first match found

  Bloom filter prevents expensive disk reads for missing keys
```

```java
public class Memtable {
    private final TreeMap<String, MemtableEntry> data = new TreeMap<>();
    private long sizeBytes = 0;
    private static final long FLUSH_THRESHOLD = 64 * 1024 * 1024; // 64MB

    public void put(String key, byte[] value, long ttl) {
        MemtableEntry entry = new MemtableEntry(value, ttl, System.currentTimeMillis());
        MemtableEntry old = data.put(key, entry);
        sizeBytes += entry.size() - (old != null ? old.size() : 0);
    }

    public Optional<byte[]> get(String key) {
        MemtableEntry entry = data.get(key);
        if (entry == null) return Optional.empty();
        if (entry.isExpired()) {
            data.remove(key); // lazy expiration
            return Optional.empty();
        }
        return Optional.of(entry.getValue());
    }

    public boolean shouldFlush() {
        return sizeBytes >= FLUSH_THRESHOLD;
    }
}
```

---

### Deep Dive 5: Failure Detection (Gossip Protocol)

```
Each node periodically tells neighbors about cluster health:
  Node A: "B is alive, C is alive, D is alive"
  Node B: "A is alive, C is down (no heartbeat 3 cycles)"

  Rumors spread through cluster in O(log N) time
  No single point of failure (no central health checker)
  Used by: Cassandra, DynamoDB, Consul

Implementation:
  Every 1 second: pick 3 random neighbors
  Send own membership list
  Receive their lists → merge → update any newer entries
  Node dead if no heartbeat for 10+ seconds

Hinted Handoff (during node outage):
  Node N3 fails
  Writes meant for N3 → stored on N4 with hint "forward to N3"
  N3 recovers → N4 replays hinted writes to N3
  N3 catches up, N4 removes hints
```

---

### Deep Dive 6: TTL Implementation

```
Two approaches:

Lazy expiration:
  Check TTL on every read
  If expired → delete + return not found
  ✅ Zero background overhead
  ❌ Expired keys remain in memory until accessed

Active expiration:
  Background thread: scan random subset of keys every 100ms
  Delete any that are expired
  ✅ Memory reclaimed promptly
  ❌ CPU overhead

Production: combine both
  Read: lazy check → immediate eviction
  Background: periodic scan → clear orphaned keys
```

---

## Phase 5: Tradeoffs

```
1. Eventual vs strong consistency:
   Default: W=1, R=1 → eventual, maximum availability
   Configurable: clients choose W and R per request
   Accepted for most KV use cases

2. Conflict resolution via LWW:
   Simple, predictable, but can silently lose writes
   Alternative: vector clocks + client merge (more complex)

3. LSM-Tree storage:
   Great write throughput, read amplification possible
   Bloom filters mitigate read cost for missing keys
   Alternative: B-Tree if read-heavy

4. Gossip protocol:
   Eventual failure detection (seconds, not milliseconds)
   False positives possible (slow network ≠ dead node)

5. Memory vs disk:
   All data on disk (LSM) → handles 10TB easily
   Hot data cached in memory → serve popular keys <1ms
```

---

## Key Takeaways

```
Designing a KV store covers the full distributed systems stack:

Components:
  Coordinator: routes requests, no state
  Storage nodes: actual data + replication
  Membership service: gossip-based failure detection

Consistent Hashing:
  Keys distributed across ring
  Virtual nodes for even distribution
  Add/remove nodes: only 1/N keys move

Replication:
  Factor N=3 for durability
  Quorum: W+R>N for consistency
  N=3,W=2,R=2: strong; N=3,W=1,R=1: eventual

Conflict Resolution:
  LWW: simple, clock skew risk
  Vector clocks: detect conflicts, need resolution
  CRDTs: automatic merge (limited to specific types)

Storage Engine:
  LSM-Tree: write-optimized (Memtable → SSTable)
  WAL: durability before memory flush
  Bloom filter: avoid disk reads for missing keys
  Compaction: merge SSTables, reclaim space

Failure Detection:
  Gossip protocol: O(log N) propagation
  No single point of failure
  Hinted handoff: buffer writes during node outage

TTL:
  Lazy expiration on read + active background scan

Inspired by: DynamoDB (consistent hashing, quorum)
             Cassandra (gossip, LSM, tunable consistency)
             Redis (in-memory, single-node simplicity)
```
