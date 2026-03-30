# Failover

**Phase 1 — Core Concepts | Topic 8 of 9**

---

## What is Failover?

Failover is the **automatic switching to a backup system when the primary system fails** — with the goal of making the failure invisible or minimally disruptive to users.

```
Normal:   Users → [Primary Server] → [Database]
                        ↓ crashes
Failover: Users → [Backup Server]  → [Database]
                   (takes over automatically)
```

The key word is **automatic**. Manual intervention during an outage is slow, error-prone, and unacceptable for high-availability systems. Good failover design means a server dies at 3am and nobody wakes up.

---

## Active-Passive Failover

One primary handles all traffic. One or more standbys sit idle, continuously receiving data from the primary but serving no traffic.

```
         ┌─────────────────────────────┐
Users →  │  Primary  │  health check   │
         │  (active) │ ←────────────── │ ← Load Balancer monitors
         └─────────────────────────────┘         │
                    ↓ fails                       │ detects failure
         ┌─────────────────────────────┐          │
         │  Standby  │ ←────────────── │ ← traffic rerouted here
         │ (passive) │                 │
         └─────────────────────────────┘
```

**Pros:**
- Simple to implement and reason about
- Standby is always in sync, ready to take over
- No conflict resolution needed — only one writer at a time

**Cons:**
- Brief downtime during switchover (seconds to minutes)
- Standby capacity is wasted during normal operation
- If standby is behind on replication when primary fails, some data may be lost

Where it's used: Database primary-replica setups, AWS RDS Multi-AZ, most traditional enterprise systems.

---

## Active-Active Failover

Multiple nodes all handle traffic simultaneously. If one fails, the others simply absorb its load with zero downtime.

```
         ┌──────────────────────────────────────┐
         │  Node A  │  Node B  │  Node C        │
Users →  │ (active) │ (active) │ (active)       │ ← Load Balancer
         └──────────────────────────────────────┘
                   Node B fails
         ┌──────────────────────────────────────┐
         │  Node A  │          │  Node C        │
Users →  │ (active) │    ✗     │ (active)       │ ← traffic redistributed
         └──────────────────────────────────────┘
```

**Pros:**
- Zero downtime during failures
- No wasted capacity — all nodes serve traffic
- Higher overall throughput

**Cons:**
- All nodes must handle writes → conflict resolution complexity
- Harder to maintain consistency across nodes
- More expensive and complex to operate

Where it's used: Stateless services (Spring Boot APIs behind load balancer), Kafka brokers, Cassandra nodes, DNS servers.

---

## Failover Components — What Actually Does the Switching?

### Load Balancer Health Checks
Load balancers continuously ping backend servers. If a server misses N consecutive health checks, traffic is rerouted away from it automatically.

```java
// Spring Boot — expose health endpoint for load balancer
// Actuator does this automatically at /actuator/health
management.endpoints.web.exposure.include=health
```

### Database Failover Agents
Tools like **MHA (MySQL)**, **Patroni (PostgreSQL)**, or **AWS RDS Multi-AZ** monitor the primary database and promote a replica to primary automatically when the primary dies.

### DNS Failover
Route 53 (AWS) monitors endpoints and updates DNS records to point to backup systems when primary fails. Slower due to DNS TTL propagation, but works across regions.

### Heartbeats
Nodes continuously send "I'm alive" signals to a monitoring system. Silence = failure. The monitoring system triggers failover when heartbeats stop.

---

## Failover Metrics That Matter

### RTO — Recovery Time Objective
How long can the system be down before it's unacceptable?

```
RTO = 0        → Active-Active required (zero downtime)
RTO = seconds  → Active-Passive with fast automated failover
RTO = minutes  → Active-Passive acceptable
RTO = hours    → Disaster recovery scenario
```

### RPO — Recovery Point Objective
How much data loss is acceptable?

```
RPO = 0        → Synchronous replication required
                 (every write confirmed on backup before ack)
RPO = seconds  → Semi-synchronous replication
RPO = minutes  → Asynchronous replication acceptable
```

```
Timeline:
─────────────────────────────────────────────────────→ time
    Last backup    Failure occurs    System restored
         │               │                │
         │←────RPO───────│                │
                          │←─────RTO──────│
```

**RTO and RPO together define your failover strategy.** Low RTO + Low RPO = expensive active-active with synchronous replication. High RTO + High RPO = simple active-passive with async replication.

---

## Split-Brain — The Danger of Failover

```
Network partition between Primary and Standby:

[Primary] ✗————✗ [Standby]

Primary thinks: "Standby is dead, I'll keep serving"
Standby thinks: "Primary is dead, I'll promote myself"

Now BOTH are accepting writes independently
→ Two sources of truth
→ Data diverges
→ When partition heals, conflict is unresolvable
```

This is called **split-brain**. It's catastrophic in databases because you end up with two different versions of reality.

### How to prevent it:

**Quorum** — a node only promotes itself to primary if it gets acknowledgment from a majority of nodes.

```
3 nodes: A, B, C
Network splits: [A, B] | [C]

[A, B] side: has 2 votes → can elect a primary ✅
[C] side:    has 1 vote  → cannot elect a primary, stays standby ✅
→ Only one primary ever exists
```

**Fencing / STONITH** (Shoot The Other Node In The Head) — when a new primary is elected, it sends a command to kill or fence the old primary before taking over.

**ZooKeeper / etcd** — external coordination services that provide distributed consensus for leader election. Kafka uses ZooKeeper (or KRaft) for exactly this purpose.

---

## Failover in Kafka

```
Broker 1 (Leader for Partition 0) crashes
    ↓
Controller detects via heartbeat timeout
    ↓
Controller elects Broker 2 (in-sync replica) as new leader
    ↓
Producers/consumers fetch new metadata
    ↓
Processing resumes — typically <30 seconds total
```

- **Consumer Failover** — if a consumer instance dies, Kafka triggers a group rebalance and redistributes partitions automatically.
- **Controller Failover** — uses ZooKeeper/KRaft for leader election, preventing split-brain.

---

## Failover vs Fallback

- **Failover** = switching TO a backup when primary fails
- **Fallback** = switching BACK to the primary after it recovers

Fallback is not always automatic — sometimes manual validation is needed before restoring the original primary to ensure it's fully caught up and healthy.

---

## Key Takeaways

```
Failover:       Automatic switch to backup when primary fails

Active-Passive: Simple, brief downtime, wasted standby capacity
Active-Active:  Zero downtime, full capacity, more complex

RTO:  How long can you be down?     → determines failover speed needed
RPO:  How much data can you lose?   → determines replication mode needed

Split-Brain:    Both primary and backup think they're primary → catastrophic
Prevention:     Quorum voting, fencing, external coordination (ZooKeeper)

Kafka:          Automatic broker/consumer/controller failover built-in
```
