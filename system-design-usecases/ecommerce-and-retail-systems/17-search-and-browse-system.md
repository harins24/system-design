---
title: Search and Browse System
layout: default
---

# Search and Browse System — Deep Dive Design

> **Scenario:** Design a large-scale search and browsing system for an e-commerce platform. Requirements include keyword search ("red nike shoes size 10"), autocomplete suggestions, advanced filters (price, brand, category, ratings, availability), faceted navigation, typo tolerance and synonyms, and personalized result ranking. Must handle 50,000 searches/second at peak with a catalog of 10 million products.
>
> **Tech Stack:** Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka (Elasticsearch for search)

## Table of Contents

1. [Requirements & Constraints](#requirements--constraints)
2. [Capacity Estimation](#capacity-estimation)
3. [High-Level Architecture](#high-level-architecture)
4. [Core Design Questions Answered](#core-design-questions-answered)
5. [Microservices Breakdown](#microservices-breakdown)
6. [Database Design](#database-design)
7. [Redis Data Structures](#redis-data-structures)
8. [Kafka Event Flow](#kafka-event-flow)
9. [Implementation Code](#implementation-code)
10. [Failure Scenarios & Mitigations](#failure-scenarios--mitigations)
11. [Scaling Strategy](#scaling-strategy)
12. [Monitoring & Observability](#monitoring--observability)
13. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Requirements & Constraints

### Functional Requirements
- Keyword search with typo tolerance and synonyms
- Autocomplete suggestions (prefix-based)
- Advanced filtering: price ranges, brands, categories, ratings, availability
- Faceted navigation with counts
- Result ranking by relevance, popularity, and personalization
- Search result pagination (20 results per page)
- Real-time index updates (new products indexed within 5 minutes)
- Support 10 million products in catalog
- Handle seasonal trends and trending searches

### Non-Functional Requirements
- Search latency: < 200ms (95th percentile)
- Autocomplete latency: < 100ms
- Throughput: 50,000 searches/sec peak
- Index freshness: < 5 minutes
- Search relevance: Top-5 precision > 90%
- System availability: 99.99%
- Support concurrent searches and indexing without conflicts

### Constraints
- No real-time personalization (would exceed latency budget)
- Relevance ranking must balance speed and quality
- Storage: ~500GB for full inverted index
- Network bandwidth: ~50Gbps peak for result delivery

---

## Capacity Estimation

| Metric | Calculation | Value |
|--------|-------------|-------|
| **Daily Searches** | Estimated platform traffic | 2,000,000,000 (2B) |
| **Searches/Second (Avg)** | 2B / 86400s | ~23,100 searches/sec |
| **Searches/Second (Peak)** | 5x average during sales | ~115,500 searches/sec |
| **Unique Search Terms/Day** | 1-5% of total searches | ~40-100M unique queries |
| **Autocomplete Requests/Sec** | 60% of searches have autocomplete | ~69,000 autocomplete/sec |
| **Avg Search Latency** | Target < 200ms | 150ms target |
| **Avg Results per Search** | Pagination: 20 results | 20 products |
| **Index Size (Elasticsearch)** | 10M products * 50KB avg | ~500GB |
| **Index Shards** | 500GB / 50GB per shard | 10 primary shards |
| **Replicas** | 1 replica per shard | 20 total shards |
| **Indexing Rate** | 10M catalog updated daily | ~115 products/sec |
| **Fresh Index Updates** | New/updated products | ~1000 updates/sec peak |
| **Query Cache Hit Rate** | Top 100 queries repeated | ~40% cache hits expected |
| **Memory (Redis for Autocomplete)** | Top 1M queries, ~100 bytes each | ~100MB |

---

## High-Level Architecture

```
┌─────────────────────────────────────┐
│        User Frontend                │
│  (Web Browser / Mobile App)         │
└────────────┬────────────────────────┘
             │
             ▼
    ┌────────────────────────┐
    │  Load Balancer (HAProxy)│
    │  50k searches/sec       │
    └────────────┬───────────┘
             │
    ┌────────┴────────────────┐
    │                         │
    ▼                         ▼
┌─────────────────┐   ┌─────────────────┐
│ Search Service  │   │ Search Service  │
│ (Instance 1)    │   │ (Instance N)    │
└────────┬────────┘   └────────┬────────┘
         │                     │
    ┌────┴─────────────────────┴────┐
    │                               │
    ▼                               ▼
┌──────────────────┐   ┌──────────────────────┐
│  Elasticsearch   │   │  Query Cache (Redis) │
│  - 10 shards     │   │  - Top queries       │
│  - 1 replica     │   │  - TTL: 1 hour       │
│  - 500GB         │   └──────────────────────┘
└────────┬─────────┘
         │
    ┌────┴────────────────────────┐
    │                             │
    ▼                             ▼
┌─────────────────┐   ┌──────────────────────┐
│ Product Catalog │   │ Personalization      │
│ (PostgreSQL)    │   │ (Redis + MongoDB)    │
└─────────────────┘   └──────────────────────┘

Indexing Pipeline:
┌─────────────────────────────────┐
│   Catalog Service               │
│   (catalog.updated events)      │
└────────────┬────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │  Kafka Topic       │
    │  catalog.updated   │
    └────────────┬───────┘
             │
             ▼
    ┌────────────────────┐
    │ Index Consumer     │
    │ (SearchIndexing    │
    │  Service)          │
    └────────────┬───────┘
             │
             ▼
    ┌────────────────────┐
    │ Elasticsearch      │
    │ (Bulk Index)       │
    └────────────────────┘
```

---

## Core Design Questions Answered

### 1. What search technology would you use?

**Answer:** Elasticsearch (open-source, scalable search engine).

**Why Elasticsearch:**
- **Inverted index:** Fast full-text search
- **Horizontal scaling:** Distributed sharding
- **Real-time indexing:** Low latency for index updates (100ms+)
- **Advanced features:** Fuzzy matching (typos), synonyms, analyzers
- **Faceted search:** Built-in aggregations for filters
- **Relevance scoring:** BM25 algorithm + custom scoring
- **High availability:** Replica shards for fault tolerance

**Alternatives Considered:**
| Solution | Pros | Cons |
|----------|------|------|
| **Elasticsearch** | Feature-rich, distributed, proven | Memory-heavy (500GB), operational complexity |
| **Solr** | Similar to ES, lighter weight | Less popular, smaller ecosystem |
| **Database Full-Text** (PostgreSQL) | Simpler ops, cheaper | Poor performance at scale, limited features |
| **Custom Inverted Index** | Full control, lightweight | High engineering effort, bug-prone |

**Decision:** Elasticsearch for production (feature completeness, proven at scale).

### 2. How do you index products for fast search?

**Answer:** Two-tier indexing strategy: batch indexing for full catalog + real-time indexing for updates.

**Indexing Strategy:**

```
Phase 1: Batch Index (Daily, off-peak)
  - Full catalog: 10M products
  - Time: 2-4 hours
  - Process:
    1. Export products from PostgreSQL
    2. Enrich with search signals (views, ratings)
    3. Build inverted index
    4. Create new ES index (blue-green strategy)
    5. Switch read traffic to new index
    6. Keep old index for rollback (24h)

Phase 2: Real-Time Indexing (Continuous)
  - New/updated products: ~1000/sec peak
  - Latency: < 5 seconds
  - Process:
    1. Catalog service publishes catalog.updated event
    2. Kafka consumer buffers updates
    3. Bulk index every 1 second (micro-batches)
    4. Update goes live immediately in ES

Index Update Frequency:
  - New product listings: Real-time (< 5 sec)
  - Price changes: Real-time
  - Inventory updates: Real-time
  - Popularity metrics (views/ratings): Hourly batch
```

**Mapping (Schema):**
```json
{
  "mappings": {
    "properties": {
      "product_id": { "type": "keyword" },
      "sku": { "type": "keyword" },

      "name": {
        "type": "text",
        "analyzer": "search_analyzer",
        "fields": {
          "keyword": { "type": "keyword" },
          "autocomplete": { "type": "text", "analyzer": "edge_ngram_analyzer" }
        }
      },

      "description": {
        "type": "text",
        "analyzer": "search_analyzer"
      },

      "category": { "type": "keyword" },
      "subcategory": { "type": "keyword" },
      "brand": { "type": "keyword" },

      "price": { "type": "scaled_float", "scaling_factor": 100 },
      "original_price": { "type": "scaled_float", "scaling_factor": 100 },

      "rating": { "type": "float", "index": false },
      "review_count": { "type": "integer", "index": false },

      "inventory": { "type": "integer", "index": false },
      "in_stock": { "type": "boolean" },

      "popularity_score": { "type": "float" }, // views + sales
      "freshness_score": { "type": "float" }, // days since update

      "tags": { "type": "keyword" },
      "colors": { "type": "keyword" },
      "sizes": { "type": "keyword" },

      "image_url": { "type": "keyword", "index": false },
      "created_at": { "type": "date" },
      "updated_at": { "type": "date" }
    }
  }
}
```

### 3. How do you implement autocomplete with low latency?

**Answer:** Two-level caching: Redis for top queries + Elasticsearch suggest API.

**Autocomplete Strategy:**

```
Request Flow:
  1. User types: "red" (3 characters)
  2. Frontend sends: GET /autocomplete?q=red
  3. Backend checks Redis (hot cache): O(1) lookup
  4. If hit: Return top 10 suggestions immediately
  5. If miss: Query Elasticsearch suggest API
  6. Cache result in Redis (1 hour TTL)

Data Structures:
  Redis: Sorted Set (by frequency)
    ZREVRANGE top:queries:2026-04-01 0 9
    → ["red nike shoes", "red adidas", "red puma", ...]

Elasticsearch: Edge N-gram analyzer
    "red nike shoes" → ["r", "re", "red", "red ", "red n", ...]
    Prefix match: O(log N) lookup in trie

Redis Cache:
  Key: autocomplete:{query}:{date}
  Value: [suggestions with counts]
  TTL: 1 hour
  Size: ~100MB for top 1M queries
```

**Implementation:**
```java
// Cache hit: < 10ms
GET redis: autocomplete:red:2026-04-01
→ ["red nike shoes:85000", "red adidas shoes:45000", ...]

// Cache miss: ~50-100ms
GET elasticsearch: POST /products/_search
  {
    "suggest": {
      "product-suggest": {
        "prefix": "red",
        "completion": {
          "field": "name.autocomplete",
          "size": 10
        }
      }
    }
  }
```

### 4. How do you handle typos and synonyms?

**Answer:** Elasticsearch analyzers with fuzzy matching and synonym expansion.

**Typo Tolerance:**

```
Approach 1: Fuzzy Matching (Edit Distance)
  Query: "nikee shoes"
  Fuzzy: { "name": { "fuzz": { "value": "nike", "fuzziness": "AUTO" } } }
  Matches: "nike" (1 character edit distance)
  Latency: Slow (expensive operation), use sparingly

Approach 2: N-gram Analyzer (Bi-grams)
  Index: "nike" → tokens: ["n", "ni", "ik", "ke", "e"]
  Query: "nikee" → tokens: ["n", "ni", "ik", "ke", "ee"]
  Overlap: 4/5 tokens match → Score 0.8
  Latency: Fast, pre-indexed

Approach 3: Phonetic Matching (Soundex/Metaphone)
  Query: "nike" (primary: "NK")
  Match: "nike", "nikee", "nyke" (same phonetic code)
  Use: Only for brand names or premium searches

Chosen: N-gram (default) + optional phonetic for premium queries
```

**Synonym Expansion:**

```
Elasticsearch Synonym Filter:
  sneaker, sneaker shoe, trainer, running shoe
  sports shoes, athletic footwear

Query: "sports shoes"
Expands to: (sports OR sneaker OR trainer OR "athletic footwear")

Storage: Synonym list in PostgreSQL + cached in Elasticsearch
Update: Rebuild analyzer configuration when synonyms change
```

### 5. How do you personalize search results?

**Answer:** Feature injection at query time (no ML model needed, < 200ms latency).

**Personalization Strategy:**

```
User Signals Collected (Real-time):
  - Recent browsed categories
  - Previous purchases
  - Search history
  - Wishlist items
  - Preferred brands

Feature Injection:
  1. Fetch user's top 3 categories from Redis (10ms)
  2. Boost products from those categories +20 points
  3. Fetch user's wishlist items (20ms)
  4. Boost wishlist items +50 points
  5. Apply user's minimum price preference
  6. Adjust filters based on purchase history

Example Query Transformation:
  Original query: "shoes"
  User profile: Prefers Nike, purchased running shoes, browses women's

  Boosted query:
    {
      "bool": {
        "must": [{ "match": { "name": "shoes" } }],
        "should": [
          { "term": { "brand": { "value": "Nike", "boost": 2.0 } } },
          { "term": { "category": { "value": "Women's", "boost": 1.5 } } },
          { "term": { "category": { "value": "Running", "boost": 1.5 } } }
        ]
      }
    }

Cache user profile in Redis:
  user:{userId}:search_profile
  Fields: favorite_brands, categories, min_price, max_price, etc.
  TTL: 1 day

Personalization Latency Budget:
  Total: 200ms
  ├─ Load user profile (Redis): 10ms
  ├─ Build query: 5ms
  ├─ Search (Elasticsearch): 150ms
  └─ Format results: 35ms
```

### 6. How do you keep search index in sync with product catalog?

**Answer:** Event-driven indexing via Kafka + dead letter queue (DLQ) for failures.

**Sync Strategy:**

```
Guarantee: Eventually consistent (within 5 seconds)

Flow:
  1. Product updated in PostgreSQL
  2. Catalog service publishes catalog.updated event to Kafka
  3. SearchIndexingConsumer subscribes to catalog.updated
  4. Consumer validates product data
  5. Consumer sends bulk index request to Elasticsearch
  6. If success: commit Kafka offset
  7. If failure: send to DLQ for retry

DLQ Retry Policy:
  Attempt 1: 1 second later
  Attempt 2: 10 seconds later
  Attempt 3: 1 minute later
  Attempt 4: 10 minutes later
  Final: Manual review + re-index

Deduplication:
  Index contains: (product_id, version)
  New version arrives: If version > current, update
  Prevent: Replay of old versions with stale data

Rollback Safety:
  Blue-green indexing:
    1. Index batch writes to ES index "products_v2"
    2. Verify quality (sample search tests)
    3. Switch traffic: PUT /products/_alias → products_v2
    4. Keep old index 24h for rollback
```

---

## Microservices Breakdown

| Service | Responsibility | Tech Stack | Latency |
|---------|-----------------|-------------|---------|
| **Search Service** | Executes search queries, applies filters | Spring Boot + Elasticsearch Java Client | < 200ms |
| **Autocomplete Service** | Prefix-based autocomplete suggestions | Spring Boot + Redis + Elasticsearch | < 100ms |
| **Search Indexing Consumer** | Consumes catalog events, bulk indexes | Spring Boot + Kafka Consumer | < 5s latency |
| **Search Ranking Service** | BM25 + popularity + personalization scoring | Spring Boot + Elasticsearch | Included in search latency |
| **Facet Builder** | Aggregates facets (brands, categories, prices) | Spring Boot + Elasticsearch | < 50ms |
| **Synonym Manager** | Manages and updates synonyms | Spring Boot REST + PostgreSQL | Admin-level |
| **Search Analytics Service** | Tracks popular searches, trending | Spring Boot + Kafka Consumer + MongoDB | Real-time aggregation |

---

## Database Design

### PostgreSQL Schema: Product Catalog & Metadata

```sql
-- Products (source of truth)
CREATE TABLE products (
  product_id BIGINT PRIMARY KEY,
  sku VARCHAR(100) UNIQUE NOT NULL,
  name VARCHAR(500) NOT NULL,
  description TEXT,
  category VARCHAR(100) NOT NULL,
  subcategory VARCHAR(100),
  brand VARCHAR(100) NOT NULL,

  price DECIMAL(12, 2) NOT NULL,
  original_price DECIMAL(12, 2),
  discount_percentage DECIMAL(5, 2),

  rating DECIMAL(3, 2),
  review_count INT DEFAULT 0,

  inventory INT DEFAULT 0,
  in_stock BOOLEAN DEFAULT FALSE,

  image_url VARCHAR(500),
  thumbnail_url VARCHAR(500),

  popularity_score FLOAT DEFAULT 0, -- Updated hourly
  view_count BIGINT DEFAULT 0,
  sale_count BIGINT DEFAULT 0,

  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),

  INDEX idx_brand_category (brand, category),
  INDEX idx_price (price),
  INDEX idx_rating (rating),
  INDEX idx_in_stock (in_stock),
  INDEX idx_updated_at (updated_at DESC),
  FULLTEXT INDEX idx_name_description (name, description)
);

-- Product attributes (colors, sizes, etc.)
CREATE TABLE product_attributes (
  attribute_id BIGINT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
  product_id BIGINT NOT NULL REFERENCES products(product_id),
  attribute_type VARCHAR(50), -- 'color', 'size', 'material', etc.
  attribute_value VARCHAR(100),
  created_at TIMESTAMP DEFAULT NOW(),
  INDEX idx_product_id (product_id),
  INDEX idx_type_value (attribute_type, attribute_value)
);

-- Search synonyms (updateable without ES restart)
CREATE TABLE search_synonyms (
  synonym_id BIGINT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
  synonym_group VARCHAR(255), -- e.g., "shoe_types"
  terms TEXT NOT NULL, -- CSV: "sneaker, trainer, running shoe"
  enabled BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  created_by VARCHAR(100),
  INDEX idx_enabled (enabled)
);

-- Search analytics (popular queries, trends)
CREATE TABLE search_analytics (
  analytics_id BIGINT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
  query_text VARCHAR(500) NOT NULL,
  query_date DATE NOT NULL,
  search_count INT DEFAULT 0,
  click_count INT DEFAULT 0,
  conversion_count INT DEFAULT 0,
  average_result_count INT,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE (query_text, query_date),
  INDEX idx_date_search (query_date DESC, search_count DESC)
);

-- Catalog update log (for auditing and replaying)
CREATE TABLE catalog_update_log (
  log_id BIGINT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
  product_id BIGINT NOT NULL,
  change_type VARCHAR(50), -- 'CREATE', 'UPDATE', 'DELETE'
  change_details JSONB,
  updated_at TIMESTAMP DEFAULT NOW(),
  updated_by VARCHAR(100),
  INDEX idx_product_id (product_id),
  INDEX idx_updated_at (updated_at DESC)
);
```

### MongoDB Schema: Search Analytics & Trending

```javascript
// Collection: search_queries (real-time search tracking)
db.createCollection("search_queries", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["_id", "query", "user_id", "timestamp"],
      properties: {
        _id: { bsonType: "objectId" },
        query: { bsonType: "string" },
        user_id: { bsonType: "long" },
        session_id: { bsonType: "string" },

        result_count: { bsonType: "int" },
        clicked_products: { bsonType: "array" },
        first_click_position: { bsonType: "int" },

        filters_applied: {
          bsonType: "object",
          properties: {
            categories: { bsonType: "array" },
            brands: { bsonType: "array" },
            price_range: { bsonType: "object" },
            rating_min: { bsonType: "double" }
          }
        },

        session_duration_ms: { bsonType: "int" },
        converted: { bsonType: "bool" },
        conversion_value: { bsonType: "decimal" },

        device_type: { enum: ["MOBILE", "TABLET", "DESKTOP"] },
        region: { bsonType: "string" },

        timestamp: { bsonType: "date" },
        created_at: { bsonType: "date" }
      }
    }
  }
});

db.search_queries.createIndex({ "query": 1, "timestamp": -1 });
db.search_queries.createIndex({ "user_id": 1, "timestamp": -1 });
db.search_queries.createIndex({ "timestamp": 1 }, { expireAfterSeconds: 7776000 }); // 90-day TTL

// Collection: trending_queries (hourly aggregation)
db.createCollection("trending_queries", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["_id", "query", "hour", "search_count"],
      properties: {
        _id: { bsonType: "objectId" },
        query: { bsonType: "string" },
        hour: { bsonType: "date" }, // Start of hour
        search_count: { bsonType: "int" },
        click_rate: { bsonType: "double" },
        conversion_rate: { bsonType: "double" },
        avg_result_count: { bsonType: "double" },
        top_clicked_products: { bsonType: "array" },
        created_at: { bsonType: "date" }
      }
    }
  }
});

db.trending_queries.createIndex({ "hour": -1, "search_count": -1 });
db.trending_queries.createIndex({ "query": 1, "hour": -1 });
```

---

## Redis Data Structures

```redis
# 1. Autocomplete Cache (Top Queries)
autocomplete:{query}:{date}
  Type: Sorted Set
  Score: frequency (search count)
  Member: suggestion text
  TTL: 1 day
  Example: autocomplete:red:2026-04-01 → ["red shoes:10000", "red dress:5000"]

# 2. User Search Profile (Personalization)
user:{userId}:search_profile
  Type: Hash
  Fields: {
    favorite_brands: ["Nike", "Adidas"],
    favorite_categories: ["Shoes", "Apparel"],
    min_price: 50.00,
    max_price: 500.00,
    search_history: [...],
    last_updated: timestamp
  }
  TTL: 1 day

# 3. Query Result Cache
search:result:{hash(query + filters)}
  Type: String (serialized JSON)
  Value: cached search results (products)
  TTL: 1 hour
  Example: search:result:abc123def456 → [product objects]

# 4. Facet Counts Cache
search:facets:{query}
  Type: Hash
  Fields: {
    "Nike": 5000,
    "Adidas": 3500,
    "Puma": 2200,
    ...
  }
  TTL: 1 hour

# 5. Popular Searches (Hourly)
popular:searches:{hour}
  Type: Sorted Set
  Score: search count
  Member: query text
  TTL: 30 days
  Example: popular:searches:2026-04-01-14 → ["nike shoes:5000", "adidas pants:3000"]

# 6. Search Trending (Real-time Delta)
trending:searches:today
  Type: Sorted Set (by velocity)
  Score: search_count_today / search_count_yesterday (trend factor)
  Member: query text
  TTL: 1 day
  Example: trending:searches:today → ["summer shoes:5.2", "waterproof jacket:4.8"]

# 7. Index Version (Blue-Green Switching)
search:index:version
  Type: String
  Value: "v20260401_001"
  TTL: No expiry (manual update)

search:index:alias:{alias}
  Type: String
  Value: "products_v20260401_001"
  TTL: No expiry

# 8. Rate Limiter (Search API)
ratelimit:search:{userId}
  Type: String (Counter)
  Value: request count
  TTL: 1 minute
  Example: ratelimit:search:user123 → "450"
```

---

## Kafka Event Flow

### Topics & Partitioning

```
Topic: catalog.updated
  Partitions: 100 (by product_id)
  Retention: 7 days
  Event Schema:
    {
      "event_id": "uuid",
      "product_id": 123456,
      "sku": "NIKE-AIR-001",
      "change_type": "UPDATE", // CREATE, UPDATE, DELETE
      "change_fields": {
        "name": "Nike Air Max",
        "price": 150.00,
        "inventory": 50,
        "rating": 4.5
      },
      "timestamp": "2026-04-01T10:00:00Z"
    }

Topic: search.query.completed
  Partitions: 50 (by user_id)
  Retention: 90 days
  Event Schema:
    {
      "event_id": "uuid",
      "query": "red nike shoes",
      "user_id": 12345,
      "session_id": "sess_xyz",
      "result_count": 250,
      "filters": { "brand": ["Nike"], "price_max": 200 },
      "clicked_products": [123, 456],
      "conversion": false,
      "latency_ms": 145,
      "timestamp": "2026-04-01T10:00:00Z"
    }

Topic: search.index.operation
  Partitions: 10
  Retention: 30 days
  Event Schema:
    {
      "operation_type": "BULK_INDEX",
      "product_count": 10000,
      "index_version": "v20260401_001",
      "duration_ms": 2500,
      "status": "SUCCESS",
      "timestamp": "2026-04-01T02:00:00Z"
    }
```

---

## Implementation Code

### 1. Search Service (Main Query Handler)

```java
@Service
@Slf4j
public class SearchService {

    @Autowired private RestHighLevelClient elasticsearchClient;
    @Autowired private RedisTemplate<String, Object> redisTemplate;
    @Autowired private UserRepository userRepository;
    @Autowired private MeterRegistry meterRegistry;
    @Autowired private ObjectMapper objectMapper;
    @Autowired private KafkaTemplate<String, String> kafkaTemplate;

    private static final int DEFAULT_PAGE_SIZE = 20;
    private static final int SEARCH_TIMEOUT_MS = 200;
    private static final String CACHE_KEY_PREFIX = "search:result:";

    /**
     * Execute search with filters, ranking, and personalization
     */
    public SearchResults search(SearchQuery searchQuery, String userId) {
        Timer.Sample sample = Timer.start(meterRegistry);
        long startTime = System.currentTimeMillis();

        try {
            log.info("Searching for: {} (user: {}, filters: {})",
                searchQuery.getQuery(), userId, searchQuery.getFilters());

            // 1. Check cache for identical query
            String cacheKey = buildCacheKey(searchQuery, userId);
            SearchResults cachedResults = getFromCache(cacheKey);
            if (cachedResults != null) {
                meterRegistry.counter("search.cache.hits").increment();
                return cachedResults;
            }

            // 2. Build Elasticsearch query
            SearchRequest elasticsearchQuery = buildElasticsearchQuery(searchQuery, userId);

            // 3. Execute search with timeout
            SearchResults results = executeSearchWithTimeout(elasticsearchQuery, SEARCH_TIMEOUT_MS);

            // 4. Personalize results if user is logged in
            if (userId != null && !userId.isEmpty()) {
                personalizeResults(results, userId);
            }

            // 5. Cache results
            cacheResults(cacheKey, results);

            // 6. Publish search analytics event (async)
            publishSearchAnalyticsEvent(searchQuery, results, userId);

            long duration = System.currentTimeMillis() - startTime;
            log.info("Search completed in {}ms. Results: {}",
                duration, results.getTotalHits());

            // 7. Record metrics
            recordSearchMetrics(searchQuery, results, duration);

            return results;

        } catch (Exception e) {
            log.error("Search failed: {}", e.getMessage(), e);
            sample.stop(meterRegistry.timer("search.error"));
            throw new SearchException("Search execution failed", e);
        }
    }

    /**
     * Build Elasticsearch query with filters and ranking
     */
    private SearchRequest buildElasticsearchQuery(SearchQuery searchQuery, String userId) {
        SearchRequest request = new SearchRequest("products");
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();

        // 1. Text search (multi-field with boosting)
        BoolQueryBuilder boolQuery = QueryBuilders.boolQuery();

        if (searchQuery.getQuery() != null && !searchQuery.getQuery().isEmpty()) {
            MultiMatchQueryBuilder textQuery = QueryBuilders.multiMatchQuery(
                searchQuery.getQuery()
            )
            .field("name", 3.0f)           // Boost name matches
            .field("description", 1.0f)
            .field("brand", 2.0f)          // Boost brand matches
            .field("tags", 1.5f)
            .type(MultiMatchQueryBuilder.Type.BEST_FIELDS)
            .fuzziness(Fuzziness.AUTO)     // Typo tolerance
            .prefixLength(2);

            boolQuery.must(textQuery);
        }

        // 2. Filter by selected facets
        if (searchQuery.getFilters() != null) {
            if (searchQuery.getFilters().getCategories() != null) {
                boolQuery.filter(QueryBuilders.termsQuery(
                    "category",
                    searchQuery.getFilters().getCategories()
                ));
            }

            if (searchQuery.getFilters().getBrands() != null) {
                boolQuery.filter(QueryBuilders.termsQuery(
                    "brand",
                    searchQuery.getFilters().getBrands()
                ));
            }

            if (searchQuery.getFilters().getPriceMin() != null &&
                searchQuery.getFilters().getPriceMax() != null) {
                boolQuery.filter(QueryBuilders.rangeQuery("price")
                    .gte(searchQuery.getFilters().getPriceMin())
                    .lte(searchQuery.getFilters().getPriceMax())
                );
            }

            if (searchQuery.getFilters().getRatingMin() != null) {
                boolQuery.filter(QueryBuilders.rangeQuery("rating")
                    .gte(searchQuery.getFilters().getRatingMin())
                );
            }

            if (searchQuery.getFilters().isInStockOnly()) {
                boolQuery.filter(QueryBuilders.termQuery("in_stock", true));
            }
        }

        // 3. Personalization boosts (if user logged in)
        if (userId != null && !userId.isEmpty()) {
            applyPersonalizationBoosts(boolQuery, userId);
        }

        // 4. Scoring: BM25 + popularity + freshness
        FunctionScoreQueryBuilder scoringQuery = QueryBuilders.functionScoreQuery(boolQuery)
            .addScriptScore(
                new Script("Math.log(2 + doc['popularity_score'].value)"),
                ScoreFunctionBuilders.scriptFunction(
                    new Script("Math.log(2 + doc['popularity_score'].value)")
                ),
                0.3f  // 30% weight to popularity
            )
            .addScriptScore(
                new Script("(1.0 / (1.0 + Math.abs(now - doc['updated_at'].value) / 86400000))"),
                ScoreFunctionBuilders.scriptFunction(
                    new Script("1.0 / (1.0 + Math.abs(Date.now() - doc['updated_at'].value) / 86400000)")
                ),
                0.1f  // 10% weight to freshness
            )
            .boostMode(FunctionScoreQueryBuilder.FunctionBoostMode.MULTIPLY)
            .scoreMode(FunctionScoreQueryBuilder.FunctionScoreMode.SUM);

        sourceBuilder.query(scoringQuery);

        // 5. Pagination
        int from = (searchQuery.getPage() - 1) * searchQuery.getPageSize();
        sourceBuilder.from(from);
        sourceBuilder.size(searchQuery.getPageSize() != 0 ?
            searchQuery.getPageSize() : DEFAULT_PAGE_SIZE);

        // 6. Facet aggregations
        addFacetAggregations(sourceBuilder);

        // 7. Highlight snippets
        HighlightBuilder highlighter = new HighlightBuilder()
            .field("name")
            .field("description")
            .preTags("<em>")
            .postTags("</em>");
        sourceBuilder.highlighter(highlighter);

        request.source(sourceBuilder);
        return request;
    }

    /**
     * Apply personalization boosts to query
     */
    private void applyPersonalizationBoosts(BoolQueryBuilder boolQuery, String userId) {
        try {
            // Fetch user search profile from Redis
            String profileKey = "user:" + userId + ":search_profile";
            Object profileObj = redisTemplate.opsForHash().getAll(profileKey);

            if (profileObj != null && !((Map) profileObj).isEmpty()) {
                Map<String, Object> profile = (Map<String, Object>) profileObj;

                // Boost favorite brands
                if (profile.containsKey("favorite_brands")) {
                    List<String> favoriteBrands = (List<String>) profile.get("favorite_brands");
                    for (String brand : favoriteBrands) {
                        boolQuery.should(QueryBuilders.termQuery("brand", brand.toLowerCase())
                            .boost(2.0f));
                    }
                }

                // Boost favorite categories
                if (profile.containsKey("favorite_categories")) {
                    List<String> favoriteCategories = (List<String>) profile.get("favorite_categories");
                    for (String category : favoriteCategories) {
                        boolQuery.should(QueryBuilders.termQuery("category", category.toLowerCase())
                            .boost(1.5f));
                    }
                }
            }

        } catch (Exception e) {
            log.warn("Failed to apply personalization boosts: {}", e.getMessage());
            // Fail gracefully: continue with non-personalized search
        }
    }

    /**
     * Add facet aggregations (brands, categories, price ranges)
     */
    private void addFacetAggregations(SearchSourceBuilder sourceBuilder) {
        // Brand facet
        sourceBuilder.aggregation(AggregationBuilders.terms("brands")
            .field("brand")
            .size(50)
        );

        // Category facet
        sourceBuilder.aggregation(AggregationBuilders.terms("categories")
            .field("category")
            .size(50)
        );

        // Price range facet
        sourceBuilder.aggregation(AggregationBuilders.histogram("price_ranges")
            .field("price")
            .fixedInterval(new HistogramInterval(100))
        );

        // Rating facet
        sourceBuilder.aggregation(AggregationBuilders.terms("ratings")
            .field("rating")
            .size(5)
        );
    }

    /**
     * Execute search with timeout protection
     */
    private SearchResults executeSearchWithTimeout(SearchRequest request,
                                                  int timeoutMs) throws IOException {
        try {
            SearchResponse response = elasticsearchClient.search(
                request,
                RequestOptions.DEFAULT
            );

            SearchResults results = new SearchResults();
            results.setTotalHits(response.getHits().getTotalHits().value);
            results.setPage(1);
            results.setPageSize(request.source().size());

            // Parse products
            List<ProductSearchResult> products = new ArrayList<>();
            for (SearchHit hit : response.getHits().getHits()) {
                Map<String, Object> source = hit.getSourceAsMap();
                ProductSearchResult product = new ProductSearchResult();
                product.setProductId(Long.parseLong((String) source.get("product_id")));
                product.setName((String) source.get("name"));
                product.setPrice(((Number) source.get("price")).doubleValue());
                product.setRating(((Number) source.get("rating")).floatValue());
                product.setImageUrl((String) source.get("image_url"));
                product.setRelevanceScore(hit.getScore());

                // Highlight snippets if available
                if (hit.getHighlightFields().containsKey("name")) {
                    product.setHighlight(hit.getHighlightFields()
                        .get("name").getFragments()[0].toString());
                }

                products.add(product);
            }

            results.setProducts(products);

            // Parse facets
            FacetBuilder facetBuilder = new FacetBuilder();
            results.setFacets(facetBuilder.buildFacets(response.getAggregations()));

            return results;

        } catch (IOException e) {
            log.error("Search execution failed", e);
            throw e;
        }
    }

    private String buildCacheKey(SearchQuery searchQuery, String userId) {
        String cacheKey = CACHE_KEY_PREFIX + searchQuery.getQuery() +
            ":" + (searchQuery.getFilters() != null ?
            searchQuery.getFilters().hashCode() : "none") +
            ":" + (userId != null ? userId : "anon");

        return DigestUtils.md5Hex(cacheKey);
    }

    private SearchResults getFromCache(String cacheKey) {
        try {
            String cached = (String) redisTemplate.opsForValue().get(cacheKey);
            if (cached != null) {
                return objectMapper.readValue(cached, SearchResults.class);
            }
        } catch (Exception e) {
            log.warn("Cache retrieval failed", e);
        }
        return null;
    }

    private void cacheResults(String cacheKey, SearchResults results) {
        try {
            String json = objectMapper.writeValueAsString(results);
            redisTemplate.opsForValue().set(cacheKey, json, Duration.ofHours(1));
        } catch (JsonProcessingException e) {
            log.warn("Failed to cache search results", e);
        }
    }

    private void personalizeResults(SearchResults results, String userId) {
        // Reorder results based on user preferences
        // (Simple personalization, more complex logic in separate service)
        // Placeholder for now
    }

    private void publishSearchAnalyticsEvent(SearchQuery searchQuery,
                                             SearchResults results, String userId) {
        try {
            SearchCompletedEvent event = SearchCompletedEvent.builder()
                .query(searchQuery.getQuery())
                .userId(userId)
                .resultCount(results.getTotalHits())
                .filters(searchQuery.getFilters())
                .duration(System.currentTimeMillis())
                .timestamp(Instant.now())
                .build();

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("search.query.completed",
                userId != null ? userId : "anon", eventJson);

        } catch (JsonProcessingException e) {
            log.error("Failed to publish search analytics", e);
        }
    }

    private void recordSearchMetrics(SearchQuery searchQuery,
                                     SearchResults results, long duration) {
        meterRegistry.counter("search.queries", "query", searchQuery.getQuery()).increment();
        meterRegistry.timer("search.duration").record(duration, TimeUnit.MILLISECONDS);
        meterRegistry.gauge("search.results.count", results.getTotalHits());
    }
}
```

### 2. Autocomplete Service

```java
@Service
@Slf4j
public class AutocompleteService {

    @Autowired private RestHighLevelClient elasticsearchClient;
    @Autowired private RedisTemplate<String, Object> redisTemplate;
    @Autowired private MeterRegistry meterRegistry;

    private static final int AUTOCOMPLETE_TIMEOUT_MS = 100;
    private static final int TOP_SUGGESTIONS = 10;

    /**
     * Get autocomplete suggestions (< 100ms)
     */
    public List<AutocompleteSuggestion> suggest(String prefix, String userId) {
        Timer.Sample sample = Timer.start(meterRegistry);

        try {
            if (prefix == null || prefix.length() < 2) {
                return Collections.emptyList();
            }

            String normalizedPrefix = prefix.toLowerCase().trim();

            // 1. Try Redis cache first (very fast)
            List<AutocompleteSuggestion> cached = getCachedSuggestions(normalizedPrefix);
            if (!cached.isEmpty()) {
                meterRegistry.counter("autocomplete.cache.hits").increment();
                sample.stop(meterRegistry.timer("autocomplete.cached"));
                return cached;
            }

            // 2. Query Elasticsearch with edge n-gram
            List<AutocompleteSuggestion> suggestions =
                queryElasticsearchAutocomplete(normalizedPrefix);

            // 3. Cache for future requests
            cacheSuggestions(normalizedPrefix, suggestions);

            // 4. Track popular queries
            trackQuery(normalizedPrefix);

            sample.stop(meterRegistry.timer("autocomplete.success"));
            return suggestions;

        } catch (Exception e) {
            log.error("Autocomplete failed for prefix: {}", prefix, e);
            sample.stop(meterRegistry.timer("autocomplete.error"));

            // Fallback: return empty list (better than error)
            return Collections.emptyList();
        }
    }

    /**
     * Get suggestions from Redis cache
     */
    private List<AutocompleteSuggestion> getCachedSuggestions(String prefix) {
        try {
            String cacheKey = "autocomplete:" + prefix + ":" +
                LocalDate.now().toString();

            Set<Object> cachedSuggestions = redisTemplate.opsForZSet()
                .reverseRange(cacheKey, 0, TOP_SUGGESTIONS - 1);

            if (cachedSuggestions != null && !cachedSuggestions.isEmpty()) {
                List<AutocompleteSuggestion> suggestions = new ArrayList<>();

                for (Object item : cachedSuggestions) {
                    String[] parts = item.toString().split(":");
                    suggestions.add(AutocompleteSuggestion.builder()
                        .text(parts[0])
                        .frequency(Integer.parseInt(parts[1]))
                        .source("CACHE")
                        .build());
                }

                return suggestions;
            }

        } catch (Exception e) {
            log.warn("Cache retrieval failed", e);
        }

        return Collections.emptyList();
    }

    /**
     * Query Elasticsearch for autocomplete with edge n-gram
     */
    private List<AutocompleteSuggestion> queryElasticsearchAutocomplete(
            String prefix) throws IOException {

        SearchRequest request = new SearchRequest("products");
        SearchSourceBuilder sourceBuilder = new SearchSourceBuilder();

        // Match on edge n-gram field (prefix match)
        sourceBuilder.query(QueryBuilders.matchQuery(
            "name.autocomplete", prefix
        ).operator(Operator.AND));

        // Aggregate to get top product names
        sourceBuilder.aggregation(AggregationBuilders.terms("suggestions")
            .field("name.keyword")
            .size(TOP_SUGGESTIONS)
            .order(BucketOrder.aggregation("_count", false))
        );

        request.source(sourceBuilder);

        SearchResponse response = elasticsearchClient.search(
            request, RequestOptions.DEFAULT
        );

        List<AutocompleteSuggestion> suggestions = new ArrayList<>();

        Terms aggregation = response.getAggregations().get("suggestions");
        for (Terms.Bucket bucket : aggregation.getBuckets()) {
            suggestions.add(AutocompleteSuggestion.builder()
                .text(bucket.getKeyAsString())
                .frequency((int) bucket.getDocCount())
                .source("ELASTICSEARCH")
                .build());
        }

        return suggestions;
    }

    /**
     * Cache autocomplete suggestions in Redis
     */
    private void cacheSuggestions(String prefix,
                                 List<AutocompleteSuggestion> suggestions) {
        try {
            String cacheKey = "autocomplete:" + prefix + ":" +
                LocalDate.now().toString();

            for (int i = 0; i < suggestions.size(); i++) {
                AutocompleteSuggestion suggestion = suggestions.get(i);
                String member = suggestion.getText() + ":" + suggestion.getFrequency();

                // Score by frequency (descending)
                redisTemplate.opsForZSet().add(cacheKey, member,
                    TOP_SUGGESTIONS - i);
            }

            // Set expiration (1 day)
            redisTemplate.expire(cacheKey, Duration.ofDays(1));

        } catch (Exception e) {
            log.warn("Failed to cache suggestions", e);
        }
    }

    /**
     * Track query for trending/popular analysis
     */
    private void trackQuery(String query) {
        try {
            String trendingKey = "trending:queries:" +
                LocalDate.now().toString();

            redisTemplate.opsForZSet().incrementScore(trendingKey, query, 1);
            redisTemplate.expire(trendingKey, Duration.ofDays(1));

        } catch (Exception e) {
            log.warn("Failed to track query", e);
        }
    }
}

@Data
@Builder
class AutocompleteSuggestion {
    private String text;
    private int frequency;
    private String source; // CACHE or ELASTICSEARCH
}
```

### 3. Search Indexing Consumer (Kafka)

```java
@Service
@Slf4j
public class SearchIndexingConsumer {

    @Autowired private RestHighLevelClient elasticsearchClient;
    @Autowired private MeterRegistry meterRegistry;
    @Autowired private ObjectMapper objectMapper;

    private static final int BULK_SIZE = 5000;
    private static final String INDEX_NAME = "products";
    private BulkProcessor bulkProcessor;

    @PostConstruct
    public void initBulkProcessor() {
        BulkProcessor.Listener listener = new BulkProcessor.Listener() {
            @Override
            public void beforeBulk(long executionId, BulkRequest request) {
                log.debug("Bulk request #{} starting with {} actions",
                    executionId, request.numberOfActions());
            }

            @Override
            public void afterBulk(long executionId, BulkRequest request,
                                 BulkResponse response) {
                if (response.hasFailures()) {
                    log.error("Bulk request #{} had failures: {}",
                        executionId, response.buildFailureMessage());

                    // Track failed operations
                    meterRegistry.counter("search.indexing.bulk.failures")
                        .increment(response.buildFailureMessage().split("\n").length);
                } else {
                    log.info("Bulk request #{} completed successfully with {} items",
                        executionId, request.numberOfActions());

                    meterRegistry.counter("search.indexing.bulk.success")
                        .increment(request.numberOfActions());
                }
            }

            @Override
            public void afterBulk(long executionId, BulkRequest request, Throwable failure) {
                log.error("Bulk request #{} failed with exception", executionId, failure);
                meterRegistry.counter("search.indexing.bulk.errors").increment();
            }
        };

        bulkProcessor = BulkProcessor.builder(
                (request, bulkListener) -> elasticsearchClient.bulkAsync(
                    request, RequestOptions.DEFAULT, bulkListener
                ),
                listener
            )
            .setBulkActions(BULK_SIZE)
            .setBulkSize(new ByteSizeValue(100, ByteSizeUnit.MB))
            .setFlushInterval(TimeValue.timeValueSeconds(1))
            .setConcurrentRequests(5)
            .setBackoffPolicy(BackoffPolicy.exponentialBackoff(
                TimeValue.timeValueMillis(100), 3
            ))
            .build();
    }

    /**
     * Consume catalog.updated events and index products
     */
    @KafkaListener(topics = "catalog.updated", groupId = "search-indexing-service")
    public void indexProduct(ConsumerRecord<String, String> record) {
        try {
            CatalogUpdateEvent event = objectMapper.readValue(
                record.value(), CatalogUpdateEvent.class
            );

            log.debug("Indexing product {} ({})", event.getProductId(),
                event.getChangeType());

            // 1. Build document from event
            Map<String, Object> productDocument = buildProductDocument(event);

            // 2. Handle based on change type
            switch (event.getChangeType()) {
                case "CREATE":
                case "UPDATE":
                    // Index (create or update)
                    IndexRequest indexRequest = new IndexRequest(INDEX_NAME)
                        .id(String.valueOf(event.getProductId()))
                        .source(productDocument, XContentType.JSON);

                    bulkProcessor.add(indexRequest);
                    break;

                case "DELETE":
                    // Delete
                    DeleteRequest deleteRequest = new DeleteRequest(INDEX_NAME)
                        .id(String.valueOf(event.getProductId()));

                    bulkProcessor.add(deleteRequest);
                    break;
            }

            meterRegistry.counter("search.indexing.products", "type",
                event.getChangeType()).increment();

        } catch (JsonProcessingException e) {
            log.error("Failed to parse catalog event", e);
            meterRegistry.counter("search.indexing.parse.errors").increment();
        }
    }

    /**
     * Build Elasticsearch document from catalog event
     */
    private Map<String, Object> buildProductDocument(CatalogUpdateEvent event) {
        Map<String, Object> document = new LinkedHashMap<>();

        document.put("product_id", String.valueOf(event.getProductId()));
        document.put("sku", event.getChangeFields().get("sku"));
        document.put("name", event.getChangeFields().get("name"));
        document.put("description", event.getChangeFields().get("description"));
        document.put("category", event.getChangeFields().get("category"));
        document.put("brand", event.getChangeFields().get("brand"));
        document.put("price", event.getChangeFields().get("price"));
        document.put("rating", event.getChangeFields().get("rating"));
        document.put("review_count", event.getChangeFields().get("review_count"));
        document.put("inventory", event.getChangeFields().get("inventory"));
        document.put("in_stock",
            ((Number) event.getChangeFields().get("inventory")).intValue() > 0);
        document.put("image_url", event.getChangeFields().get("image_url"));
        document.put("popularity_score", event.getChangeFields().get("popularity_score"));
        document.put("updated_at", event.getTimestamp());

        return document;
    }

    @PreDestroy
    public void shutdown() {
        try {
            bulkProcessor.awaitClose(30, TimeUnit.SECONDS);
            log.info("Bulk processor closed gracefully");
        } catch (InterruptedException e) {
            log.error("Bulk processor shutdown interrupted", e);
            Thread.currentThread().interrupt();
        }
    }
}

@Data
class CatalogUpdateEvent {
    private String eventId;
    private Long productId;
    private String changeType; // CREATE, UPDATE, DELETE
    private Map<String, Object> changeFields;
    private Instant timestamp;
}
```

---

## Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| **Elasticsearch Cluster Down** | All searches fail | Circuit breaker → fallback to PostgreSQL full-text search (slower) |
| **Network Latency to ES > 200ms** | Searches timeout, poor UX | Cache hot queries, partial results timeout fallback |
| **Index Sync Lag > 5 minutes** | Stale search results | Alert if lag exceeds threshold, force manual reindex |
| **Cache (Redis) Down** | Cache misses, higher ES load | Graceful degradation: increase ES cache, circuit breaker to DB |
| **Ranking Algorithm Bug** | Irrelevant results, revenue loss | Feature flags for A/B testing, quick rollback capability |
| **Analyzer Change Breaks Queries** | Search stops working | Version control analyzer config, test before deployment |
| **Memory Exhaustion on ES** | Node crashes, shard unassignment | Monitoring heap usage, aggressive cleanup of old indices |
| **Bulk Indexing Lag > 1 minute** | New products not searchable | Manual trigger reindex, prioritize hot products |

---

## Scaling Strategy

### Horizontal Scaling

**Search Service:**
- Scale: 1 instance per 10,000 searches/sec (50k = 5 instances minimum)
- Load balancer: Sticky sessions for autocomplete requests
- Caching: Local HTTP cache + distributed Redis cache

**Elasticsearch:**
- Shards: 10 (each shard ~50GB)
- Replicas: 2-3 per shard (HA + read distribution)
- Nodes: 20+ nodes for 500GB index
- JVM: 31GB heap per node (max safe heap)

**Search Indexing Service:**
- Instances: 2-3 for fault tolerance
- Partitions: 100 in Kafka for parallel processing
- Bulk batch size: 5,000 documents per request

### Vertical Scaling

**Elasticsearch:**
- Node memory: 64GB total (31GB JVM heap)
- CPU: 8+ cores per node
- Storage: SSD for index shards (NVMe preferred)

**Redis:**
- For cache: 16GB+ for 100MB autocomplete + result caches
- For analytics: Separate Redis cluster for trending

---

## Monitoring & Observability

### Key Metrics

```
# Search Performance
search.queries (counter, by query term)
search.duration (histogram, percentiles)
search.results.count (gauge)
search.cache.hits (counter)
search.cache.miss (counter)

# Autocomplete
autocomplete.requests (counter)
autocomplete.latency (histogram)
autocomplete.cache.hits (counter)

# Indexing
search.indexing.bulk.success (counter)
search.indexing.bulk.failures (counter)
search.indexing.lag_seconds (gauge)
search.index.size (gauge)

# Elasticsearch Health
es.cluster.health (gauge: green=1, yellow=0.5, red=0)
es.node.heap_usage_percent (gauge)
es.shard.count (gauge)
es.search.latency (histogram)
es.indexing.rate (counter)

# Quality
search.top_5_precision (gauge)
search.result.click_through_rate (gauge)
search.conversion_rate (gauge)
```

### Dashboards

1. **Real-Time Search Dashboard**
   - Queries per minute
   - Search latency (p50, p95, p99)
   - Cache hit rate
   - Top searches trending

2. **Elasticsearch Health Dashboard**
   - Cluster status (green/yellow/red)
   - Node heap usage
   - Shard distribution
   - Index size and document count

3. **Search Quality Dashboard**
   - Click-through rate by query
   - Conversion rate by search
   - Precision of top results
   - User satisfaction (CSAT) if available

---

## Summary Cheat Sheet

### Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| **Elasticsearch** | Industry standard, feature-rich, proven at scale |
| **Two-level caching** | Hot queries in Redis (10ms), others in ES (150ms) |
| **Edge n-gram analyzer** | Fast prefix matching, handles typos |
| **BM25 + popularity** | Balanced relevance + trending products |
| **Async indexing** | Doesn't block catalog updates, eventual consistency |
| **Blue-green indexing** | Zero-downtime index updates, easy rollback |

### Latency Budget (200ms total)

| Component | Budget | Allocation |
|-----------|--------|-----------|
| Load balancer routing | 5ms | Network + LB overhead |
| Redis cache check | 10ms | For cached queries |
| Elasticsearch query | 150ms | Query execution + aggregations |
| Result formatting | 20ms | Pagination, highlight, serialization |
| Reserve | 15ms | Network jitter, buffer |

### Search Query Optimization Tips

```
1. Query Complexity: Avoid deeply nested bool queries
   → Limit must/should clauses to < 10

2. Facet Aggregations: Expensive on large result sets
   → Pre-compute facets, cache results

3. Pagination Depth: Avoid large offset values
   → Use search_after (scroll) for deep pagination

4. Ranking Computation: Custom scoring is expensive
   → Use pre-computed popularity_score field

5. Filter Ordering: Fast filters before expensive queries
   → term filters before range → match queries
```

### Cost Optimization

- **Index compression:** Use best_compression codec (reduces size by 30%)
- **Shard sizing:** 40-50GB per shard (optimal balance)
- **Replication:** Min 1 replica for HA (2 is overkill for most)
- **Retention:** Archive old indices (> 90 days) to cold storage

### Tech Stack Justification

- **Java 17:** Spring Data Elasticsearch client, strong typing
- **Spring Boot 3:** Embedded Elasticsearch test client, auto-configuration
- **PostgreSQL:** Fallback full-text search, product catalog master
- **MongoDB:** Search analytics (flexible schema)
- **Redis:** Query result caching, autocomplete cache
- **Kafka:** Decoupled indexing, replay capability
- **Elasticsearch:** Feature completeness, proven search engine
