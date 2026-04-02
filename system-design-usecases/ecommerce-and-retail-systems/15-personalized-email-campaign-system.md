---
title: Personalized Email Campaign System
layout: default
---

# Personalized Email Campaign System — Deep Dive Design

> **Scenario:** Build a personalized email campaign system for a large e-commerce platform. Requirements include abandoned cart reminders (if cart inactive for 24 hours), price drop alerts for wishlisted items, back-in-stock notifications, personalized recommendations weekly, birthday/anniversary offers, and post-purchase review requests (7 days after delivery). Must send **10 million emails daily** and track open rates, click rates, and conversions.
>
> **Tech Stack:** Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka

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
- Trigger emails based on 6+ user behavior events (cart abandonment, price drops, back-in-stock, recommendations, birthdays, reviews)
- Personalize email content at scale with user-specific data (name, items, prices, recommendations)
- Send 10 million emails per day (115 emails/sec sustained, 500+ emails/sec peak)
- Track email engagement (open, click) and attribute conversions back to emails
- Prevent duplicate emails via deduplication
- Retry failed email sends with exponential backoff
- Support multiple email templates and dynamic content injection

### Non-Functional Requirements
- Campaign execution latency: < 1 second trigger-to-queue
- Email send latency: 95th percentile < 500ms per batch
- Deduplication accuracy: 99.99%
- Email delivery success rate: > 98%
- Re-triggering prevention: zero duplicates within 24 hours of first send
- Tracking accuracy: > 99% for clickthrough and open events

### Constraints
- No real-time SMS/push integration (email only)
- SES/SendGrid API rate limits (~10k/sec per account)
- Email template rendering must be fast (< 100ms per email)
- Cost optimization required (10M emails * $0.0001 per email ≈ $1000/day)

---

## Capacity Estimation

| Metric | Calculation | Value |
|--------|-------------|-------|
| **Daily Emails** | Requirement | 10,000,000 |
| **Emails/Second (Avg)** | 10M / (86400s) | ~115 emails/sec |
| **Emails/Second (Peak)** | Assuming 5x avg during business hours | ~575 emails/sec |
| **Event Rate (Triggers)** | Estimated 15-20% user engagement | 2,000,000 events/day |
| **Daily Engagement Events** | Opens + Clicks + Bounces | ~8,000,000 events |
| **Event Rate (Engagement)** | 8M / 86400s | ~92 events/sec |
| **Redis Dedup Keys** | Stored 7 days, 10M emails/day | ~70M keys @ 0.5KB each = ~35GB |
| **Campaign History (PostgreSQL)** | 10M/day * 365 days | ~3.6B rows/year |
| **Engagement Logs (MongoDB)** | 8M/day * 365 days | ~2.9B documents/year |
| **Storage (PostgreSQL)** | 3.6B rows * 300 bytes avg | ~1TB/year |
| **Storage (MongoDB)** | 2.9B docs * 200 bytes avg | ~580GB/year |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Event Sources                               │
│  (Cart Service, Catalog, Order Service, User Service)           │
└────────────────┬────────────────────────────────────────────────┘
                 │
                 ▼
        ┌────────────────────┐
        │   Kafka Topics     │
        │ - cart.abandoned   │
        │ - wishlist.price   │
        │ - order.delivered  │
        └────────┬───────────┘
                 │
                 ▼
    ┌────────────────────────────────┐
    │  Campaign Trigger Service      │
    │  (Kafka Consumer)              │
    └────────────────────────────────┘
         │              │
         ▼              ▼
    ┌─────────────┐  ┌──────────────────┐
    │  Deduplica- │  │  Personalization │
    │  tion (Redis)   │  Engine          │
    └─────────────┘  └──────────────────┘
         │              │
         ▼              ▼
    ┌─────────────────────────────────┐
    │  Email Batch Sender             │
    │  (Spring Batch Partitioner)     │
    └────────────┬────────────────────┘
                 │
                 ▼
        ┌────────────────────┐
        │  SES/SendGrid API  │
        │  Rate: 10k/sec     │
        └────────┬───────────┘
                 │
    ┌────────────┴────────────┐
    ▼                         ▼
┌─────────────┐      ┌──────────────────┐
│ Sent Queue  │      │  Failed Queue    │
│ (Kafka DLQ) │      │  (Kafka DLQ)     │
└────────┬────┘      └────────┬─────────┘
         │                    │
         ▼                    ▼
    ┌─────────────────────────────────┐
    │  Email Tracking Service         │
    │  (Listening on Sent Queue)      │
    └────────────────────────────────┘
         │              │              │
         ▼              ▼              ▼
    Opens         Clicks         Bounces
     │              │              │
     └──────┬───────┴──────┬───────┘
            ▼
    ┌──────────────────────┐
    │  Analytics Pipeline  │
    │  MongoDB            │
    └──────────────────────┘
```

---

## Core Design Questions Answered

### 1. How do you trigger different types of emails based on user behavior?

**Answer:** Use Kafka event streaming with dedicated consumers for each trigger type. Events from upstream services (cart, order, catalog) are published to Kafka topics, and dedicated trigger handlers subscribe to them.

**Design Rationale:**
- **Event-driven decoupling:** Cart Service doesn't know about Email Service; communication happens via Kafka
- **Scalability:** Each trigger type can be independently scaled
- **Extensibility:** Add new trigger types by adding new consumers without modifying existing code
- **Exactly-once semantics:** Kafka guarantees at-least-once delivery; deduplication happens downstream

**Trigger Mapping:**
| Trigger | Event Topic | Delay | Example |
|---------|------------|-------|---------|
| Abandoned Cart | `cart.abandoned` | 24h | Cart inactive for 24 hours |
| Price Drop | `wishlist.item.price_dropped` | Immediate | Wishlisted item price drops 10%+ |
| Back-in-Stock | `inventory.back_in_stock` | Immediate | User's wishlist item restocked |
| Weekly Recs | `user.weekly_digest_trigger` | Weekly (Thu 9am) | Scheduled digest generation |
| Birthday | `user.birthday_approaching` | 2 days before | Birthday/anniversary date |
| Review Request | `order.delivered` | 7 days after delivery | Order marked delivered |

### 2. How do you personalize email content at scale?

**Answer:** Use a template-based approach with dynamic content injection at send time. Store user-specific data in hot storage (Redis) for fast lookups, and inject at the moment of sending.

**Personalization Layers:**
1. **Static Template:** HTML template with placeholders (e.g., `{{firstName}}`, `{{cartTotal}}`)
2. **User Data Context:** Fetch user data from Redis (name, email, preferences)
3. **Dynamic Content:** Fetch product recommendations, prices, images from PostgreSQL/MongoDB
4. **Rendering:** Use Thymeleaf template engine to render HTML in-process
5. **Tracking URLs:** Append unique token to clickthrough URLs for attribution

**Data Sources:**
- **User profile:** Redis (10 min TTL) → PostgreSQL
- **Personalized recommendations:** MongoDB (computed offline) → Redis hot tier
- **Product data:** Redis product catalog (1h TTL) → PostgreSQL
- **Current prices:** Redis real-time → PostgreSQL

### 3. How do you schedule and batch email sending?

**Answer:** Use Spring Batch with partitioned jobs and rate-limiting to 10,000 emails/second.

**Batching Strategy:**
- **Partition Size:** 100,000 emails per partition (scalable)
- **Partitions:** 10M / 100k = 100 partitions
- **Parallel Threads:** 10 threads × 100 partitions = 1000 threads
- **Rate Limit:** 10,000/sec = 100µs per email; semaphore-based throttling
- **Retry Policy:** Exponential backoff with DLQ for ultimate failures

### 4. How do you prevent sending duplicate emails?

**Answer:** Multi-layered deduplication using Redis keys with TTL and idempotency checks.

**Deduplication Strategy:**
```
Key: email:sent:{userId}:{campaignType}:{date}
Value: timestamp (Unix epoch)
TTL: 7 days
```

**Checks:**
1. **Pre-enqueue check:** Before adding to batch queue, check Redis key
2. **Pre-send check:** Immediately before calling SES API, re-check key (in-flight safety)
3. **Atomic set-if-not-exists:** Use Redis SET NX with TTL for atomicity

**Idempotency:**
- Each email has UUID `{userId}:{campaignType}:{eventId}:{timestamp}`
- If SES call fails but email was sent, idempotency key prevents re-delivery to SES
- DLQ consumer checks: if email already in `email_sends` table, skip retry

### 5. How do you handle email delivery failures and retries?

**Answer:** Kafka Dead Letter Queue (DLQ) with exponential backoff and manual intervention limits.

**Retry Strategy:**
```
Attempt 1: Immediate
Attempt 2: 1 minute later
Attempt 3: 5 minutes later
Attempt 4: 30 minutes later
Attempt 5: 2 hours later
Attempt 6: Next day
After 6 attempts: Move to manual review queue (operator-handled)
```

**Error Handling:**
- **Soft errors** (429 rate limit, 500 server error): Retry
- **Hard errors** (400 bad email, 550 mailbox invalid): Log and discard
- **Transient errors** (DNS timeout, network timeout): Retry with backoff
- **Account-level errors** (quota exceeded): Pause all sends, alert on-call engineer

### 6. How do you track email engagement and attribute conversions?

**Answer:** Generate unique tracking URLs and pixels, publish engagement events to Kafka, persist in MongoDB.

**Tracking Mechanism:**
```
Original URL: https://example.com/product/nike-shoes-123
Tracked URL: https://tracking.example.com/click?t={token}&email_id={emailId}
Pixel: <img src="https://tracking.example.com/open?t={token}&email_id={emailId}" />

