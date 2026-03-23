# SQL vs NoSQL

**Phase 4 — Database Fundamentals | Topic 2 of 9**

---

## The Fundamental Difference

```
SQL (Relational):
  Data stored in tables with rows and columns
  Rigid schema — structure defined upfront
  Relationships enforced by foreign keys
  ACID transactions built in
  Query with SQL — powerful, expressive

NoSQL (Non-Relational):
  Data stored in various formats (document, key-value, graph, column)
  Flexible schema — structure can vary per record
  Relationships handled in application code
  ACID varies by database and configuration
  Query language varies by database
```

Neither is universally better. The right choice depends entirely on your data model, access patterns, and scale requirements.

---

## SQL — Relational Databases

### How It Works

Data is organized into tables. Tables relate to each other through keys. The schema enforces structure.

```sql
-- Users table
CREATE TABLE users (
    id         UUID PRIMARY KEY,
    email      VARCHAR(255) UNIQUE NOT NULL,
    name       VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Orders table — related to users
CREATE TABLE orders (
    id         UUID PRIMARY KEY,
    user_id    UUID REFERENCES users(id),  -- foreign key
    total      DECIMAL(10,2) NOT NULL,
    status     VARCHAR(20) CHECK (status IN ('PENDING','SHIPPED','DELIVERED')),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Powerful JOIN across tables
SELECT u.name, COUNT(o.id) as order_count, SUM(o.total) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2026-01-01'
GROUP BY u.id, u.name
HAVING SUM(o.total) > 1000
ORDER BY total_spent DESC;
```

### Strengths of SQL
```
✅ ACID transactions — financial accuracy guaranteed
✅ Powerful queries — JOINs, aggregations, window functions
✅ Schema enforcement — data integrity at DB level
✅ Normalized data — no duplication, update once affects everywhere
✅ Mature ecosystem — decades of tooling, expertise, best practices
✅ Complex relationships — naturally expressed
✅ Ad-hoc queries — business intelligence, reporting
```

### Weaknesses of SQL
```
❌ Rigid schema — adding columns requires ALTER TABLE (can lock table)
❌ Horizontal scaling is hard — sharding relational data is complex
❌ Object-relational impedance mismatch — code objects ≠ table rows
❌ Not great for hierarchical/nested data (JSON is better)
❌ Performance at extreme scale requires significant tuning
```

### When to Use SQL
```
✅ Financial systems — transactions must be ACID
✅ User accounts and authentication
✅ Complex reporting and analytics
✅ Any system with complex relationships between entities
✅ When data integrity is more important than write speed
✅ When your access patterns are unknown (SQL handles anything)

Examples: PostgreSQL, MySQL, Oracle, SQL Server
```

---

## NoSQL — The Four Main Types

### 1. Document Databases

Data stored as documents (JSON/BSON). Each document is self-contained. No fixed schema.

```json
// MongoDB document — everything about an order in one document
{
  "_id": "order_456",
  "userId": "user_123",
  "status": "PENDING",
  "total": 99.99,
  "items": [
    {
      "productId": "prod_A",
      "name": "Laptop Stand",
      "price": 49.99,
      "quantity": 1
    },
    {
      "productId": "prod_B",
      "name": "USB Hub",
      "price": 29.99,
      "quantity": 2
    }
  ],
  "shippingAddress": {
    "street": "123 Main St",
    "city": "Omaha",
    "state": "NE"
  },
  "createdAt": "2026-02-14T10:00:00Z"
}
```

```
✅ Natural fit for object-oriented code (document = object)
✅ No JOIN overhead — related data in one document
✅ Flexible schema — add fields without migration
✅ Good for read-heavy workloads (one read = all data)

❌ Data duplication — product name stored in every order
   → Update product name? Must update all orders too
❌ Transactions across documents are complex
❌ Ad-hoc queries less powerful than SQL

Use for: Product catalogs, user profiles, content management,
         e-commerce orders, any entity with nested structure

Examples: MongoDB, CouchDB, Firestore
```

---

### 2. Key-Value Stores

