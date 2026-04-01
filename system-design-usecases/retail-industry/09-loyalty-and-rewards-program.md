---
title: Loyalty and Rewards Program
layout: default
---

# Loyalty and Rewards Program — Deep Dive Design

## Scenario

You're designing a loyalty program for a major retail chain. Customers earn points on purchases (1 point = $1 spent), with bonus multipliers during promotions (3x on weekends). Points can be redeemed for discounts (100 points = $1 off), and customers progress through tiers (Bronze → Silver → Gold → Platinum) based on annual spending. Points expire after 12 months, tier benefits include free shipping, birthday discounts, and early access to sales. You must support 10 million active members with point updates happening in real-time during checkout.

**Tech Stack:** Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka

---

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
- Customers earn points at time of purchase (1 point per $1, with variable multipliers)
- Real-time points balance visibility
- Redemption during checkout with guaranteed atomicity
- Points expiration after 12 months (sliding window)
- Tier calculation based on annual spend
- Tier benefits: free shipping, birthday discount, early access to sales
- Fraud prevention and idempotency guarantees
- Points ledger history for audits

### Non-Functional Requirements
- Support 10M active members
- 100K transactions per second (peak)
- Points balance queries must be sub-100ms
- Redemption must complete within 2 seconds
- 99.99% availability
- No points loss under any failure scenario

### Constraints
- Tier calculations run daily (batch)
- Points expire sliding window (12 months from earn date)
- Idempotency required (use order IDs as keys)
- Double-crediting must be impossible

---

## Capacity Estimation

| Metric | Value | Calculation |
|--------|-------|-------------|
| **Active Members** | 10M | Given |
| **Daily Transactions** | 5M | Assume 50% daily active users |
| **TPS (Peak)** | 100K | 5M transactions / 50 seconds |
| **Points Ledger Entries/Year** | 1.8B | 5M × 365 |
| **Redis Memory (Balances)** | 200GB | 10M × 20KB (customer ID + balance + metadata) |
| **PostgreSQL Ledger Growth/Year** | 500GB | 1.8B rows × ~300 bytes |
| **Tier Recalc Volume/Day** | 10M | All members, batch job |
| **Kafka Events/Second** | 200K | earn + redeem + expire events |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        API Gateway                          │
├─────────────────────────────────────────────────────────────┤
│  Loyalty Service      │ Redemption Service  │  Tier Service  │
│  (earn points)        │ (redeem points)     │ (calculate tier)|
└────────┬──────────────┴─────────────┬───────┴────────┬───────┘
         │                           │                 │
         ├──────────────────────────────────────────────┤
         │         Kafka Event Bus                      │
         │  (PointsEarned, PointsRedeemed,             │
         │   PointsExpired, TierChanged)                │
         └────────┬──────────────┬──────────────┬────────┘
                  │              │              │
         ┌────────▼──┐  ┌────────▼──┐  ┌───────▼────┐
         │ PostgreSQL │  │  Redis    │  │ MongoDB    │
         │            │  │           │  │            │
         │ Ledger     │  │ Balances  │  │ Tier Data  │
         │ Tier Info  │  │ Sessions  │  │ Analytics  │
         │ Member     │  │ Cache     │  │            │
         └────────────┘  └───────────┘  └────────────┘

         ┌──────────────────────────────────┐
         │  Scheduled Jobs (Daily)          │
         │  - Expiration Job                │
         │  - Tier Calculation Job          │
         └──────────────────────────────────┘
```

---

## Core Design Questions Answered

### 1. How do you structure the points database?

**Answer:** Use an event-sourcing approach with a ledger table. Each earn, redeem, or expiration is a row; balance is calculated as the sum of all entries for a customer. This provides auditability, idempotency, and point recovery on failures.

**Schema:**
```sql
CREATE TABLE points_ledger (
  id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  order_id VARCHAR(255) UNIQUE,  -- idempotency key
  transaction_type VARCHAR(50),  -- EARN, REDEEM, EXPIRE
  amount INT NOT NULL,
  multiplier DECIMAL(3,2) DEFAULT 1.0,
  reason VARCHAR(255),
  expiration_date DATE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  metadata JSONB,
  INDEX idx_customer_id_created (customer_id, created_at)
);

