# Distributed Caching

**Phase 5 — Caching + Async Communication | Topic 4**

---

## What is Distributed Caching?

A distributed cache is a cache that spans multiple nodes, appearing as a single unified cache to the application while physically storing data across a cluster of machines.

```
Single-node cache:
  [App] → [Redis Single Node: 16GB RAM]
  Limit: one machine's memory
  SPOF: node dies → cache gone → DB overwhelmed

Distributed cache:
  [App Servers] → [Redis Cluster]
                     ├── Node 1: 16GB (slots 0-5460)
                     ├── Node 2: 16GB (slots 5461-10922)
                     └── Node 3: 16GB (slots 10923-16383)
  Total: 48GB across 3 nodes
  Resilient: node dies → other nodes continue
  Scalable: add nodes → more capacity
```

---

## Why Distributed Caching is Necessary

```
Single-node limits:
  Memory ceiling:    one machine = max ~1-2TB RAM (expensive)
  Throughput ceiling: one CPU = max ~100K ops/sec
  SPOF:              one node = one point of failure

At scale:
  Instagram:  billions of photos → metadata can't fit one node
  Twitter:    timeline caches for 300M users
  Netflix:    movie metadata for 200M subscribers

  These require distributed caches:
  Petabytes of cache memory
  Millions of operations/second
  Zero tolerance for cache-wide failure
```

---

## Redis Cluster — The Standard

Redis Cluster is the built-in distributed caching solution that shards data across nodes using consistent hashing (hash slots).

### Hash Slots

Redis Cluster divides the key space into 16,384 hash slots.

```
Hash slot assignment:
  slot = CRC16(key) % 16384

Default distribution across 3 nodes:
  Node 1: slots 0     – 5,460
  Node 2: slots 5,461 – 10,922
  Node 3: slots 10,923 – 16,383

Key routing:
  key = "product:123"
  CRC16("product:123") % 16384 = 7832
  7832 is in range 5,461–10,922 → Node 2
  Client connects directly to Node 2
```

### Topology

Each master node has replica(s) for failover:

```
[Master 1] ←replica→ [Replica 1]   slots 0-5460
[Master 2] ←replica→ [Replica 2]   slots 5461-10922
[Master 3] ←replica→ [Replica 3]   slots 10923-16383

Failure handling:
  Master 1 fails
  → Replica 1 promoted to master automatically
  → Cluster continues serving all 16,384 slots
  → No data lost (replica was in sync)

Minimum production cluster: 3 masters + 3 replicas = 6 nodes
```

### Hash Tags — Controlling Key Placement

```
Problem: MGET across multiple keys may hit different nodes
         Multi-key operations require all keys on same node

Solution: Hash tags force keys to same slot

Hash tag: the part of the key in {}
  Slot = CRC16(content inside {}) % 16384

Examples:
  {user:123}:profile  → slot = CRC16("user:123") % 16384
  {user:123}:sessions → slot = CRC16("user:123") % 16384
  {user:123}:orders   → slot = CRC16("user:123") % 16384

  All three on same node → MGET, transactions work

Without hash tag:
  user:123:profile   → slot = CRC16("user:123:profile") % 16384 = node 1
  user:123:sessions  → slot = CRC16("user:123:sessions") % 16384 = node 3
  Can't do atomic operations across them
```

### Redis Cluster — Spring Boot Integration

```java
@Configuration
public class RedisClusterConfig {

    @Bean
    public LettuceConnectionFactory redisConnectionFactory() {
        RedisClusterConfiguration clusterConfig =
            new RedisClusterConfiguration(List.of(
                "redis-node1:6379",
                "redis-node2:6379",
                "redis-node3:6379"
            ));

        clusterConfig.setMaxRedirects(3); // follow MOVED redirects

        LettuceClientConfiguration clientConfig =
            LettuceClientConfiguration.builder()
                .readFrom(ReadFrom.REPLICA_PREFERRED) // reads from replica
                .build();

        return new LettuceConnectionFactory(clusterConfig, clientConfig);
    }
}

@Service
public class ProductCacheService {

    private final RedisTemplate<String, Product> redis;

    // Hash tags ensure all user data on same node
    public void cacheUserData(String userId,
                               Product product,
                               Session session) {
        String prefix = "{user:" + userId + "}";

        redis.opsForValue().set(prefix + ":product", product, Duration.ofHours(1));
        redis.opsForValue().set(prefix + ":session", session, Duration.ofHours(1));

        // These can now be fetched atomically (same node)
        redis.executePipelined((RedisCallback<?>) connection -> {
            connection.get((prefix + ":product").getBytes());
            connection.get((prefix + ":session").getBytes());
            return null;
        });
    }
}
```

---

## Cache Partitioning Strategies

How data is distributed across cache nodes:

### 1. Range Partitioning

