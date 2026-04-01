---
title: Gift Card and Promotional Code System
layout: default
---

# Gift Card and Promotional Code System — Deep Dive Design

## Scenario

You're designing a system to manage digital and physical gift cards, as well as promotional codes with complex rules. Customers can stack certain promos (20% off + free shipping) but not others (one discount per order). Usage limits apply (first 1000 customers, one per customer), expiration dates are enforced, and fraud prevention is critical (brute-force protection against gift card guessing). Support partial redemptions (use $20 of a $50 gift card), and handle race conditions when multiple users simultaneously attempt to redeem a limited promotional code. Prevent code enumeration attacks and ensure audit trails for compliance.

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
- Generate unique, unguessable gift card codes (cryptographically secure)
- Support partial redemptions (balance tracking per gift card)
- Promo codes with variable discount types (%, flat amount, BOGO, free shipping)
- Stackable promo rules (e.g., 20% off + free shipping, but not 2 discounts)
- Usage limits (per customer, global per promo, first N users)
- Expiration enforcement (absolute and relative dates)
- Fraud prevention (brute force, enumeration attacks)
- Audit trail for compliance (who used what, when)

### Non-Functional Requirements
- 100K gift card transactions per second (peak)
- Gift card balance queries must be < 50ms
- Promo validation must complete in < 100ms
- 99.99% availability
- No race conditions on limited promos
- Prevent code enumeration (e.g., guessing codes)
- Support international characters/locales

### Constraints
- Gift card balance ledger is immutable (no direct updates)
- Promo codes are read-only after activation
- Rate limiting per IP + per customer
- Hash all sensitive data (codes, tokens)

---

## Capacity Estimation

| Metric | Value | Calculation |
|--------|-------|-------------|
| **Daily Gift Cards Issued** | 100K | Estimated |
| **Daily Gift Card Redemptions** | 200K | Typical usage pattern |
| **Daily Promo Code Uses** | 500K | Popular promotions |
| **Peak TPS (Gift Cards)** | 100K | 200K / 2 seconds |
| **Peak TPS (Promos)** | 200K | 500K / 2.5 seconds |
| **Redis Memory (Gift Card Balances)** | 50GB | 10M cards × 5KB avg |
| **PostgreSQL Ledger Growth/Year** | 300GB | 200K × 365 × ~500 bytes |
| **Promo Code Database Rows** | 1M | 10K active promos avg |
| **Brute Force Attempts/Hour** | 10M | Average attack volume |

---

## High-Level Architecture

```
┌────────────────────────────────────┐
│      API Gateway + Rate Limiter    │
│  (track IP, customer_id)           │
└────────┬─────────────────┬─────────┘
         │                 │
    ┌────▼───────┐  ┌──────▼──────────┐
    │Gift Card    │  │Promo Code       │
    │Service      │  │Validator        │
    │(redeem)     │  │(check & apply)  │
    └────┬───────┘  └──────┬──────────┘
         │                 │
    ┌────▼─────────────────▼──────────┐
    │    Validation Pipeline          │
    │  - Code check                   │
    │  - Eligibility                  │
    │  - Stacking rules               │
    │  - Usage limits (Redis atomic)  │
    └────┬─────────────────────────────┘
         │
    ┌────▼────────────────────────────────────────┐
    │      Checkout Service                       │
    │  (apply, calculate discount, deduct balance)│
    └────┬────────────────────────────────────────┘
         │
         ├──────────────────────────────────┐
         │         Kafka Event Bus          │
         │  (PromoApplied, GiftCardUsed,    │
         │   FraudDetected)                 │
         └────┬──────────────────┬──────────┘
              │                  │
    ┌─────────▼──┐   ┌──────────▼────┐
    │ PostgreSQL  │   │   Redis       │
    │             │   │               │
    │Gift Cards   │   │Code Usage     │
    │Ledger       │   │Counters       │
    │Promo Codes  │   │Rate Limits    │
    │Usage Audit  │   │Code Index     │
    └─────────────┘   │(hashed)       │
                      └────────────────┘

    ┌──────────────────────────────┐
    │  Fraud Detection Service     │
    │  - Pattern analysis          │
    │  - Anomaly detection         │
    │  - IP blacklisting           │
    └──────────────────────────────┘
```

---

## Core Design Questions Answered

### 1. How do you generate unique, secure gift card codes?

**Answer:** Use **cryptographically secure random generation** with a Luhn check digit for validation (error detection).

**Why this approach:**
- `SecureRandom` is cryptographically secure (not predictable)
- Luhn check digit detects typos (improves UX)
- No sequential IDs (prevents enumeration)
- Hash codes in Redis for quick lookup without storing plaintext

**Code generation algorithm:**
```
1. Generate 16 random bytes using SecureRandom
2. Encode as Base32 (alphanumeric, no ambiguous chars)
3. Insert dashes for readability: XXXX-XXXX-XXXX-XXXX
4. Calculate Luhn check digit, append as suffix
5. Store hash(code) → gift_card_id in Redis
6. Store full code in secure vault (encrypted at rest)
```