Simplest NoSQL model. Data stored as key → value pairs. Extremely fast lookups.

```
Key:   "user:session:abc123"
Value: {"userId": "123", "role": "admin", "expiresAt": "..."}

Key:   "product:price:prod_A"
Value: "49.99"

Key:   "rate_limit:user_123:2026-02-14:10"
Value: "87"  ← request count in this hour window

Operations:
GET key         → O(1) lookup
SET key value   → O(1) write
DEL key         → O(1) delete
EXPIRE key ttl  → O(1) set expiry
```

```
✅ Extremely fast — O(1) operations
✅ Simple mental model
✅ Scales horizontally easily (keys distribute naturally)
✅ TTL support — built-in expiry

❌ No query by value — can only lookup by exact key
❌ No relationships — you manage them in application code
❌ Not suitable for complex data relationships

Use for: Session storage, caching, rate limiting,
         leaderboards, shopping carts, feature flags

Examples: Redis, DynamoDB (also supports document), Memcached
```

---

### 3. Column-Family Databases (Wide-Column)

Data stored in rows, but each row can have a different set of columns. Optimized for time-series and write-heavy workloads.

```
Think of it as a sorted map of maps:

Row key         Column family: metrics
──────────────────────────────────────────────────────────────
"sensor:001"    timestamp:1708934000 → 72.5°F
                timestamp:1708934060 → 72.8°F
                timestamp:1708934120 → 73.1°F

"sensor:002"    timestamp:1708934000 → 68.2°F
                timestamp:1708934060 → 68.0°F

Each row has different columns
Columns sorted by name → range scans extremely fast
Designed for massive write throughput
Data automatically partitioned across nodes
```

```
✅ Massive write throughput (designed for it)
✅ Efficient range scans (columns sorted)
✅ Scales linearly — add nodes, capacity grows
✅ Time-series data naturally modeled
✅ No single point of failure

❌ No joins, limited query flexibility
❌ Data model must be designed around access patterns
❌ Eventual consistency by default

Use for: IoT sensor data, time-series metrics,
         activity feeds, audit logs, weather data

Examples: Apache Cassandra, HBase, Google Bigtable
```

---

### 4. Graph Databases

Data stored as nodes and edges. Optimized for highly connected data where relationships ARE the data.

```
Nodes: Person, Movie, Genre
Edges: ACTED_IN, DIRECTED, LIKES, FOLLOWS

(Hari)-[LIKES]->(System Design)
(Hari)-[FOLLOWS]->(Ashish)
(Ashish)-[CREATED]->(AlgoMaster)
(AlgoMaster)-[TEACHES]->(System Design)

Query: "Find all topics Hari might like based on people he follows"
MATCH (u:Person {name: "Hari"})-[:FOLLOWS]->(friend)
      -[:LIKES]->(topic)
WHERE NOT (u)-[:LIKES]->(topic)
RETURN topic.name, COUNT(friend) as mutual_friends
ORDER BY mutual_friends DESC
```

```
✅ Natural for relationship-heavy data
✅ Multi-hop queries that would require many JOINs in SQL
✅ Recommendation engines, fraud detection, social networks
✅ Intuitive data model for connected domains

❌ Not great for non-graph data
❌ Harder to scale horizontally than other NoSQL
❌ Less mainstream tooling

Use for: Social networks, fraud detection, recommendation engines,
         knowledge graphs, network topology, access control

Examples: Neo4j, Amazon Neptune, JanusGraph
```

---

## SQL vs NoSQL — Direct Comparison

| Property | SQL | NoSQL |
|---------|-----|-------|
| Schema | Fixed, defined upfront | Flexible, per document |
| ACID Transactions | Full support | Varies by DB |
| Scaling | Vertical (primarily) | Horizontal (designed for) |
| Query Power | Very high (SQL) | Limited (by type) |
| Consistency | Strong (default) | Eventual (often default) |
| Relationships | First-class (JOINs) | Application-managed |
| Maturity | Decades | Younger |
| Best for | Complex queries, transactions | Simple patterns, massive scale |