Token = Base64(userId:campaignId:timestamp:hash(emailId))
```

**Engagement Pipeline:**
1. User clicks tracked link → GET to tracking service
2. Tracking service decodes token, logs click event to Kafka `email.engagement` topic
3. Kafka consumer persists to MongoDB `email_engagements` collection
4. Analytics pipeline joins with `email_sends` to compute metrics

**Attribution:**
- Track UTM parameters appended to all tracked URLs
- Join email engagement → user session → order conversion within 30-day window
- Calculate: email_id → order_id → revenue attribution

---

## Microservices Breakdown

| Service | Responsibility | Tech Stack | Throughput |
|---------|-----------------|-------------|-----------|
| **Campaign Trigger Service** | Consumes Kafka events, validates triggers, enqueues campaigns | Spring Boot + Kafka Consumer | 2M events/day |
| **Email Deduplication Service** | Checks/sets Redis dedup keys | Spring Boot + Jedis | 575 ops/sec peak |
| **Email Personalization Service** | Renders templates with user data | Spring Boot + Thymeleaf | 500 renders/sec |
| **Email Batch Sender** | Partitions and sends batches via SES | Spring Batch + AWS SDK | 10,000 emails/sec |
| **Email Tracking Service** | Processes click/open events from users | Spring Boot + Kafka Consumer | 92 events/sec |
| **Campaign Metrics Aggregator** | Computes open rates, CTR, conversions | Spring Boot + Scheduled Tasks | Hourly aggregation |
| **Admin API** | Campaign creation, manual send, reporting | Spring Boot REST | Admin-level |

---

## Database Design

### PostgreSQL Schema: Campaign Management & History

```sql
-- Campaigns (stores campaign metadata)
CREATE TABLE campaigns (
  campaign_id BIGINT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
  campaign_type VARCHAR(50) NOT NULL, -- 'abandoned_cart', 'price_drop', etc.
  name VARCHAR(255) NOT NULL,
  template_id BIGINT NOT NULL REFERENCES email_templates(template_id),
  trigger_condition JSONB NOT NULL, -- e.g., {"inactivity_hours": 24}
  segment_id BIGINT, -- Optional: specific user segment
  enabled BOOLEAN DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  created_by VARCHAR(100),
  INDEX idx_campaign_type (campaign_type),
  INDEX idx_enabled_created (enabled, created_at DESC)
);

-- Email Templates
CREATE TABLE email_templates (
  template_id BIGINT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
  name VARCHAR(255) NOT NULL,
  subject_template VARCHAR(500) NOT NULL,
  html_template TEXT NOT NULL,
  text_template TEXT,
  template_variables JSONB, -- e.g., ["firstName", "cartTotal", "items"]
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- Email Sends (immutable log of sent emails)
CREATE TABLE email_sends (
  email_send_id BIGINT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
  email_id UUID NOT NULL UNIQUE,
  campaign_id BIGINT NOT NULL REFERENCES campaigns(campaign_id),
  user_id BIGINT NOT NULL,
  recipient_email VARCHAR(255) NOT NULL,
  subject VARCHAR(500),
  personalization_context JSONB, -- e.g., {"firstName": "John", "cartTotal": 150.00}
  status VARCHAR(50) DEFAULT 'SENT', -- SENT, BOUNCED, COMPLAINED, FAILED
  sent_at TIMESTAMP DEFAULT NOW(),
  bounce_type VARCHAR(20), -- PERMANENT, TRANSIENT
  bounce_reason TEXT,
  INDEX idx_user_campaign (user_id, campaign_id),
  INDEX idx_sent_at (sent_at DESC),
  INDEX idx_email_id (email_id)
);

-- Email Engagement Aggregates (hourly summaries from MongoDB)
CREATE TABLE email_engagement_summaries (
  summary_id BIGINT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
  campaign_id BIGINT NOT NULL REFERENCES campaigns(campaign_id),
  hour_bucket TIMESTAMP NOT NULL, -- 2026-04-01 14:00:00
  sent_count BIGINT DEFAULT 0,
  open_count BIGINT DEFAULT 0,
  click_count BIGINT DEFAULT 0,
  bounce_count BIGINT DEFAULT 0,
  complaint_count BIGINT DEFAULT 0,
  open_rate DECIMAL(5, 2), -- percentage
  click_rate DECIMAL(5, 2),
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE (campaign_id, hour_bucket),
  INDEX idx_campaign_hour (campaign_id, hour_bucket DESC)
);

-- User Email Preferences
CREATE TABLE user_email_preferences (
  user_id BIGINT PRIMARY KEY REFERENCES users(user_id),
  abandoned_cart_enabled BOOLEAN DEFAULT TRUE,
  price_drop_enabled BOOLEAN DEFAULT TRUE,
  back_in_stock_enabled BOOLEAN DEFAULT TRUE,
  weekly_digest_enabled BOOLEAN DEFAULT TRUE,
  birthday_offer_enabled BOOLEAN DEFAULT TRUE,
  review_request_enabled BOOLEAN DEFAULT TRUE,
  unsubscribe_timestamp TIMESTAMP,
  unsubscribe_reason VARCHAR(255),
  updated_at TIMESTAMP DEFAULT NOW()
);
```

### MongoDB Schema: Real-Time Engagement Events

```javascript
// Collection: email_engagements
db.createCollection("email_engagements", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["_id", "email_id", "event_type", "user_id", "campaign_id", "timestamp"],
      properties: {
        _id: { bsonType: "objectId" },
        email_id: { bsonType: "string", description: "UUID of email send" },
        event_type: { enum: ["OPEN", "CLICK", "BOUNCE", "COMPLAINT", "DELIVERY"] },
        user_id: { bsonType: "long" },
        campaign_id: { bsonType: "long" },
        recipient_email: { bsonType: "string" },
        timestamp: { bsonType: "date" },
        click_url: { bsonType: "string" }, // if CLICK event
        clicked_link_type: { bsonType: "string" }, // "product", "recommend", "offer"
        user_agent: { bsonType: "string" },
        ip_address: { bsonType: "string" },
        device_type: { enum: ["MOBILE", "TABLET", "DESKTOP"] },
        client: { bsonType: "string" }, // e.g., "gmail", "outlook"
        session_id: { bsonType: "string" },
        bounce_type: { enum: ["PERMANENT", "TRANSIENT", "COMPLAINT"] },
        bounce_reason: { bsonType: "string" }
      }
    }
  }
});