CREATE TABLE customer_point_balance (
  customer_id BIGINT PRIMARY KEY,
  current_balance INT NOT NULL,
  last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (customer_id) REFERENCES customers(id)
);
```

**Why this design:**
- Each transaction is immutable (audit trail)
- Balance can be recalculated if Redis is lost
- Supports point recovery: reverse REDEEM, add back EXPIRE
- Query for specific point sources (e.g., expiring points soon)

---

### 2. How do you calculate and award points in real-time?

**Answer:** Use a LoyaltyPointsService that fetches multiplier from promotion engine, calculates points with idempotency, and publishes a PointsEarned event. Redis caches are updated asynchronously, PostgreSQL is the source of truth.

**Idempotency:** Use order_id as unique constraint. If the same order is processed twice, the second INSERT fails gracefully (update cache instead).

---

### 3. How do you handle points redemption during checkout?

**Answer:** Implement a Kafka-driven saga pattern:
1. Checkout initiates RedemptionRequested event
2. Loyalty Service reserves points (checks balance, holds in Redis)
3. Payment processes
4. Payment success → Loyalty Service deducts points (converts reserve to ledger entry)
5. Payment failure → Loyalty Service releases reserve

**Ensures:** Points never go negative, no race conditions at checkout time.

---

### 4. How do you implement points expiration efficiently?

**Answer:** Use a scheduled batch job that runs daily. For each customer with points older than 12 months:
- Insert EXPIRE ledger entries
- Mark expired points in Redis with TTL
- Do NOT delete old rows (audit trail)

**Query for expiration:**
```sql
SELECT customer_id, amount
FROM points_ledger
WHERE transaction_type = 'EARN'
  AND created_at < CURRENT_DATE - INTERVAL '12 months'
  AND id NOT IN (
    SELECT DISTINCT ledger_id FROM points_ledger WHERE transaction_type = 'EXPIRE'
  );
```

---

### 5. How do you calculate customer tier based on annual spend?

**Answer:** Run a batch TierCalculationJob daily:
1. Sum purchases in last 12 months from Order Service
2. Determine tier: Bronze (0-499), Silver (500-1999), Gold (2000-4999), Platinum (5000+)
3. If tier changed, publish TierChanged event
4. Cache tier + benefits in Redis for fast reads

---

### 6. How do you prevent points fraud or double-crediting?

**Answer:**
- **Idempotency:** order_id as UNIQUE constraint in points_ledger
- **Validation:** Verify order exists and amount matches order total
- **Rate Limiting:** Limit rapid point redemption attempts per customer
- **Anomaly Detection:** Publish events to monitoring pipeline (unusual point earn patterns)
- **Code signing:** Promo multipliers signed by promotion service

---

## Microservices Breakdown

| Service | Responsibility | Data Owned |
|---------|-----------------|-----------|
| **Loyalty Points Service** | Earn points, balance queries, ledger | Points ledger, customer balance |
| **Redemption Service** | Point redemption, reserve/release | Redemption holds in Redis |
| **Tier Service** | Calculate tier, manage tier benefits | Tier metadata, tier upgrade events |
| **Points Expiration Service** | Scheduled expiration job | Ledger entries for expired points |
| **Points Analytics Service** | Dashboards, reports | Pre-aggregated metrics (MongoDB) |

---

## Database Design

### PostgreSQL: Ledger (Source of Truth)
```sql
-- Points ledger (immutable event stream)
CREATE TABLE points_ledger (
  id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  order_id VARCHAR(255),
  transaction_type VARCHAR(50) NOT NULL, -- EARN, REDEEM, EXPIRE, ADJUST
  amount INT NOT NULL,
  multiplier DECIMAL(4,2) DEFAULT 1.0,
  reason VARCHAR(255),
  expiration_date DATE,
  promo_code_id BIGINT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  metadata JSONB,
  INDEX idx_customer_id (customer_id),
  INDEX idx_created_at (created_at),
  UNIQUE(order_id, transaction_type)
);

