# Distributed Tracing and Observability

**Phase 6 — Distributed Systems + Architecture | Topic 7**

---

## The Observability Problem

In a monolith, debugging is straightforward: one server, one log file, one place to look. In microservices, a single user request may touch 10+ services across dozens of servers.

```
User reports: "Checkout is slow — takes 8 seconds"

Without observability:
  Which service is slow? Unknown.
  Which database query? Unknown.
  Was it a network issue? Unknown.
  Is it happening for all users or just some? Unknown.
  Did it start after a deployment? Unknown.

  You're blind. Best guess: deploy a revert and hope.

With observability:
  Trace for that user's request shows:
  API Gateway:        12ms
  Order Service:      45ms
  → Payment Service:  7,823ms  ← BOTTLENECK
    → Stripe API:     7,801ms  ← external call is slow
  → Inventory Service: 23ms
  Total:              7,903ms

  Root cause: Stripe API latency spike
  Action: increase timeout, add fallback, contact Stripe
  Time to diagnosis: seconds, not hours
```

---

## The Three Pillars of Observability

```
Logs:    What happened (events, errors, state changes)
Metrics: How much / how often (counters, gauges, histograms)
Traces:  How long / in what order (request flow across services)

Each pillar answers different questions:
  Logs:    "What error occurred in payment service at 10:32am?"
  Metrics: "What is the error rate of payment service over time?"
  Traces:  "Which service is causing the 8-second checkout latency?"

All three are necessary. One without the others is incomplete.
```

---

## Pillar 1: Logs

Structured, searchable records of events.

### Structured Logging

```java
// Unstructured (hard to search):
log.info("Order 123 created for user 456 with total $99.99");

// Structured (JSON, easily searchable):
log.info("Order created",
    kv("orderId", "123"),
    kv("userId", "456"),
    kv("total", 99.99),
    kv("currency", "USD"),
    kv("traceId", MDC.get("traceId")),
    kv("spanId", MDC.get("spanId"))
);

// Output:
{
  "timestamp": "2026-02-14T10:00:00.123Z",
  "level": "INFO",
  "service": "order-service",
  "traceId": "abc123def456",
  "spanId": "789xyz",
  "orderId": "123",
  "userId": "456",
  "total": 99.99,
  "currency": "USD",
  "message": "Order created"
}
```

### Log Correlation

```java
// Trace ID in every log line across all services
// When debugging, grep by traceId → see all logs for one request

@Component
public class TraceIdFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                         FilterChain chain) throws IOException, ServletException {
        String traceId = ((HttpServletRequest) request)
            .getHeader("X-Trace-ID");

        if (traceId == null) {
            traceId = UUID.randomUUID().toString();
        }

        MDC.put("traceId", traceId);
        ((HttpServletResponse) response).setHeader("X-Trace-ID", traceId);

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear(); // always clean up
        }
    }
}
```

### Log Levels — Use Correctly

```
ERROR: System cannot function, immediate attention required
  → DB connection pool exhausted
  → Critical dependency unavailable
  → Data corruption detected

WARN:  System functions but something is wrong
  → Circuit breaker opened
  → Retry attempt N (before giving up)
  → Rate limit approaching threshold
  → Slow query (> 1 second)

INFO:  Normal operations worth recording
  → Order created (key business events)
  → Service started/stopped
  → Config loaded

DEBUG: Detailed information for debugging
  → Request/response payloads
  → SQL queries executed
  → Cache hit/miss
  → Never in production (too verbose, performance impact)

TRACE: Most detailed (method entry/exit)
  → Only in development
  → Never in production

Production log level: INFO (with WARN/ERROR escalation to alerts)
```

### ELK / EFK Stack

```
Centralized log aggregation:

Each service → [Filebeat/Fluentd] → [Kafka] → [Logstash/Fluentd] → [Elasticsearch] → [Kibana]

Why Kafka in the middle:
  Buffer spikes (log burst doesn't overwhelm Elasticsearch)
  Multiple consumers (security team, on-call, analytics)
  Replay logs if Elasticsearch was down

Kibana:
  Full-text search across all services
  Filter by: service, level, traceId, userId, time range
  "Show me all ERROR logs from payment-service in last hour"
  "Show me all logs with traceId=abc123"
```

---

## Pillar 2: Metrics

Numerical measurements over time.

