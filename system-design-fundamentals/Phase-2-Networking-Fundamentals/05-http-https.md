# HTTP / HTTPS

**Phase 2 — Networking Fundamentals | Topic 5 of 8**

---

## What is HTTP?

HTTP (HyperText Transfer Protocol) is the foundation of data communication on the web. It's a **request-response protocol** — a client sends a request, a server sends a response.

```
Client                          Server
  │                               │
  │── GET /api/users/123 ────────►│
  │                               │ (processes request)
  │◄── 200 OK { "name": "Hari" } ─│
  │                               │
```

HTTP is **stateless** — each request is independent. The server has no memory of previous requests. This is what makes HTTP services easy to scale horizontally — any server can handle any request.

---

## HTTP Methods

| Method | Idempotent | Safe | Purpose |
|--------|-----------|------|---------|
| GET | ✅ | ✅ | Retrieve a resource. No body. |
| POST | ❌ | ❌ | Create a resource. Has body. |
| PUT | ✅ | ❌ | Replace entire resource. |
| PATCH | ❌ | ❌ | Partial update. |
| DELETE | ✅ | ❌ | Remove resource. |
| HEAD | ✅ | ✅ | Like GET but returns only headers, no body. Used for checking if resource exists. |
| OPTIONS | ✅ | ✅ | Returns supported methods. Used by CORS preflight requests. |

**Idempotent** means calling it N times has the same result as calling it once.

---

## HTTP Status Codes

```
1xx → Informational
  100 Continue

2xx → Success
  200 OK              → standard success
  201 Created         → resource created (POST response)
  204 No Content      → success, no body (DELETE response)

3xx → Redirection
  301 Moved Permanently → bookmark the new URL
  302 Found             → temporary redirect
  304 Not Modified      → cached version is still valid

4xx → Client Error (client did something wrong)
  400 Bad Request       → malformed request, validation failed
  401 Unauthorized      → not authenticated (no/invalid token)
  403 Forbidden         → authenticated but not authorized
  404 Not Found         → resource doesn't exist
  409 Conflict          → state conflict (duplicate, optimistic lock)
  422 Unprocessable     → valid syntax but semantic errors
  429 Too Many Requests → rate limited

5xx → Server Error (server did something wrong)
  500 Internal Server Error → generic server error
  502 Bad Gateway           → upstream server returned invalid response
  503 Service Unavailable   → server overloaded or down
  504 Gateway Timeout       → upstream server didn't respond in time
```

**Key distinctions:**
- `401` = "I don't know who you are" → send credentials
- `403` = "I know who you are, but you can't do this" → don't bother retrying
- `502` = your Nginx got a bad response from Spring Boot
- `504` = your Nginx waited too long for Spring Boot to respond

---

## HTTP Versions

### HTTP/1.0
One TCP connection per request. Connection closes after each response.

**Problem:** TCP handshake overhead for every single request.

### HTTP/1.1
Persistent connections (keep-alive) and pipelining. Multiple requests over one TCP connection.

**Problem:** Head-of-line blocking — if request 1 is slow, requests 2 and 3 wait.

Still the most common version for most APIs today.

### HTTP/2
Multiplexing — multiple requests in parallel over a **single TCP connection**. No head-of-line blocking at HTTP level.

```
Single TCP Connection:
  Stream 1: GET /index.html ─────────────────► response
  Stream 2: GET /style.css  ────────────────────────────► response
  Stream 3: GET /script.js  ──────────────────────────────────► response

All three travel simultaneously, responses interleaved
```

Also adds: header compression (HPACK), server push, binary framing.

> **gRPC is built on HTTP/2** — this is why gRPC gets multiplexing and streaming for free.

### HTTP/3
Replaces TCP with **QUIC (built on UDP)**. Solves TCP-level head-of-line blocking.

```
HTTP/2 problem: TCP packet loss stalls ALL streams
HTTP/3 solution: QUIC manages streams independently
                 Packet loss in stream 1 doesn't affect stream 2
                 Better performance on unreliable networks (mobile)
```

---

## HTTPS — HTTP + TLS

HTTPS is HTTP with **TLS (Transport Layer Security)** encryption layered on top. Everything in the HTTP request and response is encrypted — headers, body, URL path, everything.

```
HTTP:  Data travels in plain text
       Anyone on the network can read it

HTTPS: Data encrypted end-to-end
       Intercepted traffic is unreadable
       Server identity verified via certificates
```

### TLS Handshake (simplified)

```
Client                              Server
  │── ClientHello ───────────────►│  (TLS version, cipher suites)
  │◄── ServerHello ───────────────│  (chosen cipher, certificate)
  │  [Client validates certificate against trusted CA]
  │── Key Exchange ──────────────►│  (encrypted with server's public key)
  │  [Both derive same session key]
  │◄──── Encrypted Data ─────────►│  (all HTTP traffic encrypted)
```

TLS 1.3 reduces this to 1 round trip vs TLS 1.2's 2 round trips.

---

## HTTP in System Design

### SSL Termination at Reverse Proxy

```
Client ──HTTPS──► [ALB / Nginx]  ──HTTP──► [Spring Boot Services]
                  TLS handled here          plain HTTP internally
                  certificate managed here  no cert needed on each service
```

Backend services communicate over plain HTTP within the private VPC — TLS overhead avoided on internal traffic.

### Health Check Endpoints

```java
// Spring Boot Actuator — used by load balancer health checks
GET /actuator/health → 200 OK {"status": "UP"}

// Load balancer configuration
health_check_path: /actuator/health
healthy_threshold: 2    # 2 consecutive 200s = healthy
unhealthy_threshold: 3  # 3 consecutive failures = unhealthy
interval: 30s
```

### Inter-Service Security Options

```
Option 1: Mutual TLS (mTLS)
          Both client and server present certificates
          Each service verifies the other's identity

Option 2: Internal HTTP + JWT
          Plain HTTP within VPC (trusted network)
          JWT token verifies caller identity

Option 3: Service Mesh (Istio)
          Automatically handles mTLS between all services
          No code changes needed
```

---

## Key Takeaways

```
HTTP:    Stateless request-response protocol
         Stateless = easy to scale horizontally

Methods: GET (read), POST (create), PUT (replace),
         PATCH (update), DELETE (remove)

Status:  2xx success, 3xx redirect, 4xx client error, 5xx server error
         401 = not authenticated, 403 = not authorized
         502 = bad gateway response, 504 = gateway timeout

Versions:
  HTTP/1.1 → persistent connections, head-of-line blocking
  HTTP/2   → multiplexing, header compression, used by gRPC
  HTTP/3   → QUIC over UDP, no TCP head-of-line blocking

HTTPS:   HTTP + TLS encryption
         SSL terminate at load balancer
         Plain HTTP within private network

TLS 1.3: 1 round trip handshake, most secure and fast
```
