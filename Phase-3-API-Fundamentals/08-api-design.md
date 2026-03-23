# API Design

**Phase 3 — API Fundamentals | Topic 8 of 8**

---

## What is API Design?

API design is the practice of deliberately crafting the interface between systems so that it is intuitive, consistent, maintainable, and evolvable. Good API design is the difference between an interface developers love using and one they dread.

```
Bad API:
POST /api/doUserLoginAndReturnTokenWithPermissions
GET  /api/getUserDataByIdOrEmailOptionallyWithOrders?userId=123&includeOrders=true&v=2

Good API:
POST /auth/sessions          → login
GET  /users/123              → get user
GET  /users/123?include=orders → get user with orders
```

The best APIs feel obvious in retrospect. Users rarely need to read the docs.

---

## The 10 Principles of Good API Design

### 1. Resource-Oriented Design

Model your API around resources (nouns), not actions (verbs). HTTP methods express the action.

```
Bad (RPC style):
POST /api/createUser
POST /api/getUser
POST /api/updateUserEmail
POST /api/deleteUser

Good (Resource style):
POST   /users          → create
GET    /users/{id}     → read
PUT    /users/{id}     → update
DELETE /users/{id}     → delete

The resource IS the noun. HTTP method IS the verb.
Never put actions in the URL when an HTTP method exists for it.
```

When no HTTP method fits (e.g., "send invoice", "publish post", "archive order"):
```
Use a sub-resource or action endpoint:
POST /invoices/{id}/send       ← action on a resource
POST /posts/{id}/publish       ← state transition
POST /orders/{id}/archive      ← state change

These are acceptable when HTTP methods don't express the semantics
```

---

### 2. Consistent Naming Conventions

```
URLs:
✅ Plural nouns:    /users, /orders, /products
✅ Lowercase:       /product-categories (kebab-case)
✅ Hierarchical:    /users/{id}/orders/{orderId}
❌ Mixed case:      /productCategories
❌ Verbs:           /getUsers, /createOrder
❌ Underscores:     /product_categories

Response fields:
✅ camelCase JSON:  {"userId": "123", "createdAt": "..."}
✅ Consistent dates: ISO 8601 always → "2026-02-14T10:00:00Z"
✅ Consistent IDs:  always strings, never mix string/int

Booleans:
✅ "isActive": true   (not "active": 1 or "status": "Y")

Enums:
✅ SCREAMING_SNAKE: "status": "IN_PROGRESS"
   (unambiguous, consistent across languages)
```

---

### 3. API Versioning

APIs evolve. Existing clients break when you change them. Versioning lets both old and new clients coexist.

```
URL versioning (most common, most visible):
/api/v1/users   ← old clients
/api/v2/users   ← new clients

Pros: Explicit, easy to route in gateway, visible in logs
Cons: URL "pollution", clients must change URLs

Header versioning:
GET /api/users
Accept: application/vnd.yourapi.v2+json

Pros: Clean URLs
Cons: Less visible, harder to test in browser

Query param versioning:
GET /api/users?version=2

Pros: Easy to override
Cons: Easy to forget, pollutes query params

Recommendation: URL versioning for external APIs
                Header versioning for internal/partner APIs
```

**Breaking vs non-breaking changes:**
```
Non-breaking (safe to release without version bump):
✅ Adding new optional fields to response
✅ Adding new optional request parameters
✅ Adding new endpoints
✅ Adding new enum values (risky if client validates strictly)
✅ Making required field optional

Breaking (MUST version):
❌ Removing fields from response
❌ Renaming fields
❌ Changing field types (string → int)
❌ Making optional field required
❌ Changing URL structure
❌ Changing authentication scheme
❌ Changing error format
```

---

### 4. Meaningful HTTP Status Codes

Design your API to return the RIGHT status code:

```
POST /users         → 201 Created (not 200)
DELETE /users/123   → 204 No Content (not 200)
GET /users/999      → 404 Not Found (not 200 with null body)
POST /users (duplicate email) → 409 Conflict (not 400)
GET /admin (not admin role)   → 403 Forbidden (not 401)
POST /payments (card declined) → 402 Payment Required (not 500)

Status codes are part of your API contract.
Don't always return 200. Don't use 500 for business errors.
```

---