// Indexes
db.email_engagements.createIndex({ "email_id": 1 }, { unique: true, sparse: true });
db.email_engagements.createIndex({ "user_id": 1, "timestamp": -1 });
db.email_engagements.createIndex({ "campaign_id": 1, "timestamp": -1 });
db.email_engagements.createIndex({ "event_type": 1, "timestamp": -1 });
db.email_engagements.createIndex({ "timestamp": -1 }, { expireAfterSeconds: 7776000 }); // 90-day TTL

// Collection: personalization_recommendations (pre-computed, updated hourly)
db.personalization_recommendations.createIndex({ "user_id": 1 });
db.personalization_recommendations.createIndex({ "computed_at": 1 }, { expireAfterSeconds: 86400 });
```

---

## Redis Data Structures

```redis
# 1. Deduplication Keys (prevents duplicate sends)
email:sent:{userId}:{campaignType}:{date}
  Type: String
  Value: timestamp
  TTL: 7 days
  Example: email:sent:12345:abandoned_cart:2026-04-01 → "1743667200"

# 2. Campaign Enqueue Lock (prevents concurrent re-enqueue)
campaign:enqueue:lock:{campaignId}:{userId}
  Type: String
  Value: "locked"
  TTL: 5 minutes

# 3. User Email Preference Cache
user:email:prefs:{userId}
  Type: Hash
  Fields: { abandoned_cart_enabled: 1, price_drop_enabled: 1, ... }
  TTL: 1 hour
  Example: HGETALL user:email:prefs:12345

# 4. Hot Product Cache (for personalization)
product:{productId}:data
  Type: Hash
  Fields: { name, price, image_url, category, rating }
  TTL: 1 hour

# 5. User Recent Activity Cache (for engagement signals)
user:{userId}:recent:activity
  Type: Sorted Set
  Score: timestamp
  Member: activity type (viewed, clicked, purchased)
  TTL: 30 days

# 6. Campaign Trigger Locks (distributed locking for exactly-once)
trigger:processed:{eventId}
  Type: String
  Value: "1"
  TTL: 24 hours

# 7. Email Engagement Counters (for real-time dashboards)
email:engagement:{campaignId}:{hour}:opens
  Type: String
  Value: counter
  TTL: 24 hours (or longer for historical data)
  Example: email:engagement:42:2026-04-01-14:opens → "15000"

# 8. Rate Limiter (for SES/SendGrid API)
ratelimit:email:send:{serviceName}
  Type: String (with INCR)
  Value: counter
  TTL: 1 second

# 9. Top Autocomplete Queries (for search)
top:queries:{date}
  Type: Sorted Set (for leaderboard)
  Score: frequency
  Member: query text
  TTL: 30 days
```

---

## Kafka Event Flow

### Topics & Partitioning

```
Topic: cart.abandoned
  Partitions: 50 (by user_id)
  Retention: 7 days
  Replication Factor: 3
  Event Schema:
    {
      "event_id": "uuid",
      "user_id": 12345,
      "cart_id": "cart_abc123",
      "items": [...],
      "cart_total": 150.00,
      "last_activity": "2026-04-01T10:00:00Z",
      "timestamp": 1743667200
    }

Topic: wishlist.item.price_dropped
  Partitions: 50 (by user_id)
  Event Schema:
    {
      "event_id": "uuid",
      "user_id": 12345,
      "product_id": 789,
      "old_price": 100.00,
      "new_price": 75.00,
      "currency": "USD"
    }

Topic: order.delivered
  Partitions: 50 (by user_id)
  Event Schema:
    {
      "event_id": "uuid",
      "order_id": 54321,
      "user_id": 12345,
      "delivered_at": "2026-04-01T18:00:00Z"
    }

Topic: email.campaign.triggered
  Partitions: 100 (by campaign_id)
  Retention: 30 days
  Event Schema:
    {
      "campaign_trigger_id": "uuid",
      "campaign_id": 42,
      "user_id": 12345,
      "trigger_event_id": "original_event_uuid",
      "scheduled_send_time": 1743753600,
      "personalization_data": { ... }
    }

Topic: email.sent
  Partitions: 100 (by email_id hash)
  Retention: 90 days
  Event Schema:
    {
      "email_id": "uuid",
      "campaign_id": 42,
      "user_id": 12345,
      "recipient_email": "user@example.com",
      "sent_at": 1743667200,
      "ses_response": { ... }
    }

Topic: email.engagement
  Partitions: 100 (by user_id)
  Retention: 90 days
  Event Schema:
    {
      "event_id": "uuid",
      "email_id": "uuid",
      "event_type": "OPEN" | "CLICK" | "BOUNCE" | "COMPLAINT",
      "user_id": 12345,
      "timestamp": 1743667200,
      "click_url": "https://tracking.example.com/click?t=..."
    }

Topic: email.campaign.failed
  Partitions: 10 (for manual review)
  Retention: 30 days
  (DLQ for failed email sends)
```

---

## Implementation Code

### 1. Campaign Trigger Service (Kafka Consumer)

```java
@Service
@Slf4j
public class CampaignTriggerService {

    @Autowired private KafkaTemplate<String, String> kafkaTemplate;
    @Autowired private RedisTemplate<String, String> redisTemplate;
    @Autowired private CampaignRepository campaignRepository;
    @Autowired private ObjectMapper objectMapper;

