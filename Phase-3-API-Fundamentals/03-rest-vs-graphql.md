# REST vs GraphQL

**Phase 3 — API Fundamentals | Topic 3 of 8**

---

## REST — The Foundation

REST (Representational State Transfer) is an architectural style for designing networked APIs. It's not a protocol — it's a set of constraints that, when followed, produce scalable, maintainable APIs.

### The 6 REST Constraints

```
1. Client-Server
   Client and server are separate, evolve independently
   Client doesn't care how server stores data
   Server doesn't care how client displays data

2. Stateless
   Each request contains all information needed
   Server stores no client session state
   → Enables horizontal scaling (any server handles any request)

3. Cacheable
   Responses explicitly marked as cacheable or not
   GET responses cached by browsers, CDNs, proxies
   → Reduces load, improves performance

4. Uniform Interface
   Consistent way to interact with resources
   → Nouns for URLs, HTTP methods as verbs
   → Standard response formats

5. Layered System
   Client doesn't know if it's talking to origin server,
   load balancer, CDN, or cache
   → Enables infrastructure transparency

6. Code on Demand (optional)
   Server can send executable code to client
   → JavaScript in browsers
```

Most people call any HTTP JSON API "REST." True REST follows all these constraints. For interviews, constraints 1-4 matter most.

---

## REST API Design — The Right Way

### Resource Modeling

Everything is a resource, identified by a URL:

```
/users              → collection of users
/users/123          → specific user
/users/123/orders   → orders belonging to user 123
/orders/456/items   → items in order 456

Rules:
✅ Use nouns:   /users, /orders, /products
❌ Not verbs:   /getUsers, /createOrder, /deleteProduct

✅ Plural:      /users, /orders
❌ Not singular: /user, /order

✅ Lowercase:   /product-categories
❌ Not camel:   /productCategories
```

### HTTP Methods Mapped to Operations

```
Resource: /users

GET    /users         → list all users (paginated)
POST   /users         → create new user
GET    /users/123     → get user 123
PUT    /users/123     → replace user 123 entirely
PATCH  /users/123     → partially update user 123
DELETE /users/123     → delete user 123

Nested resource:
GET    /users/123/orders     → all orders for user 123
POST   /users/123/orders     → create order for user 123
GET    /users/123/orders/456 → specific order for user 123
```

### REST Response Design

```json
// GET /users/123 → 200 OK
{
  "id": "123",
  "name": "Hari Kumar",
  "email": "hari@example.com",
  "role": "senior_engineer",
  "createdAt": "2024-01-15T10:00:00Z",
  "_links": {
    "self": "/users/123",
    "orders": "/users/123/orders"
  }
}

// POST /users → 201 Created
// Location header: /users/124
{
  "id": "124",
  "name": "New User"
}

// DELETE /users/123 → 204 No Content
// (empty body)

// Validation error → 400 Bad Request
{
  "error": {
    "code": "VALIDATION_ERROR",
    "details": [
      {"field": "email", "issue": "Invalid format"}
    ]
  }
}
```

---

## The Problems REST Solves — And Creates

### What REST Does Well
```
✅ Simple mental model — URLs = resources, methods = actions
✅ HTTP native — caching, status codes, headers all built in
✅ Stateless — horizontally scalable by design
✅ Widely understood — every developer knows REST
✅ Great tooling — Swagger, Postman, curl, browser
✅ CDN cacheable — GET responses cached at edge
```

### REST's Core Problems

**Over-fetching:**
```
GET /users/123
→ returns: id, name, email, phone, address, bio,
           preferences, createdAt, updatedAt, role,
           permissions, lastLoginAt, profilePicUrl...

You only needed: name and email
→ Wasted bandwidth, wasted parsing, wasted mobile data
```

**Under-fetching (N+1 problem):**
```
You want to display a feed: user info + their last 3 orders + product names

REST requires:
  GET /users/123                    → 1 request
  GET /users/123/orders?limit=3     → 1 request
  GET /products/A                   → 1 request
  GET /products/B                   → 1 request
  GET /products/C                   → 1 request
                                    = 5 requests

This is the N+1 problem.
With N orders, you need N+1 requests.
On mobile, each request = latency + battery drain.
```

