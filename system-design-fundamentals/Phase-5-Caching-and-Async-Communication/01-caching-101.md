# Caching 101

**Phase 5 — Caching + Async Communication | Topic 1**

---

## What is Caching?

Caching is the practice of storing copies of frequently accessed data in a faster storage layer so future requests for that data can be served faster without recomputing or re-fetching from the slower source.

```
Without cache:
Every request → compute or fetch from DB → return result
  DB query: 50ms × 10,000 requests/second = 500,000ms of DB work/sec
  DB overwhelmed, latency high

With cache:
First request  → miss → fetch from DB → store in cache → return
Next 9,999 req → hit  → serve from cache → return
  Cache hit: <1ms × 9,999 = 9,999ms of cache work/sec
  DB only sees 1 request instead of 10,000
```

Caching is the single highest-leverage optimization in system design. A well-designed cache can reduce database load by 90-99% while simultaneously reducing response latency by 100x.

---

## The Caching Hierarchy

Caches exist at every layer of a system. Each layer is faster but smaller than the one below:

```
Fastest, smallest
─────────────────
CPU L1 Cache         ~0.5ns    ~32KB    per core
CPU L2 Cache         ~5ns      ~256KB   per core
CPU L3 Cache         ~20ns     ~8MB     shared
RAM                  ~100ns    GBs
─────────────────
Application Cache    ~0.1ms    GBs      in-process (JVM heap)
Distributed Cache    ~1ms      GBs-TBs  Redis, Memcached
─────────────────
SSD Storage          ~0.1ms    TBs      local disk
Network Storage      ~1ms      TBs      NAS, SAN
─────────────────
Database             ~5-50ms   TBs      PostgreSQL, MySQL
Remote API           ~100ms+   ∞        external service
─────────────────
Slowest, largest
```

System design primarily concerns the application cache and distributed cache layers — the ones you control and architect.

---

## Cache Hit and Miss

```
Cache Hit:
  Request arrives → check cache → data found → return immediately
  Hit ratio = cache_hits / total_requests

Cache Miss:
  Request arrives → check cache → not found →
  fetch from source → store in cache → return

Hit ratio is the most important cache metric:
  50% hit ratio → half requests still hit DB (mediocre)
  90% hit ratio → 10x reduction in DB load (good)
  99% hit ratio → 100x reduction in DB load (excellent)

What determines hit ratio:
  Cache size (larger = more data fits = more hits)
  Access patterns (hot data hit repeatedly = high ratio)
  TTL (short TTL = data expires = more misses)
  Eviction policy (wrong policy = hot data evicted = misses)
```

---

## What to Cache

Not everything should be cached. The best candidates:

```
✅ Cache these:
  Frequently read, rarely written
    → Product catalog, user profiles, configuration
    → "read 1000x, written once"

  Expensive to compute
    → Aggregations (total orders, revenue by category)
    → Complex JOIN results
    → Machine learning predictions
    → Rendered HTML pages

  Tolerable staleness
    → Product recommendations (5 min old is fine)
    → Social media counts (likes can lag)
    → Exchange rates (1 min old acceptable)

  Shared across many users
    → High hit ratio if same data requested by many
    → Product details, public content, news articles

❌ Don't cache these:
  Highly personalized, unique per user
    → Low hit ratio (each user = different cache key)
    → Exception: per-user cache with short TTL

  Rapidly changing, accuracy critical
    → Stock prices, inventory counts (use low TTL or skip)
    → Bank account balances (must be real-time)

  Sensitive data (unless encrypted + careful TTL)
    → PII, financial data needs careful handling
    → Cache poisoning attacks become dangerous

  Cheap to compute/fetch
    → Cache overhead > computation cost
    → Simple DB lookup on indexed PK: 1ms → not worth caching
```

---

## Cache Invalidation — The Hardest Problem

Phil Karlton famously said:

> "There are only two hard things in Computer Science: cache invalidation and naming things."

When source data changes, cached data becomes stale. You need to decide what to do.

### Strategy 1: TTL (Time-To-Live)

Simplest approach. Cache entry expires after a fixed time.

```
SET product:123 {data} EX 3600  ← expires in 1 hour

Request for product:123:
  t=0:    cache miss → fetch from DB → cache with TTL=3600
  t=1800: cache hit → return cached (30 min old)
  t=3600: TTL expires → cache miss → fetch fresh from DB

Pros:  Simple, no coordination between cache and DB
Cons:  Data can be stale for up to TTL duration
       Product price changes at t=10 → cached wrong price for 50 min

TTL selection:
  Fast-changing data:   30s - 5min TTL
  Slow-changing data:   1h - 24h TTL
  Static data:          days - weeks TTL
  Session data:         match session timeout (30min typical)
```