---

## The Scaling Difference — Why It Matters

```
SQL scaling:
  Vertical: bigger machine (has ceiling)
  Horizontal (sharding):
    Shard by user_id across 10 nodes
    JOIN across shards? → must gather from multiple nodes
    → application must handle this
    → most SQL advantages disappear
    → very complex to operate

NoSQL scaling:
  Designed for horizontal from day one
  MongoDB: automatic sharding
  Cassandra: consistent hashing ring
  Redis Cluster: hash slots across nodes
  Add node → automatic rebalancing
  Linear scalability by design
```

---

## When to Choose What

```
Choose SQL (PostgreSQL/MySQL) when:
  → Financial transactions (banking, payments)
  → Complex relationships between entities
  → Complex reporting and analytics needed
  → Data integrity is paramount
  → Schema is well-understood upfront
  → Team knows SQL well

Choose MongoDB when:
  → Document-like data (product catalog, user profiles)
  → Schema evolves rapidly (startup, new product)
  → Nested/hierarchical data
  → Need to scale reads horizontally

Choose Redis when:
  → Caching (session, computed results)
  → Rate limiting counters
  → Real-time leaderboards
  → Pub/sub messaging
  → Any ephemeral, fast-access data

Choose Cassandra when:
  → Massive write throughput (IoT, logs, metrics)
  → Time-series data
  → Never need complex queries
  → Availability over consistency always
  → Global multi-region replication

Choose Neo4j when:
  → Relationship traversal is the primary operation
  → Social graph, fraud detection, recommendations
  → Data IS the relationships

Choose DynamoDB when:
  → AWS ecosystem
  → Known, simple access patterns
  → Managed, serverless, auto-scaling
  → Don't want to operate infrastructure
```

---

## The Polyglot Persistence Pattern

Real production systems don't choose one database — they use the right database for each use case:

```
E-commerce system:
  PostgreSQL   → orders, users, inventory (ACID critical)
  MongoDB      → product catalog (flexible schema, nested data)
  Redis        → sessions, cart, rate limiting, cache
  Elasticsearch→ product search (full-text, facets)
  Cassandra    → user activity events, clickstream

Railroad system:
  PostgreSQL   → train schedules, routes, bookings (ACID)
  Redis        → real-time train positions, cache
  Cassandra    → sensor telemetry from locomotives (time-series)
  Kafka        → event streaming between all systems
```

---

## NewSQL — The Middle Ground

Databases that give you SQL semantics with horizontal scalability:

```
CockroachDB:
  PostgreSQL-compatible SQL
  Distributed, auto-shards
  ACID transactions across shards
  Survives node failures automatically

Google Spanner:
  SQL + horizontal scale + global distribution
  Globally consistent transactions via TrueTime
  Used by Google for AdWords, Google Play

YugabyteDB:
  PostgreSQL-compatible
  Distributed ACID
  Open source

Use when: You want SQL semantics but need horizontal scale
          Usually worth it before abandoning SQL entirely
```

---

## Key Takeaways

```
SQL:
  Tables, schemas, ACID, JOINs, powerful queries
  Best for: transactions, complex relations, reporting
  Scales vertically, horizontal sharding is hard
  PostgreSQL → default choice when in doubt

NoSQL types:
  Document (MongoDB): nested objects, flexible schema
  Key-Value (Redis):  ultra-fast, simple lookups, cache
  Wide-Column (Cassandra): massive write throughput, time-series
  Graph (Neo4j):      relationship traversal, social/fraud

Key decision factors:
  Data model fit → does SQL or NoSQL express it naturally?
  Access patterns → simple KV vs complex joins?
  Scale requirements → vertical OK or need horizontal?
  Consistency needs → ACID required or eventual OK?
  Team expertise → known technology vs learning curve?

Real systems use multiple databases (polyglot persistence)
Right database per use case beats forcing one database everywhere

NewSQL (CockroachDB, Spanner): SQL + horizontal scale
       → Worth considering before switching to NoSQL
```