-- Customer tier info
CREATE TABLE customer_tiers (
  customer_id BIGINT PRIMARY KEY,
  tier_name VARCHAR(50) NOT NULL,
  annual_spend DECIMAL(12,2),
  tier_effective_date DATE,
  tier_benefits JSONB,
  FOREIGN KEY (customer_id) REFERENCES customers(id),
  INDEX idx_tier_name (tier_name)
);

-- Redemption holds (transient during checkout)
CREATE TABLE redemption_holds (
  id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  order_id VARCHAR(255),
  points_held INT NOT NULL,
  hold_status VARCHAR(50), -- HELD, RELEASED, CONVERTED
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expires_at TIMESTAMP,
  UNIQUE(order_id),
  INDEX idx_customer_id_status (customer_id, hold_status)
);
```

### MongoDB: Analytics (Denormalized Reads)
```json
{
  "_id": "customer_123_2026-04-01",
  "customer_id": 123,
  "date": "2026-04-01",
  "tier": "Gold",
  "points_earned_today": 250,
  "points_redeemed_today": 100,
  "points_expired_today": 50,
  "current_balance": 5000,
  "transactions_count": 5,
  "purchase_amount": 250.00,
  "multiplier_applied": 1.0
}
```

---

## Redis Data Structures

```
# Customer balance (cache, updated on each points event)
points:balance:{customer_id} → INT (current balance)
points:balance:{customer_id}:ttl → TTL of 24h (refresh daily)

# Tier cache (updated daily)
tier:{customer_id} → HASH {
  tier: "Gold",
  annual_spend: 2500.00,
  benefits: ["free_shipping", "birthday_discount", "early_access"],
  updated_at: 2026-04-01T10:00:00Z
}

# Redemption hold (during checkout saga)
redemption:hold:{order_id} → INT (points held)
redemption:hold:{order_id}:ttl → TTL of 5m (checkout timeout)

# Rate limiting (fraud prevention)
redeem_attempts:{customer_id}:{minute} → INT (count)
redeem_attempts:{customer_id}:{minute}:ttl → TTL of 60s

# Session cache (for dashboard)
session:{session_id} → HASH {
  customer_id: 123,
  balance: 5000,
  tier: "Gold",
  last_refresh: 2026-04-01T10:00:00Z
}
```

---

## Kafka Event Flow

```
Topic: loyalty-events (3 partitions, key=customer_id)

Events Published:
1. PointsEarned {
     customerId, orderId, amount, multiplier, reason, timestamp
   }
2. PointsRedeemed {
     customerId, orderId, pointsUsed, discountApplied, timestamp
   }
3. PointsExpired {
     customerId, pointsExpired, expirationDate, timestamp
   }
4. TierChanged {
     customerId, oldTier, newTier, annualSpend, timestamp
   }

Consumers:
- LoyaltyPointsService (update balance cache, publish to analytics)
- PointsAnalyticsService (aggregate daily metrics to MongoDB)
- NotificationService (send "points earned" push notification)
- FraudDetectionService (detect anomalies)
```

---

## Implementation Code

### 1. Loyalty Points Service

```java
package com.retail.loyalty.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.data.redis.core.RedisTemplate;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.Optional;
import java.util.concurrent.TimeUnit;

@Service
public class LoyaltyPointsService {

    private static final Logger log = LoggerFactory.getLogger(LoyaltyPointsService.class);
    private static final long CACHE_TTL_HOURS = 24;

    private final PointsLedgerRepository ledgerRepository;
    private final CustomerBalanceRepository balanceRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final PromotionClient promotionClient;
    private final KafkaTemplate<String, PointsEarned> kafkaTemplate;
    private final ObjectMapper objectMapper;

    public LoyaltyPointsService(
            PointsLedgerRepository ledgerRepository,
            CustomerBalanceRepository balanceRepository,
            RedisTemplate<String, String> redisTemplate,
            PromotionClient promotionClient,
            KafkaTemplate<String, PointsEarned> kafkaTemplate,
            ObjectMapper objectMapper) {
        this.ledgerRepository = ledgerRepository;
        this.balanceRepository = balanceRepository;
        this.redisTemplate = redisTemplate;
        this.promotionClient = promotionClient;
        this.kafkaTemplate = kafkaTemplate;
        this.objectMapper = objectMapper;
    }

