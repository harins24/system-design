---
title: Multi-Currency and FX Management — System Design Deep Dive
layout: default
---

# Multi-Currency and FX Management — Deep Dive Design

> **Scenario:** Support 50+ currencies, display prices in local currency, accept payments in local currency, settle with merchants in their local currency, daily FX rate updates, conversion fees (2.5%), hedging strategies, PSD2/SCA compliance.
>
> **Tech Stack:** Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka

## Table of Contents

1. [Requirements & Constraints](#requirements--constraints)
2. [Capacity Estimation](#capacity-estimation)
3. [High-Level Architecture](#high-level-architecture)
4. [Core Design Questions Answered](#core-design-questions-answered)
5. [Microservices Breakdown](#microservices-breakdown)
6. [Database Design (DDL)](#database-design-ddl)
7. [Redis Data Structures](#redis-data-structures)
8. [Kafka Event Flow](#kafka-event-flow)
9. [Implementation Code](#implementation-code)
10. [Failure Scenarios & Recovery](#failure-scenarios--recovery)
11. [Scaling Strategy](#scaling-strategy)
12. [Monitoring & Alerting](#monitoring--alerting)
13. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Requirements & Constraints

### Functional Requirements
- Support 50+ fiat currencies (USD, EUR, GBP, JPY, INR, CNY, CAD, AUD, etc.)
- Display product prices in customer's local currency in real-time
- Accept payments in any supported currency
- Settle with merchants in their preferred currency
- Daily FX rate updates from reliable sources (ECB, OpenExchangeRates)
- Apply conversion fees (configurable per currency pair, default 2.5%)
- Lock FX rate at cart creation; honor rate for 15 minutes
- Warn user if FX rate expires
- Multi-currency journal entries for accounting (all amounts tracked in both customer and settlement currency)
- PSD2/SCA compliance: Strong Customer Authentication for EU cards, exceptions for low-value (<€30) and trusted beneficiaries
- Reconciliation: daily settlement with merchants at locked rate
- Hedging: forward FX contracts to manage exposure for high-volume currency pairs

### Non-Functional Requirements
- FX rate lookup <50ms latency (p99)
- Price conversion accuracy: ±0.01 cents rounding
- Zero data loss in accounting ledger
- Audit trail for all FX conversions
- 99.9% uptime for FX service
- Support 50K merchants in 30+ countries

---

## Capacity Estimation

| Metric | Calculation | Value |
|--------|-------------|-------|
| **Daily Transactions** | Given | 10,000,000 |
| **Transactions/Second** | 10M ÷ 86,400 | ~116 txns/sec |
| **Currencies Active** | Given | 50 |
| **Currency Pair Rates** | 50 × 50 (all pairs) | 2,500 rates |
| **FX Rate Updates/Day** | 1 per day per rate | 2,500 updates/day |
| **Conversions/Second** | 116 txns × 2 conversions (cart + checkout) | ~232 conversions/sec |
| **Storage: FX Rates (annual)** | 2,500 rates × 365 days × 50 bytes | ~46 MB |
| **Storage: Journal Entries (annual)** | 10M txns/day × 365 × 400 bytes | ~1.5 TB |
| **Merchants/Countries** | Given | 50,000 merchants in 30 countries |
| **Settlement Batch/Day** | 1 per country, per currency | ~30 × 50 = 1,500 batches/day |
| **Redis Memory (rates cache)** | 2,500 rates × 100 bytes × 2 versions | ~500 KB |

---

## High-Level Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                  Customer Applications                          │
│     (Web, Mobile, Marketplace, Admin Dashboard)                 │
└────────────────┬──────────────────────────────────────────────┘
                 │
       ┌─────────┴──────────┬─────────────────┐
       │                    │                 │
   ┌───▼────────┐    ┌──────▼──────┐   ┌─────▼──────┐
   │ Product    │    │  Checkout   │   │ Settlement │
   │ Catalog    │    │  (Cart FX   │   │ Reports    │
   │ Service    │    │   lock)     │   │            │
   └───┬────────┘    └──────┬──────┘   └─────┬──────┘
       │                    │                │
       └────────────────┬───┴────────────────┘
                        │
        ┌───────────────▼────────────────┐
        │ Currency Conversion Service    │
        │  - FX rate lookup              │
        │  - Price conversion            │
        │  - Conversion fee calculation  │
        └───┬─────────────────┬──────────┘
            │                 │
    ┌───────▼─┐      ┌───────▼──────────┐
    │  Redis  │      │  PostgreSQL      │
    │ (Cache) │      │  FX Rates +      │
    │         │      │  Conversions     │
    └─────────┘      └────┬─────────────┘
                          │
        ┌─────────────────┴──────────────┐
        │  FX Rate Update Job            │
        │  (Daily: ECB/OpenExchangeRates)│
        │  Schedule: 02:00 UTC           │
        └────────┬───────────────────────┘
                 │
        ┌────────▼────────┐
        │  Kafka Topics   │
        │ - fx-rate-      │
        │   update        │
        │ - payment-      │
        │   made          │
        │ - settlement-   │
        │   batch         │
        └────────┬────────┘
                 │
        ┌────────▼──────────────┐
        │ Ledger Service        │
        │ (Multi-currency       │
        │  Journal Entries)     │
        └───────────────────────┘
```

---

## Core Design Questions Answered

### Q1: How do you manage exchange rates and updates?

**Answer:**
1. **Daily batch fetch:** Scheduled job runs at 02:00 UTC, fetches rates from ECB API
2. **Rate storage:** PostgreSQL table `fx_rates(rate_date, from_currency, to_currency, rate, source)`
3. **Redis cache:** Hash `fx:rates:{date}` keyed by `from_currency:to_currency` with rate value
4. **Stale check:** If live rate > 24h old, use previous day's rate + warning
5. **Fallback:** If ECB down, use OpenExchangeRates API as backup

**FX Rate Hierarchy:**
```
Live request
  ├─ Check Redis cache (expires 23h50m)
  ├─ If miss: Query PostgreSQL latest rate
  ├─ If >24h old: Use previous rate + log warning
  └─ If no historical: Reject transaction
```

---

### Q2: How do you handle currency conversion during checkout?

**Answer:**
1. **Cart creation:** Lock FX rate at moment of cart creation; store in cart table
2. **Price display:** Convert product base price (USD) to customer currency using locked rate
3. **Checkout:** Use same locked rate; if rate expires (>15 min), warn customer "Rate expired, refresh cart"
4. **Conversion fee:** Apply 2.5% fee on converted amount (charged to customer or absorbed by merchant)
5. **Rounding:** Use HALF_UP rounding to nearest 0.01 cents

**Example:**
```
Product base price: $100 USD
Customer currency: EUR
Cart created at 10:00 UTC
Locked rate: 1 USD = 0.92 EUR (fetched at 10:00)

Price in EUR: 100 × 0.92 = €92.00
Conversion fee: €92.00 × 0.025 = €2.30
Total: €94.30

At 10:20 UTC (checkout):
- Rate is still valid (20 min < 15 min expiry? NO, expired)
- Show warning: "Exchange rate expired, please refresh"
- Use new rate if customer agrees
```

---

### Q3: How do you display accurate prices in multiple currencies?

**Answer:**
1. **Real-time conversion API:** Endpoint `/api/convert?amount=100&from=USD&to=EUR` returns converted price with timestamp
2. **Client-side caching:** Cache converted price for 5 minutes per currency pair
3. **Batch conversion:** Bulk endpoint `/api/convert-batch` with JSON array of conversions for catalog pages
4. **Rate inclusion:** Return rate used in response (for UI to show "Rate: 0.92 EUR/USD")

**Database-backed rate lookup:**
```java
// Pseudo-code
rate = cacheOrFetchRate("USD", "EUR"); // <50ms
convertedPrice = basePrice * rate; // Multiplication
return round(convertedPrice, 2); // HALF_UP
```

---

### Q4: How do you reconcile multi-currency transactions in accounting?

**Answer:** Double-entry ledger with dual-currency recording:

1. **Every transaction creates 2 journal entries:**
   - Debit: Customer account (in customer currency)
   - Credit: Merchant settlement account (in merchant currency)
2. **Conversion entry:** Record conversion gain/loss if rates changed
3. **Reconciliation:** Daily batch process queries ledger for each currency; verify:
   - sum(debit_USD) = expected received amount
   - sum(credit_EUR) = expected settled amount with merchant
4. **FX revaluation:** Month-end: revalue open positions at current rate; record gain/loss

**Journal Entry Example:**
```
Transaction: Customer pays €94.30 (locked rate 0.92 USD/EUR)
Merchant settles in USD

Debit:  Customer Account (EUR)        €94.30
  Credit: Revenue Account (EUR)               €92.00
  Credit: Conversion Fee Account (EUR)        €2.30

Debit:  Merchant Settlement (USD)    $100.00
  Credit: Currency Gain/Loss                    $0.72 (FX profit)
```

---

### Q5: How do you handle FX rate changes between cart and checkout?

**Answer:**
- **Lock rate at cart creation:** Immutable; never change unless customer explicitly refreshes cart
- **Rate validity window:** 15 minutes (configurable)
- **Expiry warning:** At 12 minutes, show banner "Rate expires in 3 minutes"
- **Expired rate flow:**
  1. Customer sees "Cart rate expired"
  2. Customer clicks "Refresh rate"
  3. New rate fetched and locked
  4. Price recalculated; customer confirms updated total
  5. Checkout proceeds with new rate
- **No silent updates:** Never auto-refresh; respect customer's price expectation

---

### Q6: How do you comply with regulations like PSD2?

**Answer:** Strong Customer Authentication (SCA) based on risk and geography:

1. **SCA Required:**
   - EU/EEA cards (customer initiated from EU)
   - Amount > €500 (high value)
   - New device or unusual IP
2. **SCA Exemptions:**
   - Low-value (<€30): Exempt from SCA
   - Trusted beneficiaries: Customer has whitelisted merchant
   - Frictionless: 3DS frictionless protocol (device data; issuer approves without interaction)
3. **Strong Authentication Methods:**
   - SMS OTP
   - FIDO2/WebAuthn
   - Push notification from bank app
4. **Liability Shift:** If SCA completed, liability for fraud shifts to issuer

**Implementation:**
```java
if (isEUCard(card) && (amount > 5000 || isNewDevice(fingerprint))) {
  required3DS = true;
  scaExemption = false;
} else if (amount < 3000 && !isNewDevice(fingerprint)) {
  scaExemption = EXEMPTION_LOW_VALUE;
  required3DS = false;
} else if (merchant.isTrustedBeneficiary(customer)) {
  scaExemption = EXEMPTION_TRUSTED_BENEFICIARY;
  required3DS = false;
}
```

---

## Microservices Breakdown

| Service | Responsibility | Tech | Throughput |
|---------|-----------------|------|-----------|
| **FX Rate Service** | Fetch rates, cache, serve conversions | Spring Boot + Redis | 50K lookups/sec |
| **Conversion Service** | Convert prices, apply fees, rounding | Spring Boot | 232 conversions/sec |
| **Checkout Service** | Lock FX rate at cart, validate at checkout | Spring Boot + Transactional | 116 txns/sec |
| **Settlement Service** | Batch settle merchants, dual-currency ledger | Spring Batch + Kafka | 1,500 batches/day |
| **Ledger Service** | Journal entries, FX revaluation, reconciliation | Spring Boot + PostgreSQL | 232 entries/sec |
| **SCA Service** | Evaluate PSD2 risk, trigger 3DS challenges | Spring Boot + 3DS Gateway | 50 challenges/sec |
| **Rate Update Job** | Daily ECB fetch, populate DB + Redis | Spring Batch | 1 run/day |

---

## Database Design (DDL)

```sql
-- PostgreSQL Schema

CREATE TABLE fx_rates (
    id BIGSERIAL PRIMARY KEY,
    rate_date DATE NOT NULL,
    from_currency VARCHAR(3) NOT NULL, -- USD, EUR, GBP
    to_currency VARCHAR(3) NOT NULL,
    rate DECIMAL(18, 8) NOT NULL, -- E.g., 0.92345678 for USD->EUR
    source VARCHAR(50) NOT NULL, -- ECB, OpenExchangeRates, etc.
    fetched_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(rate_date, from_currency, to_currency, source)
);
CREATE INDEX idx_fx_rates_date_pair ON fx_rates(rate_date, from_currency, to_currency);

CREATE TABLE conversion_fees (
    id BIGSERIAL PRIMARY KEY,
    from_currency VARCHAR(3) NOT NULL,
    to_currency VARCHAR(3) NOT NULL,
    fee_percentage DECIMAL(5, 3) NOT NULL DEFAULT 2.5, -- 2.5% default
    effective_date DATE NOT NULL,
    merchant_id BIGINT, -- NULL = global default
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(from_currency, to_currency, effective_date, merchant_id)
);

CREATE TABLE carts (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    product_ids TEXT NOT NULL, -- JSON array of product IDs
    base_currency VARCHAR(3) DEFAULT 'USD',
    display_currency VARCHAR(3) NOT NULL,
    locked_rate DECIMAL(18, 8) NOT NULL,
    rate_locked_at TIMESTAMP NOT NULL,
    rate_expires_at TIMESTAMP NOT NULL,
    subtotal_base_cents BIGINT NOT NULL,
    subtotal_display_cents BIGINT NOT NULL,
    conversion_fee_cents BIGINT NOT NULL,
    total_display_cents BIGINT NOT NULL,
    status VARCHAR(50) DEFAULT 'ACTIVE', -- ACTIVE, CONVERTED, ABANDONED, CHECKED_OUT
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_carts_customer_status ON carts(customer_id, status);
CREATE INDEX idx_carts_rate_expires_at ON carts(rate_expires_at);

CREATE TABLE ledger_entries (
    id BIGSERIAL PRIMARY KEY,
    transaction_id VARCHAR(100) NOT NULL,
    journal_entry_id BIGINT, -- Group ID for debit/credit pair
    account VARCHAR(100) NOT NULL, -- customer_account, merchant_account, revenue, fx_gain_loss
    currency VARCHAR(3) NOT NULL,
    amount_cents BIGINT NOT NULL,
    entry_type VARCHAR(20) NOT NULL, -- DEBIT, CREDIT
    description VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_ledger_transaction ON ledger_entries(transaction_id);
CREATE INDEX idx_ledger_account ON ledger_entries(account, created_at);

CREATE TABLE settlements (
    id BIGSERIAL PRIMARY KEY,
    merchant_id BIGINT NOT NULL,
    settlement_date DATE NOT NULL,
    settlement_currency VARCHAR(3) NOT NULL,
    transactions_count INT NOT NULL,
    total_amount_cents BIGINT NOT NULL, -- In settlement currency
    fx_gain_loss_cents BIGINT, -- Gain/loss on FX conversion
    conversion_fee_cents BIGINT,
    net_amount_cents BIGINT NOT NULL,
    status VARCHAR(50) DEFAULT 'PENDING', -- PENDING, PROCESSED, PAID, RECONCILED
    locked_rate DECIMAL(18, 8), -- Rate used for settlement
    actual_rate DECIMAL(18, 8), -- Rate at settlement time (for revaluation)
    processed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_settlements_merchant_date ON settlements(merchant_id, settlement_date);
CREATE INDEX idx_settlements_status ON settlements(status);

CREATE TABLE sca_exemptions (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    merchant_id BIGINT NOT NULL,
    exemption_type VARCHAR(50) NOT NULL, -- TRUSTED_BENEFICIARY, LOW_VALUE, RECURRING
    enabled BOOLEAN DEFAULT TRUE,
    whitelisted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP, -- NULL = permanent
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(customer_id, merchant_id, exemption_type)
);
```

---

## Redis Data Structures

```python
# FX Rates Cache (Hash: all rates for a date)
fx:rates:{date} → HASH
  Fields: {from_currency}:{to_currency}, Value: {rate}
  Example: "USD:EUR" → "0.92345678"
  TTL: 23 hours 50 minutes (refreshed daily)

# Conversion Fees Cache (Hash: lookup fee by currency pair)
fx:fees:{date} → HASH
  Fields: {from_currency}:{to_currency}, Value: {fee_percentage}
  Example: "USD:EUR" → "0.025"
  TTL: 24 hours

# Cart Rate Lock (Key-Value: locked rate for cart)
cart:rate:{cart_id} → JSON
  { "from_currency": "USD", "to_currency": "EUR", "rate": 0.92345678, "locked_at": 1704067200000 }
  TTL: 20 minutes (5 min grace after expiry notification)

# Rate Update Timestamp (Key-Value: metadata)
fx:last_update → HASH
  Fields: "timestamp", "source", "count"
  Example: { "timestamp": "1704067200000", "source": "ECB", "count": "2500" }
  TTL: None (persistent)

# High-Volume Currency Pair Locks (For hedging)
fx:hedge:position:{currency_pair} → HASH
  Fields: "amount_cents", "rate", "hedge_date", "expires_at"
  Used for forward contracts on high-volume pairs
```

---

## Kafka Event Flow

```
┌────────────────────────────────────────────────────────────────┐
│ Kafka Topics & Event Flow                                      │
└────────────────────────────────────────────────────────────────┘

1. fx-rate-update (daily)
   └─ Producer: FxRateUpdateJob (scheduled)
   └─ Payload: { from_currency, to_currency, new_rate, old_rate, timestamp }
   └─ Consumers: PriceAggregatorService (update catalog prices), AlertService
   └─ Partitioning: by currency_pair

2. payment-made (every transaction)
   └─ Producer: PaymentService
   └─ Payload: { transaction_id, customer_currency, merchant_currency, amount_customer_cents, amount_merchant_cents, locked_rate, actual_rate }
   └─ Consumers: LedgerService, SettlementService, AnalyticsEngine
   └─ Partitioning: by merchant_id

3. settlement-batch (daily per country)
   └─ Producer: SettlementService
   └─ Payload: { batch_id, merchant_id, settlement_date, settlement_currency, total_amount_cents, fx_gain_loss_cents }
   └─ Consumers: LedgerService, NotificationService, BankingIntegration
   └─ Partitioning: by merchant_id

4. sca-challenge (PSD2 compliance)
   └─ Producer: PaymentService (risk assessment)
   └─ Payload: { transaction_id, customer_id, merchant_id, amount_cents, reason }
   └─ Consumers: ThreeDSecureService, NotificationService
   └─ Partitioning: by customer_id

5. fx-revaluation (month-end)
   └─ Producer: RevaluationJob (scheduled)
   └─ Payload: { month, currency, open_position_cents, revaluation_rate, gain_loss_cents }
   └─ Consumers: LedgerService, ReportingService
   └─ Partitioning: by currency

Event Example (payment-made):
{
  "transaction_id": "txn-20240101-00001",
  "customer_id": 12345,
  "merchant_id": 67890,
  "customer_currency": "EUR",
  "merchant_currency": "USD",
  "amount_customer_cents": 9430, // €94.30
  "amount_merchant_cents": 10000, // $100.00
  "locked_rate": 0.92345678,
  "actual_rate": 0.92345678,
  "conversion_fee_percent": 0.025,
  "timestamp": 1704067200000
}
```

---

## Implementation Code

### 1. FxRateService — FX Rate Management & Conversion

```java
package com.payment.fx.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDate;
import java.util.Optional;

@Service
public class FxRateService {

    @Autowired private FxRateRepository fxRateRepo;
    @Autowired private RedisTemplate<String, String> redisTemplate;
    private static final RoundingMode ROUNDING_MODE = RoundingMode.HALF_UP;

    /**
     * Get FX rate from cache or database
     * Latency: <50ms SLA
     */
    public BigDecimal getRate(String fromCurrency, String toCurrency) {
        if (fromCurrency.equals(toCurrency)) {
            return BigDecimal.ONE;
        }

        LocalDate today = LocalDate.now();
        String cacheKey = "fx:rates:" + today;
        String rateKey = fromCurrency + ":" + toCurrency;

        // Check Redis cache first
        String cachedRate = (String) redisTemplate.opsForHash().get(cacheKey, rateKey);
        if (cachedRate != null) {
            return new BigDecimal(cachedRate);
        }

        // Cache miss: query PostgreSQL
        Optional<FxRate> fxRate = fxRateRepo.findLatestRate(fromCurrency, toCurrency);

        if (fxRate.isEmpty()) {
            throw new IllegalArgumentException("No FX rate available for " + fromCurrency + "/" + toCurrency);
        }

        BigDecimal rate = fxRate.get().getRate();

        // Cache for 23h50m
        redisTemplate.opsForHash().put(cacheKey, rateKey, rate.toPlainString());
        redisTemplate.expire(cacheKey, 23 * 3600 + 50 * 60, java.util.concurrent.TimeUnit.SECONDS);

        return rate;
    }

    /**
     * Convert amount from source to target currency
     * Includes rounding to 2 decimal places
     */
    public long convertAmount(long amountCents, String fromCurrency, String toCurrency,
                             BigDecimal conversionFeePercent) {

        if (fromCurrency.equals(toCurrency)) {
            return amountCents;
        }

        BigDecimal rate = getRate(fromCurrency, toCurrency);
        BigDecimal amountDecimal = BigDecimal.valueOf(amountCents);

        // Convert: amount * rate
        BigDecimal convertedAmount = amountDecimal.multiply(rate)
            .setScale(2, ROUNDING_MODE);

        // Apply conversion fee
        BigDecimal feeMultiplier = BigDecimal.ONE.add(conversionFeePercent);
        BigDecimal finalAmount = convertedAmount.multiply(feeMultiplier)
            .setScale(2, ROUNDING_MODE);

        return finalAmount.longValue();
    }

    /**
     * Get conversion fee for a currency pair
     */
    public BigDecimal getConversionFee(String fromCurrency, String toCurrency, Long merchantId) {
        String cacheKey = "fx:fees:" + LocalDate.now();
        String feeKey = fromCurrency + ":" + toCurrency;

        // Check cache
        String cachedFee = (String) redisTemplate.opsForHash().get(cacheKey, feeKey);
        if (cachedFee != null) {
            return new BigDecimal(cachedFee);
        }

        // Query database (prioritize merchant-specific, fallback to default)
        Optional<ConversionFee> fee = merchantId != null
            ? feeRepo.findFeeByPairAndMerchant(fromCurrency, toCurrency, merchantId, LocalDate.now())
            : Optional.empty();

        if (fee.isEmpty()) {
            fee = feeRepo.findDefaultFeeByPair(fromCurrency, toCurrency, LocalDate.now());
        }

        BigDecimal feePercent = fee.isPresent()
            ? fee.get().getFeePercentage()
            : new BigDecimal("0.025"); // 2.5% default

        // Cache for 24 hours
        redisTemplate.opsForHash().put(cacheKey, feeKey, feePercent.toPlainString());
        redisTemplate.expire(cacheKey, 24 * 3600, java.util.concurrent.TimeUnit.SECONDS);

        return feePercent;
    }

    /**
     * Check if FX rate is stale (>24 hours old)
     */
    public boolean isRateStale(String fromCurrency, String toCurrency) {
        Optional<FxRate> rate = fxRateRepo.findLatestRate(fromCurrency, toCurrency);
        if (rate.isEmpty()) {
            return true;
        }

        long ageMs = System.currentTimeMillis() - rate.get().getFetchedAt().getTime();
        return ageMs > (24 * 3600 * 1000);
    }
}

@Entity
@Table(name = "fx_rates")
public class FxRate {
    @Id private Long id;
    private LocalDate rateDate;
    private String fromCurrency;
    private String toCurrency;
    private BigDecimal rate;
    private String source;
    private java.util.Date fetchedAt;

    // Getters, Setters
}

@Repository
public interface FxRateRepository extends JpaRepository<FxRate, Long> {
    @Query("""
        SELECT f FROM FxRate f
        WHERE f.fromCurrency = ?1 AND f.toCurrency = ?2
        ORDER BY f.rateDate DESC
        LIMIT 1
    """)
    Optional<FxRate> findLatestRate(String fromCurrency, String toCurrency);
}
```

### 2. ConversionService — Price Conversion with Fees

```java
package com.payment.fx.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import java.math.BigDecimal;

@Service
public class ConversionService {

    @Autowired private FxRateService fxRateService;
    @Autowired private ConversionRepository conversionRepo;

    /**
     * Convert price from base currency to display currency
     * Includes conversion fee
     */
    public ConversionResult convertPrice(long basePriceCents, String baseCurrency, String displayCurrency,
                                        Long merchantId) {

        BigDecimal feePercent = fxRateService.getConversionFee(baseCurrency, displayCurrency, merchantId);
        long displayPriceCents = fxRateService.convertAmount(basePriceCents, baseCurrency, displayCurrency, feePercent);

        BigDecimal rate = fxRateService.getRate(baseCurrency, displayCurrency);
        long feeCents = displayPriceCents - (long)(basePriceCents * rate.doubleValue());

        ConversionResult result = new ConversionResult();
        result.setBasePriceCents(basePriceCents);
        result.setDisplayPriceCents(displayPriceCents);
        result.setFeeCents(feeCents);
        result.setRate(rate);
        result.setBaseCurrency(baseCurrency);
        result.setDisplayCurrency(displayCurrency);
        result.setConvertedAt(java.time.LocalDateTime.now());

        // Audit trail
        conversionRepo.save(new Conversion(
            basePriceCents, displayPriceCents, baseCurrency, displayCurrency,
            rate, feePercent, java.time.LocalDateTime.now()
        ));

        return result;
    }

    /**
     * Batch convert multiple prices (e.g., for product catalog)
     */
    public java.util.List<ConversionResult> convertPricesBatch(
            java.util.List<Long> basePriceCents, String baseCurrency, String displayCurrency,
            Long merchantId) {

        BigDecimal feePercent = fxRateService.getConversionFee(baseCurrency, displayCurrency, merchantId);

        return basePriceCents.stream()
            .map(price -> convertPrice(price, baseCurrency, displayCurrency, merchantId))
            .collect(java.util.stream.Collectors.toList());
    }

    /**
     * Real-time conversion API endpoint
     */
    public ConversionResult getRealTimeConversion(long amount, String fromCurrency, String toCurrency) {
        BigDecimal feePercent = fxRateService.getConversionFee(fromCurrency, toCurrency, null);
        long convertedAmount = fxRateService.convertAmount(amount, fromCurrency, toCurrency, feePercent);

        ConversionResult result = new ConversionResult();
        result.setBasePriceCents(amount);
        result.setDisplayPriceCents(convertedAmount);
        result.setRate(fxRateService.getRate(fromCurrency, toCurrency));
        result.setBaseCurrency(fromCurrency);
        result.setDisplayCurrency(toCurrency);
        result.setConvertedAt(java.time.LocalDateTime.now());

        return result;
    }
}

public class ConversionResult {
    private long basePriceCents;
    private long displayPriceCents;
    private long feeCents;
    private BigDecimal rate;
    private String baseCurrency;
    private String displayCurrency;
    private java.time.LocalDateTime convertedAt;

    // Getters, Setters
}
```

### 3. CheckoutRateLockService — Rate Locking at Cart Creation

```java
package com.payment.fx.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.concurrent.TimeUnit;

@Service
public class CheckoutRateLockService {

    @Autowired private CartRepository cartRepo;
    @Autowired private FxRateService fxRateService;
    @Autowired private ConversionService conversionService;
    @Autowired private RedisTemplate<String, String> redisTemplate;

    private static final int RATE_LOCK_MINUTES = 15;
    private static final int RATE_EXPIRY_WARNING_MINUTES = 3;

    /**
     * Lock FX rate at cart creation
     */
    @Transactional
    public Cart lockCartRate(Long customerId, java.util.List<Long> productIds,
                            String baseCurrency, String displayCurrency) {

        // Fetch product prices
        long subtotalBaseCents = fetchProductSubtotal(productIds);

        // Get current rate
        BigDecimal lockedRate = fxRateService.getRate(baseCurrency, displayCurrency);

        // Convert to display currency with fee
        BigDecimal feePercent = fxRateService.getConversionFee(baseCurrency, displayCurrency, null);
        long subtotalDisplayCents = fxRateService.convertAmount(subtotalBaseCents, baseCurrency,
            displayCurrency, feePercent);

        long feeCents = (long)(subtotalBaseCents * lockedRate.doubleValue() * feePercent.doubleValue());

        // Create cart
        Cart cart = new Cart();
        cart.setCustomerId(customerId);
        cart.setProductIds(productIds.stream().map(String::valueOf).collect(java.util.stream.Collectors.joining(",")));
        cart.setBaseCurrency(baseCurrency);
        cart.setDisplayCurrency(displayCurrency);
        cart.setLockedRate(lockedRate);
        cart.setRateLockedAt(LocalDateTime.now());
        cart.setRateExpiresAt(LocalDateTime.now().plusMinutes(RATE_LOCK_MINUTES));
        cart.setSubtotalBaseCents(subtotalBaseCents);
        cart.setSubtotalDisplayCents(subtotalDisplayCents);
        cart.setConversionFeeCents(feeCents);
        cart.setTotalDisplayCents(subtotalDisplayCents + feeCents);
        cart.setStatus("ACTIVE");

        Cart savedCart = cartRepo.save(cart);

        // Cache rate lock in Redis
        String rateKey = "cart:rate:" + savedCart.getId();
        String rateJson = String.format(
            "{\"from_currency\": \"%s\", \"to_currency\": \"%s\", \"rate\": %.8f, \"locked_at\": %d}",
            baseCurrency, displayCurrency, lockedRate, System.currentTimeMillis()
        );
        redisTemplate.opsForValue().set(rateKey, rateJson);
        redisTemplate.expire(rateKey, RATE_LOCK_MINUTES + 5, TimeUnit.MINUTES);

        return savedCart;
    }

    /**
     * Check if cart rate is still valid; warn if expiring soon
     */
    public RateLockStatus checkRateLockStatus(Long cartId) {
        Cart cart = cartRepo.findById(cartId).orElseThrow();

        LocalDateTime now = LocalDateTime.now();
        LocalDateTime expiry = cart.getRateExpiresAt();

        if (now.isAfter(expiry)) {
            return new RateLockStatus(false, "Rate expired, please refresh cart", true);
        }

        long minutesUntilExpiry = java.time.temporal.ChronoUnit.MINUTES.between(now, expiry);
        if (minutesUntilExpiry <= RATE_EXPIRY_WARNING_MINUTES) {
            return new RateLockStatus(true, "Rate expires in " + minutesUntilExpiry + " minutes", true);
        }

        return new RateLockStatus(true, "Rate is valid", false);
    }

    /**
     * Refresh/renew rate lock on cart
     */
    @Transactional
    public Cart refreshCartRate(Long cartId) {
        Cart cart = cartRepo.findById(cartId).orElseThrow();

        // Clear old lock
        String oldRateKey = "cart:rate:" + cartId;
        redisTemplate.delete(oldRateKey);

        // Lock new rate
        return lockCartRate(cart.getCustomerId(),
            java.util.Arrays.stream(cart.getProductIds().split(","))
                .map(Long::parseLong)
                .collect(java.util.stream.Collectors.toList()),
            cart.getBaseCurrency(),
            cart.getDisplayCurrency()
        );
    }

    private long fetchProductSubtotal(java.util.List<Long> productIds) {
        return productIds.stream()
            .map(id -> productRepo.findById(id).orElseThrow())
            .mapToLong(Product::getPriceCents)
            .sum();
    }
}

public class RateLockStatus {
    private boolean valid;
    private String message;
    private boolean showWarning;

    public RateLockStatus(boolean valid, String message, boolean showWarning) {
        this.valid = valid;
        this.message = message;
        this.showWarning = showWarning;
    }

    // Getters, Setters
}
```

### 4. MultiCurrencyLedgerService — Dual-Currency Journal Entries

```java
package com.payment.fx.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;

@Service
public class MultiCurrencyLedgerService {

    @Autowired private LedgerEntryRepository ledgerRepo;
    @Autowired private SettlementRepository settlementRepo;

    /**
     * Record multi-currency transaction as journal entries
     * Creates debit/credit pair with FX conversion tracking
     */
    @Transactional
    public void recordTransaction(String transactionId, Long customerId, Long merchantId,
                                 String customerCurrency, String merchantCurrency,
                                 long customerAmountCents, long merchantAmountCents,
                                 BigDecimal exchangeRate, long fxGainLossCents,
                                 long conversionFeeCents) {

        // Group ID for matching debit/credit entries
        Long journalEntryId = System.currentTimeMillis();

        // Entry 1: Debit customer account (in customer currency)
        LedgerEntry debitEntry = new LedgerEntry();
        debitEntry.setTransactionId(transactionId);
        debitEntry.setJournalEntryId(journalEntryId);
        debitEntry.setAccount("customer:" + customerId);
        debitEntry.setCurrency(customerCurrency);
        debitEntry.setAmountCents(customerAmountCents);
        debitEntry.setEntryType("DEBIT");
        debitEntry.setDescription("Payment from customer");
        ledgerRepo.save(debitEntry);

        // Entry 2: Credit revenue (in customer currency)
        LedgerEntry creditRevenue = new LedgerEntry();
        creditRevenue.setTransactionId(transactionId);
        creditRevenue.setJournalEntryId(journalEntryId);
        creditRevenue.setAccount("revenue:settlement");
        creditRevenue.setCurrency(customerCurrency);
        creditRevenue.setAmountCents(merchantAmountCents - conversionFeeCents);
        creditRevenue.setEntryType("CREDIT");
        creditRevenue.setDescription("Revenue from transaction (conversion-adjusted)");
        ledgerRepo.save(creditRevenue);

        // Entry 3: Credit conversion fee
        LedgerEntry feesEntry = new LedgerEntry();
        feesEntry.setTransactionId(transactionId);
        feesEntry.setJournalEntryId(journalEntryId);
        feesEntry.setAccount("revenue:conversion_fees");
        feesEntry.setCurrency(customerCurrency);
        feesEntry.setAmountCents(conversionFeeCents);
        feesEntry.setEntryType("CREDIT");
        feesEntry.setDescription("Conversion fees");
        ledgerRepo.save(feesEntry);

        // Entry 4: Debit merchant settlement (in merchant currency) - if different
        if (!customerCurrency.equals(merchantCurrency)) {
            LedgerEntry merchantDebit = new LedgerEntry();
            merchantDebit.setTransactionId(transactionId);
            merchantDebit.setJournalEntryId(journalEntryId);
            merchantDebit.setAccount("merchant_settlement:" + merchantId);
            merchantDebit.setCurrency(merchantCurrency);
            merchantDebit.setAmountCents(merchantAmountCents);
            merchantDebit.setEntryType("DEBIT");
            merchantDebit.setDescription("Settlement to merchant in local currency");
            ledgerRepo.save(merchantDebit);

            // Entry 5: FX gain/loss
            LedgerEntry fxEntry = new LedgerEntry();
            fxEntry.setTransactionId(transactionId);
            fxEntry.setJournalEntryId(journalEntryId);
            fxEntry.setAccount(fxGainLossCents > 0 ? "revenue:fx_gain" : "expense:fx_loss");
            fxEntry.setCurrency("USD"); // Base currency for FX tracking
            fxEntry.setAmountCents(Math.abs(fxGainLossCents));
            fxEntry.setEntryType(fxGainLossCents > 0 ? "CREDIT" : "DEBIT");
            fxEntry.setDescription("FX gain/loss on rate " + exchangeRate);
            ledgerRepo.save(fxEntry);
        }
    }

    /**
     * Reconcile ledger: verify sum(debits) == sum(credits) for transaction
     */
    public boolean reconcileTransaction(String transactionId) {
        var entries = ledgerRepo.findByTransactionId(transactionId);

        long totalDebits = entries.stream()
            .filter(e -> "DEBIT".equals(e.getEntryType()))
            .mapToLong(LedgerEntry::getAmountCents)
            .sum();

        long totalCredits = entries.stream()
            .filter(e -> "CREDIT".equals(e.getEntryType()))
            .mapToLong(LedgerEntry::getAmountCents)
            .sum();

        // Note: In multi-currency, balancing is complex; simplified here
        return totalDebits == totalCredits;
    }

    /**
     * Month-end FX revaluation
     * Revalue open positions at current rates; record gain/loss
     */
    @Transactional
    public void performMonthEndRevaluation(String month, String currency) {
        // Query open settlement records for currency
        var openSettlements = settlementRepo.findOpenByMonthAndCurrency(month, currency);

        BigDecimal currentRate = fxRateService.getRate("USD", currency);

        for (Settlement settlement : openSettlements) {
            BigDecimal originalRate = settlement.getLockedRate();
            long originalAmount = settlement.getTotalAmountCents();

            // Revalue at current rate
            long revaluedAmount = (long)(originalAmount * currentRate.doubleValue());
            long revaluationGainLoss = revaluedAmount - originalAmount;

            // Record revaluation entry
            LedgerEntry revalEntry = new LedgerEntry();
            revalEntry.setTransactionId("REVAL_" + settlement.getId());
            revalEntry.setAccount("expense:fx_revaluation");
            revalEntry.setCurrency(currency);
            revalEntry.setAmountCents(Math.abs(revaluationGainLoss));
            revalEntry.setEntryType(revaluationGainLoss > 0 ? "CREDIT" : "DEBIT");
            revalEntry.setDescription("Month-end revaluation: rate " + originalRate + " → " + currentRate);
            ledgerRepo.save(revalEntry);

            settlement.setActualRate(currentRate);
            settlementRepo.save(settlement);
        }
    }
}

@Entity
@Table(name = "ledger_entries")
public class LedgerEntry {
    @Id private Long id;
    private String transactionId;
    private Long journalEntryId;
    private String account;
    private String currency;
    private long amountCents;
    private String entryType; // DEBIT, CREDIT
    private String description;
    private java.time.LocalDateTime createdAt;

    // Getters, Setters
}
```

### 5. StrongCustomerAuthService — PSD2/SCA Compliance

```java
package com.payment.fx.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class StrongCustomerAuthService {

    @Autowired private ScaExemptionRepository exemptionRepo;
    @Autowired private ThreeDSecureService threeDSecureService;

    private static final long LOW_VALUE_THRESHOLD_CENTS = 3000; // €30

    /**
     * Determine if SCA (Strong Customer Authentication) is required
     * PSD2 Compliance
     */
    public ScaDecision evaluateSCARequirement(String cardCountry, long amountCents,
                                             Long customerId, Long merchantId,
                                             String ipCountry, boolean isNewDevice) {

        boolean isEUCard = isEUCountry(cardCountry);
        boolean isEUIP = isEUCountry(ipCountry);

        // Exemption 1: Low-value transactions (<€30)
        if (amountCents < LOW_VALUE_THRESHOLD_CENTS) {
            return new ScaDecision(false, "LOW_VALUE_EXEMPTION", "Amount below €30 threshold");
        }

        // Exemption 2: Trusted beneficiary (customer has whitelisted merchant)
        var exemption = exemptionRepo.findByCustomerAndMerchant(customerId, merchantId);
        if (exemption.isPresent() && "TRUSTED_BENEFICIARY".equals(exemption.get().getExemptionType())) {
            return new ScaDecision(false, "TRUSTED_BENEFICIARY", "Merchant is trusted by customer");
        }

        // SCA Required: EU card, high amount, or unusual activity
        if (isEUCard && (amountCents > 50000 || isNewDevice || !isEUIP)) {
            return new ScaDecision(true, "HIGH_RISK", "EU card with high amount or unusual activity");
        }

        // Frictionless 3DS for moderate risk (device data only, often auto-approved)
        if (isEUCard && amountCents > 10000) {
            return new ScaDecision(true, "FRICTIONLESS_3DS", "Frictionless 3DS available");
        }

        return new ScaDecision(false, "NO_SCA_REQUIRED", "Low risk");
    }

    /**
     * Request 3DS authentication if required
     */
    public ThreeDSecureResult request3DS(String transactionId, String cardHash, long amountCents,
                                         ScaDecision scaDecision) {
        if (!scaDecision.isRequired()) {
            return new ThreeDSecureResult("NO_CHALLENGE", "SCA not required", null);
        }

        if ("FRICTIONLESS_3DS".equals(scaDecision.getReason())) {
            // Frictionless: use device data; issuer approves silently
            return threeDSecureService.initiate3DSFrictionless(transactionId, cardHash, amountCents);
        } else {
            // Full challenge: user enters password/OTP
            return threeDSecureService.initiate3DSChallenge(transactionId, cardHash, amountCents);
        }
    }

    /**
     * Whitelist merchant as trusted beneficiary
     */
    public void whitelistMerchant(Long customerId, Long merchantId) {
        ScaExemption exemption = new ScaExemption();
        exemption.setCustomerId(customerId);
        exemption.setMerchantId(merchantId);
        exemption.setExemptionType("TRUSTED_BENEFICIARY");
        exemption.setEnabled(true);
        exemption.setWhitelistedAt(java.time.LocalDateTime.now());

        exemptionRepo.save(exemption);
    }

    private boolean isEUCountry(String countryCode) {
        java.util.Set<String> euCountries = java.util.Set.of(
            "AT", "BE", "BG", "HR", "CY", "CZ", "DK", "EE", "FI", "FR",
            "DE", "GR", "HU", "IE", "IT", "LV", "LT", "LU", "MT", "NL",
            "PL", "PT", "RO", "SK", "SI", "ES", "SE"
        );
        return euCountries.contains(countryCode);
    }
}

public class ScaDecision {
    private boolean required;
    private String reason; // LOW_VALUE_EXEMPTION, TRUSTED_BENEFICIARY, HIGH_RISK, FRICTIONLESS_3DS
    private String message;

    public ScaDecision(boolean required, String reason, String message) {
        this.required = required;
        this.reason = reason;
        this.message = message;
    }

    // Getters, Setters
}
```

---

## Failure Scenarios & Recovery

| Scenario | Detection | Recovery |
|----------|-----------|----------|
| **FX rate source down (ECB)** | HTTP 500 from ECB | Fall back to OpenExchangeRates API; use previous day's rate with warning |
| **Rate cache miss during peak** | Redis connection failed | Fall back to PostgreSQL query; slower but functional |
| **Rate stale (>24h)** | Age check in getRate() | Reject conversion; alert ops to fetch rates manually |
| **Cart rate expires before checkout** | checkRateLockStatus() | Display warning; require customer to refresh cart with new rate |
| **Settlement currency unavailable** | FxRate not found for merchant currency | Revert to settlement in USD; adjust in next billing cycle |
| **Ledger debit/credit mismatch** | Monthly reconciliation fails | Alert finance team; investigate transaction; manual journal entry |
| **Concurrent cart rate updates** | Optimistic lock violation | Retry refresh; use most recent rate |

---

## Scaling Strategy

### Horizontal Scaling
1. **FX Rate Service:** Stateless; scale across 5+ nodes; load balance via round-robin
2. **Conversion Service:** Stateless; scale across 10+ nodes
3. **Redis Cluster:** 8 nodes for rate caching; auto-failover via Sentinel
4. **PostgreSQL:** Read replicas for reporting; main for transactional updates

### Caching Strategy
- **FX Rates:** Cache for 23h50m in Redis (high hit rate >99%)
- **Conversion Fees:** Cache for 24h (low change frequency)
- **Cart Rates:** TTL 20 minutes (explicit expiry)

---

## Monitoring & Alerting

### Key Metrics
```
# Throughput
conversion.requests.total (counter)
conversion.latency_ms (histogram, p99 < 50ms)
ledger.entries.created.total (counter)

# Reliability
fx_rates.fetch_failures.total (counter)
ledger.reconciliation.mismatches.total (counter)
sca.challenges.total (counter)

# Business Metrics
revenue.by_currency.daily (gauge)
conversion_fees.by_pair.monthly (gauge)
settlement.pending_amount_cents (gauge)
fx_revaluation.gain_loss_monthly (gauge)
```

### Alert Rules
```
ALERT FxRateUpdateFailed
  IF rate(fx_rates.fetch_failures.total[1h]) > 0
  FOR 30m

ALERT LedgerMismatch
  IF ledger.reconciliation.mismatches.total > 10
  FOR 1h

ALERT CartRateExpiry
  IF rate(carts.rate_expired[5m]) > 100
  FOR 10m
```

---

## Summary Cheat Sheet

| Component | Pattern | Key Code |
|-----------|---------|----------|
| **FX Rate Management** | Daily fetch from ECB → PostgreSQL + Redis cache (23h50m) | `FxRateService.getRate()` |
| **Price Conversion** | amount × rate × (1 + fee%) rounded HALF_UP | `ConversionService.convertPrice()` |
| **Cart Rate Lock** | Lock at creation, valid 15 min, warn at 12 min | `CheckoutRateLockService.lockCartRate()` |
| **Conversion Fees** | 2.5% default, per-currency-pair configurable | `fx:fees:{date}` Redis hash |
| **Dual-Currency Ledger** | Debit in customer currency, credit in merchant currency | `LedgerEntry` debit/credit pairs |
| **FX Gain/Loss** | (merchant_amount - customer_amount × rate) | Journal entry with FX account |
| **Month-End Revaluation** | Revalue open positions at current rate | `performMonthEndRevaluation()` |
| **PSD2 Compliance** | Exempt low-value (<€30) + trusted beneficiaries; require 3DS for high-risk EU | `StrongCustomerAuthService` |
| **Frictionless 3DS** | Device data only, often auto-approved | `threeDSecureService.initiate3DSFrictionless()` |
| **Rate Cache TTL** | 23h50m (refresh daily before expiry) | `redisTemplate.expire(cacheKey, 23h50m)` |

