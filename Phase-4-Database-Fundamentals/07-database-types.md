# Types of Databases

**Phase 4 — Database Fundamentals | Topic 7 of 9**

---

## Why This Topic Matters

In system design interviews, you're expected to choose the right database for each component of your system. Saying "I'd use a database" isn't enough — you need to say:

> "I'd use PostgreSQL for orders because of ACID requirements, Redis for session caching, Elasticsearch for product search, and Cassandra for IoT telemetry because of its write throughput."

---

## 1. Relational Databases (RDBMS)

**What they are:** Tables with rows and columns. Schema enforced. ACID transactions. SQL queries.

```
Core strengths:
  ACID transactions    → financial accuracy
  JOINs               → complex relationships
  Schema enforcement  → data integrity
  Mature tooling      → decades of optimization
  Ad-hoc queries      → business intelligence

Core weaknesses:
  Horizontal scaling hard
  Rigid schema → migrations painful at scale
  Not great for hierarchical/unstructured data

When to use:
✅ Financial transactions (banking, payments, accounting)
✅ User accounts and authentication
✅ Order management systems
✅ Inventory systems
✅ Any system where correctness > scale
✅ When you need complex queries you can't predict upfront
```

**Key products:**
```
PostgreSQL:  Open source, feature-rich, JSON support,
             full-text search, time-series extensions
             → Default choice for most applications

MySQL:       Most deployed DB globally, strong ecosystem
             → Web applications, e-commerce

Oracle:      Enterprise, expensive, feature-rich
             → Legacy enterprise systems

SQL Server:  Microsoft ecosystem, Azure integration
             → .NET shops, enterprise Windows environments

AWS Aurora:  Cloud-native MySQL/PostgreSQL compatible
             → 5x MySQL performance, auto-scaling storage
             → Serverless option for variable workloads
```

---

## 2. Document Databases

**What they are:** Data stored as JSON/BSON documents. Each document is self-contained. Flexible schema.

```
Core strengths:
  Natural data model  → document = application object
  Flexible schema     → add fields without migration
  Nested data         → no JOINs for contained data
  Horizontal scaling  → built-in sharding
  Developer friendly  → work with objects directly

Core weaknesses:
  No JOINs (application handles relationships)
  Potential data duplication
  Eventual consistency in some configurations
  Weaker transaction support (improving)

When to use:
✅ Product catalogs (each product has different attributes)
✅ User profiles (varied preference structures)
✅ Content management (articles, blog posts)
✅ Event logging (each event has different fields)
✅ Real-time analytics (pre-aggregated documents)
✅ Rapidly evolving schemas (startup, new product)
✅ When data is naturally hierarchical
```

**Key products:**
```
MongoDB:    Most popular document DB
            ACID transactions since v4.0
            Rich query language, aggregation pipeline
            Atlas: fully managed cloud service

CouchDB:    HTTP-based API, sync for offline-first apps
            Strong eventual consistency model

Firestore:  Google's managed document DB
            Real-time sync to mobile/web clients
            Serverless, scales automatically

DynamoDB:   AWS managed, key-value + document hybrid
            Single-digit millisecond latency at any scale
            Pay-per-request pricing
```

---

## 3. Key-Value Stores

**What they are:** Simplest data model. Map of keys to values. Extremely fast.

```
Core strengths:
  O(1) reads and writes
  Extremely high throughput
  Low latency (microseconds in-memory)
  Simple mental model
  Easy to distribute

Core weaknesses:
  Can only query by exact key
  No relationships
  No complex queries
  Values are opaque (DB doesn't understand structure)

When to use:
✅ Session storage (user_session:abc → session data)
✅ Caching (product:123 → product JSON)
✅ Rate limiting (rate:user:123:hour → count)
✅ Feature flags (feature:dark-mode:user:123 → true/false)
✅ Shopping carts (cart:user:123 → cart items)
✅ Leaderboards (sorted sets)
✅ Distributed locks (SET key value NX PX 30000)
✅ Pub/sub messaging
```

