---
title: Product Recommendation Engine
layout: default
---

# Product Recommendation Engine — Deep Dive Design

> **Scenario:** Build a real-time product recommendation system that shows:
> - "Customers who bought this also bought" (collaborative filtering)
> - "Frequently bought together" (association rules)
> - "Recommended for you" (personalized based on browsing history)
> - Must return recommendations in <100ms
> - System must learn from 1 million daily transactions
> - Handle cold start problem (new users, new products)
>
> **Tech Stack:** Java 17, Spring Boot 3, PostgreSQL, MongoDB, Redis, Kafka

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
- Track user behavior (views, clicks, purchases, wishlist adds)
- Generate 4 recommendation types:
  1. Collaborative filtering (item-item similarity)
  2. Association rules (frequently bought together)
  3. Personalized recommendations (user browsing history)
  4. Popularity-based fallback (cold start)
- Return recommendations in <100ms (p95)
- Support A/B testing of algorithms
- Explain why item is recommended (explainability)
- Handle new products and new users gracefully

### Non-Functional Requirements
- Latency: <100ms p95 for recommendation API
- Throughput: 1M daily transactions = ~12 TPS average, ~100 TPS peak
- Availability: 99.5% for recommendation service
- Consistency: Eventually consistent (stale data acceptable)
- Data retention: 2 years of user events
- Model update frequency: Daily batch + hourly incremental

---

## Capacity Estimation

| Metric | Calculation | Value |
|--------|-----------|-------|
| Daily transactions | 1M | 1,000,000 |
| Avg TPS | 1M / 86,400 sec | ~12 |
| Peak TPS (10x) | 12 × 10 | ~120 |
| Products in catalog | Typical e-commerce | 100,000 |
| Active users (30 days) | 1M daily / 3 churn | ~300,000 |
| User event records (2 years) | 1M × 365 × 2 | 730M events |
| Item-item similarity matrix | 100K × 100K × 8 bytes | ~80GB (sparse, ~5% density → ~4GB) |
| User vectors (embeddings) | 300K users × 128 dims × 4 bytes | ~150MB |
| Redis recommendation cache | 100K products × 10 recs × 20 bytes | ~20MB (with TTL 1h, ~10-20 copies) |
| Kafka event topic volume/day | 1M events × 200 bytes | ~200GB/day |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        User/Client                               │
│                    (Web/Mobile/App)                              │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼────┐    ┌─────▼─────┐  ┌────▼─────┐
    │  Product│    │ Shopping  │  │  User    │
    │ Catalog │    │   Cart    │  │ Profile  │
    │         │    │           │  │          │
    └────┬────┘    └─────┬─────┘  └────┬─────┘
         │               │             │
         └───────────────┼─────────────┘
                         │ (events: view, click, purchase)
                         ▼
         ┌───────────────────────────────────┐
         │   UserBehaviorService             │
         │   (event tracking, enrichment)    │
         └────────────────┬──────────────────┘
                          │
                          ▼
         ┌───────────────────────────────────┐
         │   Kafka Topic: user-events        │
         │   (1M msgs/day, TTL 7 days)       │
         └──┬─────────────┬──────────┬───────┘
            │             │          │
      ┌─────▼──────┐ ┌────▼──────┐ ┌▼──────────────┐
      │ RecommendationService    │ │ Analytics &   │
      │ (real-time via cache)    │ │ ML Training   │
      └──────┬──────┘ └─────┬────┘ └───────────────┘
             │              │
        ┌────▼──────────────▼──────┐
        │   ModelTrainingPipeline  │
        │   (batch daily, hourly   │
        │    incremental updates)  │
        └────┬──────────┬──────────┘
             │          │
    ┌────────▼──┐  ┌────▼─────────┐
    │  MongoDB  │  │   Redis      │
    │  (models, │  │  (precomputed│
    │   rules)  │  │   recs, CF   │
    │           │  │   matrix)    │
    └───────────┘  └──────────────┘
             │
        ┌────▼──────────────────┐
        │  RecommendationAPI    │
        │  (read cache, <100ms) │
        └───────────────────────┘
```

---

## Core Design Questions Answered

### 1. What microservices are needed?

| Service | Responsibility | Tech |
|---------|-----------------|------|
| **UserBehaviorService** | Capture views, clicks, purchases, wishlist events | Spring Boot, Kafka Producer |
| **RecommendationService** | Serve recommendations from cache, fallback to DB | Spring Boot, Redis, MongoDB |
| **ModelTrainingService** | Batch compute CF matrix, association rules, embeddings | Spark/Python, Kafka Consumer, MongoDB |
| **ColdStartService** | Generate recs for new users/products (popularity, content-based) | Spring Boot, PostgreSQL |
| **ABTestService** | Manage A/B tests, assign users to variants, track metrics | Spring Boot, PostgreSQL |
| **RecommendationAPIGateway** | Rate limiting, caching, request routing | Spring Cloud Gateway |

### 2. How do you train and serve ML models?

**Training Pipeline:**
- **Daily Batch:** Compute item-item similarity using cosine distance on user-product purchase matrix
- **Hourly Incremental:** Update association rules from last hour's transactions
- **Real-time Serving:** Load precomputed models into Redis, cache user personalization in Postgres/Redis

**Model Types:**
- **Collaborative Filtering:** Item-item similarity (computed offline, stored in Redis sorted sets)
- **Association Rules:** Apriori/FP-Growth for "frequently bought together"
- **Content-based:** Product metadata + category embeddings for new items
- **Popularity:** Simple frequency counts updated hourly

### 3. Where do you store user behavior data?

| System | Data | Purpose | TTL |
|--------|------|---------|-----|
| **Kafka** | Raw user events (view, click, purchase) | Event sourcing, real-time processing | 7 days |
| **PostgreSQL** | User interaction summaries, A/B test assignments | Fast queries, ACID transactions | Permanent |
| **MongoDB** | Training datasets, ML model metadata, association rules | Flexible schema, large documents | Permanent |
| **Redis** | Precomputed recommendations, user vectors, CF matrix | Sub-100ms lookups | 1 hour (recommendations), 24h (vectors) |

### 4. How do you update recommendations in real-time?

1. **Cache-First Strategy:** User requests hit Redis cache (95%+ hit rate)
2. **TTL-Based Refresh:** Recommendations expire hourly; re-train from last 24h events
3. **Event-Driven Updates:**
   - Purchase event → Kafka → update user recent purchases in Redis
   - Update association rules cache in background
4. **Async Model Update:**
   - Kafka Stream processes hourly windows of events
   - Computes deltas (new associations, changed popularity scores)
   - Updates MongoDB with new models
   - Redis cache invalidated via event notification

### 5. How do you handle the cold start problem?

| Scenario | Solution |
|----------|----------|
| **New User** | Return trending/popular products in category; ask for preferences; use collaborative filtering fallback |
| **New Product** | Content-based filtering (category, tags, price, reviews); show to users with similar preferences; give slight boost in popularity-based recs |
| **New Category** | Bootstrap with manual rules; monitor engagement; switch to ML-based when sufficient data |
| **Inactive User (>90 days)** | Blend trending products (50%) + user historical preferences (30%) + popular items (20%) |

**Implementation:**
```java
// Fallback strategy in RecommendationService
List<Product> recommendations = new ArrayList<>();

