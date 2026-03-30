# Scalability Patterns

**Phase 6 — Distributed Systems + Architecture | Topic 9**

---

## What is Scalability?

Scalability is the ability of a system to handle increased load by adding resources, while maintaining acceptable performance.

```
Not scalable:
  System handles 1,000 users at 100ms response time
  Add 9,000 more users → system crashes or degrades to 10,000ms
  Throwing hardware at it doesn't help (bottleneck can't be parallelized)

Scalable:
  System handles 1,000 users at 100ms
  Add 9 more servers → handles 10,000 users at 100ms
  Linear (or better) relationship between resources and capacity
```

---

## Vertical vs Horizontal Scaling

```
Vertical Scaling (Scale Up):
  Replace small machine with bigger machine
  2 CPU → 32 CPU
  16GB RAM → 512GB RAM

  Pros:  Simple, no code changes, strong consistency easy
  Cons:  Hard ceiling (biggest machine available)
         SPOF (one big machine)
         Expensive (non-linear cost)
         Downtime during upgrade

  Practical ceiling: ~128 vCPU, ~2TB RAM on cloud instances
  Good for: databases, when code can't be parallelized

Horizontal Scaling (Scale Out):
  Add more machines of same size
  1 server → 10 servers → 100 servers

  Pros:  No theoretical ceiling
         Fault tolerant (one dies, others continue)
         Cost-effective (commodity hardware)
  Cons:  Code must handle distributed operation
         State must be managed carefully
         More operational complexity

  Good for: stateless services, anything that can be parallelized
```

---

## Pattern 1: Stateless Services

Design services to hold no request-specific state in memory. Any instance can handle any request.

```
Stateful (can't scale horizontally):
  User logs in → session stored in Server A's memory
  Next request → load balanced to Server B
  Server B has no session → user logged out!

  Must use sticky sessions → defeats load balancing purpose

Stateless (scales horizontally freely):
  User logs in → JWT issued, stored client-side
  Next request → JWT included in header
  Any server can validate JWT → any server can handle request

  State stored externally:
  Sessions → Redis
  User data → Database
  File uploads → S3
  Cache → Redis
```

```java
// Stateless service example
@RestController
public class OrderController {

    // No instance variables storing request state
    // All state comes from: request, DB, Redis, external services

    @GetMapping("/orders/{id}")
    public Order getOrder(
            @PathVariable String id,
            @RequestHeader("Authorization") String jwt) {

        // State from JWT (not server memory)
        String userId = jwtService.extractUserId(jwt);

        // State from DB (not server memory)
        Order order = orderRepository.findById(id);

        // Authorization check
        if (!order.getUserId().equals(userId)) {
            throw new UnauthorizedException();
        }

        return order;
    }
}
// Any of 100 instances handles this correctly
// Load balancer can route to any instance freely
```

---

## Pattern 2: Load Balancing (Revisited)

```
Load balancer = the entry point for horizontal scaling
10 servers behind LB appear as one to the client
Adding server = adding capacity immediately

Auto Scaling:
  CloudWatch: CPU > 70% for 5 minutes → add instance
  CloudWatch: CPU < 30% for 15 minutes → remove instance

  Kubernetes HPA (Horizontal Pod Autoscaler):
  CPU > 80% → scale up
  CPU < 30% → scale down
```

```yaml
# Kubernetes HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 50
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Pods
    pods:
      metric:
        name: kafka_consumer_lag
      target:
        type: AverageValue
        averageValue: "1000"  # scale if lag > 1000 messages per pod
```

---

## Pattern 3: Caching at Every Layer

```
Cache absorbs load before it reaches the bottleneck (DB)

90% cache hit ratio:
  10M requests/day total
  9M served from Redis (<1ms)
  1M hit DB (10-50ms)

  DB sees 10x less load
  Response time 10x better (90% of requests sub-millisecond)

Cache hierarchy for scale:
  L1: Browser cache (no server cost)
  L2: CDN edge (geographically close)
  L3: API Gateway cache (before any service code)
  L4: Application cache (Caffeine, in-process)
  L5: Distributed cache (Redis)
  L6: DB query cache (not recommended for writes)

  Each layer reduces load on the next
```

---

## Pattern 4: Database Scaling Patterns

```
Scalability toolkit summary:

Read scaling:
  Read replicas: route SELECTs to replicas
  Caching: Redis absorbs repeated reads
  Denormalization: fewer JOINs = faster reads

Write scaling:
  Sharding: split data across multiple primary DBs
  CQRS: separate write model from read model
  Write-behind cache: batch writes to DB

Vertical scaling:
  Bigger machine handles more before sharding needed

Partitioning:
  Split large tables internally (same machine)
  Cheaper than sharding, improves query isolation
```

---

## Pattern 5: Asynchronous Processing

Convert synchronous user-blocking operations to async background work.

```
Synchronous (blocks user):
  User uploads 4K video (2GB)
  Server receives → transcodes to multiple resolutions → returns URL
  User waits 10 minutes staring at spinner

  Also: one slow job ties up thread
        10,000 uploads = 10,000 blocked threads

Asynchronous (non-blocking):
  User uploads 2GB video
  Server: save to S3, put job in queue → return "Processing..."
  User: immediate response, can close browser

  Background worker:
  Read from queue → transcode → update status in DB
  User checks status or gets notification when done

  Workers scale independently:
  20 uploads/sec → 20 workers
  200 uploads/sec → 200 workers (auto-scale based on queue depth)
```

```java
// Submit job and return immediately
@PostMapping("/videos")
public ResponseEntity<VideoUploadResponse> uploadVideo(
        @RequestParam MultipartFile file,
        @RequestHeader("Authorization") String jwt) {

    String userId = jwtService.extractUserId(jwt);
    String videoId = UUID.randomUUID().toString();

    // Save raw file to S3
    String s3Key = storageService.upload(file, videoId);

    // Queue transcoding job (non-blocking)
    kafkaTemplate.send("video-transcode-jobs", videoId,
        new TranscodeJob(videoId, userId, s3Key));

    // Return immediately
    return ResponseEntity.accepted()
        .body(new VideoUploadResponse(videoId, "PROCESSING",
            "/videos/" + videoId + "/status"));
}

// Worker (scales with queue depth)
@KafkaListener(topics = "video-transcode-jobs",
               groupId = "video-workers",
               concurrency = "10")  // 10 concurrent workers per instance
public void processTranscodeJob(TranscodeJob job) {
    log.info("Transcoding video {}", job.getVideoId());

    // CPU-intensive work (doesn't block API threads)
    List<TranscodedVideo> results = videoTranscoder.transcode(
        job.getS3Key(),
        List.of(Resolution._1080P, Resolution._720P, Resolution._480P)
    );

    // Update status
    videoRepository.updateStatus(job.getVideoId(), "COMPLETE", results);

    // Notify user
    notificationService.send(job.getUserId(),
        "Your video is ready: " + job.getVideoId());
}
```

---

## Pattern 6: Content Delivery Network

```
CDN = massive distributed cache for static content

Without CDN:
  All image requests hit origin servers
  10M images/day × 500KB avg = 5TB/day from your servers

With CDN:
  90% served from CDN edge nodes (50+ globally)
  Only 10% (cache misses) hit origin
  Origin serves 500GB/day instead of 5TB
  → 10x reduction in bandwidth and server load
  → Faster for users (served from nearby PoP)
```

---

## Pattern 7: Microservices and Functional Decomposition

```
Scale what needs scaling, not everything

Monolith under load:
  Black Friday: 100x normal checkout traffic
  Must scale ENTIRE monolith
  Even user settings page (0.1x load) scales to 100x
  Waste: 99.9% of extra capacity unused

Microservices:
  Checkout service: scale to 100x
  Payment service: scale to 100x
  Product catalog: scale to 10x (people browsing)
  User settings: 1x (almost no traffic)
  Recommendations: 5x

  Right-sized scaling per component
  Cost-efficient
```

---

## Pattern 8: Event-Driven Architecture

```
Decouple producers from consumers. Consumers scale independently.

Synchronous chain (tightly coupled scaling):
  User request
  → Order Service (must scale 100x)
  → Payment Service (must scale 100x, called synchronously)
  → Inventory Service (must scale 100x, called synchronously)
  → Notification Service (must scale 100x, called synchronously)

  All must scale together
  Payment Service slow → entire request slow

Event-driven (independently scalable):
  User request → Order Service (scale to match traffic)
  Order Service publishes to Kafka → returns immediately

  Payment Service:     Kafka consumer, scale to match message rate
  Inventory Service:   Kafka consumer, scale to match message rate
  Notification Service: Kafka consumer, scale slowly (emails can queue)

  Each service scales based on its own processing rate
  Notification Service can process at 10% of order rate (emails can wait)
  Payment Service must keep up (scale 1:1 with orders)
```

---

## Pattern 9: Database Denormalization for Read Scale

```
Normalized schema (optimized for writes, correct data):
  Users:    {id, name, email}
  Orders:   {id, user_id, created_at, total}
  Products: {id, name, price}
  OrderItems: {order_id, product_id, quantity, price}

  Query "get order with all details":
  SELECT o.*, u.name, u.email, p.name, oi.quantity
  FROM orders o
  JOIN users u ON u.id = o.user_id
  JOIN order_items oi ON oi.order_id = o.id
  JOIN products p ON p.id = oi.product_id
  WHERE o.id = '123'

  Cost: 4-table JOIN, complex query plan, scaling pain

Denormalized read model (optimized for reads):
  order_detail:{orderId} → Redis/Elasticsearch/DynamoDB
  {
    "orderId": "123",
    "userName": "Hari Kumar",
    "userEmail": "hari@example.com",
    "total": 99.99,
    "items": [
      {"productName": "Laptop Stand", "quantity": 1, "price": 49.99},
      {"productName": "USB Hub", "quantity": 2, "price": 24.99}
    ]
  }

  Query: GET order_detail:123 → one Redis lookup → <1ms

  Built by event-driven projection (CQRS pattern)
  Trade-off: eventual consistency, data duplication
```

---

## Pattern 10: Rate Limiting and Throttling for Stability

```
Scalability isn't just about handling more load.
It's about maintaining stability under any load.

Without rate limiting:
  One bad client sends 100K requests/second
  Server thread pool exhausted
  All other clients denied service
  System down for everyone

With rate limiting:
  Bad client throttled to 1K requests/second
  Thread pool protected
  Other clients served normally
  System stable under overload

Throttling strategies:
  Hard limit:    Reject above threshold (429 Too Many Requests)
  Soft limit:    Degrade service quality (return cached/simplified response)
  Queue:         Accept all, process at controlled rate (queue fills up)
  Backpressure:  Signal upstream to slow down (reactive systems)
```

---

## The Scalability Checklist

When designing for scale in an interview:

```
1. Stateless services?
   → Any instance handles any request
   → Horizontal scaling enabled

2. Read/write ratio identified?
   → Read-heavy: caching + read replicas
   → Write-heavy: sharding + async processing

3. Identify the bottleneck:
   → Compute? → Horizontal scaling (add servers)
   → DB reads? → Read replicas + Redis cache
   → DB writes? → Sharding + write-behind cache
   → Network? → CDN + compression
   → Single service? → Functional decomposition

4. Async where possible?
   → Long-running jobs → queue + workers
   → Fan-out operations → event-driven

5. Cache at every appropriate layer?
   → CDN for static
   → Redis for shared state
   → Local cache for hot, immutable data

6. Database strategy?
   → Indexes before anything else
   → Read replicas
   → Partitioning
   → Then sharding (last resort)

7. Autoscaling configured?
   → Scale on CPU + custom metrics (queue depth)
   → Set min/max bounds

8. Rate limiting?
   → Protect from traffic spikes
   → Protect from bad actors
```

---

## Putting It Together — Interview Pattern

When asked "How would you scale this system to 10x / 100x load?":

```
Step 1: Current bottleneck?
  "Currently a single PostgreSQL handles all reads and writes.
   At 10x load, this is the first bottleneck."

Step 2: Easy wins first
  "First I'd add Redis caching for the most frequently read data.
   With a 90% cache hit ratio, DB load immediately drops 10x."

Step 3: Scale reads
  "Add 3 read replicas. Route all SELECT queries to replicas.
   Primary handles writes only. 4x read capacity."

Step 4: Scale horizontally (app tier)
  "Application services are already stateless.
   Add HPA: scale pods based on CPU and request rate."

Step 5: Scale writes if needed
  "If write volume exceeds primary capacity,
   partition by user_id across 4 shards.
   Each shard handles 25% of write load."

Step 6: Async expensive operations
  "Email sending, report generation, recommendation updates —
   all moved to Kafka + async workers."

Step 7: CDN
  "Static assets, product images, public catalog through CloudFront.
   95% of static content served from edge."

That's the 10x to 100x scaling path,
ordered by impact and complexity.
```

---

## Key Takeaways

```
Scalability: handle more load by adding resources

Core patterns:

Stateless services:
  No request state in memory → horizontal scaling free
  JWT > sessions, Redis > in-memory state

Load balancing:
  Entry point for horizontal scaling
  HPA: autoscale on CPU + custom metrics

Caching:
  CDN → API Gateway → App → Redis → DB
  90% hit ratio = 10x load reduction on DB

Database scaling:
  Read replicas (reads) → Sharding (writes)
  Follow the ladder: index → replica → cache → shard

Async processing:
  Queue long-running work → worker pool scales independently
  Kafka consumer groups scale to message rate

Microservices:
  Scale what needs scaling, not everything
  Cost-efficient, right-sized per component

Event-driven:
  Decouple producers → consumers scale independently
  Backpressure: slow consumers don't block producers

Denormalization:
  Pre-joined read models → single-lookup reads
  CQRS pattern: separate write (normalized) from read (denormalized)

Rate limiting:
  Stability under overload
  Protect from bad actors and traffic spikes

Interview pattern:
  1. Identify bottleneck
  2. Cache first (highest ROI)
  3. Horizontal scale (stateless services)
  4. Read replicas
  5. Async processing
  6. Sharding (last resort)
```
