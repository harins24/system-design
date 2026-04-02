---
title: Split Payment System — Deep Dive Design
layout: default
---

# Split Payment System — Deep Dive Design

> **Scenario:** Customers split payment across multiple credit cards (50/50), gift card + credit card, loyalty points + credit card, company card + personal card. Handle failures (card A succeeds, card B declines). Support refunds to original split.
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
- Support multiple payment methods per transaction: credit cards, gift cards, loyalty points, ACH/bank transfers
- Define custom split ratio: 50/50, 30/70, gift card $50 + credit card $X, loyalty points max + credit card remainder
- Validate split before processing (e.g., loyalty points balance, gift card balance, credit limit)
- Two-phase processing: authorize all methods first, capture only on success
- If any method fails: rollback all successful authorizations (atomicity)
- Partial authorization handling: card authorized $80 of $100 requested → prompt user for delta
- Refunds: reverse to original split proportions (refund $50 split → $25 to card A, $25 to card B)
- Currency conversion: support multi-currency splits (USD + GBP, etc.)
- PCI compliance: never store raw card data; use tokenization via Stripe/PayPal

### Non-Functional Requirements
- **Scale:** 1M split transactions/month; 5% of all transactions
- **Latency:** <2 seconds end-to-end for auth + capture; <3 seconds for refund
- **Availability:** 99.95% uptime (payment critical path)
- **Consistency:** atomicity across payment methods; saga pattern for distributed transaction
- **Audit:** immutable audit trail of all splits, authorizations, captures, refunds

---

## Capacity Estimation

| Metric | Volume | Notes |
|--------|--------|-------|
| Transactions/Month | 20M | total transactions |
| Split Transactions/Month | 1M | 5% of total |
| Split Requests (avg 2.3 methods/split) | 2.3M | authorize calls |
| Refunds/Month | 100K | from split transactions |
| Storage (1 year) | ~50 GB | split records, audit trail, failed attempts |
| Redis Memory | ~200 MB | authorization holds, locks |
| Kafka throughput | 500 events/sec | auth, capture, refund events |

---

## High-Level Architecture

```
                         ┌─────────────────────────────┐
                         │  Checkout Service           │
                         │  (existing)                 │
                         │  - Select split options     │
                         └────────────────┬────────────┘
                                          │
                         ┌────────────────v────────────────┐
                         │  Split Payment Orchestrator     │
                         │  - Validate split plan          │
                         │  - Route to SplitPaymentSaga    │
                         └────────────────┬────────────────┘
                                          │
        ┌─────────────────────────────────┼────────────────────────────┐
        │                                  │                            │
        v                                  v                            v
   ┌────────────┐              ┌──────────────────┐         ┌──────────────────┐
   │ Auth Phase │              │ Capture Phase    │         │ Refund Phase     │
   │ (Saga)     │              │ (Saga)           │         │ (If needed)      │
   │            │              │                  │         │                  │
   │ 1. Auth    │ ─success──> 2. Capture each   │ ─────> 3. Reverse each    │
   │    each    │              │   method        │         │   method        │
   │    method  │ ─failure──>  │                 │         │   propor-       │
   │ (MULTI)    │ (rollback)   └──────────────────┘         │   tionally      │
   └────────────┘                                          └──────────────────┘
        │
        v
   Payment Gateway Calls:
   ├─ Stripe Token API (auth card)
   ├─ PayPal API (auth PayPal balance)
   ├─ Internal Loyalty API (check points + redeem)
   ├─ Gift Card Service API (check balance + hold)
   └─ Bank ACH API (auth account)

   Result:
   ├─ All success → emit SplitPaymentAuthorized → proceed to capture
   ├─ Partial failure → emit SplitPaymentPartiallyAuthorized → prompt user
   └─ All failure → emit SplitPaymentAuthorizationFailed → reject order

        ┌──────────────────────────────────────────────────┐
        │  Kafka Topics                                     │
        │  - SplitPaymentAuthStarted                        │
        │  - PaymentMethodAuthorized                        │
        │  - PaymentMethodAuthFailed                        │
        │  - SplitPaymentAuthorized (all success)           │
        │  - SplitPaymentCaptured (all captured)            │
        │  - SplitPaymentRefunded                           │
        └──────────────────────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
         v                v                v
   ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐
   │ Accounting   │  │ Reconciliation│  │ Fraud Detection  │
   │ Service      │  │ Service      │  │ Service          │
   └──────────────┘  └──────────────┘  └──────────────────┘

        ┌──────────────────────────────────┐
        │  PostgreSQL                       │
        │  - SplitPaymentPlan               │
        │  - PaymentMethodAuthorization     │
        │  - SplitPaymentRefund             │
        │  - Audit Log                      │
        └──────────────────────────────────┘

        ┌──────────────────────────────────┐
        │  MongoDB                          │
        │  - High-volume audit trail        │
        │  - PII (tokenized)                │
        └──────────────────────────────────┘

        ┌──────────────────────────────────┐
        │  Redis                            │
        │  - Auth holds (temporary)         │
        │  - Locks (saga state)             │
        │  - Rate limiting                  │
        └──────────────────────────────────┘
```

---

## Core Design Questions Answered

### 1. How do you coordinate payments across multiple payment methods?

**Answer:**
- Use **Saga Pattern** (Kafka-driven choreography): each step publishes event → next service consumes and acts
- **Order:** Validate → Auth each method (MULTI phase) → Capture each method (MULTI phase) → Settle
- **SplitPaymentPlan** defines ordered list of (method, amount, priority)
  ```
  [
    { "method": "card_a", "amount": 50, "priority": 1 },
    { "method": "loyalty_points", "amount": 30, "priority": 2 },
    { "method": "gift_card", "amount": 20, "priority": 3 }
  ]
  ```
- **Orchestrator:** `SplitPaymentOrchestrator` validates plan, publishes `SplitPaymentAuthStarted` event
- **Executor:** `PaymentMethodExecutor` consumes event, calls auth API for each method in order
  - If method auth succeeds → publish `PaymentMethodAuthorized` event
  - If method auth fails → publish `PaymentMethodAuthFailed` event with reason