// Try user-personalized first
if (user.getLastActivityWithin(Duration.ofDays(30))) {
    recommendations = getPersonalizedRecs(userId);
}

// Fill gaps with popularity or content-based
if (recommendations.size() < BATCH_SIZE) {
    recommendations.addAll(
        getPopularityBasedRecs(user.getCategory(),
            BATCH_SIZE - recommendations.size())
    );
}

// Last resort: trending in category
if (recommendations.isEmpty()) {
    recommendations = getTrendingInCategory(user.getCategory());
}
```

### 6. How do you A/B test different recommendation algorithms?

1. **Experiment Framework:**
   - Store user → variant mapping in PostgreSQL
   - Hash userId to deterministic variant (no flicker)
   - Track metrics per variant: CTR, conversion, AOV

2. **Feature Flags:** Use Spring Cloud Config to enable/disable algorithms per region

3. **Metrics Tracking:**
   - Event: `recommendation-shown` (algo, variant, products)
   - Event: `recommendation-clicked` (algo, variant, product-id)
   - Compute CTR, conversion rate, revenue per variant daily

4. **Statistical Significance:** Query PostgreSQL analytics table for p-value testing

---

## Microservices Breakdown

### UserBehaviorService
```
POST /api/users/{userId}/events
{
  "type": "PRODUCT_VIEW|CLICK|PURCHASE|WISHLIST_ADD",
  "productId": "prod-123",
  "category": "electronics",
  "price": 299.99,
  "timestamp": 1711939200000
}
Response: 202 Accepted (async)
```

### RecommendationService
```
GET /api/recommendations?userId={userId}&count=10&type=COLLABORATIVE_FILTERING
Response:
{
  "recommendations": [
    {
      "productId": "prod-456",
      "score": 0.95,
      "type": "COLLABORATIVE_FILTERING",
      "reason": "Users who bought item-1 also bought this",
      "cachedAt": 1711939000000
    }
  ],
  "timestamp": 1711939200000
}
```

### ModelTrainingService
- Scheduled daily at 2 AM UTC
- Reads 24h of events from Kafka
- Computes item-item similarity (Cosine distance)
- Stores in MongoDB, pushes hot items to Redis
- Publishes `model-trained` event to Kafka

### ColdStartService
- Triggered when user has < 5 interactions
- Returns top 20 products by category popularity
- Ranks by (trend score, rating, #reviews)

### ABTestService
- Manages test variants
- Assigns users deterministically (hash(userId) % num_variants)
- Tracks impressions and conversions
- Reports statistical significance

---

## Database Design

### PostgreSQL Schema

```sql
-- User interactions summary (for quick lookups)
CREATE TABLE user_interactions (
    user_id BIGINT PRIMARY KEY,
    last_activity_at TIMESTAMP NOT NULL,
    total_purchases INT DEFAULT 0,
    total_views INT DEFAULT 0,
    last_purchased_categories TEXT[], -- array of category IDs
    preferred_price_range DECIMAL[2], -- [min, max]
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_last_activity (last_activity_at)
);

-- A/B test assignments
CREATE TABLE ab_test_assignments (
    id BIGSERIAL PRIMARY KEY,
    test_id VARCHAR(100) NOT NULL,
    user_id BIGINT NOT NULL,
    variant VARCHAR(50) NOT NULL,
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(test_id, user_id),
    INDEX idx_test_user (test_id, user_id)
);

-- A/B test events
CREATE TABLE ab_test_events (
    id BIGSERIAL PRIMARY KEY,
    test_id VARCHAR(100) NOT NULL,
    user_id BIGINT NOT NULL,
    variant VARCHAR(50) NOT NULL,
    event_type VARCHAR(50) NOT NULL, -- IMPRESSION, CLICK, CONVERSION
    product_id BIGINT,
    event_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_test_date (test_id, event_at)
);

-- User cold start signals (for new users)
CREATE TABLE user_preferences (
    user_id BIGINT PRIMARY KEY,
    preferred_categories TEXT[],
    brand_preferences TEXT[],
    price_sensitivity VARCHAR(20), -- BUDGET, NORMAL, PREMIUM
    last_survey_at TIMESTAMP,
    FOREIGN KEY (user_id) REFERENCES user_interactions(user_id)
);
```

### MongoDB Collections

```javascript
// Association rules collection (Apriori output)
db.association_rules.createIndex({
    "antecedent_ids": 1,
    "support": -1
});

db.association_rules.insertOne({
    _id: ObjectId(),
    antecedent_ids: ["prod-123", "prod-124"], // items in basket
    consequent_id: "prod-456",                  // frequently bought with
    support: 0.025,    // 2.5% of baskets contain all 3
    confidence: 0.45,  // 45% of baskets with ant also have cons
    lift: 1.8,         // 1.8x more likely than random
    count: 2500,       // number of occurrences
    updated_at: ISODate("2024-01-15T00:00:00Z")
});

