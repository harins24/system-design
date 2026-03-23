# CAP Theorem

**Phase 1 — Core Concepts | Topic 7 of 9**

---

## What is the CAP Theorem?

Proposed by Eric Brewer in 2000, the CAP Theorem states:

> A distributed system can guarantee **at most 2 of these 3 properties** simultaneously:
> - **C** — Consistency
> - **A** — Availability
> - **P** — Partition Tolerance

---

## The Three Properties

### Consistency (C)
Every read receives the most recent write or an error. All nodes in the system see the same data at the same time.

```
User writes X=5 to Node 1
User reads X from Node 2
→ Must return 5, not an old value
```

> This is **not** the same as ACID consistency. CAP consistency = **linearizability** — reads always reflect the latest write.

### Availability (A)
Every request receives a non-error response — but not necessarily the most recent data. The system always responds, even if some nodes are down.

```
Node 2 is out of sync with Node 1
User reads X from Node 2
→ Returns a value (maybe stale), never an error
```

### Partition Tolerance (P)
The system continues operating even when network partitions occur — when nodes can't communicate with each other due to network failures.

```
Node 1 ←— network cut —→ Node 2
Both nodes continue operating despite being unable to talk to each other
```

---

## The Critical Insight — P Is Not Optional

Network partitions **WILL** happen in any distributed system. Networks are unreliable. Packets get dropped. Switches fail. Data centers lose connectivity. **You cannot build a distributed system and opt out of partition tolerance** — if you do, you're building a single-node system.

So the **real choice is always between C and A during a partition**:

```
When a network partition occurs, do you:

→ Sacrifice Availability  = CP system
  (refuse to respond until nodes sync up, ensuring correctness)

→ Sacrifice Consistency   = AP system
  (respond with possibly stale data, ensuring the system stays up)
```

---

## CP vs AP — The Real Decision

### CP Systems — Consistency over Availability
When a partition happens, the system refuses requests it can't guarantee are consistent.

```
Use when: correctness is non-negotiable
Examples: Banking, financial transactions, inventory systems
          (you cannot show a user they have $1000 when they have $500)

Real systems: HBase, Zookeeper, etcd, traditional RDBMS
```

### AP Systems — Availability over Consistency
When a partition happens, all nodes keep responding but may return stale or conflicting data. Consistency is restored eventually once the partition heals.

```
Use when: availability is non-negotiable, stale data is acceptable
Examples: Social media feeds, product catalog, DNS
          (okay if a user sees a post 2 seconds late)

Real systems: Cassandra, DynamoDB, CouchDB, Kafka
```

---

## Visualizing the Partition Scenario

```
Normal operation:
[Node 1] ←——sync——→ [Node 2]
User writes X=5 to Node 1 → syncs to Node 2 → all good

Network partition occurs:
[Node 1] ✗——————✗ [Node 2]
User writes X=10 to Node 1
User reads X from Node 2 → Node 2 still has X=5

NOW the system must choose:

CP choice: Node 2 says "I can't guarantee I'm up to date, returning error"
           → User gets an error (unavailable) but never wrong data

AP choice: Node 2 says "I'll return what I have: X=5"
           → User gets stale data but system stays up
```

---

## Where Kafka Fits

Kafka is primarily a **CP system** for data durability, but designed for high availability:

```
When you set acks=all:
  Producer waits for ALL in-sync replicas to acknowledge the write
  If replicas are unavailable, producer gets an error
  Guarantees no data loss — CP behavior

When you set acks=1:
  Producer only waits for the leader to acknowledge
  Higher availability, small risk of data loss if leader crashes
  AP-leaning behavior
```

`min.insync.replicas` controls this tradeoff directly — this is CAP theorem expressed as a configuration parameter.

---

## PACELC — The Extension of CAP

CAP only talks about behavior during partitions. PACELC extends it to normal operations:

```
If Partition → choose between A and C  (the CAP part)
Else (normal) → choose between L (Latency) and C (Consistency)
```

Even without failures, there's a tradeoff: lower latency by not waiting for all replicas to sync, or stronger consistency by waiting for all replicas at the cost of latency.

This is why DynamoDB lets you choose between **eventually consistent reads** (faster, cheaper) and **strongly consistent reads** (slower, more expensive) — PACELC in a real product.

---

## Real System Classification

| System | Type | Why |
|--------|------|-----|
| PostgreSQL (single node) | CA | No partition tolerance — single node |
| MySQL with replication | CP | Stops accepting writes if replica lag too high |
| Cassandra | AP | Always responds, eventual consistency |
| DynamoDB | AP (default) | Eventually consistent by default |
| HBase | CP | Stops if region server unreachable |
| Kafka (acks=all) | CP | Won't acknowledge if replicas unavailable |
| Redis (single) | CA | Single node, no partition tolerance |
| Redis Cluster | AP | Responds even during partition |
| ZooKeeper / etcd | CP | Used FOR coordination — must be consistent |

---

## How to Answer CAP in an Interview

Never just say "I'd choose CP" or "I'd choose AP" without justification:

> "CAP theorem tells us that during a network partition we must choose between consistency and availability. For this system — which handles financial transactions — I'd choose CP. A user seeing an error is recoverable. A user seeing an incorrect balance is not. I'd implement optimistic locking, retry logic on the client side, and clear error messages so users know to retry."

> "For the user feed service, I'd choose AP. Showing a post 2 seconds late is completely acceptable. I'd use eventual consistency with read repair and version vectors to resolve conflicts when the partition heals."

The key is always tying the choice back to **business consequences** of each failure mode.

---

## Key Takeaways

```
CAP:  Can only guarantee 2 of 3 — but P is mandatory in distributed systems
Real choice: C vs A during a network partition

CP:  Correct but sometimes unavailable  → finance, inventory, coordination
AP:  Always available but sometimes stale → social, catalog, DNS

Kafka: CP with acks=all, AP-leaning with acks=1
PACELC: Extends CAP to include latency vs consistency tradeoff in normal ops
```
