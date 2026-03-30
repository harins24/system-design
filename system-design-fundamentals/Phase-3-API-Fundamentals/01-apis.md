# APIs

**Phase 3 — API Fundamentals | Topic 1 of 8**

---

## What is an API?

API stands for Application Programming Interface. It's a **contract** that defines how two software components communicate — what requests can be made, how to make them, and what responses to expect.

```
Without API (direct access):
Service A reaches directly into Service B's database
→ Tight coupling, any DB change breaks Service A
→ No security boundary, no versioning, chaos

With API:
Service A calls Service B's well-defined interface
→ Service B can change internals freely
→ Contract stays stable, both sides evolve independently
```

An API is the boundary between systems. Everything inside that boundary is implementation detail — hidden, changeable. The API surface is the public contract.

---

## Types of APIs

### 1. REST (Representational State Transfer)
The dominant style for web APIs. Uses HTTP methods and URLs to represent operations on resources.

```
GET    /users/123          → fetch user 123
POST   /users              → create new user
PUT    /users/123          → replace user 123
PATCH  /users/123          → partially update user 123
DELETE /users/123          → delete user 123
```

### 2. GraphQL
Query language for APIs. Client specifies exactly what data it needs.

```
query {
  user(id: "123") {
    name
    email
  }
}
→ returns ONLY name and email (no over-fetching)
```

### 3. gRPC
Google's Remote Procedure Call framework. Uses Protocol Buffers (binary), runs over HTTP/2.

```protobuf
service UserService {
  rpc GetUser (UserRequest) returns (UserResponse);
  rpc StreamUsers (UserFilter) returns (stream UserResponse);
}
```

Feels like calling a local function. ~5-10x faster than REST for service-to-service calls.

### 4. WebSockets
Persistent bidirectional connection. Server can push to client without client requesting.

### 5. Message Queue APIs (Kafka, RabbitMQ)
Asynchronous communication through a broker.

---

## The Anatomy of a Good API

### Clear Resource Naming
```
Good:
GET /users/{id}
GET /orders/{id}/items
POST /products

Bad:
GET /getUser?userId=123
POST /createNewOrder
GET /fetchAllProductsList
```

APIs should be **nouns, not verbs**. The HTTP method IS the verb.

### Consistent Response Structure
```json
// Success
{
  "data": { "id": "123", "name": "Hari" },
  "meta": { "requestId": "req_abc123", "timestamp": "2026-02-14T10:00:00Z" }
}

// Error
{
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User with id 123 does not exist",
    "requestId": "req_abc123"
  }
}
```

### Versioning
```
URL versioning (most common):
/api/v1/users
/api/v2/users   ← new version, v1 still works
```

**Never break a published API.** Add new fields (non-breaking), deprecate old ones with notice, version for breaking changes.

### Pagination
```
Cursor pagination (better):
GET /orders?cursor=eyJpZCI6MTAwfQ&limit=20
→ returns 20 records after the cursor position
→ Stable regardless of inserts/deletes
→ Efficient — no scanning skipped records
```

---

## API Design Principles

- **Idempotency** — GET, PUT, DELETE are naturally idempotent; make POST idempotent with `Idempotency-Key` header
- **Rate limiting** — protect from abuse; return `429 Too Many Requests` with `Retry-After` header
- **Clear errors** — machine-readable codes + human-readable messages + request ID

---

## API Gateway Pattern

In microservices, an API Gateway is a single entry point for all clients:

```
Clients          API Gateway              Services
                 ┌──────────────┐
Mobile App ────► │ Auth         │────► User Service
Web App    ────► │ Rate Limit   │────► Order Service
3rd Party  ────► │ SSL Term     │────► Payment Service
                 │ Routing      │────► Notification Service
                 │ Logging      │
                 └──────────────┘
```

Cross-cutting concerns handled once at the gateway — not duplicated in every service.

---

## Key Takeaways

```
API: Contract defining how two components communicate
     Implementation hidden, interface stable

Types:
  REST        → HTTP + resources, dominant for web APIs
  GraphQL     → client-specified queries, flexible
  gRPC        → binary, HTTP/2, high performance inter-service
  WebSocket   → persistent bidirectional, real-time
  Message Queue → async, decoupled, Kafka/RabbitMQ

Good API properties:
  Noun-based URLs    → resources, not actions
  Consistent format  → same structure for all responses
  Versioned          → /v1/, /v2/ — never break clients
  Paginated          → cursor pagination > offset
  Idempotent         → safe to retry
  Rate limited       → protect from abuse

Interview tip: Define APIs FIRST before designing internals
```