// ML model metadata
db.models.insertOne({
    _id: "cf-model-2024-01-15",
    type: "COLLABORATIVE_FILTERING",
    version: 1,
    training_date: ISODate("2024-01-15T02:00:00Z"),
    data_points: 1000000,
    quality_metrics: {
        precision_at_10: 0.42,
        recall_at_10: 0.38,
        ndcg: 0.65
    },
    expiry: ISODate("2024-01-22T02:00:00Z"),
    redis_key: "cf-matrix:2024-01-15"
});

// User behavior events (event log, alternative to Kafka sink)
db.user_events.createIndex({ "user_id": 1, "timestamp": -1 });
db.user_events.insertOne({
    _id: ObjectId(),
    user_id: 12345,
    event_type: "PURCHASE",
    product_id: "prod-789",
    category: "electronics",
    price: 299.99,
    session_id: "sess-abc123",
    timestamp: ISODate("2024-01-15T10:30:00Z")
});
```

---

## Redis Data Structures

### Precomputed Recommendations
```
KEY: rec:{userId}:{type}
TYPE: List
TTL: 1 hour
VALUE:
[
  "prod-456:0.95:COLLABORATIVE_FILTERING:Users who bought item-1 also bought this",
  "prod-789:0.87:ASSOCIATION_RULES:Frequently bought together",
  ...
]

Example Redis command:
SETEX rec:user-12345:COLLAB 3600
  "prod-456:0.95:COLLABORATIVE_FILTERING:Customers also bought"
```

### Item-Item Similarity Matrix
```
KEY: cf-matrix:{modelDate}:{itemId}
TYPE: Sorted Set (score = similarity, member = productId)
TTL: 24 hours
VALUE:
{
  "prod-789": 0.95,      # 95% similar items
  "prod-101": 0.87,
  "prod-102": 0.76,
  ...
}

Example Redis command:
ZADD cf-matrix:2024-01-15:prod-123 0.95 prod-789 0.87 prod-101 0.76 prod-102
```

### User Recent Purchases (for personalization)
```
KEY: user:purchases:{userId}
TYPE: List (capped at 20 items)
TTL: 30 days
VALUE:
[
  "prod-456:timestamp:price",
  "prod-789:timestamp:price",
  ...
]

Example:
LPUSH user:purchases:user-12345 "prod-456:1711939200000:299.99"
LTRIM user:purchases:user-12345 0 19
EXPIRE user:purchases:user-12345 2592000  // 30 days
```

### Association Rules Cache
```
KEY: assoc-rules:{antecedent1}:{antecedent2}
TYPE: Hash (field=consequent_id, value=score)
TTL: 6 hours
VALUE:
{
  "prod-456": 0.85,     # lift-weighted score
  "prod-789": 0.72
}
```

### User Vector (embeddings)
```
KEY: user-vec:{userId}
TYPE: String (serialized float[128])
TTL: 24 hours
VALUE: Base64-encoded 128-dim float vector
```

---

## Kafka Event Flow

### Topic: `user-events`
```
Partition: userId % num_partitions (ensures user events ordered)
Retention: 7 days
Replication: 3

Message:
{
  "userId": 12345,
  "eventType": "PRODUCT_PURCHASE|VIEW|CLICK|WISHLIST_ADD",
  "productId": "prod-456",
  "category": "electronics",
  "price": 299.99,
  "sessionId": "sess-abc123",
  "timestamp": 1711939200000,
  "ip": "192.168.1.1",
  "userAgent": "Mozilla/5.0...",
  "source": "WEB|MOBILE|APP"
}
```

### Topic: `model-update-signals`
```
Message (hourly):
{
  "signalType": "ASSOCIATION_RULES|POPULARITY|CF_MATRIX",
  "modelDate": "2024-01-15",
  "version": 1,
  "redisKey": "cf-matrix:2024-01-15",
  "mongoId": "model-2024-01-15",
  "affectedItems": ["prod-456", "prod-789"],
  "timestamp": 1711939200000
}
```

### Topic: `recommendation-impressions`
```
Message (user event):
{
  "userId": 12345,
  "recommendationId": "rec-123",
  "algorithm": "COLLABORATIVE_FILTERING",
  "products": ["prod-456", "prod-789"],
  "impressionTime": 1711939200000
}

Consumer: ABTestService (updates impressions table)
```

---

## Implementation Code

### 1. UserBehaviorService - Event Producer

```java
package com.recommendation.service;

import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;

@Slf4j
@Service
public class UserBehaviorService {

    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;
    private final UserInteractionRepository userInteractionRepo;

    public UserBehaviorService(
            KafkaTemplate<String, String> kafkaTemplate,
            ObjectMapper objectMapper,
            UserInteractionRepository userInteractionRepo) {
        this.kafkaTemplate = kafkaTemplate;
        this.objectMapper = objectMapper;
        this.userInteractionRepo = userInteractionRepo;
    }

    public void trackUserEvent(UserEventDto event) {
        try {
            // Enrich event
            UserEvent enrichedEvent = UserEvent.builder()
                .userId(event.getUserId())
                .eventType(event.getEventType())
                .productId(event.getProductId())
                .category(event.getCategory())
                .price(event.getPrice())
                .sessionId(event.getSessionId())
                .timestamp(System.currentTimeMillis())
                .source(event.getSource())
                .userAgent(event.getUserAgent())
                .ip(event.getIp())
                .build();

            // Async send to Kafka
            String partitionKey = "user-" + event.getUserId();
            kafkaTemplate.send(
                "user-events",
                partitionKey,
                objectMapper.writeValueAsString(enrichedEvent)
            ).whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish user event: {}", event, ex);
                } else {
                    log.debug("Published event for user: {}", event.getUserId());
                }
            });

            // Update user interaction summary (async)
            updateUserInteractionSummary(event.getUserId());

        } catch (Exception e) {
            log.error("Error tracking user event", e);
            // Don't fail user request if tracking fails
        }
    }

    private void updateUserInteractionSummary(Long userId) {
        userInteractionRepo.findById(userId).ifPresentOrElse(
            interaction -> {
                interaction.setLastActivityAt(new Date());
                interaction.setTotalViews(interaction.getTotalViews() + 1);
                userInteractionRepo.save(interaction);
            },
            () -> {
                UserInteraction newInteraction = UserInteraction.builder()
                    .userId(userId)
                    .lastActivityAt(new Date())
                    .totalViews(1)
                    .totalPurchases(0)
                    .build();
                userInteractionRepo.save(newInteraction);
            }
        );
    }

    public void trackPurchase(Long userId, List<OrderLineItem> items) {
        for (OrderLineItem item : items) {
            UserEventDto purchaseEvent = UserEventDto.builder()
                .userId(userId)
                .eventType("PURCHASE")
                .productId(item.getProductId())
                .category(item.getCategory())
                .price(item.getPrice())
                .sessionId(UUID.randomUUID().toString())
                .build();

            trackUserEvent(purchaseEvent);
        }

        // Update purchase count
        userInteractionRepo.findById(userId).ifPresent(interaction -> {
            interaction.setTotalPurchases(interaction.getTotalPurchases() + 1);
            userInteractionRepo.save(interaction);
        });
    }
}