```
Keys divided by ranges:
  A-H → Node 1
  I-P → Node 2
  Q-Z → Node 3

Problem: uneven distribution
  Most keys start with common letters
  Node 1 might handle 60% of load
```

### 2. Hash Partitioning (Consistent Hashing)

```
node = hash(key) % num_nodes

Problem: adding nodes remaps most keys
Solution: consistent hashing ring
```

### 3. Consistent Hashing (Production Standard)

```
Nodes placed on hash ring at multiple positions (virtual nodes)
Key routes to first node clockwise

Add node: only ~1/N keys move
Remove node: only that node's keys move to next

Used by: Redis Cluster (via hash slots), Memcached (client-side)
```

---

## Cache Coherence — The Hard Problem

With multiple cache nodes (and multiple app instances with local caches), keeping all caches consistent is the hardest distributed caching problem.

### The Problem

```
3 app instances, each with local cache:

[App 1] local cache: product:123 = {price: $50}
[App 2] local cache: product:123 = {price: $50}
[App 3] local cache: product:123 = {price: $50}

Admin updates product:123 price to $60:
  → Writes to DB ($60)
  → Invalidates cache on App 1 (App 1 sent the request)
  → App 2 and App 3 still have $50 in local cache
  → Users on App 2 and App 3 see wrong price until TTL expires
```

### Solutions

**Solution 1: Short TTL (simple, always applicable)**

```
Local cache TTL = 30 seconds
Max staleness = 30 seconds
Acceptable for many use cases
Tradeoff: more cache misses (more DB load)
```

**Solution 2: Pub/Sub Invalidation**

```
When any instance updates data:
  1. Write to DB
  2. Publish invalidation message to Redis Pub/Sub

All instances subscribe to invalidation channel:
  Receive message → delete from local cache

[App 1] publishes: INVALIDATE product:123
  → [App 2] receives → deletes local product:123
  → [App 3] receives → deletes local product:123
  → [App 1] already knows, deletes own copy

Next request on any instance → cache miss → fetch fresh → re-cache
```

```java
@Service
public class CacheInvalidationListener {

    private final Cache<String, Product> localCache;

    @PostConstruct
    public void subscribe() {
        redisTemplate.execute(new SubscribeCallback(
            (message, pattern) -> {
                String key = new String(message.getBody());
                localCache.invalidate(key); // clear local cache
            },
            "cache:invalidate".getBytes()
        ));
    }
}

// When updating product:
public void updateProduct(Product p) {
    db.save(p);
    localCache.invalidate("product:" + p.getId());
    // Notify all other instances
    redis.convertAndSend("cache:invalidate", "product:" + p.getId());
}
```

**Solution 3: Write-Through to Distributed Cache Only (No Local Cache)**

```
Use only Redis (distributed) — no local cache
All instances share same Redis cache
Single invalidation point — no coherence problem

Trade-off: ~1ms Redis latency vs sub-ms local cache
For most apps: acceptable
```

---

## Replication and High Availability

A distributed cache must survive node failures without a full cache miss storm.

### Redis Sentinel

For single-master Redis with automatic failover:

```
Architecture:
  [Redis Master] → [Replica 1]
                → [Replica 2]

  [Sentinel 1]   [Sentinel 2]   [Sentinel 3]

Sentinels monitor master health
If master unreachable for 30 seconds:
  Sentinels vote (quorum = 2)
  Elected replica promoted to master
  Other replica repoints to new master
  Application connections redirect via Sentinel API
```

```java
@Bean
public LettuceConnectionFactory redisConnectionFactory() {
    RedisSentinelConfiguration sentinelConfig =
        new RedisSentinelConfiguration("mymaster", Set.of(
            "sentinel1:26379",
            "sentinel2:26379",
            "sentinel3:26379"
        ));
    return new LettuceConnectionFactory(sentinelConfig);
}
// Application transparently follows master to new node
```

### Redis Cluster Failover

```
Redis Cluster has built-in automatic failover:
  Master dies → replica detects (no PING response)
  Replica broadcasts to cluster: "master is down"
  Other masters vote (majority needed)
  Replica promoted → starts accepting writes
  Cluster topology updated

Typically completes in 10-30 seconds
Application retries handle the brief window
```

---

## Cache Warmup — Cold Start Problem

A fresh cache has 0% hit rate. Every request hits the DB — potential disaster.