- **Aggregator:** wait for N responses (one per method); if all success → publish `SplitPaymentAuthorized`
- **Capture Phase:** similar flow, but uses previously authorized transaction IDs

### 2. What happens if one payment succeeds but another fails?

**Answer:**
- **Detection:** Capture phase detects failure when calling payment gateway
- **Compensating Transaction:** Reverse all previously captured methods via refund API calls
  ```
  Scenario: [Auth card_a: $50 ✓, Auth card_b: $50 FAILED]
  → Compensate: Release hold on card_a (never actually charged because auth succeeded but capture failed)
  → Reject entire transaction
  → Emit SplitPaymentAuthorizationFailed
  ```
- **Idempotency Keys:** Every auth/capture call includes idempotency key (hash of splitPaymentId + methodId + nonce) so retries are safe
- **Timeout Handling:** If capture takes >30 seconds, trigger automatic compensating transaction (void auth, refund captures)
- **Audit Trail:** MongoDB logs every auth/capture/void attempt with timestamp and gateway response code

### 3. How do you handle partial authorization (requested $100, authorized $80)?

**Answer:**
- Payment gateway returns `{ authorizedAmount: 80, requestedAmount: 100, status: "PARTIAL" }`
- Orchestrator detects shortfall: delta = $100 - $80 = $20
- **Two options:**
  1. **Auto-fallback:** if next method in plan can cover delta (e.g., gift card $50) → adjust split:
     ```
     [card_a: $80 AUTHORIZED, gift_card: $20 (instead of original $20)]
     ```
  2. **Prompt User:** emit `SplitPaymentPartialAuthorizationWaiting` event → UI shows:
     ```
     "Card ending in 1234 authorized $80 of $100.
      Remaining: $20. Pay with gift card? [Yes] [Cancel]"
     ```
- If user cancels → release all auths via compensating transaction
- If user confirms → continue with next method for delta

### 4. How do you refund split payments?

**Answer:**
- **Refund Plan:** track original split in `SplitPaymentRefund` table:
  ```
  refundId | originalPaymentId | method | originalAmount | refundAmount | status
  uuid1    | pay123            | card_a | 50             | 50           | success
  uuid1    | pay123            | gift   | 20             | 20           | success
  uuid1    | pay123            | loyalty| 30             | 30           | success
  ```
- **Partial Refund:** if customer requests refund $60 of $100:
  - Calculate proportional split: card_a (50/100 * 60 = $30), gift ($20 eligible, but only $12 needed)
  - Refund in reverse order (gift first, then card)
- **Reverse in Reverse:** refund in opposite order of original auth for maximum success (last auth usually has most available balance)
- **Idempotency:** refundId is unique; if called twice, second call returns existing refund record (idempotent)
- **Partial Refund Failure:** if card_a refund succeeds but gift refund fails → emit alert + manual review queue

### 5. How do you ensure atomic payment processing?

**Answer:**
- **Two-Phase Commit (Saga):**
  - **Phase 1 (Auth):** hold funds on all methods; if any fails → release all holds → reject
  - **Phase 2 (Capture):** actually charge all methods; if any fails → refund all captures → reject
  - **Kafka** ensures ordering per splitPaymentId (partition key)
- **Idempotent Operations:** every auth/capture/refund call tagged with unique key:
  ```java
  String idempotencyKey = md5(splitPaymentId + methodId + paymentGateway + "auth" + nonce)
  // PaymentGateway remembers key for 24 hours; duplicate calls return cached result
  ```
- **Distributed Lock:** `split_payment:lock:{splitPaymentId}` in Redis prevents concurrent modifications
  ```
  SETNX split_payment:lock:{id} {timestamp} EX 60
  // If lock fails, return 409 Conflict; client retries with backoff
  ```
- **Event Sourcing:** every state change (auth, capture, refund) appended to PostgreSQL event log
  - Allows audit trail + replay if needed

### 6. How do you handle currency conversion for international cards?

**Answer:**
- **Input Validation:** SplitPaymentValidator checks all methods in split use compatible currencies
  - Single-currency split: all cards in USD → simple
  - Multi-currency split: card_a in USD ($100), card_b in EUR (€80)
    - Get FX rate: 1 USD = 0.92 EUR (pulled from OpenExchangeRates API, cached in Redis)
    - Convert to common currency: EUR_total = 100 * 0.92 + 80 = 172 EUR
    - Validate total >= requested amount (accounting for FX volatility)
- **Capture-Time FX:** FX rates change between auth and capture
  - PaymentGateway (Stripe/Adyen) handles FX conversion at capture time
  - If actual charged amount differs from authorized → emit FX_VARIANCE event
  - If variance > 3%: reject and prompt user to re-auth at new rate
- **Settled Currency:** all ledger entries stored in customer's home currency (via FX rate snapshot at auth time)
- **Refund Currency:** refund issued in original method's currency (Stripe refunds to card in card's currency)

---

## Microservices Breakdown

| Microservice | Port | Responsibilities | Key Dependencies |
|--------------|------|------------------|------------------|
| Split Payment Orchestrator | 8007 | Validate split plan; coordinate saga | PostgreSQL, Redis, Kafka |
| Payment Method Executor | 8008 | Auth/capture individual methods | Payment gateway APIs |
| Refund Service | 8009 | Process refunds proportionally; handle failures | Payment gateway APIs, PostgreSQL |
| Fraud Detection Service | 8010 | Check for suspicious split patterns | ML model, PostgreSQL |
| Reconciliation Service | 8011 | Match settled amounts vs. DB records | PostgreSQL, Payment gateway APIs |

---

## Database Design (DDL)

### PostgreSQL Schema