@Data
@Builder
class UserEvent {
    private Long userId;
    private String eventType;
    private String productId;
    private String category;
    private BigDecimal price;
    private String sessionId;
    private Long timestamp;
    private String source;
    private String userAgent;
    private String ip;
}
```

### 2. RecommendationService - Cache-First Strategy

```java
package com.recommendation.service;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.beans.factory.annotation.Value;
import com.mongodb.client.MongoCollection;
import org.bson.Document;
import lombok.extern.slf4j.Slf4j;
import java.util.*;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

@Slf4j
@Service
public class RecommendationService {

    private final RedisTemplate<String, String> redisTemplate;
    private final MongoCollection<Document> associationRulesCollection;
    private final ColdStartService coldStartService;
    private final UserInteractionRepository userInteractionRepo;
    private final ProductRepository productRepository;

    @Value("${recommendation.cache-ttl-minutes:60}")
    private int cacheTTLMinutes;

    @Value("${recommendation.batch-size:10}")
    private int defaultBatchSize;

    public RecommendationService(
            RedisTemplate<String, String> redisTemplate,
            MongoCollection<Document> associationRulesCollection,
            ColdStartService coldStartService,
            UserInteractionRepository userInteractionRepo,
            ProductRepository productRepository) {
        this.redisTemplate = redisTemplate;
        this.associationRulesCollection = associationRulesCollection;
        this.coldStartService = coldStartService;
        this.userInteractionRepo = userInteractionRepo;
        this.productRepository = productRepository;
    }

    public RecommendationResponse getRecommendations(
            Long userId,
            int count,
            RecommendationType type) {

        long startTime = System.currentTimeMillis();
        RecommendationResponse response = new RecommendationResponse();
        response.setRequestTime(startTime);

        try {
            // 1. Check Redis cache first (95%+ hit rate)
            String cacheKey = buildCacheKey(userId, type);
            List<RecommendationItem> cached = getFromCache(cacheKey);

            if (cached != null && !cached.isEmpty()) {
                log.debug("Cache hit for user {} type {}", userId, type);
                response.setRecommendations(cached.stream()
                    .limit(count)
                    .collect(Collectors.toList()));
                response.setSource("CACHE");
                recordLatency(System.currentTimeMillis() - startTime, "cache_hit");
                return response;
            }

            // 2. Check user interaction history
            UserInteraction interaction = userInteractionRepo.findById(userId).orElse(null);
            List<RecommendationItem> recommendations = new ArrayList<>();

            if (interaction != null && interaction.getTotalPurchases() > 5) {
                // User is warm - use personalized recommendations
                recommendations = getPersonalizedRecommendations(userId, type, count);
            } else {
                // User is cold - use fallback strategies
                recommendations = coldStartService.generateRecommendations(userId, type, count);
            }

            // 3. Cache results
            cacheRecommendations(cacheKey, recommendations);

            response.setRecommendations(recommendations.stream()
                .limit(count)
                .collect(Collectors.toList()));
            response.setSource("GENERATED");
            response.setCachedAt(System.currentTimeMillis());

            recordLatency(System.currentTimeMillis() - startTime, "full_compute");

            return response;

        } catch (Exception e) {
            log.error("Error generating recommendations for user {}", userId, e);
            // Fallback: return empty or cached data
            response.setRecommendations(List.of());
            response.setError(e.getMessage());
            recordLatency(System.currentTimeMillis() - startTime, "error");
            return response;
        }
    }

    private List<RecommendationItem> getPersonalizedRecommendations(
            Long userId,
            RecommendationType type,
            int count) {

        List<RecommendationItem> results = new ArrayList<>();

        // Get user's recent purchases
        UserInteraction interaction = userInteractionRepo.findById(userId).orElse(null);
        if (interaction == null) return results;

        @SuppressWarnings("unchecked")
        List<String> recentPurchaseIds = (List<String>) redisTemplate
            .opsForList()
            .range("user:purchases:" + userId, 0, 4);

        if (recentPurchaseIds == null || recentPurchaseIds.isEmpty()) {
            return results;
        }

        Set<String> seenProducts = new HashSet<>(recentPurchaseIds);

        // Collaborative filtering: find similar items
        if (type == RecommendationType.COLLABORATIVE_FILTERING ||
            type == RecommendationType.ALL) {

            for (String productId : recentPurchaseIds) {
                String matrixKey = "cf-matrix:2024-01-15:" + productId;

                @SuppressWarnings("unchecked")
                Set<org.springframework.data.redis.core.ZSetOperations.TypedTuple<String>>
                    similarItems = redisTemplate.opsForZSet()
                        .reverseRangeWithScores(matrixKey, 0, count * 2);

                if (similarItems != null) {
                    similarItems.stream()
                        .filter(tuple -> !seenProducts.contains(tuple.getValue()))
                        .forEach(tuple -> {
                            results.add(RecommendationItem.builder()
                                .productId(tuple.getValue())
                                .score(tuple.getScore())
                                .type(RecommendationType.COLLABORATIVE_FILTERING)
                                .reason("Customers who bought " + productId + " also bought this")
                                .build());
                            seenProducts.add(tuple.getValue());
                        });
                }
            }
        }

        // Association rules: frequently bought together
        if (type == RecommendationType.ASSOCIATION_RULES ||
            type == RecommendationType.ALL) {

            for (String productId : recentPurchaseIds) {
                String assocKey = "assoc-rules:" + productId;

                @SuppressWarnings("unchecked")
                Map<Object, Double> rules = redisTemplate.opsForHash()
                    .entries(assocKey);

                if (rules != null) {
                    rules.entrySet().stream()
                        .filter(e -> !seenProducts.contains(e.getKey()))
                        .sorted((a, b) -> Double.compare(b.getValue(), a.getValue()))
                        .forEach(e -> {
                            results.add(RecommendationItem.builder()
                                .productId((String) e.getKey())
                                .score(e.getValue())
                                .type(RecommendationType.ASSOCIATION_RULES)
                                .reason("Frequently bought together")
                                .build());
                            seenProducts.add((String) e.getKey());
                        });
                }
            }
        }

        return results.stream()
            .sorted((a, b) -> Double.compare(b.getScore(), a.getScore()))
            .limit(count)
            .collect(Collectors.toList());
    }