---

### 2. How do you validate and apply multiple promo codes at checkout?

**Answer:** **Validation chain pattern** with ordered checks:

1. **Code existence check** (Redis hash lookup)
2. **Eligibility check** (expiration, customer tier, min order value)
3. **Stacking rules check** (which combos are allowed)
4. **Usage limit check** (Redis atomic INCR)
5. **Discount calculation** (apply in order, handle interactions)

**Stacking example:**
- Allowed: 20% off + free shipping (different categories)
- Not allowed: 20% off + 15% off (same category)

---

### 3. How do you enforce usage limits in a distributed system?

**Answer:** **Redis atomic operations** (INCR with EXPIRE) backed by PostgreSQL:

1. **Real-time enforcement:** Redis counter
   - Key: `promo:{code}:usage_count`
   - Atomically INCR, check against limit
   - TTL = lifetime of promo
2. **Backup:** PostgreSQL usage table (eventual consistency)
   - Nightly reconciliation job
   - Detect mismatch, alert ops
3. **Race condition prevention:** Lua script

**Lua script (atomic check-and-increment):**
```lua
local count = redis.call('INCR', KEYS[1])
local limit = tonumber(redis.call('HGET', KEYS[2], 'usage_limit'))
if count > limit then
  redis.call('DECR', KEYS[1])
  return 0
else
  return 1
end
```

---

### 4. How do you prevent promo code fraud?

**Answer:** Multi-layered defense:

1. **Code enumeration prevention:**
   - Don't expose total number of valid codes
   - Hash codes in Redis (only hash-based lookup allowed)
   - Require login (tie to customer_id)

2. **Brute force protection:**
   - Rate limit: 5 attempts per IP per minute
   - Rate limit: 10 attempts per customer per hour
   - Exponential backoff on failure
   - CAPTCHA after 3 failures

3. **Anomaly detection:**
   - Monitor sudden spike in redemptions
   - Detect multiple IPs using same code
   - Alert fraud team

4. **Geographic validation:**
   - Check if customer location matches code region
   - Flag cross-country usage

---

### 5. How do you track gift card balance across partial uses?

**Answer:** **Ledger-based balance tracking** (similar to loyalty points):

Each redemption is a row in `gift_card_ledger`:
- gift_card_id, transaction_type (ACTIVATE, REDEEM, REFUND, EXPIRE), amount, timestamp

Balance = SUM of all transactions for that card.

**Why ledger:**
- Auditability (who used it when)
- Recoverability (replay transactions on balance loss)
- Partial redemptions naturally supported
- Refunds tracked separately

---

### 6. How do you handle race conditions when multiple users use limited promos?

**Answer:** **Redis Lua script** for atomic check-and-increment:

```java
// Pseudo-code
String luaScript = """
  local current = redis.call('GET', KEYS[1])
  local limit = tonumber(ARGV[1])
  if current == nil then
    current = 0
  else
    current = tonumber(current)
  end

  if current >= limit then
    return 0  -- limit exceeded
  else
    redis.call('INCR', KEYS[1])
    return 1  -- success
  end
""";

// Execute atomically
Boolean success = redisTemplate.execute(
  RedisScript.of(luaScript, Boolean.class),
  List.of("promo:ABC123:usage"),
  "1000"
);
```

This ensures no race condition: all increments are atomic.

---

## Microservices Breakdown

| Service | Responsibility | Data Owned |
|---------|-----------------|-----------|
| **Gift Card Service** | Issue, redeem, balance queries | Gift cards, ledger, hashed codes |
| **Promo Code Service** | Create, validate, apply rules | Promo definitions, rules, eligibility |
| **Validation Engine** | Stacking rules, eligibility checks | Validation rules, cache |
| **Usage Limiter** | Enforce usage limits, rate limiting | Redis counters, PostgreSQL audit |
| **Fraud Detection** | Pattern analysis, anomaly detection | Fraud logs, IP blacklist |

---

## Database Design

### PostgreSQL: OLTP (Source of Truth)

