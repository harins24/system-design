# Designing a Rate Limiter

**Phase 7 — Tradeoffs + Interview Problems | Topic 3 of 8**

---

## Overview

A rate limiter is one of the most common system design questions because it's a pure infrastructure problem with interesting algorithmic, distributed systems, and accuracy tradeoffs. It also appears as a component in almost every other system design question.

---

## Phase 1: Requirements

### Functional Requirements

```
Core behavior:
  Given a request, decide: allow or reject
  Track how many requests a client has made
  Enforce a limit: N requests per time window

Questions to ask:
  "What are we rate limiting — users, IPs, API keys, all of these?"
  "Per user or globally across all users?"
  "Multiple rate limit tiers? (free: 100/hr, paid: 10,000/hr)"
  "What happens when limit exceeded? 429? Queue? Degrade?"
  "Is this client-side, server-side, or middleware (API Gateway)?"
  "Hard limit or soft limit?"

For this walkthrough:
  Rate limit by: user ID (authenticated) + IP (unauthenticated)
  Multiple tiers: free (100 req/hr), premium (10,000 req/hr)
  On limit exceeded: 429 Too Many Requests
  Server-side middleware in API Gateway
  Hard limit: no burst allowance past limit
```

### Non-Functional Requirements

```
Scale:
  10M users, 100K API requests/sec total

Latency:
  Rate limit check must add <5ms to every request
  (every request goes through this — must be fast)

Accuracy:
  Should not allow more than N+small_epsilon requests
  Small tolerance acceptable (not financial-grade)

Availability:
  If rate limiter is down → fail open (allow all) vs fail closed (deny all)
  For most APIs: fail open (availability > strict limiting)

Consistency:
  Distributed: 100 servers, all must agree on count
  Slight over-counting is acceptable
  Under-counting (allowing too many) is the main risk

Geography:
  Single region initially, global stretch goal
```

---

## Phase 2: Capacity Estimation

```
Requests to check: 100K req/sec

Storage per user:
  Sliding window log: store timestamps of each request
  → 100 requests/hr → 100 timestamps × 8 bytes = 800 bytes/user

  Counter approach: just a number + timestamp
  → 16 bytes/user

Total storage:
  10M users × 16 bytes = 160 MB for counters
  → Fits entirely in Redis RAM
  → Redis handles 100K+ ops/sec easily (single-threaded, no locks)

  10M users × 800 bytes (log approach) = 8 GB
  → Still fits in Redis, but larger

Latency target:
  100K req/sec → each check must complete in <5ms
  Redis GET/SET: <1ms
  → Redis is the right store
```

---

## Phase 3: High-Level Design

```
Request flow:
[Client] → [API Gateway]
               ↓
         [Rate Limiter Middleware]
               ↓
         [Redis] ← check and increment counter

         If limit NOT exceeded:
               ↓
         [Backend Service] → response to client

         If limit exceeded:
               → 429 Too Many Requests
                 Headers: Retry-After: 3600
                          X-RateLimit-Limit: 100
                          X-RateLimit-Remaining: 0
                          X-RateLimit-Reset: 1707912000

Rate limit rule storage (separate from counter):
  [Config Service / Database] → rate limit rules per tier
  Cached in middleware memory (rules don't change often)

  Rules example:
  {tier: "free",    limit: 100,    window: 3600}  // 100/hr
  {tier: "premium", limit: 10000,  window: 3600}  // 10K/hr
  {tier: "api-key", limit: 1000,   window: 60}    // 1K/min
```

---

## Phase 4: Deep Dives

### Deep Dive 1: The Algorithms

#### Algorithm 1: Fixed Window Counter

```
Divide time into fixed windows:
  Window = each hour (0:00-1:00, 1:00-2:00, ...)

  Key: "ratelimit:{userId}:{windowStart}"
  Value: count of requests in this window

Example (limit: 5 per hour):
  12:00:01  request → key "user:123:12" = 1  → allow
  12:59:30  request → key "user:123:12" = 5  → allow
  12:59:55  request → key "user:123:12" = 6  → DENY ✅
  13:00:01  request → key "user:123:13" = 1  → allow (new window!)
```

```java
public boolean isAllowed(String userId, int limit) {
    long windowStart = Instant.now().getEpochSecond() / 3600 * 3600;
    String key = "ratelimit:" + userId + ":" + windowStart;

    Long count = redis.opsForValue().increment(key);
    if (count == 1) {
        redis.expire(key, Duration.ofHours(2)); // cleanup after 2 windows
    }

    return count <= limit;
}
```

**The Critical Flaw — Boundary Spike:**
```
Limit: 5 requests per hour

12:59:00 → 5 requests → all allowed (window 12:xx used up)
13:00:00 → 5 requests → all allowed (new window 13:xx resets)

In a 2-minute window (12:59-13:01): 10 requests allowed
= 2x the intended limit

Attackers can exploit this deliberately
```

