# Fault Tolerance and Resilience Patterns

**Phase 6 — Distributed Systems + Architecture | Topic 6**

---

## What is Fault Tolerance?

Fault tolerance is the ability of a system to continue operating correctly despite the failure of one or more of its components. Resilience is the broader ability to recover quickly from failures.

```
Reality of distributed systems:
  Servers crash (hardware fails)
  Networks partition (packets lost, delayed, reordered)
  Disks corrupt (silent data corruption)
  Services timeout (overloaded, GC pause)
  Dependencies fail (third-party API down)
  Bugs cause exceptions (code is imperfect)

Question is not IF failures happen — it's WHEN.
Design for failure from day one.
```

---

## Pattern 1: Retry

Automatically re-attempt a failed operation.

```
Transient failure:
  Network blip → request fails → retry → succeeds

Without retry:
  Service A → Service B (timeout) → error returned to user

With retry:
  Service A → Service B (timeout)
           → retry 1 (500ms) → success
  User never knows failure occurred
```

### Retry with Exponential Backoff + Jitter

```java
@Service
public class ResilientPaymentClient {

    private final PaymentServiceClient client;

    // Resilience4j retry
    @Retry(name = "payment-service", fallbackMethod = "paymentFallback")
    public PaymentResult processPayment(PaymentRequest request) {
        return client.processPayment(request);
    }

    public PaymentResult paymentFallback(PaymentRequest request, Exception e) {
        log.error("Payment service unavailable after retries", e);
        return PaymentResult.queued(request.getOrderId());
    }
}
```

```yaml
# application.yml — Retry configuration
resilience4j:
  retry:
    instances:
      payment-service:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2   # 500ms → 1000ms → 2000ms
        randomized-wait-factor: 0.3         # ±30% jitter
        retry-exceptions:
          - java.net.ConnectException
          - java.util.concurrent.TimeoutException
        ignore-exceptions:
          - com.example.InvalidRequestException  # don't retry client errors
```

**Exponential backoff + jitter:**

```
Without jitter:
  1000 clients all retry at t=500ms simultaneously
  → Thundering herd on failed service
  → Service recovers → immediately overwhelmed again

With jitter (random variance):
  Client 1 retries at t=487ms
  Client 2 retries at t=523ms
  Client 3 retries at t=441ms
  → Retries spread out → gradual load → service recovers smoothly

Formula:
  base_delay = min(cap, base * 2^attempt)
  actual_delay = random_between(0, base_delay)

  Attempt 1: random(0, 500ms)
  Attempt 2: random(0, 1000ms)
  Attempt 3: random(0, 2000ms)
```

**Retry anti-patterns:**

```
❌ Retry non-idempotent operations blindly
   POST /payments → retried → charged twice!
   Solution: idempotency key + check before retry

❌ Retry indefinitely
   Infinite retry loops → thread exhaustion
   Solution: max attempts with fallback

❌ Immediate retry
   Failed service needs time to recover
   Solution: exponential backoff

❌ Retry 4xx errors
   400 Bad Request → retry won't help (invalid input)
   Solution: only retry 5xx and network errors
```

---

## Pattern 2: Circuit Breaker

Prevent calls to a failing service, allowing it time to recover.

```
States:
  CLOSED:    normal operation, requests flow through
  OPEN:      service failing, requests fail fast (no actual call)
  HALF_OPEN: test if service recovered, allow limited traffic

State transitions:
  CLOSED:    failure rate > threshold → OPEN
  OPEN:      wait timeout → HALF_OPEN
  HALF_OPEN: test succeeds → CLOSED
             test fails    → OPEN

Example:
  t=0:   Circuit CLOSED, normal traffic
  t=10:  Payment Service starts failing (DB overloaded)
  t=10:  Failures pile up, timeout exceptions
  t=15:  5 failures in 10 seconds → threshold exceeded
  t=15:  Circuit OPENS
  t=15-75: All calls fail immediately (no actual network call)
           Threads freed immediately, Order Service stays healthy
  t=75:  Timeout passed → circuit enters HALF_OPEN
  t=75:  Single test request allowed → Payment Service recovered
  t=75:  Test succeeds → Circuit CLOSES
  t=75+: Normal traffic resumes
```

```java
@Service
public class OrderService {

    private final PaymentServiceClient paymentClient;

    // Circuit breaker + retry combined
    @CircuitBreaker(name = "payment-service",
                    fallbackMethod = "paymentCircuitFallback")
    @Retry(name = "payment-service")
    public PaymentResult processPayment(String orderId, BigDecimal amount) {
        return paymentClient.charge(orderId, amount);
    }

    // Called when circuit is OPEN or all retries exhausted
    public PaymentResult paymentCircuitFallback(
            String orderId, BigDecimal amount, CallNotPermittedException e) {
        // Circuit is open — queue for later processing
        paymentQueue.queueForRetry(orderId, amount);
        return PaymentResult.queued(orderId);
    }
}
```

```yaml
# application.yml — Circuit Breaker
resilience4j:
  circuit-breaker:
    instances:
      payment-service:
        sliding-window-type: COUNT_BASED
        sliding-window-size: 10          # look at last 10 calls
        failure-rate-threshold: 50       # open if >50% fail
        slow-call-rate-threshold: 80     # open if >80% calls are slow
        slow-call-duration-threshold: 2s # "slow" = >2 seconds
        wait-duration-in-open-state: 60s # wait 60s before HALF_OPEN
        permitted-calls-in-half-open-state: 3  # test with 3 calls
        automatic-transition-from-open-to-half-open-enabled: true
```

### Circuit Breaker Monitoring

```java
@Component
public class CircuitBreakerMonitor {

    private final CircuitBreakerRegistry registry;
    private final MeterRegistry meterRegistry;

    @PostConstruct
    public void monitor() {
        registry.getAllCircuitBreakers().forEach(cb -> {
            cb.getEventPublisher()
                .onStateTransition(event -> {
                    log.warn("Circuit Breaker {} transitioned: {} → {}",
                        cb.getName(),
                        event.getStateTransition().getFromState(),
                        event.getStateTransition().getToState());

                    meterRegistry.counter("circuit_breaker.transitions",
                        "name", cb.getName(),
                        "from", event.getStateTransition().getFromState().toString(),
                        "to", event.getStateTransition().getToState().toString()
                    ).increment();

                    // Alert if circuit OPENS
                    if (event.getStateTransition().getToState() == OPEN) {
                        alertService.sendAlert("Circuit breaker OPEN: " + cb.getName());
                    }
                });
        });
    }
}
```

---

## Pattern 3: Bulkhead

Isolate resources so failure in one area doesn't exhaust resources needed by others.

```
Ship design metaphor:
  Ship hull divided into watertight compartments (bulkheads)
  One compartment floods → others stay dry → ship floats
  Without bulkheads → one breach → entire ship sinks

Application equivalent:
  Single thread pool for all external calls:

  [App Thread Pool: 200 threads]
  ├── Payment calls (slow, using 190 threads waiting)
  ├── User calls (fast, need 5 threads but can't get them)
  └── Inventory calls (fast, need 5 threads but can't get them)

  Payment service slowness starves all other operations

With bulkhead (separate thread pools):
  [Payment Pool: 50 threads]    ← can only use 50 threads
  [User Pool: 30 threads]       ← always available
  [Inventory Pool: 20 threads]  ← always available

  Payment service slow → only affects payment pool
  User and inventory calls continue normally
```

```yaml
# Thread pool bulkhead
resilience4j:
  thread-pool-bulkhead:
    instances:
      payment-service:
        max-thread-pool-size: 50
        core-thread-pool-size: 20
        queue-capacity: 100           # queue when pool full
        keep-alive-duration: 60s

      user-service:
        max-thread-pool-size: 30
        core-thread-pool-size: 10
        queue-capacity: 50

      inventory-service:
        max-thread-pool-size: 20
        core-thread-pool-size: 10
        queue-capacity: 50
```

```java
@ThreadPoolBulkhead(name = "payment-service")
@CircuitBreaker(name = "payment-service")
public CompletableFuture<PaymentResult> processPaymentAsync(PaymentRequest req) {
    return CompletableFuture.supplyAsync(() ->
        paymentClient.processPayment(req));
}

// Semaphore bulkhead (limits concurrent calls, not threads)
resilience4j:
  bulkhead:
    instances:
      payment-service:
        max-concurrent-calls: 25  # max 25 concurrent in-flight requests
        max-wait-duration: 10ms   # wait 10ms for slot, then reject
```

---

## Pattern 4: Timeout

Never wait forever. Set maximum time for any operation.

```
Without timeout:
  Service A calls Service B
  Service B hangs (deadlock, slow query, GC pause)
  Service A thread blocked forever
  → Thread pool exhausted
  → Service A stops accepting new requests
  → Cascading failure

With timeout:
  Service A calls Service B with 2s timeout
  Service B hangs at 3s
  Timeout triggers → exception thrown → circuit breaker records failure
  Thread freed immediately
  Service A continues serving other requests
```

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient paymentServiceWebClient() {
        HttpClient httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 1000)  // 1s connect
            .responseTimeout(Duration.ofSeconds(2))               // 2s read
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(2, TimeUnit.SECONDS))
                .addHandlerLast(new WriteTimeoutHandler(1, TimeUnit.SECONDS))
            );

        return WebClient.builder()
            .baseUrl("http://payment-service")
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}

// Resilience4j timeout
resilience4j:
  timelimiter:
    instances:
      payment-service:
        timeout-duration: 2s
        cancel-running-future: true  # cancel underlying call on timeout

@TimeLimiter(name = "payment-service")
public CompletableFuture<PaymentResult> processPayment(PaymentRequest req) {
    return CompletableFuture.supplyAsync(() -> paymentClient.charge(req));
}
```

**Timeout guidelines:**

```
Connect timeout:   1-2 seconds (establishing connection)
Read timeout:      2-30 seconds (depends on operation)

Rule: timeout < upstream caller's timeout
  API Gateway timeout: 30s
  Order Service read timeout: 10s (leaves margin for own processing)
  Payment Service read timeout: 5s (leaves margin for network + processing)

Timeout waterfall:
  User expects response in 3s total
  API Gateway: 5s timeout
  Order Service: 4s timeout
  Payment Service: 3s timeout
  Bank API: 2s timeout
```

---

## Pattern 5: Rate Limiting

Rate limiting protects services from overload. Applied to fault tolerance:

```
Rate limiting protects services from:
  Runaway clients (bug causing infinite retry loop)
  Traffic spikes (Black Friday)
  DDoS attempts

From the consuming service perspective:
  Don't flood a degraded service with requests
  Honor 429 responses with backoff
```

```java
@Service
public class ResilientApiClient {

    @RateLimiter(name = "external-api", fallbackMethod = "apiRateLimitFallback")
    public ApiResponse callExternalApi(String request) {
        return externalApiClient.call(request);
    }

    public ApiResponse apiRateLimitFallback(String request,
                                             RequestNotPermitted e) {
        return ApiResponse.cached(cacheService.getLastKnown(request));
    }
}
```

```yaml
resilience4j:
  rate-limiter:
    instances:
      external-api:
        limit-for-period: 100       # 100 calls
        limit-refresh-period: 1s    # per second
        timeout-duration: 0         # fail immediately if over limit
```

---

## Pattern 6: Fallback

Provide alternative behavior when the primary path fails.

```
Fallback options (from best to worst):
  1. Cached response (stale but useful)
  2. Default value (neutral, not harmful)
  3. Degraded response (less functionality)
  4. Error message (honest about failure)
  5. Throw exception (last resort)
```

```java
@Service
public class ProductRecommendationService {

    private final MLRecommendationEngine mlEngine;
    private final RedisTemplate<String, List<Product>> cache;
    private final List<Product> popularProducts;  // pre-loaded at startup

    @CircuitBreaker(name = "ml-engine", fallbackMethod = "cachedFallback")
    public List<Product> getRecommendations(String userId) {
        return mlEngine.recommend(userId);  // primary: ML-based
    }

    // Fallback 1: cached previous recommendation
    public List<Product> cachedFallback(String userId, Exception e) {
        List<Product> cached = cache.opsForValue()
            .get("recommendations:" + userId);

        if (cached != null) {
            log.info("Using cached recommendations for user {}", userId);
            return cached;           // stale but personalized
        }

        return popularFallback(userId, e);
    }

    // Fallback 2: popular products for everyone
    public List<Product> popularFallback(String userId, Exception e) {
        log.warn("Using popular products fallback for user {}", userId);
        return popularProducts;      // not personalized but useful
    }
}
```

---

## Pattern 7: Health Checks and Self-Healing

Services must report their health and be automatically replaced when unhealthy.

```java
@Component
public class ApplicationHealthIndicator implements HealthIndicator {

    private final DataSource dataSource;
    private final RedisTemplate<?, ?> redis;
    private final KafkaProducer<?, ?> kafkaProducer;

    @Override
    public Health health() {
        Health.Builder builder = new Health.Builder();

        // Check DB connection
        try {
            dataSource.getConnection().isValid(1);
            builder.withDetail("database", "UP");
        } catch (Exception e) {
            return builder.down(e)
                .withDetail("database", "DOWN: " + e.getMessage())
                .build();
        }

        // Check Redis
        try {
            redis.execute(RedisServerCommands::ping);
            builder.withDetail("redis", "UP");
        } catch (Exception e) {
            // Redis down → degraded but not dead
            builder.withDetail("redis", "DEGRADED: " + e.getMessage());
        }

        // Check Kafka (non-fatal)
        try {
            kafkaProducer.metrics(); // lightweight check
            builder.withDetail("kafka", "UP");
        } catch (Exception e) {
            builder.withDetail("kafka", "DEGRADED: " + e.getMessage());
        }

        return builder.up().build();
    }
}
```

```yaml
# Kubernetes liveness vs readiness:
# /actuator/health/liveness  → am I alive? (restart if not)
# /actuator/health/readiness → am I ready? (remove from LB if not)

management:
  endpoint:
    health:
      probes:
        enabled: true
      group:
        liveness:
          include: livenessState
        readiness:
          include: readinessState, db, redis
```

---

## Combining Patterns — The Resilience Stack

In production, these patterns layer together:

```
Incoming request
       ↓
[Rate Limiter]        ← reject if over limit
       ↓
[Bulkhead]            ← acquire thread from isolated pool
       ↓
[Circuit Breaker]     ← fail fast if service down
       ↓
[Timeout]             ← don't wait forever
       ↓
[Retry]               ← retry transient failures
       ↓
[Target Service]
       ↓
[Fallback]            ← if all else fails, return something useful

Order matters:
  Rate limiter outermost (cheapest rejection)
  Retry inside circuit breaker
    (don't retry when circuit is open — pointless)
  Timeout inside retry
    (each retry attempt has its own timeout)
```

```java
// Resilience4j — combining all patterns
@RateLimiter(name = "payment")
@Bulkhead(name = "payment")
@CircuitBreaker(name = "payment", fallbackMethod = "fallback")
@TimeLimiter(name = "payment")
@Retry(name = "payment")
public CompletableFuture<PaymentResult> processPayment(PaymentRequest req) {
    return CompletableFuture.supplyAsync(() ->
        paymentClient.charge(req));
}
```

```yaml
# application.yml — all patterns configured
resilience4j:
  rate-limiter:
    instances:
      payment:
        limit-for-period: 1000
        limit-refresh-period: 1s
  bulkhead:
    instances:
      payment:
        max-concurrent-calls: 50
  circuit-breaker:
    instances:
      payment:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
  timelimiter:
    instances:
      payment:
        timeout-duration: 3s
  retry:
    instances:
      payment:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
```

---

## Chaos Engineering

Deliberately introduce failures in production to prove fault tolerance works:

```
Principles (Netflix Chaos Monkey approach):
  1. Define steady state (normal system behavior)
  2. Hypothesize steady state continues during chaos
  3. Introduce real-world chaos (kill servers, delay network)
  4. Verify steady state maintained

Types of chaos experiments:
  Kill random service instances
  Introduce network latency (add 200ms to all calls)
  Drop random % of network packets
  Kill database primary (test failover)
  Fill disk to 100%
  Inject high CPU load

Netflix Chaos Monkey:
  Randomly terminates EC2 instances in production
  Forces team to build for failure
  "If we can survive random terminations, we can survive anything"

AWS Fault Injection Simulator (FIS):
  Managed chaos engineering
  Inject API throttling, AZ outage, resource pressure
  Safe rollback if experiment goes wrong
```

---

## Key Takeaways

```
Fault Tolerance: system continues operating despite failures
Resilience: system recovers quickly from failures

Core patterns:

Retry:
  Re-attempt transient failures
  Exponential backoff + jitter (prevent thundering herd)
  Only retry idempotent operations + 5xx errors
  Max attempts with fallback

Circuit Breaker:
  CLOSED → OPEN → HALF_OPEN state machine
  Fail fast when service unhealthy
  Prevent cascading failure
  Allow recovery time

Bulkhead:
  Separate thread pools per dependency
  One slow service can't exhaust all threads
  Limits blast radius of failures

Timeout:
  Never wait forever
  Fail fast, free thread immediately
  Set cascading timeouts (inner < outer)

Rate Limiter:
  Protect services from overload
  Honor 429 responses from dependencies

Fallback:
  Always have plan B
  Cached → default → degraded → error

Combining patterns:
  Rate limiter → Bulkhead → Circuit Breaker →
  Timeout → Retry → Target → Fallback

Spring Boot: Resilience4j
  @Retry, @CircuitBreaker, @Bulkhead,
  @TimeLimiter, @RateLimiter
  All declarative via annotations + YAML config

Chaos Engineering:
  Prove fault tolerance by breaking things intentionally
  Netflix Chaos Monkey: random instance termination
  AWS FIS: managed chaos experiments
```
