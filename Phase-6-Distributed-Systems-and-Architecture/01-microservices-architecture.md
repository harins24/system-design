# Microservices Architecture

**Phase 6 — Distributed Systems + Architecture | Topic 1**

---

## What is Microservices Architecture?

Microservices architecture is an approach to building software where an application is decomposed into small, independent services, each responsible for a specific business capability, communicating over well-defined APIs.

```
Monolith:
  [Single Deployable Unit]
  ├── User Management
  ├── Order Processing
  ├── Payment Handling
  ├── Inventory Management
  ├── Notification System
  └── Analytics

All code in one codebase
One database
Deploy everything together
Scale everything together

Microservices:
  [User Service]       [Order Service]      [Payment Service]
  [own DB]             [own DB]             [own DB]

  [Inventory Service]  [Notification Service] [Analytics Service]
  [own DB]             [own DB]               [own DB]

Each service independent
Each service owns its data
Deploy each service independently
Scale each service independently
```

---

## Why Microservices?

```
Monolith problems at scale:

Team scaling:
  100 engineers working on same codebase
  Merge conflicts daily
  "Who changed the payment module?"
  One team's bug breaks everyone's deployment

Technical scaling:
  Payment processing needs 50 servers
  User lookup needs 5 servers
  Must scale ENTIRE monolith just to scale one component
  Waste: 45 servers running idle user lookup code

Deployment scaling:
  Change one line in notifications
  Must redeploy entire application
  Test everything before deploying anything
  Deploy once a week (if lucky)

Technology:
  Stuck with Java forever
  Can't use Python for ML component
  Can't use Node.js for real-time features
  One tech stack for everything

Microservices solve each:
  Teams own services → parallel development
  Services scale independently → efficient resources
  Services deploy independently → continuous delivery
  Services use any language → right tool per job
```

---

## Microservices Design Principles

### 1. Single Responsibility

Each service owns one business capability. Not too big, not too small.

```
Right size indicators:
  ✅ Can be rewritten in 2 weeks by 1-2 engineers
  ✅ One team owns it end-to-end
  ✅ One reason to change
  ✅ One deployable unit

Too big (should split):
  "Order Service" handles orders + payments + shipping

Too small (should merge, nanoservices anti-pattern):
  "Tax Calculation Service" (30 lines of code)
  "Currency Rounding Service" (10 lines of code)
  Network overhead exceeds service value
```

### 2. Database per Service

Each service owns its data. No shared databases.

```
✅ Correct:
  Order Service     → order-db (PostgreSQL)
  Payment Service   → payment-db (PostgreSQL)
  Product Service   → product-db (MongoDB)
  User Service      → user-db (PostgreSQL)

  Services communicate via APIs, not shared DB

❌ Wrong (shared database anti-pattern):
  All services → same database

  Problems:
  Schema change for Order Service breaks Payment Service
  Order Service can bypass Payment Service validation
  Services tightly coupled through shared schema
  Can't scale DBs independently
  Can't choose different DB types
```

### 3. API First

Define the API contract before writing implementation.

```
Define OpenAPI spec first:
  What endpoints?
  What request/response shape?
  What error codes?

Then build implementation to satisfy contract.
Allows teams to work in parallel (mock the contract).
Contract changes = versioned API (/v2/).
```

### 4. Failure Isolation

One service failing should not cascade to others.

```
Without isolation:
  Payment Service → slow
  Order Service calls Payment Service → hangs waiting
  Order Service thread pool exhausted
  Order Service → down
  User Service calls Order Service → hangs
  User Service → down
  Entire system down because Payment Service was slow

With isolation (Circuit Breaker):
  Payment Service → slow
  Order Service circuit breaker opens
  Order Service returns fallback response
  Order Service remains healthy
  User Service unaffected
  Payment Service recovers → circuit closes → normal operation
```

### 5. Decentralized Data Management

Each service chooses the right database for its needs.

```
User Service:       PostgreSQL (ACID, complex queries)
Product Service:    MongoDB (flexible schema, nested docs)
Session Service:    Redis (TTL, fast lookup)
Search Service:     Elasticsearch (full-text)
Event Log:          Cassandra (time-series, write-heavy)
Recommendation:     Neo4j (graph relationships)
```

---

## Service Communication Patterns

### Synchronous (REST / gRPC)

```
Use when: caller needs immediate response
          Request/response semantics required

REST:
  Order Service → GET /users/{id} → User Service
  Waits for response before continuing

gRPC:
  Better for internal service-to-service
  Binary protocol (smaller payload)
  HTTP/2 (multiplexing)
  Generated clients (type safety)
  Streaming support

Problems with sync communication:
  Temporal coupling: both services must be up simultaneously
  Latency chains: A calls B calls C calls D
                  Total latency = A + B + C + D
  Cascading failures: D slow → C slow → B slow → A slow
```

### Asynchronous (Kafka / Message Queue)

```
Use when: caller doesn't need immediate response
          Event notification semantics
          Fan-out to multiple services

Order Service publishes OrderCreated to Kafka
→ Returns to caller immediately
→ Payment, Inventory, Notification process independently

Benefits:
  Temporal decoupling: services don't need to be up simultaneously
  No latency chain: each service processes at its own pace
  Natural retry: Kafka retains messages
  Fan-out: one event → many consumers
```