**Multiple endpoints per client type:**
```
Mobile app needs: minimal data (small screen, slow network)
Web app needs:    rich data (full details, fast network)
Dashboard needs:  aggregated data (stats, counts)

REST solution: Build different endpoints for each
               /mobile/users/123
               /web/users/123
               /dashboard/stats/users
               → API sprawl, maintenance nightmare
```

---

## GraphQL — The Solution

GraphQL is a **query language for APIs** developed by Facebook in 2012, open-sourced in 2015. Instead of fixed endpoints returning fixed data shapes, clients describe exactly what they need.

```
One endpoint:   POST /graphql
Client sends:   a query describing exactly what it wants
Server returns: exactly that — no more, no less
```

### GraphQL Query Syntax

```graphql
# Client sends this query
query {
  user(id: "123") {
    name
    email
    orders(last: 3) {
      id
      total
      items {
        productName
        quantity
      }
    }
  }
}

# Server returns exactly this shape
{
  "data": {
    "user": {
      "name": "Hari Kumar",
      "email": "hari@example.com",
      "orders": [
        {
          "id": "456",
          "total": 99.99,
          "items": [
            {"productName": "Laptop Stand", "quantity": 1}
          ]
        }
      ]
    }
  }
}
```

The client got user + last 3 orders + product names in one request. No over-fetching, no N+1.

### GraphQL Core Concepts

**Schema — The Contract:**
```graphql
# Server defines all available types and operations
type User {
  id: ID!
  name: String!
  email: String!
  orders(last: Int): [Order!]!
  createdAt: String!
}

type Order {
  id: ID!
  total: Float!
  status: OrderStatus!
  items: [OrderItem!]!
}

enum OrderStatus {
  PENDING
  PROCESSING
  SHIPPED
  DELIVERED
}

# Query = read operations
type Query {
  user(id: ID!): User
  users(page: Int, limit: Int): [User!]!
  order(id: ID!): Order
}

# Mutation = write operations
type Mutation {
  createUser(name: String!, email: String!): User!
  updateUser(id: ID!, name: String): User!
  createOrder(userId: ID!, items: [OrderItemInput!]!): Order!
}

# Subscription = real-time
type Subscription {
  orderStatusChanged(orderId: ID!): Order!
}
```

The schema is self-documenting — clients can introspect it to discover all available queries and types.

**Subscriptions — Real-Time:**
```graphql
subscription {
  orderStatusChanged(orderId: "456") {
    id
    status
    updatedAt
  }
}
# Server pushes updates whenever order status changes
# Client receives updates over WebSocket connection
```

---

## GraphQL Problems

**N+1 Problem (different version):**
```
Query: users { orders { product { name } } }

Naive resolver:
  Fetch 10 users                    → 1 DB query
  For each user, fetch their orders → 10 DB queries
  For each order, fetch product     → 50 DB queries
  Total: 61 queries for one GraphQL request

Solution: DataLoader pattern
  Batch all product IDs → one query: WHERE id IN (...)
  Facebook's DataLoader library handles this
```

**Caching is harder:**
```
REST:   GET /users/123 → HTTP cache, CDN cache, browser cache
        URL is the cache key — simple

GraphQL: POST /graphql with query in body
         POST requests not cached by default
         Query body varies — what's the cache key?

Solutions:
  Persisted queries (store query by hash, send hash)
  Apollo Client cache (client-side, by query+variables)
  CDN edge caching with persisted queries
```

**Query complexity attacks:**
```
Malicious client sends deeply nested query:
{
  users {
    orders {
      user {
        orders {
          user {
            orders { ... deeply nested ... }
          }
        }
      }
    }
  }
}

Could trigger massive DB load with one request

Solutions:
  Query depth limiting (max 5 levels deep)
  Query complexity scoring (each field costs points, max budget)
  Query timeout
  Rate limiting on complexity score
```