```sql
-- Split Payment Plans (planned splits)
CREATE TABLE split_payment_plans (
    id UUID PRIMARY KEY,
    order_id UUID NOT NULL UNIQUE,
    customer_id UUID NOT NULL,
    total_amount DECIMAL(15, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL,  -- USD, EUR, GBP
    split_method VARCHAR(50) NOT NULL,  -- manual, preset, loyaltyFirst
    status VARCHAR(50) DEFAULT 'pending',  -- pending, authorized, captured, failed
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_split_plans_order ON split_payment_plans(order_id);
CREATE INDEX idx_split_plans_customer_status ON split_payment_plans(customer_id, status);

-- Split Payment Methods (individual payment methods in plan)
CREATE TABLE split_payment_methods (
    id UUID PRIMARY KEY,
    split_plan_id UUID NOT NULL REFERENCES split_payment_plans(id) ON DELETE CASCADE,
    method_type VARCHAR(50) NOT NULL,  -- credit_card, gift_card, loyalty_points, bank_transfer
    method_token VARCHAR(255) NOT NULL,  -- tokenized (no raw card data)
    requested_amount DECIMAL(15, 2) NOT NULL,
    authorized_amount DECIMAL(15, 2),
    captured_amount DECIMAL(15, 2),
    currency VARCHAR(3),
    priority INT NOT NULL,  -- order of execution (1, 2, 3, ...)
    auth_transaction_id VARCHAR(255),  -- from payment gateway
    auth_status VARCHAR(50),  -- pending, authorized, failed
    capture_transaction_id VARCHAR(255),
    capture_status VARCHAR(50),  -- pending, captured, failed, voided
    fx_rate DECIMAL(18, 6),  -- FX rate at auth time (if cross-currency)
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_split_methods_plan ON split_payment_methods(split_plan_id);
CREATE INDEX idx_split_methods_auth_status ON split_payment_methods(auth_status);

-- Payment Method Authorizations (detailed auth attempt records)
CREATE TABLE payment_method_authorizations (
    id UUID PRIMARY KEY,
    split_method_id UUID NOT NULL REFERENCES split_payment_methods(id),
    attempt_number INT NOT NULL,  -- retry attempt
    gateway_name VARCHAR(50),  -- stripe, paypal, adyen
    gateway_response_code VARCHAR(10),
    gateway_response_message TEXT,
    requested_amount DECIMAL(15, 2) NOT NULL,
    authorized_amount DECIMAL(15, 2),
    authorization_status VARCHAR(50),  -- success, partial, declined, timeout, error
    idempotency_key VARCHAR(255) UNIQUE,  -- for retry safety
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_auths_method ON payment_method_authorizations(split_method_id);

-- Split Payment Captures (actual charging)
CREATE TABLE split_payment_captures (
    id UUID PRIMARY KEY,
    split_method_id UUID NOT NULL REFERENCES split_payment_methods(id),
    split_plan_id UUID NOT NULL REFERENCES split_payment_plans(id),
    gateway_name VARCHAR(50),
    gateway_response_code VARCHAR(10),
    gateway_response_message TEXT,
    requested_amount DECIMAL(15, 2) NOT NULL,
    captured_amount DECIMAL(15, 2),
    capture_status VARCHAR(50),  -- success, partial, declined, timeout
    idempotency_key VARCHAR(255) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_captures_plan ON split_payment_captures(split_plan_id);

-- Refunds (for split payments)
CREATE TABLE split_payment_refunds (
    id UUID PRIMARY KEY,
    split_plan_id UUID NOT NULL REFERENCES split_payment_plans(id),
    refund_reason VARCHAR(255),  -- customer_request, failed_auth, fraud, etc.
    total_refund_amount DECIMAL(15, 2) NOT NULL,
    refund_status VARCHAR(50) DEFAULT 'pending',  -- pending, processing, completed, failed
    created_by VARCHAR(255),  -- customer_id or support_user_id
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_refunds_plan_status ON split_payment_refunds(split_plan_id, refund_status);

-- Refund Line Items (individual refunds per method)
CREATE TABLE split_payment_refund_items (
    id UUID PRIMARY KEY,
    refund_id UUID NOT NULL REFERENCES split_payment_refunds(id) ON DELETE CASCADE,
    split_method_id UUID NOT NULL REFERENCES split_payment_methods(id),
    original_captured_amount DECIMAL(15, 2) NOT NULL,
    refund_amount DECIMAL(15, 2) NOT NULL,
    gateway_refund_id VARCHAR(255),
    refund_item_status VARCHAR(50),  -- pending, success, failed
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_refund_items_refund ON split_payment_refund_items(refund_id);

-- Audit Log (all changes)
CREATE TABLE split_payment_audit (
    id UUID PRIMARY KEY,
    split_plan_id UUID NOT NULL REFERENCES split_payment_plans(id),
    action VARCHAR(100),  -- plan_created, auth_started, method_authorized, capture_started, etc.
    details JSONB,
    created_by VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_plan ON split_payment_audit(split_plan_id, created_at);

-- FX Rates Cache (for multi-currency splits)
CREATE TABLE fx_rates (
    id UUID PRIMARY KEY,
    from_currency VARCHAR(3) NOT NULL,
    to_currency VARCHAR(3) NOT NULL,
    rate DECIMAL(18, 6) NOT NULL,
    rate_timestamp TIMESTAMP NOT NULL,
    source VARCHAR(50),  -- openexchangerates, fixer, stripe
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(from_currency, to_currency, rate_timestamp)
);

CREATE INDEX idx_fx_rates_currencies_timestamp ON fx_rates(from_currency, to_currency, rate_timestamp DESC);
```

### MongoDB Schema

```json
{
  "_id": "ObjectId",
  "splitPaymentId": "UUID",
  "customerId": "UUID",
  "orderId": "UUID",
  "timestamp": "2024-01-15T10:30:00Z",
  "action": "method_authorized",
  "methodType": "credit_card",
  "methodToken": "tok_visa_4242",
  "requestedAmount": 50,
  "authorizedAmount": 50,
  "gatewayResponse": {
    "code": "100",
    "message": "Success",
    "transactionId": "ch_1234567890",
    "timestamp": "2024-01-15T10:30:01Z"
  },
  "riskScore": 0.12,
  "fraudChecks": {
    "velocity_check": "pass",
    "ip_geolocation_match": "pass",
    "cvc_check": "pass"
  },
  "metadata": {
    "userAgent": "Mozilla/5.0...",
    "ipAddress": "192.168.1.1",
    "deviceId": "device_abc123"
  }
}
```

---

## Redis Data Structures

