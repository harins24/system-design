---
title: Fraud Detection System
layout: default
---

# Fraud Detection System — Deep Dive Design

> **Scenario:** Build a real-time fraud detection system for an e-commerce platform. Requirements include detecting suspicious orders using velocity checks, location anomalies, stolen cards, and other signals. Each order must be scored 0-100 (>80 = high risk, auto-blocked; 40-80 = flagged for manual review). Must score orders in <200ms during checkout without blocking legitimate customers.
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
- Score each order in real-time using rules-based and ML-based signals
- Final score: 0-100 (>80 = BLOCK, 40-80 = REVIEW, <40 = APPROVE)
- Combine rules-based rules (velocity, address mismatch) with ML model scoring
- Collect 20+ signals: IP, device fingerprint, shipping address, card BIN, user history
- Prevent fraud losses while maintaining < 1% false positive rate (block legitimate customers)
- Support manual review queue for fraud analysts with approve/reject feedback
- Enable ML model retraining with analyst feedback
- Track fraud metrics: block rate, false positive rate, fraud catch rate

### Non-Functional Requirements
- Scoring latency: < 200ms (95th percentile during checkout)
- Scoring throughput: 10,000 orders/second peak
- Scoring accuracy: > 95% (based on downstream fraud labels)
- ML model inference latency: < 50ms
- Rules engine latency: < 30ms
- Manual review queue response time: < 2 hours (SLA)
- System availability: 99.99% (no scoring = auto-APPROVE for checkout experience)

### Constraints
- Cannot block checkout experience (synchronous scoring only)
- Limited ML model inference capacity (cost/latency tradeoff)
- Rules must be updateable without redeployment
- Feedback loop from analysts limited to ~10k reviews/day
- Historical data retention: 2 years

---

## Capacity Estimation

| Metric | Calculation | Value |
|--------|-------------|-------|
| **Daily Orders** | Peak season estimate | 1,000,000 orders |
| **Orders/Second (Avg)** | 1M / 86400s | ~11.6 orders/sec |
| **Orders/Second (Peak)** | Assuming 10x during sales | ~116 orders/sec |
| **Scoring Requests** | 1-to-1 with orders | 116 req/sec peak |
| **ML Model Inference Calls** | 100% of orders use ML | 116 inferences/sec |
| **Rules Evaluation** | 100% of orders | 116 rules/sec |
| **Velocity Check Lookups** | 3 lookups per order (user, card, IP) | 348 Redis ops/sec |
| **False Positives/Day** | 1% of approved orders | ~10k orders blocked by mistake |
| **Manual Reviews/Day** | 40-80 threshold range | ~15-30k orders flagged |
| **Analyst Feedback/Day** | Limited capacity | ~10k reviews max |
| **Storage (PostgreSQL - Scores)** | 1M orders/day * 365 days | ~365M rows/year |
| **Storage (MongoDB - Signals)** | 1M orders/day * 50KB signal data | ~50GB/day = 18.25TB/year |

---

## High-Level Architecture

```
┌────────────────────────────────┐
│    Checkout Service            │
│    (Synchronous Request)       │
└────────────────┬───────────────┘
                 │ (CreateOrder)
                 ▼
    ┌────────────────────────────┐
    │  Fraud Scoring Service     │
    │  (Real-Time, < 200ms)      │
    └────────┬───────────────────┘
             │
    ┌────────┴──────────┬──────────┬──────────┐
    │                   │          │          │
    ▼                   ▼          ▼          ▼
┌─────────┐      ┌──────────┐ ┌──────┐ ┌──────────┐
│ Rules   │      │ ML Model │ │Redis │ │ Postgres │
│ Engine  │      │ Inference│ │Velocity││ User History
└─────────┘      └──────────┘ └──────┘ └──────────┘
    │                   │          │          │
    └────────┬──────────┴──────────┴──────────┘
             │
             ▼
    ┌──────────────────────┐
    │ Score Aggregator     │
    │ (Weighted Average)   │
    └────────┬─────────────┘
             │
    ┌────────┴───────────┐
    │                    │
    ▼                    ▼
┌─────────────┐  ┌──────────────┐
│ APPROVE     │  │ BLOCK/REVIEW │
│ (< 40)      │  │ (40+)        │
└──────┬──────┘  └──────┬───────┘
       │                │
       ▼                ▼
   ┌────────┐  ┌────────────────┐
   │Checkout │  │ Manual Review  │
   │Completes│  │ Queue (Kafka)  │
   └────────┘  └────────┬───────┘
                        │
                        ▼
                ┌─────────────────┐
                │ Fraud Analyst   │
                │ Dashboard       │
                │ Approve/Reject  │
                └────────┬────────┘
                         │
                         ▼
                  ┌──────────────────┐
                  │ Feedback Loop    │
                  │ (ML Retraining)  │
                  └──────────────────┘
```

---

## Core Design Questions Answered

### 1. What signals do you use for fraud detection?

**Answer:** Combine 20+ signals across 5 categories: user behavior, transaction patterns, device/IP, card/payment, and address/shipping.

**Signal Categories:**

| Category | Signals | Rationale |
|----------|---------|-----------|
| **User Behavior** | Account age, order frequency, login velocity, new device, new location | Long-standing accounts less likely to be compromised |
| **Transaction** | Order amount, unusual items, high-value products, quantity ordered | Stolen cards = large/bulk purchases |
| **Device/Network** | Device fingerprint (CPU, Browser, Canvas), IP geolocation, new IP, IP risk score | Same device = low risk; new IP from risky geo = high risk |
| **Card/Payment** | Card type (credit/debit), BIN (Bank Identification Number), card issuer country, previous card | Stolen cards common in certain BIN ranges; mismatch geography |
| **Address/Shipping** | Address verification (AVS), shipping address ≠ billing address, risky countries, ZIP code distance | Address mismatches indicate fraud; high-risk geographies |

**Feature Matrix:**

```java
public record FraudSignals(
    // User Behavior (5 signals)
    long accountAgeInDays,
    int orderCountLastMonth,
    int loginVelocityLastHour,  // # logins
    boolean isNewDevice,
    boolean isNewLocation,

    // Transaction (4 signals)
    BigDecimal orderAmount,
    boolean hasHighValueItems,
    int itemQuantity,
    String categoryType,  // "electronics", "jewelry", etc.

    // Device/Network (5 signals)
    String deviceFingerprint,
    String ipAddress,
    String ipCountry,
    boolean isNewIp,
    double ipRiskScore,  // 0-1

    // Card/Payment (4 signals)
    String cardBin,
    String cardIssuerCountry,
    boolean isNewCard,
    int previousCardUseCount,

    // Address/Shipping (2 signals)
    String avsResult,  // "M" (match), "N" (no match), etc.
    boolean isAddressMismatch,
    boolean isHighRiskCountry
) {}
```

