---
title: Dynamic Pricing System
layout: default
---

# Dynamic Pricing System — Deep Dive Design

> **Scenario:** Design a dynamic pricing system that adjusts prices based on:
> - Competitor prices (scraped hourly)
> - Demand (views, add-to-carts)
> - Inventory levels (discount slow-moving items)
> - Time of day, day of week, seasonality
> - Customer segments (premium vs regular)
> - Pricing rules and constraints (min margin, max discount)
> - Must update prices across all systems within 1 minute
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
- Continuously monitor competitor prices (hourly scrape)
- Track demand signals (page views, add-to-cart, purchase rate)
- Adjust prices based on inventory turnover
- Apply time-of-day and seasonality multipliers
- Segment customer pricing (VIP, regular, new)
- Enforce pricing constraints (min margin 20%, max discount 50%)
- Price lock during checkout (15 min grace period)
- Propagate price changes to search, catalog, cart within 1 minute
- Audit all price changes (compliance, fraud detection)

### Non-Functional Requirements
- Pricing decision latency: <50ms
- Price update propagation: <1 minute
- Availability: 99.9% (pricing fallback to list price)
- Consistency: Strong consistency for checkout price; eventual for catalog
- Durability: Immutable audit trail
- Data retention: 7 years of pricing history

---

## Capacity Estimation

| Metric | Calculation | Value |
|--------|-----------|-------|
| Products in catalog | Typical large e-commerce | 500,000 |
| Price updates/day | 5 signals × 500K products | 2.5M updates |
| Price checks/day | 10M daily users × 3 products browsed | 30M |
| Competitor price scrapes/day | 1000 competitors × hourly | 24,000 |
| Checkout transactions/day | 100K TPS avg × 86,400 sec | ~8.6M |
| Price lock requests/day | 8.6M checkouts | 8.6M (some resume cart) |
| Audit records (7 years) | 2.5M/day × 365 × 7 | 6.4B records |
| Price cache (Redis) | 500K products × 100 bytes | ~50MB |
| Competitor price data | 1000 competitors × 500K SKUs | ~500GB (sparse) |
| Pricing rule engine | 1000 rules × 500K SKUs | ~2GB |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                 Pricing Signal Sources                           │
│  Competitor APIs | Demand Tracker | Inventory System | Events   │
└────────────────────────┬────────────────────────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼────┐    ┌─────▼──────┐  ┌────▼──────┐
    │Competitor│   │   Demand   │  │ Inventory │
    │  Scraper │   │  Tracker   │  │  Poller   │
    └────┬────┘    └─────┬──────┘  └────┬──────┘
         │               │              │
         └───────────────┼──────────────┘
                         │ (Kafka: pricing-signals)
                         ▼
         ┌────────────────────────────────┐
         │   PricingEngine                │
         │ (Rules + ML scoring)           │
         │ Output: new price, version     │
         └────────────────────────────────┘
                         │
             ┌───────────┼───────────┐
             │           │           │
        ┌────▼───┐  ┌────▼────┐  ┌──▼───────┐
        │ Redis  │  │PostgreSQL│  │ MongoDB  │
        │ (cache)│  │ (audit)  │  │ (rules)  │
        └────┬───┘  └────┬────┘  └──┬───────┘
             │           │          │
             └───────────┼──────────┘
                         │
    ┌────────────────────▼─────────────────────┐
    │      PriceChangeProducer                 │
    │      (Kafka: price-changes topic)        │
    └────────────────────┬─────────────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼────┐    ┌─────▼─────┐  ┌────▼──────┐
    │ Search  │    │  Catalog  │  │   Cart    │
    │ Service │    │  Service  │  │  Service  │
    └─────────┘    └───────────┘  └───────────┘
         │
         └──── PriceCheckoutValidator
              (version-checked validation)
```

---

## Core Design Questions Answered

### 1. How do you collect pricing signals from multiple sources?

**Multi-Source Collection:**
- **Competitor API/Scraper:** Schedule hourly, store snapshots in MongoDB
- **Demand Tracker:** Listen to product-view, add-to-cart, purchase events via Kafka
- **Inventory Poller:** Query warehouse system every 30 min for stock levels
- **Market Data:** Subscribe to external feeds (commodity prices, currency rates)

**Normalization:** Convert all signals to common format for PricingEngine

```
pricing-signals Kafka topic:
{
  "productId": "prod-456",
  "signalType": "COMPETITOR_PRICE|DEMAND|INVENTORY|MARKET_DATA",
  "value": 299.99,
  "source": "amazon|ebay|warehouse",
  "timestamp": 1711939200000
}
```

### 2. How do you calculate optimal prices (real-time vs batch)?

**Hybrid Approach:**
- **Real-time (within 5s):** Simple rule-based pricing (min/max margin checks)
- **Batch (hourly):** ML-based pricing optimization considering all signals
- **Incremental (real-time):** React to critical signals (out-of-stock competitor, demand spike)

**PricingEngine Output:**
```
{
  "productId": "prod-456",
  "listPrice": 399.99,
  "competitorLowest": 289.99,
  "calculatedPrice": 349.99,
  "multipliers": {
    "demand": 1.1,
    "inventory": 0.95,
    "competitor": 0.97,
    "time_of_day": 1.05,
    "seasonality": 1.0,
    "customer_segment": 1.0
  },
  "constraints": {
    "minPrice": 319.99,    // 20% margin
    "maxPrice": 399.99,    // list price
    "appliedConstraint": "none"
  },
  "version": 145,
  "timestamp": 1711939200000
}
```

### 3. How do you propagate price changes to all services?

**Kafka-Driven Propagation:**
1. PricingEngine publishes `price-changes` event
2. Multiple consumers subscribe:
   - **SearchService:** Updates Elasticsearch
   - **CatalogService:** Updates product feed
   - **CartService:** Updates cart item prices
   - **PriceAuditService:** Logs to PostgreSQL
3. Each service acknowledges receipt (idempotent processing)
4. Distributed transaction NOT used (eventual consistency acceptable)

**SLA:** 95th percentile propagation < 1 minute

### 4. How do you ensure pricing consistency during checkout?

**Price Lock Mechanism:**
1. Customer adds item to cart → call `reservePrice()` → get version number
2. Price locked in Redis for 15 minutes at that version
3. At checkout, validate price hasn't changed by checking version
4. If changed → ask user to confirm new price
5. Payment processor uses locked price

**Implementation:**
```java
// At add-to-cart time
PriceReservation reservation = priceLockService.reservePrice(productId);
// reservation.price = 349.99, version = 145, expiresAt = now + 15min

// At checkout time
priceValidator.validatePrice(productId, version, expectedPrice);
// if version matches → proceed
// if version mismatch and price higher → user confirms
// if version mismatch and price lower → apply new price
```

### 5. How do you audit price changes for compliance?

**Immutable Audit Trail (PostgreSQL):**
```sql
-- Every price change logged with justification
INSERT INTO price_audit_log (
    product_id, old_price, new_price, version,
    changed_by, reason, signals, timestamp
) VALUES (...)

-- Query example: find all price changes for product in date range
SELECT * FROM price_audit_log
WHERE product_id = 'prod-456'
  AND timestamp BETWEEN '2024-01-01' AND '2024-01-31'
ORDER BY timestamp DESC
```

**Compliance Checks:**
- Prevent artificial price inflation during holidays
- Alert if discount > 50%
- Alert if price changes > 3x in single day
- Track competitor price vs our price ratio

### 6. How do you handle race conditions when customer checks out during price change?

**Optimistic Locking with Version Numbers:**

| Scenario | Resolution |
|----------|-----------|
| Price locked; no change | Use locked price ✓ |
| Price locked; new price lower | Apply new price (customer benefits) ✓ |
| Price locked; new price higher | Notify customer, allow rejection |
| Lock expired (>15 min) | Re-check current price before payment |
| Concurrent price updates | Last-write-wins (version number determines order) |

**Validation Code:**
```java
CheckoutValidationResult validateCheckoutPrice(
        String productId,
        int lockedVersion,
        BigDecimal expectedPrice) {

    CurrentPrice current = priceService.getCurrentPrice(productId);

    if (current.version == lockedVersion &&
        current.price.equals(expectedPrice)) {
        // Success: price unchanged
        return CheckoutValidationResult.SUCCESS;
    }

    if (current.version > lockedVersion) {
        if (current.price < expectedPrice) {
            // Customer benefits: apply new price
            return CheckoutValidationResult.SUCCESS_NEW_PRICE(current.price);
        } else {
            // Price increased: need confirmation
            return CheckoutValidationResult.PRICE_CHANGED(current.price);
        }
    }

    if (System.currentTimeMillis() > lockedPrice.expiresAt) {
        // Lock expired
        return CheckoutValidationResult.LOCK_EXPIRED;
    }

    return CheckoutValidationResult.CONFLICT;
}
```

---

## Microservices Breakdown

| Service | Responsibility | Tech |
|---------|-----------------|------|
| **CompetitorPriceScraper** | Hourly price collection from competitors | Spring Scheduler, WebClient, MongoDB |
| **DemandTracker** | Track page views, add-to-cart, purchase rates | Kafka Consumer, Redis, PostgreSQL |
| **InventoryPoller** | Poll warehouse system for stock levels | REST Client, Kafka Producer |
| **PricingEngine** | Calculate optimal price using rules + ML | Spring Boot, MongoDB, Redis |
| **PriceLockService** | Reserve price during cart/checkout | Redis, Spring Boot |
| **PriceChangeProducer** | Publish price updates to Kafka | Kafka Producer |
| **PriceAuditService** | Log all price changes for compliance | PostgreSQL, Spring Boot |
| **CheckoutPriceValidator** | Version-checked validation at payment | Spring Boot, Redis |
| **PriceAPIGateway** | Serve current prices (catalog, API) | Spring Cloud Gateway, Redis cache |

---

## Database Design

### PostgreSQL Schema

```sql
-- Price history and audit trail
CREATE TABLE price_audit_log (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL,
    old_price DECIMAL(10,2) NOT NULL,
    new_price DECIMAL(10,2) NOT NULL,
    version INT NOT NULL,
    changed_by VARCHAR(100), -- service name or user
    reason VARCHAR(255),      -- COMPETITOR_PRICE, DEMAND, INVENTORY, etc
    signals JSONB,             -- raw signals that triggered change
    min_allowed_price DECIMAL(10,2),
    max_allowed_price DECIMAL(10,2),
    constraint_applied VARCHAR(50),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_product_timestamp (product_id, timestamp),
    INDEX idx_timestamp (timestamp)
);

-- Pricing rules (constraints)
CREATE TABLE pricing_rules (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL,
    min_margin_percent DECIMAL(5,2) NOT NULL DEFAULT 20.0,
    max_discount_percent DECIMAL(5,2) NOT NULL DEFAULT 50.0,
    competitor_price_threshold DECIMAL(10,2),
    demand_multiplier_high DECIMAL(3,2) DEFAULT 1.1,
    demand_multiplier_low DECIMAL(3,2) DEFAULT 0.9,
    customer_segment VARCHAR(50),
    enabled BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(product_id, customer_segment)
);

-- Customer segment pricing
CREATE TABLE customer_segments (
    id BIGSERIAL PRIMARY KEY,
    segment_name VARCHAR(100) UNIQUE NOT NULL, -- VIP, REGULAR, NEW
    discount_percentage DECIMAL(5,2) DEFAULT 0.0,
    price_visibility VARCHAR(50), -- FULL, HIDDEN, COMPETITOR_AWARE
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Price lock (reserved prices during cart/checkout)
CREATE TABLE price_locks (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    locked_price DECIMAL(10,2) NOT NULL,
    version INT NOT NULL,
    locked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    INDEX idx_user_id (user_id),
    INDEX idx_expires_at (expires_at)
);

-- Competitor price snapshots
CREATE TABLE competitor_prices (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL,
    competitor_name VARCHAR(100) NOT NULL,
    competitor_price DECIMAL(10,2) NOT NULL,
    competitor_url VARCHAR(500),
    in_stock BOOLEAN DEFAULT TRUE,
    scraped_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_product_competitor_date (product_id, competitor_name, scraped_at),
    UNIQUE(product_id, competitor_name, scraped_at)
);

-- Demand signals (views, add-to-cart, purchases)
CREATE TABLE demand_signals (
    id BIGSERIAL PRIMARY KEY,
    product_id BIGINT NOT NULL,
    signal_type VARCHAR(50), -- VIEW, ADD_TO_CART, PURCHASE
    count INT DEFAULT 1,
    signal_date DATE NOT NULL,
    avg_time_on_page_sec INT,
    bounce_rate DECIMAL(3,2),
    INDEX idx_product_date (product_id, signal_date)
);
```

### MongoDB Collections

```javascript
// Pricing rules and constraints (flexible schema)
db.pricing_rules.createIndex({
    "productId": 1,
    "enabled": 1
});

db.pricing_rules.insertOne({
    _id: ObjectId(),
    productId: 456,
    ruleType: "COMPETITOR_AWARE",
    config: {
        matchCompetitorThreshold: 0.95,  // 95% of competitor's price
        undercut: 0.98,                  // 2% below
        minMarginPercent: 20
    },
    seasonalAdjustments: {
        "HOLIDAY": 1.15,
        "BLACK_FRIDAY": 0.70,
        "SUMMER": 1.05
    },
    timeOfDayMultiplier: {
        "6-9": 1.1,      // Morning peak
        "12-14": 1.05,   // Lunch
        "18-22": 1.15    // Evening peak
    },
    enabled: true,
    updatedAt: ISODate("2024-01-15T00:00:00Z")
});

// ML model for pricing optimization
db.pricing_models.insertOne({
    _id: "pricing-model-v3",
    version: 3,
    trainedDate: ISODate("2024-01-15T00:00:00Z"),
    features: [
        "competitor_price_diff",
        "inventory_level",
        "demand_momentum",
        "seasonality",
        "time_of_day",
        "customer_segment"
    ],
    modelMetrics: {
        revenue_lift: 0.12,    // 12% revenue increase vs static pricing
        elasticity: -1.2,      // 1.2% demand decrease per 1% price increase
        margin_improvement: 0.08
    },
    coefficients: { /* model weights */ }
});

// Competitor price tracking
db.competitor_prices.createIndex({
    "productId": 1,
    "competitorName": 1,
    "scrapedAt": -1
});

db.competitor_prices.insertOne({
    _id: ObjectId(),
    productId: 456,
    competitorName: "amazon",
    price: 289.99,
    inStock: true,
    url: "https://amazon.com/...",
    scrapedAt: ISODate("2024-01-15T14:00:00Z")
});
```

---

## Redis Data Structures

### Current Price Cache
```
KEY: price:{productId}
TYPE: Hash
TTL: 30 seconds (refreshed on every price change)
VALUE:
{
  "price": "349.99",
  "version": "145",
  "minPrice": "319.99",
  "maxPrice": "399.99",
  "updatedAt": "1711939200000"
}

Example Redis command:
HSET price:prod-456 price 349.99 version 145 minPrice 319.99 maxPrice 399.99 updatedAt 1711939200000
EXPIRE price:prod-456 30
```

### Price Lock (Reservation)
```
KEY: price-lock:{userId}:{productId}
TYPE: Hash
TTL: 15 minutes (checkout grace period)
VALUE:
{
  "price": "349.99",
  "version": "145",
  "lockedAt": "1711939200000",
  "expiresAt": "1711940100000"
}

Example:
HSET price-lock:user-12345:prod-456 price 349.99 version 145 lockedAt 1711939200000
EXPIRE price-lock:user-12345:prod-456 900  // 15 minutes
```

### Competitor Price Cache
```
KEY: competitor-price:{productId}:{competitor}
TYPE: String (JSON)
TTL: 1 hour (next hourly scrape)
VALUE:
{
  "price": 289.99,
  "inStock": true,
  "url": "...",
  "scrapedAt": 1711939200000
}
```

### Demand Metrics (Rolling Window)
```
KEY: demand:{productId}:minute
TYPE: Counter (INCR)
TTL: 1 hour

KEY: demand:{productId}:hour
TYPE: Counter (INCR on minute rollup)
TTL: 24 hours

Example:
INCR demand:prod-456:minute  (incremented per view/add-to-cart)
```

---

## Kafka Event Flow

### Topic: `pricing-signals`
```
Partition: productId % num_partitions (ensure order per product)
Retention: 24 hours
Replication: 3

Message:
{
  "productId": 456,
  "signalType": "COMPETITOR_PRICE|DEMAND|INVENTORY|MARKET_DATA",
  "value": 289.99,
  "source": "amazon|ebay|warehouse",
  "confidence": 0.95,
  "timestamp": 1711939200000,
  "extra": {
    "inStock": true,
    "url": "...",
    "demand_views": 1523
  }
}
```

### Topic: `price-changes`
```
Partition: productId % num_partitions
Retention: 7 days (compliance requirement)
Replication: 3

Message:
{
  "productId": 456,
  "oldPrice": 379.99,
  "newPrice": 349.99,
  "version": 145,
  "reason": "COMPETITOR_PRICE_MATCH",
  "signals": { /* raw signals */ },
  "minAllowedPrice": 319.99,
  "maxAllowedPrice": 399.99,
  "timestamp": 1711939200000,
  "changedBy": "pricing-engine-v2"
}
```

### Topic: `demand-events`
```
Message (from catalog/product views):
{
  "productId": 456,
  "eventType": "VIEW|ADD_TO_CART|PURCHASE",
  "userId": 12345,
  "sessionId": "sess-abc123",
  "timestamp": 1711939200000
}

Consumer: DemandTracker (aggregates to Redis counters and PostgreSQL)
```

---

## Implementation Code

### 1. PricingEngine - Core Algorithm

```java
package com.pricing.service;

import org.springframework.stereotype.Service;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import com.mongodb.client.MongoCollection;
import org.bson.Document;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.util.*;

@Slf4j
@Service
public class PricingEngine {

    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final MongoCollection<Document> rulesCollection;
    private final MongoCollection<Document> modelsCollection;
    private final PricingRepository pricingRepo;

    public PricingEngine(
            RedisTemplate<String, String> redisTemplate,
            KafkaTemplate<String, String> kafkaTemplate,
            MongoCollection<Document> rulesCollection,
            MongoCollection<Document> modelsCollection,
            PricingRepository pricingRepo) {
        this.redisTemplate = redisTemplate;
        this.kafkaTemplate = kafkaTemplate;
        this.rulesCollection = rulesCollection;
        this.modelsCollection = modelsCollection;
        this.pricingRepo = pricingRepo;
    }

    public PriceCalculation calculatePrice(Long productId, PricingSignal signal) {
        long startTime = System.currentTimeMillis();
        PriceCalculation result = new PriceCalculation();

        try {
            // 1. Load product baseline
            Product product = getProduct(productId);
            BigDecimal listPrice = product.getListPrice();
            BigDecimal cost = product.getCost();

            // 2. Fetch current signals from cache
            PricingSignals signals = collectSignals(productId);

            // 3. Load pricing rules
            PricingRule rule = loadPricingRule(productId);

            // 4. Calculate multipliers
            double demandMultiplier = calculateDemandMultiplier(signals);
            double inventoryMultiplier = calculateInventoryMultiplier(signals);
            double competitorMultiplier = calculateCompetitorMultiplier(
                signals, rule);
            double timeMultiplier = calculateTimeMultiplier();
            double seasonalityMultiplier = calculateSeasonalityMultiplier();

            log.debug("Pricing multipliers [demand={}, inventory={}, competitor={}, " +
                     "time={}, season={}]",
                demandMultiplier, inventoryMultiplier, competitorMultiplier,
                timeMultiplier, seasonalityMultiplier);

            // 5. Apply multipliers
            BigDecimal calculatedPrice = listPrice
                .multiply(BigDecimal.valueOf(demandMultiplier))
                .multiply(BigDecimal.valueOf(inventoryMultiplier))
                .multiply(BigDecimal.valueOf(competitorMultiplier))
                .multiply(BigDecimal.valueOf(timeMultiplier))
                .multiply(BigDecimal.valueOf(seasonalityMultiplier));

            // 6. Enforce constraints
            BigDecimal minPrice = cost.multiply(BigDecimal.valueOf(1 + rule.getMinMarginPercent() / 100));
            BigDecimal maxPrice = listPrice;

            String appliedConstraint = "NONE";
            if (calculatedPrice.compareTo(minPrice) < 0) {
                calculatedPrice = minPrice;
                appliedConstraint = "MIN_MARGIN";
            } else if (calculatedPrice.compareTo(maxPrice) > 0) {
                calculatedPrice = maxPrice;
                appliedConstraint = "MAX_PRICE";
            }

            // 7. Round to standard increment (e.g., .99 ending)
            calculatedPrice = roundPrice(calculatedPrice);

            // 8. Build result
            result.setProductId(productId);
            result.setListPrice(listPrice);
            result.setCalculatedPrice(calculatedPrice);
            result.setVersion(System.currentTimeMillis());
            result.setDemandMultiplier(demandMultiplier);
            result.setInventoryMultiplier(inventoryMultiplier);
            result.setCompetitorMultiplier(competitorMultiplier);
            result.setTimeMultiplier(timeMultiplier);
            result.setSeasonalityMultiplier(seasonalityMultiplier);
            result.setMinPrice(minPrice);
            result.setMaxPrice(maxPrice);
            result.setAppliedConstraint(appliedConstraint);
            result.setTimestamp(System.currentTimeMillis());

            // 9. Check if price actually changed
            CurrentPrice currentCached = getCurrentPriceFromCache(productId);
            if (currentCached != null &&
                currentCached.getPrice().compareTo(calculatedPrice) == 0) {
                result.setPriceChanged(false);
                log.debug("Price unchanged for product {}", productId);
            } else {
                result.setPriceChanged(true);
                publishPriceChange(result, signal);
            }

            log.info("Price calculation [productId={}, calculatedPrice={}, " +
                    "changed={}, durationMs={}]",
                productId, calculatedPrice, result.isPriceChanged(),
                System.currentTimeMillis() - startTime);

            return result;

        } catch (Exception e) {
            log.error("Error calculating price for product {}", productId, e);
            result.setError(e.getMessage());
            return result;
        }
    }

    private double calculateDemandMultiplier(PricingSignals signals) {
        // Demand elasticity: views/purchases in last hour vs baseline
        long currentDemand = signals.getViewsLastHour();
        long baselineDemand = signals.getAvgViewsPerHour();

        if (baselineDemand == 0) return 1.0;

        double demandRatio = (double) currentDemand / baselineDemand;
        double multiplier = 1.0;

        if (demandRatio > 2.0) {
            // 2x+ demand spike: increase price
            multiplier = 1.1;
        } else if (demandRatio > 1.5) {
            multiplier = 1.05;
        } else if (demandRatio < 0.5) {
            // Low demand: discount
            multiplier = 0.9;
        } else if (demandRatio < 0.8) {
            multiplier = 0.95;
        }

        return multiplier;
    }

    private double calculateInventoryMultiplier(PricingSignals signals) {
        // Inventory turnover: days of stock remaining
        int inventoryLevel = signals.getInventoryLevel();
        double daysOfStock = (double) inventoryLevel / signals.getAvgDailySalesRate();

        if (daysOfStock > 30) {
            // Overstock: discount to clear
            return 0.85;
        } else if (daysOfStock > 14) {
            return 0.92;
        } else if (daysOfStock < 3) {
            // Scarcity: increase price
            return 1.08;
        } else if (daysOfStock < 7) {
            return 1.04;
        }

        return 1.0;
    }

    private double calculateCompetitorMultiplier(
            PricingSignals signals,
            PricingRule rule) {

        BigDecimal lowestCompetitorPrice = signals.getLowestCompetitorPrice();
        if (lowestCompetitorPrice == null) {
            return 1.0;
        }

        BigDecimal ourListPrice = signals.getListPrice();
        double priceDiff = lowestCompetitorPrice.doubleValue() /
                          ourListPrice.doubleValue();

        // If competitor is 10%+ cheaper, match or undercut slightly
        if (priceDiff < 0.90) {
            return priceDiff * rule.getCompetitorUndercut();  // typically 0.98
        }

        // If competitor is within 5%, maintain price
        if (priceDiff >= 0.95) {
            return 1.0;
        }

        // Otherwise, slight discount
        return 0.97;
    }

    private double calculateTimeMultiplier() {
        // Time-of-day and day-of-week multipliers
        Calendar cal = Calendar.getInstance();
        int hourOfDay = cal.get(Calendar.HOUR_OF_DAY);
        int dayOfWeek = cal.get(Calendar.DAY_OF_WEEK);

        // Evening peak (6 PM - 10 PM): +15%
        if (hourOfDay >= 18 && hourOfDay < 22) {
            return 1.15;
        }

        // Morning (6 AM - 9 AM): +10%
        if (hourOfDay >= 6 && hourOfDay < 9) {
            return 1.10;
        }

        // Overnight (12 AM - 6 AM): -5%
        if (hourOfDay >= 0 && hourOfDay < 6) {
            return 0.95;
        }

        // Weekend: +5%
        if (dayOfWeek == Calendar.SATURDAY || dayOfWeek == Calendar.SUNDAY) {
            return 1.05;
        }

        return 1.0;
    }

    private double calculateSeasonalityMultiplier() {
        // Seasonal adjustments
        Calendar cal = Calendar.getInstance();
        int month = cal.get(Calendar.MONTH);

        // Q4 (Oct-Dec): +10% (holiday season)
        if (month >= Calendar.OCTOBER && month <= Calendar.DECEMBER) {
            return 1.10;
        }

        // Summer (Jun-Aug): +5%
        if (month >= Calendar.JUNE && month <= Calendar.AUGUST) {
            return 1.05;
        }

        return 1.0;
    }

    private BigDecimal roundPrice(BigDecimal price) {
        // Round to .99 ending (psychological pricing)
        BigDecimal rounded = price.setScale(0, java.math.RoundingMode.FLOOR);
        return rounded.add(BigDecimal.valueOf(0.99));
    }

    private PricingSignals collectSignals(Long productId) {
        PricingSignals signals = new PricingSignals();

        // Get from Redis cache
        signals.setViewsLastHour(getCounterValue("demand:" + productId + ":minute"));
        signals.setAvgViewsPerHour(getCounterValue("demand:" + productId + ":hour-avg"));
        signals.setInventoryLevel(getInventoryLevel(productId));
        signals.setAvgDailySalesRate(getAverageSalesRate(productId));
        signals.setLowestCompetitorPrice(getLowestCompetitorPrice(productId));
        signals.setListPrice(getProductListPrice(productId));

        return signals;
    }

    private void publishPriceChange(PriceCalculation result, PricingSignal triggeringSignal) {
        try {
            String message = String.format(
                "{\"productId\":%d,\"oldPrice\":%.2f,\"newPrice\":%.2f," +
                "\"version\":%d,\"reason\":\"%s\",\"timestamp\":%d}",
                result.getProductId(),
                result.getListPrice(),
                result.getCalculatedPrice(),
                result.getVersion(),
                triggeringSignal.getSignalType(),
                result.getTimestamp()
            );

            kafkaTemplate.send("price-changes",
                String.valueOf(result.getProductId()),
                message);

        } catch (Exception e) {
            log.error("Failed to publish price change", e);
        }
    }

    private PricingRule loadPricingRule(Long productId) {
        // Load from MongoDB or cache
        Document doc = rulesCollection.find(
            new Document("productId", productId)
        ).first();

        if (doc != null) {
            return new PricingRule()
                .setMinMarginPercent(doc.getDouble("minMarginPercent"))
                .setCompetitorUndercut(doc.getDouble("competitorUndercut"));
        }

        // Return defaults
        return new PricingRule()
            .setMinMarginPercent(20.0)
            .setCompetitorUndercut(0.98);
    }

    private CurrentPrice getCurrentPriceFromCache(Long productId) {
        String key = "price:" + productId;
        Map<Object, Object> cached = redisTemplate.opsForHash()
            .entries(key);

        if (cached.isEmpty()) {
            return null;
        }

        return new CurrentPrice()
            .setPrice(new BigDecimal((String) cached.get("price")))
            .setVersion(Long.parseLong((String) cached.get("version")));
    }

    private long getCounterValue(String key) {
        String value = (String) redisTemplate.opsForValue().get(key);
        return value != null ? Long.parseLong(value) : 0L;
    }

    private Product getProduct(Long productId) {
        // Mock implementation
        return new Product().setId(productId)
            .setListPrice(BigDecimal.valueOf(399.99))
            .setCost(BigDecimal.valueOf(200.00));
    }

    private BigDecimal getLowestCompetitorPrice(Long productId) {
        String key = "competitor-price:" + productId + ":lowest";
        String value = (String) redisTemplate.opsForValue().get(key);
        return value != null ? new BigDecimal(value) : null;
    }

    private BigDecimal getProductListPrice(Long productId) {
        return BigDecimal.valueOf(399.99);
    }

    private int getInventoryLevel(Long productId) {
        return 150;
    }

    private double getAverageSalesRate(Long productId) {
        return 5.0; // items per day
    }
}

@Data
@Builder
class PriceCalculation {
    private Long productId;
    private BigDecimal listPrice;
    private BigDecimal calculatedPrice;
    private Long version;
    private double demandMultiplier;
    private double inventoryMultiplier;
    private double competitorMultiplier;
    private double timeMultiplier;
    private double seasonalityMultiplier;
    private BigDecimal minPrice;
    private BigDecimal maxPrice;
    private String appliedConstraint;
    private Long timestamp;
    private boolean priceChanged;
    private String error;
}

@Data
class PricingSignals {
    private long viewsLastHour;
    private long avgViewsPerHour;
    private int inventoryLevel;
    private double avgDailySalesRate;
    private BigDecimal lowestCompetitorPrice;
    private BigDecimal listPrice;
}

@Data
class PricingRule {
    private double minMarginPercent;
    private double competitorUndercut;
}
```

### 2. PriceLockService - Checkout Price Reservation

```java
package com.pricing.service;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import com.pricing.repository.PriceLockRepository;
import com.pricing.entity.PriceLock;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.util.*;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
public class PriceLockService {

    private final RedisTemplate<String, String> redisTemplate;
    private final PriceLockRepository priceLockRepo;

    @Value("${pricing.lock-duration-minutes:15}")
    private int lockDurationMinutes;

    public PriceLockService(
            RedisTemplate<String, String> redisTemplate,
            PriceLockRepository priceLockRepo) {
        this.redisTemplate = redisTemplate;
        this.priceLockRepo = priceLockRepo;
    }

    /**
     * Reserve price when customer adds item to cart
     */
    public PriceReservation reservePrice(Long userId, Long productId) {
        // Get current price
        CurrentPrice current = getCurrentPrice(productId);

        long expiresAt = System.currentTimeMillis() +
                        (lockDurationMinutes * 60 * 1000);

        // Store in Redis
        String lockKey = buildLockKey(userId, productId);
        String lockValue = String.format(
            "{\"price\":%.2f,\"version\":%d,\"lockedAt\":%d,\"expiresAt\":%d}",
            current.getPrice(), current.getVersion(),
            System.currentTimeMillis(), expiresAt
        );

        redisTemplate.opsForValue().set(
            lockKey,
            lockValue,
            lockDurationMinutes,
            TimeUnit.MINUTES
        );

        // Also store in PostgreSQL for audit/recovery
        PriceLock lock = PriceLock.builder()
            .userId(userId)
            .productId(productId)
            .lockedPrice(current.getPrice())
            .version(current.getVersion())
            .expiresAt(new Date(expiresAt))
            .build();

        priceLockRepo.save(lock);

        log.info("Price reserved [user={}, product={}, price={}, " +
                "version={}, expiresAt={}]",
            userId, productId, current.getPrice(), current.getVersion(), expiresAt);

        return PriceReservation.builder()
            .price(current.getPrice())
            .version(current.getVersion())
            .expiresAt(expiresAt)
            .build();
    }

    /**
     * Validate and retrieve locked price at checkout
     */
    public PriceReservation getLockedPrice(Long userId, Long productId) {
        String lockKey = buildLockKey(userId, productId);
        String lockValue = (String) redisTemplate.opsForValue().get(lockKey);

        if (lockValue == null) {
            return null; // Lock expired
        }

        // Parse and return
        // (In real code: use JSON parser)
        return PriceReservation.builder()
            .price(BigDecimal.valueOf(349.99))
            .version(145)
            .expiresAt(System.currentTimeMillis() + 10 * 60 * 1000)
            .build();
    }

    /**
     * Release price lock after successful checkout
     */
    public void releaseLock(Long userId, Long productId) {
        String lockKey = buildLockKey(userId, productId);
        redisTemplate.delete(lockKey);
        priceLockRepo.deleteByUserIdAndProductId(userId, productId);
        log.debug("Price lock released [user={}, product={}]", userId, productId);
    }

    /**
     * Cleanup expired locks (background job)
     */
    @Scheduled(fixedRate = 300000) // Every 5 minutes
    public void cleanupExpiredLocks() {
        List<PriceLock> expiredLocks = priceLockRepo
            .findByExpiresAtBefore(new Date());

        for (PriceLock lock : expiredLocks) {
            String lockKey = buildLockKey(lock.getUserId(), lock.getProductId());
            redisTemplate.delete(lockKey);
            priceLockRepo.delete(lock);
        }

        log.debug("Cleaned up {} expired price locks", expiredLocks.size());
    }

    private String buildLockKey(Long userId, Long productId) {
        return "price-lock:" + userId + ":" + productId;
    }

    private CurrentPrice getCurrentPrice(Long productId) {
        String key = "price:" + productId;
        Map<Object, Object> data = redisTemplate.opsForHash()
            .entries(key);

        return new CurrentPrice()
            .setPrice(new BigDecimal((String) data.get("price")))
            .setVersion(Long.parseLong((String) data.get("version")));
    }
}

@Data
@Builder
class PriceReservation {
    private BigDecimal price;
    private Long version;
    private Long expiresAt;
}

@Data
class CurrentPrice {
    private BigDecimal price;
    private Long version;
}
```

### 3. CheckoutPriceValidator - Version-Checked Validation

```java
package com.pricing.service;

import org.springframework.stereotype.Service;
import com.pricing.dto.CheckoutValidationResult;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;

@Slf4j
@Service
public class CheckoutPriceValidator {

    private final PriceLockService priceLockService;
    private final PricingRepository pricingRepo;

    public CheckoutValidationResult validateCheckoutPrice(
            Long userId,
            Long productId,
            Long lockedVersion,
            BigDecimal expectedPrice) {

        // Get locked price
        PriceReservation lockedPrice = priceLockService
            .getLockedPrice(userId, productId);

        if (lockedPrice == null) {
            // Lock expired - check if price changed
            CurrentPrice current = getCurrentPrice(productId);

            log.warn("Price lock expired [user={}, product={}], " +
                    "current version={}, expected version={}",
                userId, productId, current.getVersion(), lockedVersion);

            if (current.getPrice().compareTo(expectedPrice) == 0) {
                // Lucky! Price didn't change
                return CheckoutValidationResult.builder()
                    .status("SUCCESS")
                    .price(current.getPrice())
                    .build();
            } else {
                return CheckoutValidationResult.builder()
                    .status("LOCK_EXPIRED")
                    .price(current.getPrice())
                    .message("Price lock expired and price changed. Please review.")
                    .build();
            }
        }

        // Lock still valid
        if (lockedPrice.getVersion() == lockedVersion &&
            lockedPrice.getPrice().compareTo(expectedPrice) == 0) {
            // Success: price unchanged
            return CheckoutValidationResult.builder()
                .status("SUCCESS")
                .price(lockedPrice.getPrice())
                .build();
        }

        // Version mismatch - price updated
        if (lockedPrice.getVersion() > lockedVersion) {
            if (lockedPrice.getPrice().compareTo(expectedPrice) < 0) {
                // New price is LOWER - customer benefits!
                log.info("Price decreased for customer [user={}, old={}, new={}]",
                    userId, expectedPrice, lockedPrice.getPrice());

                return CheckoutValidationResult.builder()
                    .status("SUCCESS_NEW_PRICE")
                    .price(lockedPrice.getPrice())
                    .message("Great news! Price decreased. Applying new price.")
                    .build();
            } else {
                // New price is HIGHER - need confirmation
                log.warn("Price increased during checkout [user={}, old={}, new={}]",
                    userId, expectedPrice, lockedPrice.getPrice());

                return CheckoutValidationResult.builder()
                    .status("PRICE_CHANGED")
                    .price(lockedPrice.getPrice())
                    .requiresConfirmation(true)
                    .message("Price increased. Please confirm to continue.")
                    .build();
            }
        }

        return CheckoutValidationResult.builder()
            .status("CONFLICT")
            .message("Unexpected price state")
            .build();
    }

    private CurrentPrice getCurrentPrice(Long productId) {
        String key = "price:" + productId;
        // Fetch from Redis/cache
        return new CurrentPrice()
            .setPrice(BigDecimal.valueOf(349.99))
            .setVersion(145L);
    }
}

@Data
@Builder
class CheckoutValidationResult {
    private String status;  // SUCCESS, SUCCESS_NEW_PRICE, PRICE_CHANGED, LOCK_EXPIRED, CONFLICT
    private BigDecimal price;
    private String message;
    private boolean requiresConfirmation;
}
```

### 4. PriceAuditService - Compliance Logging

```java
package com.pricing.service;

import org.springframework.stereotype.Service;
import org.springframework.kafka.annotation.KafkaListener;
import com.pricing.repository.PriceAuditRepository;
import com.pricing.entity.PriceAuditLog;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.util.*;

@Slf4j
@Service
public class PriceAuditService {

    private final PriceAuditRepository auditRepo;
    private final ObjectMapper objectMapper;

    @KafkaListener(topics = "price-changes", groupId = "price-audit-service")
    public void auditPriceChange(String priceChangeMessage) {
        try {
            // Parse price-change event
            Map<String, Object> event = objectMapper.readValue(
                priceChangeMessage, Map.class);

            Long productId = ((Number) event.get("productId")).longValue();
            BigDecimal oldPrice = new BigDecimal(
                event.get("oldPrice").toString());
            BigDecimal newPrice = new BigDecimal(
                event.get("newPrice").toString());
            Long version = ((Number) event.get("version")).longValue();
            String reason = (String) event.get("reason");

            // Store in PostgreSQL (immutable)
            PriceAuditLog log = PriceAuditLog.builder()
                .productId(productId)
                .oldPrice(oldPrice)
                .newPrice(newPrice)
                .version(version)
                .changedBy("pricing-engine")
                .reason(reason)
                .signals(event.toString()) // Store raw signals
                .timestamp(new Date())
                .build();

            auditRepo.save(log);

            // Check compliance rules
            checkComplianceViolations(productId, oldPrice, newPrice, reason);

            log.info("Price change audited [product={}, old={}, new={}, reason={}]",
                productId, oldPrice, newPrice, reason);

        } catch (Exception e) {
            log.error("Error auditing price change", e);
        }
    }

    private void checkComplianceViolations(
            Long productId,
            BigDecimal oldPrice,
            BigDecimal newPrice,
            String reason) {

        // Calculate discount percentage
        double discountPercent = oldPrice.subtract(newPrice)
            .divide(oldPrice, 4, java.math.RoundingMode.HALF_UP)
            .multiply(BigDecimal.valueOf(100))
            .doubleValue();

        // Rule: no discount > 50%
        if (discountPercent > 50.0) {
            log.warn("COMPLIANCE ALERT: Excessive discount [product={}, " +
                    "discount={}%]", productId, discountPercent);
            // Could trigger: email alert, manual review flag, etc.
        }

        // Rule: no artificial price inflation during holidays
        Calendar cal = Calendar.getInstance();
        if (isHolidaySeason(cal) && newPrice.compareTo(oldPrice) > 0) {
            double increasePercent = newPrice.subtract(oldPrice)
                .divide(oldPrice, 4, java.math.RoundingMode.HALF_UP)
                .multiply(BigDecimal.valueOf(100))
                .doubleValue();

            if (increasePercent > 20.0) {
                log.warn("COMPLIANCE ALERT: Holiday season price inflation " +
                        "[product={}, increase={}%]", productId, increasePercent);
            }
        }

        // Rule: max 3x price changes per day
        long changesLastDay = auditRepo.countByProductIdAndTimestampAfter(
            productId,
            new Date(System.currentTimeMillis() - 24 * 60 * 60 * 1000)
        );

        if (changesLastDay > 3) {
            log.warn("COMPLIANCE ALERT: Excessive price changes [product={}, " +
                    "changes={}]", productId, changesLastDay);
        }
    }

    private boolean isHolidaySeason(Calendar cal) {
        int month = cal.get(Calendar.MONTH);
        return month >= Calendar.OCTOBER; // Q4
    }

    /**
     * Query audit trail for compliance report
     */
    public List<PriceAuditLog> getAuditTrail(
            Long productId,
            Date startDate,
            Date endDate) {

        return auditRepo.findByProductIdAndTimestampBetween(
            productId, startDate, endDate);
    }
}
```

---

## Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| Redis cache down | Fall through to PostgreSQL (slower) | Multi-region Redis, cluster mode |
| Pricing engine crash | Prices become stale | Use previous price version; circuit breaker |
| Kafka broker down | Can't publish price changes | Kafka replication factor 3; retry logic |
| Competitor API down | Can't update competitor prices | Use cached snapshots; exponential backoff |
| Race condition at checkout | Customer charged different price | Version-checked validation, price lock |
| Audit DB fails | Compliance trail broken | Kafka topic retention fallback (7 days) |

---

## Scaling Strategy

### Horizontal
- **PricingEngine:** Stateless, scale behind load balancer
- **Kafka:** 500+ partitions (one per SKU/product family)
- **Redis:** Cluster mode with 64+ nodes for global distribution

### Vertical
- Increase Redis memory for larger competitor price cache
- Increase PostgreSQL IOPS for audit log writes
- Increase Kafka retention (7 days compliance requirement)

### Caching
- Redis for current prices (TTL 30 sec)
- Redis for competitor prices (TTL 1 hour)
- PostgreSQL for audit trail (immutable)
- Kafka topic for distribution backlog

---

## Monitoring & Observability

### Key Metrics
```
pricing_engine_latency_ms (histogram)
  - p50, p95, p99
  - tags: product_category

price_change_propagation_ms (histogram)
  - time from engine decision to service update
  - tags: target_service

pricing_rule_violations (counter)
  - tags: violation_type (min_margin, max_discount)

competitor_price_freshness_minutes (gauge)
  - time since last successful scrape

price_lock_expiry_rate (gauge)
  - % of locks that expire unused
```

### Alerts
- Price change propagation > 60 seconds
- Competitor scraper failure (no update in 2 hours)
- Pricing engine error rate > 1%
- Audit log insertion latency > 1 second
- Redis memory > 80% capacity

---

## Summary Cheat Sheet

| Component | Tech | Key Details |
|-----------|------|------------|
| **Signals** | Kafka | Competitor, demand, inventory, market data |
| **Engine** | Spring Boot | Real-time rules + ML scoring |
| **Price Cache** | Redis | 30-sec TTL, versioned |
| **Price Lock** | Redis | 15-min checkout grace period |
| **Audit** | PostgreSQL | Immutable compliance trail |
| **Rules** | MongoDB | Flexible constraints & multipliers |
| **Propagation** | Kafka | <60sec SLA to all services |
| **Consistency** | Version numbers | Optimistic locking + validation |
| **SLA** | <50ms | Pricing decision latency |
| **Scaling** | Horizontal | Stateless services, Kafka partitions |

---

**End of Deep Dive**