### 5. Pagination, Filtering, Sorting

Never return unbounded collections:

```
GET /orders → returns 10 million orders? ← catastrophic

Pagination (cursor-based — the right way):
GET /orders?limit=20&cursor=eyJpZCI6MTAwfQ

Response:
{
  "data": [...20 orders...],
  "pagination": {
    "limit": 20,
    "nextCursor": "eyJpZCI6MTIwfQ",
    "hasMore": true
  }
}

Filtering:
GET /orders?status=PENDING&userId=123
GET /products?minPrice=10&maxPrice=100&category=electronics
GET /users?createdAfter=2026-01-01&role=admin

Sorting:
GET /orders?sort=createdAt&order=desc
GET /products?sort=price,name&order=asc,asc

Field selection (partial response):
GET /users/123?fields=id,name,email
→ returns only requested fields
→ reduces bandwidth for mobile clients
```

---

### 6. Consistent Error Response Format

```json
// Every error, every endpoint, same structure:
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "code": "INVALID_FORMAT",
        "message": "Must be a valid email address"
      },
      {
        "field": "age",
        "code": "OUT_OF_RANGE",
        "message": "Must be between 18 and 120"
      }
    ],
    "requestId": "req_a3f8c2d1",
    "timestamp": "2026-02-14T10:00:00Z",
    "documentation": "https://docs.yourapi.com/errors#VALIDATION_ERROR"
  }
}
```

```
Error codes should be:
  Machine-readable: PAYMENT_FAILED, USER_NOT_FOUND, RATE_LIMIT_EXCEEDED
  Stable: don't rename them (clients check these)
  Documented: link to docs explaining cause and resolution

Never return raw exception messages:
  ❌ "NullPointerException at PaymentService.java:147"
  ✅ "Payment processing failed. Please try again."
```

---

### 7. Request/Response Design

**Request design:**
```
Use request body for complex data (POST, PUT, PATCH):
POST /orders
{
  "userId": "123",
  "items": [
    {"productId": "A", "quantity": 2},
    {"productId": "B", "quantity": 1}
  ],
  "shippingAddressId": "addr_456"
}

Use path params for resource identity:
GET /users/{id}
DELETE /orders/{id}

Use query params for filtering, pagination, optional behavior:
GET /products?category=electronics&page=2&limit=20

Never put sensitive data in URLs:
❌ GET /users?password=secret123    (appears in logs)
✅ POST /auth/sessions with body {"password": "secret123"}
```

**Response design:**
```json
// Envelope pattern — consistent wrapper
{
  "data": {
    "id": "123",
    "name": "Hari"
  },
  "meta": {
    "requestId": "req_abc",
    "timestamp": "2026-02-14T10:00:00Z",
    "version": "v2"
  }
}

// Or flat (simpler, used by Stripe):
{
  "id": "123",
  "name": "Hari",
  "object": "user"
}
```

Pick one and be consistent everywhere.

---

### 8. Idempotency and Safety

```
Every state-changing POST endpoint MUST support:
  Idempotency-Key header for at-least-once safety
  Or use natural idempotency (PUT, DELETE)

Design endpoints to be as safe as possible:
  GET endpoints: never modify state, always safe to retry
  Webhooks: document which events carry idempotency keys
  Batch operations: atomic or document partial failure handling
```

---

### 9. API Security Design

```
Authentication:
  Bearer token in Authorization header:
  Authorization: Bearer eyJhbGc...

  Never in URL: GET /users?token=xxx (logged, cached, visible)
  Never in body for GET: semantically wrong

Authorization:
  Return 401 if not authenticated (no token)
  Return 403 if authenticated but not authorized
  Don't leak existence: return 404 instead of 403 for
  resources the user shouldn't know exist
  ("User 123's secret document" → 404, not 403)

Input validation:
  Validate everything at the API boundary
  Reject unknown fields (or ignore them safely)
  Sanitize before passing to DB (prevent injection)
  Max sizes for strings, arrays, nested depth

Sensitive data:
  Never return passwords, even hashed
  Mask card numbers: "****4242"
  Truncate tokens in responses: show first 8 chars
  Audit log access to sensitive endpoints
```

---

### 10. Documentation and Discoverability

```
OpenAPI (Swagger) spec:
  Machine-readable description of every endpoint
  Auto-generates: interactive docs, client SDKs
```

```java
// Spring Boot:
@Operation(summary = "Create a new order")
@ApiResponses({
  @ApiResponse(responseCode = "201", description = "Order created"),
  @ApiResponse(responseCode = "400", description = "Validation error"),
  @ApiResponse(responseCode = "402", description = "Payment failed"),
  @ApiResponse(responseCode = "429", description = "Rate limit exceeded")
})
@PostMapping("/orders")
public ResponseEntity<Order> createOrder(
    @Valid @RequestBody CreateOrderRequest request,
    @RequestHeader("Idempotency-Key") String idempotencyKey) {
    ...
}
```

**HATEOAS (Hypermedia):**
```json
{
  "id": "123",
  "status": "PENDING",
  "_links": {
    "self":   {"href": "/orders/123"},
    "cancel": {"href": "/orders/123/cancel"},
    "pay":    {"href": "/orders/123/pay"},
    "items":  {"href": "/orders/123/items"}
  }
}
```

Client discovers available actions from response. Less coupling between client and API structure.

---

## API Design Checklist

Before shipping any API, verify:

```
Resources:
□ URLs are nouns, not verbs
□ Plural resource names
□ Hierarchical where appropriate
□ Actions as sub-resources when needed

HTTP:
□ Correct status codes (201 for create, 204 for delete)
□ Correct methods (GET never modifies state)
□ Idempotent where possible

Responses:
□ Consistent envelope format
□ Consistent date format (ISO 8601)
□ Consistent ID format (always string)
□ Paginated collections (cursor-based)
□ Field selection supported

Errors:
□ Machine-readable error codes
□ Human-readable messages
□ Request ID in every error
□ No stack traces in production responses

Security:
□ Auth via Authorization header
□ Input validated and sanitized
□ Sensitive data masked
□ Rate limiting headers returned

Evolution:
□ Versioned
□ Breaking changes documented
□ Deprecation notices in headers
□ Changelog maintained
```

---

## Putting It All Together — Order API Design

```
Resource model:
  /users                    users
  /users/{id}/orders        orders for a user
  /orders/{id}              specific order
  /orders/{id}/items        items in an order
  /orders/{id}/payments     payments for an order

Endpoints:
  POST   /users/{id}/orders           create order
  GET    /users/{id}/orders           list user's orders (paginated)
  GET    /orders/{id}                 get order details
  PATCH  /orders/{id}                 update order
  POST   /orders/{id}/cancel          cancel order (action)
  POST   /orders/{id}/pay             pay for order (action)
  GET    /orders/{id}/items           list order items

Request (create order):
  POST /users/123/orders
  Authorization: Bearer token
  Idempotency-Key: uuid-v4
  {
    "items": [{"productId": "A", "quantity": 2}],
    "shippingAddressId": "addr_456",
    "couponCode": "SAVE10"
  }

Response (201 Created):
  Location: /orders/456
  {
    "data": {
      "id": "456",
      "status": "PENDING",
      "total": 89.99,
      "currency": "USD",
      "createdAt": "2026-02-14T10:00:00Z",
      "_links": {
        "self":   "/orders/456",
        "pay":    "/orders/456/pay",
        "cancel": "/orders/456/cancel",
        "items":  "/orders/456/items"
      }
    }
  }
```

---

## Key Takeaways

```
API Design Principles:
  1. Resource-oriented    → nouns in URLs, methods as verbs
  2. Consistent naming    → plural, lowercase, camelCase fields
  3. Versioned            → /v1/, /v2/ for breaking changes
  4. Correct status codes → 201 create, 204 delete, 409 conflict
  5. Pagination           → cursor-based, never unbounded
  6. Consistent errors    → machine code + human message + requestId
  7. Clean request design → body for data, path for ID, query for filter
  8. Idempotent           → Idempotency-Key for POST
  9. Secure               → auth in header, validate everything
  10. Documented          → OpenAPI, HATEOAS links

Breaking changes require versioning
Non-breaking changes are safe to add anytime

Checklist before shipping:
  Resource URLs ✓ HTTP methods ✓ Status codes ✓
  Pagination ✓ Error format ✓ Security ✓ Versioning ✓
```