```
Scenario:
  Deploy new service → empty Redis cache
  Traffic arrives immediately
  100% cache miss rate
  DB receives full production load
  DB may not be sized to handle this without cache
  → Outage on deployment

Solutions:

1. Pre-warming (proactive):
   Before switching traffic:
   → Run warmup job: read most popular keys, populate cache
   → "Top 1000 products" → pre-cached before traffic arrives

   // Warmup job runs before deployment completes
   @Component
   public class CacheWarmupRunner implements ApplicationRunner {
       @Override
       public void run(ApplicationArguments args) {
           List<Product> popular = db.findTop1000ByAccessCountDesc();
           popular.forEach(p -> cache.set("product:" + p.getId(), p, Duration.ofHours(1)));
           log.info("Cache warmed with {} items", popular.size());
       }
   }

2. Shadow traffic (gradual):
   Route small % of traffic to new cache first
   Cache warms gradually before full cutover
   Blue-green deployment with traffic shifting

3. Lazy warming (accept cold start):
   Accept 100% miss rate initially
   Cache fills naturally as traffic flows
   DB survives because service starts with limited traffic
   Use circuit breaker to protect DB if miss rate spikes

4. Persistent cache (Redis AOF):
   Redis writes every operation to AOF log
   On restart: replay AOF to restore cache state
   Cache survives restarts fully warm
```

---

## Distributed Cache Patterns

### Cache-as-a-Service

```
Multiple microservices sharing a single Redis cluster:
  Order Service    → cache:orders:*
  Product Service  → cache:products:*
  User Service     → cache:users:*

Problems:
  One service fills cache → evicts other services' data
  One service causes hotspot → affects all services
  Noisy neighbor problem

Solution: Separate Redis instances per service (or namespaces with quotas)
  Order Service    → Redis cluster 1 (maxmemory 4GB)
  Product Service  → Redis cluster 2 (maxmemory 8GB)
  User Service     → Redis cluster 3 (maxmemory 2GB)
```

### Read-Aside with Fallback

```java
public Product getProduct(String id) {
    try {
        // Try cache first
        Product cached = redis.opsForValue().get("product:" + id);
        if (cached != null) return cached;

        // Cache miss — try DB
        Product product = db.findById(id).orElseThrow();

        try {
            // Populate cache (best effort — don't fail if Redis is down)
            redis.opsForValue().set("product:" + id, product, Duration.ofHours(1));
        } catch (RedisException e) {
            log.warn("Failed to populate cache for product {}", id, e);
            // Continue — product still returned from DB
        }

        return product;

    } catch (RedisException e) {
        // Redis completely down — fall back to DB directly
        log.error("Redis unavailable, falling back to DB", e);
        return db.findById(id).orElseThrow();
    }
}
```

**The cache must degrade gracefully.** If Redis is down, the system should continue with higher latency, not fail completely.

---

## Geo-Distributed Caching

For global systems serving users across multiple regions:

```
US users  → US Redis Cluster
EU users  → EU Redis Cluster
AP users  → AP Redis Cluster

Each cluster caches region-relevant data
Write flow:
  EU user updates profile
  → Write to EU DB (primary)
  → Publish event to Kafka
  → US and AP clusters invalidate their copy
  → Next read in each region re-caches from local DB replica

Benefits:
  ✅ Cache served from nearby node (<5ms)
  ✅ No cross-region cache reads
  ✅ Region isolation (EU outage doesn't affect US cache)

Challenges:
  ❌ Cross-region invalidation eventual consistency
  ❌ More infrastructure
  ❌ Data residency requirements per region (GDPR)
```

---

## Performance Characteristics

```
Operation         Single Redis    Redis Cluster    Local Cache
─────────────────────────────────────────────────────────────
GET               <1ms           <1ms             <0.1ms
SET               <1ms           <1ms             <0.1ms
MGET (same node)  <1ms           <1ms             N/A
MGET (cross-node) N/A            cluster error    N/A
Max memory        1 machine      N machines       JVM heap
Max throughput    ~100K/sec      ~N × 100K/sec    millions/sec
Failover          Sentinel/Cluster automatic      restart needed
Data persistence  RDB + AOF      RDB + AOF        none (typically)
```

---

## Key Takeaways

```
Distributed Cache: Cache spanning multiple nodes
  Single cache view, distributed storage
  Memory: N × single node capacity
  Throughput: N × single node throughput
  Resilience: node failure → cluster continues

Redis Cluster:
  16,384 hash slots across masters
  Each master has replicas for failover
  Hash tags: force related keys to same node
  Automatic failover: 10-30 seconds

Partitioning:
  Consistent hashing → minimal key movement on scale
  Virtual nodes → even distribution

Cache coherence:
  Multiple local caches → staleness problem
  Solutions: short TTL, pub/sub invalidation,
             shared Redis only (no local cache)

High availability:
  Redis Sentinel: single master + auto-failover
  Redis Cluster:  built-in sharding + failover

Cache warmup:
  Pre-warm before traffic arrives
  Persist with AOF for restart warmth
  Accept cold start with circuit breaker on DB

Graceful degradation:
  Redis down → fall back to DB (not fail)
  Cache is enhancement, not hard dependency
  Wrap all cache operations in try/catch

Geo-distribution:
  Regional clusters per user region
  Kafka-based cross-region invalidation
  Sub-5ms cache reads for all global users
```