### 2. How do you score orders in real-time without blocking checkout?

**Answer:** Use asynchronous + synchronous hybrid approach:
- **Synchronous path:** Rules-based checks (fast, < 30ms) + cached ML prediction
- **Asynchronous path:** Full ML model inference happens post-checkout, async feedback to decision

**Scoring Pipeline:**
```
T+0ms: Order created, fraud scoring starts
T+30ms: Rules engine evaluates (velocity, BIN, etc.) → rules_score
T+50ms: ML inference completes (cached model) → ml_score
T+100ms: Aggregate scores, return decision to checkout
T+0.5-5s: Full ML inference (expensive model) happens async, updates decision if needed
T+1min: Decision finalized, order proceeds
```

### 3. How do you combine rules-based and ML-based scoring?

**Answer:** Weighted aggregation with fallback logic.

```
final_score = 0.4 * rules_score + 0.6 * ml_score

Where:
  rules_score = 0-100 (calculated synchronously)
  ml_score = 0-100 (from pre-trained model)

Decision Logic:
  if final_score > 80 → BLOCK (high confidence fraud)
  if 40 <= final_score <= 80 → REVIEW (manual investigation)
  if final_score < 40 → APPROVE (low risk)

Override Rules:
  if account_age > 5 years AND order_count > 100 → max_score = 60 (whitelist)
  if order_amount > $5000 AND is_first_time → force REVIEW
  if is_corporate_account → lower thresholds (more stringent)
```

### 4. How do you handle false positives (blocking legitimate customers)?

**Answer:** Multi-layered approach: whitelisting, risk-based thresholds, analyst review, and A/B testing.

**False Positive Mitigation:**

1. **Whitelist Rules:**
   - Account age > 5 years, no fraud history → max_score = 50 (never auto-block)
   - Corporate accounts with verified identity → max_score = 30
   - Prior fraud claims filed → all orders from this customer reviewed

2. **Dynamic Thresholds:**
   - Day-of-week adjustment (e.g., weekend orders higher threshold)
   - Seasonal adjustment (e.g., holidays more lenient)
   - Customer segment adjustment (e.g., VIP tier: +10 tolerance)

3. **Risk-Based Delays:**
   - Score 40-60: Allow order, require email verification
   - Score 60-75: Allow order with 2FA or phone callback
   - Score 75-80: Manual review required before ship

4. **Analyst Feedback Loop:**
   - Disputed charges collected from customers
   - Analyst confirms: legitimate or fraud
   - Feedback used to retrain ML model

5. **Chargeback Analysis:**
   - When chargeback occurs 30-60 days later, retroactively label order
   - Update analyst feedback scores for model retraining

### 5. How do you implement a manual review queue for analysts?

**Answer:** Kafka topic + MongoDB for queue state + Spring Boot REST API for analyst UI.

**Queue Design:**
- Topic: `fraud.review_queue` (partitioned by fraud_score for priority)
- State: MongoDB document with order details + analyst notes
- API: CRUD endpoints for claim/review/verdict
- UI: Analyst dashboard showing orders sorted by risk score

### 6. How do you retrain ML models with feedback from analysts?

**Answer:** Weekly batch retraining pipeline with offline model evaluation.

**Retraining Process:**
```
T+0: Analysts provide feedback on 10k orders (approved/rejected)
T+7d: Weekly retraining job runs
  1. Fetch labeled dataset from MongoDB
  2. Engineer features from raw signals
  3. Train XGBoost/LightGBM model
  4. Evaluate on holdout test set
  5. If accuracy > prior model, push to production
  6. Shadow traffic on new model for validation
  7. Gradual rollout (10% → 50% → 100%)
T+14d: New model fully deployed
```

---

## Microservices Breakdown

| Service | Responsibility | Tech Stack | Latency |
|---------|-----------------|-------------|---------|
| **Fraud Scoring Service** | Orchestrates scoring, aggregates signals | Spring Boot | < 200ms |
| **Rules Engine** | Evaluates fraud rules (velocity, BIN, etc.) | Spring Boot + Custom DSL | < 30ms |
| **Velocity Checker** | Checks user/card/IP velocity from Redis | Spring Boot + Jedis | < 10ms |
| **ML Model Inference** | Calls XGBoost model API for predictions | Spring Boot + Java-ML library | < 50ms |
| **Signal Collection Service** | Gathers signals from various sources | Spring Boot | < 100ms |
| **Manual Review Service** | Manages review queue and analyst feedback | Spring Boot + Kafka | Real-time |
| **Model Retraining Service** | Weekly ML model retraining pipeline | Python + Spark | Batch |
| **Fraud Analyst API** | REST API for analyst dashboard | Spring Boot REST | Admin-level |

---

## Database Design

### PostgreSQL Schema: Fraud Decisions & History

```sql
-- Orders with fraud decisions
CREATE TABLE fraud_orders (
  order_id BIGINT PRIMARY KEY,
  user_id BIGINT NOT NULL,
  email VARCHAR(255) NOT NULL,
  total_amount DECIMAL(12, 2) NOT NULL,

  -- Fraud Scoring
  rules_score SMALLINT DEFAULT 0,
  ml_score SMALLINT DEFAULT 0,
  final_score SMALLINT NOT NULL,

  -- Decision
  decision VARCHAR(20) NOT NULL, -- 'APPROVE', 'REVIEW', 'BLOCK'
  decision_timestamp TIMESTAMP DEFAULT NOW(),
  decision_reason VARCHAR(500),

  -- Resolution
  analyst_decision VARCHAR(20), -- 'APPROVE', 'REJECT'
  analyst_name VARCHAR(100),
  analyst_timestamp TIMESTAMP,
  analyst_notes TEXT,

  -- Outcome labels (for model retraining)
  was_fraud BOOLEAN, -- Discovered fraud label (chargeback, dispute)
  is_chargeback BOOLEAN DEFAULT FALSE,
  chargeback_timestamp TIMESTAMP,

  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),

  INDEX idx_user_id (user_id),
  INDEX idx_final_score (final_score),
  INDEX idx_decision (decision),
  INDEX idx_analyst_decision (analyst_decision),
  INDEX idx_created_at (created_at DESC),
  INDEX idx_was_fraud (was_fraud)
);

-- Signal history (for debugging and analysis)
CREATE TABLE fraud_signals (
  signal_id BIGINT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
  order_id BIGINT NOT NULL REFERENCES fraud_orders(order_id),
  user_id BIGINT NOT NULL,

  -- Signal categories
  account_age_days INT,
  order_count_last_month INT,
  login_velocity_last_hour INT,
  is_new_device BOOLEAN,
  is_new_location BOOLEAN,

  order_amount DECIMAL(12, 2),
  has_high_value_items BOOLEAN,
  item_quantity INT,
  category_type VARCHAR(50),

  device_fingerprint VARCHAR(255),
  ip_address VARCHAR(45),
  ip_country VARCHAR(2),
  is_new_ip BOOLEAN,
  ip_risk_score DECIMAL(3, 2),

  card_bin VARCHAR(6),
  card_issuer_country VARCHAR(2),
  is_new_card BOOLEAN,
  previous_card_use_count INT,

  avs_result VARCHAR(10),
  is_address_mismatch BOOLEAN,
  is_high_risk_country BOOLEAN,

  created_at TIMESTAMP DEFAULT NOW(),
  INDEX idx_order_id (order_id),
  INDEX idx_user_id (user_id)
);

-- Fraud rules (updatable without redeployment)
CREATE TABLE fraud_rules (
  rule_id BIGINT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
  rule_name VARCHAR(255) NOT NULL,
  rule_type VARCHAR(50), -- 'velocity', 'bin_check', 'address_match', etc.
  rule_expression TEXT NOT NULL, -- DSL expression
  score_adjustment SMALLINT DEFAULT 0,
  enabled BOOLEAN DEFAULT TRUE,
  priority INT DEFAULT 100,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  created_by VARCHAR(100),

  UNIQUE(rule_name),
  INDEX idx_enabled (enabled),
  INDEX idx_priority (priority DESC)
);

-- Model versions (track ML model deployments)
CREATE TABLE fraud_models (
  model_id BIGINT PRIMARY KEY GENERATED BY DEFAULT AS IDENTITY,
  model_name VARCHAR(100),
  model_version VARCHAR(20), -- "v1.2.3"
  model_path VARCHAR(500), -- S3 path to serialized model
  model_type VARCHAR(50), -- "XGBoost", "LightGBM", etc.

  -- Model performance
  accuracy DECIMAL(5, 4),
  precision DECIMAL(5, 4),
  recall DECIMAL(5, 4),
  f1_score DECIMAL(5, 4),
  auc_roc DECIMAL(5, 4),

  -- Deployment
  deployed_at TIMESTAMP,
  deployment_status VARCHAR(20), -- 'ACTIVE', 'STAGING', 'ARCHIVED'
  traffic_percentage SMALLINT DEFAULT 0, -- 0-100 (for gradual rollout)

  -- Training metadata
  training_date TIMESTAMP,
  training_samples_count BIGINT,
  feature_importance JSONB,

  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(model_version),
  INDEX idx_deployed_at (deployed_at DESC),
  INDEX idx_deployment_status (deployment_status)
);
```

### MongoDB Schema: Real-Time Fraud Signals & Review Queue

```javascript
// Collection: fraud_review_queue (manual review tasks)
db.createCollection("fraud_review_queue", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["_id", "order_id", "user_id", "final_score", "status"],
      properties: {
        _id: { bsonType: "objectId" },
        order_id: { bsonType: "long" },
        user_id: { bsonType: "long" },
        email: { bsonType: "string" },
        total_amount: { bsonType: "decimal" },

        final_score: { bsonType: "int", minimum: 0, maximum: 100 },
        rules_score: { bsonType: "int" },
        ml_score: { bsonType: "int" },

        status: { enum: ["PENDING", "CLAIMED", "REVIEWED", "RESOLVED"] },
        priority: { bsonType: "int" }, // 1=highest, 5=lowest

        claimed_by: { bsonType: "string" }, // analyst name
        claimed_at: { bsonType: "date" },

        analyst_decision: { enum: ["APPROVE", "REJECT"] },
        analyst_notes: { bsonType: "string" },
        reviewed_at: { bsonType: "date" },

        // Order details for analyst review
        order_details: {
          items: { bsonType: "array" },
          shipping_address: { bsonType: "object" },
          billing_address: { bsonType: "object" },
          payment_method: { bsonType: "string" },
          created_at: { bsonType: "date" }
        },

        // Top contributing signals
        top_risk_signals: { bsonType: "array" }, // e.g., ["new_device", "high_velocity"]

        created_at: { bsonType: "date" },
        updated_at: { bsonType: "date" }
      }
    }
  }
});

db.fraud_review_queue.createIndex({ "status": 1, "final_score": -1 });
db.fraud_review_queue.createIndex({ "claimed_by": 1, "status": 1 });
db.fraud_review_queue.createIndex({ "created_at": -1 });
db.fraud_review_queue.createIndex({ "analyst_decision": 1, "reviewed_at": -1 });

// Collection: fraud_signals_archive (detailed signal data for training)
db.createCollection("fraud_signals_archive", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["_id", "order_id", "user_id", "signals", "timestamp"],
      properties: {
        _id: { bsonType: "objectId" },
        order_id: { bsonType: "long" },
        user_id: { bsonType: "long" },

        signals: {
          bsonType: "object",
          properties: {
            account_age_days: { bsonType: "int" },
            order_count_last_month: { bsonType: "int" },
            login_velocity_last_hour: { bsonType: "int" },
            is_new_device: { bsonType: "bool" },
            is_new_location: { bsonType: "bool" },

            order_amount: { bsonType: "decimal" },
            has_high_value_items: { bsonType: "bool" },
            item_quantity: { bsonType: "int" },

            device_fingerprint: { bsonType: "string" },
            ip_address: { bsonType: "string" },
            ip_country: { bsonType: "string" },
            ip_risk_score: { bsonType: "double" },

            card_bin: { bsonType: "string" },
            avs_result: { bsonType: "string" },
            is_address_mismatch: { bsonType: "bool" }
          }
        },

        // Label for ML training
        label: { enum: ["FRAUD", "LEGITIMATE"] },
        label_confidence: { bsonType: "double" }, // 0-1
        label_source: { enum: ["CHARGEBACK", "ANALYST", "CUSTOMER_DISPUTE", "AUTO"] },

        timestamp: { bsonType: "date" },
        created_at: { bsonType: "date" }
      }
    }
  }
});

db.fraud_signals_archive.createIndex({ "label": 1, "timestamp": -1 });
db.fraud_signals_archive.createIndex({ "user_id": 1, "timestamp": -1 });
db.fraud_signals_archive.createIndex({ "order_id": 1 }, { unique: true });
db.fraud_signals_archive.createIndex({ "timestamp": 1 }, { expireAfterSeconds: 63072000 }); // 2-year TTL
```

---

## Redis Data Structures

```redis
# 1. Velocity Counters (prevent rapid-fire orders)
fraud:velocity:user:{userId}:{period}
  Type: String (Counter)
  Value: order count
  TTL: 1 hour (for "last hour" metrics)
  Example: fraud:velocity:user:12345:1h → "5"

fraud:velocity:card:{cardHash}:{period}
  Type: String (Counter)
  Value: order count
  TTL: 1 day
  Example: fraud:velocity:card:hash_abc123:1d → "2"

fraud:velocity:ip:{ipHash}:{period}
  Type: String (Counter)
  Value: order count
  TTL: 1 hour
  Example: fraud:velocity:ip:hash_xyz:1h → "10"

# 2. Device Fingerprint Cache
fraud:device:{deviceFingerprint}
  Type: Hash
  Fields: { last_user_id, last_ip, last_country, first_seen }
  TTL: 30 days

# 3. IP Risk Score Cache (from external IP geolocation service)
fraud:ip:risk:{ipAddress}
  Type: Hash
  Fields: { country, risk_score (0-1), is_vpn, is_proxy }
  TTL: 7 days

# 4. Card Velocity & History
fraud:card:{cardHash}:velocity
  Type: String
  Value: count
  TTL: 1 day

fraud:card:{cardHash}:users
  Type: Set
  Value: user IDs that used this card
  TTL: 90 days

# 5. User Account Status Cache
fraud:user:{userId}:status
  Type: Hash
  Fields: { account_age, order_count, has_dispute, is_whitelisted }
  TTL: 1 hour

# 6. Model Version Flag (for canary deployment)
fraud:model:version:active
  Type: String
  Value: "v1.2.3"
  TTL: No expiry (manual update)

fraud:model:version:traffic
  Type: Hash
  Fields: { "v1.2.3": 90, "v1.2.4": 10 } (traffic percentages)
  TTL: No expiry
```

---

## Kafka Event Flow

### Topics & Partitioning

```
Topic: order.created
  Partitions: 50 (by order_id)
  Retention: 7 days
  Event Schema:
    {
      "order_id": 123456,
      "user_id": 12345,
      "email": "user@example.com",
      "total_amount": 250.00,
      "items": [...],
      "shipping_address": { ... },
      "billing_address": { ... },
      "card_token": "tok_xxx",
      "device_fingerprint": "fp_xyz",
      "ip_address": "1.2.3.4",
      "user_agent": "Mozilla/5.0...",
      "created_at": "2026-04-01T10:00:00Z"
    }

Topic: fraud.scoring.completed
  Partitions: 50 (by order_id)
  Retention: 30 days
  Event Schema:
    {
      "order_id": 123456,
      "user_id": 12345,
      "final_score": 65,
      "decision": "REVIEW",
      "rules_score": 50,
      "ml_score": 75,
      "scoring_duration_ms": 145,
      "scored_at": "2026-04-01T10:00:00.150Z"
    }

Topic: fraud.review_queue (manual review items)
  Partitions: 10 (by priority + score)
  Retention: 30 days
  Event Schema:
    {
      "order_id": 123456,
      "user_id": 12345,
      "final_score": 65,
      "priority": 1,
      "top_risk_signals": ["new_device", "high_velocity"],
      "enqueued_at": "2026-04-01T10:00:01Z"
    }

Topic: fraud.analyst_feedback
  Partitions: 1 (for ordered processing)
  Retention: 2 years (for retraining)
  Event Schema:
    {
      "order_id": 123456,
      "user_id": 12345,
      "analyst_decision": "APPROVE",
      "analyst_notes": "Legitimate customer, verified with phone call",
      "analyst_name": "john.smith",
      "reviewed_at": "2026-04-01T10:15:00Z",
      "actual_fraud_label": "LEGITIMATE"
    }

Topic: fraud.chargeback_detected
  Partitions: 50 (by user_id)
  Retention: 2 years
  Event Schema:
    {
      "order_id": 123456,
      "user_id": 12345,
      "chargeback_amount": 250.00,
      "chargeback_date": "2026-05-01",
      "detected_at": "2026-05-02T08:00:00Z"
    }

Topic: fraud.model_training_completed
  Partitions: 1
  Retention: 90 days
  Event Schema:
    {
      "model_version": "v1.2.4",
      "model_id": 42,
      "accuracy": 0.9501,
      "precision": 0.9200,
      "recall": 0.8900,
      "f1_score": 0.9045,
      "training_samples": 500000,
      "training_completed_at": "2026-04-08T02:00:00Z"
    }
```

---

## Implementation Code

### 1. Fraud Scoring Service (Main Orchestrator)

```java
@Service
@Slf4j
public class FraudScoringService {

    @Autowired private RulesEngine rulesEngine;
    @Autowired private MLInferenceService mlInferenceService;
    @Autowired private VelocityChecker velocityChecker;
    @Autowired private SignalCollectionService signalCollectionService;
    @Autowired private FraudOrderRepository fraudOrderRepository;
    @Autowired private KafkaTemplate<String, String> kafkaTemplate;
    @Autowired private ObjectMapper objectMapper;
    @Autowired private MeterRegistry meterRegistry;

    private static final int RULES_WEIGHT = 40;
    private static final int ML_WEIGHT = 60;
    private static final int APPROVE_THRESHOLD = 40;
    private static final int REVIEW_THRESHOLD = 80;
    private static final int SCORING_TIMEOUT_MS = 200;

    /**
     * Score an order for fraud risk in real-time (< 200ms)
     */
    public FraudScore scoreOrder(Order order, HttpServletRequest request) {
        Timer.Sample sample = Timer.start(meterRegistry);
        long startTime = System.currentTimeMillis();

        try {
            log.info("Scoring order {} for user {}", order.getId(), order.getUserId());

            // 1. Collect all fraud signals (in parallel where possible)
            FraudSignals signals = signalCollectionService.collectSignals(order, request);

            // 2. Rules-based scoring (fast, < 30ms)
            int rulesScore = rulesEngine.evaluate(signals, order);
            log.debug("Rules score for order {}: {}", order.getId(), rulesScore);

            // 3. ML model inference (< 50ms with caching)
            int mlScore = mlInferenceService.inferScore(signals);
            log.debug("ML score for order {}: {}", order.getId(), mlScore);

            // 4. Velocity checks (< 10ms)
            int velocityBoost = velocityChecker.calculateVelocityAdjustment(order, signals);
            rulesScore = Math.min(100, rulesScore + velocityBoost);

            // 5. Aggregate scores
            int finalScore = aggregateScores(rulesScore, mlScore);

            // 6. Determine decision
            FraudDecision decision = determineDecision(finalScore, order, signals);

            // 7. Apply whitelisting rules
            if (isWhitelisted(order)) {
                decision = FraudDecision.APPROVE;
                finalScore = Math.min(60, finalScore);
                log.info("Order {} whitelisted (account age/history)", order.getId());
            }

            long scoringDurationMs = System.currentTimeMillis() - startTime;

            FraudScore score = FraudScore.builder()
                .orderId(order.getId())
                .userId(order.getUserId())
                .finalScore(finalScore)
                .rulesScore(rulesScore)
                .mlScore(mlScore)
                .decision(decision)
                .scoringDurationMs(scoringDurationMs)
                .signals(signals)
                .scoredAt(Instant.now())
                .build();

            // 8. Persist score to database (async, non-blocking)
            persistScoreAsync(score, signals);

            // 9. Publish scoring event (async)
            publishScoringCompletedEvent(score);

            // 10. Enqueue for manual review if needed
            if (decision == FraudDecision.REVIEW) {
                enqueueFraudReview(score);
            }

            log.info("Order {} fraud score: {} (decision: {}). Duration: {}ms",
                order.getId(), finalScore, decision, scoringDurationMs);

            // 11. Record metrics
            recordMetrics(score);

            return score;

        } catch (Exception e) {
            log.error("Error scoring order {}: {}", order.getId(), e.getMessage(), e);

            // Fail open: if scoring fails, approve order (prefer user experience)
            Timer.Sample.stop(sample, meterRegistry.timer("fraud.scoring.error"));
            return FraudScore.builder()
                .orderId(order.getId())
                .userId(order.getUserId())
                .finalScore(0)
                .decision(FraudDecision.APPROVE)
                .scoringDurationMs(System.currentTimeMillis() - startTime)
                .error(e.getMessage())
                .scoredAt(Instant.now())
                .build();
        }
    }

    /**
     * Aggregate rules and ML scores using weighted average
     */
    private int aggregateScores(int rulesScore, int mlScore) {
        return (int) Math.round(
            (rulesScore * RULES_WEIGHT + mlScore * ML_WEIGHT) / 100.0
        );
    }

    /**
     * Determine fraud decision based on final score
     */
    private FraudDecision determineDecision(int finalScore, Order order, FraudSignals signals) {
        if (finalScore > REVIEW_THRESHOLD) {
            return FraudDecision.BLOCK;
        } else if (finalScore >= APPROVE_THRESHOLD) {
            // Check secondary conditions for REVIEW
            if (order.getTotalAmount() > 5000 && signals.accountAgeInDays() < 30) {
                return FraudDecision.REVIEW; // New account + high value = review
            }
            return FraudDecision.REVIEW;
        } else {
            return FraudDecision.APPROVE;
        }
    }

    /**
     * Check if customer should be whitelisted
     */
    private boolean isWhitelisted(Order order) {
        // Very old account (5+ years) = low risk
        if (order.getUser().getAccountAgeInDays() > 1825) {
            return true;
        }

        // Many historical orders (100+) without fraud = low risk
        if (order.getUser().getTotalOrderCount() > 100) {
            return true;
        }

        // Corporate/verified account = low risk
        if (order.getUser().isVerifiedCorporateAccount()) {
            return true;
        }

        return false;
    }

    private void persistScoreAsync(FraudScore score, FraudSignals signals) {
        CompletableFuture.runAsync(() -> {
            try {
                FraudOrder fraudOrder = FraudOrder.builder()
                    .orderId(score.getOrderId())
                    .userId(score.getUserId())
                    .finalScore(score.getFinalScore())
                    .rulesScore(score.getRulesScore())
                    .mlScore(score.getMlScore())
                    .decision(score.getDecision().toString())
                    .decisionTimestamp(Instant.now())
                    .build();

                fraudOrderRepository.save(fraudOrder);

                log.debug("Persisted fraud score for order {}", score.getOrderId());

            } catch (Exception e) {
                log.error("Failed to persist fraud score for order {}", score.getOrderId(), e);
            }
        });
    }

    private void publishScoringCompletedEvent(FraudScore score) {
        try {
            FraudScoringCompletedEvent event = FraudScoringCompletedEvent.builder()
                .orderId(score.getOrderId())
                .userId(score.getUserId())
                .finalScore(score.getFinalScore())
                .decision(score.getDecision().toString())
                .scoringDurationMs(score.getScoringDurationMs())
                .scoredAt(score.getScoredAt())
                .build();

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("fraud.scoring.completed",
                String.valueOf(score.getOrderId()), eventJson);

        } catch (JsonProcessingException e) {
            log.error("Failed to publish fraud.scoring.completed event", e);
        }
    }

    private void enqueueFraudReview(FraudScore score) {
        try {
            FraudReviewQueueItem queueItem = FraudReviewQueueItem.builder()
                .orderId(score.getOrderId())
                .userId(score.getUserId())
                .finalScore(score.getFinalScore())
                .priority(calculatePriority(score.getFinalScore()))
                .status("PENDING")
                .enqueuedAt(Instant.now())
                .build();

            String itemJson = objectMapper.writeValueAsString(queueItem);
            kafkaTemplate.send("fraud.review_queue",
                String.valueOf(score.getFinalScore()), itemJson); // Partition by score

        } catch (JsonProcessingException e) {
            log.error("Failed to enqueue fraud review", e);
        }
    }

    private int calculatePriority(int finalScore) {
        if (finalScore >= 80) return 1;      // Highest priority
        if (finalScore >= 70) return 2;
        if (finalScore >= 60) return 3;
        if (finalScore >= 50) return 4;
        return 5;                             // Lowest priority
    }

    private void recordMetrics(FraudScore score) {
        meterRegistry.counter("fraud.orders.scored",
            "decision", score.getDecision().toString()).increment();

        meterRegistry.gauge("fraud.order.score", score.getFinalScore());
        meterRegistry.timer("fraud.scoring.duration").record(
            score.getScoringDurationMs(), TimeUnit.MILLISECONDS);
    }
}
```

### 2. Rules Engine with DSL

```java
@Service
@Slf4j
public class RulesEngine {

    @Autowired private FraudRuleRepository ruleRepository;
    @Autowired private VelocityChecker velocityChecker;
    @Autowired private RedisTemplate<String, Object> redisTemplate;

    private final ExpressionParser parser = new SpelExpressionParser();
    private final SimpleEvaluationContext context = SimpleEvaluationContext.forReadOnlyDataBinding();

    /**
     * Evaluate all fraud rules and calculate composite score
     */
    public int evaluate(FraudSignals signals, Order order) {
        List<FraudRule> enabledRules = ruleRepository.findByEnabledTrue();
        enabledRules.sort((a, b) -> Integer.compare(b.getPriority(), a.getPriority()));

        int totalScore = 0;
        StringBuilder ruleLog = new StringBuilder();

        for (FraudRule rule : enabledRules) {
            try {
                int ruleScore = evaluateRule(rule, signals, order);
                if (ruleScore > 0) {
                    totalScore += ruleScore;
                    ruleLog.append(rule.getRuleName()).append("=").append(ruleScore).append("; ");
                    log.debug("Rule '{}' scored {}", rule.getRuleName(), ruleScore);
                }
            } catch (Exception e) {
                log.error("Error evaluating rule '{}': {}", rule.getRuleName(), e.getMessage());
            }
        }

        log.debug("Rules evaluation for order {}: total score = {}. Breakdown: {}",
            order.getId(), Math.min(100, totalScore), ruleLog.toString());

        return Math.min(100, totalScore); // Cap at 100
    }

    /**
     * Evaluate a single rule using Spring EL
     */
    private int evaluateRule(FraudRule rule, FraudSignals signals, Order order) {
        try {
            Expression expression = parser.parseExpression(rule.getRuleExpression());

            // Build evaluation context with signals
            EvaluationContext evalContext = new StandardEvaluationContext();
            evalContext.setVariable("signals", signals);
            evalContext.setVariable("order", order);
            evalContext.setVariable("velocity", velocityChecker);

            // Evaluate expression (should return boolean)
            Boolean ruleMatches = expression.getValue(evalContext, Boolean.class);

            if (Boolean.TRUE.equals(ruleMatches)) {
                return rule.getScoreAdjustment();
            } else {
                return 0;
            }

        } catch (Exception e) {
            log.error("Failed to evaluate rule expression: {}", rule.getRuleExpression(), e);
            return 0;
        }
    }
}

@Entity
@Data
@Table(name = "fraud_rules")
public class FraudRule {
    @Id
    private Long ruleId;

    private String ruleName;
    private String ruleType;

    // Spring EL expression (e.g., "#signals.loginVelocityLastHour > 5")
    private String ruleExpression;

    private Integer scoreAdjustment;
    private Boolean enabled;
    private Integer priority;
}

// Example fraud rules (stored in database, dynamically evaluated):
/*
1. Velocity Check (High Login Activity)
   Expression: "#signals.loginVelocityLastHour > 5"
   Score: +20

2. New Device + Large Transaction
   Expression: "#signals.isNewDevice AND #order.totalAmount > 1000"
   Score: +25

3. High-Risk BIN
   Expression: "#signals.cardBin.startsWith('4111')"
   Score: +15

4. Address Mismatch
   Expression: "#signals.isAddressMismatch AND #signals.avsResult.equals('N')"
   Score: +30

5. Multiple Cards from Same IP
   Expression: "#velocity.getCardCountFromIp(#signals.ipAddress) > 3"
   Score: +25

6. Impossible Travel (same IP, different countries)
   Expression: "#signals.isNewLocation AND #signals.ipCountry != #order.shippingCountry"
   Score: +20

7. Card Used in Multiple Countries (24h)
   Expression: "#velocity.getCountriesUsedInLastDay(#signals.cardBin) > 2"
   Score: +30

8. High-Value Items (Electronics, Luxury)
   Expression: "#signals.hasHighValueItems AND #order.totalAmount > 2000"
   Score: +15
*/
```

### 3. Velocity Checker (Redis-Based)

```java
@Service
@Slf4j
public class VelocityChecker {

    @Autowired private RedisTemplate<String, String> redisTemplate;
    @Autowired private MeterRegistry meterRegistry;

    private static final int USER_VELOCITY_LIMIT = 5;       // orders/hour
    private static final int CARD_VELOCITY_LIMIT = 2;       // orders/day
    private static final int IP_VELOCITY_LIMIT = 10;        // orders/hour

    /**
     * Calculate velocity-based score adjustment
     */
    public int calculateVelocityAdjustment(Order order, FraudSignals signals) {
        int totalAdjustment = 0;

        // Check user velocity (prevent account takeover)
        int userVelocity = getVelocity("user", String.valueOf(order.getUserId()), "1h");
        if (userVelocity > USER_VELOCITY_LIMIT) {
            totalAdjustment += 20;
            meterRegistry.counter("fraud.velocity.user.exceeded").increment();
        }

        // Check card velocity
        String cardHash = hashCard(order.getCardToken());
        int cardVelocity = getVelocity("card", cardHash, "1d");
        if (cardVelocity > CARD_VELOCITY_LIMIT) {
            totalAdjustment += 25;
            meterRegistry.counter("fraud.velocity.card.exceeded").increment();
        }

        // Check IP velocity
        String ipHash = hashIp(signals.ipAddress());
        int ipVelocity = getVelocity("ip", ipHash, "1h");
        if (ipVelocity > IP_VELOCITY_LIMIT) {
            totalAdjustment += 15;
            meterRegistry.counter("fraud.velocity.ip.exceeded").increment();
        }

        return totalAdjustment;
    }

    /**
     * Get velocity count from Redis
     */
    private int getVelocity(String type, String identifier, String period) {
        String key = String.format("fraud:velocity:%s:%s:%s", type, identifier, period);

        String value = redisTemplate.opsForValue().get(key);
        int count = value != null ? Integer.parseInt(value) : 0;

        // Increment counter
        redisTemplate.opsForValue().increment(key);

        // Set TTL if this is the first increment
        if (count == 0) {
            int ttlSeconds = period.equals("1h") ? 3600 : 86400;
            redisTemplate.expire(key, Duration.ofSeconds(ttlSeconds));
        }

        return count;
    }

    /**
     * Check how many countries a card was used in (last 24h)
     */
    public int getCountriesUsedInLastDay(String cardBin) {
        String key = "fraud:card:" + hashCard(cardBin) + ":countries:1d";
        Set<Object> countries = redisTemplate.opsForSet().members(key);
        return countries != null ? countries.size() : 0;
    }

    /**
     * Check how many cards were used from a specific IP
     */
    public int getCardCountFromIp(String ipAddress) {
        String key = "fraud:ip:" + hashIp(ipAddress) + ":cards";
        Set<Object> cards = redisTemplate.opsForSet().members(key);
        return cards != null ? cards.size() : 0;
    }

    private String hashCard(String cardToken) {
        return DigestUtils.sha256Hex(cardToken);
    }

    private String hashIp(String ipAddress) {
        return DigestUtils.sha256Hex(ipAddress);
    }
}
```

### 4. ML Model Inference Service