```
# Split Payment Lock
split_payment:lock:{splitPaymentId}
  └─ value: lock_timestamp (expires after 60 seconds)

# Authorization Hold (temporary reserve)
split_auth:hold:{splitPaymentId}:{methodId}
  ├─ authorized_amount: 50
  ├─ gateway_auth_id: ch_1234567890
  └─ expires: 600 (10 minutes; release if not captured within window)

# Saga State (current phase)
split_saga:state:{splitPaymentId}
  ├─ phase: "capture"
  ├─ methods_completed: ["method_1", "method_2"]
  ├─ methods_failed: ["method_3"]
  └─ expires: 3600 (1 hour)

# Rate Limit (prevent rapid retries per customer)
rate_limit:split_payment:{customerId}
  └─ count: 5 (resets every minute)

# Early-Warning: Suspicious Patterns
fraud:suspicious:{customerId}
  ├─ split_count_today: 3
  ├─ total_amount_today: 1500
  ├─ last_split_timestamp: 1705313400
  └─ alert_threshold_reached: false
```

---

## Kafka Event Flow

```
Topic: split-payment-events
Partition Key: splitPaymentId (ensures order per transaction)

Event 1: SplitPaymentAuthStarted
{
  "eventId": "uuid",
  "splitPaymentId": "uuid",
  "customerId": "uuid",
  "orderId": "uuid",
  "totalAmount": 100.00,
  "currency": "USD",
  "methods": [
    { "methodId": "uuid", "type": "credit_card", "amount": 60, "priority": 1 },
    { "methodId": "uuid", "type": "loyalty_points", "amount": 40, "priority": 2 }
  ],
  "timestamp": "2024-01-15T10:30:00Z"
}
Consumers:
  - PaymentMethodExecutor (start auth sequence)
  - AuditService (log event)

Event 2: PaymentMethodAuthorized
{
  "eventId": "uuid",
  "splitPaymentId": "uuid",
  "methodId": "uuid",
  "methodType": "credit_card",
  "requestedAmount": 60,
  "authorizedAmount": 60,
  "gatewayTransactionId": "ch_1234567890",
  "timestamp": "2024-01-15T10:30:05Z"
}
Consumers:
  - SplitPaymentOrchestrator (aggregate responses, check if all authorized)
  - AuditService

Event 3: PaymentMethodAuthFailed (if card declined)
{
  "eventId": "uuid",
  "splitPaymentId": "uuid",
  "methodId": "uuid",
  "methodType": "credit_card",
  "requestedAmount": 60,
  "failureReason": "insufficient_funds",
  "gatewayErrorCode": "card_declined",
  "timestamp": "2024-01-15T10:30:06Z"
}
Consumers:
  - SplitPaymentOrchestrator (trigger rollback all authorizations)
  - FraudDetectionService (log for pattern analysis)

Event 4: SplitPaymentAuthorized (all methods authorized successfully)
{
  "eventId": "uuid",
  "splitPaymentId": "uuid",
  "customerId": "uuid",
  "orderId": "uuid",
  "totalAmount": 100.00,
  "authorizedMethods": [
    { "methodId": "uuid", "gatewayTransactionId": "ch_...", "amount": 60 },
    { "methodId": "uuid", "gatewayTransactionId": "loy_...", "amount": 40 }
  ],
  "timestamp": "2024-01-15T10:30:10Z"
}
Consumers:
  - PaymentMethodExecutor (start capture phase)
  - OrderService (update order status to PAYMENT_AUTHORIZED)

Event 5: SplitPaymentCaptured (all methods successfully charged)
{
  "eventId": "uuid",
  "splitPaymentId": "uuid",
  "customerId": "uuid",
  "orderId": "uuid",
  "totalAmount": 100.00,
  "capturedMethods": [
    { "methodId": "uuid", "captureTransactionId": "ch_cap_...", "amount": 60 },
    { "methodId": "uuid", "captureTransactionId": "loy_cap_...", "amount": 40 }
  ],
  "timestamp": "2024-01-15T10:30:15Z"
}
Consumers:
  - OrderService (finalize order)
  - AccountingService (record revenue)
  - ReconciliationService (mark for settlement)

Event 6: SplitPaymentRefunded
{
  "eventId": "uuid",
  "splitPaymentId": "uuid",
  "refundId": "uuid",
  "customerId": "uuid",
  "totalRefundAmount": 50.00,
  "refundedMethods": [
    { "methodId": "uuid", "refundAmount": 30, "refundTransactionId": "ref_..." },
    { "methodId": "uuid", "refundAmount": 20, "refundTransactionId": "ref_..." }
  ],
  "timestamp": "2024-01-15T11:00:00Z"
}
Consumers:
  - OrderService (update order refund status)
  - AccountingService (reverse revenue entry)
  - ReconciliationService (mark refund for settlement)
```

---

## Implementation Code

### SplitPaymentOrchestrator

