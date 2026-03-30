# Rate Limiting

**Phase 3 — API Fundamentals | Topic 7 of 8**

---

## What is Rate Limiting?

Rate limiting is the process of controlling how many requests a client can make to a system within a given time window. It protects your system from being overwhelmed — whether by malicious attacks, buggy clients, or legitimate users generating unexpected load.

```
Without rate limiting:
One buggy client sends 50,000 requests/second
→ Server CPU maxes out
→ All other users experience degraded service
→ Database connection pool exhausted
→ System goes down

With rate limiting:
Client sends 50,000 requests/second
→ First 1,000 pass through
→ Remaining 49,000 rejected with 429 Too Many Requests
→ Other users unaffected
→ System stays healthy
```

---

## Why Rate Limiting Is Needed

```
Security:
  Brute force attacks     → try 1M passwords per second
  DDoS attacks            → flood server with requests
  Credential stuffing     → replay breached credentials
  Web scraping            → extract entire database

Availability:
  Buggy client in retry loop → accidental self-DDoS
  Viral traffic spike        → one endpoint overwhelms others
  Expensive query repeated   → database melted

Business:
  API monetization → free tier gets 100 req/day, paid gets 10,000
  Fair usage       → one customer can't starve others
  Cost control     → LLM API calls cost money per request
```

---

## Rate Limiting Algorithms

### 1. Token Bucket

The most common and flexible algorithm. Each client has a bucket that holds tokens. Each request consumes one token. Tokens refill at a fixed rate.

```
Bucket capacity: 10 tokens  (max burst)
Refill rate:     2 tokens/second

t=0s:  Bucket has 10 tokens
       Client sends 6 requests → 6 tokens consumed → 4 remain

t=1s:  +2 tokens added → 6 tokens
       Client sends 3 requests → 3 tokens consumed → 3 remain

t=2s:  +2 tokens → 5 tokens
       Client sends 8 requests → 5 pass, 3 rejected (429)

Key properties:
✅ Allows bursting up to bucket capacity
✅ Smooth average rate enforced over time
✅ Simple to implement
✅ Memory efficient (just store token count + last refill time)
```

```java
public class TokenBucket {
    private final long capacity;        // max tokens
    private final double refillRate;    // tokens per second
    private double tokens;
    private long lastRefillTime;

    public synchronized boolean allowRequest() {
        refill();
        if (tokens >= 1) {
            tokens -= 1;
            return true;
        }
        return false; // reject
    }

    private void refill() {
        long now = System.currentTimeMillis();
        double elapsed = (now - lastRefillTime) / 1000.0;
        tokens = Math.min(capacity, tokens + elapsed * refillRate);
        lastRefillTime = now;
    }
}
```

**Redis implementation (distributed):**
```
For each client, store: {tokens, lastRefillTime} in Redis
Atomic Lua script to check + consume in one operation
(prevents race condition between check and consume)
```

---

### 2. Leaky Bucket

Requests enter a queue (the bucket) and are processed at a fixed rate — like water leaking from a bucket at constant speed. Excess overflows (rejected).

```
Queue capacity: 10 requests
Drain rate:     2 requests/second

Requests enter at any rate
Queue fills up
Requests processed at steady 2/sec
If queue full → new requests dropped

Effect: Smooths out bursts, enforces strict output rate
        No matter how fast requests arrive,
        processing rate is constant

         Requests in           Requests out
─────────────────────────────────────────────
Fast burst: ████████████  →  ██  ██  ██  ██
                              (steady 2/sec)
```

**Difference from Token Bucket:**
```
Token Bucket:  Output rate varies, can burst up to capacity
Leaky Bucket:  Output rate is always constant, no bursting
```

Use Token Bucket when you want to allow bursting. Use Leaky Bucket when you need strictly smooth output (shaping traffic to downstream service).

---

### 3. Fixed Window Counter

Divide time into fixed windows. Count requests in each window. Reject when count exceeds limit.

```
Window size: 1 minute
Limit:       100 requests per window

Minute 1 (00:00-01:00): 100 requests → last request at 00:59 ✅
Minute 2 (01:00-02:00): counter resets to 0

Problem — boundary exploit:
  00:30-01:00: 100 requests (fills window 1)
  01:00-01:30: 100 requests (fills window 2 immediately)

  200 requests in 60 seconds!
  Double the intended limit at the boundary
```

Simple to implement, but the boundary problem is a real vulnerability.

---

### 4. Sliding Window Log

Keep a log of all request timestamps. Count requests in the last N seconds.

