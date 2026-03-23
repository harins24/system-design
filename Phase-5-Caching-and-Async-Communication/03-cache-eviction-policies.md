# Cache Eviction Policies

**Phase 5 — Caching + Async Communication | Topic 3**

---

## What is Cache Eviction?

A cache has finite memory. When it's full and a new item needs to be stored, something must be removed to make room. The eviction policy determines which item gets removed.

```
Cache capacity: 3 items
Cache state:    [A, B, C]  ← full

New item D arrives:
  Must evict something: A? B? C?

  Wrong choice: evict A → next request for A = cache miss → DB hit
  Right choice: evict whichever is least useful to keep

  "Most useful" = most likely to be requested again
```

Choosing the right eviction policy directly impacts your cache hit rate. Evict the wrong data and you constantly re-fetch from the database.

---

## 1. LRU — Least Recently Used

Evicts the item that was accessed least recently.

```
Access sequence: A, B, C, A, B, D

Cache state after each access (capacity = 3):
  After A:   [A]
  After B:   [A, B]
  After C:   [A, B, C]  ← full
  After A:   [B, C, A]  ← A moved to front (recently used)
  After B:   [C, A, B]  ← B moved to front
  After D:   [A, B, D]  ← C evicted (least recently used)

C was evicted because it hadn't been accessed since its initial insertion.
```

### Why LRU Works

LRU exploits **temporal locality** — data accessed recently is more likely to be accessed again soon. This assumption holds true for most workloads:

- Users browsing product pages → same products requested repeatedly in a session
- Hot blog posts → accessed many times in a short window
- Database query results → same queries run repeatedly

### Implementation — Doubly Linked List + HashMap

```java
public class LRUCache<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>();  // MRU end
    private final Node<K, V> tail = new Node<>();  // LRU end

    // O(1) get + O(1) eviction

    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;  // cache miss

        moveToFront(node);  // mark as recently used
        return node.value;
    }

    public void put(K key, V value) {
        Node<K, V> existing = map.get(key);

        if (existing != null) {
            existing.value = value;
            moveToFront(existing);
            return;
        }

        Node<K, V> newNode = new Node<>(key, value);
        map.put(key, newNode);
        addToFront(newNode);

        if (map.size() > capacity) {
            Node<K, V> lru = removeTail();  // evict LRU item
            map.remove(lru.key);
        }
    }
}
```

### LRU Weakness — Cache Pollution

```
Cache size: 1000 items
Normal usage: 100 hot items accessed repeatedly

Scan query runs: SELECT * FROM products (10,000 items)
→ Loads 10,000 items into cache one by one
→ Evicts all 100 hot items
→ Cache now full of cold scan data accessed only once
→ All subsequent requests for hot items = cache misses

This is called "cache pollution" or "cache thrashing"
LRU-K and LIRS variants solve this but are more complex
```

**Best for:** General-purpose workloads with temporal locality. Default choice for most applications.

---

## 2. LFU — Least Frequently Used

**Evicts the item that has been accessed the fewest number of times.**

```
Access sequence: A(×5), B(×3), C(×1), D(×2)
Frequencies:     A=5, B=3, C=1, D=2

New item E arrives (cache full):
  Evict C → lowest frequency (accessed only once)

Next sequence: A(×5), B(×3), D(×2), E(×1)
  Frequencies: A=5, B=3, D=2, E=1
```

### Why LFU Works

LFU exploits **frequency locality** — data accessed frequently in the past is likely to remain popular. Good for:
- Product pages with stable popularity rankings
- News articles with persistent traffic
- API endpoints where some are always more popular

### LFU Weakness — Frequency Decay Problem

```
Old popular item:
  "iPhone 15 review" → accessed 10,000 times in 2023
  Still in cache, frequency = 10,000

New popular item:
  "iPhone 17 review" → accessed 500 times today
  Frequency = 500

LFU will never evict the old item (frequency 10,000 > 500)
Even though the old item is no longer being accessed
→ Old items "haunt" the cache forever

Solution: Frequency decay (aging)
  Periodically halve all frequencies
  Old items' frequencies decay over time
  New popular items can overtake them
```

**Best for:** Stable popularity distributions where hot items stay hot long-term. Music streaming (popular songs), video platforms (evergreen content).

---

## 3. FIFO — First In, First Out

**Evicts the oldest item in the cache regardless of access frequency.**

```
Insertion order: A → B → C → D

Cache full (capacity 3): [A, B, C]

Insert D:
  Evict A (inserted first)
  Cache: [B, C, D]

Insert E:
  Evict B (now oldest)
  Cache: [C, D, E]
```

### Why FIFO is Usually Wrong

