# Designing a URL Shortener

**Phase 7 — Tradeoffs + Interview Problems | Topic 2**

---

## Overview

This is the most common warm-up system design question. It's deceptively simple but reveals whether you can think systematically about even straightforward problems.

---

## Phase 1: Requirements

### Functional Requirements

```
Core features:
  1. Given a long URL, generate a short URL
     https://www.example.com/some/very/long/path → https://short.ly/abc123

  2. Given a short URL, redirect to the original long URL

  3. (Optional, confirm scope):
     Custom aliases: user chooses short code (short.ly/mylink)
     Expiration: links expire after N days
     Analytics: track click count, referrer, geography

Questions to ask:
  "Should users be able to create custom short codes?"
  "Should links expire?"
  "Do we need click analytics?"
  "Is this public (anyone can shorten) or authenticated?"

For this walkthrough: Yes to custom aliases, Yes to expiration,
                       Basic analytics (click count), authenticated users
```

### Non-Functional Requirements

```
Scale:
  "100M DAU, 1B URLs shortened to date"
  100M DAU clicking links: how many redirects/day?
    Average user clicks 5 short links/day
    100M × 5 = 500M clicks/day = ~5,800 clicks/sec

  New URLs created:
    100M users, maybe 0.1% create a URL each day
    = 100,000 new URLs/day = ~1.2 writes/sec

  Read:write ratio = 5,800 : 1.2 ≈ 5,000:1
  This is extremely read-heavy

Latency:
  Redirect must be fast: <10ms P99
  URL creation: <100ms acceptable

Availability:
  99.99% → 52 minutes downtime/year
  Redirect outage = broken links everywhere → unacceptable

Durability:
  URLs must never be lost

Consistency:
  New URL available within a few seconds is fine (eventual)
  Redirect must always return same destination (strong, immutable)
```

---

## Phase 2: Capacity Estimation

```
Reads (redirects):
  5,800 redirects/sec average
  Peak (2x): ~12,000 redirects/sec

Writes (URL creation):
  1.2 new URLs/sec (barely anything)

Storage:
  1B URLs in total
  Per URL: shortCode (7 bytes) + longURL (200 bytes avg) +
           userId (8 bytes) + createdAt (8 bytes) +
           expiresAt (8 bytes) = ~231 bytes
  1B × 231 bytes = 231 GB total
  → Fits in a single large DB, but sharding is wise for redundancy

Bandwidth:
  Redirect response is just HTTP 302 + Location header: ~500 bytes
  12,000 × 500 bytes = 6 MB/sec outbound — trivial

Cache sizing:
  80/20 rule: 20% of URLs drive 80% of traffic
  1B × 0.20 = 200M hot URLs
  200M × 231 bytes = 46 GB hot set
  → Fits in Redis cluster

Conclusion:
  Read-heavy (5,000:1) → aggressive caching essential
  Storage small enough → no sharding required initially
  Redirect latency critical → cache must be primary path
```

---

## Phase 3: High-Level Design

### Key APIs

```
POST /api/urls
  Request:  { longUrl, customAlias?, expiresInDays? }
  Response: { shortCode, shortUrl, expiresAt }

GET /{shortCode}
  Response: HTTP 302 redirect to longUrl
            (or 404 if not found, 410 Gone if expired)

GET /api/urls/{shortCode}/stats
  Response: { clicks, createdAt, expiresAt, topReferrers }
```

### System Diagram

```
Write path:
[Client]
   → [API Gateway (auth, rate limiting)]
   → [URL Shortener Service]
   → [ID Generator]          ← generates unique shortCode
   → [PostgreSQL]            ← stores URL mapping
   → [Kafka]                 ← publishes UrlCreated event
   → returns shortUrl

Read path (redirect):
[Browser] → [DNS] → [Load Balancer]
   → [Redirect Service]
   → [Redis Cache]           ← check cache first
   (cache hit: 95%)          → return 302 redirect immediately
   (cache miss: 5%)          → [PostgreSQL read replica]
                             → populate cache → return 302 redirect

   Asynchronous click tracking:
   [Redirect Service] → [Kafka: click-events]
   → [Analytics Service] → [ClickHouse / TimescaleDB]

Expiration:
   [Cleanup Job (nightly)] → scan expired URLs → delete from DB + cache
```

---

## Phase 4: Deep Dives

### Deep Dive 1: Short Code Generation

This is the core technical challenge. How do you generate the short code?

**Option 1: Random + Check**

```
Generate random 7-character alphanumeric string
Check if already exists in DB
If exists: generate another
If not: use it

Characters: [a-z A-Z 0-9] = 62 characters
7-character codes: 62^7 = 3.5 trillion combinations

Probability of collision at 1B URLs:
  1B / 3.5T = 0.03% chance of collision

✅ Simple to implement
✅ Unpredictable codes (privacy)
❌ DB check on every creation (extra round trip)
❌ Collision probability grows as DB fills
❌ Race condition: two services check, both get "not found", both insert
   → Handle with UNIQUE constraint + retry on violation
```