```java
package com.payment.service.split;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.kafka.core.KafkaTemplate;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.util.*;
import java.util.concurrent.CompletableFuture;

@Slf4j
@Service
public class SplitPaymentOrchestrator {

    private final SplitPaymentRepository splitPaymentRepository;
    private final SplitPaymentMethodRepository methodRepository;
    private final SplitPaymentValidator validator;
    private final PaymentMethodExecutor executor;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final RedisTemplate<String, Object> redisTemplate;

    @Transactional
    public CompletableFuture<SplitPaymentAuthorizedEvent> processPayment(
            UUID customerId, UUID orderId, BigDecimal totalAmount, String currency,
            List<PaymentMethodRequest> methods) {

        UUID splitPaymentId = UUID.randomUUID();

        // Step 1: Validate split plan
        try {
            validator.validateSplitPlan(customerId, totalAmount, currency, methods);
        } catch (SplitPaymentValidationException e) {
            log.warn("Invalid split plan: {}", e.getMessage());
            CompletableFuture<SplitPaymentAuthorizedEvent> cf = new CompletableFuture<>();
            cf.completeExceptionally(e);
            return cf;
        }

        // Step 2: Create split payment record in database
        SplitPaymentPlan plan = new SplitPaymentPlan();
        plan.setId(splitPaymentId);
        plan.setOrderId(orderId);
        plan.setCustomerId(customerId);
        plan.setTotalAmount(totalAmount);
        plan.setCurrency(currency);
        plan.setStatus("pending");

        List<SplitPaymentMethod> paymentMethods = new ArrayList<>();
        for (int i = 0; i < methods.size(); i++) {
            PaymentMethodRequest req = methods.get(i);
            SplitPaymentMethod method = new SplitPaymentMethod();
            method.setId(UUID.randomUUID());
            method.setSplitPlanId(splitPaymentId);
            method.setMethodType(req.getType());
            method.setMethodToken(req.getToken());  // tokenized
            method.setRequestedAmount(req.getAmount());
            method.setPriority(i + 1);
            method.setAuthStatus("pending");
            paymentMethods.add(method);
        }

        plan.setPaymentMethods(paymentMethods);
        splitPaymentRepository.save(plan);

        // Step 3: Acquire lock to prevent concurrent modifications
        String lockKey = "split_payment:lock:" + splitPaymentId;
        Boolean lockAcquired = (Boolean) redisTemplate.execute(conn -> {
            byte[] keyBytes = lockKey.getBytes();
            return conn.setNX(keyBytes, String.valueOf(System.currentTimeMillis()).getBytes(), 60);
        });

        if (!lockAcquired) {
            throw new IllegalStateException("Could not acquire lock for split payment");
        }

        // Step 4: Emit event to start auth phase
        SplitPaymentAuthStartedEvent authEvent = new SplitPaymentAuthStartedEvent(
            UUID.randomUUID(),
            splitPaymentId,
            customerId,
            orderId,
            totalAmount,
            currency,
            paymentMethods.stream()
                .map(m -> new MethodSplit(m.getId(), m.getMethodType(), m.getRequestedAmount(), m.getPriority()))
                .toList(),
            Instant.now()
        );

        kafkaTemplate.send("split-payment-events", splitPaymentId.toString(), authEvent);

        // Step 5: Wait for authorization responses (with timeout)
        CompletableFuture<SplitPaymentAuthorizedEvent> result = new CompletableFuture<>();
        java.util.concurrent.ScheduledExecutorService scheduler =
            java.util.concurrent.Executors.newScheduledThreadPool(1);

        scheduler.schedule(() -> {
            // Aggregate authorization responses
            List<SplitPaymentMethod> updatedMethods = methodRepository.findBySplitPlanId(splitPaymentId);
            int successCount = (int) updatedMethods.stream()
                .filter(m -> "authorized".equals(m.getAuthStatus()))
                .count();
            int failureCount = (int) updatedMethods.stream()
                .filter(m -> "failed".equals(m.getAuthStatus()))
                .count();

            log.info("Auth aggregation: splitPaymentId={}, success={}, failed={}",
                     splitPaymentId, successCount, failureCount);

            if (failureCount > 0 && successCount == 0) {
                // All failed or some failed with no success
                plan.setStatus("failed");
                splitPaymentRepository.save(plan);
                result.completeExceptionally(
                    new SplitPaymentAuthorizationFailedException("All methods failed to authorize")
                );
            } else if (failureCount > 0 && successCount > 0) {
                // Partial authorization
                plan.setStatus("partially_authorized");
                splitPaymentRepository.save(plan);
                SplitPaymentAuthorizedEvent event = new SplitPaymentAuthorizedEvent(
                    UUID.randomUUID(),
                    splitPaymentId,
                    customerId,
                    orderId,
                    totalAmount,
                    updatedMethods.stream()
                        .filter(m -> "authorized".equals(m.getAuthStatus()))
                        .map(m -> new AuthorizedMethod(m.getId(), m.getAuthTransactionId(), m.getAuthorizedAmount()))
                        .toList(),
                    "PARTIAL",
                    Instant.now()
                );
                result.complete(event);
            } else {
                // All authorized successfully
                plan.setStatus("authorized");
                splitPaymentRepository.save(plan);
                kafkaTemplate.send("split-payment-events", splitPaymentId.toString(),
                    new SplitPaymentAuthorizedEvent(
                        UUID.randomUUID(),
                        splitPaymentId,
                        customerId,
                        orderId,
                        totalAmount,
                        updatedMethods.stream()
                            .map(m -> new AuthorizedMethod(m.getId(), m.getAuthTransactionId(), m.getAuthorizedAmount()))
                            .toList(),
                        "SUCCESS",
                        Instant.now()
                    )
                );
            }

            // Release lock
            redisTemplate.delete(lockKey);
        }, 5, java.util.concurrent.TimeUnit.SECONDS);  // Wait 5 seconds for responses

        return result;
    }
}

record MethodSplit(UUID methodId, String type, BigDecimal amount, int priority) { }
record AuthorizedMethod(UUID methodId, String gatewayTransactionId, BigDecimal authorizedAmount) { }
```

### PaymentMethodExecutor