    // Consume cart abandonment events
    @KafkaListener(topics = "cart.abandoned", groupId = "email-trigger-service")
    public void handleAbandonedCart(ConsumerRecord<String, String> record) {
        try {
            AbandonedCartEvent event = objectMapper.readValue(record.value(), AbandonedCartEvent.class);

            // 1. Check deduplication
            String dedupKey = String.format("email:sent:%d:abandoned_cart:%s",
                event.getUserId(), LocalDate.now());

            if (redisTemplate.hasKey(dedupKey)) {
                log.warn("Abandoned cart email already sent for user {} today", event.getUserId());
                return;
            }

            // 2. Check user preferences
            if (!isEmailTypeEnabled(event.getUserId(), "ABANDONED_CART")) {
                log.debug("User {} has abandoned cart emails disabled", event.getUserId());
                return;
            }

            // 3. Create campaign trigger
            CampaignTrigger trigger = CampaignTrigger.builder()
                .campaignId(ABANDONED_CART_CAMPAIGN_ID)
                .userId(event.getUserId())
                .triggerEventId(event.getEventId())
                .triggerType("ABANDONED_CART")
                .personalizationData(Map.of(
                    "cartId", event.getCartId(),
                    "items", event.getItems(),
                    "cartTotal", event.getCartTotal(),
                    "lastActivity", event.getLastActivity()
                ))
                .scheduledSendTime(Instant.now().plus(Duration.ofMinutes(5)))
                .status("PENDING")
                .createdAt(Instant.now())
                .build();

            // 4. Publish campaign trigger event
            String triggerJson = objectMapper.writeValueAsString(trigger);
            kafkaTemplate.send("email.campaign.triggered",
                String.valueOf(trigger.getUserId()), triggerJson)
                .whenComplete((result, ex) -> {
                    if (ex == null) {
                        log.info("Campaign trigger published for user {}", trigger.getUserId());
                    } else {
                        log.error("Failed to publish campaign trigger", ex);
                    }
                });

        } catch (JsonProcessingException e) {
            log.error("Failed to parse abandoned cart event", e);
        }
    }

    // Consume price drop events
    @KafkaListener(topics = "wishlist.item.price_dropped", groupId = "email-trigger-service")
    public void handlePriceDropAlert(ConsumerRecord<String, String> record) {
        try {
            PriceDropEvent event = objectMapper.readValue(record.value(), PriceDropEvent.class);

            String dedupKey = String.format("email:sent:%d:price_drop:%s:%d",
                event.getUserId(), LocalDate.now(), event.getProductId());

            if (redisTemplate.hasKey(dedupKey)) {
                return; // Already sent today
            }

            if (!isEmailTypeEnabled(event.getUserId(), "PRICE_DROP")) {
                return;
            }

            CampaignTrigger trigger = CampaignTrigger.builder()
                .campaignId(PRICE_DROP_CAMPAIGN_ID)
                .userId(event.getUserId())
                .triggerEventId(event.getEventId())
                .triggerType("PRICE_DROP")
                .personalizationData(Map.of(
                    "productId", event.getProductId(),
                    "oldPrice", event.getOldPrice(),
                    "newPrice", event.getNewPrice(),
                    "savingsPercent", calculateSavingsPercent(event.getOldPrice(), event.getNewPrice())
                ))
                .scheduledSendTime(Instant.now())
                .status("PENDING")
                .createdAt(Instant.now())
                .build();

            String triggerJson = objectMapper.writeValueAsString(trigger);
            kafkaTemplate.send("email.campaign.triggered",
                String.valueOf(trigger.getUserId()), triggerJson);

        } catch (JsonProcessingException e) {
            log.error("Failed to parse price drop event", e);
        }
    }

    // Consume post-delivery review request events
    @KafkaListener(topics = "order.delivered", groupId = "email-trigger-service")
    public void handlePostPurchaseReview(ConsumerRecord<String, String> record) {
        try {
            OrderDeliveredEvent event = objectMapper.readValue(record.value(), OrderDeliveredEvent.class);

            // Calculate 7-day delayed send
            Instant sendTime = event.getDeliveredAt().plus(Duration.ofDays(7));

            CampaignTrigger trigger = CampaignTrigger.builder()
                .campaignId(REVIEW_REQUEST_CAMPAIGN_ID)
                .userId(event.getUserId())
                .triggerEventId(event.getEventId())
                .triggerType("REVIEW_REQUEST")
                .personalizationData(Map.of(
                    "orderId", event.getOrderId(),
                    "deliveredDate", event.getDeliveredAt().toString()
                ))
                .scheduledSendTime(sendTime)
                .status("PENDING")
                .createdAt(Instant.now())
                .build();

            String triggerJson = objectMapper.writeValueAsString(trigger);
            kafkaTemplate.send("email.campaign.triggered",
                String.valueOf(trigger.getUserId()), triggerJson);

        } catch (JsonProcessingException e) {
            log.error("Failed to parse order delivered event", e);
        }
    }

    private boolean isEmailTypeEnabled(Long userId, String emailType) {
        String prefKey = "user:email:prefs:" + userId;
        Boolean enabled = (Boolean) redisTemplate.opsForHash()
            .get(prefKey, emailType.toLowerCase() + "_enabled");

        if (enabled != null) {
            return enabled;
        }

        // Fall back to database if not in cache
        return true; // Default: enabled
    }

    private double calculateSavingsPercent(BigDecimal oldPrice, BigDecimal newPrice) {
        return oldPrice.subtract(newPrice)
            .multiply(BigDecimal.valueOf(100))
            .divide(oldPrice, 2, RoundingMode.HALF_UP)
            .doubleValue();
    }
}
```

### 2. Email Deduplication & Personalization Service

```java
@Service
@Slf4j
public class EmailPersonalizationService {

    @Autowired private RedisTemplate<String, Object> redisTemplate;
    @Autowired private ThymeleafEngineConfiguration thymeleafEngine;
    @Autowired private EmailTemplateRepository templateRepository;
    @Autowired private UserRepository userRepository;
    @Autowired private ProductRepository productRepository;
    @Autowired private ObjectMapper objectMapper;

    private static final String DEDUP_KEY_PREFIX = "email:sent:";
    private static final int DEDUP_TTL_DAYS = 7;

    /**
     * Render personalized email content with user-specific data
     */
    public PersonalizedEmail renderPersonalizedEmail(
            Long campaignId,
            Long userId,
            EmailTemplate template,
            Map<String, Object> personalizationContext) {

        try {
            // 1. Fetch user profile from database or cache
            User user = fetchUser(userId);

            // 2. Enrich personalization context
            Map<String, Object> enrichedContext = new HashMap<>(personalizationContext);
            enrichedContext.put("firstName", user.getFirstName());
            enrichedContext.put("lastName", user.getLastName());
            enrichedContext.put("email", user.getEmail());
            enrichedContext.put("preferredLanguage", user.getPreferredLanguage());
            enrichedContext.put("tierLevel", user.getTierLevel());

            // 3. Fetch product data if campaign includes product recommendations
            if (personalizationContext.containsKey("productIds")) {
                List<Long> productIds = (List<Long>) personalizationContext.get("productIds");
                List<Product> products = productRepository.findAllById(productIds);
                enrichedContext.put("products", products);
            }

            // 4. Render subject line
            String subject = renderTemplate(template.getSubjectTemplate(), enrichedContext);

            // 5. Render HTML body with Thymeleaf
            String htmlBody = renderTemplate(template.getHtmlTemplate(), enrichedContext);

            // 6. Append tracking pixel
            String trackingToken = generateTrackingToken(userId, campaignId);
            String pixelHtml = String.format(
                "<img src='https://tracking.example.com/open?t=%s' width='1' height='1' />",
                trackingToken
            );
            htmlBody += pixelHtml;

            // 7. Generate unique email ID for idempotency
            String emailId = UUID.randomUUID().toString();

            return PersonalizedEmail.builder()
                .emailId(emailId)
                .campaignId(campaignId)
                .userId(userId)
                .recipientEmail(user.getEmail())
                .subject(subject)
                .htmlBody(htmlBody)
                .textBody(template.getTextTemplate() != null ?
                    renderTemplate(template.getTextTemplate(), enrichedContext) : null)
                .trackingToken(trackingToken)
                .personalizationContext(enrichedContext)
                .renderedAt(Instant.now())
                .build();

        } catch (Exception e) {
            log.error("Failed to render personalized email for user {}", userId, e);
            throw new EmailRenderingException("Failed to render email", e);
        }
    }