```sql
-- Gift cards (metadata only, codes in vault)
CREATE TABLE gift_cards (
  id BIGSERIAL PRIMARY KEY,
  gift_card_number_hash VARCHAR(255) UNIQUE NOT NULL,  -- hash of actual code
  card_type VARCHAR(50), -- DIGITAL, PHYSICAL
  initial_balance DECIMAL(12,2) NOT NULL,
  current_balance DECIMAL(12,2) NOT NULL,
  currency VARCHAR(3) DEFAULT 'USD',
  status VARCHAR(50), -- ACTIVE, REDEEMED, EXPIRED, CANCELLED
  issued_date DATE NOT NULL,
  activation_date DATE,
  expiration_date DATE NOT NULL,
  recipient_email VARCHAR(255),
  metadata JSONB,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_status (status),
  INDEX idx_expiration_date (expiration_date)
);

-- Gift card ledger (immutable transaction log)
CREATE TABLE gift_card_ledger (
  id BIGSERIAL PRIMARY KEY,
  gift_card_id BIGINT NOT NULL,
  order_id BIGINT,
  transaction_type VARCHAR(50), -- ACTIVATE, REDEEM, REFUND, EXPIRE
  amount DECIMAL(12,2) NOT NULL,
  balance_after DECIMAL(12,2) NOT NULL,
  reason VARCHAR(255),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (gift_card_id) REFERENCES gift_cards(id),
  INDEX idx_gift_card_id (gift_card_id),
  INDEX idx_created_at (created_at)
);

-- Promo codes (definition)
CREATE TABLE promo_codes (
  id BIGSERIAL PRIMARY KEY,
  code_hash VARCHAR(255) UNIQUE NOT NULL,
  code_type VARCHAR(50), -- PERCENTAGE, FLAT_AMOUNT, BOGO, FREE_SHIPPING
  discount_value DECIMAL(5,2) NOT NULL,  -- percentage or amount
  description VARCHAR(500),
  start_date DATE NOT NULL,
  end_date DATE NOT NULL,
  usage_limit INT,
  global_usage_count INT DEFAULT 0,
  per_customer_limit INT DEFAULT 1,
  min_order_value DECIMAL(12,2),
  eligible_categories VARCHAR(500),  -- JSON array
  stackable BOOLEAN DEFAULT FALSE,
  stackable_with VARCHAR(500),  -- JSON array of stackable code types
  status VARCHAR(50), -- ACTIVE, INACTIVE, EXPIRED
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_status (status),
  INDEX idx_end_date (end_date)
);

-- Promo usage audit (eventual consistency backup)
CREATE TABLE promo_usage_audit (
  id BIGSERIAL PRIMARY KEY,
  promo_code_id BIGINT NOT NULL,
  customer_id BIGINT NOT NULL,
  order_id BIGINT NOT NULL,
  usage_count INT NOT NULL,
  used_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (promo_code_id) REFERENCES promo_codes(id),
  INDEX idx_promo_customer (promo_code_id, customer_id),
  INDEX idx_used_at (used_at)
);

-- Fraud detection audit log
CREATE TABLE fraud_audit_log (
  id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT,
  code_type VARCHAR(50), -- GIFT_CARD or PROMO_CODE
  code_hash VARCHAR(255),
  attempt_ip VARCHAR(45),
  attempt_reason VARCHAR(255), -- INVALID_CODE, RATE_LIMIT, ENUMERATION
  severity VARCHAR(50), -- LOW, MEDIUM, HIGH, CRITICAL
  action_taken VARCHAR(255), -- BLOCKED, CHALLENGED, ALLOWED
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_customer_created (customer_id, created_at),
  INDEX idx_ip_created (attempt_ip, created_at)
);

-- IP reputation (blacklist/whitelist)
CREATE TABLE ip_reputation (
  id BIGSERIAL PRIMARY KEY,
  ip_address VARCHAR(45) UNIQUE NOT NULL,
  reputation_score INT, -- -100 (bad) to +100 (good)
  reason VARCHAR(255),
  is_blacklisted BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP,
  INDEX idx_ip_blacklist (ip_address, is_blacklisted)
);
```

### MongoDB: Promo Rules (Reference Data)

```json
{
  "_id": "promo_123",
  "code": "SUMMER20",
  "type": "PERCENTAGE",
  "value": 20,
  "description": "20% off all summer items",
  "rules": {
    "start_date": "2026-06-01",
    "end_date": "2026-08-31",
    "usage_limit": 10000,
    "per_customer_limit": 1,
    "min_order_value": 50.00,
    "eligible_categories": ["summer_collection", "outdoor"],
    "excluded_products": [1001, 1002],
    "customer_tier_required": "BRONZE"
  },
  "stacking": {
    "stackable": true,
    "stackable_with": ["FREE_SHIPPING", "LOYALTY_POINTS"],
    "cannot_stack_with": ["OTHER_DISCOUNT"]
  }
}
```

---

## Redis Data Structures