### Strategy 2: Cache-Aside with Active Invalidation

Explicitly delete or update cache when source data changes.

```
Write flow:
  Application updates DB
  Application immediately deletes cache key
  Next request → cache miss → fetches fresh → re-caches

DELETE product:123  ← when product is updated

Pros:  Cache always fresh after write
Cons:  Race condition possible:

       Thread A: reads DB (old value = $50)
       Thread B: updates DB ($60), deletes cache
       Thread A: writes old value ($50) to cache
       → Cache has stale value despite invalidation!

       Solution: short TTL as safety net + event-driven invalidation
```

### Strategy 3: Event-Driven Invalidation

Use events (Kafka) to propagate cache invalidation across services.

```
Product Service updates product:
  → writes to DB
  → publishes ProductUpdated event to Kafka

Cache Invalidation Consumer:
  → receives ProductUpdated event
  → deletes cache key: product:{id}
  → all service instances invalidate their local cache

Guarantees eventual consistency of all caches
Decoupled — Product Service doesn't know about caches
```

### Strategy 4: Cache-Busting with Versioning

Embed version in cache key. "Invalidation" just means old version never accessed.

```
Cache key: product:{id}:v{version}

product:123:v1 → cached data (old)
Product updated → version incremented to v2
Next request → fetches product:123:v2 (miss, old version ignored)
product:123:v1 → eventually evicted (TTL or LRU)

Used for: static assets (CSS, JS files)
  <link href="/styles.css?v=a3f8c2d1">  ← hash-based version
  CDN caches this URL
  Update CSS → new hash → new URL → CDN miss → fresh fetch
  Old URL still served from CDN (users on old page)
  New URL served fresh (users on new page)
```

---

## Cache Stampede (Thundering Herd)

A common production problem:

```
Popular product cached with TTL=3600
At t=3600: TTL expires
1000 concurrent users request product:123
All see cache miss simultaneously
All 1000 fire DB queries simultaneously
DB overwhelmed → timeouts → cascading failure
```

**Solutions:**

```
1. Probabilistic early expiration (PER):
   Before TTL expires, randomly re-fetch with increasing probability
   Some requests "proactively" refresh before expiration
   Spreads re-fetch load, avoids simultaneous expiry

2. Mutex / distributed lock:
   On cache miss, acquire Redis lock
   Only first request fetches from DB + re-caches
   Other requests wait, then read from cache

   String lockKey = "lock:product:" + id;
   if (redis.set(lockKey, "1", SetArgs.NX().EX(10))) {
       // Acquired lock, fetch from DB
       data = db.fetch(id);
       cache.set("product:" + id, data, 3600);
       redis.del(lockKey);
   } else {
       // Wait and retry (lock held by another request)
       Thread.sleep(50);
       return cache.get("product:" + id);
   }

3. Stale-while-revalidate:
   Serve stale data immediately
   Refresh cache asynchronously in background
   Next request gets fresh data
   Zero latency impact, eventual freshness
```

---

## Cache Penetration

Requests for data that doesn't exist keep missing the cache and hitting the DB.

```
Scenario:
  Attacker sends requests for product:999999, product:999998, ...
  None exist in DB
  All are cache misses (null result not cached)
  All hit DB → DB overwhelmed

Solutions:

1. Cache null results:
   DB returns null → cache "NULL" with short TTL (5 min)
   Next request for same key → cache hit, returns null
   DB not hit again for 5 minutes

   cache.set("product:999999", "NULL", 300);  ← cache the absence

2. Bloom filter:
   Maintain Bloom filter of all valid product IDs
   Before cache check: "does this product ID exist?"
   → "Definitely NOT" → return 404 immediately (no cache, no DB)
   → "Possibly" → check cache → check DB

   Eliminates DB hits for nonexistent keys entirely
```

---

## Cache Avalanche

Many cache entries expire simultaneously → massive DB load spike.

```
Scenario:
  At midnight, system pre-caches 10,000 products
  All with TTL = 3600
  At 1am, all 10,000 expire simultaneously
  All 10,000 requests miss cache → hit DB → DB overwhelmed

Solutions:

1. Jitter on TTL:
   TTL = base_ttl + random(0, base_ttl * 0.1)
   base_ttl = 3600, jitter = random(0, 360)

   Products expire spread over 60 minutes instead of same second

2. Staggered pre-warming:
   Load cache entries in batches with delays

3. Persistent cache:
   Redis with persistence (RDB/AOF)
   Cache survives restarts → no avalanche on redeploy

4. Circuit breaker on DB:
   If DB overloaded → return stale data or graceful error
   Prevent total system failure during avalanche
```

---

## Distributed Cache vs Local Cache