    private List<RecommendationItem> getFromCache(String key) {
        try {
            String cached = redisTemplate.opsForValue().get(key);
            if (cached != null && !cached.isEmpty()) {
                return Arrays.asList(cached.split("\\|"));
            }
        } catch (Exception e) {
            log.warn("Error reading from cache: {}", e.getMessage());
        }
        return null;
    }

    private void cacheRecommendations(String key, List<RecommendationItem> items) {
        try {
            String value = items.stream()
                .map(item -> item.getProductId() + ":" + item.getScore() + ":" +
                     item.getType() + ":" + item.getReason())
                .collect(Collectors.joining("|"));

            redisTemplate.opsForValue().set(key, value, cacheTTLMinutes, TimeUnit.MINUTES);
        } catch (Exception e) {
            log.warn("Error caching recommendations: {}", e.getMessage());
        }
    }

    private String buildCacheKey(Long userId, RecommendationType type) {
        return "rec:" + userId + ":" + type.name();
    }

    private void recordLatency(long millis, String source) {
        // Send to monitoring system (Prometheus, CloudWatch, etc.)
        log.debug("Recommendation latency [{}]: {}ms", source, millis);
    }
}

@Data
@Builder
class RecommendationItem {
    private String productId;
    private Double score;
    private RecommendationType type;
    private String reason;
}

@Data
class RecommendationResponse {
    private List<RecommendationItem> recommendations;
    private String source;
    private Long requestTime;
    private Long cachedAt;
    private String error;
}

enum RecommendationType {
    COLLABORATIVE_FILTERING,
    ASSOCIATION_RULES,
    PERSONALIZED,
    POPULARITY,
    ALL
}
```

### 3. ColdStartService - New User/Product Handling

```java
package com.recommendation.service;

import org.springframework.stereotype.Service;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.domain.PageRequest;
import com.recommendation.entity.Product;
import com.recommendation.repository.ProductRepository;
import lombok.extern.slf4j.Slf4j;
import java.util.*;
import java.util.stream.Collectors;

@Slf4j
@Service
public class ColdStartService {

    private final ProductRepository productRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final UserInteractionRepository userInteractionRepo;

    public ColdStartService(
            ProductRepository productRepository,
            RedisTemplate<String, String> redisTemplate,
            UserInteractionRepository userInteractionRepo) {
        this.productRepository = productRepository;
        this.redisTemplate = redisTemplate;
        this.userInteractionRepo = userInteractionRepo;
    }

    public List<RecommendationItem> generateRecommendations(
            Long userId,
            RecommendationType type,
            int count) {

        UserInteraction interaction = userInteractionRepo
            .findById(userId)
            .orElse(null);

        // For brand new user: use popularity or trending
        if (interaction == null || interaction.getTotalPurchases() == 0) {
            return getTrendingProducts(count);
        }

        // For user with category preference but few purchases
        String[] preferredCategories = interaction.getLastPurchasedCategories();
        if (preferredCategories != null && preferredCategories.length > 0) {
            return getTopProductsInCategories(preferredCategories, count);
        }

        // Default fallback: trending
        return getTrendingProducts(count);
    }

    private List<RecommendationItem> getTrendingProducts(int count) {
        try {
            // Get trending from Redis cache (updated hourly)
            String trendingKey = "trending:products";

            @SuppressWarnings("unchecked")
            Set<org.springframework.data.redis.core.ZSetOperations.TypedTuple<String>>
                trending = redisTemplate.opsForZSet()
                    .reverseRangeWithScores(trendingKey, 0, count - 1);

            if (trending != null && !trending.isEmpty()) {
                return trending.stream()
                    .map(tuple -> RecommendationItem.builder()
                        .productId(tuple.getValue())
                        .score(tuple.getScore())
                        .type(RecommendationType.POPULARITY)
                        .reason("Trending now")
                        .build())
                    .collect(Collectors.toList());
            }

            // Fallback: query database
            List<Product> products = productRepository.findTop10ByOrderByViewCountDesc();
            return products.stream()
                .map(p -> RecommendationItem.builder()
                    .productId(p.getId())
                    .score((double) p.getViewCount())
                    .type(RecommendationType.POPULARITY)
                    .reason("Popular")
                    .build())
                .limit(count)
                .collect(Collectors.toList());

        } catch (Exception e) {
            log.error("Error getting trending products", e);
            return Collections.emptyList();
        }
    }

    private List<RecommendationItem> getTopProductsInCategories(
            String[] categories,
            int count) {

        Set<String> seenProducts = new HashSet<>();
        List<RecommendationItem> results = new ArrayList<>();

        for (String category : categories) {
            if (results.size() >= count) break;

            // Try Redis cache first
            String categoryKey = "top-in-category:" + category;

            @SuppressWarnings("unchecked")
            Set<String> topProducts = redisTemplate.opsForSet()
                .members(categoryKey);

            if (topProducts != null) {
                topProducts.stream()
                    .filter(p -> !seenProducts.contains(p))
                    .limit(count - results.size())
                    .forEach(productId -> {
                        results.add(RecommendationItem.builder()
                            .productId(productId)
                            .score(0.5)  // placeholder
                            .type(RecommendationType.POPULARITY)
                            .reason("Popular in " + category)
                            .build());
                        seenProducts.add(productId);
                    });
            }
        }

        return results;
    }

