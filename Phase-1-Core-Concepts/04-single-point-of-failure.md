# Single Point of Failure (SPOF)

**Phase 1 — Core Concepts | Topic 4 of 9**

---

## What is a SPOF?

A Single Point of Failure is any component whose failure causes the entire system to stop working.

```
Users → [Load Balancer] → [App Server] → [Database]
                                              ↑
                                         Only one DB.
                                     DB goes down = full outage.
                                     This is a SPOF.
```

The dangerous thing about SPOFs is they're often invisible until disaster strikes. Everything looks fine in normal operations — the SPOF only reveals itself at the worst possible moment.

---

## SPOFs Are Everywhere — You Have to Hunt for Them

Most engineers think about the obvious ones (database, server). The real skill is spotting the hidden ones:

| Component | How it becomes a SPOF |
|-----------|----------------------|
| Database | Single instance, no replica |
| Load Balancer | Only one load balancer in front of your servers |
| Message Broker | Single Kafka broker, no replication |
| DNS Server | All traffic routed through one DNS resolver |
| Network Switch | Single switch connecting all servers in a rack |
| Third-party API | Your payment flow depends entirely on Stripe being up |
| Deployment Pipeline | Only one engineer has production deploy access |
| Configuration Server | All microservices pull config from one server on startup |
| Human | Only one person knows how the system works |

That last one is real. **"Bus factor"** — how many people need to get hit by a bus before the team can't function? If the answer is one, that person is a SPOF.

---

## How to Eliminate SPOFs

### Active-Passive Redundancy
One primary handles all traffic. A standby sits idle and takes over if the primary fails. Simple, but there's a brief downtime during failover and the standby is wasted capacity during normal operation.

```
[Primary DB] ──writes──→ [Replica DB]
     ↑                        ↑
 handles traffic          takes over on failure
```

### Active-Active Redundancy
Multiple instances all handle traffic simultaneously. If one fails, the others absorb its load with zero downtime. More complex — requires careful handling of shared state.

```
[DB Node 1] ←→ [DB Node 2] ←→ [DB Node 3]
     ↑               ↑               ↑
all three handle reads and writes simultaneously
```

### Geographic Redundancy
Deploy across multiple data centers or cloud regions. Protects against data center level failures — power outages, natural disasters, network cuts.

---

## The SPOF Detection Process

In an interview, when asked to design a system, walk through every layer and ask: **"What happens if THIS component fails?"**

- → If the answer is "everything breaks" → it's a SPOF → add redundancy
- → If the answer is "this part degrades but core function continues" → acceptable

Layer by layer:
- **DNS** — use multiple DNS providers or anycast
- **Load Balancer** — run multiple LBs, use DNS failover between them
- **Application Servers** — always multiple instances, never single
- **Cache** — Redis Sentinel or Redis Cluster, not a single Redis node
- **Database** — primary + replicas, automatic failover (like AWS RDS Multi-AZ)
- **Message Broker** — Kafka with replication factor ≥ 3
- **Network** — multiple network paths, multiple ISPs for critical systems

---

## SPOF in Your Kafka Architecture

| Component | Risk |
|-----------|------|
| **Kafka Controller** | In older Kafka, one broker acted as controller. KRaft mode (Kafka 3.x) distributes this responsibility. |
| **ZooKeeper** | Classic SPOF in older Kafka setups. Fix: ZooKeeper ensemble of 3 or 5 nodes. |
| **Schema Registry** | If you use Avro/Protobuf and Schema Registry goes down, producers/consumers fail. Fix: run multiple Schema Registry instances. |
| **Consumer Group Coordinator** | Automatically reassigned if broker fails — handled automatically. |

---

## The Cost vs Risk Tradeoff

Eliminating every SPOF costs money. The right conversation in a system design interview:

> "We could add redundancy here, but it doubles our infrastructure cost. Given that this is an internal reporting service with 99.9% SLA, a brief failover window is acceptable. For the payment service, we need active-active because downtime directly means lost revenue."

Always tie the decision back to **business impact of failure**.

---

## Key Takeaway for Interviews

When designing any system, say this proactively:

> "Let me walk through each layer and identify potential single points of failure. For each one, I'll evaluate whether active-passive or active-active redundancy is appropriate based on the cost of downtime at that layer."

Interviewers love when you proactively identify SPOFs without being asked — it signals senior-level thinking.
