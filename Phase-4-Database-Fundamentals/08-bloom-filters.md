# Bloom Filters

**Phase 4 — Database Fundamentals | Topic 8 of 9**

---

## What is a Bloom Filter?

A Bloom filter is a space-efficient probabilistic data structure that answers one question extremely fast:

> **"Is this element definitely NOT in the set, or possibly in the set?"**

```
Bloom filter can say:
  "Definitely NOT in set" → 100% accurate (no false negatives)
  "Possibly in set"       → might be wrong (false positives possible)

What it cannot say:
  "Definitely IN set"     → impossible, always probabilistic
```

This seems limited, but it's incredibly useful. The key insight: a definitive NO is just as valuable as a definitive YES — and Bloom filters give you definitive NOs with tiny memory.

---

## The Problem It Solves

```
Scenario: URL shortener receiving billions of short URLs
          Need to check if a URL already exists before creating

Naive approach:
  Query database for every new URL request
  → 10,000 requests/second × 1 DB query each = 10,000 DB queries/sec
  → Database overwhelmed
  → Expensive and slow

With Bloom filter:
  Bloom filter in memory (< 1MB for millions of URLs)
  Check Bloom filter first:
    → "Definitely NOT in DB" → skip DB query, create new URL
    → "Possibly in DB"       → check DB to confirm

  With 1% false positive rate:
  99% of new URLs → immediate NO → skip DB query
  1% false positives → unnecessary DB query (but no incorrect result)

  DB queries reduced by ~99%
```

---

## How It Works Internally

A Bloom filter is a **bit array** of m bits (all initialized to 0) combined with k independent hash functions.

### Adding an Element

```
Bit array: [0,0,0,0,0,0,0,0,0,0]  (10 bits, indices 0-9)

Add "google.com":
  hash1("google.com") % 10 = 3 → set bit[3] = 1
  hash2("google.com") % 10 = 7 → set bit[7] = 1
  hash3("google.com") % 10 = 1 → set bit[1] = 1

Bit array: [0,1,0,1,0,0,0,1,0,0]
               ↑   ↑       ↑
               1   3       7

Add "github.com":
  hash1("github.com") % 10 = 4 → set bit[4] = 1
  hash2("github.com") % 10 = 1 → set bit[1] = 1 (already 1)
  hash3("github.com") % 10 = 9 → set bit[9] = 1

Bit array: [0,1,0,1,1,0,0,1,0,1]
               ↑   ↑ ↑     ↑   ↑
               1   3 4     7   9
```

### Checking if Element Exists

```
Check "amazon.com":
  hash1("amazon.com") % 10 = 2 → bit[2] = 0 → STOP
  → At least one bit is 0 → "Definitely NOT in set" ✅

Check "google.com":
  hash1("google.com") % 10 = 3 → bit[3] = 1 ✓
  hash2("google.com") % 10 = 7 → bit[7] = 1 ✓
  hash3("google.com") % 10 = 1 → bit[1] = 1 ✓
  → All bits are 1 → "Possibly in set" ✅ (actually is in set)

Check "yahoo.com" (never added):
  hash1("yahoo.com") % 10 = 4 → bit[4] = 1 ✓
  hash2("yahoo.com") % 10 = 9 → bit[9] = 1 ✓
  hash3("yahoo.com") % 10 = 1 → bit[1] = 1 ✓
  → All bits are 1 → "Possibly in set"
  → But yahoo.com was never added! → FALSE POSITIVE ❌
  → Bits were set by "github.com" hash collisions
```

This is why Bloom filters have **false positives but never false negatives** — if even one bit is 0, the element is definitely absent. If all bits are 1, those bits may have been set by other elements.

---

## The Math Behind Bloom Filters

```
Parameters:
  n = expected number of elements
  m = number of bits in array
  k = number of hash functions

False positive rate:
  p ≈ (1 - e^(-kn/m))^k

Practical sizing:
  For 1% false positive rate with n elements:
  m ≈ 9.6 × n bits
  k ≈ 7 hash functions

Example:
  1 million URLs, 1% false positive rate:
  m = 9.6 × 1,000,000 = 9.6 million bits ≈ 1.2 MB

  Compare: storing 1M URLs as strings
  = 1M × 30 bytes average = 30 MB

  Bloom filter uses 25x less memory with 1% false positive rate

Key relationships:
  More bits (larger m)      → lower false positive rate
  More elements (larger n)  → higher false positive rate
  More hash functions (k)   → lower false positive rate up to optimal k
                              too many → all bits become 1 → 100% false positive
```

---

## Bloom Filter Limitations

```
Cannot delete elements:
  Setting bits to 0 when deleting would unset bits
  shared by other elements → false negatives

  Solution: Counting Bloom Filter
  Each position stores count instead of bit
  Increment on add, decrement on delete
  Cost: ~4x more memory

False positive rate increases with load:
  As more elements added, more bits set to 1
  More chances of false positives

  Solution: Size appropriately upfront
  Or: use scalable Bloom filters that grow automatically

No enumeration:
  Cannot list elements in the filter
  Cannot retrieve elements

Not suitable when false positives are catastrophic:
  Medical diagnosis → false negative = missed disease
  Security authentication → false positive = unauthorized access
```