```java
package com.payment.service.split;

import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import lombok.extern.slf4j.Slf4j;
import java.util.*;

@Slf4j
@Service
public class PaymentMethodExecutor {

    private final SplitPaymentRepository splitPaymentRepository;
    private final SplitPaymentMethodRepository methodRepository;
    private final StripePaymentGateway stripeGateway;
    private final PayPalPaymentGateway paypalGateway;
    private final LoyaltyPointsService loyaltyService;
    private final GiftCardService giftCardService;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @KafkaListener(topics = "split-payment-events", groupId = "payment-method-executor")
    public void onAuthStarted(SplitPaymentAuthStartedEvent event) {
        log.info("Starting auth sequence for splitPaymentId: {}", event.splitPaymentId());

        SplitPaymentPlan plan = splitPaymentRepository.findById(event.splitPaymentId())
            .orElseThrow();

        List<SplitPaymentMethod> methods = plan.getPaymentMethods();
        methods.sort(Comparator.comparingInt(SplitPaymentMethod::getPriority));

        for (SplitPaymentMethod method : methods) {
            try {
                authorizeMethod(method, event.splitPaymentId());
            } catch (Exception e) {
                log.error("Failed to authorize method {} for split {}", method.getId(), event.splitPaymentId(), e);
                // Publish failure event; orchestrator will handle rollback
                PaymentMethodAuthFailedEvent failEvent = new PaymentMethodAuthFailedEvent(
                    UUID.randomUUID(),
                    event.splitPaymentId(),
                    method.getId(),
                    method.getMethodType(),
                    e.getMessage(),
                    Instant.now()
                );
                kafkaTemplate.send("split-payment-events", event.splitPaymentId().toString(), failEvent);
            }
        }
    }

    @Transactional
    private void authorizeMethod(SplitPaymentMethod method, UUID splitPaymentId) throws Exception {
        String idempotencyKey = generateIdempotencyKey(splitPaymentId, method.getId(), "auth");

        PaymentAuthorizationResponse response = switch (method.getMethodType()) {
            case "credit_card" -> {
                StripeAuthRequest req = new StripeAuthRequest(
                    method.getMethodToken(),
                    method.getRequestedAmount(),
                    "USD",  // TODO: get from plan
                    idempotencyKey
                );
                yield stripeGateway.authorize(req);
            }
            case "loyalty_points" -> {
                LoyaltyAuthRequest req = new LoyaltyAuthRequest(
                    method.getMethodToken(),  // customerId
                    (long) method.getRequestedAmount().doubleValue(),
                    idempotencyKey
                );
                yield loyaltyService.authorizePoints(req);
            }
            case "gift_card" -> {
                GiftCardAuthRequest req = new GiftCardAuthRequest(
                    method.getMethodToken(),
                    method.getRequestedAmount(),
                    idempotencyKey
                );
                yield giftCardService.authorize(req);
            }
            default -> throw new UnsupportedOperationException("Unsupported method type: " + method.getMethodType());
        };

        // Update method with auth result
        method.setAuthTransactionId(response.getTransactionId());
        method.setAuthorizedAmount(response.getAuthorizedAmount());
        method.setAuthStatus(response.getStatus());  // "authorized", "failed", "partial"

        methodRepository.save(method);

        // Emit event
        if ("authorized".equals(response.getStatus()) || "partial".equals(response.getStatus())) {
            PaymentMethodAuthorizedEvent authEvent = new PaymentMethodAuthorizedEvent(
                UUID.randomUUID(),
                splitPaymentId,
                method.getId(),
                method.getMethodType(),
                method.getRequestedAmount(),
                response.getAuthorizedAmount(),
                response.getTransactionId(),
                Instant.now()
            );
            kafkaTemplate.send("split-payment-events", splitPaymentId.toString(), authEvent);
        } else {
            PaymentMethodAuthFailedEvent failEvent = new PaymentMethodAuthFailedEvent(
                UUID.randomUUID(),
                splitPaymentId,
                method.getId(),
                method.getMethodType(),
                response.getFailureReason(),
                Instant.now()
            );
            kafkaTemplate.send("split-payment-events", splitPaymentId.toString(), failEvent);
        }
    }

    private String generateIdempotencyKey(UUID splitPaymentId, UUID methodId, String action) {
        String combined = splitPaymentId.toString() + methodId.toString() + action + System.nanoTime();
        return org.apache.commons.codec.digest.DigestUtils.md5Hex(combined);
    }
}
```

### SplitPaymentValidator

```java
package com.payment.service.split;

import org.springframework.stereotype.Service;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.util.List;
import java.util.UUID;

@Slf4j
@Service
public class SplitPaymentValidator {

    private final CustomerRepository customerRepository;
    private final LoyaltyPointsService loyaltyService;
    private final GiftCardService giftCardService;
    private final FraudDetectionService fraudService;

    public void validateSplitPlan(UUID customerId, BigDecimal totalAmount, String currency,
                                   List<PaymentMethodRequest> methods) throws SplitPaymentValidationException {

        // Validate customer exists
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new SplitPaymentValidationException("Customer not found"));

        // Validate total amount > 0
        if (totalAmount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new SplitPaymentValidationException("Total amount must be positive");
        }

        // Validate methods list not empty
        if (methods.isEmpty()) {
            throw new SplitPaymentValidationException("At least one payment method required");
        }

        // Validate sum of requested amounts matches total
        BigDecimal sumRequested = methods.stream()
            .map(PaymentMethodRequest::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        if (sumRequested.compareTo(totalAmount) != 0) {
            throw new SplitPaymentValidationException(
                "Sum of payment methods (" + sumRequested + ") does not match total (" + totalAmount + ")"
            );
        }

        // Validate each method has sufficient balance
        for (PaymentMethodRequest method : methods) {
            switch (method.getType()) {
                case "loyalty_points" -> {
                    long availablePoints = loyaltyService.getAvailablePoints(customerId);
                    long requestedPoints = (long) method.getAmount().doubleValue();
                    if (requestedPoints > availablePoints) {
                        throw new SplitPaymentValidationException(
                            "Insufficient loyalty points: requested=" + requestedPoints + ", available=" + availablePoints
                        );
                    }
                }
                case "gift_card" -> {
                    BigDecimal giftCardBalance = giftCardService.getBalance(method.getToken());
                    if (method.getAmount().compareTo(giftCardBalance) > 0) {
                        throw new SplitPaymentValidationException(
                            "Insufficient gift card balance: requested=" + method.getAmount() + ", available=" + giftCardBalance
                        );
                    }
                }
                case "credit_card" -> {
                    // Credit card validation happens at auth time (gateway decides)
                    log.info("Credit card {} will be validated at authorization time", method.getToken());
                }
            }
        }

        // Fraud check: suspicious split patterns
        fraudService.checkSuspiciousSplitPattern(customerId, totalAmount);

        log.info("Split payment plan validated for customer {}: methods={}, total={}",
                 customerId, methods.size(), totalAmount);
    }
}

exception class SplitPaymentValidationException extends Exception {
    public SplitPaymentValidationException(String message) {
        super(message);
    }
}
```

### SplitPaymentRefundService