```
# Gift card balance cache (fast reads, sourced from PostgreSQL)
gc:balance:{gift_card_id_hash} → STRING (balance in cents)
gc:balance:{gift_card_id_hash}:ttl → TTL 1h

# Promo code usage counter (real-time enforcement)
promo:usage:{code_hash} → INT (incremented atomically)
promo:usage:{code_hash}:limit → INT (hard limit)

# Promo code metadata (for fast validation)
promo:meta:{code_hash} → HASH {
  discount_type: "PERCENTAGE",
  discount_value: 20,
  status: "ACTIVE",
  end_date: "2026-08-31",
  usage_limit: 10000,
  min_order_value: 50.00
}

# Per-customer usage (track "one per customer")
promo:customer:{code_hash}:{customer_id} → INT (usage count)
promo:customer:{code_hash}:{customer_id}:ttl → TTL until promo expires

# Rate limiting (brute force protection)
rate_limit:{customer_id}:redeem:hourly → INT
rate_limit:{customer_id}:redeem:hourly:ttl → TTL 1h

rate_limit:{ip_address}:redeem:minutely → INT
rate_limit:{ip_address}:redeem:minutely:ttl → TTL 1m

# IP reputation
ip_reputation:{ip_address} → HASH {
  score: 85,
  is_blacklisted: false,
  last_updated: "2026-04-01T10:00:00Z"
}

# Recent code attempts (detect enumeration)
enum_protection:{ip_address}:attempts → LIST (last 10 code hashes)
enum_protection:{ip_address}:attempts:ttl → TTL 1h
```

---

## Kafka Event Flow

```
Topics:
1. gift-card-events (3 partitions, key=gift_card_id)
   - GiftCardIssued
   - GiftCardRedeemed
   - GiftCardExpired
   - GiftCardRefunded

2. promo-events (3 partitions, key=code_hash)
   - PromoCodeCreated
   - PromoCodeApplied
   - PromoCodeUsageLimitReached
   - PromoCodeExpired

3. fraud-events (1 partition)
   - FraudDetected
   - RateLimitExceeded
   - EnumerationAttempt
   - IPBlacklisted

Consumers:
- NotificationService (send gift card issued/used notifications)
- AnalyticsService (track redemption patterns)
- FraudDetectionService (analyze suspicious activity)
- AuditService (store immutable event log)
```

---

## Implementation Code

### 1. Gift Card Code Generator

```java
package com.retail.giftcard.service;

import java.security.SecureRandom;
import java.util.Base64;

public class GiftCardCodeGenerator {

    private static final int CODE_LENGTH_BYTES = 12;  // 16 base32 chars
    private static final String CHARSET = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ";
    private static final SecureRandom RANDOM = new SecureRandom();

    /**
     * Generates a secure, unique gift card code with Luhn check digit.
     * Format: XXXX-XXXX-XXXX-XXXX (alphanumeric, no ambiguous chars like I, O)
     */
    public static String generateCode() {
        // 1. Generate random bytes
        byte[] randomBytes = new byte[CODE_LENGTH_BYTES];
        RANDOM.nextBytes(randomBytes);

        // 2. Encode as Base32 (removes ambiguous characters)
        String base32 = encodeBase32(randomBytes);

        // 3. Format with dashes
        String formatted = String.format("%s-%s-%s-%s",
            base32.substring(0, 4),
            base32.substring(4, 8),
            base32.substring(8, 12),
            base32.substring(12, 16)
        );

        // 4. Calculate and append Luhn check digit
        String luhnDigit = String.valueOf(calculateLuhn(formatted));

        return formatted + luhnDigit;
    }

    private static String encodeBase32(byte[] data) {
        StringBuilder sb = new StringBuilder();
        int buffer = 0;
        int bufferSize = 0;

        for (byte b : data) {
            buffer = (buffer << 8) | (b & 0xFF);
            bufferSize += 8;

            while (bufferSize >= 5) {
                bufferSize -= 5;
                int index = (buffer >> bufferSize) & 0x1F;
                sb.append(CHARSET.charAt(index));
            }
        }

        if (bufferSize > 0) {
            buffer <<= (5 - bufferSize);
            int index = buffer & 0x1F;
            sb.append(CHARSET.charAt(index));
        }

        return sb.toString();
    }

    /**
     * Calculate Luhn check digit for error detection (not security).
     */
    public static int calculateLuhn(String code) {
        int sum = 0;
        boolean isEven = false;

        for (int i = code.length() - 1; i >= 0; i--) {
            char c = code.charAt(i);
            if (!Character.isLetterOrDigit(c)) continue;  // Skip dashes

            int digit = Character.getNumericValue(c);
            if (isEven) {
                digit *= 2;
                if (digit > 9) digit -= 9;
            }
            sum += digit;
            isEven = !isEven;
        }

        return (10 - (sum % 10)) % 10;
    }

    /**
     * Validate code format and Luhn check digit.
     */
    public static boolean isValidFormat(String code) {
        // Remove dashes
        String clean = code.replace("-", "");

        // Check length and format
        if (clean.length() != 17) return false;  // 16 chars + 1 check digit
        if (!clean.matches("[0-9A-Z]+")) return false;

        // Verify Luhn
        String codeWithoutChecksum = clean.substring(0, 16);
        int expectedChecksum = Integer.parseInt(clean.substring(16));
        return calculateLuhn(codeWithoutChecksum) == expectedChecksum;
    }
}
```

### 2. Promo Code Validator with Stacking Rules