**Option 2: Hash of Long URL**

```
MD5 or SHA-256 the long URL → take first 7 characters

"https://example.com/path"
→ MD5 → "5f4dcc3b5aa765d61d8327de..."
→ take first 7: "5f4dcc3"

✅ Deterministic: same URL → same code
   (deduplication: two users shorten same URL → same short code)
✅ No DB check needed
❌ Hash collision: two different URLs → same first 7 chars
   → Must still check DB and append suffix on collision
❌ Enumerable: attacker can predict patterns
❌ Can't have multiple short codes for same URL (one-to-one mapping)
```

**Option 3: Auto-increment ID + Base62 Encoding (Best)**

```
Maintain auto-incrementing counter
Encode integer as base62 string

ID 1        → "1"
ID 62       → "Z" (or "10" in base62)
ID 12345678 → "FpGH" (7 chars covers up to 62^7 = 3.5T)

Base62 encoding:
  Characters: 0-9, a-z, A-Z
  ID 12345678:
    12345678 % 62 = 16 → 'g'
    12345678 / 62 = 199,123
    199123 % 62 = 47 → 'V'
    ...continue until 0

✅ Guaranteed unique (no collisions)
✅ No DB check needed
✅ Short (7 chars handles 3.5T URLs)
✅ Fast
❌ Sequential: attacker can enumerate all URLs (abc000, abc001...)
❌ Need distributed counter (single point of failure risk)

Solution for distributed counter:
  Database auto-increment: simple, works for moderate scale
  Redis INCR: atomic counter, fast, persistent
  Snowflake ID: distributed unique ID (covered in Phase 1)
```

```java
// Base62 encoding
public class Base62Encoder {
    private static final String CHARS =
        "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ";

    public String encode(long id) {
        StringBuilder sb = new StringBuilder();
        while (id > 0) {
            sb.append(CHARS.charAt((int)(id % 62)));
            id /= 62;
        }
        return sb.reverse().toString();
    }

    public long decode(String shortCode) {
        long id = 0;
        for (char c : shortCode.toCharArray()) {
            id = id * 62 + CHARS.indexOf(c);
        }
        return id;
    }
}

// Redis atomic counter
@Service
public class ShortCodeGenerator {

    private final RedisTemplate<String, Long> redis;
    private final Base62Encoder encoder;

    public String generate() {
        // Atomic increment: no race conditions
        Long id = redis.opsForValue().increment("url:counter");
        return encoder.encode(id);
    }
}
```

---

### Deep Dive 2: Redirect — 301 vs 302

```
Two HTTP redirect options:

301 Permanent Redirect:
  Browser caches the redirect
  Future visits to shortUrl go directly to longUrl
  (browser never contacts our servers again)

  ✅ Zero server load after first redirect (cached in browser)
  ✅ Fastest possible (no network call)
  ❌ Analytics broken: clicks after caching never counted
  ❌ Can't update destination (cached permanently)

302 Temporary Redirect:
  Browser does NOT cache the redirect
  Every click hits our servers

  ✅ Analytics accurate (every click tracked)
  ✅ Can update destination URL
  ❌ Every click = server load
  ❌ Slightly slower (extra network round trip)

Which to use?
  If analytics matter → 302 (always track clicks)
  If analytics not needed → 301 (free CDN behavior)

Most URL shorteners use 302 (analytics is the revenue model)

With Redis cache:
  302 latency impact minimized
  Cache hit (<1ms) + 302 response ≈ same UX as cached 301
```

---

### Deep Dive 3: Custom Aliases

```
User wants: short.ly/my-company instead of short.ly/k8j4m2

Storage:
  custom_aliases table: {alias, shortCode}
  Or just store custom alias AS the shortCode directly

Conflict resolution:
  "my-company" already taken → return error
  Let user choose different alias

Validation:
  Max length: 20 characters
  Allowed chars: [a-z 0-9 - _]
  Forbidden: reserved words (api, admin, www, health...)

  // Check forbidden words
  private static final Set<String> RESERVED = Set.of(
      "api", "admin", "www", "health", "static", "cdn");

  if (RESERVED.contains(alias.toLowerCase())) {
      throw new InvalidAliasException("Reserved word: " + alias);
  }

Race condition:
  Two users simultaneously try to claim "my-company"
  → UNIQUE constraint on alias column in DB
  → First INSERT wins, second gets constraint violation
  → Return "alias already taken" to second user
```

---

### Deep Dive 4: Expiration Handling

