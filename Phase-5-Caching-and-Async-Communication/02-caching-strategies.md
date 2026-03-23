# Caching Strategies

**Phase 5 — Caching + Async Communication | Topic 2 of 8**

---

## Overview

A caching strategy defines when data gets loaded into the cache, when it gets written back to the source, and who is responsible for managing that flow. Choosing the wrong strategy causes either stale data, data loss, or wasted cache space.

There are five main strategies. Each fits different read/write patterns.

---

## 1. Cache-Aside (Lazy Loading)

The application manages the cache directly. Cache is only populated when data is requested.

```
Read flow:
  1. Application checks cache for key
  2. Cache HIT  → return data
  3. Cache MISS → fetch from DB
                  store in cache
                  return data

Write flow:
  1. Application writes to DB
  2. Application invalidates (deletes) cache key
  3. Next read will populate cache fresh

         Application
        /     |      \
  Cache     miss      DB
    ↑                  |
    └── populate ──────┘
```

### Implementation

```java
@Service
public class ProductService {

    private final RedisTemplate<String, Product> redis;
    private final ProductRepository db;
    private static final Duration TTL = Duration.ofMinutes(30);

    // READ: Cache-Aside
    public Product getProduct(String id) {
        String key = "product:" + id;

        // Step 1: Check cache
        Product cached = redis.opsForValue().get(key);
        if (cached != null) {
            return cached; // Cache HIT
        }

        // Step 2: Cache MISS — fetch from DB
        Product product = db.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));

        // Step 3: Populate cache
        redis.opsForValue().set(key, product, TTL);

        return product;
    }

    // WRITE: Invalidate cache
    public Product updateProduct(String id, ProductUpdate update) {
        Product product = db.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));

        product.apply(update);
        db.save(product);

        // Invalidate — next read will fetch fresh
        redis.delete("product:" + id);

        return product;
    }
}
```

### When to Use

```
✅ Read-heavy workloads
✅ Cache only what's actually needed (lazy — nothing cached upfront)
✅ Tolerates brief inconsistency (between write and next cache populate)
✅ Default strategy for most applications

❌ Cache miss on every first access per key (cold start)
❌ Stale data window between write and invalidation + re-cache
❌ Race condition: two threads read same stale value simultaneously
```

**Race condition (classic problem):**
```
Thread A: reads DB (value = $50)
Thread B: updates DB ($60), deletes cache key
Thread A: writes $50 to cache (overwrites "deleted")
Cache now has $50 even though DB has $60

Solution: short TTL as safety net
          Even if race occurs, data self-corrects on TTL expiry
```

---

## 2. Read-Through

Cache sits in front of DB. Application always talks to cache. Cache manages DB interaction.

```
Read flow:
  1. Application requests data from cache
  2. Cache HIT  → cache returns data
  3. Cache MISS → CACHE fetches from DB (not application)
                  cache stores result
                  cache returns to application

Application never talks to DB directly for reads.

Application → Cache → (on miss) → DB
                 ↑                  |
                 └── auto-populate──┘
```

### Difference from Cache-Aside

```
Cache-Aside:   Application checks cache, then DB
               Application populates cache on miss
               Cache is a dumb store

Read-Through:  Application only talks to cache
               Cache pulls from DB on miss automatically
               Cache is an intelligent proxy
```

### Implementation

```java
// With Spring Cache abstraction (acts like read-through)
@Cacheable(value = "products", key = "#id",
           unless = "#result == null")
public Product getProduct(String id) {
    // This method only called on cache miss
    // Spring auto-populates cache with returned value
    return db.findById(id).orElse(null);
}

// Caffeine with a loader (true read-through)
LoadingCache<String, Product> cache = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(30, MINUTES)
    .build(id -> db.findById(id).orElse(null)); // auto-load on miss

Product product = cache.get(productId); // never misses, auto-loads
```

### When to Use

```
✅ Application code is simpler (no manual cache management)
✅ Cache and DB always in sync (cache handles population)
✅ Consistent read pattern for the same data

❌ First request always slow (cache miss goes through cache to DB)
❌ Cache node failure means application can't read data
   (unless fallback to DB is implemented)
❌ Cache may load data that never gets requested again
```

---

## 3. Write-Through

Every write goes through the cache first, then to DB. Cache and DB always in sync.

```
Write flow:
  1. Application writes to cache
  2. Cache SYNCHRONOUSLY writes to DB
  3. Both updated — cache confirmed up to date
  4. Return success

Read flow (typically paired with Read-Through):
  Reads always hit cache (always has latest data)
  No stale data problem

Application → [Cache] → DB
              writes       writes
              both         synchronously
```

### Implementation

```java
@Service
public class WriteThoughProductService {

    private final RedisTemplate<String, Product> redis;
    private final ProductRepository db;

    public Product updateProduct(String id, ProductUpdate update) {
        Product product = db.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));

        product.apply(update);

        // Write to DB first
        Product saved = db.save(product);

        // Immediately update cache (write-through)
        redis.opsForValue().set(
            "product:" + id,
            saved,
            Duration.ofMinutes(30));

        return saved;
    }
}
```