    /**
     * Handle new product: blend content-based recommendations with popularity
     */
    public List<RecommendationItem> getRecommendationsForNewProduct(
            String productId,
            int count) {

        Product product = productRepository.findById(productId).orElse(null);
        if (product == null) {
            return Collections.emptyList();
        }

        // Content-based: find similar products by category/tags
        List<Product> similarByCategory = productRepository
            .findByCategory(product.getCategory(), PageRequest.of(0, count * 2))
            .getContent();

        return similarByCategory.stream()
            .filter(p -> !p.getId().equals(productId))
            .map(p -> RecommendationItem.builder()
                .productId(p.getId())
                .score(calculateContentSimilarity(product, p))
                .type(RecommendationType.PERSONALIZED)
                .reason("Similar product")
                .build())
            .limit(count)
            .collect(Collectors.toList());
    }

    private double calculateContentSimilarity(Product p1, Product p2) {
        double similarity = 0.0;

        // Category match
        if (p1.getCategory().equals(p2.getCategory())) {
            similarity += 0.3;
        }

        // Brand match
        if (p1.getBrand() != null && p1.getBrand().equals(p2.getBrand())) {
            similarity += 0.3;
        }

        // Price range proximity
        double priceDiff = Math.abs(p1.getPrice().doubleValue() -
                                   p2.getPrice().doubleValue());
        double priceScore = Math.max(0, 1.0 - (priceDiff / 1000.0));
        similarity += priceScore * 0.2;

        // Rating proximity
        if (p1.getAverageRating() > 0 && p2.getAverageRating() > 0) {
            double ratingDiff = Math.abs(p1.getAverageRating() -
                                        p2.getAverageRating());
            double ratingScore = Math.max(0, 1.0 - (ratingDiff / 5.0));
            similarity += ratingScore * 0.2;
        }

        return Math.min(1.0, similarity);
    }
}
```

### 4. ABTestService - Experiment Management

```java
package com.recommendation.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.recommendation.repository.ABTestAssignmentRepository;
import com.recommendation.repository.ABTestEventRepository;
import com.recommendation.entity.ABTestAssignment;
import com.recommendation.entity.ABTestEvent;
import lombok.extern.slf4j.Slf4j;
import java.util.*;

@Slf4j
@Service
public class ABTestService {

    private final ABTestAssignmentRepository assignmentRepo;
    private final ABTestEventRepository eventRepo;

    public ABTestService(
            ABTestAssignmentRepository assignmentRepo,
            ABTestEventRepository eventRepo) {
        this.assignmentRepo = assignmentRepo;
        this.eventRepo = eventRepo;
    }

    /**
     * Assign user to variant deterministically based on userId hash
     * Ensures no flicker: same userId always gets same variant
     */
    @Transactional
    public String assignUserToVariant(Long userId, String testId, int numVariants) {
        // Check if already assigned
        Optional<ABTestAssignment> existing = assignmentRepo
            .findByTestIdAndUserId(testId, userId);

        if (existing.isPresent()) {
            return existing.get().getVariant();
        }

        // Deterministic assignment
        int variantIndex = Math.abs(userId.hashCode()) % numVariants;
        String[] variants = {"control", "variant_a", "variant_b", "variant_c"};
        String assignedVariant = variants[Math.min(variantIndex, variants.length - 1)];

        ABTestAssignment assignment = ABTestAssignment.builder()
            .testId(testId)
            .userId(userId)
            .variant(assignedVariant)
            .assignedAt(new Date())
            .build();

        assignmentRepo.save(assignment);
        log.info("Assigned user {} to variant {} for test {}",
            userId, assignedVariant, testId);

        return assignedVariant;
    }

    /**
     * Track recommendation impression for A/B test
     */
    public void trackImpression(
            Long userId,
            String testId,
            List<String> productIds,
            String algorithm) {

        Optional<ABTestAssignment> assignment = assignmentRepo
            .findByTestIdAndUserId(testId, userId);

        if (assignment.isEmpty()) {
            log.warn("No assignment found for user {} test {}", userId, testId);
            return;
        }

        ABTestEvent event = ABTestEvent.builder()
            .testId(testId)
            .userId(userId)
            .variant(assignment.get().getVariant())
            .eventType("IMPRESSION")
            .productIds(String.join(",", productIds))
            .algorithm(algorithm)
            .eventAt(new Date())
            .build();

        eventRepo.save(event);
    }

    /**
     * Track click on recommended product
     */
    public void trackClick(
            Long userId,
            String testId,
            String productId,
            String algorithm) {

        Optional<ABTestAssignment> assignment = assignmentRepo
            .findByTestIdAndUserId(testId, userId);

        if (assignment.isEmpty()) {
            return;
        }

        ABTestEvent event = ABTestEvent.builder()
            .testId(testId)
            .userId(userId)
            .variant(assignment.get().getVariant())
            .eventType("CLICK")
            .productIds(productId)
            .algorithm(algorithm)
            .eventAt(new Date())
            .build();

        eventRepo.save(event);
    }

    /**
     * Track conversion for A/B test
     */
    public void trackConversion(
            Long userId,
            String testId,
            String purchasedProductId,
            double revenue,
            String algorithm) {

        Optional<ABTestAssignment> assignment = assignmentRepo
            .findByTestIdAndUserId(testId, userId);

        if (assignment.isEmpty()) {
            return;
        }

        ABTestEvent event = ABTestEvent.builder()
            .testId(testId)
            .userId(userId)
            .variant(assignment.get().getVariant())
            .eventType("CONVERSION")
            .productIds(purchasedProductId)
            .algorithm(algorithm)
            .revenue(new java.math.BigDecimal(revenue))
            .eventAt(new Date())
            .build();

        eventRepo.save(event);
    }