```
Problem: Doesn't consider access patterns at all

Scenario:
  A was inserted first AND is accessed 1000 times/hour
  C was inserted third and accessed once

  FIFO evicts A anyway
  → Very hot item evicted
  → Massive cache miss rate

FIFO treats all items as equal regardless of popularity
Only appropriate when: items naturally expire in order (message queues)
                        all items have similar access patterns
```

**Best for:** Message queues, job queues, simple sequential processing. Not appropriate for general caching.

---

## 4. MRU — Most Recently Used

**Evicts the most recently used item — the opposite of LRU.**

```
Evict the item just accessed? Sounds wrong — when does this help?

Scan pattern:
  Database full table scan reads every row once
  Never reads the same row twice in sequence
  "Most recently accessed" = least likely to be needed again
  → MRU is correct for sequential scan patterns

Example — video streaming:
  User watches video A → A evicted (won't watch same video twice right away)
  User watches video B → B evicted
  Cache holds "next likely to be watched" videos instead
```

**Best for:** Cyclic or scan access patterns where most recently accessed data is least likely to be re-accessed. Rarely appropriate for general use.

---

## 5. Random Replacement

**Evicts a randomly chosen item.**

```
Cache: [A, B, C, D]
New item E arrives:
  Pick random item to evict → say B
  Cache: [A, C, D, E]
```

```
Pros:
  ✅ Extremely simple to implement
  ✅ No overhead (no tracking, no ordering)
  ✅ Resistant to adversarial access patterns
     (attacker can't craft access pattern to defeat random)
  ✅ At large cache sizes, statistically approaches optimal

Cons:
  ❌ May evict very hot items by chance
  ❌ Not deterministic — hard to reason about behavior

Used by: CPU hardware caches (cheap in hardware), some CDNs
         Some algorithms use randomness + LRU (RRIP approximations)
```

---

## 6. LRU-K

Extension of LRU. Item not eligible for retention until accessed K times.

```
LRU-2:
  Item must be accessed at least 2 times before cached
  First access: stored in "probationary" buffer
  Second access: promoted to "protected" region of cache

  Benefits:
  One-time scans never pollute cache (accessed once, never promoted)
  Only genuinely popular items reach protected region

  Used by: PostgreSQL buffer pool (uses clock algorithm variant)
```

---

## 7. ARC — Adaptive Replacement Cache

**Dynamically balances between LRU and LFU based on workload.**

```
ARC maintains four lists:
  T1: Recently accessed once (LRU candidates)
  T2: Accessed more than once (LFU candidates)
  B1: Ghost entries evicted from T1 (metadata only)
  B2: Ghost entries evicted from T2 (metadata only)

Adaptation logic:
  Cache miss AND in B1 (recently evicted LFU candidate):
    → Increase T1 size (workload benefits from recency)

  Cache miss AND in B2 (recently evicted LRU candidate):
    → Increase T2 size (workload benefits from frequency)

ARC automatically adapts to:
  Scan-heavy workloads (grows LFU region)
  Recency-heavy workloads (grows LRU region)
  Mixed workloads (balanced)
```

**Best for:** Unknown or mixed workloads. ARC consistently outperforms both LRU and LFU across different patterns. Used by ZFS filesystem, some databases.

---

## 8. CLOCK (Second Chance)

Approximation of LRU with lower overhead. Used in OS page replacement.

```
Arrange cache items in a circular buffer
Each item has a "reference bit" (0 or 1)

When item accessed: set reference bit = 1
When eviction needed:
  Advance "clock hand" around circle
  If reference bit = 1: set to 0, advance hand (give second chance)
  If reference bit = 0: evict this item

Example:
  [A:1] → [B:1] → [C:0] → [D:1]
              ↑ hand

  Eviction needed:
  Hand at B: bit=1 → set to 0, advance → B gets second chance
  Hand at C: bit=0 → evict C

  [A:1] → [B:0] → [E:1] → [D:1]
                  ↑ new item E
```

```
Pros:
  ✅ Approximates LRU with O(1) operations
  ✅ Simple hardware implementation
  ✅ Less overhead than true LRU (no linked list reordering)

Used by: Linux kernel (page frame reclamation), many OS page tables
         PostgreSQL clock-sweep buffer replacement
```

---

## Redis Eviction Policies

Redis supports 8 eviction policies. This is frequently tested:

```
noeviction (default):
  Return error when memory full
  Application must handle "OOM" errors
  Use: when data loss is unacceptable, cache as primary store

allkeys-lru:
  Evict any key using LRU
  Use: general cache, all keys are candidates

volatile-lru:
  Evict only keys with TTL set, using LRU
  Keys without TTL are never evicted
  Use: mixed store (some permanent data, some cache data)

allkeys-lfu:
  Evict any key using LFU
  Use: stable popularity distribution, evergreen content

volatile-lfu:
  Evict only TTL keys using LFU
  Use: same as volatile-lru but frequency-based

allkeys-random:
  Evict any key randomly
  Use: uniform access distribution (every key equally popular)

volatile-random:
  Evict random TTL keys
  Use: when you want randomness but protect non-TTL keys

volatile-ttl:
  Evict key with shortest remaining TTL
  Use: expire soon anyway, might as well evict now
```

**Which to choose:**

```
Cache (all keys expendable):     allkeys-lru (most common)
Mixed store (some permanent):    volatile-lru
High reuse of popular items:     allkeys-lfu
Explicit TTL management:         volatile-ttl
Never lose data:                 noeviction (handle OOM in app)
```

```
# Redis configuration
maxmemory 2gb
maxmemory-policy allkeys-lru
maxmemory-samples 10  # LRU approximation sample size
                       # Higher = more accurate, more CPU
                       # 5-10 is usually sufficient
```

---

## Redis LRU — Approximate, Not Exact

Redis does NOT implement true LRU (no linked list across all keys — too expensive for millions of keys). Instead:

```
On eviction:
  Sample maxmemory-samples random keys
  Evict the LRU key among the sample

maxmemory-samples = 5 (default):
  Sample 5 random keys → evict oldest among them
  Good approximation with minimal overhead

maxmemory-samples = 10:
  Better approximation, slightly more CPU
  Approaches true LRU accuracy

Redis docs: "samples=10 is close to true LRU
             but uses slightly more CPU"
```

---

## Eviction Policy Selection Framework

```
What is your access pattern?

Temporal locality (recent = popular):
  → LRU (most common choice)
  → Good for: web sessions, product pages, user data

Frequency locality (frequent = popular):
  → LFU
  → Good for: media streaming, static content, evergreen articles

Unknown / mixed:
  → ARC (adapts automatically)
  → Or: LRU with short TTL as safety net

Sequential scan / cyclic:
  → MRU or LRU-K (prevent scan pollution)
  → Good for: database buffer pools, analytics queries

All keys have equal value:
  → Random replacement
  → Simple, no overhead

Redis specifically:
  Default cache:        allkeys-lru
  Protect permanent:    volatile-lru
  Stable popularity:    allkeys-lfu
  Never lose writes:    noeviction
```

---

## Eviction vs Expiration

Important distinction:

```
Expiration (TTL):
  Item removed after a fixed time
  Regardless of how often it's accessed
  Controlled by you (SET key value EX 3600)
  Item becomes stale and is deleted

Eviction:
  Item removed because cache is full
  Based on eviction policy (LRU, LFU, etc.)
  Controlled by eviction algorithm
  Item might have been perfectly valid but had to make room

Both can happen:
  Item expires (TTL) → removed even if cache not full
  Cache full → evict item even if TTL hasn't expired

In Redis: expiration and eviction are independent mechanisms
  Key with TTL=3600 AND cache full:
  → May be evicted before TTL expires (if eviction policy targets it)
  → Will definitely be removed at TTL if not evicted first
```

---

## Key Takeaways

```
Eviction policy: decides what to remove when cache is full

Main policies:
  LRU (Least Recently Used):
    Evict item not accessed longest
    Best for: general workloads, temporal locality
    Weakness: cache pollution from scans

  LFU (Least Frequently Used):
    Evict item accessed fewest times
    Best for: stable popularity, evergreen content
    Weakness: old popular items haunt cache (use decay)

  FIFO (First In, First Out):
    Evict oldest inserted item
    Best for: queues, sequential processing
    Not for: general caching (ignores access patterns)

  MRU (Most Recently Used):
    Evict most recently accessed
    Best for: cyclic/scan patterns (almost never used)

  Random:
    Evict randomly
    Best for: uniform access, simple hardware

  CLOCK:
    Approximate LRU, O(1) operations
    Used by: OS kernel, PostgreSQL buffer pool

  ARC:
    Adaptive, balances LRU and LFU automatically
    Best for: unknown/mixed workloads
    Used by: ZFS filesystem

Redis eviction policies:
  allkeys-lru    → general cache (most common)
  volatile-lru   → mixed store (some permanent keys)
  allkeys-lfu    → stable popularity distribution
  noeviction     → never lose data (handle OOM in app)

Redis uses approximate LRU (sampled) not true LRU
maxmemory-samples controls accuracy vs CPU tradeoff

Eviction ≠ Expiration:
  TTL expires  → time-based removal
  Eviction     → space-based removal
  Both can apply to same key
```