### When to Use

```
✅ Cache always has fresh data (no staleness)
✅ Read after write is immediately consistent
✅ Good for data that is written and immediately read

❌ Higher write latency (must write to BOTH cache and DB)
❌ Cache filled with data that may never be read
   (write everything, but only some reads happen)
❌ Every write goes to both → not suitable for write-heavy workloads

Classic pairing: Write-Through + Read-Through
  → Writes: go through cache to DB (always fresh)
  → Reads:  always from cache (fast, consistent)
  → DB and cache always in sync
```

---

## 4. Write-Behind (Write-Back)

Write to cache immediately, write to DB asynchronously later.

```
Write flow:
  1. Application writes to cache
  2. Return success IMMEDIATELY (don't wait for DB)
  3. Cache asynchronously writes to DB in background
     (batched, after delay, or on eviction)

Read flow:
  Always from cache (has latest data)
  DB may be slightly behind

Application → Cache → (async, batched) → DB
             fast                         eventually
             return
```

### Why Write-Behind Exists

```
Problem with Write-Through:
  Write to cache (1ms) + write to DB (10ms) = 11ms per write
  Write-heavy system: 100K writes/sec × 11ms = bottleneck

Write-Behind:
  Write to cache (1ms) = 1ms per write → return immediately
  Batch 1000 DB writes together → one round trip for 1000 writes
  DB write cost amortized across many operations

  10x lower write latency
  10x higher write throughput
```

### Implementation (Conceptual)

```java
@Service
public class WriteBackProductService {

    private final RedisTemplate<String, Product> redis;
    private final ProductRepository db;

    // Write queue in Redis
    private static final String DIRTY_KEYS = "dirty:products";

    public void updateProduct(String id, ProductUpdate update) {
        String key = "product:" + id;

        Product product = getFromCacheOrDB(id);
        product.apply(update);

        // Write to cache immediately
        redis.opsForValue().set(key, product);

        // Mark as dirty (needs DB sync)
        redis.opsForSet().add(DIRTY_KEYS, id);

        // Return immediately — DB write happens asynchronously
    }

    // Background job — runs every second
    @Scheduled(fixedDelay = 1000)
    public void flushDirtyKeys() {
        Set<String> dirtyIds = redis.opsForSet().members(DIRTY_KEYS);
        if (dirtyIds == null || dirtyIds.isEmpty()) return;

        List<Product> toSave = dirtyIds.stream()
            .map(id -> redis.opsForValue().get("product:" + id))
            .filter(Objects::nonNull)
            .collect(toList());

        // Batch write to DB (much more efficient)
        db.saveAll(toSave);

        // Clear dirty set
        redis.opsForSet().remove(DIRTY_KEYS, dirtyIds.toArray());
    }
}
```

### When to Use

```
✅ Write-heavy workloads (gaming leaderboards, real-time counters)
✅ Batch processing makes DB writes more efficient
✅ Lowest write latency (writes return before DB persisted)
✅ Reduces DB write load significantly

❌ Data loss risk: cache crashes before flush → lost writes
   (must persist cache with Redis AOF for durability)
❌ Complexity: need to handle crash recovery, partial flushes
❌ DB may be significantly behind cache
   → Other services reading directly from DB see stale data
❌ Not suitable when writes must be immediately durable

Mitigation for data loss:
  Redis persistence: AOF (Append Only File) with fsync every second
  Replication: Redis sentinel/cluster with multiple replicas
  Accept small loss: some use cases tolerate losing last second of writes
```

---

## 5. Refresh-Ahead (Proactive Caching)

Cache predicts what will be needed and refreshes it before expiry.

```
Without Refresh-Ahead:
  t=0:    cache populated, TTL=3600
  t=3600: TTL expires → cache miss → fetch from DB → latency spike
  t=3600: users experience slow response during refresh

With Refresh-Ahead:
  t=0:    cache populated, TTL=3600
  t=3540: cache detects "will expire soon" (e.g., <10% TTL remaining)
  t=3540: cache proactively fetches fresh data from DB
  t=3541: cache updated with fresh data, TTL reset
  t=3600: no expiry event — data already fresh
  t=3600: users experience no latency spike
```

### Implementation

```java
@Service
public class RefreshAheadCacheService {

    private final RedisTemplate<String, Product> redis;
    private final ProductRepository db;
    private final ScheduledExecutorService scheduler;

    private static final Duration TTL = Duration.ofSeconds(3600);
    private static final long REFRESH_THRESHOLD_SECS = 360; // 10%

    public Product getProduct(String id) {
        String key = "product:" + id;

        Product cached = redis.opsForValue().get(key);

        if (cached != null) {
            // Check remaining TTL
            Long remainingTtl = redis.getExpire(key, TimeUnit.SECONDS);

            if (remainingTtl != null && remainingTtl < REFRESH_THRESHOLD_SECS) {
                // Proactively refresh in background
                scheduler.submit(() -> refreshProduct(id));
            }

            return cached; // Return current data immediately (no wait)
        }

        // Cache miss — synchronous fetch
        return refreshProduct(id);
    }

    private Product refreshProduct(String id) {
        Product product = db.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
        redis.opsForValue().set("product:" + id, product, TTL);
        return product;
    }
}
```

