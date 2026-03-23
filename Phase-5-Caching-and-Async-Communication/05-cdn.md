# Content Delivery Network (CDN)

**Phase 5 — Caching + Async Communication | Topic 5**

---

## What is a CDN?

A CDN is a globally distributed network of servers (edge nodes/PoPs) that cache and serve content from locations geographically close to users, reducing latency and offloading traffic from your origin servers.

```
Without CDN:
  User in Tokyo requests image stored in AWS us-east-1 (Virginia)
  Round trip: Tokyo → Virginia → Tokyo = ~150ms × 2 = 300ms
  + Origin server processes request
  + Full image bytes travel across Pacific Ocean

With CDN:
  User in Tokyo requests same image
  CDN edge node in Tokyo has cached copy
  Round trip: Tokyo → Tokyo CDN PoP = ~5ms
  Origin server never contacted
  Image served from local cache
```

CDNs are most impactful for static assets (images, CSS, JS, videos) but modern CDNs also accelerate dynamic content and APIs.

---

## CDN Architecture

```
Origin Server (your infrastructure)
  │
  │  pulls content (on cache miss)
  ▼
Regional CDN Nodes (fewer, larger)
  │
  │  pulls content (on cache miss)
  ▼
Edge PoPs / Points of Presence (many, worldwide)
  │
  │  serves content
  ▼
End Users

PoP locations: typically 200+ worldwide
Major providers: Cloudflare (285+ cities), AWS CloudFront (450+ PoPs),
                 Akamai (4,000+ PoPs), Fastly, Azure CDN
```

---

## How CDN Caching Works

### Cache Miss — First Request

```
t=0:   User in Singapore requests /images/product-123.jpg
t=1:   CDN edge node in Singapore checks cache → MISS
t=2:   Edge node requests from origin (AWS us-east-1)
t=100: Origin responds with image + cache headers:
       Cache-Control: public, max-age=86400
       ETag: "a3f8c2d1"
t=101: Edge node caches image locally
t=102: User receives image (100ms total — slow first time)
```

### Cache Hit — Subsequent Requests

```
t=200: Another user in Singapore requests same image
t=201: CDN edge node checks cache → HIT
t=202: Serves from local cache (no origin contact)
       (2ms total — 50x faster)
```

---

## Cache Headers — Controlling CDN Behavior

```
Cache-Control: public, max-age=86400
  → CDN can cache this
  → Cache for 86400 seconds (24 hours)
  → "public" = safe for shared cache (CDN, proxy)

Cache-Control: private, max-age=3600
  → Only browser can cache this (not CDN)
  → Personalized content: user-specific data

Cache-Control: no-cache
  → Must revalidate before serving (check ETag/Last-Modified)
  → Not "no caching" — just "always validate freshness"

Cache-Control: no-store
  → Never cache at all — not CDN, not browser
  → Sensitive data: PII, financial, one-time tokens

Cache-Control: s-maxage=3600, max-age=0
  → CDN caches for 1 hour
  → Browser caches for 0 seconds (always goes to CDN)
  → Useful: content changes, CDN invalidation is fast,
            you want control at CDN level not browser level

ETag: "a3f8c2d1"
  → Fingerprint of content
  → Browser/CDN sends: If-None-Match: "a3f8c2d1"
  → If unchanged: 304 Not Modified (no body sent)
  → Saves bandwidth even when validating

Surrogate-Control: max-age=3600 (Varnish/Fastly)
  → CDN-specific cache instructions, stripped before client
  → CDN caches 1 hour, browser caches whatever Cache-Control says
```

---

## CDN Use Cases

### 1. Static Asset Delivery (Core Use Case)

Assets that never change per request:
- Images, CSS, JavaScript, fonts, PDFs, documents

```
Best practices:
  Long TTL (1 year) + cache-busting filenames

  /static/styles.a3f8c2d1.css  ← hash in filename
  Cache-Control: public, max-age=31536000, immutable

  When CSS changes:
  → New hash → new filename → new CDN URL
  → Old URL still cached (users on old page still work)
  → New URL fetches fresh from origin
  → No cache invalidation needed (different URL)

  "immutable" hint: browser never revalidates even on refresh
  (filename already guarantees uniqueness)
```

### 2. Video Streaming

```
Video files are large (100MB - 50GB per file)
Serving from origin to millions of viewers = impossible

CDN approach:
  Video split into small segments (HLS: 2-10 second chunks)
  Each segment cached at edge

  Netflix example:
  Open Connect (Netflix's own CDN)
  Upload popular content to edge servers before traffic arrives
  60% of North American internet traffic at peak
  All served from ISP-embedded Netflix appliances

Key insight: video is pre-cached during off-peak hours
             Not reactive caching — proactive placement
```