    /**
     * Calculate A/B test metrics
     */
    public ABTestMetrics getTestMetrics(String testId) {
        List<ABTestEvent> events = eventRepo.findByTestId(testId);

        Map<String, Long> impressionsByVariant = new HashMap<>();
        Map<String, Long> clicksByVariant = new HashMap<>();
        Map<String, Double> revenueByVariant = new HashMap<>();

        events.forEach(event -> {
            String variant = event.getVariant();
            impressionsByVariant.putIfAbsent(variant, 0L);
            clicksByVariant.putIfAbsent(variant, 0L);
            revenueByVariant.putIfAbsent(variant, 0.0);

            if ("IMPRESSION".equals(event.getEventType())) {
                impressionsByVariant.put(variant,
                    impressionsByVariant.get(variant) + 1);
            } else if ("CLICK".equals(event.getEventType())) {
                clicksByVariant.put(variant,
                    clicksByVariant.get(variant) + 1);
            } else if ("CONVERSION".equals(event.getEventType())) {
                revenueByVariant.put(variant,
                    revenueByVariant.get(variant) + event.getRevenue().doubleValue());
            }
        });

        // Calculate CTR and conversion rate per variant
        Map<String, Double> ctrByVariant = new HashMap<>();
        Map<String, Double> conversionRateByVariant = new HashMap<>();

        impressionsByVariant.forEach((variant, impressions) -> {
            long clicks = clicksByVariant.getOrDefault(variant, 0L);
            double ctr = impressions > 0 ? (double) clicks / impressions : 0.0;
            ctrByVariant.put(variant, ctr);
        });

        clicksByVariant.forEach((variant, clicks) -> {
            long conversions = events.stream()
                .filter(e -> variant.equals(e.getVariant()) &&
                        "CONVERSION".equals(e.getEventType()))
                .count();
            double convRate = clicks > 0 ? (double) conversions / clicks : 0.0;
            conversionRateByVariant.put(variant, convRate);
        });

        return ABTestMetrics.builder()
            .testId(testId)
            .impressionsByVariant(impressionsByVariant)
            .ctrByVariant(ctrByVariant)
            .conversionRateByVariant(conversionRateByVariant)
            .revenueByVariant(revenueByVariant)
            .build();
    }
}

@Data
@Builder
class ABTestMetrics {
    private String testId;
    private Map<String, Long> impressionsByVariant;
    private Map<String, Double> ctrByVariant;
    private Map<String, Double> conversionRateByVariant;
    private Map<String, Double> revenueByVariant;
}
```

### 5. ModelTrainingService - Batch Pipeline

```java
package com.recommendation.service;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.data.redis.core.RedisTemplate;
import com.mongodb.client.MongoCollection;
import org.bson.Document;
import lombok.extern.slf4j.Slf4j;
import java.util.*;

@Slf4j
@Service
public class ModelTrainingService {

    private final KafkaTemplate<String, String> kafkaTemplate;
    private final RedisTemplate<String, String> redisTemplate;
    private final MongoCollection<Document> associationRulesCollection;
    private final MongoCollection<Document> modelsCollection;

    @Scheduled(cron = "0 0 2 * * *") // Daily at 2 AM
    public void trainCollaborativeFilteringModel() {
        log.info("Starting collaborative filtering model training");
        long startTime = System.currentTimeMillis();

        try {
            // 1. Read last 24h of events from Kafka
            Map<String, Set<String>> userItemMatrix = readEventWindow(24);

            // 2. Compute item-item similarity using cosine distance
            Map<String, Map<String, Double>> cfMatrix = computeItemSimilarity(userItemMatrix);

            // 3. Store in Redis with expiry
            String modelDate = new java.text.SimpleDateFormat("yyyy-MM-dd")
                .format(new Date());
            String redisKeyPrefix = "cf-matrix:" + modelDate;

            cfMatrix.forEach((itemId, similarities) -> {
                String key = redisKeyPrefix + ":" + itemId;
                similarities.forEach((similarItemId, score) -> {
                    redisTemplate.opsForZSet()
                        .add(key, similarItemId, score);
                });
                redisTemplate.expire(key, java.time.Duration.ofDays(1));
            });

            // 4. Store metadata in MongoDB
            Document model = new Document()
                .append("_id", "cf-model-" + modelDate)
                .append("type", "COLLABORATIVE_FILTERING")
                .append("version", 1)
                .append("training_date", new Date())
                .append("data_points", userItemMatrix.size())
                .append("quality_metrics", new Document()
                    .append("precision_at_10", 0.42)
                    .append("recall_at_10", 0.38)
                    .append("ndcg", 0.65))
                .append("redis_key", redisKeyPrefix)
                .append("expiry", new Date(System.currentTimeMillis() +
                                          7 * 24 * 60 * 60 * 1000)); // 7 days

            modelsCollection.insertOne(model);

            // 5. Publish model-trained event
            publishModelUpdate("COLLABORATIVE_FILTERING", modelDate, redisKeyPrefix);

            log.info("CF model training completed in {}ms",
                System.currentTimeMillis() - startTime);

        } catch (Exception e) {
            log.error("CF model training failed", e);
        }
    }

    @Scheduled(fixedRate = 3600000) // Every hour
    public void updateAssociationRules() {
        log.info("Starting association rules update");
        long startTime = System.currentTimeMillis();

        try {
            // Read last hour of transactions
            Map<String, Integer> baskets = readTransactionWindow(1);

            // Compute Apriori rules (simplified)
            Map<List<String>, AssociationRule> rules = computeAprioriRules(baskets);

            // Store in MongoDB and Redis cache
            rules.forEach((antecedent, rule) -> {
                Document ruleDoc = new Document()
                    .append("antecedent_ids", antecedent)
                    .append("consequent_id", rule.getConsequentId())
                    .append("support", rule.getSupport())
                    .append("confidence", rule.getConfidence())
                    .append("lift", rule.getLift())
                    .append("count", rule.getCount())
                    .append("updated_at", new Date());

                associationRulesCollection.insertOne(ruleDoc);

                // Cache in Redis for fast lookup
                String cacheKey = "assoc-rules:" + antecedent.get(0);
                redisTemplate.opsForHash()
                    .put(cacheKey, rule.getConsequentId(),
                         String.valueOf(rule.getLift()));
                redisTemplate.expire(cacheKey, java.time.Duration.ofHours(6));
            });

            publishModelUpdate("ASSOCIATION_RULES",
                new java.text.SimpleDateFormat("yyyy-MM-dd").format(new Date()),
                "assoc-rules");

            log.info("Association rules update completed in {}ms",
                System.currentTimeMillis() - startTime);

        } catch (Exception e) {
            log.error("Association rules update failed", e);
        }
    }