### When to Use

```
✅ Frequently accessed data where latency spikes on TTL expiry are unacceptable
✅ Predictable access patterns (you know data will be needed)
✅ Data that is expensive to fetch (external API, complex query)

❌ Wastes resources refreshing data that may not be needed again
❌ Complex to implement correctly
❌ May serve stale data for brief window during background refresh

Real use: CDNs use this pattern
  Edge server detects content about to expire
  Fetches fresh content from origin before expiry
  Users never see cache miss latency
```

---

## Strategy Comparison

```
Strategy         Read Path          Write Path         Staleness    Complexity
─────────────────────────────────────────────────────────────────────────────
Cache-Aside      App → Cache/DB     App → DB + delete   Brief        Low
Read-Through     App → Cache → DB   App → DB + delete   Brief        Medium
Write-Through    Cache handles      App → Cache → DB    None         Medium
Write-Behind     Cache handles      App → Cache         Possible     High
Refresh-Ahead    Cache handles      App → DB            None         High
```

---

## Combining Strategies

Real systems combine strategies per data type:

```
E-commerce system:

Product catalog:     Cache-Aside + TTL 1 hour
  → Read-heavy, occasional writes, brief staleness OK

User sessions:       Write-Through
  → Must be consistent, read immediately after write

Shopping cart:       Write-Behind
  → High write rate (every item add/remove), can batch DB writes

Recommendations:     Refresh-Ahead
  → Expensive ML computation, must never have cold cache

Rate limit counters: Write-Behind
  → Extreme write rate, slight inaccuracy acceptable
  → Redis INCR (atomic) + periodic DB sync
```

---

## Strategy Selection Framework

```
Question 1: Is staleness acceptable?
  No  → Write-Through or Refresh-Ahead
  Yes → Cache-Aside or Write-Behind

Question 2: What is the read/write ratio?
  Read-heavy  → Cache-Aside or Read-Through
  Write-heavy → Write-Behind
  Balanced    → Write-Through

Question 3: Can you afford data loss on cache failure?
  No  → Write-Through (DB always up to date)
  Yes → Write-Behind (faster writes, risk on cache crash)

Question 4: Who should manage cache population?
  Application → Cache-Aside
  Cache layer → Read-Through, Write-Through

Question 5: Are write latency spikes unacceptable?
  Yes → Write-Behind (async DB writes)
  No  → Write-Through (simpler, consistent)
```

### Spring Boot Implementation

```java
// Spring Cache abstraction covers most strategies:

// Cache-Aside / Read-Through hybrid:
@Cacheable("products")        // read from cache, populate on miss
public Product get(String id) { return db.findById(id); }

// Write-Through:
@CachePut("products")         // always update cache + execute method
public Product update(Product p) { return db.save(p); }

// Cache invalidation:
@CacheEvict("products")       // remove from cache
public void delete(String id) { db.deleteById(id); }

// Evict all:
@CacheEvict(value = "products", allEntries = true)
public void clearAll() { }

// Multiple cache operations:
@Caching(evict = {
    @CacheEvict("products"),
    @CacheEvict("product-list")
})
public Product update(Product p) { return db.save(p); }

// Configuration:
@Bean
public CacheManager cacheManager(RedisConnectionFactory factory) {
    RedisCacheConfiguration config = RedisCacheConfiguration
        .defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(30))
        .serializeValuesWith(
            RedisSerializationContext.SerializationPair
                .fromSerializer(new GenericJackson2JsonRedisSerializer()));

    return RedisCacheManager.builder(factory)
        .cacheDefaults(config)
        .withCacheConfiguration("sessions",
            config.entryTtl(Duration.ofHours(1)))
        .withCacheConfiguration("products",
            config.entryTtl(Duration.ofMinutes(30)))
        .build();
}
```

---

## Key Takeaways

```
5 caching strategies:

Cache-Aside:    App manages cache manually
                Read: check cache → miss → fetch DB → populate
                Write: update DB → delete cache key
                Use: default for most apps, read-heavy

Read-Through:   Cache manages DB population
                App only talks to cache
                Use: simplifies app code, consistent reads

Write-Through:  Every write → cache → DB synchronously
                Cache always fresh, higher write latency
                Use: when reads immediately after writes must be fresh

Write-Behind:   Write to cache → return → async DB flush
                Lowest write latency, data loss risk on cache crash
                Use: write-heavy (gaming, counters, carts)

Refresh-Ahead:  Cache proactively refreshes before TTL expiry
                No latency spikes on expiry
                Use: expensive-to-fetch data, predictable access

Selection rules:
  Staleness OK? → Cache-Aside / Write-Behind
  Must be fresh? → Write-Through / Refresh-Ahead
  Write-heavy? → Write-Behind
  Simplicity? → Cache-Aside (default)

Spring Boot: @Cacheable, @CachePut, @CacheEvict
             covers Cache-Aside and Write-Through patterns
```