**Key products:**
```
Redis:      In-memory, sub-millisecond latency
            Rich data structures: strings, lists, sets,
            sorted sets, hashes, streams, HyperLogLog
            Persistence options (RDB snapshots, AOF log)
            Pub/sub, Lua scripting, transactions
            → Swiss army knife of databases
            → Default choice for caching and sessions

Memcached:  Simpler than Redis, pure cache
            Multi-threaded
            No persistence, no data structures beyond strings
            → Legacy, Redis is usually better choice

DynamoDB:   Also functions as key-value store at massive scale
            AWS managed, auto-scaling
            → When you need managed KV at internet scale

etcd:       Distributed KV store for configuration
            Strong consistency (Raft consensus)
            Used by Kubernetes for cluster state
            → Configuration and service discovery
```

---

## 4. Wide-Column (Column-Family) Databases

**What they are:** Rows with dynamic columns. Each row can have different columns. Sorted by row key. Optimized for writes.

```
Core strengths:
  Massive write throughput
  Efficient range scans (sorted storage)
  Linear scalability (add nodes = more capacity)
  High availability (no single point of failure)
  Time-series data naturally modeled
  Multi-datacenter replication built-in

Core weaknesses:
  Limited query flexibility (design around access patterns)
  Eventual consistency by default
  No JOINs
  Data model must be planned upfront for access patterns
  Operational complexity
```

**Cassandra Data Model:**
```sql
-- Partition key: determines which node stores the row
-- Clustering key: determines sort order within partition

CREATE TABLE sensor_readings (
    sensor_id    TEXT,
    recorded_at  TIMESTAMP,
    temperature  DECIMAL,
    humidity     DECIMAL,
    PRIMARY KEY (sensor_id, recorded_at)
    -- sensor_id = partition key
    -- recorded_at = clustering key (sorted)
);

-- Query: last 100 readings for sensor_001
SELECT * FROM sensor_readings
WHERE sensor_id = 'sensor_001'
ORDER BY recorded_at DESC
LIMIT 100;
-- → One partition, sorted scan → extremely fast

-- Design rule: queries you CANNOT predict should NOT go to Cassandra
-- Design your schema for your specific access patterns
```

```
When to use:
✅ IoT sensor data (billions of writes/day)
✅ Time-series metrics (infrastructure monitoring)
✅ User activity logs (clickstream, audit trails)
✅ Message storage at massive scale (Discord uses Cassandra)
✅ Write-heavy workloads with high availability requirement
✅ Multi-datacenter replication requirement
✅ When you have clear, predictable access patterns

Key products:
Apache Cassandra: Open source, industry standard
                  Used by: Netflix, Discord, Instagram, Apple

Google Bigtable:  Managed, powers Google Search
                  Basis for Apache HBase design

AWS Keyspaces:    Managed Cassandra-compatible service
                  → Cassandra without operational overhead

HBase:            Hadoop ecosystem, HDFS storage
                  Strong consistency (vs Cassandra eventual)
```

---

## 5. Search Engines

**What they are:** Specialized databases optimized for full-text search, relevance ranking, and faceted filtering.

```
Core strengths:
  Full-text search with relevance scoring
  Fuzzy matching (typo tolerance)
  Faceted search (filter by category, price range, rating)
  Aggregations and analytics
  Near-real-time indexing
  Distributed, scalable

Core weaknesses:
  Not a primary data store (sync from source of truth)
  Eventual consistency
  Operational complexity
  Not ACID transactional
  Expensive for large datasets

When to use:
✅ Product search (e-commerce search bar)
✅ Full-text document search
✅ Log analysis (ELK stack)
✅ Autocomplete / typeahead
✅ Geospatial search
✅ Complex faceted filtering
✅ Any search box that needs relevance ranking
```