```java
@Service
@Slf4j
public class MLInferenceService {

    @Autowired private RestTemplate restTemplate;
    @Autowired private RedisTemplate<String, Object> redisTemplate;
    @Autowired private FraudModelRepository modelRepository;
    @Autowired private MeterRegistry meterRegistry;

    private static final String ML_SERVICE_URL = "http://fraud-ml-service:8080/infer";
    private static final int INFERENCE_TIMEOUT_MS = 50;

    /**
     * Call ML model for fraud score prediction
     */
    public int inferScore(FraudSignals signals) {
        Timer.Sample sample = Timer.start(meterRegistry);

        try {
            // 1. Get active model version
            FraudModel activeModel = getActiveModel();

            // 2. Build inference request
            MLInferenceRequest request = MLInferenceRequest.builder()
                .modelVersion(activeModel.getModelVersion())
                .features(featureVectorFromSignals(signals))
                .build();

            // 3. Call ML service with timeout
            Callable<Integer> inferenceTask = () -> {
                try {
                    ResponseEntity<MLInferenceResponse> response = restTemplate.postForEntity(
                        ML_SERVICE_URL,
                        request,
                        MLInferenceResponse.class
                    );

                    if (response.getStatusCode() == HttpStatus.OK && response.getBody() != null) {
                        int rawScore = response.getBody().getFraudScore();
                        log.debug("ML inference returned score: {}", rawScore);
                        return rawScore;
                    } else {
                        log.warn("Unexpected response from ML service: {}", response.getStatusCode());
                        return 50; // Default score on error
                    }
                } catch (Exception e) {
                    log.error("ML inference request failed", e);
                    return 50; // Default score on error
                }
            };

            // 4. Execute with timeout
            ExecutorService executor = Executors.newSingleThreadExecutor();
            Future<Integer> futureScore = executor.submit(inferenceTask);

            try {
                Integer score = futureScore.get(INFERENCE_TIMEOUT_MS, TimeUnit.MILLISECONDS);
                sample.stop(meterRegistry.timer("fraud.ml.inference.success"));
                return score;

            } catch (TimeoutException e) {
                log.warn("ML inference timed out after {}ms", INFERENCE_TIMEOUT_MS);
                futureScore.cancel(true);
                sample.stop(meterRegistry.timer("fraud.ml.inference.timeout"));
                return 50; // Default score on timeout
            } finally {
                executor.shutdown();
            }

        } catch (Exception e) {
            log.error("Error in ML inference", e);
            sample.stop(meterRegistry.timer("fraud.ml.inference.error"));
            return 50; // Default score on any error
        }
    }

    /**
     * Get active model version (with canary rollout support)
     */
    private FraudModel getActiveModel() {
        // Check Redis for active model version
        String activeVersion = (String) redisTemplate.opsForValue()
            .get("fraud:model:version:active");

        if (activeVersion == null) {
            activeVersion = "v1.2.3"; // Fallback to known good version
        }

        return modelRepository.findByModelVersionAndDeploymentStatus(
            activeVersion, "ACTIVE"
        ).orElseThrow(() -> new ModelNotFoundException("Active model not found"));
    }

    /**
     * Convert fraud signals to ML feature vector
     */
    private double[] featureVectorFromSignals(FraudSignals signals) {
        return new double[]{
            // Numeric features (normalized 0-1)
            normalize(signals.accountAgeInDays(), 0, 10000),
            normalize(signals.orderCountLastMonth(), 0, 100),
            normalize(signals.loginVelocityLastHour(), 0, 20),
            boolToDouble(signals.isNewDevice()),
            boolToDouble(signals.isNewLocation()),

            normalize(signals.orderAmount().doubleValue(), 0, 10000),
            boolToDouble(signals.hasHighValueItems()),
            normalize(signals.itemQuantity(), 0, 50),

            normalize(signals.ipRiskScore(), 0, 1),
            boolToDouble(signals.isNewIp()),

            boolToDouble(signals.isNewCard()),
            normalize(signals.previousCardUseCount(), 0, 100),

            boolToDouble(signals.isAddressMismatch()),
            boolToDouble(signals.isHighRiskCountry())
        };
    }

    private double normalize(double value, double min, double max) {
        if (max == min) return 0;
        return (value - min) / (max - min);
    }

    private double boolToDouble(boolean value) {
        return value ? 1.0 : 0.0;
    }
}

@Data
@Builder
class MLInferenceRequest {
    private String modelVersion;
    private double[] features;
}

@Data
class MLInferenceResponse {
    private int fraudScore;
    private double confidence;
    private long inferenceTimeMs;
}
```

### 5. Manual Review Service & Analyst API

```java
@Service
@Slf4j
public class ManualReviewService {

    @Autowired private MongoTemplate mongoTemplate;
    @Autowired private FraudOrderRepository fraudOrderRepository;
    @Autowired private KafkaTemplate<String, String> kafkaTemplate;
    @Autowired private ObjectMapper objectMapper;

    /**
     * Get pending reviews for analyst (sorted by priority)
     */
    public List<FraudReviewQueueItem> getPendingReviews(int limit) {
        Query query = new Query()
            .addCriteria(Criteria.where("status").is("PENDING"))
            .with(Sort.by(Sort.Direction.ASC, "priority", "final_score").descending())
            .limit(limit);

        return mongoTemplate.find(query, FraudReviewQueueItem.class, "fraud_review_queue");
    }

    /**
     * Claim review item (mark as being reviewed by analyst)
     */
    public void claimReview(Long orderId, String analystName) {
        Update update = new Update()
            .set("status", "CLAIMED")
            .set("claimed_by", analystName)
            .set("claimed_at", Instant.now());

        mongoTemplate.updateFirst(
            Query.query(Criteria.where("order_id").is(orderId)),
            update,
            "fraud_review_queue"
        );

        log.info("Review for order {} claimed by {}", orderId, analystName);
    }

    /**
     * Submit analyst decision (approve/reject)
     */
    public void submitReviewDecision(Long orderId, String decision,
                                     String analystName, String notes) {
        Update update = new Update()
            .set("status", "REVIEWED")
            .set("analyst_decision", decision)
            .set("analyst_name", analystName)
            .set("analyst_notes", notes)
            .set("reviewed_at", Instant.now())
            .set("actual_fraud_label", decision.equals("REJECT") ? "FRAUD" : "LEGITIMATE");

        mongoTemplate.updateFirst(
            Query.query(Criteria.where("order_id").is(orderId)),
            update,
            "fraud_review_queue"
        );

        // Update PostgreSQL
        fraudOrderRepository.updateAnalystDecision(
            orderId, decision, analystName, notes
        );

        // Publish analyst feedback event (for ML retraining)
        publishAnalystFeedbackEvent(orderId, decision, analystName);

        log.info("Review decision submitted for order {}: {}", orderId, decision);
    }

    private void publishAnalystFeedbackEvent(Long orderId, String decision,
                                             String analystName) {
        try {
            FraudAnalystFeedbackEvent event = FraudAnalystFeedbackEvent.builder()
                .orderId(orderId)
                .analystDecision(decision)
                .analystName(analystName)
                .reviewedAt(Instant.now())
                .actualFraudLabel(decision.equals("REJECT") ? "FRAUD" : "LEGITIMATE")
                .build();

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("fraud.analyst_feedback",
                String.valueOf(orderId), eventJson);

        } catch (JsonProcessingException e) {
            log.error("Failed to publish analyst feedback event", e);
        }
    }
}

@RestController
@RequestMapping("/api/fraud/analyst")
@Slf4j
public class FraudAnalystController {

    @Autowired private ManualReviewService reviewService;
    @Autowired private AuthenticationService authService;

    @GetMapping("/queue")
    public List<FraudReviewQueueItem> getReviewQueue(
            @RequestParam(defaultValue = "20") int limit,
            HttpServletRequest request) {
        String analystName = authService.getAuthenticatedUser(request);
        log.info("Analyst {} fetched review queue (limit: {})", analystName, limit);

        return reviewService.getPendingReviews(limit);
    }

    @PostMapping("/claim/{orderId}")
    public void claimReview(@PathVariable Long orderId, HttpServletRequest request) {
        String analystName = authService.getAuthenticatedUser(request);
        reviewService.claimReview(orderId, analystName);
    }

    @PostMapping("/{orderId}/decision")
    public void submitDecision(
            @PathVariable Long orderId,
            @RequestBody ReviewDecisionRequest request,
            HttpServletRequest httpRequest) {
        String analystName = authService.getAuthenticatedUser(httpRequest);

        reviewService.submitReviewDecision(
            orderId,
            request.getDecision(),
            analystName,
            request.getNotes()
        );
    }
}

@Data
class ReviewDecisionRequest {
    private String decision; // "APPROVE" or "REJECT"
    private String notes;
}
```