    /**
     * Check if email has already been sent (deduplication)
     */
    public boolean isDuplicate(Long userId, String campaignType, LocalDate date) {
        String dedupKey = String.format("%s%d:%s:%s",
            DEDUP_KEY_PREFIX, userId, campaignType, date);

        return Boolean.TRUE.equals(redisTemplate.hasKey(dedupKey));
    }

    /**
     * Mark email as sent in Redis (idempotency)
     */
    public void markAsSent(Long userId, String campaignType, LocalDate date, String emailId) {
        String dedupKey = String.format("%s%d:%s:%s",
            DEDUP_KEY_PREFIX, userId, campaignType, date);

        redisTemplate.opsForValue().set(dedupKey, emailId,
            Duration.ofDays(DEDUP_TTL_DAYS));

        log.debug("Marked email as sent: {}", dedupKey);
    }

    /**
     * Generate unique tracking token for attribution
     */
    private String generateTrackingToken(Long userId, Long campaignId) {
        String tokenData = String.format("%d:%d:%d:%s",
            userId, campaignId, System.currentTimeMillis(), UUID.randomUUID());

        return Base64.getEncoder().encodeToString(tokenData.getBytes(StandardCharsets.UTF_8));
    }

    /**
     * Render Thymeleaf template with context
     */
    private String renderTemplate(String templateString, Map<String, Object> context) {
        try {
            org.thymeleaf.context.Context ctx = new org.thymeleaf.context.Context();
            ctx.setVariables(context);
            return thymeleafEngine.process(templateString, ctx);
        } catch (Exception e) {
            log.error("Failed to render template", e);
            throw new TemplateRenderingException("Template rendering failed", e);
        }
    }

    private User fetchUser(Long userId) {
        String cacheKey = "user:profile:" + userId;

        // Try cache first
        User cachedUser = (User) redisTemplate.opsForValue().get(cacheKey);
        if (cachedUser != null) {
            return cachedUser;
        }

        // Fetch from database
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + userId));

        // Cache for 1 hour
        redisTemplate.opsForValue().set(cacheKey, user, Duration.ofHours(1));

        return user;
    }
}
```

### 3. Email Batch Sender (Spring Batch)

```java
@Configuration
@Slf4j
public class EmailBatchConfiguration {

    @Autowired private JobRepository jobRepository;
    @Autowired private PlatformTransactionManager transactionManager;
    @Autowired private CampaignTriggerRepository campaignTriggerRepository;
    @Autowired private EmailSendService emailSendService;

    @Bean
    public Job sendEmailBatchJob() {
        return new JobBuilder("sendEmailBatchJob", jobRepository)
            .start(partitionEmailJob())
            .build();
    }

    @Bean
    public Step partitionEmailJob() {
        return new StepBuilder("partitionEmailStep", jobRepository)
            .partitioner(masterStep(), emailPartitioner())
            .step(workerStep())
            .gridSize(10) // 10 parallel threads
            .taskExecutor(emailBatchTaskExecutor())
            .build();
    }

    @Bean
    public Step masterStep() {
        return new StepBuilder("masterStep", jobRepository)
            .build();
    }

    @Bean
    public Step workerStep() {
        return new StepBuilder("workerStep", jobRepository)
            .<CampaignTrigger, EmailBatch>chunk(100000, transactionManager) // 100k emails per batch
            .reader(campaignTriggerReader())
            .processor(emailBatchProcessor())
            .writer(emailBatchWriter())
            .faultTolerant()
            .retry(Exception.class)
            .retryLimit(3)
            .skip(Exception.class)
            .skipLimit(1000)
            .listener(new JobExecutionListener())
            .build();
    }

    @Bean
    public Partitioner emailPartitioner() {
        return gridSize -> {
            Map<String, ExecutionContext> partitions = new HashMap<>();
            long totalTriggers = campaignTriggerRepository.countPending();
            long partitionSize = totalTriggers / gridSize;

            for (int i = 0; i < gridSize; i++) {
                ExecutionContext context = new ExecutionContext();
                context.putLong("partitionNumber", i);
                context.putLong("minId", i * partitionSize);
                context.putLong("maxId", (i + 1) * partitionSize);
                partitions.put("partition" + i, context);
            }
            return partitions;
        };
    }

    @Bean
    public JdbcPagingItemReader<CampaignTrigger> campaignTriggerReader() {
        return new JdbcPagingItemReaderBuilder<CampaignTrigger>()
            .name("campaignTriggerReader")
            .dataSource(dataSource())
            .selectClause("SELECT *")
            .fromClause("FROM campaign_triggers")
            .whereClause("status = 'PENDING' AND scheduled_send_time <= NOW()")
            .orderByClause("campaign_trigger_id ASC")
            .pageSize(10000)
            .rowMapper(campaignTriggerRowMapper())
            .build();
    }

    @Bean
    public ItemProcessor<CampaignTrigger, EmailBatch> emailBatchProcessor() {
        return trigger -> {
            // Load full context for email
            PersonalizedEmail personalizedEmail = emailPersonalizationService
                .renderPersonalizedEmail(
                    trigger.getCampaignId(),
                    trigger.getUserId(),
                    emailTemplateRepository.findById(trigger.getTemplateId()).orElse(null),
                    trigger.getPersonalizationData()
                );

            return EmailBatch.builder()
                .campaignTriggerId(trigger.getId())
                .personalizedEmail(personalizedEmail)
                .processedAt(Instant.now())
                .build();
        };
    }

    @Bean
    public ItemWriter<EmailBatch> emailBatchWriter() {
        return emails -> {
            // Rate-limited batch send via SES
            emailSendService.sendBatch(emails);
        };
    }

    @Bean
    public TaskExecutor emailBatchTaskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(20);
        executor.setQueueCapacity(1000);
        executor.setThreadNamePrefix("email-batch-");
        executor.initialize();
        return executor;
    }
}

@Service
@Slf4j
public class EmailSendService {

    @Autowired private AmazonSES amazonSES;
    @Autowired private RateLimiter emailRateLimiter;
    @Autowired private KafkaTemplate<String, String> kafkaTemplate;
    @Autowired private CampaignTriggerRepository campaignTriggerRepository;
    @Autowired private ObjectMapper objectMapper;

    private static final int BATCH_SIZE = 1000;
    private static final int MAX_EMAILS_PER_SECOND = 10000;

    /**
     * Send batch of personalized emails with rate limiting
     */
    public void sendBatch(List<EmailBatch> emailBatches) {
        log.info("Starting batch send for {} emails", emailBatches.size());

        for (int i = 0; i < emailBatches.size(); i += BATCH_SIZE) {
            int end = Math.min(i + BATCH_SIZE, emailBatches.size());
            List<EmailBatch> subBatch = emailBatches.subList(i, end);

            sendSubBatch(subBatch);
        }
    }

    private void sendSubBatch(List<EmailBatch> subBatch) {
        for (EmailBatch emailBatch : subBatch) {
            // Rate limiting: 10,000 emails/sec = 100µs per email
            emailRateLimiter.acquire();

            try {
                PersonalizedEmail email = emailBatch.getPersonalizedEmail();

                // Build SES SendEmail request
                SendEmailRequest request = new SendEmailRequest()
                    .withSource("noreply@example.com")
                    .withDestination(new Destination()
                        .withToAddresses(email.getRecipientEmail()))
                    .withMessage(new Message()
                        .withSubject(new Content(email.getSubject()))
                        .withBody(new Body()
                            .withHtml(new Content(email.getHtmlBody()))
                            .withText(new Content(email.getTextBody() != null ?
                                email.getTextBody() : ""))));

                // Idempotency header
                request.addCustomHeaderEntry("X-Email-ID", email.getEmailId());

                // Send via SES
                SendEmailResult result = amazonSES.sendEmail(request);

                log.info("Email sent successfully: {}", result.getMessageId());

                // Publish email.sent event to Kafka
                publishEmailSentEvent(email, result);

                // Update campaign trigger status
                campaignTriggerRepository.updateStatus(
                    emailBatch.getCampaignTriggerId(),
                    "SENT",
                    result.getMessageId()
                );

            } catch (Exception e) {
                log.error("Failed to send email", e);

                // Publish to DLQ for retry
                publishEmailFailedEvent(emailBatch, e);
            }
        }
    }

    private void publishEmailSentEvent(PersonalizedEmail email, SendEmailResult result) {
        try {
            EmailSentEvent event = EmailSentEvent.builder()
                .emailId(email.getEmailId())
                .campaignId(email.getCampaignId())
                .userId(email.getUserId())
                .recipientEmail(email.getRecipientEmail())
                .sesMessageId(result.getMessageId())
                .sentAt(Instant.now())
                .build();

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("email.sent", email.getEmailId(), eventJson);

        } catch (JsonProcessingException e) {
            log.error("Failed to publish email.sent event", e);
        }
    }

    private void publishEmailFailedEvent(EmailBatch emailBatch, Exception e) {
        try {
            EmailFailedEvent event = EmailFailedEvent.builder()
                .emailId(emailBatch.getPersonalizedEmail().getEmailId())
                .campaignId(emailBatch.getPersonalizedEmail().getCampaignId())
                .userId(emailBatch.getPersonalizedEmail().getUserId())
                .failureReason(e.getMessage())
                .failedAt(Instant.now())
                .retryAttempt(0)
                .build();

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("email.campaign.failed",
                String.valueOf(emailBatch.getPersonalizedEmail().getUserId()), eventJson);

        } catch (JsonProcessingException ex) {
            log.error("Failed to publish email.failed event", ex);
        }
    }
}
```

### 4. Email Tracking Service (Kafka Consumer)

```java
@Service
@Slf4j
public class EmailTrackingService {

    @Autowired private MongoTemplate mongoTemplate;
    @Autowired private EmailEngagementRepository engagementRepository;
    @Autowired private KafkaTemplate<String, String> kafkaTemplate;
    @Autowired private ObjectMapper objectMapper;

    @KafkaListener(topics = "email.sent", groupId = "email-tracking-service")
    public void trackEmailSent(ConsumerRecord<String, String> record) {
        try {
            EmailSentEvent event = objectMapper.readValue(record.value(), EmailSentEvent.class);

            // Store in MongoDB for analytics
            EmailEngagement engagement = EmailEngagement.builder()
                .emailId(event.getEmailId())
                .eventType("SENT")
                .campaignId(event.getCampaignId())
                .userId(event.getUserId())
                .recipientEmail(event.getRecipientEmail())
                .timestamp(event.getSentAt())
                .build();

            mongoTemplate.save(engagement, "email_engagements");

            log.info("Email sent tracked: {}", event.getEmailId());

        } catch (JsonProcessingException e) {
            log.error("Failed to parse email.sent event", e);
        }
    }

    /**
     * REST endpoint called when user opens email (via pixel)
     */
    @GetMapping("/tracking/open")
    public void trackOpen(@RequestParam String t, HttpServletRequest request) {
        try {
            String token = URLDecoder.decode(t, StandardCharsets.UTF_8);
            Map<String, String> tokenData = decodeTrackingToken(token);

            String emailId = tokenData.get("emailId");
            long userId = Long.parseLong(tokenData.get("userId"));
            long campaignId = Long.parseLong(tokenData.get("campaignId"));

            // Record open event
            EmailEngagement engagement = EmailEngagement.builder()
                .emailId(emailId)
                .eventType("OPEN")
                .userId(userId)
                .campaignId(campaignId)
                .timestamp(Instant.now())
                .userAgent(request.getHeader("User-Agent"))
                .ipAddress(getClientIp(request))
                .deviceType(detectDeviceType(request.getHeader("User-Agent")))
                .build();

            mongoTemplate.save(engagement, "email_engagements");

            // Publish engagement event to Kafka
            EmailEngagementEvent engagementEvent = EmailEngagementEvent.builder()
                .emailId(emailId)
                .eventType("OPEN")
                .userId(userId)
                .campaignId(campaignId)
                .timestamp(Instant.now())
                .build();

            String eventJson = objectMapper.writeValueAsString(engagementEvent);
            kafkaTemplate.send("email.engagement", emailId, eventJson);

            // Return 1x1 transparent pixel
            response.setContentType("image/gif");
            response.getOutputStream().write(new byte[]{
                0x47, 0x49, 0x46, 0x38, 0x39, 0x61, 0x01, 0x00,
                0x01, 0x00, 0x80, 0x00, 0x00, 0xFF, 0xFF, 0xFF,
                0x00, 0x00, 0x00, 0x2C, 0x00, 0x00, 0x00, 0x00,
                0x01, 0x00, 0x01, 0x00, 0x00, 0x02, 0x02, 0x44, 0x01, 0x00, 0x3B
            });

        } catch (Exception e) {
            log.error("Failed to track open event", e);
        }
    }

    /**
     * REST endpoint called when user clicks tracked link
     */
    @GetMapping("/tracking/click")
    public void trackClick(@RequestParam String t, @RequestParam String url,
                          HttpServletRequest request, HttpServletResponse response)
            throws IOException {
        try {
            String token = URLDecoder.decode(t, StandardCharsets.UTF_8);
            Map<String, String> tokenData = decodeTrackingToken(token);

            String emailId = tokenData.get("emailId");
            long userId = Long.parseLong(tokenData.get("userId"));
            long campaignId = Long.parseLong(tokenData.get("campaignId"));

            // Record click event
            EmailEngagement engagement = EmailEngagement.builder()
                .emailId(emailId)
                .eventType("CLICK")
                .userId(userId)
                .campaignId(campaignId)
                .clickUrl(url)
                .timestamp(Instant.now())
                .userAgent(request.getHeader("User-Agent"))
                .ipAddress(getClientIp(request))
                .deviceType(detectDeviceType(request.getHeader("User-Agent")))
                .build();

            mongoTemplate.save(engagement, "email_engagements");

            // Publish click event
            EmailEngagementEvent engagementEvent = EmailEngagementEvent.builder()
                .emailId(emailId)
                .eventType("CLICK")
                .userId(userId)
                .campaignId(campaignId)
                .clickUrl(url)
                .timestamp(Instant.now())
                .build();

            String eventJson = objectMapper.writeValueAsString(engagementEvent);
            kafkaTemplate.send("email.engagement", emailId, eventJson);

            // Redirect to actual URL
            response.sendRedirect(url);

        } catch (Exception e) {
            log.error("Failed to track click event", e);
            response.sendError(HttpServletResponse.SC_BAD_REQUEST);
        }
    }