### 3. API Acceleration

```
CDNs can cache API responses too, not just static files:

GET /api/products/catalog
Cache-Control: s-maxage=300  ← CDN caches 5 minutes

Benefits:
  Reduces origin API calls
  Lower latency for API consumers globally
  Protects origin from traffic spikes

Limitations:
  Only works for GET requests (idempotent, safe)
  POST/PUT/DELETE must go to origin
  Personalized responses can't be shared

Cloudflare Workers / Fastly Compute@Edge:
  Run code at edge nodes
  Fetch and transform data at edge
  Build API responses without hitting origin for common patterns
```

### 4. DDoS Protection

```
Large CDN providers absorb attack traffic at edge:

DDoS attack: 2 Tbps of traffic toward your origin
Without CDN: origin overwhelmed → down
With CDN:    Cloudflare/Akamai absorbs at 200+ global PoPs
             Scrubs malicious traffic before reaching origin
             Origin sees only legitimate requests

Cloudflare capacity: 100+ Tbps globally
Your origin: probably 10 Gbps
→ CDN is your DDoS shield
```

### 5. SSL/TLS Termination at Edge

```
Without CDN:
  User → TLS handshake → your origin (100ms if far away)
  TLS handshake alone = 1-2 round trips = 200-400ms before first byte

With CDN:
  User → TLS handshake → nearby CDN PoP (5ms)
  CDN → keeps persistent connection to origin
  → TLS overhead: 5ms instead of 200ms
  → Origin connection reused across users
```

---

## CDN Cache Invalidation

The hardest CDN problem — how do you remove stale content?

### Approach 1: TTL Expiration (Simple)

```
Set appropriate TTL upfront:
  Static assets (versioned):   1 year (immutable)
  Product images:              24 hours
  News articles:               1 hour
  API responses:               5 minutes

Accept staleness within TTL window
No active invalidation needed
Simple, predictable

Limitation: deployed wrong content? Wait for TTL.
            Product image updated? Wrong image shown for 24 hours.
```

### Approach 2: Cache Purge (Active Invalidation)

```
// Cloudflare API
curl -X POST "https://api.cloudflare.com/client/v4/zones/{zone_id}/purge_cache" \
  -H "Authorization: Bearer {token}" \
  -d '{"files": ["https://yoursite.com/images/product-123.jpg"]}'

// AWS CloudFront
aws cloudfront create-invalidation \
  --distribution-id ABCDEFGHIJK \
  --paths "/images/product-123.jpg" "/images/*"
```

```java
// Spring Boot integration
@Service
public class CdnInvalidationService {

    @Value("${cloudfront.distribution-id}")
    private String distributionId;

    public void invalidate(List<String> paths) {
        CreateInvalidationRequest request = CreateInvalidationRequest.builder()
            .distributionId(distributionId)
            .invalidationBatch(InvalidationBatch.builder()
                .callerReference(UUID.randomUUID().toString())
                .paths(Paths.builder()
                    .quantity(paths.size())
                    .items(paths)
                    .build())
                .build())
            .build();

        cloudFrontClient.createInvalidation(request);
    }
}

// Triggered when product updated:
@EventListener(ProductUpdatedEvent.class)
public void onProductUpdated(ProductUpdatedEvent event) {
    cdnInvalidationService.invalidate(List.of(
        "/images/product-" + event.getProductId() + ".*",
        "/api/products/" + event.getProductId()
    ));
}
```

CloudFront invalidation cost: first 1,000 paths/month free, then $0.005/path. Wildcard `/images/*` counts as 1 path.

### Approach 3: Cache-Busting (Best for Static Assets)

```
Embed content hash in filename or URL:
  /static/styles.css?v=a3f8c2d1   ← query param version
  /static/styles.a3f8c2d1.css     ← filename hash (preferred)
  /static/v42/styles.css          ← directory version

No invalidation needed:
  File changes → new hash → new URL → CDN treats as new resource
  Old URL still cached (users on old pages still work)
  New URL always served fresh

Build tools (webpack, Vite) generate hashes automatically
```

---

## CDN and Dynamic Content

### Edge Side Includes (ESI)

```html
<!-- Assemble page from cached fragments at edge -->
<html>
  <esi:include src="/header" ttl="3600"/>  <!-- cached 1 hour -->

  <main>
    <!-- Dynamic, user-specific, not cached -->
    <esi:include src="/api/user/cart" ttl="0"/>
  </main>

  <esi:include src="/footer" ttl="86400"/> <!-- cached 24 hours -->
</html>
```

Cache what you can (header, footer), fetch what you must (cart, user data).

### Geo-Routing and Load Balancing

```
CDN can route traffic based on:
  Geographic proximity → nearest origin region
  Origin health → route away from unhealthy regions
  Latency → measure actual RTT, route to fastest

AWS CloudFront + ALB:
  CloudFront → Route 53 latency routing →
  us-east-1 ALB (US users)
  eu-west-1 ALB (EU users)
  ap-southeast-1 ALB (Asian users)
```

---

## CDN Configuration — Real World

### AWS CloudFront

```
Origin: your ALB or S3 bucket
Behaviors (path-based rules):

/static/*
  → S3 bucket origin
  → Cache: TTL max 1 year
  → Compress: gzip/brotli
  → No query string forwarding

/api/*
  → ALB origin
  → Cache: TTL 0 (dynamic)
  → Forward all headers, cookies
  → Origin shield enabled (collapse cache misses to single origin request)

/*.html
  → ALB origin
  → Cache: TTL 5 minutes
  → Forward Accept-Encoding only

Default behavior:
  → ALB origin
  → Cache based on Cache-Control headers from origin
```

### Origin Shield

```
Problem without Origin Shield:
  CDN has 200 PoPs globally
  Popular content expires → 200 edge nodes simultaneously
  request fresh copy from origin
  → 200 concurrent requests to origin
  → Origin overwhelmed

With Origin Shield:
  Single regional node sits between all edge nodes and origin

  Edge nodes (200) → [Origin Shield: us-east-1] → Origin

  Cache miss:
  Edge 1 (Singapore) misses → asks Origin Shield
  Edge 2 (Tokyo) misses → asks Origin Shield
  Origin Shield collapses both to ONE request to origin

  Origin sees 1 request, not 200
```

---

## CDN Metrics

```
Cache Hit Ratio:
  CDN hits / total requests
  Target: > 90% for static assets
  Low hit ratio → check TTL, check if URLs are consistent
  (URL params cause misses: /image.jpg?t=123456789)

Bandwidth Offload:
  % of bytes served from CDN vs origin
  Target: > 90%

Edge Latency:
  Time from user to CDN edge
  Should be < 10ms for major markets

Origin Pull Rate:
  Requests reaching your origin
  Lower is better
  Monitor for spikes (TTL expiry, cache purge)

Error Rate:
  5xx from origin visible at edge
  CDN can serve stale content on origin error (stale-if-error)
```

---

## CDN in System Design Interviews

When to mention CDN:

**Any system with media/static assets:**
> "All static assets — images, CSS, JavaScript — are served through CloudFront with a 1-year TTL and content-hash filenames. This offloads 95% of static traffic from our origin servers."

**Video platform:**
> "Videos are chunked into HLS segments and distributed across CDN edge nodes. Popular content is proactively pushed to edge nodes before traffic arrives, so the first viewer in each region isn't a cache miss."

**Global product:**
> "CloudFront handles SSL termination at the nearest PoP, reducing TLS handshake latency from ~200ms to ~5ms for international users. API responses for public product catalog are cached at edge for 5 minutes."

**DDoS protection:**
> "CloudFront sits in front of everything, providing DDoS protection. Our origin is in a private VPC only accessible via CloudFront — direct origin IP is never exposed."

---

## Key Takeaways

```
CDN: Globally distributed cache of edge nodes
     Serve content from near the user
     Offload origin, reduce latency, protect against DDoS

Core mechanism:
  First request: edge miss → pull from origin → cache → serve
  Subsequent: edge hit → serve from local cache

Cache headers:
  Cache-Control: public, max-age=86400  → CDN + browser cache
  Cache-Control: private                → browser only (not CDN)
  Cache-Control: no-store               → never cache (sensitive)
  s-maxage                              → CDN-specific TTL
  ETag                                  → conditional requests

Invalidation strategies:
  TTL:           simple, accept staleness window
  Active purge:  immediate, API call per URL
  Cache-busting: hash in filename, no invalidation needed (best)

Use cases:
  Static assets:   long TTL + versioned filenames
  Video:           HLS segments, pre-populated at edge
  API:             GET responses with short TTL
  DDoS:            absorb at edge, scrub malicious traffic
  SSL:             terminate at edge, 5ms vs 200ms globally

CDN providers:
  Cloudflare:   285+ PoPs, DDoS protection, Workers
  CloudFront:   AWS native, 450+ PoPs, deep integration
  Fastly:       Compute@Edge, real-time purge
  Akamai:       Largest network, enterprise-focused

Origin Shield:
  Collapses parallel cache misses from 200 edges to 1 origin request
  Protects origin from traffic spikes on popular content expiry

In interviews:
  Always put CDN in front of static assets
  Mention for any global or media-heavy system
  Connect to DDoS protection and SSL termination
```
