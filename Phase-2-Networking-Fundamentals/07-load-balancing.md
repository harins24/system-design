# Load Balancing

**Phase 2 — Networking Fundamentals | Topic 7 of 8**

---

## What is Load Balancing?

Load balancing is the process of distributing incoming traffic across multiple servers so no single server becomes a bottleneck or single point of failure.

```
Without Load Balancer:
Users ──────────────────────────► [Server A]  ← overwhelmed
                                  [Server B]  ← idle
                                  [Server C]  ← idle

With Load Balancer:
         ┌──────────────────────► [Server A]  ← ~33% load
Users ──►│Load Balancer ────────► [Server B]  ← ~33% load
         └──────────────────────► [Server C]  ← ~33% load
```

---

## Load Balancing Algorithms

### 1. Round Robin
Requests distributed sequentially in rotation.

```
Request 1 → Server A
Request 2 → Server B
Request 3 → Server C
Request 4 → Server A  ← cycles back
```

**Best for:** Identical hardware, stateless requests of similar complexity.
**Problem:** Doesn't account for server load.

### 2. Weighted Round Robin
Same as round robin but servers with more capacity get proportionally more requests.

```
Server A: weight 3  →  A, A, A, B, A, A, A, B...
Server B: weight 1
```

**Best for:** Heterogeneous server fleets.

### 3. Least Connections
New request goes to the server with the fewest active connections.

```
Server A: 10 active connections
Server B: 3 active connections   ← next request goes here
Server C: 7 active connections
```

**Best for:** Long-lived connections, file uploads, streaming.

### 4. IP Hash (Sticky Sessions)
Hash the client's IP address to always route the same client to the same server.

```
Client IP 192.168.1.1 → hash → always Server A
```

**Best for:** Stateful applications where session data is stored on the server.
**Problem:** Breaks horizontal scaling. **Better solution:** Store session state externally in Redis.

### 5. Least Response Time
Routes to the server with the lowest combination of active connections AND response time.

### 6. Random
Picks a server randomly. Surprisingly effective at large scale (law of large numbers).

### 7. Resource Based (Adaptive)
Load balancer checks actual CPU and memory utilization and routes to the least loaded server.

---

## Layer 4 vs Layer 7 Load Balancing

### Layer 4 Load Balancer (Transport Layer)
Routes based on **IP address and TCP/UDP port only**. Doesn't inspect the content of packets.

```
Decision based on:
  Source IP: 203.45.67.89
  Dest IP:   10.0.1.100
  Port:      443

No inspection of: HTTP headers, URL path, cookies, body
```

**Pros:** Extremely fast, low latency, handles any TCP/UDP protocol.
**Cons:** Can't make intelligent routing decisions based on content. Can't do SSL termination.
**AWS equivalent:** Network Load Balancer (NLB)
**Use when:** Raw TCP performance, non-HTTP protocols (Kafka, database), ultra-low latency.

### Layer 7 Load Balancer (Application Layer)
Routes based on **HTTP content** — URL path, headers, cookies, query parameters.

```
Decision based on:
  URL path:    /api/users → User Service
               /api/orders → Order Service
               /static → CDN / S3
  Headers:     X-Region: EU → EU servers
  Cookies:     session_id → specific server
```

**Pros:** Intelligent content-based routing, SSL termination, authentication, rate limiting, A/B testing, canary deployments.
**Cons:** Higher latency (must read and parse HTTP), more CPU intensive.
**AWS equivalent:** Application Load Balancer (ALB)
**Use when:** HTTP/HTTPS traffic, microservices routing, SSL termination.

---

## Health Checks

Load balancers continuously verify backend servers are healthy:

```
Active Health Check:
  LB sends HTTP GET /health every 30 seconds
  200 OK → server is healthy, keep sending traffic
  Non-200 or timeout → mark unhealthy, stop sending traffic
  After N consecutive successes → mark healthy again
```

```java
// Spring Boot Actuator health endpoint
// ALB health check hits this automatically
GET /actuator/health
→ {"status":"UP","components":{"db":{"status":"UP"},"kafka":{"status":"UP"}}}
```

This enables **zero-downtime deployments** — deploy new version, health check confirms it's ready, LB starts sending traffic, old version gracefully drains.

---

## Global Load Balancing

```
User in Tokyo    ──► Route 53 / Cloudflare
User in London   ──► (global DNS-based LB)
User in New York ──►
                      │
                      ├──► AP region (Tokyo)     ← Tokyo user routed here
                      ├──► EU region (Frankfurt) ← London user routed here
                      └──► US region (Virginia)  ← New York user routed here
```

---

## Key Takeaways

```
Load Balancing: Distribute traffic across servers for
                scalability, availability, fault tolerance

Algorithms:
  Round Robin          → equal distribution, simple
  Weighted Round Robin → heterogeneous servers
  Least Connections    → variable request duration
  IP Hash              → sticky sessions (avoid if possible)
  Least Response Time  → dynamic performance-aware routing

Layer 4 LB: Routes by IP/port → fast, dumb, any protocol → AWS NLB
Layer 7 LB: Routes by HTTP content → smart, flexible, HTTP only → AWS ALB

Health checks: Continuous monitoring, auto-remove unhealthy servers
               /actuator/health in Spring Boot

Global LB: Route 53 latency routing → nearest region
Service mesh: Envoy sidecar → service-to-service

Always put a load balancer in front of your services.
Never expose backend servers directly to the internet.
```