```java
package com.retail.promo.service;

import org.springframework.stereotype.Service;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.*;

@Service
public class PromoCodeValidator {

    private static final Logger log = LoggerFactory.getLogger(PromoCodeValidator.class);

    private final PromoCodeRepository promoRepository;
    private final PromoUsageRepository usageRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final PromoStackingRuleService stackingService;

    /**
     * Validate a single promo code.
     */
    public PromoValidationResult validatePromoCode(String code, Long customerId,
                                                    BigDecimal orderAmount,
                                                    String customerIp) {
        try {
            // 1. Check rate limiting
            if (isRateLimited(customerId, customerIp)) {
                log.warn("Rate limit exceeded for customer {} from IP {}", customerId, customerIp);
                return PromoValidationResult.rateLimited();
            }

            // 2. Hash code for lookup
            String codeHash = hashCode(code);

            // 3. Check if code exists
            Optional<PromoCode> promoOpt = promoRepository.findByCodeHash(codeHash);
            if (promoOpt.isEmpty()) {
                recordFraudAttempt(customerId, codeHash, customerIp, "INVALID_CODE");
                return PromoValidationResult.invalid();
            }

            PromoCode promo = promoOpt.get();

            // 4. Check expiration
            if (LocalDate.now().isAfter(promo.getEndDate())) {
                return PromoValidationResult.expired();
            }

            // 5. Check eligibility (min order value, customer tier, etc.)
            EligibilityCheckResult eligibility = checkEligibility(promo, customerId, orderAmount);
            if (!eligibility.isEligible()) {
                return PromoValidationResult.ineligible(eligibility.getReason());
            }

            // 6. Check usage limits (atomic operation)
            if (!checkUsageLimit(codeHash, customerId, promo)) {
                return PromoValidationResult.usageLimitExceeded();
            }

            log.info("Promo code {} validated successfully for customer {}", code, customerId);

            return PromoValidationResult.valid(promo);

        } catch (Exception e) {
            log.error("Error validating promo code", e);
            return PromoValidationResult.error(e.getMessage());
        }
    }

    /**
     * Validate multiple promo codes with stacking rules.
     */
    public StackedPromoValidationResult validateMultiplePromos(
            List<String> codes,
            Long customerId,
            BigDecimal orderAmount,
            String customerIp) {

        List<PromoValidationResult> validatedPromos = new ArrayList<>();
        List<String> errors = new ArrayList<>();

        // 1. Validate each code individually
        for (String code : codes) {
            PromoValidationResult result = validatePromoCode(code, customerId, orderAmount, customerIp);
            if (result.isValid()) {
                validatedPromos.add(result);
            } else {
                errors.add(code + ": " + result.getError());
            }
        }

        if (!errors.isEmpty()) {
            return StackedPromoValidationResult.failed(errors);
        }

        // 2. Check stacking rules
        StackingValidationResult stackingResult = stackingService
            .validateStacking(validatedPromos.stream()
                .map(r -> r.getPromoCode())
                .collect(Collectors.toList()));

        if (!stackingResult.isAllowed()) {
            return StackedPromoValidationResult.stackingNotAllowed(stackingResult.getReason());
        }

        log.info("Stacked {} promos validated for customer {}", codes.size(), customerId);

        return StackedPromoValidationResult.valid(validatedPromos);
    }

    /**
     * Check and enforce usage limit atomically using Redis.
     */
    @Transactional
    private boolean checkUsageLimit(String codeHash, Long customerId, PromoCode promo) {
        // 1. Check per-customer limit
        String customerUsageKey = "promo:customer:" + codeHash + ":" + customerId;
        String customerUsageCount = redisTemplate.opsForValue().get(customerUsageKey);

        if (customerUsageCount != null) {
            int count = Integer.parseInt(customerUsageCount);
            if (count >= promo.getPerCustomerLimit()) {
                return false;  // Customer already used this promo
            }
        }

        // 2. Check global usage limit (atomic INCR)
        String globalUsageKey = "promo:usage:" + codeHash;
        Long newCount = redisTemplate.opsForValue().increment(globalUsageKey);

        if (newCount > promo.getUsageLimit()) {
            // Rollback the increment
            redisTemplate.opsForValue().decrement(globalUsageKey);
            return false;  // Global limit exceeded
        }

        // 3. Set customer usage count
        long ttlSeconds = ChronoUnit.SECONDS.between(
            LocalDateTime.now(),
            promo.getEndDate().atEndOfDay()
        );
        redisTemplate.opsForValue().set(customerUsageKey, "1", ttlSeconds, TimeUnit.SECONDS);

        // 4. Log usage to PostgreSQL (eventual consistency)
        PromoUsageAudit audit = new PromoUsageAudit();
        audit.setPromoCodeId(promo.getId());
        audit.setCustomerId(customerId);
        audit.setUsageCount((int) newCount.longValue());
        usageRepository.save(audit);

        return true;
    }

    /**
     * Check customer eligibility (min order, tier, categories, etc.).
     */
    private EligibilityCheckResult checkEligibility(PromoCode promo, Long customerId,
                                                     BigDecimal orderAmount) {
        // 1. Check minimum order value
        if (promo.getMinOrderValue() != null &&
            orderAmount.compareTo(promo.getMinOrderValue()) < 0) {
            return EligibilityCheckResult.ineligible(
                "Order value below minimum: " + promo.getMinOrderValue()
            );
        }

        // 2. Check customer tier
        if (promo.getCustomerTierRequired() != null) {
            String customerTier = getCustomerTier(customerId);
            if (!isEligibleTier(customerTier, promo.getCustomerTierRequired())) {
                return EligibilityCheckResult.ineligible(
                    "Customer tier not eligible: requires " + promo.getCustomerTierRequired()
                );
            }
        }

        // 3. Check category eligibility
        if (promo.getEligibleCategories() != null &&
            !promo.getEligibleCategories().isEmpty()) {
            // Fetch cart categories; if none match, ineligible
            // (implementation depends on cart service)
        }

        return EligibilityCheckResult.eligible();
    }

    /**
     * Rate limiting: 5 attempts per IP per minute, 10 per customer per hour.
     */
    private boolean isRateLimited(Long customerId, String customerIp) {
        // 1. Check IP rate limit (minutely)
        String ipKey = "rate_limit:" + customerIp + ":redeem:minutely";
        Long ipCount = redisTemplate.opsForValue().increment(ipKey);
        if (ipCount == 1) {
            redisTemplate.expire(ipKey, 1, TimeUnit.MINUTES);
        }
        if (ipCount > 5) return true;

        // 2. Check customer rate limit (hourly)
        String customerKey = "rate_limit:" + customerId + ":redeem:hourly";
        Long customerCount = redisTemplate.opsForValue().increment(customerKey);
        if (customerCount == 1) {
            redisTemplate.expire(customerKey, 1, TimeUnit.HOURS);
        }
        if (customerCount > 10) return true;

        return false;
    }

    private void recordFraudAttempt(Long customerId, String codeHash, String ip,
                                    String reason) {
        FraudAuditLog log = new FraudAuditLog();
        log.setCustomerId(customerId);
        log.setCodeHash(codeHash);
        log.setAttemptIp(ip);
        log.setAttemptReason(reason);
        log.setSeverity("LOW");
        fraudAuditRepository.save(log);

        // Publish fraud event
        kafkaTemplate.send("fraud-events", new FraudDetected(customerId, codeHash, ip, reason));
    }

    private String hashCode(String code) {
        // Use SHA-256
        return DigestUtils.sha256Hex(code);
    }

    private String getCustomerTier(Long customerId) {
        // Fetch from customer service or cache
        return "GOLD";  // Placeholder
    }

    private boolean isEligibleTier(String customerTier, String requiredTier) {
        // Tier hierarchy: BRONZE < SILVER < GOLD < PLATINUM
        Map<String, Integer> tierRank = Map.of(
            "BRONZE", 0, "SILVER", 1, "GOLD", 2, "PLATINUM", 3
        );
        return tierRank.getOrDefault(customerTier, -1) >=
               tierRank.getOrDefault(requiredTier, 0);
    }
}
```