### Metric Types

```
Counter:   Monotonically increasing number
  orders.created.total = 1,847,293
  http.requests.total{status=500} = 347

  Never decreases (resets only on restart)
  Good for: request counts, error counts, events

Gauge:     Current value that can go up or down
  jvm.memory.used = 2.3GB
  db.connections.active = 45
  queue.depth = 1,234

  Good for: current state, resource utilization

Histogram: Distribution of values, with buckets
  http.request.duration.seconds{le="0.1"} = 9,234
  http.request.duration.seconds{le="0.5"} = 9,891
  http.request.duration.seconds{le="1.0"} = 9,954
  http.request.duration.seconds{le="+Inf"} = 10,000

  Good for: latency distributions, request sizes
  Enables: P50, P95, P99 calculations

Summary:   Like histogram but calculates quantiles at source
  http.request.duration.p50 = 0.045s
  http.request.duration.p95 = 0.312s
  http.request.duration.p99 = 0.891s

  Good for: when you know which percentiles you need
  Trade-off: can't aggregate across instances (histogram can)
```

### Spring Boot Metrics with Micrometer

```java
@Service
public class OrderService {

    private final MeterRegistry registry;
    private final Counter ordersCreated;
    private final Counter ordersFailedValidation;
    private final Timer orderProcessingTime;
    private final DistributionSummary orderValueDist;

    public OrderService(MeterRegistry registry) {
        this.registry = registry;

        this.ordersCreated = Counter.builder("orders.created")
            .description("Total orders successfully created")
            .tag("service", "order-service")
            .register(registry);

        this.ordersFailedValidation = Counter.builder("orders.failed")
            .description("Orders failed validation")
            .tag("reason", "validation")
            .register(registry);

        this.orderProcessingTime = Timer.builder("order.processing.duration")
            .description("Time to process order end-to-end")
            .publishPercentileHistogram(true)
            .percentiles(0.5, 0.95, 0.99)
            .register(registry);

        this.orderValueDist = DistributionSummary.builder("order.value")
            .description("Distribution of order values in USD")
            .baseUnit("dollars")
            .register(registry);
    }

    public Order createOrder(CreateOrderRequest request) {
        return orderProcessingTime.record(() -> {
            try {
                Order order = doCreateOrder(request);
                ordersCreated.increment();
                orderValueDist.record(order.getTotal().doubleValue());

                // Dynamic tags for per-category metrics
                registry.counter("orders.by.category",
                    "category", order.getCategory()).increment();

                return order;
            } catch (ValidationException e) {
                ordersFailedValidation.increment();
                throw e;
            }
        });
    }
}
```

### The Four Golden Signals (Google SRE)

```
These four metrics define service health:

1. Latency:    How long requests take
   Target:     P99 < 200ms for user-facing APIs
   Alert when: P99 > 1 second for 5 minutes

2. Traffic:    How many requests per second
   Target:     Varies by service
   Alert when: Drops to 0 (no traffic = something is wrong)
               Spikes 3x above baseline (potential attack/bug)

3. Errors:     Error rate (5xx / total requests)
   Target:     < 0.1% for critical services
   Alert when: > 1% for 5 minutes (SLO breach)

4. Saturation: How full is the system?
   CPU utilization, memory usage, queue depth
   Alert when: CPU > 80% sustained
               Queue depth growing without bound
```

```yaml
# Spring Boot
management:
  metrics:
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99

# Grafana dashboard showing all four golden signals per service
```

### Prometheus + Grafana Stack

```yaml
# Prometheus scrape config
scrape_configs:
  - job_name: 'order-service'
    static_configs:
      - targets: ['order-service:8080']
    metrics_path: '/actuator/prometheus'
    scrape_interval: 15s
```

```
# PromQL queries for key metrics:
# Error rate
rate(http_server_requests_seconds_count{status=~"5.."}[5m]) /
rate(http_server_requests_seconds_count[5m])

# P99 latency
histogram_quantile(0.99,
  rate(http_server_requests_seconds_bucket[5m]))

# Request rate
rate(http_server_requests_seconds_count[1m])

# JVM memory
jvm_memory_used_bytes / jvm_memory_max_bytes
```

---

## Pillar 3: Distributed Tracing

Track a single request as it flows through multiple services.

### Core Concepts