---

## Real-World Uses

### Google Chrome — Safe Browsing

```
Chrome needs to check every URL you visit against
a list of ~650,000 malicious URLs.

Approach:
  Download compressed Bloom filter of malicious URLs (~200KB)
  Check every URL locally against filter:
    → "Definitely safe" → proceed immediately (no network call)
    → "Possibly malicious" → check Google's server (1% of URLs)

  Result:
  99% of URLs: instant local check, no privacy risk, zero latency
  1% of suspicious URLs: server verification

  Alternative (no Bloom filter):
  Every URL → Google server → privacy issue + latency + cost
```

### Apache Cassandra — SSTable Lookups

```
Cassandra stores data in SSTables (immutable files on disk).
Reading a key requires checking potentially many SSTables.

Without Bloom filter:
  Read "user_123" → check SSTable 1? Open file, scan → no
                  → check SSTable 2? Open file, scan → no
                  → check SSTable 3? Open file, scan → yes
  Disk I/O for every SSTable checked

With Bloom filter (one per SSTable, in memory):
  Read "user_123" → check BF for SSTable 1? → "Definitely NOT" → skip
                  → check BF for SSTable 2? → "Definitely NOT" → skip
                  → check BF for SSTable 3? → "Possibly" → read SSTable

  ~99% of SSTable checks eliminated → dramatically less disk I/O
  Bloom filters fit in RAM → microsecond checks vs millisecond disk reads
```

### Database JOIN Optimization

```
JOIN optimization:
  Table A has 100M rows
  Table B has 1K rows

  Build Bloom filter of Table B's join keys
  Scan Table A: check each row's join key against filter
    → "Definitely NOT in B" → skip (no join needed)
    → "Possibly in B" → attempt join

  Filters out most of Table A before JOIN
  → Dramatically reduces JOIN computation

  Used by: Spark, Hive, Impala for large-scale JOINs
```

### Redis — Bloom Filter Module

```java
// Redis with RedisBloom module
// Add element
redisBloom.add("url-filter", "google.com");

// Check existence
boolean mightExist = redisBloom.exists("url-filter", "amazon.com");
// false → definitely not in set, skip DB query
// true  → might exist, check DB

// Batch check
Map<String, Boolean> results = redisBloom.existsMulti(
    "url-filter", "url1.com", "url2.com", "url3.com");

// Create with custom parameters
BFReserveParams params = BFReserveParams.reserveParams()
    .errorRate(0.001)      // 0.1% false positive rate
    .initialCapacity(1000000); // 1M elements
redisBloom.createFilter("url-filter", params);
```

---

## Bloom Filter vs Hash Set

| | Hash Set | Bloom Filter |
|--|----------|-------------|
| Stores actual elements | Yes | No (only bits) |
| False positives | None | Possible (tunable) |
| Lookup | O(1) | O(k) — constant |
| Memory | O(n) × element size | O(n) × ~10 bits |
| Delete elements | Yes | No (without Counting variant) |
| Enumerate elements | Yes | No |

```
When to use Bloom Filter over HashSet:
  Elements are large (URLs, IDs, documents)
  Memory is constrained
  False positives are acceptable
  You only need membership testing, not retrieval

When to use HashSet:
  Elements are small
  Memory is not constrained
  False positives are NOT acceptable
  You need to retrieve elements
```

---

## Bloom Filter in System Design Interviews

**URL Shortener:**
> "Before querying the database to check if a short code exists, I'd check a Bloom filter first. A definitive 'not in set' skips the DB query entirely. Only when the filter says 'possibly exists' do we hit the database."

**Content Deduplication (like Gmail or Google Drive):**
> "When a user uploads a file, I check a Bloom filter of all known file hashes. If the filter says 'definitely not seen before', skip the expensive dedup check entirely. Only possible duplicates get full hash comparison."

**Recommendation System:**
> "Before recommending an article to a user, check a per-user Bloom filter of already-seen articles. 'Definitely not seen' → safe to recommend. 'Possibly seen' → verify before recommending. Avoids storing a full list of viewed articles per user."

---

## Key Takeaways

```
Bloom Filter: Space-efficient probabilistic membership testing

Answers: "Definitely NOT in set" (always correct)
         "Possibly in set" (may be wrong — false positive)

Never gives: false negatives (if element was added, always detected)

How it works:
  Bit array of m bits
  k hash functions
  Add: set k bit positions
  Check: if any bit position is 0 → definitely absent
         if all bit positions are 1 → possibly present

Tradeoffs:
  More bits (m ↑) → lower false positive rate, more memory
  More elements added → higher false positive rate
  k optimal ≈ 7 for typical configurations

Cannot delete (without Counting Bloom Filter)
Cannot enumerate elements
Not suitable when false positives are catastrophic

Real uses:
  Chrome Safe Browsing    → malicious URL check locally
  Cassandra/HBase         → skip SSTables/HFiles on read
  Redis BloomFilter       → distributed membership testing
  Spark JOIN optimization → filter rows before JOIN
  DB query planner        → skip index pages

Use in interview when:
  Need to reduce expensive lookups (DB, network, disk)
  Can tolerate small % false positives
  Memory efficiency matters
  Only need "is it there?" not "give it to me"
```