---

## Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| **ML Model Service Down** | All orders: rules_only scoring (0.4 weight) | Circuit breaker, fallback to ensemble average |
| **Redis Velocity Unavailable** | Risk of duplicates in short windows | Fallback to PostgreSQL velocity (slower) |
| **Rules Engine Crashes** | All orders bypass rules (only ML) | Try-catch per rule, default score 50 |
| **High False Positive Rate** | Legitimate customers blocked, revenue loss | A/B test thresholds, analyst feedback loop |
| **High False Negative Rate** | Fraudsters get through, chargebacks increase | Increase ML_WEIGHT from 60% to 70% |
| **Analyst Queue Backlog** | Reviews pile up, SLA breached | Auto-escalate >80 scores to senior analysts |
| **Model Performance Degrades** | Old model no longer captures new fraud patterns | Weekly retraining, holdout test set monitoring |
| **Chargeback Label Delay** | ML training on stale labels | Manual labeling from analyst feedback + chargebacks |

---

## Scaling Strategy

### Horizontal Scaling

**Fraud Scoring Service:**
- Scale: 1 instance per 10,000 orders/sec (116/sec = 1 instance sufficient)
- Load balancer: Round-robin across instances
- Stateless design: All state in Redis, PostgreSQL, MongoDB

**ML Inference Service:**
- Scale: GPU-accelerated instances (XGBoost on GPU)
- Caching: Model cached in memory, inference batched
- Throughput: 500 inferences/sec per GPU instance

**Manual Review Service:**
- Scale: 1 instance per 5 analysts (horizontal scale with analysts)
- Queue partitioning: By score range for better load distribution

### Vertical Scaling

**Redis:**
- Current: ~100MB for velocity counters
- Scaling: Redis Cluster with 6 nodes (3 primary, 3 replica)

**PostgreSQL:**
- Read replicas for analyst queries
- Partitioning `fraud_orders` by date for faster scans

**MongoDB:**
- Sharded cluster on `order_id`
- TTL index for auto-cleanup (2-year retention)

---

## Monitoring & Observability

### Key Metrics

```
# Fraud Scoring
fraud.orders.scored (counter, by decision)
fraud.order.score (gauge, histogram)
fraud.scoring.duration (histogram, percentiles)
fraud.rules.evaluations (counter, by rule)
fraud.ml.inference.latency (histogram)

# Velocity
fraud.velocity.user.exceeded (counter)
fraud.velocity.card.exceeded (counter)
fraud.velocity.ip.exceeded (counter)

# Decision Quality
fraud.false_positive_rate (gauge)
fraud.false_negative_rate (gauge)
fraud.chargeback_rate (gauge)
fraud.analyst_agreement_rate (gauge)

# Manual Review
fraud.review_queue.size (gauge)
fraud.analyst.review_time (histogram)
fraud.analyst_decision.approved (counter)
fraud.analyst_decision.rejected (counter)

# Model Performance
fraud.model.accuracy (gauge)
fraud.model.precision (gauge)
fraud.model.recall (gauge)
fraud.model.f1_score (gauge)
```

### Dashboards

1. **Real-Time Fraud Dashboard**
   - Orders scored per minute
   - Decision breakdown (APPROVE/REVIEW/BLOCK)
   - Scoring latency (p50, p95, p99)
   - False positive rate trend

2. **Manual Review Dashboard**
   - Queue size and aging
   - Analyst productivity (reviews/hour)
   - Decision distribution (approve vs reject)
   - Review SLA compliance

3. **Model Performance Dashboard**
   - Accuracy trends
   - Precision/recall tradeoff
   - Feature importance
   - Model version performance

---

## Summary Cheat Sheet

### Decision Thresholds

| Score | Decision | Action |
|-------|----------|--------|
| 0-39 | APPROVE | Proceed with checkout |
| 40-79 | REVIEW | Enqueue for manual analyst review |
| 80+ | BLOCK | Decline order, request verification |

### Latency Budget (200ms total)

| Component | Budget | Allocation |
|-----------|--------|-----------|
| Signal collection | 100ms | Device fingerprint, address lookup |
| Rules engine | 30ms | Fast rules evaluation |
| ML inference | 50ms | Model prediction call |
| Score aggregation | 10ms | Weighting and decision logic |
| Reserve | 10ms | Network latency, buffer |

### Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| **Rules + ML hybrid** | Explainability + accuracy. Rules catch known patterns fast, ML learns new frauds |
| **Synchronous scoring** | Sub-200ms latency required for checkout experience |
| **Fail-open (approve on error)** | Prefer user experience over fraud prevention |
| **Manual review queue** | Analyst feedback improves model accuracy |
| **Weekly retraining** | Balance between capturing new fraud and stability |

### Cost Optimization

- **Inference batching:** Batch 100 orders for GPU utilization (vs 1-by-1)
- **Caching:** Pre-compute risk scores for common card BINs
- **Cold filtering:** Pre-filter obvious frauds with rules before expensive ML