```
Trace:  Represents a complete end-to-end request
        (one user clicking "Place Order")

Span:   A single unit of work within a trace
        (one service call, one DB query, one Kafka message)
        Has: start time, duration, service name, operation name

Parent/Child: Spans form a tree hierarchy

Trace ID:  Unique ID for the entire trace (propagated in headers)
Span ID:   Unique ID for one span
Parent Span ID: ID of the parent span (links tree together)
```

### Trace Structure

```
Trace ID: abc123

[API Gateway]               0ms ──────────────────── 950ms
  [Order Service]          10ms ─────────── 940ms
    [User Service call]    15ms ── 45ms
    [Payment Service call] 50ms ──────────── 900ms  ← slow!
      [Stripe API call]    55ms ────────── 895ms    ← root cause
    [Inventory call]      905ms ─ 925ms
    [DB INSERT]           926ms ─ 935ms
  [Response]              940ms ─ 950ms

Flame graph makes bottleneck immediately obvious:
Payment Service / Stripe API = 840ms out of 950ms total
```

### OpenTelemetry — The Standard

```java
// Spring Boot with OpenTelemetry

// dependencies:
//   implementation 'io.micrometer:micrometer-tracing-bridge-otel'
//   implementation 'io.opentelemetry:opentelemetry-exporter-otlp'

// application.yml
management:
  tracing:
    sampling:
      probability: 0.1  # trace 10% of requests (adjust per traffic)

spring:
  application:
    name: order-service

// Spans auto-created for:
// - HTTP requests (incoming and outgoing)
// - Database queries (JDBC)
// - Kafka producer/consumer
// - Redis operations
// - Scheduled methods

// Custom spans for business operations
@Service
public class OrderService {

    private final Tracer tracer;

    public Order processOrder(CreateOrderRequest request) {
        // Create custom span for business operation
        Span span = tracer.nextSpan().name("order.process").start();

        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.tag("orderId", request.getOrderId());
            span.tag("userId", request.getUserId());
            span.tag("itemCount", String.valueOf(request.getItems().size()));

            Order order = doProcess(request);

            span.tag("status", "success");
            span.tag("total", order.getTotal().toString());
            return order;

        } catch (Exception e) {
            span.tag("error", e.getMessage());
            span.tag("status", "failure");
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### Trace Context Propagation

```
Trace ID must be passed between services in HTTP headers:

Order Service → Payment Service (HTTP call):
  Request headers:
  traceparent: 00-abc123def456-789xyz-01
               ↑  ↑           ↑      ↑
              ver trace_id  span_id flags

  Payment Service:
  Reads traceparent header
  Creates new span with parent = 789xyz
  Continues the trace

For Kafka messages:
  Trace context added to Kafka headers automatically
  Consumer reads headers → continues trace in new span

W3C TraceContext standard:
  traceparent header is the standard (all major frameworks support)
  OpenTelemetry propagates automatically for HTTP and Kafka
```

### Sampling Strategy

```
Tracing every request at high traffic = expensive:
  1M requests/sec × trace overhead = significant CPU/storage

Sampling strategies:

Head-based sampling (decide at trace start):
  10% random sample: trace 100K/sec out of 1M
  Simple, low overhead
  Misses: rare errors may not be sampled

Tail-based sampling (decide after trace completes):
  Capture all traces briefly
  After completion: keep if:
    → Error occurred
    → P99 exceeded (slow)
    → High-value user
    → Otherwise, discard most

  Better: captures all errors and slow traces
  More complex: need to buffer complete traces

Spring Boot default: 10% probability sampling
  management.tracing.sampling.probability: 0.1

For errors: always sample
  span.tag("error", true) → always kept regardless of probability
```

---

## The Observability Stack

### Production Architecture

```
[Services] → OpenTelemetry Collector → Fan out to:
               ├── Prometheus   → Grafana (metrics dashboards)
               ├── Jaeger/Tempo → Grafana (trace visualization)
               └── Loki         → Grafana (log aggregation)

Or cloud-native:
  AWS: CloudWatch Metrics + CloudWatch Logs + X-Ray (traces)
  GCP: Cloud Monitoring + Cloud Logging + Cloud Trace
  Azure: Monitor + Log Analytics + Application Insights

Third-party:
  Datadog: metrics + logs + traces (unified, expensive)
  New Relic: similar to Datadog
  Dynatrace: AI-powered, automatic instrumentation
  Honeycomb: traces + high-cardinality event analytics