    private Map<String, String> decodeTrackingToken(String token) {
        byte[] decodedBytes = Base64.getDecoder().decode(token);
        String decoded = new String(decodedBytes, StandardCharsets.UTF_8);

        String[] parts = decoded.split(":");
        return Map.of(
            "userId", parts[0],
            "campaignId", parts[1],
            "timestamp", parts[2],
            "emailId", parts[3]
        );
    }

    private String detectDeviceType(String userAgent) {
        if (userAgent == null) return "UNKNOWN";
        if (userAgent.contains("Mobile")) return "MOBILE";
        if (userAgent.contains("Tablet")) return "TABLET";
        return "DESKTOP";
    }

    private String getClientIp(HttpServletRequest request) {
        String[] headers = {
            "X-Forwarded-For",
            "Proxy-Client-IP",
            "WL-Proxy-Client-IP",
            "HTTP_X_FORWARDED_FOR",
            "HTTP_X_FORWARDED",
            "HTTP_X_FORWARDED_HOST"
        };

        for (String header : headers) {
            String ip = request.getHeader(header);
            if (ip != null && !ip.isEmpty() && !"unknown".equalsIgnoreCase(ip)) {
                return ip.split(",")[0];
            }
        }

        return request.getRemoteAddr();
    }
}
```

### 5. Campaign Metrics Aggregator (Scheduled Task)

```java
@Service
@Slf4j
public class CampaignMetricsAggregator {

    @Autowired private MongoTemplate mongoTemplate;
    @Autowired private JdbcTemplate jdbcTemplate;
    @Autowired private EmailEngagementSummaryRepository summaryRepository;

    @Scheduled(fixedDelay = 300000, initialDelay = 60000) // Every 5 minutes
    public void aggregateMetricsHourly() {
        log.info("Starting hourly metrics aggregation");

        // Get the start of the current hour
        LocalDateTime hourStart = LocalDateTime.now()
            .truncatedTo(ChronoUnit.HOURS);
        LocalDateTime hourEnd = hourStart.plusHours(1);

        Instant hourStartInstant = hourStart.atZone(ZoneId.systemDefault()).toInstant();
        Instant hourEndInstant = hourEnd.atZone(ZoneId.systemDefault()).toInstant();

        // Query MongoDB for engagements in this hour
        Query query = new Query();
        query.addCriteria(Criteria.where("timestamp")
            .gte(hourStartInstant)
            .lt(hourEndInstant));

        List<EmailEngagement> engagements = mongoTemplate.find(
            query, EmailEngagement.class, "email_engagements");

        // Group by campaign and calculate metrics
        Map<Long, CampaignMetrics> metricsMap = new HashMap<>();

        for (EmailEngagement engagement : engagements) {
            CampaignMetrics metrics = metricsMap.computeIfAbsent(
                engagement.getCampaignId(),
                k -> new CampaignMetrics(k, hourStart)
            );

            metrics.increment(engagement.getEventType());
        }

        // Fetch sent counts from PostgreSQL
        String sentCountQuery =
            "SELECT campaign_id, COUNT(*) as sent_count " +
            "FROM email_sends " +
            "WHERE sent_at >= ? AND sent_at < ? " +
            "GROUP BY campaign_id";

        List<Map<String, Object>> sentCounts = jdbcTemplate.queryForList(
            sentCountQuery,
            java.sql.Timestamp.from(hourStartInstant),
            java.sql.Timestamp.from(hourEndInstant)
        );

        for (Map<String, Object> row : sentCounts) {
            Long campaignId = ((Number) row.get("campaign_id")).longValue();
            Long sentCount = ((Number) row.get("sent_count")).longValue();

            CampaignMetrics metrics = metricsMap.computeIfAbsent(
                campaignId,
                k -> new CampaignMetrics(k, hourStart)
            );

            metrics.setSentCount(sentCount);
        }

        // Save summaries to PostgreSQL
        for (CampaignMetrics metrics : metricsMap.values()) {
            EmailEngagementSummary summary = EmailEngagementSummary.builder()
                .campaignId(metrics.getCampaignId())
                .hourBucket(Timestamp.valueOf(metrics.getHour()))
                .sentCount(metrics.getSentCount())
                .openCount(metrics.getOpenCount())
                .clickCount(metrics.getClickCount())
                .bounceCount(metrics.getBounceCount())
                .complaintCount(metrics.getComplaintCount())
                .openRate(metrics.calculateOpenRate())
                .clickRate(metrics.calculateClickRate())
                .build();

            summaryRepository.save(summary);

            log.info("Aggregated metrics for campaign {}: {} sent, {} opens, {} clicks",
                metrics.getCampaignId(), metrics.getSentCount(),
                metrics.getOpenCount(), metrics.getClickCount());
        }
    }

    @Data
    @AllArgsConstructor
    private static class CampaignMetrics {
        private Long campaignId;
        private LocalDateTime hour;
        private Long sentCount = 0L;
        private Long openCount = 0L;
        private Long clickCount = 0L;
        private Long bounceCount = 0L;
        private Long complaintCount = 0L;

        public void increment(String eventType) {
            switch (eventType) {
                case "OPEN" -> openCount++;
                case "CLICK" -> clickCount++;
                case "BOUNCE" -> bounceCount++;
                case "COMPLAINT" -> complaintCount++;
            }
        }

        public BigDecimal calculateOpenRate() {
            if (sentCount == 0) return BigDecimal.ZERO;
            return BigDecimal.valueOf(openCount * 100)
                .divide(BigDecimal.valueOf(sentCount), 2, RoundingMode.HALF_UP);
        }

        public BigDecimal calculateClickRate() {
            if (openCount == 0) return BigDecimal.ZERO;
            return BigDecimal.valueOf(clickCount * 100)
                .divide(BigDecimal.valueOf(openCount), 2, RoundingMode.HALF_UP);
        }
    }
}
```

---

## Failure Scenarios & Mitigations

| Failure Scenario | Impact | Mitigation |
|------------------|--------|-----------|
| **SES API Rate Limit (429)** | Batch send paused, emails delayed | Kafka DLQ with exponential backoff, circuit breaker pattern, fallback to SendGrid |
| **SES Account Bounce Limit** | Account disabled, no emails sent | Pre-flight validation, bounce rate monitoring, dedicated IPs for high-volume senders |
| **Redis Dedup Cache Down** | Risk of duplicate emails (catastrophic) | Kafka offset management, dual-write to PostgreSQL, manual dedup on recovery |
| **PostgreSQL Connection Pool Exhaustion** | Database unavailable, no campaign triggers processed | Connection pooling (HikariCP), circuit breaker for DB calls, queue bursting |
| **Kafka Broker Failure** | Event pipeline blocked, triggers not processed | 3-node Kafka cluster, topic replication, consumer offset management |
| **MongoDB Engagement Write Failure** | Tracking data lost, metrics incomplete | Write-ahead logs in PostgreSQL, async MongoDB writes with fallback |
| **Network Latency to Thymeleaf Rendering** | Email send latency increases > 500ms | In-process template caching, pre-compiled templates, async rendering |
| **Email Template Syntax Error** | Batch fails, emails not sent | Template validation on upload, version control, canary deployments |
| **User Preference Cache Stale** | Wrong emails sent (e.g., to opted-out users) | Database fallback lookup, 1-hour TTL, batch verification |
| **Dedup TTL Expiration** | Duplicates possible after 7 days (acceptable) | Longer TTL for critical campaigns, manual re-send prevention |
| **Tracking URL Generation Failure** | Untrackable emails sent | Fallback to generic tracking URL, manual audit trail |
| **Email Personalization Timeout** | Email sent with missing data | Default fallback values, template validation |

---

## Scaling Strategy

### Horizontal Scaling

**Campaign Trigger Service:**
- Scale: 1 consumer per Kafka partition (50 partitions = 50 instances max)
- Consumer group: `email-trigger-service`
- Rebalancing: Automatic on instance addition

**Email Batch Sender:**
- Scale: 1 Spring Batch job per 10 million emails
- Partitions: 100 (each handler 100k emails)
- Threads: 10 parallel workers per instance

**Email Tracking Service:**
- Scale: 1 consumer per Kafka partition (100 partitions = 100 instances max)
- Handles 92 engagement events/sec (highly scalable)

### Vertical Scaling

**Redis:**
- Current: 35GB for 7-day dedup keys
- Scaling: Redis Cluster with 6 nodes (3 primary, 3 replica)
- Eviction: LRU if memory exceeds

**PostgreSQL:**
- Current: 1TB/year storage
- Scaling: Read replicas for reporting, write master for campaigns
- Partitioning: Partition `email_sends` by date for faster queries

**MongoDB:**
- Current: 580GB/year storage
- Scaling: Sharded cluster on `user_id` and `timestamp`
- TTL Index: Auto-delete documents > 90 days old

**Kafka:**
- Brokers: 5+ brokers for fault tolerance
- Topic Replication Factor: 3
- Partition Count: 50-100 per topic

### Load Balancing

```
SES/SendGrid Rate Limit: 10,000 emails/sec
├─ Consumer 1: 2,500 emails/sec (25% quota)
├─ Consumer 2: 2,500 emails/sec (25% quota)
├─ Consumer 3: 2,500 emails/sec (25% quota)
└─ Consumer 4: 2,500 emails/sec (25% quota)

Kafka Partitioner: Hash(userId) → 100 partitions
```

---

## Monitoring & Observability

### Metrics to Track

```
# Campaign Trigger Service
- trigger_events_consumed_total (counter, by trigger type)
- trigger_processing_duration_seconds (histogram)
- trigger_dedup_hits_total (counter)
- trigger_enqueue_failures_total (counter)

# Email Personalization Service
- personalization_render_duration_seconds (histogram)
- personalization_cache_hits_total (counter)
- personalization_failures_total (counter)

# Email Batch Sender
- email_send_duration_seconds (histogram, by SES/SendGrid)
- email_send_success_total (counter)
- email_send_failure_total (counter)
- email_rate_limit_triggered_total (counter)

# Email Tracking Service
- engagement_events_consumed_total (counter, by event type)
- engagement_processing_duration_seconds (histogram)
- engagement_persistence_failures_total (counter)

# Campaign Metrics
- campaign_open_rate (gauge, by campaign)
- campaign_click_rate (gauge, by campaign)
- campaign_bounce_rate (gauge, by campaign)
```

### Dashboards

1. **Campaign Health Dashboard**
   - Real-time sent/open/click counts
   - Open rate & click rate trends
   - Bounce rate alerts
   - By campaign and by trigger type

2. **System Performance Dashboard**
   - Email send latency (p50, p95, p99)
   - Kafka consumer lag
   - Redis hit rate
   - Database query performance

3. **Failure & Retry Dashboard**
   - DLQ message count
   - Retry rate by error type
   - Failed email destination
   - SES account bounce rate

### Alerting Rules

```
ALERT: CampaignTriggerServiceLag
IF kafka_consumer_lag > 100000 FOR 5m

ALERT: EmailSendFailureRate
IF rate(email_send_failure_total[5m]) > 0.02

ALERT: HighEmailLatency
IF histogram_quantile(0.95, email_send_duration_seconds) > 0.5

ALERT: RedisDown
IF redis_up == 0

ALERT: PostgreSQLSlowQueries
IF pg_slow_query_count > 10 FOR 5m

ALERT: MongoDBWriteErrors
IF mongodb_write_errors_total > 1000
```

---

## Summary Cheat Sheet

### Key Thresholds

| Component | Threshold | Action |
|-----------|-----------|--------|
| Email send latency (p95) | 500ms | Page on-call if exceeded |
| Campaign trigger enqueue | < 1 sec | Async processing via Kafka |
| Dedup cache hit rate | > 95% | Indicates effective deduplication |
| Email delivery success rate | > 98% | Investigate failures > 2% |
| Email open rate | 15-25% (avg) | Compare across campaigns |
| Email click rate | 2-5% (avg) | Track conversion attribution |
| Kafka consumer lag | < 10k messages | Critical lag alert threshold |
| Redis memory usage | < 80GB | Evict old keys or add nodes |

### Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| **Kafka for events** | Decouples services, enables replay, scale independently |
| **Redis for dedup** | Sub-millisecond lookups, TTL-based expiration |
| **PostgreSQL for campaigns** | ACID compliance, complex queries, historical tracking |
| **MongoDB for engagement** | Flexible schema, horizontal scalability, time-series data |
| **Spring Batch for sending** | Chunked processing, fault tolerance, partitioned execution |
| **Thymeleaf for templating** | Type-safe, Spring-native, in-process rendering |
| **Tracking pixels + URLs** | Non-intrusive open detection, click attribution |

### Tech Stack Justification

- **Java 17:** Latest LTS, strong typing, performance optimizations
- **Spring Boot 3:** Production-ready framework, dependency injection, auto-configuration
- **PostgreSQL:** Relational schema, ACID, complex queries for campaigns
- **MongoDB:** Document storage for flexible engagement events
- **Redis:** In-memory for sub-millisecond deduplication
- **Kafka:** Event streaming, exactly-once semantics (with coordination)
- **Spring Batch:** Enterprise batch processing with built-in partitioning
- **Thymeleaf:** Template engine integrated with Spring, type-safe
- **AWS SES:** Reliable email delivery, cost-effective at scale

### Costs Estimation (Monthly)

```
Email Sends: 10M/day * 30 days = 300M emails/month
├─ SES Cost: 300M * $0.0001 = $30,000

Infrastructure:
├─ Kafka (5 nodes): $500/month
├─ Redis Cluster (6 nodes): $2,000/month
├─ PostgreSQL (24GB RAM): $3,000/month
├─ MongoDB (100GB storage): $1,500/month
├─ Spring Boot Instances (10 instances): $5,000/month

Total Monthly: ~$41,500 for 300M emails
Cost per email: $0.000138
```