    @Transactional
    public PointsEarningResult earnPoints(Long customerId, String orderId,
                                          BigDecimal orderAmount, String promoCode) {
        try {
            // 1. Fetch multiplier from promotion service
            BigDecimal multiplier = promotionClient.getPointsMultiplier(promoCode);

            // 2. Calculate points
            int basePoints = orderAmount.intValue();  // $1 = 1 point
            int earnedPoints = (int) (basePoints * multiplier.doubleValue());

            // 3. Check idempotency: does order_id already exist?
            Optional<PointsLedger> existing = ledgerRepository.findByOrderId(orderId);
            if (existing.isPresent()) {
                log.warn("Order {} already processed for points", orderId);
                return PointsEarningResult.duplicate(existing.get().getAmount());
            }

            // 4. Create ledger entry
            PointsLedger ledger = new PointsLedger();
            ledger.setCustomerId(customerId);
            ledger.setOrderId(orderId);
            ledger.setTransactionType("EARN");
            ledger.setAmount(earnedPoints);
            ledger.setMultiplier(multiplier);
            ledger.setReason("Purchase");
            ledger.setExpirationDate(LocalDate.now().plusMonths(12));
            ledger.setPromoCodeId(promoCode != null ? promotionClient.getPromoCodeId(promoCode) : null);
            ledger.setMetadata(Map.of(
                "basePoints", basePoints,
                "multiplier", multiplier.toString(),
                "orderAmount", orderAmount.toString()
            ));

            PointsLedger saved = ledgerRepository.save(ledger);

            // 5. Update balance in PostgreSQL
            updateCustomerBalance(customerId);

            // 6. Update cache in Redis
            updateBalanceCache(customerId);

            // 7. Publish event
            PointsEarned event = new PointsEarned(customerId, orderId, earnedPoints,
                                                    multiplier.doubleValue());
            kafkaTemplate.send("loyalty-events", String.valueOf(customerId), event);

            log.info("Earned {} points for customer {} from order {}",
                     earnedPoints, customerId, orderId);

            return PointsEarningResult.success(earnedPoints);

        } catch (Exception e) {
            log.error("Error earning points for customer {}, order {}", customerId, orderId, e);
            throw new PointsException("Failed to earn points", e);
        }
    }

    public int getPointsBalance(Long customerId) {
        // 1. Try cache first
        String cacheKey = "points:balance:" + customerId;
        String cached = redisTemplate.opsForValue().get(cacheKey);

        if (cached != null) {
            return Integer.parseInt(cached);
        }

        // 2. Fallback to database
        int balance = ledgerRepository.sumPointsByCustomerId(customerId);

        // 3. Update cache
        redisTemplate.opsForValue().set(
            cacheKey,
            String.valueOf(balance),
            CACHE_TTL_HOURS,
            TimeUnit.HOURS
        );

        return balance;
    }

    @Transactional
    private void updateCustomerBalance(Long customerId) {
        int balance = ledgerRepository.sumPointsByCustomerId(customerId);

        CustomerBalance existing = balanceRepository.findById(customerId).orElse(null);
        if (existing != null) {
            existing.setCurrentBalance(balance);
            existing.setLastUpdated(LocalDateTime.now());
            balanceRepository.save(existing);
        } else {
            CustomerBalance newBalance = new CustomerBalance();
            newBalance.setCustomerId(customerId);
            newBalance.setCurrentBalance(balance);
            balanceRepository.save(newBalance);
        }
    }

    private void updateBalanceCache(Long customerId) {
        int balance = ledgerRepository.sumPointsByCustomerId(customerId);
        String cacheKey = "points:balance:" + customerId;
        redisTemplate.opsForValue().set(
            cacheKey,
            String.valueOf(balance),
            CACHE_TTL_HOURS,
            TimeUnit.HOURS
        );
    }
}
```

### 2. Redemption Saga Service

```java
package com.retail.loyalty.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.kafka.annotation.KafkaListener;

@Service
public class RedemptionSagaService {

    private static final Logger log = LoggerFactory.getLogger(RedemptionSagaService.class);
    private static final long HOLD_TIMEOUT_SECONDS = 300;  // 5 minutes

