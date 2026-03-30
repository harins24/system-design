# Consistent Hashing

**Phase 1 — Core Concepts | Topic 6 of 9**

---

## The Problem It Solves

Before understanding consistent hashing, understand why naive hashing fails.

Suppose you have 3 servers and distribute requests using simple modulo hashing:

```
server = hash(key) % 3

key="user_123" → hash=456 → 456 % 3 = 0 → Server 0
key="user_456" → hash=789 → 789 % 3 = 1 → Server 1
key="user_789" → hash=999 → 999 % 3 = 2 → Server 2
```

Works perfectly. Now you add a 4th server:

```
server = hash(key) % 4   ← denominator changed!

key="user_123" → hash=456 → 456 % 4 = 0 → Server 0  ✅ same
key="user_456" → hash=789 → 789 % 4 = 1 → Server 1  ✅ same
key="user_789" → hash=999 → 999 % 4 = 3 → Server 3  ❌ moved!
```

Almost every key remaps to a different server. In a caching system, this means a **cache miss storm** — suddenly all traffic hits the database because cached data is on the "wrong" server. At scale, this is catastrophic.

---

## The Core Idea — The Hash Ring

Instead of mapping keys to servers directly, consistent hashing maps both keys and servers onto a **circular ring** (0 to 2³² - 1).

```
          0
      ┌───────┐
  270 │       │ 90
      │  Ring │
  180 │       │
      └───────┘

Servers placed on ring by hashing their ID:
  Server A → position 90
  Server B → position 180
  Server C → position 270
```

To find which server handles a key: hash the key, find its position on the ring, walk clockwise until you hit a server.

```
key="user_123" → position 45  → walk clockwise → hits Server A (90)
key="user_456" → position 130 → walk clockwise → hits Server B (180)
key="user_789" → position 220 → walk clockwise → hits Server C (270)
```

---

## Why Adding/Removing Servers Is Now Cheap

Add Server D at position 135:

```
Before:                After:

key at 130 → B         key at 130 → D  ← only keys between 90-135 move
key at 45  → A         key at 45  → A  ✅ unchanged
key at 220 → C         key at 220 → C  ✅ unchanged
```

Only the keys between the previous server and the new server's position get remapped. Everything else stays exactly where it is.

```
Naive hashing:      adding 1 server remaps ~K keys (nearly all)
Consistent hashing: adding 1 server remaps ~K/N keys (just 1/N of them)
```

At scale, the difference between remapping 100% vs 3% of your cache is the difference between a disaster and a non-event.

---

## Virtual Nodes (VNodes)

With only 3-4 servers on the ring, they're unlikely to be evenly spaced. One server might handle 60% of keys while another handles 10%.

**Solution:** Instead of placing each server once on the ring, place it multiple times using different hash values:

```
Server A → placed at positions: 15, 90, 200
Server B → placed at positions: 45, 150, 280
Server C → placed at positions: 75, 170, 310
```

Each of these positions is a virtual node. With enough virtual nodes (typically 100-200 per server), keys distribute statistically evenly across all servers.

---

## Where Consistent Hashing Is Used

| System | How it uses consistent hashing |
|--------|-------------------------------|
| **Redis Cluster** | Divides key space into 16,384 hash slots distributed across nodes |
| **Cassandra** | Consistent hashing with virtual nodes to distribute data across the ring |
| **Amazon DynamoDB** | Consistent hashing at its core for partition distribution |
| **Load Balancers** | Sticky sessions — same client always hits same backend |

---

## Consistent Hashing in Kafka

When you send a Kafka message with a key:

```java
ProducerRecord<String, String> record =
    new ProducerRecord<>("topic", "user_123", message);
```

Kafka hashes `"user_123"` and assigns it to a specific partition — always the same partition for the same key. This guarantees **ordering per key** — all events for `user_123` are processed sequentially by the same consumer.

> **Critical:** If you add partitions to a topic, existing keys may remap to different partitions — exactly the same problem as naive modulo hashing. This is why you should set partition count correctly upfront and avoid changing it on live topics.

---

## How to Explain It in an Interview

> "Consistent hashing places both servers and keys on a circular hash ring. A key is assigned to the first server clockwise from its position. When a server is added or removed, only K/N keys need remapping instead of nearly all of them. Virtual nodes ensure even distribution even with few physical servers."

**Then draw the ring.** Interviewers love when you draw the ring.

---

## Key Takeaways

```
Problem:    Naive hashing remaps almost all keys when servers change
Solution:   Hash ring where only neighboring keys are affected

Problem 2:  Uneven distribution with few servers
Solution 2: Virtual nodes — each server occupies multiple ring positions

Real use:   Kafka partitioning, Redis Cluster, Cassandra, DynamoDB
```