**Elasticsearch example:**
```json
// Index a product
POST /products/_doc/123
{
  "name": "Mechanical Keyboard RGB Wireless",
  "category": "Electronics",
  "price": 149.99,
  "rating": 4.5,
  "description": "Compact TKL layout with blue switches..."
}

// Search with relevance, filters, facets
POST /products/_search
{
  "query": {
    "bool": {
      "must": {
        "match": {"name": "wireless keyboard"}
      },
      "filter": [
        {"range": {"price": {"lte": 200}}},
        {"term":  {"category": "Electronics"}}
      ]
    }
  },
  "aggs": {
    "price_ranges": {
      "range": {
        "field": "price",
        "ranges": [{"to": 50}, {"from": 50, "to": 150}, {"from": 150}]
      }
    }
  },
  "sort": [{"_score": "desc"}, {"rating": "desc"}]
}
```

**Key products:**
```
Elasticsearch: Most popular, Lucene-based, ELK stack
               → Product search, log analysis, monitoring

OpenSearch:    AWS fork of Elasticsearch (open source)
               → Same API, AWS managed

Typesense:     Newer, simpler, fast typo-tolerance
               → Smaller scale, easier to operate

Meilisearch:   Open source, developer-friendly
               → Typo-tolerant, instant search
```

---

## 6. Time-Series Databases

**What they are:** Optimized for timestamped data points. Automatic downsampling, retention policies, specialized time-based aggregations.

```
Core strengths:
  Extremely efficient time-series storage
  Built-in downsampling (aggregate old data)
  Retention policies (auto-delete old data)
  Time-based aggregations (avg, min, max over time windows)
  Compression optimized for sequential numeric data
  Fast ingestion of many small writes

Core weaknesses:
  Only useful for time-series data
  Not a general-purpose database
  Limited query flexibility outside time-based queries

When to use:
✅ Infrastructure metrics (CPU, memory, disk over time)
✅ Application performance monitoring (request rate, latency)
✅ IoT sensor readings
✅ Financial market data (stock prices)
✅ Any data where time IS the primary dimension
```

**InfluxDB example:**
```
Measurement: cpu_usage
Tags (indexed): host=server01, region=us-east
Fields: value=82.5
Timestamp: 2026-02-14T10:00:00Z

-- Query: average CPU per host last hour, 1-min intervals
SELECT MEAN(value) FROM cpu_usage
WHERE time > NOW() - 1h
GROUP BY time(1m), host

Retention policy: keep raw data 7 days
                 keep 1-min aggregates 30 days
                 keep 1-hour aggregates 1 year
                 auto-delete older raw data
```

**Key products:**
```
InfluxDB:      Most popular time-series DB
               → Infrastructure monitoring, IoT

Prometheus:    Metrics collection + alerting
               → Kubernetes monitoring standard
               → Pull-based metrics scraping

TimescaleDB:   PostgreSQL extension for time-series
               → SQL familiarity + time-series optimization
               → If you already use PostgreSQL

AWS Timestream: Managed time-series, serverless
```

---

## 7. Graph Databases

**What they are:** Data stored as nodes and edges. Optimized for traversing relationships.

```
Core strengths:
  Natural for highly connected data
  Multi-hop traversals fast (no expensive JOINs)
  Intuitive data model for relationships
  Powerful graph query languages (Cypher, Gremlin)
  Fraud detection patterns

Core weaknesses:
  Not horizontally scalable (most products)
  Overkill for simple relationships
  Less mature ecosystem
  SQL teams face learning curve

When to use:
✅ Social networks (friends of friends)
✅ Fraud detection (transaction patterns, device sharing)
✅ Recommendation engines (users who bought X also bought Y)
✅ Knowledge graphs
✅ Network/infrastructure topology
✅ Access control (role hierarchies, permission inheritance)
```

**Neo4j example:**
```cypher
-- Fraud detection: find accounts sharing device
MATCH (a1:Account)-[:USED_DEVICE]->(d:Device)<-[:USED_DEVICE]-(a2:Account)
WHERE a1 <> a2
AND a1.flagged = true
RETURN a2.id, a2.email, d.deviceId
ORDER BY a2.createdAt DESC

-- This query is trivial in graph DB
-- Equivalent SQL requires multiple JOINs and is slow at scale
```