### Service Mesh

```
Sidecar proxy injected alongside each service:
  [Service] + [Envoy Sidecar]

Sidecar handles:
  Service discovery (find other services)
  Load balancing
  Circuit breaking
  Retries
  mTLS (mutual TLS between services)
  Observability (metrics, traces, logs)

Application code doesn't handle these concerns
Infrastructure layer handles them transparently

Istio:   Most popular, feature-rich, complex
Linkerd: Simpler, lighter weight
Consul:  HashiCorp, also service discovery

Your services talk to sidecar (localhost)
Sidecar handles all networking concerns
```

---

## Service Discovery

### Client-Side Discovery

```
Service registry stores: {service-name → [ip:port list]}

Service A wants to call Service B:
  1. Query registry: "Where is payment-service?"
  2. Registry returns: ["10.0.1.5:8080", "10.0.1.6:8080", "10.0.1.7:8080"]
  3. Service A picks one (round robin, random)
  4. Service A calls directly

Registration:
  Service B starts → registers itself in Eureka/Consul
  Service B healthy → sends heartbeat every 30s
  Service B dies → heartbeat stops → registry removes after TTL
```

```java
// Spring Cloud Eureka
@EnableEurekaServer  // on registry service
@EnableEurekaClient  // on each microservice

// Service A calls Service B by name
@LoadBalanced  // Ribbon/Spring LoadBalancer
RestTemplate restTemplate;

restTemplate.getForObject(
    "http://payment-service/api/payments/{id}",
    Payment.class, paymentId);
// Spring resolves "payment-service" → actual IP via Eureka
```

### Server-Side Discovery (Load Balancer)

```
Service A → [Load Balancer / API Gateway] → Service B

Load balancer queries registry → routes to healthy instance
Service A doesn't know Service B's address
Service A only knows the load balancer address

Kubernetes:
  Service B has a ClusterIP (stable virtual IP)
  kube-proxy routes ClusterIP → actual pod IPs

  DNS: payment-service.default.svc.cluster.local
       → resolves to ClusterIP
       → kube-proxy routes to healthy pod

  Service A just uses service name → Kubernetes handles rest
  No Eureka/Consul needed in Kubernetes
```

---

## API Gateway in Microservices

```
❌ Mobile app calls: user-service:8081, order-service:8082,
                     payment-service:8083...
  → Client knows internal topology
  → Changing service address breaks clients
  → No central auth, rate limiting, logging

✅ All external traffic through API Gateway:
  Mobile app → API Gateway → user-service
                           → order-service
                           → payment-service

API Gateway responsibilities:
  Route /api/users/* → user-service
  Route /api/orders/* → order-service
  Auth: validate JWT once (not per service)
  Rate limiting: per client
  SSL termination
  Request/response transformation
  Aggregation: combine responses from multiple services
  Observability: log all external requests
```

---

## Data Consistency Across Services

The hardest microservices problem. No shared DB = no ACID transactions.

### Eventual Consistency

```
Services update their own DBs
Events propagate changes via Kafka
Other services update eventually

Example: Order creation
  Order Service: saves order (status: PENDING)
  Publishes: OrderCreated → Kafka

  Payment Service: receives OrderCreated
  Charges card
  Publishes: PaymentSucceeded

  Inventory Service: receives PaymentSucceeded
  Decrements stock

  Order Service: receives PaymentSucceeded
  Updates order (status: CONFIRMED)

  All consistent eventually
  Brief window where order is PENDING but payment succeeded
  Acceptable for most use cases
```

### Saga Pattern

Manage distributed transactions via compensating actions.

```
If step fails → run compensating transactions in reverse:
  Payment failed → Cancel order
  Inventory depleted → Refund payment → Cancel order

Two implementations:
  Choreography: services react to events (simpler, less control)
  Orchestration: central saga orchestrator (more control, single point)
```

### Outbox Pattern

```
Problem: How to atomically write to DB AND publish to Kafka?
  Order Service:
    1. Save order to DB ✅
    2. Publish to Kafka ❌ (Kafka down)
    → Order in DB but event never published
    → Other services never notified

Outbox Pattern:
  1. Save order AND outbox message in SAME DB transaction:
     INSERT INTO orders (...)
     INSERT INTO outbox (event_type, payload, status=PENDING)
     COMMIT; ← atomic

  2. Separate process (CDC or polling) reads outbox table
     Publishes pending messages to Kafka
     Marks as published

  Guarantees: order saved ↔ event published
              Either both happen or neither

Implemented via:
  Debezium (Change Data Capture):
    Reads PostgreSQL WAL → publishes to Kafka
    Zero application code change

  Polling publisher:
    @Scheduled job reads unpublished outbox messages
    Publishes to Kafka → marks as published
    Simple but higher latency
```

---

## Observability in Microservices

With 50 services, you can't debug by reading logs on one server.

### The Three Pillars

```
Logs:       What happened in each service
Metrics:    Numerical measurements over time (latency, error rate)
Traces:     End-to-end request flow across services
```

### Distributed Tracing

```
User request hits API Gateway
→ Order Service (trace continues)
  → User Service (trace continues)
  → Payment Service (trace continues)
    → Bank API (trace continues)
  → Inventory Service (trace continues)
→ Response to user

Without tracing:
  "Why is this request slow?"
  Check logs on API Gateway... check Order Service logs...
  Time-consuming, error-prone

With distributed tracing (Jaeger, Zipkin):
  Every request has a Trace ID propagated in headers
  Each service records a Span with timing
  Trace viewer shows: entire request as flame graph

  Payment Service took 450ms → bottleneck identified instantly

Spring Boot:
  implementation 'io.micrometer:micrometer-tracing-bridge-otel'
  implementation 'io.opentelemetry:opentelemetry-exporter-jaeger'

  // Trace ID auto-propagated via OpenTelemetry
  // All logs include traceId automatically:
  // 2026-02-14 10:00:00 [traceId=abc123] Payment processed
```

### Service Health Metrics

```java
// Spring Boot Actuator + Micrometer
@Component
public class OrderMetrics {

    private final MeterRegistry registry;
    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Timer orderProcessingTime;

    public OrderMetrics(MeterRegistry registry) {
        this.ordersCreated = Counter.builder("orders.created")
            .tag("service", "order-service")
            .register(registry);

        this.ordersFailed = Counter.builder("orders.failed")
            .tag("service", "order-service")
            .register(registry);

        this.orderProcessingTime = Timer.builder("orders.processing.time")
            .register(registry);
    }

    public Order createOrder(CreateOrderRequest request) {
        return orderProcessingTime.record(() -> {
            try {
                Order order = processOrder(request);
                ordersCreated.increment();
                return order;
            } catch (Exception e) {
                ordersFailed.increment();
                throw e;
            }
        });
    }
}
// Prometheus scrapes → Grafana dashboards
// Alert: error rate > 1% for 5 minutes → page on-call
```

---

## Microservices Anti-Patterns

```
Distributed Monolith:
  Services split by technical layer, not business capability
  UI Service, Business Logic Service, Data Service
  All deployed together anyway
  All have same DB
  "Microservices" in name only
  All microservices complexity, none of the benefits

Chatty Services:
  Order Service calls User Service 5 times per request
  Network overhead > computation
  Solution: batch calls, cache locally, denormalize data

Shared Library Coupling:
  "Common library" used by all services
  Change common library → redeploy ALL services
  Defeats independent deployment
  Prefer: API contracts over shared code
  Exception: shared utility libs (logging, metrics) are OK

Too Fine-Grained (Nanoservices):
  PaymentValidationService, TaxCalculationService,
  CurrencyRoundingService
  Network hops cost more than computation
  Merge into coherent Payment Service

Direct DB Access Across Services:
  Order Service queries payment_db directly
  Bypasses Payment Service API
  Breaks encapsulation
  Schema changes silently break Order Service
```

---

## Monolith vs Microservices Decision

```
Start with monolith when:
  ✅ Early stage, domain not understood yet
  ✅ Small team (< 10 engineers)
  ✅ Simple domain
  ✅ Fast time-to-market priority
  ✅ "Strangler Fig" — migrate to microservices when needed

Move to microservices when:
  ✅ Clear bounded contexts identified
  ✅ Multiple teams stepping on each other
  ✅ Different scaling requirements per component
  ✅ Different deployment cadences needed
  ✅ Organization can support distributed systems ops
  ✅ Reliability investment is justified

Don't go microservices because:
  ❌ "It's modern"
  ❌ "Netflix does it"
  ❌ Small team
  ❌ Domain is poorly understood
  ❌ No observability tooling
  ❌ No CI/CD pipeline

"Microservices are not a starting point.
 They're a destination you grow into."
```

---

## Key Takeaways

```
Microservices: decompose app into independent business services
Each service: owns code, DB, deployment, scaling

Why:
  Independent team ownership
  Independent scaling (pay only for what you need)
  Independent deployment (deploy one service, not all)
  Technology flexibility per service
  Fault isolation

Design principles:
  Single responsibility (one business capability)
  Database per service (no shared DB)
  API first (contract before implementation)
  Failure isolation (circuit breakers)
  Decentralized data (right DB per service)

Communication:
  Sync (REST/gRPC): caller needs response, latency acceptable
  Async (Kafka):    events, fan-out, no immediate response needed
  Service mesh:     cross-cutting concerns handled by sidecar

Data consistency:
  No distributed ACID → eventual consistency via events
  Saga: compensating transactions for distributed workflows
  Outbox: atomic DB write + event publish

Service discovery:
  Client-side: Eureka (Spring Cloud)
  Server-side: Kubernetes Service DNS

Observability:
  Logs + Metrics + Traces (three pillars)
  Distributed tracing: trace ID across all services
  Spring Boot Actuator + Micrometer + Jaeger

Anti-patterns:
  Distributed monolith, chatty services,
  shared library coupling, nanoservices,
  cross-service DB access

Decision: monolith first, microservices when justified
```