---

#### Algorithm 2: Sliding Window Log

```java
public boolean isAllowed(String userId, int limit, long windowSeconds) {
    long now = Instant.now().toEpochMilli();
    long windowStart = now - (windowSeconds * 1000);
    String key = "ratelimit:" + userId;

    // Atomic Lua script: add + remove old + count
    String script = """
        local key = KEYS[1]
        local now = tonumber(ARGV[1])
        local window_start = tonumber(ARGV[2])
        local limit = tonumber(ARGV[3])

        redis.call('ZADD', key, now, now)
        redis.call('ZREMRANGEBYSCORE', key, 0, window_start)
        local count = redis.call('ZCARD', key)
        redis.call('EXPIRE', key, 7200)

        if count <= limit then
            return 1  -- allowed
        else
            return 0  -- denied
        end
        """;

    Long result = redis.execute(
        new DefaultRedisScript<>(script, Long.class),
        List.of(key),
        String.valueOf(now), String.valueOf(windowStart), String.valueOf(limit)
    );

    return result == 1L;
}
```

```
✅ Exact: no boundary spike
✅ Accurate per-user tracking

❌ High memory: store every request timestamp
   1M users × 100 requests = 100M entries in sorted sets
❌ More Redis operations per check
```

---

#### Algorithm 3: Sliding Window Counter (Hybrid — Recommended)

```
Combines fixed window's efficiency with sliding window's accuracy.
Maintain counters for current and previous windows.
Estimate current window's count with weighted average.

Formula:
  current_count =
    previous_window_count × (1 - elapsed_ratio) + current_window_count

Example (limit: 100/hr, window = 1 hour):
  Previous window: 80 requests
  Current window: 40 requests (started 30 minutes ago)
  Elapsed ratio: 0.5 (30min / 60min)

  Estimated count = 80 × (1 - 0.5) + 40 = 40 + 40 = 80
  80 < 100 → allow
```

```java
public boolean isAllowed(String userId, int limit, long windowSeconds) {
    long now = Instant.now().getEpochSecond();
    long currentWindowStart = now / windowSeconds * windowSeconds;
    long previousWindowStart = currentWindowStart - windowSeconds;

    String currentKey  = "ratelimit:" + userId + ":" + currentWindowStart;
    String previousKey = "ratelimit:" + userId + ":" + previousWindowStart;

    String script = """
        local current_count = tonumber(redis.call('GET', KEYS[1]) or 0)
        local previous_count = tonumber(redis.call('GET', KEYS[2]) or 0)

        local elapsed_ratio = (tonumber(ARGV[1]) - tonumber(ARGV[2])) / tonumber(ARGV[3])
        local estimated = previous_count * (1 - elapsed_ratio) + current_count

        if estimated < tonumber(ARGV[4]) then
            redis.call('INCR', KEYS[1])
            redis.call('EXPIRE', KEYS[1], tonumber(ARGV[3]) * 2)
            return 1  -- allowed
        end
        return 0  -- denied
        """;

    Long result = redis.execute(
        new DefaultRedisScript<>(script, Long.class),
        List.of(currentKey, previousKey),
        String.valueOf(now), String.valueOf(currentWindowStart),
        String.valueOf(windowSeconds), String.valueOf(limit)
    );

    return result == 1L;
}
```

```
✅ Memory efficient: 2 counters per user (not full log)
✅ Approximates sliding window: boundary spikes reduced ~10x
✅ Fast: 2 Redis ops + math

❌ Approximate (~10% over-limit possible at boundary)
   In practice: best general-purpose algorithm
```

---

#### Algorithm 4: Token Bucket

```
Bucket holds N tokens. Each request consumes 1 token.
Tokens refilled at fixed rate (R tokens per second).
If bucket empty: request denied.

Example (10 tokens, refill 1/sec):
  t=0:    bucket = 10 tokens
  Burst: 10 requests → bucket = 0 → all allowed
  t=1:    +1 token → bucket = 1
  Then: sustained rate of 1/sec

Allows controlled burst then steady rate
```

```
✅ Handles bursts naturally (tokens accumulate during quiet periods)
✅ Smooth rate limiting over time
✅ Memory efficient: 2 values per user

❌ Parameters harder to reason about (capacity vs refill rate)
```

---

#### Algorithm 5: Leaky Bucket

```
Requests go into a queue (bucket).
Queue drains at fixed rate.
If bucket full: request dropped.

vs Token Bucket:
  Token bucket: allow burst now
  Leaky bucket: smooth output regardless of burst input

✅ Perfectly smooth output rate
✅ Protects downstream with even load

❌ Requests may wait in queue (adds latency)
❌ Harder to implement correctly in distributed setting

Used by: traffic shaping at network level
         Less common for API rate limiting
```

---

