---
title: Merchant Payout and Settlement System — Deep Dive Design
layout: default
---

# Merchant Payout and Settlement System — Deep Dive Design

> **Scenario:** Platform takes 15% commission. Sellers paid weekly for completed orders. Deduct refunds and chargebacks from payouts. Handle different payout methods (bank transfer, PayPal). Different fee structures per seller tier. Tax reporting (1099 forms). 10K sellers.
>
> **Tech Stack:** Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka

---

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
10. [Failure Scenarios](#failure-scenarios)
11. [Scaling Strategy](#scaling-strategy)
12. [Monitoring](#monitoring)
13. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Requirements & Constraints

### Functional Requirements
- **Weekly settlement:** pay sellers every Friday for orders completed in previous week
- **Commission deduction:** 15% platform fee applied at settlement (configurable per tier: standard 15%, premium 10%, enterprise custom)
- **Refund handling:** deduct from NEXT payout (clawback) if refund issued after payout
- **Chargeback handling:** deduct from payouts; dispute claims tracked separately
- **Payout methods:** ACH bank transfer (USA), Wire (international), PayPal, Stripe Connect
- **Tax reporting:** 1099-NEC forms generated annually for sellers earning >$600; IRS e-file submission
- **Holds & reserves:** hold 5% of payout for disputes; release after 90 days if no chargebacks
- **Failed payouts:** retry 3x (T+1, T+3, T+7); then hold for manual review
- **Multi-currency:** support USD, EUR, GBP; FX settled weekly

### Non-Functional Requirements
- **Scale:** 10,000 sellers; ~$500M GMV/month; ~50,000 payouts/week
- **Latency:** payout calculation <5 seconds; ledger operations <100ms
- **Availability:** 99.9% uptime (payment critical)
- **Consistency:** ledger entries immutable; all transactions double-entry bookkeeping
- **Audit:** complete audit trail; PCI compliance for bank account data (tokenized)

---

## Capacity Estimation

| Metric | Volume | Notes |
|--------|--------|-------|
| Sellers (10K) | 10,000 | active marketplace sellers |
| Weekly Payouts | 50,000 | ~5 per seller (some inactive) |
| Monthly Settlement Revenue | $75M | $500M GMV × 15% commission |
| Payout Ledger Entries/Week | 500,000 | credit + debit per order + adjustments |
| Chargebacks/Week | 5,000 | ~0.1% of transactions; deducted from payouts |
| Refunds/Week | 50,000 | ~10% of orders; clawed back from next payout |
| 1099 Forms Generated/Year | 10,000 | all sellers earning >$600 |
| Storage (2 years) | ~500 GB | ledger entries, payout records, audit trail |
| Redis Memory | ~100 MB | seller balances, payout state cache |

---

## High-Level Architecture

```
                    ┌────────────────────────────────────┐
                    │  Order Service                      │
                    │  - Order placement, completion      │
                    │  - Emit: OrderCompleted (Kafka)     │
                    └────────────────┬────────────────────┘
                                     │
                    ┌────────────────v────────────────────┐
                    │  Settlement Service                  │
                    │  - Track completed orders            │
                    │  - Build payout ledger               │
                    └────────────────┬────────────────────┘
                                     │
                    ┌────────────────v────────────────────┐
                    │  Payout Calculation Job (Weekly)    │
                    │  - Aggregate ledger per seller       │
                    │  - Apply commission + tiers          │
                    │  - Deduct refunds/chargebacks        │
                    │  - Apply hold/reserve (5%)           │
                    │  - Create PayoutRecord               │
                    └────────────────┬────────────────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         │                           │                           │
         v                           v                           v
    ┌─────────────┐         ┌──────────────────┐      ┌──────────────────┐
    │ ACH/Bank    │         │ PayPal/PayPal    │      │ Stripe Connect   │
    │ Gateway     │         │ Gateway          │      │ Gateway          │
    │ (slow)      │         │ (medium)         │      │ (fast)           │
    └─────────────┘         └──────────────────┘      └──────────────────┘
         │                           │                           │
         └───────────────────────────┼───────────────────────────┘
                                     │
                    ┌────────────────v────────────────────┐
                    │  Kafka Topics                        │
                    │  - PayoutCreated                     │
                    │  - PayoutProcessed                   │
                    │  - PayoutFailed                      │
                    │  - ReserveHoldReleased               │
                    │  - ChargebackReceived                │
                    │  - RefundIssued                      │
                    └─────────────┬────────────────────────┘
                                  │
         ┌───────────────────────┬─┼────────────────┬──────────────────┐
         │                       │ │                │                  │
         v                       v v                v                  v
    ┌──────────┐          ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
    │ Retry    │          │ Accounting   │   │ Tax Reporting│   │ Reconciliation
    │ Service  │          │ Service      │   │ Service      │   │ Service
    └──────────┘          └──────────────┘   └──────────────┘   └──────────────┘

        ┌──────────────────────────────────────────┐
        │  PostgreSQL                               │
        │  - Sellers, Payout Methods                │
        │  - Payout Ledger (immutable)              │
        │  - Payout Records, Audit Log              │
        │  - Chargeback/Refund Claims               │
        │  - Tax Data (1099 tracking)               │
        └──────────────────────────────────────────┘

        ┌──────────────────────────────────────────┐
        │  MongoDB                                  │
        │  - High-volume audit trail                │
        │  - Chargeback evidence files              │
        │  - Tax forms (JSON/PDF)                   │
        └──────────────────────────────────────────┘

        ┌──────────────────────────────────────────┐
        │  Redis                                    │
        │  - Seller balances (cache)                │
        │  - Payout state (in-progress)             │
        │  - Reserve tracking                       │
        └──────────────────────────────────────────┘
```

---

## Core Design Questions Answered

### 1. How do you calculate seller payouts accurately?

**Answer:**
- **Payout Ledger:** immutable append-only table with entries:
  ```
  seller_id | type    | amount   | related_order_id | created_at
  sell1     | CREDIT  | 850      | order123         | 2024-01-08 10:00
  sell1     | DEBIT   | 150      | order123         | 2024-01-08 10:00 (commission)
  sell1     | CREDIT  | 1200     | order456         | 2024-01-08 12:00
  sell1     | DEBIT   | 180      | order456         | 2024-01-08 12:00 (commission)
  sell1     | DEBIT   | 100      | order123         | 2024-01-10 14:00 (refund clawback)
  ```
- **Payout Calculation (Weekly):**
  ```
  SELECT seller_id,
         SUM(CASE WHEN type = 'CREDIT' THEN amount ELSE 0 END) as credits,
         SUM(CASE WHEN type = 'DEBIT' THEN amount ELSE 0 END) as debits,
         SUM(CASE WHEN type = 'CREDIT' THEN amount ELSE 0 END) -
         SUM(CASE WHEN type = 'DEBIT' THEN amount ELSE 0 END) as payout_amount
  FROM payout_ledger
  WHERE seller_id = ?
    AND created_at >= DATE_TRUNC('week', NOW()) - INTERVAL '1 week'
    AND created_at < DATE_TRUNC('week', NOW())
  GROUP BY seller_id
  ```
- **Application Reserve:** hold 5% of payout in escrow:
  ```
  payout_to_seller = total_amount * 0.95
  reserve_held = total_amount * 0.05
  ```
- **Idempotency:** if payout calculation is re-run for same week → returns same result (immutable ledger)

### 2. How do you handle refunds that occur after payout?

**Answer:**
- **Post-Payout Refund Scenario:**
  ```
  Timeline:
  Friday Jan 5: Payout Week 1 processed for seller ($10K)
  Tuesday Jan 9: Customer requests refund for order from Week 1
  Friday Jan 12: Payout Week 2 calculated
  ```
- **Solution:** clawback via negative ledger entry in following week:
  ```
  payout_ledger INSERT:
    seller_id: sell1
    type: DEBIT (negative adjustment)
    amount: 100 (refund amount × (1 - commission_rate)) = 100 × 0.85
    reason: refund_clawback_for_order_123
    created_at: 2024-01-10 (refund request date)
    related_order_id: order123
  ```
- **Week 2 Payout Calculation:** includes the DEBIT entry:
  ```
  Week 2 payout = credit_entries - debit_entries
               = week2_credits - week2_debits (includes clawback debit)
  ```
- **Partial Refund:** deduct proportional to seller's net (before commission)
  ```
  refund_amount = 50 (customer refund)
  seller_share = 50 * (1 - commission_rate) = 50 * 0.85 = 42.50
  clawback_entry = 42.50 DEBIT
  ```

### 3. How do you process payouts to different destinations?

**Answer:**
- **Payout Method Abstraction:**
  ```java
  interface PayoutGateway {
    PayoutResponse pay(PayoutRequest request);
    PayoutStatusResponse getStatus(String payoutId);
    PayoutRefundResponse refundPayout(String payoutId, BigDecimal amount);
  }

  class ACHPayoutGateway implements PayoutGateway { ... }
  class PayPalPayoutGateway implements PayoutGateway { ... }
  class StripeConnectPayoutGateway implements PayoutGateway { ... }
  ```
- **Routing:** based on seller's preferred method + country:
  ```java
  Seller.payoutMethod = "ach"      // ACH bank transfer
  Seller.country = "US"
  → use ACHPayoutGateway

  Seller.payoutMethod = "paypal"
  → use PayPalPayoutGateway

  Seller.payoutMethod = "stripe"
  → use StripeConnectPayoutGateway (Stripe account already linked)
  ```
- **Idempotent Payout:**
  - PayoutRequest includes `idempotencyKey` = hash(payoutRecordId + seller + amount)
  - Gateway remembers key for 24 hours; duplicate calls return same result
  - If gateway returns "already processed" → use cached payout ID
- **Bank Account Security:** all bank details tokenized via payment processor (Stripe, Plaid); raw account numbers never stored

### 4. How do you implement different commission structures?

**Answer:**
- **Seller Tiers:** stored in `sellers` table:
  ```
  seller_id | tier        | monthly_gmv    | commission_rate
  sell1     | standard    | 50,000         | 0.15
  sell2     | premium     | 200,000        | 0.10
  sell3     | enterprise  | 5,000,000      | 0.05
  ```
- **Commission Calculation (at order completion):**
  ```java
  @Transactional
  public void recordOrderCompletion(Order order) {
      Seller seller = order.getSeller();
      BigDecimal commissionRate = getCommissionRateForTier(seller.getTier());
      BigDecimal commission = order.getAmount().multiply(commissionRate);

      // CREDIT entry for seller
      PayoutLedgerEntry credit = new PayoutLedgerEntry(
          UUID.randomUUID(),
          seller.getId(),
          "CREDIT",
          order.getAmount(),  // gross amount
          order.getId(),
          Instant.now()
      );
      ledgerRepository.save(credit);

      // DEBIT entry for commission
      PayoutLedgerEntry debit = new PayoutLedgerEntry(
          UUID.randomUUID(),
          seller.getId(),
          "DEBIT",
          commission,
          order.getId(),
          Instant.now()
      );
      ledgerRepository.save(debit);
  }
  ```
- **Tier Calculation (monthly):** SellerTierService runs at month-end:
  ```java
  public void updateSellerTiers() {
    for (Seller seller : sellers) {
        BigDecimal monthlyGmv = calculateMonthlyGMV(seller.getId());
        String newTier = determineNewTier(monthlyGmv);
        if (!newTier.equals(seller.getTier())) {
            seller.setTier(newTier);
            seller.setTierEffectiveDate(LocalDate.now().plusMonths(1));
            sellerRepository.save(seller);
            // Emit event for audit
        }
    }
  }

  private String determineNewTier(BigDecimal gmv) {
    if (gmv.compareTo(new BigDecimal("1000000")) >= 0) return "enterprise"; // >$1M
    if (gmv.compareTo(new BigDecimal("100000")) >= 0) return "premium";     // >$100K
    return "standard";
  }
  ```

### 5. How do you generate tax reports at year-end?

**Answer:**
- **1099-NEC Eligibility:** seller earning >$600 in calendar year
- **Annual Tax Calculation (Jan 1, 12:00 UTC):**
  ```sql
  SELECT seller_id, SUM(payout_amount) as annual_earnings
  FROM payout_records
  WHERE EXTRACT(YEAR FROM created_at) = 2023
    AND status = 'completed'
  GROUP BY seller_id
  HAVING SUM(payout_amount) > 600
  ```
- **1099 Form Generation:**
  ```java
  public void generateAnnual1099s() {
    List<Seller> taxableEligibleSellers = getTaxableSellers(currentYear);

    for (Seller seller : taxableEligibleSellers) {
        BigDecimal annualEarnings = calculateAnnualEarnings(seller.getId(), currentYear);

        Form1099NEC form = new Form1099NEC();
        form.setRecipientId(seller.getId());
        form.setRecipientName(seller.getLegalName());
        form.setRecipientTin(seller.getTaxId());  // SSN or EIN
        form.setPayerName("Platform Inc");
        form.setPayerTin("XX-XXXXXXX");
        form.setNonEmployeeCompensation(annualEarnings);  // Box 1
        form.setTaxYear(currentYear);

        // Generate PDF via iTextPDF
        byte[] pdfContent = generateForm1099Pdf(form);

        // Store in MongoDB
        mongoTemplate.save(new Tax1099Document(
            UUID.randomUUID(),
            seller.getId(),
            currentYear,
            form.toJson(),
            pdfContent,
            Instant.now()
        ));

        // Track for IRS e-file submission
        addToIRSEFileQueue(form);
    }

    // Submit to IRS (via TurboTax/e-services provider)
    submitToIRS(currentYear);
  }
  ```
- **Multi-Tax-ID Tracking:** if seller has multiple SSN/EIN (LLC + personal), generate separate 1099 for each
- **Backup Withholding:** if seller missing or invalid TIN → withhold 24% of payouts; track for IRS form W-8BEN

### 6. How do you handle payout failures and retries?

**Answer:**
- **Retry Strategy:**
  ```
  Attempt 1: T+1 (next day): exponential backoff, 1 hour
  Attempt 2: T+3 (3 days): exponential backoff, 2 hours
  Attempt 3: T+7 (7 days): exponential backoff, 4 hours
  After 3 failures: HOLD_FOR_REVIEW → manual intervention queue
  ```
- **Failure Handling:**
  ```java
  @Scheduled(cron = "0 6 * * *")  // Daily at 6 AM
  public void retryFailedPayouts() {
    List<PayoutRecord> failedPayouts = payoutRepository
        .findByStatusAndNextRetryTimeLessThanEqual("failed", Instant.now());

    for (PayoutRecord payout : failedPayouts) {
        if (payout.getRetryCount() >= 3) {
            // Move to manual review
            payout.setStatus("hold_for_review");
            payout.setHoldReason("Max retries exceeded");
            payoutRepository.save(payout);

            // Alert support team
            alertService.sendAlert(
                "Payout #" + payout.getId() + " requires manual review",
                "critical"
            );
            continue;
        }

        try {
            PayoutResponse response = payoutGateway.pay(payout.toRequest());
            if ("success".equals(response.getStatus())) {
                payout.setStatus("completed");
                payout.setProcessedAt(Instant.now());
                payout.setGatewayPayoutId(response.getPayoutId());
            } else if ("pending".equals(response.getStatus())) {
                payout.setStatus("pending");
                payout.setNextRetryTime(calculateNextRetryTime(payout.getRetryCount() + 1));
            } else {
                payout.setStatus("failed");
                payout.setFailureReason(response.getErrorMessage());
                payout.setRetryCount(payout.getRetryCount() + 1);
                payout.setNextRetryTime(calculateNextRetryTime(payout.getRetryCount()));
            }
            payoutRepository.save(payout);
        } catch (Exception e) {
            log.error("Error retrying payout {}", payout.getId(), e);
            payout.setStatus("failed");
            payout.setFailureReason(e.getMessage());
            payout.setRetryCount(payout.getRetryCount() + 1);
            payout.setNextRetryTime(calculateNextRetryTime(payout.getRetryCount()));
            payoutRepository.save(payout);
        }
    }
  }

  private Instant calculateNextRetryTime(int retryCount) {
    long delayMinutes = (long) Math.pow(2, retryCount);  // 2, 4, 8, ... minutes
    return Instant.now().plusSeconds(delayMinutes * 60);
  }
  ```

---

## Microservices Breakdown

| Microservice | Port | Responsibilities | Key Dependencies |
|--------------|------|------------------|------------------|
| Settlement Service | 8011 | Track order completions; build ledger | PostgreSQL, Kafka |
| Payout Calculation Service | 8012 | Weekly payout calculation; apply commissions | PostgreSQL, Redis |
| Payout Gateway Router | 8013 | Route payouts to ACH/PayPal/Stripe | Payment gateway APIs |
| Payout Retry Service | 8014 | Handle failed payouts; exponential backoff | PostgreSQL, Payment gateway APIs |
| Tax Reporting Service | 8015 | Generate 1099s; IRS e-file submission | MongoDB, IRS e-services, PostgreSQL |
| Chargeback Service | 8016 | Track chargebacks; apply deductions | PostgreSQL, Kafka |

---

## Database Design (DDL)

### PostgreSQL Schema

```sql
-- Sellers
CREATE TABLE sellers (
    id UUID PRIMARY KEY,
    legal_name VARCHAR(255) NOT NULL,
    business_name VARCHAR(255),
    email VARCHAR(255) NOT NULL UNIQUE,
    tier VARCHAR(50) DEFAULT 'standard',  -- standard, premium, enterprise
    tier_effective_date DATE,
    tax_id VARCHAR(20),  -- SSN or EIN (encrypted)
    country VARCHAR(2) DEFAULT 'US',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_sellers_tier ON sellers(tier);
CREATE INDEX idx_sellers_tax_year ON sellers(created_at);

-- Seller Payout Methods
CREATE TABLE seller_payout_methods (
    id UUID PRIMARY KEY,
    seller_id UUID NOT NULL UNIQUE REFERENCES sellers(id),
    payout_method VARCHAR(50) NOT NULL,  -- ach, paypal, stripe
    method_token VARCHAR(255) NOT NULL,  -- tokenized account details
    account_last_four VARCHAR(4),
    currency VARCHAR(3) DEFAULT 'USD',
    verified_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payout_methods_seller ON seller_payout_methods(seller_id);

-- Payout Ledger (immutable append-only)
CREATE TABLE payout_ledger (
    id UUID PRIMARY KEY,
    seller_id UUID NOT NULL REFERENCES sellers(id),
    entry_type VARCHAR(50) NOT NULL,  -- CREDIT (order), DEBIT (commission, refund, chargeback)
    amount DECIMAL(15, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    related_order_id UUID,
    ledger_entry_reason VARCHAR(100),  -- order_completion, refund_clawback, chargeback_deduction
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT check_amount_positive CHECK (amount > 0)
);

CREATE INDEX idx_ledger_seller_date ON payout_ledger(seller_id, created_at);
CREATE INDEX idx_ledger_entry_type ON payout_ledger(entry_type);
CREATE INDEX idx_ledger_reason ON payout_ledger(ledger_entry_reason);

-- Payout Records (weekly settlement batches)
CREATE TABLE payout_records (
    id UUID PRIMARY KEY,
    seller_id UUID NOT NULL REFERENCES sellers(id),
    payout_week_start_date DATE NOT NULL,
    payout_week_end_date DATE NOT NULL,
    total_credits DECIMAL(15, 2) NOT NULL,  -- sum of CREDIT entries
    total_debits DECIMAL(15, 2) NOT NULL,   -- sum of DEBIT entries (commission, etc.)
    payout_amount DECIMAL(15, 2) NOT NULL,  -- credits - debits (before reserve)
    reserve_held DECIMAL(15, 2) NOT NULL DEFAULT 0,  -- 5% hold
    net_payout_amount DECIMAL(15, 2) NOT NULL,  -- after reserve
    payout_method VARCHAR(50) NOT NULL,
    gateway_payout_id VARCHAR(255),
    status VARCHAR(50) DEFAULT 'pending',  -- pending, processing, completed, failed, hold_for_review
    failure_reason TEXT,
    retry_count INT DEFAULT 0,
    next_retry_time TIMESTAMP,
    processed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payout_records_seller_status ON payout_records(seller_id, status);
CREATE INDEX idx_payout_records_week ON payout_records(payout_week_start_date, payout_week_end_date);
CREATE INDEX idx_payout_records_status ON payout_records(status, next_retry_time);

-- Reserve Holds (5% escrow from each payout)
CREATE TABLE reserve_holds (
    id UUID PRIMARY KEY,
    payout_record_id UUID NOT NULL UNIQUE REFERENCES payout_records(id),
    seller_id UUID NOT NULL REFERENCES sellers(id),
    hold_amount DECIMAL(15, 2) NOT NULL,
    hold_reason VARCHAR(100),  -- dispute_reserve
    hold_until_date DATE NOT NULL,  -- typically 90 days from payout
    release_status VARCHAR(50) DEFAULT 'pending',  -- pending, released, disputed
    released_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_reserves_seller_status ON reserve_holds(seller_id, release_status);
CREATE INDEX idx_reserves_release_date ON reserve_holds(hold_until_date);

-- Chargebacks
CREATE TABLE chargebacks (
    id UUID PRIMARY KEY,
    seller_id UUID NOT NULL REFERENCES sellers(id),
    order_id UUID NOT NULL,
    customer_id UUID NOT NULL,
    dispute_reason VARCHAR(255) NOT NULL,
    chargeback_amount DECIMAL(15, 2) NOT NULL,
    chargeback_date DATE NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',  -- pending, under_review, won, lost
    related_payout_id UUID REFERENCES payout_records(id),
    deducted_from_payout_id UUID REFERENCES payout_records(id),
    deducted_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_chargebacks_seller_status ON chargebacks(seller_id, status);
CREATE INDEX idx_chargebacks_order ON chargebacks(order_id);

-- Chargeback Evidence (supporting docs)
CREATE TABLE chargeback_evidence (
    id UUID PRIMARY KEY,
    chargeback_id UUID NOT NULL REFERENCES chargebacks(id) ON DELETE CASCADE,
    evidence_type VARCHAR(50),  -- order_confirmation, shipment_proof, communication, refund_denial
    document_url VARCHAR(500),  -- S3 URL or MongoDB reference
    uploaded_by VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_evidence_chargeback ON chargeback_evidence(chargeback_id);

-- Tax 1099 Tracking
CREATE TABLE tax_1099_tracking (
    id UUID PRIMARY KEY,
    seller_id UUID NOT NULL REFERENCES sellers(id),
    tax_year INT NOT NULL,
    annual_earnings DECIMAL(15, 2) NOT NULL,
    form_status VARCHAR(50) DEFAULT 'generated',  -- generated, printed, efiled, delivered
    form_document_id UUID,  -- reference to MongoDB
    irs_efiled_at TIMESTAMP,
    vendor_submission_id VARCHAR(255),  -- TurboTax, etc.
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(seller_id, tax_year)
);

CREATE INDEX idx_1099_tax_year ON tax_1099_tracking(tax_year);
CREATE INDEX idx_1099_status ON tax_1099_tracking(form_status);

-- Seller Commission History (tier changes)
CREATE TABLE seller_commission_history (
    id UUID PRIMARY KEY,
    seller_id UUID NOT NULL REFERENCES sellers(id),
    tier VARCHAR(50) NOT NULL,
    commission_rate DECIMAL(5, 4) NOT NULL,
    effective_date DATE NOT NULL,
    ended_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_commission_seller_date ON seller_commission_history(seller_id, effective_date);

-- Audit Log (all payout operations)
CREATE TABLE payout_audit_log (
    id UUID PRIMARY KEY,
    entity_type VARCHAR(100),  -- payout, reserve, chargeback, 1099
    entity_id UUID,
    action VARCHAR(100),
    details JSONB,
    user_id VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_entity ON payout_audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created_at ON payout_audit_log(created_at);
```

### MongoDB Schema

```json
{
  "_id": "ObjectId",
  "sellerId": "UUID",
  "taxYear": 2023,
  "formType": "1099-NEC",
  "recipientName": "John Seller",
  "recipientTin": "XX-XXXXXXX",
  "nonEmployeeCompensation": 50000,
  "formJson": { ... },
  "pdfContent": "BinData",
  "efiledAt": "2024-01-31T10:00:00Z",
  "irsBatchId": "batch_123456",
  "createdAt": "2024-01-15T08:00:00Z"
}
```

---

## Redis Data Structures

```
# Seller Balance Cache (for quick lookup)
seller_balance:{sellerId}
  ├─ available: 10500.00
  ├─ reserved: 500.00
  ├─ chargebacks_pending: 150.00
  └─ expires: 3600 (1 hour)

# Payout Status (in-progress batch)
payout_batch:week:{startDate}
  ├─ seller_count: 500
  ├─ completed_count: 350
  ├─ failed_count: 5
  ├─ total_amount: 500000.00
  └─ expires: 86400 (24 hours)

# Weekly Settlement Lock (prevent concurrent calculations)
settlement:lock:{week_start_date}
  └─ value: lock_timestamp (expires after 600 seconds)

# Commission Rate Cache (by tier and date)
commission_rate:{tier}:{date}
  ├─ rate: 0.15
  └─ expires: 604800 (1 week)
```

---

## Kafka Event Flow

```
Topic: payout-events
Partition Key: sellerId

Event 1: OrderCompleted
{
  "eventId": "uuid",
  "orderId": "uuid",
  "sellerId": "uuid",
  "orderAmount": 1000.00,
  "completedAt": "2024-01-08T10:00:00Z"
}
Consumers:
  - SettlementService (add CREDIT to ledger; calculate commission; add DEBIT)
  - ReportingService (aggregate for seller dashboard)

Event 2: PayoutCreated
{
  "eventId": "uuid",
  "payoutId": "uuid",
  "sellerId": "uuid",
  "payoutWeekStartDate": "2024-01-08",
  "payoutWeekEndDate": "2024-01-14",
  "payoutAmount": 8500.00,
  "reserveHeld": 500.00,
  "netPayoutAmount": 8000.00,
  "timestamp": "2024-01-15T06:00:00Z"
}
Consumers:
  - PayoutGatewayRouter (route to ACH/PayPal/Stripe)
  - ReportingService (seller dashboard update)
  - AccountingService (record payable transaction)

Event 3: PayoutProcessed
{
  "eventId": "uuid",
  "payoutId": "uuid",
  "sellerId": "uuid",
  "payoutAmount": 8000.00,
  "gatewayPayoutId": "payout_ach_123456",
  "processedAt": "2024-01-15T06:15:00Z"
}
Consumers:
  - ReportingService (mark payout complete)
  - ReconciliationService (match against gateway settlement)

Event 4: PayoutFailed
{
  "eventId": "uuid",
  "payoutId": "uuid",
  "sellerId": "uuid",
  "failureReason": "Bank account invalid",
  "retryCount": 1,
  "nextRetryTime": "2024-01-16T06:00:00Z",
  "timestamp": "2024-01-15T06:20:00Z"
}
Consumers:
  - PayoutRetryService (schedule next attempt)
  - AlertService (notify if max retries exceeded)

Event 5: ChargebackReceived
{
  "eventId": "uuid",
  "chargebackId": "uuid",
  "sellerId": "uuid",
  "orderId": "uuid",
  "chargebackAmount": 150.00,
  "chargebackDate": "2024-01-12",
  "timestamp": "2024-01-13T10:00:00Z"
}
Consumers:
  - PayoutCalculationService (add to ledger as DEBIT in next payout)
  - ChargebackService (track for dispute)
  - ReportingService (update chargeback metrics)

Event 6: RefundIssued
{
  "eventId": "uuid",
  "refundId": "uuid",
  "orderId": "uuid",
  "sellerId": "uuid",
  "refundAmount": 100.00,
  "issuedAt": "2024-01-10T14:00:00Z"
}
Consumers:
  - SettlementService (add DEBIT to ledger for clawback)
  - PayoutCalculationService (include in next week's payout calc)

Event 7: ReserveHoldReleased
{
  "eventId": "uuid",
  "reserveHoldId": "uuid",
  "sellerId": "uuid",
  "releasedAmount": 500.00,
  "timestamp": "2024-04-15T06:00:00Z"
}
Consumers:
  - PayoutCalculationService (add as CREDIT for next payout)
  - ReportingService (seller cash flow update)
```

---

## Implementation Code

### PayoutCalculationService

```java
package com.marketplace.service.payout;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.kafka.core.KafkaTemplate;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;
import java.util.*;
import java.util.stream.Collectors;

@Slf4j
@Service
public class PayoutCalculationService {

    private final PayoutLedgerRepository ledgerRepository;
    private final PayoutRecordRepository payoutRepository;
    private final SellerRepository sellerRepository;
    private final SellerCommissionHistoryRepository commissionHistoryRepository;
    private final ReserveHoldRepository reserveRepository;
    private final ChargebackRepository chargebackRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    private static final BigDecimal RESERVE_PERCENTAGE = new BigDecimal("0.05");  // 5%

    @Scheduled(cron = "0 6 * * FRI")  // Every Friday at 6 AM UTC
    @Transactional
    public void calculateWeeklyPayouts() {
        log.info("Starting weekly payout calculation");

        LocalDate today = LocalDate.now();
        LocalDate weekStart = today.minus(7, ChronoUnit.DAYS);

        // Acquire settlement lock
        String lockKey = "settlement:lock:" + weekStart;
        Boolean lockAcquired = acquireLock(lockKey);
        if (!lockAcquired) {
            log.warn("Could not acquire settlement lock for week {}", weekStart);
            return;
        }

        try {
            // Get all active sellers
            List<Seller> sellers = sellerRepository.findAll();
            int pageSize = 100;
            int totalPages = (sellers.size() + pageSize - 1) / pageSize;

            for (int page = 0; page < totalPages; page++) {
                List<Seller> pageOfSellers = sellers.stream()
                    .skip((long) page * pageSize)
                    .limit(pageSize)
                    .collect(Collectors.toList());

                for (Seller seller : pageOfSellers) {
                    try {
                        calculateSellerPayout(seller, weekStart, today);
                    } catch (Exception e) {
                        log.error("Error calculating payout for seller {}", seller.getId(), e);
                        // Continue to next seller; this one will be flagged for manual review
                    }
                }
            }

            log.info("Weekly payout calculation completed");

        } finally {
            releaseLock(lockKey);
        }
    }

    @Transactional
    private void calculateSellerPayout(Seller seller, LocalDate weekStart, LocalDate weekEnd) {
        UUID sellerId = seller.getId();

        // Query ledger entries for this seller for this week
        List<PayoutLedgerEntry> ledgerEntries = ledgerRepository
            .findBySellerIdAndCreatedAtBetween(sellerId, weekStart.atStartOfDay(), weekEnd.atTime(23, 59, 59));

        if (ledgerEntries.isEmpty()) {
            log.info("No ledger entries for seller {} for week {}", sellerId, weekStart);
            return;  // No payout this week
        }

        // Calculate credits and debits
        BigDecimal totalCredits = ledgerEntries.stream()
            .filter(e -> "CREDIT".equals(e.getEntryType()))
            .map(PayoutLedgerEntry::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal totalDebits = ledgerEntries.stream()
            .filter(e -> "DEBIT".equals(e.getEntryType()))
            .map(PayoutLedgerEntry::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal payoutAmount = totalCredits.subtract(totalDebits);

        if (payoutAmount.compareTo(BigDecimal.ZERO) <= 0) {
            log.info("Seller {} has non-positive payout amount: {}", sellerId, payoutAmount);
            return;  // No payout if amount <= 0
        }

        // Calculate reserve hold (5%)
        BigDecimal reserveHeld = payoutAmount.multiply(RESERVE_PERCENTAGE)
            .setScale(2, RoundingMode.HALF_UP);
        BigDecimal netPayoutAmount = payoutAmount.subtract(reserveHeld);

        // Get seller's payout method
        SellerPayoutMethod payoutMethod = seller.getPayoutMethod();
        if (payoutMethod == null) {
            log.warn("Seller {} has no payout method configured", sellerId);
            return;  // Skip until method configured
        }

        // Create payout record
        PayoutRecord payoutRecord = new PayoutRecord();
        payoutRecord.setId(UUID.randomUUID());
        payoutRecord.setSellerId(sellerId);
        payoutRecord.setPayoutWeekStartDate(weekStart);
        payoutRecord.setPayoutWeekEndDate(weekEnd);
        payoutRecord.setTotalCredits(totalCredits);
        payoutRecord.setTotalDebits(totalDebits);
        payoutRecord.setPayoutAmount(payoutAmount);
        payoutRecord.setReserveHeld(reserveHeld);
        payoutRecord.setNetPayoutAmount(netPayoutAmount);
        payoutRecord.setPayoutMethod(payoutMethod.getPayoutMethod());
        payoutRecord.setStatus("pending");

        payoutRepository.save(payoutRecord);

        // Create reserve hold record
        ReserveHold reserve = new ReserveHold();
        reserve.setId(UUID.randomUUID());
        reserve.setPayoutRecordId(payoutRecord.getId());
        reserve.setSellerId(sellerId);
        reserve.setHoldAmount(reserveHeld);
        reserve.setHoldReason("dispute_reserve");
        reserve.setHoldUntilDate(weekEnd.plusDays(90));  // 90-day hold
        reserve.setReleaseStatus("pending");

        reserveRepository.save(reserve);

        // Emit event
        PayoutCreatedEvent event = new PayoutCreatedEvent(
            UUID.randomUUID(),
            payoutRecord.getId(),
            sellerId,
            weekStart,
            weekEnd,
            payoutAmount,
            reserveHeld,
            netPayoutAmount,
            Instant.now()
        );
        kafkaTemplate.send("payout-events", sellerId.toString(), event);

        log.info("Payout calculated for seller {}: amount={}, reserve={}, net={}",
                 sellerId, payoutAmount, reserveHeld, netPayoutAmount);
    }

    private boolean acquireLock(String lockKey) {
        return (Boolean) redisTemplate.execute(conn -> {
            byte[] keyBytes = lockKey.getBytes();
            return conn.setNX(keyBytes, String.valueOf(System.currentTimeMillis()).getBytes(), 600);
        });
    }

    private void releaseLock(String lockKey) {
        redisTemplate.delete(lockKey);
    }
}

record PayoutCreatedEvent(
    UUID eventId,
    UUID payoutId,
    UUID sellerId,
    LocalDate weekStart,
    LocalDate weekEnd,
    BigDecimal payoutAmount,
    BigDecimal reserveHeld,
    BigDecimal netPayoutAmount,
    Instant timestamp
) { }
```

### PayoutGatewayRouter

```java
package com.marketplace.service.payout;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import lombok.extern.slf4j.Slf4j;
import java.util.UUID;

@Slf4j
@Service
public class PayoutGatewayRouter {

    private final PayoutRecordRepository payoutRepository;
    private final SellerPayoutMethodRepository methodRepository;
    private final ACHPayoutGateway achGateway;
    private final PayPalPayoutGateway paypalGateway;
    private final StripeConnectPayoutGateway stripeGateway;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "payout-events", groupId = "payout-gateway-router")
    public void onPayoutCreated(PayoutCreatedEvent event) {
        log.info("Routing payout {} to gateway", event.payoutId());

        PayoutRecord payout = payoutRepository.findById(event.payoutId())
            .orElseThrow();

        SellerPayoutMethod method = methodRepository.findBySellerId(payout.getSellerId())
            .orElseThrow(() -> new IllegalStateException("Payout method not found"));

        try {
            PayoutResponse response = switch (method.getPayoutMethod()) {
                case "ach" -> achGateway.pay(buildPayoutRequest(payout, method));
                case "paypal" -> paypalGateway.pay(buildPayoutRequest(payout, method));
                case "stripe" -> stripeGateway.pay(buildPayoutRequest(payout, method));
                default -> throw new UnsupportedOperationException(
                    "Unsupported payout method: " + method.getPayoutMethod()
                );
            };

            handlePayoutResponse(payout, response);

        } catch (Exception e) {
            log.error("Error processing payout {} to {}", payout.getId(), method.getPayoutMethod(), e);
            handlePayoutFailure(payout, e.getMessage());
        }
    }

    @Transactional
    private void handlePayoutResponse(PayoutRecord payout, PayoutResponse response) {
        if ("success".equals(response.getStatus())) {
            payout.setStatus("completed");
            payout.setGatewayPayoutId(response.getPayoutId());
            payout.setProcessedAt(Instant.now());
            payoutRepository.save(payout);

            // Emit success event
            PayoutProcessedEvent event = new PayoutProcessedEvent(
                UUID.randomUUID(),
                payout.getId(),
                payout.getSellerId(),
                payout.getNetPayoutAmount(),
                response.getPayoutId(),
                Instant.now()
            );
            kafkaTemplate.send("payout-events", payout.getSellerId().toString(), event);

            log.info("Payout {} processed successfully", payout.getId());

        } else if ("pending".equals(response.getStatus())) {
            payout.setStatus("processing");
            payout.setGatewayPayoutId(response.getPayoutId());
            payoutRepository.save(payout);

            log.info("Payout {} in processing state", payout.getId());

        } else {
            handlePayoutFailure(payout, response.getErrorMessage());
        }
    }

    @Transactional
    private void handlePayoutFailure(PayoutRecord payout, String failureReason) {
        payout.setStatus("failed");
        payout.setFailureReason(failureReason);
        payout.setRetryCount(payout.getRetryCount() + 1);

        Instant nextRetryTime = calculateNextRetryTime(payout.getRetryCount());
        payout.setNextRetryTime(nextRetryTime);

        payoutRepository.save(payout);

        // Emit failure event
        PayoutFailedEvent event = new PayoutFailedEvent(
            UUID.randomUUID(),
            payout.getId(),
            payout.getSellerId(),
            failureReason,
            payout.getRetryCount(),
            nextRetryTime,
            Instant.now()
        );
        kafkaTemplate.send("payout-events", payout.getSellerId().toString(), event);

        log.warn("Payout {} failed: {}. Retry {} at {}", payout.getId(), failureReason,
                 payout.getRetryCount(), nextRetryTime);
    }

    private PayoutRequest buildPayoutRequest(PayoutRecord payout, SellerPayoutMethod method) {
        return new PayoutRequest(
            payout.getId(),
            payout.getSellerId(),
            payout.getNetPayoutAmount(),
            method.getMethodToken(),
            method.getCurrency(),
            generateIdempotencyKey(payout.getId()),
            payout.getCreatedAt()
        );
    }

    private Instant calculateNextRetryTime(int retryCount) {
        long delayHours = (long) Math.pow(2, retryCount);  // 2, 4, 8 hours
        return Instant.now().plusSeconds(delayHours * 3600);
    }

    private String generateIdempotencyKey(UUID payoutId) {
        return org.apache.commons.codec.digest.DigestUtils.md5Hex(payoutId.toString());
    }
}

record PayoutCreatedEvent(...) { }
record PayoutProcessedEvent(...) { }
record PayoutFailedEvent(...) { }
```

### SettlementService

```java
package com.marketplace.service.payout;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.time.Instant;
import java.util.UUID;

@Slf4j
@Service
public class SettlementService {

    private final PayoutLedgerRepository ledgerRepository;
    private final SellerRepository sellerRepository;
    private final SellerCommissionHistoryRepository commissionHistoryRepository;
    private final OrderRepository orderRepository;

    @KafkaListener(topics = "order-events", groupId = "settlement-service")
    @Transactional
    public void onOrderCompleted(OrderCompletedEvent event) {
        log.info("Recording order {} for settlement", event.orderId());

        Order order = orderRepository.findById(event.orderId())
            .orElseThrow();

        UUID sellerId = order.getSellerId();
        BigDecimal orderAmount = order.getTotalAmount();

        // Get commission rate for seller's current tier
        Seller seller = sellerRepository.findById(sellerId)
            .orElseThrow();

        SellerCommissionHistory commissionHistory = commissionHistoryRepository
            .findCurrentCommissionRateForSeller(sellerId, event.completedAt().toLocalDateTime().toLocalDate())
            .orElseThrow(() -> new IllegalStateException("No commission rate found for seller"));

        BigDecimal commissionRate = commissionHistory.getCommissionRate();
        BigDecimal commissionAmount = orderAmount.multiply(commissionRate)
            .setScale(2, java.math.RoundingMode.HALF_UP);

        // Create CREDIT entry for seller
        PayoutLedgerEntry creditEntry = new PayoutLedgerEntry();
        creditEntry.setId(UUID.randomUUID());
        creditEntry.setSellerId(sellerId);
        creditEntry.setEntryType("CREDIT");
        creditEntry.setAmount(orderAmount);
        creditEntry.setCurrency(order.getCurrency());
        creditEntry.setRelatedOrderId(event.orderId());
        creditEntry.setLedgerEntryReason("order_completion");
        creditEntry.setCreatedAt(Instant.now());

        ledgerRepository.save(creditEntry);

        // Create DEBIT entry for commission
        PayoutLedgerEntry debitEntry = new PayoutLedgerEntry();
        debitEntry.setId(UUID.randomUUID());
        debitEntry.setSellerId(sellerId);
        debitEntry.setEntryType("DEBIT");
        debitEntry.setAmount(commissionAmount);
        debitEntry.setCurrency(order.getCurrency());
        debitEntry.setRelatedOrderId(event.orderId());
        debitEntry.setLedgerEntryReason("platform_commission");
        debitEntry.setCreatedAt(Instant.now());

        ledgerRepository.save(debitEntry);

        log.info("Order {} recorded in payout ledger: credit={}, commission={}",
                 event.orderId(), orderAmount, commissionAmount);
    }

    @KafkaListener(topics = "refund-events", groupId = "settlement-service")
    @Transactional
    public void onRefundIssued(RefundIssuedEvent event) {
        log.info("Recording refund {} for settlement clawback", event.refundId());

        Order order = orderRepository.findById(event.orderId())
            .orElseThrow();

        UUID sellerId = order.getSellerId();
        BigDecimal refundAmount = event.refundAmount();

        // Get seller's commission rate at time of original order
        SellerCommissionHistory commissionHistory = commissionHistoryRepository
            .findCommissionRateAtDate(sellerId, order.getCreatedAt().toLocalDateTime().toLocalDate())
            .orElseThrow();

        BigDecimal commissionRate = commissionHistory.getCommissionRate();

        // Seller's share of refund = refund_amount * (1 - commission_rate)
        BigDecimal sellerRefundShare = refundAmount.multiply(BigDecimal.ONE.subtract(commissionRate))
            .setScale(2, java.math.RoundingMode.HALF_UP);

        // Create DEBIT entry for refund clawback
        PayoutLedgerEntry clawbackEntry = new PayoutLedgerEntry();
        clawbackEntry.setId(UUID.randomUUID());
        clawbackEntry.setSellerId(sellerId);
        clawbackEntry.setEntryType("DEBIT");
        clawbackEntry.setAmount(sellerRefundShare);
        clawbackEntry.setCurrency(order.getCurrency());
        clawbackEntry.setRelatedOrderId(event.orderId());
        clawbackEntry.setLedgerEntryReason("refund_clawback");
        clawbackEntry.setCreatedAt(event.issuedAt());

        ledgerRepository.save(clawbackEntry);

        log.info("Refund {} recorded as clawback in ledger: amount={}", event.refundId(), sellerRefundShare);
    }

    @KafkaListener(topics = "chargeback-events", groupId = "settlement-service")
    @Transactional
    public void onChargebackReceived(ChargebackReceivedEvent event) {
        log.info("Recording chargeback {} for settlement deduction", event.chargebackId());

        UUID sellerId = event.sellerId();
        BigDecimal chargebackAmount = event.chargebackAmount();

        // Create DEBIT entry for chargeback
        PayoutLedgerEntry chargebackEntry = new PayoutLedgerEntry();
        chargebackEntry.setId(UUID.randomUUID());
        chargebackEntry.setSellerId(sellerId);
        chargebackEntry.setEntryType("DEBIT");
        chargebackEntry.setAmount(chargebackAmount);
        chargebackEntry.setCurrency("USD");  // TODO: get from chargeback
        chargebackEntry.setRelatedOrderId(event.orderId());
        chargebackEntry.setLedgerEntryReason("chargeback_deduction");
        chargebackEntry.setCreatedAt(event.timestamp());

        ledgerRepository.save(chargebackEntry);

        log.info("Chargeback {} recorded in ledger: amount={}", event.chargebackId(), chargebackAmount);
    }
}

record OrderCompletedEvent(UUID orderId, UUID sellerId, BigDecimal amount, Instant completedAt) { }
record RefundIssuedEvent(UUID refundId, UUID orderId, UUID sellerId, BigDecimal refundAmount, Instant issuedAt) { }
record ChargebackReceivedEvent(UUID chargebackId, UUID orderId, UUID sellerId, BigDecimal chargebackAmount, Instant timestamp) { }
```

### TaxReportingService

```java
package com.marketplace.service.payout;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.data.mongodb.core.MongoTemplate;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.YearMonth;
import java.util.*;
import java.util.stream.Collectors;

@Slf4j
@Service
public class TaxReportingService {

    private final PayoutRecordRepository payoutRepository;
    private final SellerRepository sellerRepository;
    private final Tax1099TrackingRepository tax1099Repository;
    private final MongoTemplate mongoTemplate;
    private final IrsEFileService irsEFileService;

    private static final BigDecimal TAX_THRESHOLD = new BigDecimal("600");

    @Scheduled(cron = "0 0 1 1 *")  // January 1st at midnight
    @Transactional
    public void generateAnnual1099Forms() {
        log.info("Starting annual 1099 form generation");

        int currentYear = YearMonth.now().getYear() - 1;  // Previous calendar year

        // Get all sellers with earnings > $600 in the previous year
        List<Seller> eligibleSellers = getEligibleSellersForTaxReporting(currentYear);

        log.info("Generating 1099 forms for {} sellers", eligibleSellers.size());

        for (Seller seller : eligibleSellers) {
            try {
                generateForm1099NEC(seller, currentYear);
            } catch (Exception e) {
                log.error("Error generating 1099 for seller {}", seller.getId(), e);
                // Continue to next seller; failed ones logged for manual review
            }
        }

        // Submit to IRS e-file service
        try {
            submitToIRS(currentYear);
        } catch (Exception e) {
            log.error("Error submitting forms to IRS", e);
            // Alert tax compliance team
        }

        log.info("Annual 1099 form generation completed");
    }

    @Transactional
    private void generateForm1099NEC(Seller seller, int taxYear) throws Exception {
        UUID sellerId = seller.getId();

        // Calculate annual earnings
        BigDecimal annualEarnings = calculateAnnualEarnings(sellerId, taxYear);

        if (annualEarnings.compareTo(TAX_THRESHOLD) < 0) {
            log.info("Seller {} earnings below threshold: {}", sellerId, annualEarnings);
            return;
        }

        // Build 1099-NEC form
        Form1099NEC form = new Form1099NEC();
        form.setRecipientId(sellerId);
        form.setRecipientName(seller.getLegalName());
        form.setRecipientTin(seller.getTaxId());  // SSN or EIN
        form.setPayerName("Platform Inc");
        form.setPayerTin("XX-XXXXXXX");
        form.setNonEmployeeCompensation(annualEarnings);  // Box 1
        form.setTaxYear(taxYear);
        form.setFormNumber("1099-NEC");
        form.setGeneratedDate(LocalDate.now());

        // Generate PDF
        byte[] pdfContent = generateForm1099Pdf(form);

        // Store in MongoDB
        Tax1099Document document = new Tax1099Document();
        document.setId(UUID.randomUUID());
        document.setSellerId(sellerId.toString());
        document.setTaxYear(taxYear);
        document.setFormType("1099-NEC");
        document.setFormJson(form.toJson());
        document.setPdfContent(pdfContent);
        document.setCreatedAt(Instant.now());

        mongoTemplate.save(document);

        // Track in PostgreSQL
        Tax1099Tracking tracking = new Tax1099Tracking();
        tracking.setId(UUID.randomUUID());
        tracking.setSellerId(sellerId);
        tracking.setTaxYear(taxYear);
        tracking.setAnnualEarnings(annualEarnings);
        tracking.setFormStatus("generated");

        tax1099Repository.save(tracking);

        // Queue for IRS e-file
        irsEFileService.addToQueue(document);

        log.info("Generated 1099-NEC for seller {}: earnings={}", sellerId, annualEarnings);
    }

    private BigDecimal calculateAnnualEarnings(UUID sellerId, int taxYear) {
        return payoutRepository.findBySellerIdAndYearAndStatus(sellerId, taxYear, "completed")
            .stream()
            .map(PayoutRecord::getNetPayoutAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    private List<Seller> getEligibleSellersForTaxReporting(int taxYear) {
        return sellerRepository.findAll().stream()
            .filter(seller -> {
                BigDecimal earnings = calculateAnnualEarnings(seller.getId(), taxYear);
                return earnings.compareTo(TAX_THRESHOLD) >= 0;
            })
            .filter(seller -> seller.getTaxId() != null && !seller.getTaxId().isEmpty())
            .collect(Collectors.toList());
    }

    private byte[] generateForm1099Pdf(Form1099NEC form) throws Exception {
        // Generate PDF using iTextPDF
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        PdfWriter writer = new PdfWriter(baos);
        Document document = new Document(new PdfDocument(writer));

        // Add form fields
        document.add(new Paragraph("FORM 1099-NEC").setBold().setFontSize(14));
        document.add(new Paragraph("Miscellaneous Income").setFontSize(10));
        document.add(new Paragraph("Copy for Recipient"));

        // Add boxes
        document.add(new Paragraph("\nBox 1 - Nonemployee Compensation: $" + form.getNonEmployeeCompensation()));
        document.add(new Paragraph("Recipient Name: " + form.getRecipientName()));
        document.add(new Paragraph("Recipient TIN: " + maskTin(form.getRecipientTin())));
        document.add(new Paragraph("Tax Year: " + form.getTaxYear()));

        // Add compliance statement
        document.add(new Paragraph("\nThis is an important tax document. Keep it for your records.")
            .setItalic().setFontSize(9));

        document.close();
        return baos.toByteArray();
    }

    private void submitToIRS(int taxYear) throws Exception {
        List<Tax1099Tracking> pendingForms = tax1099Repository
            .findByTaxYearAndFormStatus(taxYear, "generated");

        log.info("Submitting {} pending 1099 forms to IRS for tax year {}", pendingForms.size(), taxYear);

        for (Tax1099Tracking tracking : pendingForms) {
            try {
                String batchId = irsEFileService.submitBatch(taxYear);
                tracking.setFormStatus("efiled");
                tracking.setIrsEfiledAt(Instant.now());
                tracking.setVendorSubmissionId(batchId);
                tax1099Repository.save(tracking);

                log.info("1099 for seller {} submitted to IRS with batch ID: {}", tracking.getSellerId(), batchId);
            } catch (Exception e) {
                log.error("Failed to submit 1099 for seller {} to IRS", tracking.getSellerId(), e);
            }
        }
    }

    private String maskTin(String tin) {
        if (tin == null || tin.length() < 4) {
            return "XX-XXXXXX";
        }
        return "XX-XXX" + tin.substring(tin.length() - 4);
    }
}

@Data
class Tax1099Document {
    private UUID id;
    private String sellerId;
    private int taxYear;
    private String formType;
    private Map<String, Object> formJson;
    private byte[] pdfContent;
    private Instant createdAt;
}
```

---

## Failure Scenarios

| Scenario | Handling |
|----------|----------|
| Bank account invalid / ACH rejection | Retry 3x (T+1, T+3, T+7); on final failure → hold for manual review; notify seller to update account |
| PayPal/Stripe API timeout | Retry within 5 minutes; if still down, mark payout as "pending" for next day's retry |
| Partial payout (gateway accepted $8K of $9K) | Update payout record with actual amount; create adjustment entry for diff; retry remaining amount |
| Seller dispute within 90-day reserve window | Hold released funds; clawback from next payout; track dispute in chargeback table |
| Chargebacks after payout issued | Create negative ledger entry; clawed back from next week's payout; seller balance can go negative |
| Refund issued after payout completed | Add DEBIT entry to payout ledger for following week; recalculate next week's payout |
| 1099 generation fails (invalid TIN) | Alert seller to verify tax ID in account settings; forms not generated until resolved |
| IRS e-file submission fails | Mark forms as "efile_failed"; retry next day; alert compliance team if 3 consecutive failures |
| Duplicate payout (same seller, same week) | Idempotency check: if payout exists for seller + week, skip generation (idempotent) |

---

## Scaling Strategy

### Horizontal Scaling

| Component | Scaling Strategy |
|-----------|------------------|
| SettlementService | Kafka consumer group; scale with order-events partition count (20+) |
| PayoutCalculationService | Single-threaded weekly job; no horizontal scaling needed |
| PayoutGatewayRouter | Kafka consumer group; scale per gateway (ACH: 5 instances, PayPal: 10) |
| Tax Reporting Service | Batch job; run on dedicated instance; parallelized by seller range (sharding) |
| Retry Service | Kafka consumer group; scale based on failed payout volume |

### Database Optimization

- **Partitioning:** payout_ledger by seller_id; payout_records by payout_week_start_date (weekly)
- **Indexing:** (seller_id, created_at), (status, next_retry_time), (payout_week_start_date)
- **Archive:** ledger entries older than 2 years → cold storage (Glacier); keep 2 years hot for tax audits
- **Read Replicas:** dedicated replica for reporting service (avoid contention with settlement writes)

### Kafka Tuning

- **Partitions:** 20+ for order-events; 50+ for payout-events (per gateway type)
- **Consumer Groups:** separate for settlement, payout routing, tax reporting, accounting
- **Retention:** 90 days (allow disputes + chargebacks to surface)
- **Batch Size:** 50 payouts/batch for throughput

---

## Monitoring

### Key Metrics

| Metric | Target | Tool |
|--------|--------|------|
| Payout calculation time | <10 seconds | Prometheus timer |
| Payout success rate | >99% | Prometheus counter |
| Average payout amount | $X (baseline) | Prometheus gauge |
| Ledger entry latency | <200ms | Prometheus histogram |
| Failed payout retry success rate | >80% | Prometheus counter |
| Reserve hold release % | >95% | Prometheus gauge |
| 1099 generation time | <5 minutes per 1K sellers | Prometheus timer |

### Alerts

```yaml
- alert: PayoutCalculationFailure
  expr: job_failures_total{job="payoutCalculation"} > 0
  for: 1m
  action: page on-call

- alert: PayoutSuccessRateLow
  expr: rate(payout_success_total[1h]) / rate(payout_total[1h]) < 0.99
  for: 10m
  action: notify payments-team

- alert: RetryFailedPayoutsStaging
  expr: payout_records{status="hold_for_review"} > 100
  for: 1h
  action: notify finance + ops

- alert: TaxReportingJobFailed
  expr: tax_reporting_job_failures_total > 0
  for: 1m
  action: page compliance-team

- alert: ChargebackRate High
  expr: rate(chargebacks_received_total[24h]) > 0.005
  for: 6h
  action: notify fraud + customer-service
```

### Dashboards

- **Payout Dashboard:** weekly payout volume, success rate, avg payout amount, failures breakdown
- **Finance Dashboard:** total seller payouts, commission revenue, reserves held, chargebacks
- **Tax Dashboard:** 1099 progress, forms generated, IRS submissions, pending verification
- **Operational Dashboard:** job execution times, Kafka lag, error rates by gateway

---

## Summary Cheat Sheet

```
PAYOUT CALCULATION:
  - Immutable append-only payout_ledger: CREDIT (order) + DEBIT (commission, refund, chargeback)
  - Weekly batch: aggregate ledger per seller → calculate net payout
  - Commission tiers: standard 15%, premium 10%, enterprise custom; applied at order completion
  - Payout = credits - debits; reserve 5%; net = payout - reserve

REFUND HANDLING:
  - Refund issued after payout: add DEBIT entry to ledger for following week
  - Clawback amount = refund_amount × (1 - commission_rate) = customer share refunded to seller
  - Post-payout refund deducted from NEXT week's payout (clawback mechanism)

CHARGEBACK HANDLING:
  - Chargeback received: add DEBIT entry to payout ledger
  - Deducted from next available payout
  - Dispute tracking: evidence stored in MongoDB; dispute outcome recorded

PAYOUT METHODS:
  - Adapter pattern: ACHPayoutGateway, PayPalPayoutGateway, StripeConnectPayoutGateway
  - Routing: seller.payoutMethod + country → select gateway
  - Idempotency: every payout call includes idempotencyKey = hash(payoutRecordId + seller + amount)
  - Bank details tokenized via Stripe/Plaid (no raw account numbers stored)

PAYOUT RETRY:
  - Attempt 1: T+1 (1 hour backoff)
  - Attempt 2: T+3 (2 hour backoff)
  - Attempt 3: T+7 (4 hour backoff)
  - After 3: mark "hold_for_review" → manual intervention
  - Exponential backoff: delay = 2^retryCount hours

RESERVE HOLDS:
  - 5% of each payout held in escrow for 90 days
  - Released automatically after 90 days if no chargebacks
  - If chargeback filed within 90 days: hold disputed amount indefinitely
  - Released as CREDIT entry in following payout when dispute resolved

TAX REPORTING:
  - 1099-NEC generated for sellers earning >$600 in calendar year
  - Annual calculation: Jan 1 at midnight → aggregate all payouts from previous year
  - Form stored as PDF + JSON in MongoDB; tracked in PostgreSQL
  - Multi-TIN support: if seller has LLC + personal → generate separate 1099 for each
  - IRS e-file submission via TurboTax/e-services provider

COMMISSION TIERS:
  - Seller.tier changes based on monthly GMV
  - Tier changed effective next month (not retroactive)
  - History tracked in seller_commission_history
  - Commission rate applied at order completion time; effective rate captured in ledger
  - Examples: <$100K/mo = 15%, $100K-$1M = 10%, >$1M = 5%

KEY TABLES:
  - payout_ledger: immutable entries (CREDIT/DEBIT); queryable per seller + date range
  - payout_records: weekly settlement batches; tracks status (pending → completed/failed)
  - reserve_holds: 5% escrow per payout; 90-day hold
  - chargebacks: dispute tracking; evidence links
  - tax_1099_tracking: annual tax form status; IRS submission tracking

KAFKA TOPICS:
  - payout-events: key=sellerId; events: PayoutCreated, PayoutProcessed, PayoutFailed, ReserveReleased
  - order-events → SettlementService: records ledger entries
  - refund-events → SettlementService: records clawback debit entries
  - chargeback-events → SettlementService: records chargeback debit entries

REDIS KEYS:
  - seller_balance:{sellerId}: cache (1h TTL) available balance for quick lookup
  - payout_batch:week:{startDate}: in-progress batch tracking
  - settlement:lock:{weekStart}: prevent concurrent payout calculation (10 min TTL)
  - commission_rate:{tier}:{date}: commission rate cache (1 week TTL)
```

---

**Last Updated:** 2024-01-15 | **Version:** 1.0 | **Owner:** Finance & Settlements Team