### Local Cache (In-Process)

Cache lives inside the application process (JVM heap).

```java
// Caffeine (best Java local cache)
Cache<String, Product> localCache = Caffeine.newBuilder()
    .maximumSize(10_000)          // max 10K entries
    .expireAfterWrite(5, MINUTES) // TTL
    .expireAfterAccess(2, MINUTES)// evict if not accessed
    .recordStats()                 // hit/miss metrics
    .build();

Product product = localCache.get(productId,
    id -> productRepository.findById(id)); // load on miss
```

```
Pros:
  ✅ Sub-millisecond (no network hop, just RAM)
  ✅ No serialization overhead
  ✅ No external dependency

Cons:
  ❌ Not shared — each instance has own cache
  ❌ Consistency issues across instances:
     Instance 1 updates product → invalidates local cache
     Instance 2, 3 still have stale copy
  ❌ Wasted memory (3 instances = 3 copies of same data)
  ❌ Lost on restart

Use for:
  Immutable/rarely-changing data (config, feature flags)
  Per-request caching (avoid duplicate DB calls in one request)
  When stale data across instances is acceptable
```

### Distributed Cache (Redis)

Cache lives in a separate shared service.

```java
// Spring Boot + Redis
@Cacheable(value = "products", key = "#id")
public Product getProduct(String id) {
    return productRepository.findById(id)
        .orElseThrow(() -> new ProductNotFoundException(id));
}

@CacheEvict(value = "products", key = "#product.id")
public Product updateProduct(Product product) {
    return productRepository.save(product);
}

@CachePut(value = "products", key = "#product.id")
public Product createProduct(Product product) {
    return productRepository.save(product);
}
```

```
Pros:
  ✅ Shared across all service instances → consistent
  ✅ Survives individual service restarts
  ✅ Rich data structures (sorted sets, lists, pub/sub)
  ✅ Large capacity (dedicated memory)
  ✅ Atomic operations

Cons:
  ❌ Network hop (~1ms vs sub-millisecond local)
  ❌ Serialization/deserialization overhead
  ❌ External dependency (Redis must be available)
  ❌ More infrastructure to operate

Use for:
  Session data (must be consistent across instances)
  Rate limiting (counters must be shared)
  Shared state (shopping cart, user permissions)
  Any data where consistency across instances matters
```

### Two-Level Cache (Best of Both)

```
Request → L1 (local Caffeine, <1ms) → hit → return
                                    → miss → L2 (Redis, 1ms)
                                             → hit → populate L1 → return
                                             → miss → DB (50ms)
                                                    → populate L2 → L1 → return

L1: hot data, very fast, small size, short TTL
L2: warm data, fast, larger size, longer TTL
DB: cold data, slow, complete

Trade-off: L1 consistency issues (stale for short TTL window)
Acceptable for: read-heavy data that can tolerate brief staleness
```

---

## Cache Metrics to Monitor

```
Hit Rate:        cache_hits / total_requests
                 Target: > 90% for hot data

Miss Rate:       cache_misses / total_requests
                 Inverse of hit rate

Eviction Rate:   how often LRU evicts data
                 High rate → cache too small for working set

Memory Usage:    current / max
                 > 90% → consider increasing size or adjusting TTL

Latency:         p50, p99 for cache get/set
                 > 5ms p99 for Redis → investigate

Expiry Pattern:  how many keys expire vs evicted
                 All expiring together → jitter TTLs
```

---

## Key Takeaways

```
Caching: Store frequently accessed data in faster storage layer
         Cache hit: sub-millisecond; DB: 50ms → 50x improvement

Caching hierarchy:
  CPU cache → RAM → Application cache → Redis → DB
  Faster but smaller at each level up

What to cache:
  Frequently read, rarely written, tolerable staleness
  Expensive to compute, shared across many users

Cache invalidation strategies:
  TTL:              simple, stale for TTL duration
  Active delete:    fresh but race condition possible
  Event-driven:     Kafka → eventual consistency
  Versioned keys:   never invalidate, just version

Cache problems and solutions:
  Stampede:    mutex lock, probabilistic early expiry, jitter
  Penetration: cache null values, Bloom filter
  Avalanche:   jitter TTL, staggered loading, circuit breaker

Local cache (Caffeine):
  Sub-ms, no network, not shared, per-instance
  Use for: immutable config, per-request dedup

Distributed cache (Redis):
  ~1ms, shared across instances, rich structures
  Use for: sessions, rate limiting, shared state

Two-level cache:
  L1 Caffeine (hot) → L2 Redis (warm) → DB (cold)
  Best performance + consistency tradeoff

Key metric: cache hit rate
  Target > 90% for frequently accessed data
```