    private Map<String, Set<String>> readEventWindow(int hours) {
        Map<String, Set<String>> userItemMatrix = new HashMap<>();
        // Read from Kafka: user-events topic
        // Group by user, collect items purchased
        return userItemMatrix;
    }

    private Map<String, Integer> readTransactionWindow(int hours) {
        Map<String, Integer> baskets = new HashMap<>();
        // Read from Kafka: aggregate baskets by transaction
        return baskets;
    }

    private Map<String, Map<String, Double>> computeItemSimilarity(
            Map<String, Set<String>> userItemMatrix) {

        Map<String, Map<String, Double>> result = new HashMap<>();
        List<String> items = new ArrayList<>(
            userItemMatrix.values().stream()
                .flatMap(Set::stream)
                .collect(java.util.stream.Collectors.toSet())
        );

        // Compute cosine similarity for all item pairs
        for (int i = 0; i < items.size(); i++) {
            String item1 = items.get(i);
            Map<String, Double> similarities = new HashMap<>();

            for (int j = i + 1; j < items.size(); j++) {
                String item2 = items.get(j);
                double similarity = computeCosineSimilarity(
                    item1, item2, userItemMatrix);
                similarities.put(item2, similarity);
            }

            result.put(item1, similarities);
        }

        return result;
    }

    private double computeCosineSimilarity(
            String item1,
            String item2,
            Map<String, Set<String>> userItemMatrix) {

        Set<String> users1 = new HashSet<>();
        Set<String> users2 = new HashSet<>();

        userItemMatrix.forEach((user, items) -> {
            if (items.contains(item1)) users1.add(user);
            if (items.contains(item2)) users2.add(user);
        });

        int intersection = (int) users1.stream()
            .filter(users2::contains)
            .count();

        double denominator = Math.sqrt(users1.size() * users2.size());
        return denominator > 0 ? intersection / denominator : 0.0;
    }

    private Map<List<String>, AssociationRule> computeAprioriRules(
            Map<String, Integer> baskets) {

        Map<List<String>, AssociationRule> rules = new HashMap<>();
        // Simplified Apriori implementation
        // In production: use Spark MLlib or specialized library
        return rules;
    }

    private void publishModelUpdate(String type, String modelDate, String redisKey) {
        try {
            String message = String.format(
                "{\"signalType\":\"%s\",\"modelDate\":\"%s\",\"redisKey\":\"%s\",\"timestamp\":%d}",
                type, modelDate, redisKey, System.currentTimeMillis()
            );
            kafkaTemplate.send("model-update-signals", message);
        } catch (Exception e) {
            log.error("Failed to publish model update", e);
        }
    }
}

@Data
@Builder
class AssociationRule {
    private String consequentId;
    private double support;     // % of all baskets containing all items
    private double confidence;  // % of antecedent baskets also containing consequent
    private double lift;        // (confidence / consequent_support)
    private long count;         // number of co-occurrences
}
```

---

## Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| Redis cache down | Recommendations slow (fall through to DB) | Multi-region Redis, failover to MongoDB |
| Kafka broker down | Can't process new events | Kafka replication factor 3; circuit breaker |
| Model training fails | Stale recommendations for 24h | Run incremental hourly; keep 2 versions |
| Cold start for new user | Return empty recs | Fallback to popularity-based + trending |
| Recommendation API rate limited | User sees no recs | Local caching; serve stale data gracefully |
| MongoDB slow query | API latency spike | Index on frequently queried fields; Redis cache |

---

## Scaling Strategy

### Horizontal Scaling
- **RecommendationService:** Stateless, scale horizontally behind load balancer
- **Kafka:** Increase partitions (partition by userId % num_partitions)
- **Redis:** Cluster mode with 16 shards for item-similarity matrix
- **MongoDB:** Sharding by user_id or product_id

### Vertical Scaling (temporary)
- Increase Redis memory for larger CF matrices
- Increase Kafka replication/retention for more historical data
- Increase PostgreSQL IOPS for A/B test event writes

### Caching Strategy
- Redis for hot data (recommendations TTL 1h, vectors TTL 24h)
- PostgreSQL for warm data (user interactions, test assignments)
- MongoDB for cold data (historical models, training logs)

### Rate Limiting
```java
@RestController
@RequestMapping("/api/recommendations")
public class RecommendationController {

    @RateLimiter(name = "recommendation-api")
    @GetMapping
    public RecommendationResponse getRecommendations(
            @RequestParam Long userId,
            @RequestParam int count) {
        return recommendationService.getRecommendations(userId, count);
    }
}
```

---

## Monitoring & Observability

### Metrics
```
recommendation_api_latency_ms (histogram)
  - tags: algorithm, cache_hit, user_segment
  - p50, p95, p99

recommendation_cache_hit_rate (gauge)
  - tags: type (COLLAB, ASSOC, POPULARITY)

model_training_duration_ms (histogram)
  - tags: type (CF, APRIORI)

kafka_lag (gauge)
  - topic: user-events
  - tags: consumer_group

redis_memory_bytes (gauge)
  - tags: key_pattern
```

### Alerts
- Recommendation API p95 > 100ms
- Cache hit rate < 80%
- Kafka consumer lag > 10 minutes
- Model training failure (no model update in 26 hours)
- Cold start rate > 5% (indicates caching issues)

### Logging
```java
log.info("Recommendation served in {}ms [cache_hit={}, user={}, count={}]",
    latency, cacheHit, userId, recommendationCount);
```

---

## Summary Cheat Sheet

| Component | Tech | Key Details |
|-----------|------|------------|
| **User Events** | Kafka | 1M/day, TTL 7 days, partitioned by userId |
| **Recommendation Cache** | Redis | Precomputed, TTL 1h, 20MB size |
| **CF Matrix** | Redis Sorted Set | Item-item similarity, computed daily |
| **Association Rules** | MongoDB | Apriori output, hourly updates |
| **Cold Start** | Popularity + Content | Fallback strategy for new users/products |
| **A/B Testing** | PostgreSQL | Deterministic assignment, CTR/conversion tracking |
| **SLA** | <100ms p95 | Cache-first strategy, 95%+ hit rate |
| **Training** | Daily batch | 24h events → compute models → update Redis |
| **Scaling** | Horizontal | Stateless services, Redis cluster, Kafka partitions |

---

**End of Deep Dive**