### Algorithm Comparison

```
Algorithm            Memory     Accuracy    Burst    Use when
─────────────────────────────────────────────────────────────
Fixed Window         Low        Low         None     Simple, high scale
Sliding Window Log   High       Exact       None     Exact accuracy needed
Sliding Window Ctr   Low        ~Exact      None     Best general purpose
Token Bucket         Low        Good        Yes      API rate limiting
Leaky Bucket         Medium     Exact rate  No       Traffic shaping

Recommendation: Token Bucket for APIs (allows reasonable burst)
                Sliding Window Counter when burst should not be allowed
```

---

### Deep Dive 2: Distributed Rate Limiting

```
Problem: 100 API Gateway instances, each has local counter.
  User sends 100 requests:
  10 to server 1: counter = 10
  10 to server 2: counter = 10
  ...
  All 100 allowed! Limit of 100 is 10x exceeded.

Solutions:

Option 1: Centralized Redis
  All servers check and update ONE Redis counter

  ✅ Exactly consistent
  ❌ Redis becomes bottleneck (every request hits Redis)
  At 100K req/sec: Redis handles this fine

Option 2: Local counter + periodic sync
  Each server maintains local counter
  Periodically (every 1 second) sync to Redis

  ✅ Reduces Redis pressure significantly
  ❌ Up to 1 second of over-counting possible
  Good for: high-scale, small over-limit acceptable

Option 3: Redis Cluster with hash slots
  hash(userId) → specific Redis node
  All requests for same user → same Redis node

  ✅ Scales well, consistent per user
  ❌ Hot users → hot Redis nodes
```

### Deep Dive 3: Atomicity

```
Race condition without atomic operations:

Server 1: GET counter → 99
Server 2: GET counter → 99
Server 1: counter < 100 → allow → SET counter 100
Server 2: counter < 100 → allow → SET counter 100
→ Two requests allowed when only one should be!

Solution: Redis Lua scripts execute atomically
  No interleaving between GET, check, and SET
  All algorithms above use Lua scripts for this reason
```

### Deep Dive 4: Rate Limit Response Headers

```java
@Component
public class RateLimitFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain chain) throws Exception {
        String userId = extractUserId(request);
        RateLimitResult result = rateLimiter.check(userId);

        response.setHeader("X-RateLimit-Limit", String.valueOf(result.getLimit()));
        response.setHeader("X-RateLimit-Remaining", String.valueOf(result.getRemaining()));
        response.setHeader("X-RateLimit-Reset", String.valueOf(result.getResetTime()));

        if (!result.isAllowed()) {
            response.setHeader("Retry-After", String.valueOf(result.getRetryAfter()));
            response.setStatus(429);
            response.getWriter().write("{\"error\": \"Rate limit exceeded\"}");
            return;
        }

        chain.doFilter(request, response);
    }
}
```

---

## Phase 5: Tradeoffs

```
1. Algorithm choice:
   Sliding Window Counter chosen
   Trade-off: ~10% over-limit possible at boundary
   Accepted: accuracy vs memory/performance

2. Centralized Redis:
   Trade-off: Redis is critical path for every request
   Mitigation: Redis Cluster + circuit breaker
   Fail-open on Redis outage (allow traffic, alert)

3. Fail-open vs fail-closed:
   Fail-open: if rate limiter down → all traffic allowed
   Better for availability-critical APIs

   Fail-closed: if rate limiter down → all denied
   Better when security/cost is priority

4. Global vs per-region:
   Global: accurate worldwide count, cross-region latency
   Per-region: fast, may allow 2x if user switches regions
   Choice: per-region for performance, global for billing-critical

5. Granularity:
   Per-user: fair, user can't be affected by others
   Per-IP: protects unauthenticated endpoints
   Per-API-key: for developer APIs
   Global: protect from total overload
   Real systems combine: per-user AND global limit
```

---

## Interview Answer Skeleton

> "A rate limiter sits in front of every API request and decides allow or deny based on how many requests a client has made recently.
>
> The core algorithm question is the most interesting part. I'd use the Sliding Window Counter: maintain counters for the current and previous window, estimate the weighted count. This gives near-exact accuracy with minimal memory — just two integers per user rather than a full timestamp log.
>
> For storage, Redis is the right choice. It handles 100K+ ops/sec with sub-millisecond latency, and its Lua scripting enables atomic check-and-increment to prevent race conditions.
>
> The hard distributed systems problem: 100 API gateway instances all sharing one rate limiter. All servers point to the same Redis cluster. userId hashes to a specific Redis node so all requests for the same user go to the same node.
>
> The key tradeoffs: Centralized Redis is a critical dependency — I'd use a circuit breaker and fail-open on Redis outage. Sliding Window Counter allows ~10% over-limit at window boundaries — acceptable for API rate limiting but not for billing or security-critical operations."