    private final PointsLedgerRepository ledgerRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final RedisTemplate<String, Long> redisLongTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final CheckoutService checkoutService;

    @Transactional
    public RedemptionResponse initiateRedemption(Long customerId, String orderId, int pointsToRedeem) {
        try {
            // 1. Check balance
            int currentBalance = getBalance(customerId);
            if (currentBalance < pointsToRedeem) {
                return RedemptionResponse.insufficient();
            }

            // 2. Place hold in Redis (atomic)
            String holdKey = "redemption:hold:" + orderId;
            Boolean holdPlaced = redisTemplate.opsForValue()
                .setIfAbsent(holdKey, String.valueOf(pointsToRedeem));

            if (!holdPlaced) {
                log.warn("Redemption hold already exists for order {}", orderId);
                return RedemptionResponse.duplicate();
            }

            // 3. Set expiration on hold (timeout)
            redisTemplate.expire(holdKey, HOLD_TIMEOUT_SECONDS, TimeUnit.SECONDS);

            // 4. Save hold record in PostgreSQL
            ReddemptionHold hold = new ReddemptionHold();
            hold.setCustomerId(customerId);
            hold.setOrderId(orderId);
            hold.setPointsHeld(pointsToRedeem);
            hold.setHoldStatus("HELD");
            hold.setExpiresAt(LocalDateTime.now().plusSeconds(HOLD_TIMEOUT_SECONDS));
            redemptionHoldRepository.save(hold);

            // 5. Publish RedemptionRequested event
            RedemptionRequested event = new RedemptionRequested(customerId, orderId, pointsToRedeem);
            kafkaTemplate.send("redemption-events", String.valueOf(customerId), event);

            log.info("Placed hold for {} points on order {}", pointsToRedeem, orderId);

            return RedemptionResponse.success(pointsToRedeem);

        } catch (Exception e) {
            log.error("Error initiating redemption", e);
            throw new RedemptionException("Failed to initiate redemption", e);
        }
    }

    @KafkaListener(topics = "payment-events", groupId = "loyalty-redemption")
    public void handlePaymentSuccess(PaymentSuccessEvent event) {
        try {
            String orderId = event.getOrderId();
            Long customerId = event.getCustomerId();
            int pointsRedeemed = getHeldPoints(orderId);

            if (pointsRedeemed <= 0) {
                log.warn("No hold found for order {}", orderId);
                return;
            }

            // 1. Convert hold to ledger entry (REDEEM transaction)
            PointsLedger ledger = new PointsLedger();
            ledger.setCustomerId(customerId);
            ledger.setOrderId(orderId);
            ledger.setTransactionType("REDEEM");
            ledger.setAmount(-pointsRedeemed);  // Negative to indicate deduction
            ledger.setReason("Redemption for discount");
            ledgerRepository.save(ledger);

            // 2. Update balance cache
            updateBalanceCache(customerId);

            // 3. Release Redis hold
            String holdKey = "redemption:hold:" + orderId;
            redisTemplate.delete(holdKey);

            // 4. Update hold record to CONVERTED
            ReddemptionHold hold = redemptionHoldRepository.findByOrderId(orderId);
            hold.setHoldStatus("CONVERTED");
            redemptionHoldRepository.save(hold);

            // 5. Publish PointsRedeemed event
            PointsRedeemed redemptionEvent = new PointsRedeemed(
                customerId, orderId, pointsRedeemed, event.getDiscount()
            );
            kafkaTemplate.send("loyalty-events", String.valueOf(customerId), redemptionEvent);

            log.info("Redeemed {} points for order {}", pointsRedeemed, orderId);

        } catch (Exception e) {
            log.error("Error processing payment success", e);
        }
    }

    @KafkaListener(topics = "payment-events", groupId = "loyalty-redemption")
    public void handlePaymentFailure(PaymentFailureEvent event) {
        try {
            String orderId = event.getOrderId();

            // 1. Release Redis hold
            String holdKey = "redemption:hold:" + orderId;
            redisTemplate.delete(holdKey);

            // 2. Update hold record to RELEASED
            ReddemptionHold hold = redemptionHoldRepository.findByOrderId(orderId);
            if (hold != null) {
                hold.setHoldStatus("RELEASED");
                redemptionHoldRepository.save(hold);
            }

            log.info("Released hold for order {} due to payment failure", orderId);

        } catch (Exception e) {
            log.error("Error handling payment failure", e);
        }
    }

    private int getBalance(Long customerId) {
        String cacheKey = "points:balance:" + customerId;
        String cached = redisTemplate.opsForValue().get(cacheKey);

        if (cached != null) {
            return Integer.parseInt(cached);
        }

        return ledgerRepository.sumPointsByCustomerId(customerId);
    }

    private int getHeldPoints(String orderId) {
        String holdKey = "redemption:hold:" + orderId;
        String held = redisTemplate.opsForValue().get(holdKey);
        return held != null ? Integer.parseInt(held) : 0;
    }

    private void updateBalanceCache(Long customerId) {
        int balance = ledgerRepository.sumPointsByCustomerId(customerId);
        String cacheKey = "points:balance:" + customerId;
        redisTemplate.opsForValue().set(cacheKey, String.valueOf(balance), 24, TimeUnit.HOURS);
    }
}
```

### 3. Points Expiration Job

```java
package com.retail.loyalty.job;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class PointsExpirationJob {

    private static final Logger log = LoggerFactory.getLogger(PointsExpirationJob.class);
    private static final int BATCH_SIZE = 10000;

    private final PointsLedgerRepository ledgerRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, PointsExpired> kafkaTemplate;

    // Runs daily at 2 AM
    @Scheduled(cron = "0 2 * * *")
    @Transactional
    public void expirePoints() {
        log.info("Starting points expiration job");

        LocalDate expirationThreshold = LocalDate.now().minusMonths(12);
        int totalExpired = 0;

        try {
            // 1. Find all EARN entries that are older than 12 months
            List<PointsLedger> expiredEntries = ledgerRepository
                .findExpiredPointsNotYetProcessed(expirationThreshold, BATCH_SIZE);

            while (!expiredEntries.isEmpty()) {
                // 2. For each expired entry, create an EXPIRE ledger entry
                for (PointsLedger entry : expiredEntries) {
                    createExpirationEntry(entry);
                    totalExpired += entry.getAmount();
                }

                // 3. Invalidate customer balance cache
                expiredEntries.forEach(entry -> {
                    String cacheKey = "points:balance:" + entry.getCustomerId();
                    redisTemplate.delete(cacheKey);
                });

                // 4. Fetch next batch
                expiredEntries = ledgerRepository
                    .findExpiredPointsNotYetProcessed(expirationThreshold, BATCH_SIZE);
            }

            log.info("Completed points expiration job. Total points expired: {}", totalExpired);

        } catch (Exception e) {
            log.error("Error in points expiration job", e);
            throw new ExpirationException("Points expiration failed", e);
        }
    }

    @Transactional
    private void createExpirationEntry(PointsLedger originalEntry) {
        Long customerId = originalEntry.getCustomerId();

        // Create EXPIRE entry
        PointsLedger expireEntry = new PointsLedger();
        expireEntry.setCustomerId(customerId);
        expireEntry.setTransactionType("EXPIRE");
        expireEntry.setAmount(-originalEntry.getAmount());  // Negative to indicate deduction
        expireEntry.setReason("Points expired after 12 months");
        expireEntry.setMetadata(Map.of(
            "original_ledger_id", originalEntry.getId().toString(),
            "original_earn_date", originalEntry.getCreatedAt().toString()
        ));

        ledgerRepository.save(expireEntry);

        // Publish PointsExpired event
        PointsExpired event = new PointsExpired(
            customerId,
            originalEntry.getAmount(),
            originalEntry.getCreatedAt().toLocalDate()
        );
        kafkaTemplate.send("loyalty-events", String.valueOf(customerId), event);

        log.debug("Expired {} points for customer {}", originalEntry.getAmount(), customerId);
    }
}
```

### 4. Tier Calculation Service

```java
package com.retail.loyalty.service;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class TierCalculationService {

    private static final Logger log = LoggerFactory.getLogger(TierCalculationService.class);
    private static final int BATCH_SIZE = 50000;

    private final CustomerTierRepository tierRepository;
    private final OrderServiceClient orderClient;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, TierChanged> kafkaTemplate;
    private final ObjectMapper objectMapper;

    // Runs daily at 1 AM
    @Scheduled(cron = "0 1 * * *")
    @Transactional
    public void calculateTiers() {
        log.info("Starting tier calculation job");

        LocalDate startDate = LocalDate.now().minusMonths(12);
        int totalUpdates = 0;
        int offset = 0;

        try {
            while (true) {
                // 1. Fetch batch of customers
                List<Long> customerIds = tierRepository.findAllCustomerIds(offset, BATCH_SIZE);

                if (customerIds.isEmpty()) break;

                // 2. For each customer, calculate annual spend
                for (Long customerId : customerIds) {
                    BigDecimal annualSpend = orderClient.getAnnualSpend(customerId, startDate);
                    String newTier = calculateTier(annualSpend);

                    // 3. Get current tier
                    CustomerTier current = tierRepository.findById(customerId).orElse(null);
                    String oldTier = current != null ? current.getTierName() : "Bronze";

                    // 4. If tier changed, update and publish event
                    if (!oldTier.equals(newTier)) {
                        updateTier(customerId, oldTier, newTier, annualSpend);
                        totalUpdates++;
                    }
                }

                offset += BATCH_SIZE;
            }

            log.info("Completed tier calculation. Total tier changes: {}", totalUpdates);

        } catch (Exception e) {
            log.error("Error in tier calculation job", e);
            throw new TierCalculationException("Tier calculation failed", e);
        }
    }

    @Transactional
    private void updateTier(Long customerId, String oldTier, String newTier,
                            BigDecimal annualSpend) {
        // 1. Update database
        CustomerTier tier = new CustomerTier();
        tier.setCustomerId(customerId);
        tier.setTierName(newTier);
        tier.setAnnualSpend(annualSpend);
        tier.setTierEffectiveDate(LocalDate.now());
        tier.setTierBenefits(getTierBenefits(newTier));

        tierRepository.save(tier);

        // 2. Update Redis cache
        String cacheKey = "tier:" + customerId;
        Map<String, Object> tierData = Map.of(
            "tier", newTier,
            "annual_spend", annualSpend.toString(),
            "benefits", getTierBenefits(newTier),
            "updated_at", LocalDateTime.now().toString()
        );

        redisTemplate.opsForHash().putAll(cacheKey,
            objectMapper.convertValue(tierData, Map.class));
        redisTemplate.expire(cacheKey, 24, TimeUnit.HOURS);

        // 3. Publish TierChanged event
        TierChanged event = new TierChanged(customerId, oldTier, newTier, annualSpend);
        kafkaTemplate.send("loyalty-events", String.valueOf(customerId), event);

        log.info("Updated tier for customer {} from {} to {}", customerId, oldTier, newTier);
    }

    private String calculateTier(BigDecimal annualSpend) {
        if (annualSpend.compareTo(BigDecimal.valueOf(5000)) >= 0) {
            return "Platinum";
        } else if (annualSpend.compareTo(BigDecimal.valueOf(2000)) >= 0) {
            return "Gold";
        } else if (annualSpend.compareTo(BigDecimal.valueOf(500)) >= 0) {
            return "Silver";
        }
        return "Bronze";
    }

    private Map<String, List<String>> getTierBenefits(String tier) {
        return Map.of(
            "Bronze", List.of("standard_rewards"),
            "Silver", List.of("free_shipping", "birthday_discount"),
            "Gold", List.of("free_shipping", "birthday_discount", "early_access", "2x_points"),
            "Platinum", List.of("free_shipping", "birthday_discount", "early_access", "3x_points",
                                "vip_support", "exclusive_events")
        ).getOrDefault(tier, List.of());
    }
}
```

### 5. Points Ledger Repository

```java
package com.retail.loyalty.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import java.time.LocalDate;
import java.util.List;
import java.util.Optional;