```
Limit: 100 requests per minute

At t=75s, new request arrives:
  Look at log entries from t=15s to t=75s (last 60 seconds)
  Count = 87 → under limit → allow, add t=75 to log

At t=80s, new request arrives:
  Look at log entries from t=20s to t=80s (last 60 seconds)
  Count = 100 → at limit → reject

Precise: No boundary exploit
         Window truly slides second by second

Problem:
  Memory: must store timestamp of every request
  1000 req/min per user × 1M users = 1B log entries in memory
  Not practical at scale
```

---

### 5. Sliding Window Counter (Hybrid — Best of Both)

Combines Fixed Window's efficiency with Sliding Window's accuracy. Uses two windows with weighted interpolation.

```
Limit: 100 requests per minute
Current time: 01:15 (15 seconds into minute 2)

Previous window (00:00-01:00): 80 requests
Current window  (01:00-02:00): 30 requests so far

Weight of previous window:
  60s - 15s elapsed = 45s remaining of overlap
  Overlap fraction = 45/60 = 0.75

Estimated requests in sliding window:
  = (previous × overlap_fraction) + current
  = (80 × 0.75) + 30
  = 60 + 30 = 90

90 < 100 → allow request

This approximates sliding window with O(1) memory
Only store two counters per client, not full log
```

This is what **Redis + rate limiting libraries** typically implement — most accurate algorithm that's also practical at scale.

---

## Where to Enforce Rate Limits

```
Layer 1: API Gateway / Load Balancer (first line of defense)
  → Reject before request reaches your services
  → Cheapest rejection (no service processing)
  → Per IP, per API key, per user

Layer 2: Application Service
  → Business logic rate limits
  → "Max 5 password reset emails per hour per account"
  → "Max 3 payment attempts per card per day"
  → Fine-grained per feature

Layer 3: Database / Cache
  → Connection pool limits
  → Query rate limits on expensive operations
```

---

## Rate Limiting Keys — What You Limit By

```
By IP address:
  100 requests/min per IP
  Problem: NAT (thousands of users share one IP)
           IPv6 (easy to generate new IPs)

By API key:
  Free tier:     100 req/hour
  Pro tier:    1,000 req/hour
  Enterprise: unlimited
  Best for public APIs

By User ID:
  Authenticated requests only
  Fair per-user limits
  Survives behind proxies/NAT

By Endpoint:
  POST /login          → 5 attempts/minute (brute force protection)
  POST /payments       → 10 requests/minute (expensive operation)
  GET  /search         → 100 requests/minute (lighter)
  GET  /public/prices  → 1000 requests/minute (cacheable read)

Combined:
  (userId + endpoint + method) → granular limits per feature
```

---

## Distributed Rate Limiting

Single server rate limiting is easy — just an in-memory counter. The challenge is distributing limits across multiple service instances.

```
Problem:
  Limit: 100 req/min per user
  3 service instances, each with local counter

  User sends 90 requests → all hit Instance 1 → counter = 90
  User sends 90 more → load balanced to Instances 2 and 3
  → Instance 2 counter = 45, Instance 3 counter = 45
  → Both allow (under 100 locally)
  → User actually sent 180 requests!
  → Local counters don't work across instances

Solution: Centralized counter in Redis

Redis key:  rate_limit:{userId}:{window}
Value:      request count
TTL:        window size
Operation:  atomic INCR + check
```

```java
@Service
public class DistributedRateLimiter {

    private final RedisTemplate<String, String> redis;

    public boolean isAllowed(String userId, String endpoint, int limit) {

        String window = String.valueOf(
            System.currentTimeMillis() / 60_000); // 1-minute windows

        String key = String.format("rate_limit:%s:%s:%s",
            userId, endpoint, window);

        // Atomic increment and get
        Long count = redis.opsForValue().increment(key);

        // Set TTL on first request in window
        if (count == 1) {
            redis.expire(key, Duration.ofMinutes(2));
        }

        return count <= limit;
    }
}

// Lua script for sliding window counter (atomic)
String luaScript = """
    local key = KEYS[1]
    local now = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local limit = tonumber(ARGV[3])

    -- Remove entries outside window
    redis.call('ZREMRANGEBYSCORE', key, 0, now - window)

    -- Count entries in window
    local count = redis.call('ZCARD', key)

    if count < limit then
        redis.call('ZADD', key, now, now)
        redis.call('EXPIRE', key, window)
        return 1  -- allowed
    end
    return 0  -- rejected
""";
```

---

## Rate Limit Response Headers