**Key products:**
```
Neo4j:          Most mature, Cypher query language
                → Social, recommendation, fraud

Amazon Neptune: Managed, supports Gremlin + SPARQL
                → AWS ecosystem graph workloads

JanusGraph:     Open source, scales horizontally
                → Large-scale graphs, Cassandra backend
```

---

## 8. NewSQL

**What they are:** SQL semantics with horizontal scalability.

```
Core strengths:
  Standard SQL interface
  ACID transactions distributed across nodes
  Horizontal write scalability
  Auto-sharding and rebalancing
  High availability built-in

When to use:
✅ Need SQL + horizontal write scale
✅ Global distribution with strong consistency
✅ When sharding traditional DB is too complex
✅ Replacing legacy Oracle/SQL Server at scale

Key products:
CockroachDB:  PostgreSQL-compatible, distributed ACID
              Survives datacenter failures automatically

Google Spanner: Global consistency via TrueTime
                Petabyte-scale

YugabyteDB:   PostgreSQL-compatible, open source

PlanetScale:  MySQL-compatible, serverless
```

---

## The Database Selection Framework

```
Question 1: Do I need ACID transactions?
  Yes, complex → PostgreSQL / MySQL
  Yes, distributed → CockroachDB / Spanner
  No → consider NoSQL

Question 2: What is my primary access pattern?
  Key lookup → Redis / DynamoDB
  Document retrieval → MongoDB
  Full-text search → Elasticsearch
  Time-series → InfluxDB / TimescaleDB
  Relationship traversal → Neo4j
  Wide-column writes → Cassandra

Question 3: What is my scale requirement?
  Moderate (< 10M rows) → any SQL DB
  Large (10M-1B rows) → SQL with read replicas + caching
  Massive (> 1B rows) → NoSQL or sharded SQL

Question 4: What is my consistency requirement?
  Strong → PostgreSQL, CockroachDB
  Eventual is OK → Cassandra, DynamoDB, MongoDB

Question 5: Is my schema stable or evolving?
  Stable, well-understood → SQL
  Evolving, varied → Document DB
```

---

## Real System Database Choices

```
Netflix:
  MySQL → billing, user accounts (ACID required)
  Cassandra → viewing history, activity (massive write scale)
  Elasticsearch → search
  EVCache (Redis) → distributed cache

Discord:
  Cassandra → messages (trillions, write-heavy)
  PostgreSQL → users, servers, channels
  Redis → presence, typing indicators

Uber:
  MySQL/PostgreSQL → trips, users, payments
  Cassandra → location updates (billions/day)
  Elasticsearch → search
  Redis → caching, rate limiting
```

---

## Key Takeaways

```
8 database types and their sweet spots:

Relational (PostgreSQL):  ACID, complex queries, relationships
                          → Default choice, financial data

Document (MongoDB):       Flexible schema, nested data
                          → Product catalog, user profiles

Key-Value (Redis):        Ultra-fast, O(1) lookups
                          → Cache, sessions, rate limiting

Wide-Column (Cassandra):  Massive write throughput, time-series
                          → IoT, activity logs, metrics

Search (Elasticsearch):   Full-text, relevance, facets
                          → Product search, log analysis

Time-Series (InfluxDB):   Timestamps, downsampling, retention
                          → Metrics, IoT, monitoring

Graph (Neo4j):            Relationship traversal
                          → Social, fraud, recommendations

NewSQL (CockroachDB):     SQL + horizontal scale
                          → When you need both

Selection framework:
  ACID needed? → SQL / NewSQL
  Access pattern? → matches DB strength
  Scale? → drives horizontal requirement
  Consistency? → strong vs eventual

Real systems use multiple databases
Right database per use case beats one-size-fits-all
```