```
URLs have optional expiration dates.

On read (redirect):
  Check expiresAt in cache/DB
  If expired → return 410 Gone (not 404)

Proactive cleanup (nightly job):
  DELETE FROM urls WHERE expires_at < NOW()
  Run during off-peak hours
  → Keeps DB small, reduces storage cost

Cache handling:
  Set Redis TTL = URL expiration time
  Expired URL → Redis auto-evicts → DB also expired → 410 Gone

  // Set cache with TTL matching expiration
  Duration ttl = Duration.between(Instant.now(), url.getExpiresAt());
  redis.opsForValue().set(shortCode, longUrl, ttl);

  // Or if no expiration:
  redis.opsForValue().set(shortCode, longUrl, Duration.ofDays(30));
```

---

### Deep Dive 5: Analytics

```
Track per-click: timestamp, shortCode, userId (if logged in),
                 IP (for geo), referrer, user-agent

Scale:
  5,800 clicks/sec → 500M click events/day
  Too many writes for PostgreSQL directly

Architecture:
  Redirect Service → Kafka: click-events (async, non-blocking)
  Analytics Consumer → aggregate in batches → ClickHouse

  ClickHouse: columnar DB optimized for analytics
  Query: "clicks by day for last 30 days" → milliseconds

  Don't slow down redirect path for analytics:
  Response sent to user → THEN publish to Kafka
  Or: use fire-and-forget (don't wait for Kafka ACK)

  // Non-blocking analytics
  public String redirect(String shortCode) {
      String longUrl = getUrl(shortCode); // fast path

      // Fire and forget — don't block redirect response
      CompletableFuture.runAsync(() ->
          kafkaTemplate.send("click-events",
              new ClickEvent(shortCode, request)));

      return longUrl; // respond immediately
  }
```

---

## Phase 5: Tradeoffs and Scaling

```
Known tradeoffs in this design:

1. Sequential IDs → URL enumeration
   Short codes are incrementing → attacker can enumerate all URLs
   Trade-off accepted: most URL shorteners accept this
   Mitigation: private URLs require auth to redirect
   Alternative: add random salt to encoding (slight complexity)

2. Redis as primary cache → eventual consistency on creation
   New URL may not be in cache for first few hundred ms
   Trade-off: acceptable (not financial data)

3. Async analytics → possible click loss on crash
   Fire-and-forget Kafka may lose a click event if service crashes
   Trade-off: analytics can tolerate <0.01% event loss
   Alternative: synchronous Kafka ACK (adds ~5ms latency)

Scaling path:
  Current design handles 12,000 redirects/sec easily

  Scale to 10x (120,000 redirects/sec):
  → More Redis nodes (horizontal cluster)
  → More Redirect Service instances (stateless, easy HPA)
  → Read replicas for DB (rare misses anyway)

  Scale to 100x (1.2M redirects/sec):
  → Multi-region: US, EU, AP each with local Redis + DB replica
  → CDN: cache redirects at edge (works for 302 with short TTL)
  → DB sharding by shortCode hash (only if DB is bottleneck)
```

### The Complete Data Model

```sql
CREATE TABLE urls (
    id           BIGINT PRIMARY KEY,        -- auto-increment
    short_code   VARCHAR(20) UNIQUE NOT NULL, -- base62 encoded or custom
    long_url     TEXT NOT NULL,
    user_id      UUID NOT NULL,
    created_at   TIMESTAMP DEFAULT NOW(),
    expires_at   TIMESTAMP,                 -- NULL = never expires
    is_active    BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_short_code ON urls(short_code);
CREATE INDEX idx_user_id    ON urls(user_id);
CREATE INDEX idx_expires_at ON urls(expires_at) WHERE expires_at IS NOT NULL;

-- Redis schema:
-- Key:   "url:{shortCode}"
-- Value: "{longUrl}"
-- TTL:   set to expiration time (or 30 days if no expiration)
```

---

## Interview Answer Skeleton

```
"A URL shortener has two core operations: shorten and redirect.
The system is extremely read-heavy — about 5,000 reads per write.

For shortening, I'd use auto-increment IDs encoded in base62.
This guarantees uniqueness without DB checks.

For redirects, every request hits Redis first.
With a 95% hit rate, most redirects complete in under 5ms.
We use 302 redirects to preserve analytics.

The key scale challenge is the redirect path:
12,000 redirects/sec is trivial for Redis.
The redirect service is stateless, so it scales horizontally.

The main tradeoffs are:
Sequential IDs are enumerable — trade-off accepted for simplicity.
Async analytics trades accuracy for speed — acceptable at sub-0.01% loss.

The design handles 100x growth by adding Redis nodes,
scaling stateless redirect services via HPA,
and deploying regionally for low latency globally."
```