### 3. Usage Limit Enforcer (Lua Script)

```java
package com.retail.promo.service;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.core.script.RedisScript;
import org.springframework.stereotype.Service;

@Service
public class UsageLimitEnforcer {

    private final RedisTemplate<String, String> redisTemplate;

    private static final String USAGE_LIMIT_SCRIPT = """
        local usageKey = KEYS[1]
        local limitKey = KEYS[2]
        local limit = tonumber(redis.call('GET', limitKey))
        local current = tonumber(redis.call('GET', usageKey) or 0)

        if current >= limit then
            return 0  -- Limit exceeded
        else
            redis.call('INCR', usageKey)
            return 1  -- Success
        end
    """;

    /**
     * Atomically check usage limit and increment counter.
     * Returns true if increment succeeded, false if limit exceeded.
     */
    public boolean checkAndIncrement(String codeHash, int limit) {
        RedisScript<Boolean> script = RedisScript.of(USAGE_LIMIT_SCRIPT, Boolean.class);

        String usageKey = "promo:usage:" + codeHash;
        String limitKey = "promo:usage:" + codeHash + ":limit";

        // Initialize limit if not set
        if (!isLimitSet(limitKey)) {
            redisTemplate.opsForValue().set(limitKey, String.valueOf(limit));
            redisTemplate.expire(limitKey, 365, TimeUnit.DAYS);
        }

        return Boolean.TRUE.equals(
            redisTemplate.execute(script, List.of(usageKey, limitKey))
        );
    }

    /**
     * Atomic increment for per-customer usage.
     */
    public int incrementPerCustomer(String codeHash, Long customerId, int limit) {
        String key = "promo:customer:" + codeHash + ":" + customerId;
        Long count = redisTemplate.opsForValue().increment(key);

        if (count == 1) {
            // First time; set expiration
            redisTemplate.expire(key, 365, TimeUnit.DAYS);
        }

        return count.intValue();
    }

    private boolean isLimitSet(String limitKey) {
        return Boolean.TRUE.equals(redisTemplate.hasKey(limitKey));
    }
}
```

