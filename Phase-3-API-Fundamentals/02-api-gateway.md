# API Gateway

**Phase 3 — API Fundamentals | Topic 2 of 8**

---

## What is an API Gateway?

An API Gateway is a server that acts as the **single entry point** for all client requests into your microservices architecture.

```
Without API Gateway:
Mobile App ──────────────────────────► User Service :8081
Web App    ──────────────────────────► Order Service :8082
Partner    ──────────────────────────► Payment Service :8083
                                       Inventory Service :8084

Problems:
  Each client knows about every service
  Auth implemented in every service
  Rate limiting duplicated everywhere
  Services exposed directly to internet

With API Gateway:
Mobile App ──┐
Web App    ──┼──► [API Gateway] ──► User Service
Partner    ──┘         │        ──► Order Service
                       │        ──► Payment Service
                       │        ──► Inventory Service

One entry point. Everything else is internal.
```

---

## What an API Gateway Does

### 1. Request Routing
```
/api/users/*     → User Service
/api/orders/*    → Order Service
/api/payments/*  → Payment Service
/                → Frontend Service
```

### 2. Authentication and Authorization
```
Request arrives with JWT token
        ↓
Gateway validates token (signature, expiry, claims)
        ↓
Invalid → 401 Unauthorized (never reaches backend)
Valid   → gateway extracts user info, adds to headers:
  X-User-Id: 123
  X-User-Role: admin
  X-User-Email: hari@example.com
(no need to validate token again in each service)
```

### 3. Rate Limiting
```
Free tier:    100 requests/hour
Pro tier:     10,000 requests/hour
Enterprise:   unlimited

Per endpoint:
POST /payments  → 10 requests/minute (sensitive)
GET /products   → 1000 requests/minute (read-heavy)
```

### 4. SSL Termination
```
Client ──HTTPS──► [Gateway] ──HTTP──► Backend Services
                  decrypts             plain text internally
```

### 5. Request/Response Transformation
- Add headers: `X-Request-Id`, `X-Timestamp`
- Remove sensitive headers before forwarding
- Aggregate responses from multiple services

### 6. Caching
```
GET /api/products/catalog
→ Cache for 5 minutes
→ Subsequent requests served from cache
→ Backend not hit for 5 minutes
```

### 7. Observability
Single point to log every request/response across the entire system — timestamp, client IP, user ID, endpoint, response code, latency.

### 8. Circuit Breaking
```
Order Service returns 500s → circuit opens
Gateway returns cached response or graceful error
Order Service recovers → circuit closes
```

---

## API Gateway vs Load Balancer vs Reverse Proxy

| | Reverse Proxy | Load Balancer | API Gateway |
|--|--------------|---------------|-------------|
| Routing | Basic forwarding | Traffic distribution | Content-based routing |
| Auth | No | No | Yes |
| Rate limiting | No | No | Yes |
| Request transformation | Basic | No | Yes |
| Examples | Nginx | AWS ALB, HAProxy | Kong, AWS API GW, Spring Cloud GW |

> "A reverse proxy is infrastructure, an API gateway is a product feature."

---

## BFF — Backend for Frontend Pattern

Different clients need different data shapes. Create specialized gateways per client type:

```
Mobile App ──► [Mobile BFF]   ──► Microservices
               - Small payloads
               - Offline support

Web App    ──► [Web BFF]      ──► Microservices
               - Rich data
               - Server-side rendering

Partner    ──► [Partner BFF]  ──► Microservices
               - Stable versioned API
               - Usage metering
```

Each BFF is owned by the frontend team — they control their own data contract.

---

## API Gateway Failure Modes

**The API gateway is itself a potential SPOF.**

```
Mitigations:
1. Run multiple gateway instances behind a load balancer
2. Make the gateway stateless
   Rate limit state in Redis (shared across instances)
   Auth validated from JWT (no local session)
3. Health checks and auto-scaling
4. Graceful degradation
   If downstream service fails, return cached response
```

---

## Performance Considerations

```
Without gateway: client → service = 1ms
With gateway:    client → gateway → service = 1ms + 2ms = 3ms

JWT validation (~1ms) + Rate limit check Redis (~1ms) + Route lookup (<1ms)
Total added latency: ~2-3ms — acceptable for most systems
```

**Rule of thumb:** Gateway only for **external-facing** traffic. Internal service-to-service calls bypass the gateway.

```
External:  Client ──► API Gateway ──► Service A ──► Service B
                                      (internal, no gateway)
```

---

## Key Takeaways

```
API Gateway: Single entry point for all external traffic
             Does: routing, auth, rate limiting, SSL,
                   transformation, caching, observability

vs Reverse Proxy: infrastructure-level forwarding
vs Load Balancer: traffic distribution
API Gateway:      business-level request management

BFF Pattern: Specialized gateways per client type

Implementations:
  Kong          → open source, Nginx-based, plugin ecosystem
  AWS API GW    → managed, serverless-friendly
  Spring Cloud GW → Spring ecosystem, Java-native

Critical: Gateway must NOT be a SPOF
  → Multiple stateless instances
  → Shared state in Redis
  → LB in front of gateways

Internal traffic bypasses gateway
Only external traffic goes through it
```