```java
package com.payment.service.split;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.util.*;

@Slf4j
@Service
public class SplitPaymentRefundService {

    private final SplitPaymentRepository splitPaymentRepository;
    private final SplitPaymentRefundRepository refundRepository;
    private final SplitPaymentRefundItemRepository refundItemRepository;
    private final StripePaymentGateway stripeGateway;
    private final PayPalPaymentGateway paypalGateway;
    private final LoyaltyPointsService loyaltyService;
    private final GiftCardService giftCardService;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public SplitPaymentRefundedEvent refundPayment(UUID splitPaymentId, BigDecimal refundAmount,
                                                     String reason) throws SplitPaymentRefundException {

        SplitPaymentPlan plan = splitPaymentRepository.findById(splitPaymentId)
            .orElseThrow(() -> new SplitPaymentRefundException("Split payment not found"));

        if (!"captured".equals(plan.getStatus())) {
            throw new SplitPaymentRefundException("Can only refund captured payments");
        }

        // Create refund record
        SplitPaymentRefund refund = new SplitPaymentRefund();
        refund.setId(UUID.randomUUID());
        refund.setSplitPaymentId(splitPaymentId);
        refund.setRefundReason(reason);
        refund.setTotalRefundAmount(refundAmount);
        refund.setRefundStatus("pending");

        // Calculate proportional split
        List<SplitPaymentMethod> capturedMethods = plan.getPaymentMethods().stream()
            .filter(m -> m.getCapturedAmount() != null && m.getCapturedAmount().compareTo(BigDecimal.ZERO) > 0)
            .sorted(Comparator.comparingInt(SplitPaymentMethod::getPriority).reversed())  // reverse order
            .toList();

        BigDecimal totalCaptured = capturedMethods.stream()
            .map(SplitPaymentMethod::getCapturedAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        List<SplitPaymentRefundItem> refundItems = new ArrayList<>();
        BigDecimal remainingRefund = refundAmount;

        for (SplitPaymentMethod method : capturedMethods) {
            BigDecimal methodProportion = method.getCapturedAmount().divide(totalCaptured, 4, java.math.RoundingMode.HALF_UP);
            BigDecimal methodRefundAmount = refundAmount.multiply(methodProportion)
                .setScale(2, java.math.RoundingMode.HALF_UP);

            // Cap at method's captured amount
            methodRefundAmount = methodRefundAmount.min(method.getCapturedAmount());
            remainingRefund = remainingRefund.subtract(methodRefundAmount);

            SplitPaymentRefundItem item = new SplitPaymentRefundItem();
            item.setId(UUID.randomUUID());
            item.setRefundId(refund.getId());
            item.setSplitMethodId(method.getId());
            item.setOriginalCapturedAmount(method.getCapturedAmount());
            item.setRefundAmount(methodRefundAmount);
            item.setRefundItemStatus("pending");

            refundItems.add(item);
        }

        // If remaining > 0 due to rounding, add to last item
        if (remainingRefund.compareTo(BigDecimal.ZERO) > 0) {
            SplitPaymentRefundItem lastItem = refundItems.get(refundItems.size() - 1);
            lastItem.setRefundAmount(lastItem.getRefundAmount().add(remainingRefund));
        }

        // Save refund records
        refundRepository.save(refund);
        refundItemRepository.saveAll(refundItems);

        // Process refunds to gateways (reverse order for max success)
        for (SplitPaymentRefundItem item : refundItems) {
            try {
                processRefundItem(item);
            } catch (Exception e) {
                log.error("Failed to refund item {}", item.getId(), e);
                item.setRefundItemStatus("failed");
                refundItemRepository.save(item);
            }
        }

        // Check if all refunds succeeded
        long successCount = refundItems.stream()
            .filter(i -> "success".equals(i.getRefundItemStatus()))
            .count();

        refund.setRefundStatus(successCount == refundItems.size() ? "completed" : "partial_failed");
        refundRepository.save(refund);

        // Emit event
        SplitPaymentRefundedEvent event = new SplitPaymentRefundedEvent(
            UUID.randomUUID(),
            splitPaymentId,
            plan.getCustomerId(),
            refundAmount,
            refundItems.stream()
                .map(i -> new RefundedMethod(i.getSplitMethodId(), i.getRefundAmount()))
                .toList(),
            Instant.now()
        );
        kafkaTemplate.send("split-payment-events", splitPaymentId.toString(), event);

        return event;
    }

    @Transactional
    private void processRefundItem(SplitPaymentRefundItem item) throws SplitPaymentRefundException {
        SplitPaymentMethod method = methodRepository.findById(item.getSplitMethodId())
            .orElseThrow();

        switch (method.getMethodType()) {
            case "credit_card" -> {
                StripeRefundRequest req = new StripeRefundRequest(
                    method.getCaptureTransactionId(),
                    item.getRefundAmount(),
                    generateIdempotencyKey(method.getId(), "refund")
                );
                StripeRefundResponse resp = stripeGateway.refund(req);
                item.setGatewayRefundId(resp.getRefundId());
                item.setRefundItemStatus("success");
            }
            case "loyalty_points" -> {
                loyaltyService.returnPoints(
                    method.getMethodToken(),
                    (long) item.getRefundAmount().doubleValue()
                );
                item.setRefundItemStatus("success");
            }
            case "gift_card" -> {
                giftCardService.addBalance(
                    method.getMethodToken(),
                    item.getRefundAmount()
                );
                item.setRefundItemStatus("success");
            }
        }

        refundItemRepository.save(item);
    }

    private String generateIdempotencyKey(UUID methodId, String action) {
        return org.apache.commons.codec.digest.DigestUtils.md5Hex(
            methodId.toString() + action + System.nanoTime()
        );
    }
}

record RefundedMethod(UUID methodId, BigDecimal refundAmount) { }
exception class SplitPaymentRefundException extends Exception {
    public SplitPaymentRefundException(String message) { super(message); }
    public SplitPaymentRefundException(String message, Throwable cause) { super(message, cause); }
}
```

---

## Failure Scenarios

| Scenario | Handling |
|----------|----------|
| Auth timeout (no response from gateway) | 30-second timeout; emit TIMEOUT event; rollback all previous auths; reject transaction |
| Partial authorization (card approved $80 of $100) | Emit PartialAuthorizationWaiting event; prompt user to cover delta with another method or cancel |
| Method 1 auth succeeds, Method 2 auth fails | Compensate Method 1 (void auth); emit AuthFailed event; reject entire transaction |
| Capture succeeds for Method 1, fails for Method 2 | Refund Method 1 immediately; flag for manual review; notify customer + support |
| Refund fails (gateway down) | Retry 3x (exponential backoff); mark item as "failed"; create manual refund task; notify support |
| Idempotency key collision (duplicate request) | Gateway returns cached response (safe retry) |
| Race condition (concurrent auth + capture) | Distributed lock in Redis prevents concurrent modifications; second request waits/fails with 409 |
| Fraud detected (velocity too high) | Emit FraudAlertEvent; block transaction; require customer verification; escalate to support |