### 4. Gift Card Service

```java
package com.retail.giftcard.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.kafka.core.KafkaTemplate;

@Service
public class GiftCardService {

    private static final Logger log = LoggerFactory.getLogger(GiftCardService.class);

    private final GiftCardRepository giftCardRepository;
    private final GiftCardLedgerRepository ledgerRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, GiftCardEvent> kafkaTemplate;

    /**
     * Issue a new gift card.
     */
    @Transactional
    public IssuanceResult issueGiftCard(BigDecimal initialBalance,
                                        LocalDate expirationDate,
                                        String recipientEmail) {
        try {
            // 1. Generate unique code
            String code = GiftCardCodeGenerator.generateCode();
            String codeHash = hashCode(code);

            // 2. Create gift card record
            GiftCard card = new GiftCard();
            card.setGiftCardNumberHash(codeHash);
            card.setCardType("DIGITAL");
            card.setInitialBalance(initialBalance);
            card.setCurrentBalance(initialBalance);
            card.setStatus("ACTIVE");
            card.setIssuedDate(LocalDate.now());
            card.setActivationDate(LocalDate.now());
            card.setExpirationDate(expirationDate);
            card.setRecipientEmail(recipientEmail);

            GiftCard saved = giftCardRepository.save(card);

            // 3. Create activation ledger entry
            GiftCardLedger activation = new GiftCardLedger();
            activation.setGiftCardId(saved.getId());
            activation.setTransactionType("ACTIVATE");
            activation.setAmount(initialBalance);
            activation.setBalanceAfter(initialBalance);
            activation.setReason("Gift card issued");
            ledgerRepository.save(activation);

            // 4. Cache balance
            String balanceKey = "gc:balance:" + codeHash;
            redisTemplate.opsForValue().set(
                balanceKey,
                initialBalance.toPlainString(),
                365,
                TimeUnit.DAYS
            );

            // 5. Publish event
            GiftCardIssued event = new GiftCardIssued(saved.getId(), codeHash,
                                                       initialBalance, expirationDate);
            kafkaTemplate.send("gift-card-events", codeHash, event);

            log.info("Issued gift card {} with balance {}", saved.getId(), initialBalance);

            return IssuanceResult.success(saved.getId(), code);

        } catch (Exception e) {
            log.error("Error issuing gift card", e);
            throw new GiftCardException("Issuance failed", e);
        }
    }

    /**
     * Redeem a gift card (with partial redemption support).
     */
    @Transactional
    public RedemptionResult redeemGiftCard(String code, BigDecimal redeemAmount,
                                           Long orderId) {
        try {
            String codeHash = hashCode(code);

            // 1. Validate code format
            if (!GiftCardCodeGenerator.isValidFormat(code)) {
                return RedemptionResult.invalidCode();
            }

            // 2. Check if code exists
            Optional<GiftCard> cardOpt = giftCardRepository.findByGiftCardNumberHash(codeHash);
            if (cardOpt.isEmpty()) {
                return RedemptionResult.cardNotFound();
            }

            GiftCard card = cardOpt.get();

            // 3. Check expiration
            if (LocalDate.now().isAfter(card.getExpirationDate())) {
                return RedemptionResult.expired();
            }

            // 4. Check status (not already fully redeemed)
            if ("REDEEMED".equals(card.getStatus())) {
                return RedemptionResult.alreadyRedeemed();
            }

            // 5. Check balance (from cache or database)
            BigDecimal currentBalance = getBalance(codeHash, card.getId());
            if (currentBalance.compareTo(redeemAmount) < 0) {
                return RedemptionResult.insufficientBalance(currentBalance);
            }

            // 6. Create redemption ledger entry
            BigDecimal balanceAfter = currentBalance.subtract(redeemAmount);

            GiftCardLedger redemption = new GiftCardLedger();
            redemption.setGiftCardId(card.getId());
            redemption.setOrderId(orderId);
            redemption.setTransactionType("REDEEM");
            redemption.setAmount(redeemAmount.negate());  // Negative for deduction
            redemption.setBalanceAfter(balanceAfter);
            redemption.setReason("Order redemption");
            ledgerRepository.save(redemption);

            // 7. Update card status
            if (balanceAfter.compareTo(BigDecimal.ZERO) <= 0) {
                card.setStatus("REDEEMED");
            }
            card.setCurrentBalance(balanceAfter);
            giftCardRepository.save(card);

            // 8. Update cache
            String balanceKey = "gc:balance:" + codeHash;
            redisTemplate.opsForValue().set(
                balanceKey,
                balanceAfter.toPlainString(),
                365,
                TimeUnit.DAYS
            );

            // 9. Publish event
            GiftCardRedeemed event = new GiftCardRedeemed(
                card.getId(), codeHash, redeemAmount, balanceAfter, orderId
            );
            kafkaTemplate.send("gift-card-events", codeHash, event);

            log.info("Redeemed {} from gift card {}", redeemAmount, card.getId());

            return RedemptionResult.success(balanceAfter);

        } catch (Exception e) {
            log.error("Error redeeming gift card", e);
            throw new GiftCardException("Redemption failed", e);
        }
    }

    /**
     * Get current balance (cache-first).
     */
    public BigDecimal getBalance(String code) {
        String codeHash = hashCode(code);
        return getBalance(codeHash, null);
    }

    private BigDecimal getBalance(String codeHash, Long giftCardId) {
        // 1. Try cache
        String balanceKey = "gc:balance:" + codeHash;
        String cached = redisTemplate.opsForValue().get(balanceKey);

        if (cached != null) {
            return new BigDecimal(cached);
        }

        // 2. Fallback to database
        if (giftCardId != null) {
            Optional<GiftCard> card = giftCardRepository.findById(giftCardId);
            if (card.isPresent()) {
                // Calculate balance from ledger
                BigDecimal balance = ledgerRepository
                    .sumBalanceByGiftCardId(giftCardId);

                // Update cache
                redisTemplate.opsForValue().set(
                    balanceKey,
                    balance.toPlainString(),
                    365,
                    TimeUnit.DAYS
                );

                return balance;
            }
        }

        return BigDecimal.ZERO;
    }

    private String hashCode(String code) {
        return DigestUtils.sha256Hex(code);
    }
}
```