@Repository
public interface PointsLedgerRepository extends JpaRepository<PointsLedger, Long> {

    Optional<PointsLedger> findByOrderId(String orderId);

    @Query("""
        SELECT COALESCE(SUM(pl.amount), 0)
        FROM PointsLedger pl
        WHERE pl.customerId = :customerId
          AND pl.transactionType IN ('EARN', 'REDEEM', 'EXPIRE', 'ADJUST')
    """)
    int sumPointsByCustomerId(@Param("customerId") Long customerId);

    @Query("""
        SELECT pl FROM PointsLedger pl
        WHERE pl.transactionType = 'EARN'
          AND pl.createdAt < :threshold
          AND pl.id NOT IN (
            SELECT pl2.id FROM PointsLedger pl2
            WHERE pl2.transactionType = 'EXPIRE'
          )
        LIMIT :limit
    """)
    List<PointsLedger> findExpiredPointsNotYetProcessed(
        @Param("threshold") LocalDate threshold,
        @Param("limit") int limit
    );

    @Query("""
        SELECT pl FROM PointsLedger pl
        WHERE pl.customerId = :customerId
        ORDER BY pl.createdAt DESC
        LIMIT :limit
    """)
    List<PointsLedger> findRecentByCustomerId(
        @Param("customerId") Long customerId,
        @Param("limit") int limit
    );
}
```

---

## Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| **Redis cache miss at peak** | Balance queries hit PostgreSQL | Warm cache daily; use fallback queries |
| **Double earn via order reprocess** | Customer gains extra points | UNIQUE constraint on (order_id, transaction_type) |
| **Redemption saga timeout** | Points held indefinitely | Redis TTL + scheduled cleanup job |
| **Database failure during earn** | Points lost | Event sourcing—replay events after recovery |
| **Tier calculation job failure** | Stale tier benefits | Retry with exponential backoff; alert ops |
| **Kafka broker unavailable** | Events not published | Store events in PostgreSQL, replay when recovered |

---

## Scaling Strategy

### Horizontal Scaling

1. **Ledger sharding by customer_id:** Partition PostgreSQL table by customer_id mod N
2. **Redis cluster:** Use Redis Cluster for distributed caching (replicas for high availability)
3. **Kafka partitions:** One partition per customer_id shard to maintain ordering
4. **Service replicas:** Run Loyalty Service with N instances, load balanced by API Gateway

### Vertical Scaling

1. **PostgreSQL:** Add read replicas for balance queries; use connection pooling (HikariCP)
2. **Redis:** Increase memory; use Redis pipelining for batch operations
3. **Kafka:** Increase partitions and broker count based on throughput

### Pre-aggregation

- Daily snapshots in MongoDB for analytics (avoid expensive joins)
- Cache tier benefits in Redis to reduce database lookups

---

## Monitoring & Observability

### Metrics

```java
// Micrometer metrics
@Timed(value = "loyalty.points.earned", description = "Points earned per transaction")
public PointsEarningResult earnPoints(...) { ... }

meterRegistry.counter("loyalty.points.redeemed",
    "customer_tier", tier).increment(pointsRedeemed);

meterRegistry.gauge("loyalty.balance.cache_hits",
    atomicInteger::get);
```

### Logs

- Log all points transactions at INFO level (customerId, amount, orderId)
- Log cache hits/misses at DEBUG level
- Log reconciliation mismatches at ERROR level

### Alerts

- Points balance cache hit rate < 80%
- Redemption saga timeout > 100 per minute
- PostgreSQL query latency > 500ms
- Kafka lag > 1 minute

---

## Summary Cheat Sheet

| Component | Choice | Why |
|-----------|--------|-----|
| **Points storage** | PostgreSQL ledger (event sourcing) | Auditability, recoverability, idempotency |
| **Balance cache** | Redis with 24h TTL | Sub-100ms reads, fallback to DB |
| **Redemption pattern** | Kafka saga (reserve→pay→deduct) | Atomic, handles payment failures |
| **Expiration** | Daily batch job with ledger entries | Sliding window, audit trail, no data loss |
| **Tier calculation** | Daily batch, cached in Redis | Efficient, read-optimized |
| **Fraud prevention** | Idempotency key + rate limiting | Prevents double-crediting and abuse |
| **Tier benefits** | MongoDB (denormalized) or Redis HASH | Fast lookups, no joins |

---

**Generated:** 2026-04-01 | **Tech Stack:** Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka
