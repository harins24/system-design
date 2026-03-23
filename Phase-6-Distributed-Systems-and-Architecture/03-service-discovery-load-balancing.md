# Service Discovery and Load Balancing

**Phase 6 — Distributed Systems + Architecture | Topic 3**

---

## The Problem

In a microservices system, services need to find and communicate with each other.

```
Static configuration (naive approach):
  Order Service config:
    payment.service.url=http://10.0.1.5:8080
    inventory.service.url=http://10.0.1.6:8080

Problems:
  IP 10.0.1.5 changes when Payment Service restarts
  Payment Service scales to 5 instances — which IP?
  Payment Service instance dies — code still has dead IP
  Deploy to different environment — all IPs different
  Kubernetes pods get new IPs on every restart

Need: dynamic, automatic discovery of where services are
```

---

## Service Discovery — Two Models

### Client-Side Discovery

The client queries a registry and chooses which instance to call.

```
[Service Registry]
  payment-service → [10.0.1.5:8080, 10.0.1.6:8080, 10.0.1.7:8080]
  inventory-service → [10.0.1.8:8080, 10.0.1.9:8080]

Order Service wants to call Payment Service:
  1. Query registry: "Where is payment-service?"
  2. Get list: [10.0.1.5, 10.0.1.6, 10.0.1.7]
  3. Pick one (load balancing algorithm)
  4. Call directly: http://10.0.1.5:8080/api/payments

Registration:
  Payment Service starts → registers with registry
  Sends heartbeat every 30s → "still alive"
  Dies → heartbeat stops → registry removes after TTL
```

```java
// Netflix Eureka — Spring Cloud

// Eureka Server (registry)
@SpringBootApplication
@EnableEurekaServer
public class RegistryApplication { ... }

// Service registration (in each microservice)
@SpringBootApplication
@EnableDiscoveryClient
public class PaymentServiceApplication { ... }

# application.yml — Payment Service
spring:
  application:
    name: payment-service
eureka:
  client:
    service-url:
      defaultZone: http://eureka-server:8761/eureka
  instance:
    prefer-ip-address: true
    lease-renewal-interval-in-seconds: 30
    lease-expiration-duration-in-seconds: 90

// Client calling another service by name
@Bean
@LoadBalanced  // enables service-name resolution via Eureka
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// Use service name, not IP
restTemplate.getForObject(
    "http://payment-service/api/payments/{id}",
    Payment.class, paymentId);
// Spring resolves "payment-service" → actual IP via Eureka
```

```
Pros/Cons:
✅ Client has full control over load balancing algorithm
✅ One fewer network hop (direct client → server)
✅ Can implement custom load balancing logic

❌ Client must implement service discovery logic
❌ Registry client library needed per language
❌ Client must handle stale registry data
```

### Server-Side Discovery

A load balancer queries the registry and routes requests. Client knows only the load balancer address.

```
[Order Service] → [Load Balancer / API Gateway]
                         ↓ queries registry
                  [Service Registry]
                         ↓ routes to healthy instance
                  [Payment Service Instance 2]

Client:
  Only knows: http://payment-lb.internal
  Doesn't know: how many instances, which one handles request
```

**Kubernetes implements server-side discovery natively:**

```
Payment Service deployed as Kubernetes Deployment:
  3 pods running: 10.0.1.5, 10.0.1.6, 10.0.1.7

Kubernetes Service (stable virtual IP):
  payment-service → ClusterIP: 10.96.0.5 (never changes)

kube-proxy routes: 10.96.0.5 → one of [10.0.1.5, 10.0.1.6, 10.0.1.7]

DNS:
  payment-service.default.svc.cluster.local → 10.96.0.5

Order Service just calls:
  http://payment-service/api/payments
  → Kubernetes handles discovery + load balancing transparently
  → Order Service never knows pod IPs
```

```
Pros/Cons:
✅ Client is simple (no registry library needed)
✅ Works with any language/framework
✅ Infrastructure handles routing complexity
✅ Native in Kubernetes

❌ Extra network hop (LB between client and server)
❌ Load balancer must be highly available
❌ Less client control over routing decisions
```

---

## Load Balancing Algorithms — In Depth

### Round Robin

```
Requests distributed sequentially across instances:
  Request 1 → Instance A
  Request 2 → Instance B
  Request 3 → Instance C
  Request 4 → Instance A (cycles back)
  Request 5 → Instance B
  ...

✅ Simple, even distribution
✅ No state required
❌ Ignores instance load/capacity
   (Instance A: 100% CPU, Instance B: 10% CPU — still equal share)
```

### Weighted Round Robin

```
Instances assigned weights proportional to capacity:
  Instance A: weight 3 (powerful server)
  Instance B: weight 1 (small server)

  Distribution: A, A, A, B, A, A, A, B...
  Instance A handles 75% of traffic, Instance B handles 25%

✅ Accounts for heterogeneous capacity
❌ Static weights — doesn't adapt to real-time load
```

### Least Connections

```
Route to instance with fewest active connections:
  Instance A: 50 active connections
  Instance B: 12 active connections  ← route here
  Instance C: 31 active connections

  New request → Instance B (lowest connections)

✅ Accounts for varying request duration
✅ Naturally balances load dynamically
❌ Requires tracking connection counts
❌ Doesn't distinguish "active" from "slow/stuck"
```

### Least Response Time

```
Combine: active connections × average response time
  Instance A: 10 connections × 50ms = 500 (score)
  Instance B: 5  connections × 20ms = 100 (score) ← route here
  Instance C: 8  connections × 40ms = 320 (score)

  Route to lowest score (fastest effective service)

✅ Accounts for both load AND speed
✅ Naturally penalizes slow instances
❌ More complex to implement
❌ Response time measurement adds overhead
```

### IP Hash (Sticky Sessions)

```
hash(client IP) % num_instances → always same instance

Client 10.0.5.2 → always Instance A
Client 10.0.5.3 → always Instance B

✅ Session affinity (stateful apps, cached user data)
✅ Predictable routing for debugging
❌ Uneven distribution if clients clustered
❌ Defeats purpose if instance fails (sessions lost anyway)
❌ Not suitable for stateless services (no reason for stickiness)
```

### Consistent Hashing (for Caches)

```
Used when routing to cache nodes — want same key on same node:
  hash(request.key) → always same cache node

  Cache miss only on first request to that node
  Adding/removing nodes: minimal key remapping

Used by: distributed caches (Redis Cluster), CDN routing
Not for: general service load balancing
```

### Random

```
Pick any instance at random:
  Statistically approximates round robin at scale
  Very simple to implement

✅ No state, minimal overhead
✅ Resistant to adversarial patterns
❌ May unevenly load at small scale
```

---

## Health Checks — Routing Around Failures

Load balancers must know which instances are healthy.

### Active Health Checks

```
Load balancer periodically pings each instance:
  GET http://payment-service-1:8080/health
  Response: 200 OK → healthy
  Response: 500 / timeout → unhealthy

  Check interval: 10 seconds
  Unhealthy threshold: 2 consecutive failures → mark unhealthy
  Healthy threshold: 2 consecutive successes → mark healthy again

Spring Boot Actuator:
  GET /actuator/health

  Response (healthy):
  {
    "status": "UP",
    "components": {
      "db": {"status": "UP"},
      "redis": {"status": "UP"},
      "kafka": {"status": "UP"}
    }
  }

  Response (unhealthy — DB down):
  {
    "status": "DOWN",
    "components": {
      "db": {"status": "DOWN", "details": {"error": "Connection refused"}}
    }
  }
  HTTP 503 → load balancer stops routing here
```

### Passive Health Checks

```
Monitor real traffic responses:
  5xx response from instance → increment error counter
  Error rate > 5% over 30 seconds → mark unhealthy

Advantage: No extra health check traffic
Advantage: Detects slow instances (not just dead ones)
Disadvantage: Real users experience failures before detection
```

### Readiness vs Liveness

```
Kubernetes has two separate health checks:

Liveness probe: "Is this container alive?"
  Fails → Kubernetes restarts the container
  Use for: deadlocks, infinite loops, unrecoverable errors

  GET /actuator/health/liveness
  Check: can the process respond? (just "I'm alive")

Readiness probe: "Is this container ready to receive traffic?"
  Fails → Remove from load balancer (but don't restart)
  Use for: warming up, DB connection not ready,
           dependency unavailable

  GET /actuator/health/readiness
  Check: dependencies healthy? DB connected? Cache available?

Startup probe: "Has this container finished starting?"
  Slow-starting apps → don't kill before they're ready
  Checked once at startup
```

```yaml
# Kubernetes deployment
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

---

## Layer 4 vs Layer 7 Load Balancing

```
Layer 4 (Transport Layer):
  Routing based on: IP address + TCP/UDP port
  Sees: raw TCP connections
  Doesn't understand: HTTP content

  "Route all traffic to port 8080 → one of these IPs"

  Very fast (no packet inspection)
  AWS NLB (Network Load Balancer)

  Use: TCP services, database proxies, anything not HTTP
       Extreme throughput requirements

Layer 7 (Application Layer):
  Routing based on: HTTP method, path, headers, cookies, body
  Sees: HTTP requests/responses
  Understands: URL paths, JWT tokens, content types

  Route /api/orders → order-service
  Route /api/payments → payment-service
  Route if JWT role=admin → admin-service

  AWS ALB (Application Load Balancer)
  Nginx, HAProxy, Envoy

  Use: HTTP microservices (almost everything)
       SSL termination, path-based routing, auth
```

---

## Service Mesh — Beyond Simple Load Balancing

```
Without service mesh:
  Each service implements:
  → Service discovery
  → Load balancing
  → Circuit breaking
  → Retries
  → mTLS
  → Metrics collection
  → Distributed tracing

  50 services = 50 implementations (in different languages)
  Inconsistent, hard to change globally

With service mesh (Istio):
  Sidecar proxy (Envoy) injected alongside each service pod

  [Order Service Pod]
  ├── Order Service container (your code)
  └── Envoy sidecar (networking layer)

  Your code → localhost → Envoy sidecar → network → destination Envoy → target service

  Envoy handles:
  → Service discovery (via control plane)
  → Load balancing (multiple algorithms)
  → Circuit breaking
  → Automatic retries with backoff
  → mTLS (mutual TLS between all services)
  → Request metrics (latency, error rate)
  → Distributed traces (automatic span creation)

  You get all of this without writing any code
```

### Istio Traffic Management

```yaml
# Route 10% of traffic to canary version
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - match:
    - headers:
        x-canary:
          exact: "true"
    route:
    - destination:
        host: payment-service
        subset: canary
  - route:
    - destination:
        host: payment-service
        subset: stable
      weight: 90
    - destination:
        host: payment-service
        subset: canary
      weight: 10
```

---

## Global Load Balancing

```
DNS-based (Route 53 Latency Routing):
  User in Singapore → DNS lookup → nearest region

  Route 53 policy: latency-based routing
  → Measures latency from user to each region
  → Returns IP of lowest-latency region

  User requests payment-api.yourcompany.com
  → Route 53 measures: us-east-1 = 180ms, ap-southeast-1 = 15ms
  → Returns Singapore ALB IP (15ms wins)
  → User connects to Singapore datacenter

Anycast (Cloudflare / AWS Global Accelerator):
  Same IP advertised from multiple locations
  BGP routing delivers to nearest PoP

  User connects to 1.2.3.4
  BGP: "1.2.3.4 is closest at Singapore"
  → Traffic routed to Singapore automatically

  Benefit: no DNS TTL delay
           instant failover (BGP re-routes in seconds)
           lower latency than DNS-based
```

### Spring Cloud LoadBalancer (Modern Approach)

```java
@Configuration
public class LoadBalancerConfig {

    // Custom load balancing algorithm
    @Bean
    public ReactorLoadBalancer<ServiceInstance> randomLoadBalancer(
            Environment environment,
            LoadBalancerClientFactory loadBalancerClientFactory) {

        String name = environment.getProperty(
            LoadBalancerClientFactory.PROPERTY_NAME);

        return new RandomLoadBalancer(
            loadBalancerClientFactory.getLazyProvider(
                name, ServiceInstanceListSupplier.class),
            name);
    }
}

// WebClient with load balancing
@Bean
@LoadBalanced
public WebClient.Builder loadBalancedWebClientBuilder() {
    return WebClient.builder();
}

@Service
public class PaymentClient {

    private final WebClient webClient;

    public PaymentClient(@LoadBalanced WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("http://payment-service") // service name, not IP
            .build();
    }

    public Mono<Payment> getPayment(String id) {
        return webClient.get()
            .uri("/api/payments/{id}", id)
            .retrieve()
            .bodyToMono(Payment.class);
    }
}
```

---

## Key Takeaways

```
Service Discovery: dynamically find where services are

Client-side (Eureka):
  Client queries registry → gets instance list → picks one
  Client does load balancing
  Spring Cloud: @LoadBalanced + service name in URL

Server-side (Kubernetes):
  Client calls stable address (ClusterIP or DNS)
  Infrastructure routes to healthy instance
  Transparent to application code
  Preferred in Kubernetes environments

Load Balancing Algorithms:
  Round Robin:        simple, even, ignores load
  Weighted RR:        proportional to capacity
  Least Connections:  routes to least busy
  Least Response Time: routes to fastest
  IP Hash:            sticky sessions
  Random:             simple, statistically even

Health Checks:
  Active:    LB pings /health endpoint periodically
  Passive:   LB monitors real traffic error rates
  Readiness: is service ready for traffic? (Kubernetes)
  Liveness:  is service alive? restart if not (Kubernetes)

L4 vs L7:
  L4: fast, TCP-level, no HTTP awareness
  L7: smart, HTTP-aware, path/header routing, SSL termination

Service Mesh (Istio):
  Sidecar (Envoy) handles all networking
  Service discovery, LB, circuit breaking, mTLS, tracing
  Zero application code changes
  Global traffic management (canary, circuit breaking)

Global Load Balancing:
  DNS (Route 53): latency-based routing, DNS TTL delay
  Anycast (Cloudflare/GA): same IP worldwide, instant failover
```