**Introspection in production:**
```
GraphQL allows clients to query the schema itself
→ Convenient for developers
→ Security risk: attackers discover all types and fields

Solution: Disable introspection in production
          Allow only for authenticated developers
```

---

## REST vs GraphQL — Direct Comparison

| Feature | REST | GraphQL |
|---------|------|---------|
| Data fetching | Fixed shape per endpoint | Client-specified |
| Over-fetching | Common | Eliminated |
| Under-fetching | N+1 problem | Solved (one query) |
| Caching | HTTP native, easy | Complex, manual |
| Learning curve | Low (everyone knows it) | Higher |
| Tooling | Excellent | Good (Apollo, etc.) |
| Error handling | HTTP status codes | Always 200, errors in body |
| File upload | Simple multipart | Awkward |
| Real-time | Polling / SSE | Subscriptions (WebSocket) |
| Schema | OpenAPI (separate) | Built-in, introspectable |
| Best for | Simple CRUD, public APIs | Complex, multiple clients |

---

## When to Use Which

**Use REST when:**
```
→ Public API consumed by third parties you don't control
  (REST is universally understood)
→ Simple CRUD operations, straightforward data shapes
→ Heavy caching needed (CDN, browser cache)
→ File upload/download
→ Team unfamiliar with GraphQL
→ Microservice-to-microservice communication
  (gRPC is actually better here)
```

**Use GraphQL when:**
```
→ Multiple client types with different data needs
  (mobile, web, TV app all query same backend)
→ Rapidly changing frontend requirements
  (add new field in query, no backend change needed)
→ Aggregate data from multiple services in one request
→ Heavy read operations with complex data relationships
→ Development speed more important than HTTP caching
```

**Real world:**
```
Facebook → built GraphQL for mobile (slow networks, need minimal data)
GitHub   → offers both REST v3 and GraphQL v4
Shopify  → GraphQL for their storefront API
Twitter  → REST (public API, third party clients)
Netflix  → Federated GraphQL (multiple services, one schema)
```

---

## GraphQL Federation — Microservices + GraphQL

The killer feature for microservices: each service owns part of the schema, stitched into one unified graph.

```
User Service owns:
  type User { id, name, email }

Order Service owns:
  type Order { id, total, user: User }
  extend type User { orders: [Order] }

Product Service owns:
  type Product { id, name, price }
  extend type Order { items: [Product] }

Federation Gateway stitches them together:
  Client queries one endpoint
  Gateway routes sub-queries to correct services
  Assembles complete response
  Client sees one unified schema
```

Apollo Federation is the dominant implementation. This is how large companies like Netflix run GraphQL at scale.

### Spring Boot + GraphQL

```java
// Spring Boot + GraphQL (spring-boot-starter-graphql)
@Controller
public class UserController {

    @QueryMapping
    public User user(@Argument String id) {
        return userService.findById(id);
    }

    @MutationMapping
    public User createUser(@Argument String name,
                           @Argument String email) {
        return userService.create(name, email);
    }

    @SubscriptionMapping
    public Flux<Order> orderStatusChanged(@Argument String orderId) {
        return orderService.subscribeToStatus(orderId);
    }

    // DataLoader for N+1 prevention
    @BatchMapping
    public Map<User, List<Order>> orders(List<User> users) {
        return orderService.findByUsers(users); // one DB query
    }
}
```

---

## Key Takeaways

```
REST:
  Resources + HTTP methods + status codes
  Stateless, cacheable, universally understood
  Problems: over-fetching, under-fetching, N+1

GraphQL:
  Client specifies exactly what data it needs
  One endpoint, one request for complex data
  Solves: over-fetching, under-fetching, multiple clients
  Problems: caching harder, N+1 at resolver level, complexity attacks

Choose REST when:
  Public API, simple CRUD, CDN caching, file operations

Choose GraphQL when:
  Multiple client types, complex queries, rapid frontend iteration

Both in practice:
  Many companies use REST for external APIs
  GraphQL for internal/BFF layer
  gRPC for service-to-service

Federation: Multiple GraphQL services → one unified schema
DataLoader: Batch DB queries → solve resolver N+1 problem
```