```

### OpenTelemetry Collector

```yaml
# otel-collector.yaml
receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 1024

  # Filter sensitive data before export
  attributes:
    actions:
      - key: http.request.header.authorization
        action: delete  # never log auth headers
      - key: db.statement
        action: hash    # hash SQL (may contain PII)

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"

  jaeger:
    endpoint: jaeger:14250

  loki:
    endpoint: http://loki:3100/loki/api/v1/push

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch, attributes]
      exporters: [jaeger]
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    logs:
      receivers: [otlp]
      processors: [batch, attributes]
      exporters: [loki]
```

---

## Alerting — Connecting Metrics to Action

```
SLO (Service Level Objective): target for reliability
  "99.9% of requests succeed" (allows 0.1% errors)
  "P99 latency < 200ms for 95% of 5-minute windows"

SLI (Service Level Indicator): actual measurement
  Current error rate: 0.05% ← within SLO
  Current P99: 187ms ← within SLO

Error Budget: how much you can fail within SLO
  99.9% uptime → 0.1% failures allowed
  43.8 minutes/month of failures allowed

  If error budget consumed → freeze deployments
  If error budget healthy → invest in features

Alert design:
  Alert on symptoms, not causes

  BAD:  CPU > 80% (cause, may not affect users)
  GOOD: Error rate > 1% for 5 minutes (symptom, users affected)

  BAD:  Kafka consumer lag > 1000 messages (cause)
  GOOD: Order processing P99 > 5 seconds (symptom)

Alerting levels:
  Page immediately:  Error rate > 5%, P99 > 5s (users severely impacted)
  Alert soon:        Error rate > 1%, P99 > 1s (SLO burning fast)
  Ticket:            Slow query, high memory, increasing error trend
```

---

## Connecting the Three Pillars

The real power of observability: jump from one pillar to another.

```
Scenario: "Something is wrong with checkout at 10:32am"

Start with metrics:
  Grafana: error rate spike at 10:31am in payment-service
  Latency also spiked: P99 jumped from 150ms to 8s

Drill into traces:
  Filter traces: payment-service, 10:31-10:33, errors only
  Find trace: order_id=789, took 8.2s, ERROR
  Trace shows: Stripe API call timed out at 7.8s

Jump to logs:
  Filter: traceId=abc123, service=payment-service
  Find log: "Stripe API timeout after 7800ms: connection refused"
  Find log: "Circuit breaker OPENED for stripe-api"

Root cause identified in 2 minutes:
  Stripe API outage caused timeout
  Circuit breaker eventually opened (should have faster)
  Fix: lower timeout threshold, check Stripe status page
```

---

## Key Takeaways

```
Observability: ability to understand internal state from outputs
Three pillars: Logs + Metrics + Traces (all three needed)

Logs:
  Structured JSON with trace ID in every line
  Levels: ERROR/WARN/INFO in production, DEBUG/TRACE never
  ELK/EFK stack: centralized search across all services
  Correlation: grep by traceId to see full request flow

Metrics:
  Counter: monotonic counts (requests, errors)
  Gauge: current values (memory, connections)
  Histogram: latency distributions (enables percentiles)

  Four Golden Signals: Latency, Traffic, Errors, Saturation
  Stack: Prometheus (collection) → Grafana (visualization)
  Alerting: on symptoms (error rate, latency), not causes (CPU)

Distributed Tracing:
  Trace: entire request end-to-end
  Span: one operation within trace
  TraceID propagated through all services in headers

  OpenTelemetry: standard, auto-instruments Spring Boot
  Jaeger/Tempo: trace storage and visualization
  Sampling: 10% random, or tail-based (keep errors + slow)

  Custom spans: business operations worth tracking
  Tags: orderId, userId, amount — key business context

Alerting:
  SLO: target (99.9% success, P99 < 200ms)
  SLI: measurement (current error rate)
  Error budget: how much failure is allowed
  Alert on symptoms → pages; alert on causes → tickets

Production stack:
  OpenTelemetry → Prometheus + Grafana + Jaeger
  Or: Datadog / New Relic (managed, unified)

Power of connected pillars:
  Metrics alert → drill into traces → jump to logs
  Root cause in minutes, not hours
```