---

## Scaling Strategy

### Horizontal Scaling

| Component | Scaling Strategy |
|-----------|------------------|
| SplitPaymentOrchestrator | Stateless; scale horizontally; Redis for distributed locks |
| PaymentMethodExecutor | Kafka consumer group; scale with partitions (10+ partitions per method type) |
| SplitPaymentRefundService | Async service; scale with worker pool; prioritize by age (oldest first) |
| Fraud Detection | Real-time scoring; cache customer profiles in Redis for <10ms checks |

### Database Optimization

- **Partitioning:** split_payment_plans by customer_id; split_payment_refunds by created_at (monthly)
- **Indexing:** (split_plan_id, auth_status), (customer_id, created_at), (split_method_id, capture_status)
- **Archive:** refunds older than 1 year → cold storage (S3); keep active refunds in hot table

### Kafka Tuning

- **Partitions:** 10+ partitions per method type (card, paypal, loyalty) to enable parallel processing
- **Consumer Group:** separate for auth-phase, capture-phase, refund-phase
- **Retention:** 30 days (enable replay for reconciliation)
- **Batch Size:** 100 messages/batch for throughput

---

## Monitoring

### Key Metrics

| Metric | Target | Tool |
|--------|--------|------|
| Auth success rate | >98% | Prometheus counter |
| Capture success rate (given auth success) | >99.5% | Prometheus counter |
| Avg auth latency | <1.5s | Prometheus histogram |
| Refund processing latency | <5s | Prometheus histogram |
| Fraud detection F1 score | >0.90 | Custom ML metrics |
| Partial auth % of total | <2% | Prometheus gauge |
| Kafka consumer lag | <10 seconds | Kafka console |

### Alerts

```yaml
- alert: AuthFailureRateHigh
  expr: rate(split_payment_auth_failures_total[5m]) > 0.05
  for: 5m
  action: page on-call

- alert: CaptureFailureHigh
  expr: rate(split_payment_capture_failures_total[5m]) > 0.01
  for: 5m
  action: notify payments-team

- alert: RefundProcessingLatency
  expr: histogram_quantile(0.95, split_payment_refund_duration_seconds) > 5
  for: 10m
  action: investigate payment gateway performance

- alert: KafkaConsumerLagHigh
  expr: kafka_consumergroup_lag > 10000
  for: 5m
  action: scale consumer group
```

### Dashboards

- **Payment Flow Dashboard:** auth rate by method type, capture success rate, latency heatmaps
- **Failure Dashboard:** auth failures by reason (declined, timeout, error), capture failures by gateway
- **Refund Dashboard:** refund success rate, latency breakdown, manual tasks backlog
- **Fraud Dashboard:** fraud score distribution, blocked transactions, customer risk scores

---

## Summary Cheat Sheet

```
SPLIT PAYMENT FLOW:
  1. ValidateSplitPlan: check balances, FX rates, fraud score
  2. AuthPhase: call each method's gateway in priority order; wait for all responses
  3. CapturePhase: if all auth success → capture all methods
  4. Failure: any auth/capture fails → compensate all previous → reject transaction

TWO-PHASE COMMIT (SAGA):
  - Auth Phase: hold funds on all methods; if any fails → release all holds
  - Capture Phase: actually charge all methods; if any fails → refund all captures
  - Idempotency: every call includes unique idempotencyKey (hash of splitPaymentId + methodId + action + nonce)
  - Timeout: 30s for auth, 5s for capture; exceed timeout → trigger compensating transaction

PARTIAL AUTHORIZATION:
  - If method authorizes less than requested (e.g., $80 of $100 requested)
  - Emit PartialAuthorizationWaiting event
  - UI prompts user: cover delta with another method or cancel
  - If user confirms → adjust split and retry remaining delta with next method

REFUNDS:
  - Calculate proportional split: method's refund % = method's captured % of total
  - Process refunds in reverse order (last authorized first) for max success
  - Each refund item tracked separately; partial refund allowed if some items fail
  - Idempotency: refundId is unique; second call returns cached result

MULTI-CURRENCY:
  - FX rate pulled from OpenExchangeRates at auth time, cached in Redis (1-hour TTL)
  - All ledger entries converted to customer's home currency using rate snapshot
  - Refunds issued in original method's currency (Stripe/PayPal handles conversion)
  - If FX variance > 3% between auth and capture → reject, prompt re-auth

FRAUD DETECTION:
  - Real-time scoring: velocity (splits/min), device fingerprint, IP geolocation, amount outlier
  - Score >0.8 → block transaction, require verification (OTP, 3D Secure)
  - Redis cache: customer risk profile (last 24h splits, total amount, unique devices)
  - ML model: trained on historical fraud labels; re-trained weekly

KEY TABLES:
  - split_payment_plans: one row per transaction
  - split_payment_methods: one row per method in split (avg 2-3 per transaction)
  - payment_method_authorizations: one row per auth attempt (retry attempts)
  - split_payment_refunds: one row per refund request
  - split_payment_refund_items: one row per method in refund

KAFKA TOPICS:
  - split-payment-events: key=splitPaymentId; events: AuthStarted, MethodAuthorized, MethodAuthFailed,
                          SplitPaymentAuthorized, SplitPaymentCaptured, SplitPaymentRefunded
  - Partition key ensures all events for one transaction go to same partition (ordering guaranteed)

REDIS KEYS:
  - split_payment:lock:{splitPaymentId}: prevent concurrent modifications (60s TTL)
  - split_auth:hold:{splitPaymentId}:{methodId}: temporary auth hold (10 min TTL)
  - split_saga:state:{splitPaymentId}: current saga phase (1h TTL)
  - rate_limit:split_payment:{customerId}: rapid-fire attempt prevention (1 min TTL)
  - fraud:suspicious:{customerId}: cumulative risk profile (24h TTL)
```

---

**Last Updated:** 2024-01-15 | **Version:** 1.0 | **Owner:** Payments Team