Tell clients about their limits so they can adapt:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit:     1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset:     1708934400
X-RateLimit-Window:    3600

HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit:     1000
X-RateLimit-Remaining: 0
X-RateLimit-Reset:     1708934400
Retry-After:           3600
Content-Type: application/json

{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Retry after 3600 seconds.",
    "retryAfter": 3600
  }
}
```

Good clients read `Retry-After` and back off. Bad clients ignore it and get blocked longer.

---

## Rate Limiting in Spring Boot

```java
// Using Resilience4j RateLimiter
@Configuration
public class RateLimiterConfig {

    @Bean
    public RateLimiterRegistry rateLimiterRegistry() {
        RateLimiterConfig config = RateLimiterConfig.custom()
            .limitRefreshPeriod(Duration.ofMinutes(1))
            .limitForPeriod(100)
            .timeoutDuration(Duration.ofMillis(0)) // Don't wait, reject immediately
            .build();

        return RateLimiterRegistry.of(config);
    }
}

@RestController
public class SearchController {

    @GetMapping("/search")
    public ResponseEntity<SearchResult> search(
            @RequestParam String query,
            @RequestHeader("X-User-Id") String userId) {

        // Per-user rate limiter
        RateLimiter limiter = rateLimiterRegistry
            .rateLimiter("search-" + userId);

        if (!limiter.acquirePermission()) {
            return ResponseEntity
                .status(429)
                .header("Retry-After", "60")
                .body(null);
        }

        return ResponseEntity.ok(searchService.search(query));
    }
}

// Or use annotation
@RateLimiter(name = "search", fallbackMethod = "searchFallback")
@GetMapping("/search")
public SearchResult search(@RequestParam String query) {
    return searchService.search(query);
}

public SearchResult searchFallback(String query, RequestNotPermitted e) {
    throw new TooManyRequestsException("Rate limit exceeded");
}
```

### Rate Limiting at the API Gateway Level

```java
// In Spring Cloud Gateway:
@Bean
public RouteLocator routes(RouteLocatorBuilder builder,
                           RateLimiter redisRateLimiter) {
    return builder.routes()
        .route("payment-service", r -> r
            .path("/api/payments/**")
            .filters(f -> f
                .requestRateLimiter(c -> c
                    .setRateLimiter(redisRateLimiter)
                    .setKeyResolver(exchange ->
                        Mono.just(exchange.getRequest()
                            .getHeaders()
                            .getFirst("X-User-Id")))
                    .setDenyEmptyKey(true)
                    .setEmptyKeyStatus("401")))
            .uri("lb://payment-service"))
        .build();
}

@Bean
public RedisRateLimiter redisRateLimiter() {
    return new RedisRateLimiter(
        10,    // replenishRate: tokens/second
        20,    // burstCapacity: max bucket size
        1      // requestedTokens: cost per request
    );
}
```

Rate limiting at the gateway means it's enforced before any service code runs — cheapest possible rejection.

---

## Algorithm Comparison

| Algorithm | Memory | Accuracy | Burst | Best For |
|-----------|--------|----------|-------|----------|
| Token Bucket | O(1) | Good | Yes | General purpose |
| Leaky Bucket | O(n) | Good | No | Smooth output rate |
| Fixed Window | O(1) | Poor | Edge bug | Simple counting |
| Sliding Window Log | O(n) | Exact | No | Accuracy critical |
| Sliding Window Counter | O(1) | Good | No | Production default |

```
n = requests in window

Production recommendation: Sliding Window Counter
  → Best accuracy-to-memory tradeoff
  → No boundary exploit
  → Easy Redis implementation
```

---

## Key Takeaways

```
Rate Limiting: Control request rate to protect system
               and enforce fair usage

Why needed:
  Security: DDoS, brute force, credential stuffing
  Availability: runaway clients, traffic spikes
  Business: API tiers, cost control

Algorithms:
  Token Bucket           → allow bursting, general purpose ✅
  Leaky Bucket           → smooth output, no bursting
  Fixed Window           → simple but boundary exploit ❌
  Sliding Window Log     → exact but memory heavy
  Sliding Window Counter → production default ✅

Distribute with Redis:
  Centralized counter across all service instances
  Atomic INCR + TTL per window key
  Lua scripts for complex sliding window logic

Limit by: IP, API key, user ID, endpoint, or combination
Response: 429 + Retry-After + X-RateLimit-* headers

Spring Boot: Resilience4j @RateLimiter
Gateway:     Spring Cloud Gateway + Redis RateLimiter
```