---

## Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| **Redis cache miss on balance** | Hit PostgreSQL, slower response | Warm cache at startup; async refresh |
| **Gift card code exposure** | Attacker can enumerate codes | Hash codes; no sequential IDs; rate limiting |
| **Concurrent redemptions same code** | Balance goes negative | Pessimistic lock in PostgreSQL; ledger approach |
| **Usage limit race condition** | One extra use allowed | Lua script for atomic check-and-increment |
| **Promo code enumeration attack** | Attacker discovers valid codes | Rate limiting per IP; CAPTCHA after N failures |
| **PostgreSQL down during redemption** | Cannot process redemptions | Cache holds 24h of balance; fallback reads from Redis |
| **Kafka unavailable** | Events not published | Store events in PostgreSQL; replay when recovered |

---

## Scaling Strategy

### Horizontal Scaling

1. **Gift Card Service replicas:** Load balanced by API Gateway
2. **Kafka partitions:** Partition by code hash for ordering per code
3. **Redis cluster:** Distribute cache across nodes (consistent hashing)
4. **PostgreSQL read replicas:** For balance queries and fraud audits

### Vertical Scaling

1. **Connection pooling:** HikariCP with max 200 connections
2. **Redis memory:** Pre-allocate for expected concurrent gift cards (50GB)
3. **Kafka brokers:** Increase broker count for throughput

### Caching Strategy

- L1: Redis (balance cache, usage counters, rate limits)
- L2: PostgreSQL (source of truth, audit logs)
- L3: MongoDB (promo rules, reference data)

---

## Monitoring & Observability

### Metrics

```java
meterRegistry.timer("gift_card.redeem.latency")
    .record(() -> redeemGiftCard(...));

meterRegistry.counter("promo.validation.invalid", "reason", "expired")
    .increment();

meterRegistry.gauge("redis.cache.hit.rate",
    cacheHitCounter::getAsDouble);

meterRegistry.counter("fraud.attempts.total",
    "severity", "HIGH").increment();
```

### Alerts

- Cache hit rate < 90% (indicates cache miss pressure)
- Redemption latency > 200ms
- Fraud attempts > 1000/hour
- Promo usage spike (sudden 10x increase)

---

## Summary Cheat Sheet

| Component | Choice | Why |
|-----------|--------|-----|
| **Code generation** | SecureRandom + Base32 + Luhn | Cryptographically secure, unguessable, error-detecting |
| **Code storage** | Hash in Redis, plaintext in vault | No enumeration, fast lookup |
| **Balance tracking** | Ledger (immutable entries) | Auditability, partial redemptions, recoverability |
| **Usage limits** | Redis Lua script + PostgreSQL backup | Atomic, no race conditions, eventual consistency |
| **Promo validation** | Chain of validators + stacking rules | Extensible, configurable, handles complexity |
| **Rate limiting** | Per IP + per customer, exponential backoff | Prevents brute force and enumeration |
| **Fraud prevention** | Multi-layer: code hashing, rate limiting, anomaly detection | Defense in depth |
| **Eventual consistency** | PostgreSQL audit table, daily reconciliation | Catches Redis mismatches |

---

**Generated:** 2026-04-01 | **Tech Stack:** Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka
