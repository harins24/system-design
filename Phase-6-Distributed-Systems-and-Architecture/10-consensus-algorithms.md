# Consensus Algorithms

**Phase 6 — Distributed Systems + Architecture | Topic 10**

---

## What is Distributed Consensus?

Consensus is the process by which multiple nodes in a distributed system agree on a single value or decision, even in the presence of failures.

```
Why is this hard?

In a single process:
  value = 5;  // trivial, one memory location

In a distributed system:
  Node 1: "The leader should be Node 1"
  Node 2: "The leader should be Node 2"
  Node 3: "The leader should be Node 1"

  Network partition occurs:
  Node 3 can't see Node 2
  Node 3 thinks Node 2 is dead
  Node 3 and Node 1 agree: elect Node 1 as leader

  Meanwhile:
  Node 2 thinks Node 1 is dead
  Node 2 elects itself as leader

  Now TWO leaders accept writes → data diverges → catastrophe

Consensus algorithms prevent this:
  Ensure exactly ONE value agreed upon
  Works even if some nodes fail or messages are delayed
  Never returns incorrect result
```

---

## The Consensus Problem

Formally, consensus requires three properties:

```
Termination:  All non-faulty nodes eventually decide on a value
              (the system makes progress, doesn't hang forever)

Agreement:    All non-faulty nodes decide on the SAME value
              (no two nodes reach different conclusions)

Validity:     The decided value must be a value proposed by
              some node (not an arbitrary or invented value)

AND must tolerate:
  Node failures:    Up to f nodes can fail (crash)
  Network delays:   Messages can be arbitrarily delayed
  Message loss:     Messages can be lost (detected by timeout)

What consensus CANNOT tolerate (by FLP impossibility theorem):
  Byzantine failures (nodes that lie or send corrupted data)
  → Requires separate Byzantine Fault Tolerant (BFT) protocols
```

---

## Paxos — The Original Consensus Algorithm

Invented by Leslie Lamport (1989). Foundational but notoriously difficult to understand.

### Roles

```
Proposer:  Proposes values for consensus
           Any node can be a proposer

Acceptor:  Votes to accept or reject proposals
           Quorum (majority) of acceptors must agree

Learner:   Learns the chosen value
           Doesn't participate in voting

In practice: one node plays multiple roles
```

### Paxos Phase 1 — Prepare

```
Proposer wants to propose value V:

1. Proposer picks unique proposal number N (higher than any seen)
2. Sends PREPARE(N) to all acceptors

Each acceptor receiving PREPARE(N):
  If N > highest prepare seen:
    Promise to not accept proposals < N
    Return highest accepted proposal (if any)
  Else:
    Reject (already promised to a higher proposal)

Proposer receives majority responses:
  Collects any previously accepted values
  If any: must use highest-numbered accepted value (not its own V)
  If none: free to use its own value V
```

### Paxos Phase 2 — Accept

```
Proposer sends ACCEPT(N, V) to all acceptors:

Each acceptor receiving ACCEPT(N, V):
  If N >= highest prepare promised:
    Accept the proposal, store (N, V)
    Notify learners
  Else:
    Reject (promised not to accept N)

Proposer:
  Receives majority accepts → Value V is chosen!

  Receives reject → Another proposer with higher N exists
                    Retry with even higher N
```

### Why Paxos Is Hard

```
Multi-Paxos (practical version):
  Basic Paxos runs 2 rounds for every value
  Multi-Paxos elects a leader first
  Leader runs Phase 2 for all values (Phase 1 done once)

  Implementation challenges:
  → Leader failure mid-round
  → Competing proposers cause livelock
  → Reconfiguration (adding/removing nodes)
  → Log compaction
  → Catching up lagging nodes

Lamport himself said:
  "The dirty little secret of the Paxos algorithm
   is that it is not easy to understand."

Most real systems use Raft instead.
```

---

## Raft — The Understandable Consensus Algorithm

Designed by Diego Ongaro (2014) explicitly for understandability. Same guarantees as Paxos, much clearer structure.

### Raft Basics

```
One elected LEADER handles all writes
Followers replicate from leader
Candidate: node trying to become leader

Normal operation:
  Client → Leader (all writes go here)
  Leader → replicates to followers → majority confirms → commits

Leader failure:
  Followers detect timeout
  One starts election
  Gets majority votes → new leader
  Continues from where old leader left off
```

### Term — The Logical Clock

```
Time divided into terms (logical, not real-time):
  Term 1: Node A is leader
  Term 2: Node A fails → election → Node B elected leader
  Term 3: Node B fails → election → Node C elected leader

Each term has at most one leader
Nodes always follow the highest-term leader they see
Stale leader (low term number) immediately steps down
```

### Leader Election

```
Initial state: all nodes are Followers
Each Follower has election timeout (random 150-300ms)

Normal operation:
  Leader sends heartbeats every 50ms to all Followers
  Followers reset election timeout on heartbeat

Leader failure:
  Leader stops sending heartbeats
  One follower's timeout expires first (randomized)
  Follower → Candidate
  Increments its term
  Votes for itself
  Sends RequestVote(term, candidateId, lastLogIndex, lastLogTerm) to all

Voting rules:
  Vote YES if:
    Candidate's term ≥ my term
    AND haven't voted in this term yet
    AND candidate's log is at least as up-to-date as mine
  Vote NO otherwise

Candidate receives majority votes → becomes Leader
  Sends heartbeat immediately to stop other elections

Why randomized timeouts?
  Node A timeout: 150ms
  Node B timeout: 275ms
  Node C timeout: 220ms

  A times out first → becomes candidate → wins election
  B and C receive heartbeat before their timeout → stay followers

  Prevents all nodes timing out simultaneously
  One clear winner in most cases

Split vote (two candidates simultaneously):
  A gets 1 vote, B gets 1 vote, C votes for neither (already voted)
  Timeout → new election → new random timeouts → one wins
```

### Log Replication

```
Client sends command to Leader:
  "SET x = 5"

Leader:
  1. Appends to local log (uncommitted)
     [index=1, term=2, command="SET x=5"]

  2. Sends AppendEntries to all Followers:
     AppendEntries(term=2, entry=[index=1, "SET x=5"], commitIndex=0)

  3. Follower receives, appends to local log (uncommitted)
     Returns success

  4. Leader receives majority success
     Commits entry (advances commitIndex)
     Applies to state machine: x = 5
     Returns success to client

  5. Next AppendEntries tells Followers new commitIndex
     Followers commit and apply to their state machines

At-least majority nodes have every committed entry
Quorum of 3/5 or 2/3 = data survives any minority failure
```

### Log Consistency

```
After leader election, new leader may have:
  Missing entries (was partitioned)
  Extra uncommitted entries (from previous leadership)

Raft forces followers to match leader's log:
  Leader sends AppendEntries with previous log entry info
  Follower: "Does my log match at this position?"
    Yes → Accept new entries
    No  → Tell leader to go back and resync from earlier position

  Leader finds the divergence point
  Overwrites follower's log from that point with leader's entries

Safety guarantee:
  Once an entry is committed (majority replicated),
  it will NEVER be overwritten
  Any new leader will have all committed entries
  (won't be elected if log is behind)
```

### Java Implementation (Conceptual)

```java
// Simplified Raft node
public class RaftNode {

    private RaftState state = FOLLOWER;
    private int currentTerm = 0;
    private String votedFor = null;
    private List<LogEntry> log = new ArrayList<>();
    private int commitIndex = 0;
    private int lastApplied = 0;

    // Leader state
    private Map<String, Integer> nextIndex;   // per follower: next log index to send
    private Map<String, Integer> matchIndex;  // per follower: highest replicated index

    // Follower: reset timeout on any AppendEntries from current leader
    public synchronized AppendEntriesResponse appendEntries(
            AppendEntriesRequest request) {

        // Reject old term
        if (request.getTerm() < currentTerm) {
            return AppendEntriesResponse.failure(currentTerm);
        }

        // Update term, become follower if we were candidate/leader
        if (request.getTerm() > currentTerm) {
            currentTerm = request.getTerm();
            state = FOLLOWER;
            votedFor = null;
        }

        resetElectionTimeout(); // stay as follower

        // Check log consistency at prevLogIndex/prevLogTerm
        if (!logMatchesAt(request.getPrevLogIndex(), request.getPrevLogTerm())) {
            return AppendEntriesResponse.failure(currentTerm);
        }

        // Append new entries
        appendToLog(request.getEntries());

        // Update commit index
        if (request.getLeaderCommit() > commitIndex) {
            commitIndex = Math.min(request.getLeaderCommit(), log.size() - 1);
            applyCommitted();
        }

        return AppendEntriesResponse.success(currentTerm);
    }

    // Candidate: request votes
    public synchronized VoteResponse requestVote(VoteRequest request) {

        if (request.getTerm() < currentTerm) {
            return VoteResponse.deny(currentTerm);
        }

        if (request.getTerm() > currentTerm) {
            currentTerm = request.getTerm();
            state = FOLLOWER;
            votedFor = null;
        }

        boolean grantVote =
            (votedFor == null || votedFor.equals(request.getCandidateId()))
            && candidateLogIsUpToDate(request);

        if (grantVote) {
            votedFor = request.getCandidateId();
        }

        return grantVote
            ? VoteResponse.grant(currentTerm)
            : VoteResponse.deny(currentTerm);
    }
}
```

---

## Where Raft Is Used

```
etcd (Kubernetes backing store):
  All Kubernetes cluster state stored in etcd
  Pod definitions, service configs, secrets
  Must be strongly consistent (no stale cluster state)
  Uses Raft for consensus across etcd cluster

CockroachDB:
  Each "range" (partition of data) has its own Raft group
  3-5 nodes per Raft group
  Writes require Raft consensus within the group
  → Distributed ACID transactions

TiKV (TiDB storage layer):
  Same as CockroachDB approach
  Each region = independent Raft group

HashiCorp Consul:
  Service discovery + distributed KV store
  Leader election via Raft
  Config changes require Raft consensus

Apache Kafka (KRaft mode):
  Replaces ZooKeeper with Raft
  Kafka 3.0+: built-in consensus without ZooKeeper
  Controller election via Raft

ZooKeeper (ZAB protocol):
  ZAB (ZooKeeper Atomic Broadcast) is similar to Raft
  Used by: older Kafka, HBase, Storm coordination
```

---

## Raft vs Paxos

```
Property         Raft              Paxos
────────────────────────────────────────────────
Understandability High (designed for it) Low (notoriously hard)
Strong leader     Yes               No (any node proposes)
Leader election   Explicit phase    Implicit in proposal phase
Log structure     Strongly typed    Abstract
Reconfiguration   Joint consensus   Complex (add-one-at-a-time)
Implementations   Many (etcd, etc.) Few (mostly academic)
Real-world use    Dominant          Some (Spanner, some DBs)

Both provide: same safety and liveness guarantees
Raft: easier to implement correctly, better documented
Paxos: original, some implementations claim higher performance
```

---

## Leader Election in Practice — Beyond Raft

Not every system needs full Raft. Sometimes you just need leader election:

```
ZooKeeper-based leader election (older pattern):
  All nodes try to create ephemeral ZNode: /service/leader
  ZooKeeper: only one can succeed (atomic create)
  Creator = leader, others watch for node deletion

  Leader dies → ephemeral node deleted → watchers notified
  All try to create again → one wins → new leader

Redis-based leader election (SETNX + TTL):
  SET leader node-1 NX EX 10
  NX: only set if key doesn't exist
  EX 10: expires in 10 seconds

  Leader renews every 5 seconds:
  SET leader node-1 XX EX 10
  XX: only set if key exists (renewal, not takeover)

  Leader crashes → TTL expires → key gone
  Another node runs SET NX → becomes leader

  Risk: network partition + clock skew = brief dual leadership
  Use: low-stakes leader election (not financial transactions)

Kubernetes leader election:
  Uses ConfigMap/Lease objects + optimistic locking
  Kubernetes API server (backed by etcd) provides atomic updates
  Client-go leaderelection package
```

---

## Byzantine Fault Tolerance (BFT)

Raft assumes **crash-fault tolerance**: nodes can fail by stopping, but not by sending incorrect data. BFT handles nodes that actively lie.

```
Crash fault: "Node 3 is down, no responses"
             Easy: majority quorum ignores it

Byzantine fault: "Node 3 is sending false data"
                 "Node 3 tells Node 1 the value is 5"
                 "Node 3 tells Node 2 the value is 7"
                 Hard: can't detect lies without more replicas

BFT requirement: tolerates f Byzantine nodes
                 requires 3f+1 total nodes
                 (vs Raft: tolerates f crashes, requires 2f+1)

To tolerate 1 Byzantine node: need 4 nodes (3×1+1)
To tolerate 2 Byzantine nodes: need 7 nodes

PBFT (Practical Byzantine Fault Tolerance):
  3-phase protocol: pre-prepare, prepare, commit
  Expensive: O(n²) message complexity
  Not used in most systems (too costly)

Used by: blockchain (Bitcoin, Ethereum)
         Some safety-critical systems

Not used by: typical enterprise distributed systems
             (assume trusted nodes, no Byzantine behavior needed)
```

---

## Quorum — The Core Concept

All consensus algorithms rely on quorums:

```
Quorum = majority = floor(N/2) + 1

3-node cluster: quorum = 2
5-node cluster: quorum = 3
7-node cluster: quorum = 4

Why majority?
  Any two quorums overlap by at least one node
  That overlapping node has the latest committed value
  New leader can always find the latest state

  3 nodes, quorum = 2:
  Write quorum: {A, B} both confirm
  Read quorum:  {B, C} both respond
  B is in both quorums → B has latest value

  Without majority quorum:
  Write {A}: only 1 confirms
  Read  {B}: different node
  B never saw the write → stale read → split brain possible

Why odd numbers?
  N=3: can tolerate 1 failure, quorum=2
  N=4: can tolerate 1 failure, quorum=3 (same tolerance as N=3)
  N=5: can tolerate 2 failures, quorum=3

  N=4 gives no benefit over N=3 but costs more
  Always use odd numbers: 3, 5, 7
```

---

## Key Takeaways

```
Consensus: multiple nodes agree on one value despite failures

Why needed:
  Leader election (who handles writes?)
  Distributed locking (only one holder)
  Config coordination (all nodes see same config)
  Replication (was write committed on majority?)

Paxos:
  Original algorithm (Lamport 1989)
  Two phases: Prepare (promise) + Accept (commit)
  Correct but notoriously hard to implement
  Multi-Paxos: leader elected for efficiency

Raft:
  Designed for understandability (Ongaro 2014)
  Same guarantees as Paxos, clearer structure
  Three roles: Leader, Follower, Candidate
  Leader handles all writes, replicates to followers
  Leader election via randomized timeouts
  Log replication: append + replicate + commit on majority
  Used by: etcd, CockroachDB, TiKV, Consul, Kafka (KRaft)

Quorum = majority = floor(N/2) + 1
  Any two quorums overlap → latest value always findable
  Use odd numbers: 3, 5, 7 nodes

Byzantine Fault Tolerance:
  Handles lying/malicious nodes (not just crashes)
  Requires 3f+1 nodes for f failures (expensive)
  Used by blockchain, not typical enterprise systems

Real-world usage:
  etcd → Kubernetes cluster state
  Raft groups → CockroachDB per-range consensus
  KRaft → Kafka metadata management
  ZooKeeper → Legacy coordination (Kafka pre-3.0)
```
