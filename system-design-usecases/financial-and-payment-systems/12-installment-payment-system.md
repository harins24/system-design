---
title: Installment Payment System (Pay in 4)
layout: default
---

# Installment Payment System (Pay in 4) — Deep Dive Design

> **Scenario:** Customer pays 25% at checkout, then 3 installments over 6 weeks. Auto-charge on schedule. Late fees for missed payments. Early payoff option. Credit bureau integration. Risk assessment. Handle payment failures.
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
10. [Failure Scenarios](#failure-scenarios)
11. [Scaling Strategy](#scaling-strategy)
12. [Monitoring & Alerting](#monitoring--alerting)
13. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Requirements & Constraints

| Requirement | Details |
|---|---|
| **Payment Schedule** | 4 payments: 25% at checkout, then 25% each at weeks 2, 4, 6 |
| **Auto-Charge** | Scheduled charges; 3 retries on failure (D+1, D+3, D+7) |
| **Late Fees** | $7 flat or 5% of installment (whichever lower); max $50 |
| **Early Payoff** | Customer can pay remaining balance; minor discount (0.5% per month saved) |
| **Risk Assessment** | Credit score proxy using order history, return rate, account age, payment history |
| **Risk Threshold** | Approval if score ≥ 60/100 |
| **Default Handling** | COLLECTIONS status after 3 retries; escalate to external agency after 30 days |
| **Credit Bureau** | Monthly Metro 2 format; report to Equifax/TransUnion |
| **Throughput** | 100K installment plans/day; peak 1.5K plans/hour |
| **Reporting** | Daily delinquency reports; weekly collections summary |
| **Compliance** | ECOA, FCRA, Regulation Z (Truth in Lending) |
| **Data Retention** | 7 years (regulatory); 10 years for credit bureau |

---

## Capacity Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Daily Plans Created** | 100K/day | 100,000 plans |
| **Total Installments** | 100K × 4 installments | 400K installments/day |
| **Peak Charges/Hour** | 400K ÷ 168 hours (6 weeks) | ~2.4K charges/hour |
| **Retry Charges** | 2.4K × 0.03 (3% fail rate) × 3 retries | ~216 retries/hour |
| **Storage (PostgreSQL)** | 100K × 365 × 2 years × 2KB/plan | ~146 GB |
| **Installment Records** | 400K/day × 365 × 2 years × 500 bytes | ~73 GB |
| **Redis Cache** | In-flight charges + payment reminders | ~100 MB |
| **Kafka Partitions** | installment_charges topic | 20 partitions |
| **Credit Bureau Files** | Monthly Metro 2 exports | ~50 MB per file |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    INSTALLMENT PAYMENT SYSTEM (PAY IN 4)                     │
└─────────────────────────────────────────────────────────────────────────────┘

CHECKOUT FLOW
─────────────
1. Customer selects "Pay in 4" on checkout
2. InstallmentEligibilityService evaluates risk
   ├─ Risk score calculation (0-100)
   ├─ Approval decision (threshold: ≥60)
   └─ Return offer: 4 payment dates
3. If approved: Create InstallmentPlan + 4 InstallmentSchedules
4. Charge first installment (25%) immediately
5. Return to merchant with success

┌──────────────────────────────────────────────────────────────────────────────┐
│              INSTALLMENT PLAN SERVICE (Spring Boot, Multi-instance)          │
├──────────────────────────────────────────────────────────────────────────────┤
│ REST API:                                                                    │
│ · POST   /installments/apply         → RiskAssessment → Approval/Denial      │
│ · GET    /installments/{planId}      → Plan details + schedule               │
│ · POST   /installments/{planId}/payoff → Early payoff calculation            │
│ · GET    /installments/due           → Upcoming charges (Scheduled job)       │
│ · POST   /installments/{planId}/retry → Manual retry                         │
└──────────────────────────────────────────────────────────────────────────────┘
                                    ↓

        ┌───────────────────────────────────────────────────────┐
        │        RISK ASSESSMENT & APPROVAL ENGINE              │
        ├───────────────────────────────────────────────────────┤
        │ InstallmentRiskAssessor:                              │
        │ ├─ Credit score proxy (account age, return rate, etc) │
        │ ├─ Historical payment success rate                    │
        │ ├─ Current outstanding balance                        │
        │ ├─ Device fingerprint / IP velocity checks            │
        │ ├─ Score 0-100; threshold ≥60 for approval            │
        │ └─ Return: RiskDecision (APPROVED/DENIED + reason)    │
        └───────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────────┐
│                   SCHEDULED CHARGE EXECUTION (Quartz)                         │
├──────────────────────────────────────────────────────────────────────────────┤
│ InstallmentChargeJob (Daily triggers every 6 hours):                         │
│ ├─ Cron: "0 0 0,6,12,18 * * *" (Every 6 hours)                              │
│ ├─ Query: Due installments (due_date ≤ TODAY)                                │
│ ├─ Lock: Prevent concurrent charges on same installment (version check)      │
│ ├─ For each due:                                                             │
│ │  ├─ Charge payment gateway (Stripe/Adyen)                                  │
│ │  ├─ Idempotent: Dedup on (installment_id, attempt_number)                  │
│ │  ├─ If success: Mark PAID; publish event                                   │
│ │  └─ If fail: Schedule retry; update status to FAILED_ATTEMPT_1             │
│ └─ Partition by date range for parallel processing                           │
└──────────────────────────────────────────────────────────────────────────────┘
                                    ↓
    ┌──────────────────────────────────────────────────────────┐
    │       PAYMENT GATEWAY INTEGRATION                         │
    ├──────────────────────────────────────────────────────────┤
    │ PaymentChargeService:                                    │
    │ ├─ Tokenized card on file                               │
    │ ├─ Fallback payment method (bank account)               │
    │ ├─ 3 retries: D+1, D+3, D+7                             │
    │ ├─ Soft decline (insufficient funds) → retry             │
    │ └─ Hard decline (fraud, lost card) → COLLECTIONS         │
    └──────────────────────────────────────────────────────────┘
                                    ↓
        ┌───────────────────────────────────────────────────────┐
        │      LATE FEE & COLLECTIONS MANAGEMENT                │
        ├───────────────────────────────────────────────────────┤
        │ LateFeeCalculator:                                    │
        │ ├─ Fee = MIN($7, installment * 0.05)                  │
        │ ├─ Applied at MISSED + 1 day                          │
        │ ├─ Max fee cap: $50 per installment                   │
        │ └─ Add to next charge (or separate debit)             │
        │                                                       │
        │ CollectionsManager:                                   │
        │ ├─ Status: COLLECTIONS after 3 failed retries         │
        │ ├─ Escalate to external agency after 30 days          │
        │ └─ Track escalation status (sent, received, paid)     │
        └───────────────────────────────────────────────────────┘
                                    ↓
                    ┌──────────────────────────────┐
                    │    KAFKA EVENT STREAM        │
                    ├──────────────────────────────┤
                    │ installment_plans_created    │
                    │ installment_charges_due      │
                    │ installment_charged_success  │
                    │ installment_charged_failed   │
                    │ installment_missed_payment   │
                    │ installment_paid_off         │
                    │ credit_bureau_reports        │
                    └──────────────────────────────┘
                                    ↓
    ┌──────────────────────────────────────────────────────────┐
    │   CREDIT BUREAU REPORTING (Monthly Metro 2 Export)       │
    ├──────────────────────────────────────────────────────────┤
    │ CreditBureauReporter:                                    │
    │ ├─ Monthly batch job (1st of month)                      │
    │ ├─ Generate Metro 2 format file                          │
    │ ├─ Report status: I (Installment) with payment history   │
    │ ├─ SFTP to Equifax, TransUnion, Experian                 │
    │ └─ Track submission + confirmation                       │
    └──────────────────────────────────────────────────────────┘
                                    ↓
        ┌───────────────────────────────────────────────────────┐
        │      REPORTING & ANALYTICS (MongoDB + REST API)       │
        ├───────────────────────────────────────────────────────┤
        │ · Daily delinquency report (overdue by age)           │
        │ · Weekly collections summary (status breakdown)       │
        │ · Monthly default rate analysis                       │
        │ · Cohort analysis (score bucket vs actual defaults)   │
        │ · Risk model performance tracking                     │
        └───────────────────────────────────────────────────────┘
                                    ↓
        ┌───────────────────────────────────────────────────────┐
        │      DATA LAYER (PostgreSQL + MongoDB + Redis)        │
        ├───────────────────────────────────────────────────────┤
        │ PostgreSQL:              │ MongoDB:           │ Redis:│
        │ · installment_plans      │ · credit_decisions │ · duedate_idx
        │ · installment_schedules  │ · payment_history  │ · retry_queue
        │ · charge_records         │ · dispute_cases    │ · lock:{plan}
        │ · late_fees              │ · audit_logs       │ · charge_tokens
        │ · credit_bureau_exports  │                    │       │
        └───────────────────────────────────────────────────────┘
```

---

## Core Design Questions Answered

### 1. How do you assess risk and approve installment eligibility?

**Risk Scoring Model (0-100):**

```
Score Calculation:
─────────────────
account_age_score (0-20):
  - Age < 30 days: 0
  - Age 30-90 days: 5
  - Age 90-180 days: 10
  - Age 180-365 days: 15
  - Age > 365 days: 20

payment_history_score (0-25):
  - No history: 0
  - Success rate 0-50%: 5
  - Success rate 50-75%: 10
  - Success rate 75-90%: 15
  - Success rate 90-99%: 20
  - Success rate 100%: 25

return_rate_score (0-15):
  - Return rate > 10%: 0
  - Return rate 5-10%: 5
  - Return rate 2-5%: 10
  - Return rate < 2%: 15

financial_health_score (0-20):
  - Outstanding balance > 5K: 0
  - Outstanding balance 2K-5K: 5
  - Outstanding balance 500-2K: 10
  - Outstanding balance < 500: 15
  - New customer (no balance): 20

behavioral_score (0-20):
  - High velocity (3+ attempts same day): 0
  - Device/IP mismatch: 5
  - Known fraud signals: 0
  - Clean checks: 20

FINAL_SCORE = min(100, account_age + payment_history + return_rate + financial_health + behavioral)

Approval Threshold: SCORE >= 60
Soft Decline (manual review): SCORE 50-59
Hard Decline: SCORE < 50
```

### 2. How do you schedule installment payments reliably?

**Quartz Scheduler Integration:**

- Job: `InstallmentChargeJob`
- Trigger: Cron-based (every 6 hours) + database-driven (check due dates)
- Idempotency: `UNIQUE(installment_id, attempt_number, execution_date)`
- Partitioning: Process 100K charges in 20 parallel steps (5K each)
- Retry Strategy: Built-in Quartz job listener for failures

### 3. How do you handle missed payments and collections?

**State Machine:**

```
PENDING → CHARGED_SUCCESS → PAID (end)
       ↓
       CHARGED_FAILED_ATTEMPT_1 (D+1 retry)
       ↓
       CHARGED_FAILED_ATTEMPT_2 (D+3 retry)
       ↓
       CHARGED_FAILED_ATTEMPT_3 (D+7 retry)
       ↓
       MISSED → COLLECTIONS (external agency)
                ↓
                ESCALATED_D30 (30 days late)
                ↓
                WRITTEN_OFF (90 days late)
```

Failure reasons: `SOFT_DECLINE`, `HARD_DECLINE`, `INVALID_CARD`, `ACCOUNT_CLOSED`, `INSUFFICIENT_FUNDS`

### 4. How do you calculate and apply late fees?

**Late Fee Logic:**

```
IF (today > due_date) AND status IN (MISSED, FAILED_ATTEMPT_*):
  late_fee_cents = MIN(
    700,  // $7 flat
    (installment_amount_cents * 5) / 100  // 5% of installment
  )

IF late_fee_cents > 0:
  - Create LateFeeRecord
  - Add to next charge (or separate debit)
  - Record in installment_ledger for audit
```

### 5. How do you allow early payoff?

**Early Payoff Calculation:**

```
Remaining Balance = SUM(unpaid installments)
Prepayment Discount = (months_remaining * 0.5%) of remaining balance
Final Amount = Remaining Balance - Prepayment Discount

Example:
- Original plan: $1000 (4 × $250)
- Paid: 2 installments ($500)
- Remaining: $500 (2 installments)
- Discount: $500 × 2 months × 0.005 = $5
- Payoff amount: $495
```

### 6. How do you report to credit bureaus?

**Metro 2 Format Export:**

- Monthly file generation (1st of month)
- Fields: Account number, Consumer name, Date opened, Current balance, Payment history, Status code
- Status codes: I (Installment), 0 (Current), 1 (30 days late), etc.
- SFTP delivery to 3 bureaus
- Track confirmation + resubmission on errors

---

## Microservices Breakdown

| Service | Responsibility | Tech | Instances |
|---|---|---|---|
| **Installment Plan Service** | Create plans, assess risk, approve/deny | Spring Boot + PostgreSQL | 5 |
| **Installment Charge Service** | Scheduled charges, retries, state transitions | Spring Boot + Quartz | 3 |
| **Late Fee & Collections Service** | Calculate fees, escalate to agency | Spring Boot + MongoDB | 2 |
| **Credit Bureau Service** | Metro 2 export, SFTP delivery | Spring Boot + Batch | 1 |
| **Reporting Service** | Delinquency reports, analytics | Spring Boot + SQL | 2 |

---

## Database Design (DDL)

```sql
-- PostgreSQL

CREATE TABLE installment_plans (
    id BIGSERIAL PRIMARY KEY,
    order_id VARCHAR(100) NOT NULL UNIQUE,
    customer_id BIGINT NOT NULL,
    total_amount_cents BIGINT NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    interest_rate_percent DECIMAL(5,2) DEFAULT 0,
    status VARCHAR(30) DEFAULT 'ACTIVE',
    risk_score INT,
    risk_decision VARCHAR(30), -- APPROVED, DENIED, MANUAL_REVIEW
    risk_reason TEXT,
    first_charge_at TIMESTAMP NOT NULL,
    approval_time_ms BIGINT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP,
    CONSTRAINT valid_risk_score CHECK (risk_score >= 0 AND risk_score <= 100)
);

CREATE TABLE installment_schedules (
    id BIGSERIAL PRIMARY KEY,
    plan_id BIGINT NOT NULL REFERENCES installment_plans(id) ON DELETE CASCADE,
    installment_number INT NOT NULL,
    due_date DATE NOT NULL,
    amount_cents BIGINT NOT NULL,
    status VARCHAR(30) DEFAULT 'PENDING', -- PENDING, CHARGED_SUCCESS, FAILED_ATTEMPT_1/2/3, MISSED, PAID, COLLECTIONS
    charged_at TIMESTAMP,
    last_failed_at TIMESTAMP,
    last_failure_reason VARCHAR(100),
    attempt_count INT DEFAULT 0,
    late_fee_cents BIGINT DEFAULT 0,
    version BIGINT DEFAULT 1,
    UNIQUE(plan_id, installment_number)
);

CREATE TABLE charge_records (
    id BIGSERIAL PRIMARY KEY,
    schedule_id BIGINT NOT NULL REFERENCES installment_schedules(id),
    plan_id BIGINT NOT NULL REFERENCES installment_plans(id),
    attempt_number INT NOT NULL,
    execution_date DATE NOT NULL,
    amount_cents BIGINT NOT NULL,
    gateway_transaction_id VARCHAR(100),
    status VARCHAR(30), -- SUCCESS, SOFT_DECLINE, HARD_DECLINE, TIMEOUT
    failure_message TEXT,
    retry_scheduled_date DATE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    UNIQUE(schedule_id, attempt_number, execution_date)
);

CREATE TABLE late_fees (
    id BIGSERIAL PRIMARY KEY,
    schedule_id BIGINT NOT NULL REFERENCES installment_schedules(id),
    fee_cents BIGINT NOT NULL,
    fee_type VARCHAR(30), -- LATE_FEE, NSF_FEE
    applied_date DATE NOT NULL,
    charged_with_installment_id BIGINT REFERENCES installment_schedules(id),
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE collections_cases (
    id BIGSERIAL PRIMARY KEY,
    plan_id BIGINT NOT NULL REFERENCES installment_plans(id),
    case_status VARCHAR(30) DEFAULT 'OPEN', -- OPEN, ESCALATED_D30, WRITTEN_OFF, RECOVERED
    escalated_date DATE,
    external_agency_reference VARCHAR(100),
    last_contact_date DATE,
    recovery_amount_cents BIGINT DEFAULT 0,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP
);

CREATE TABLE credit_bureau_exports (
    id BIGSERIAL PRIMARY KEY,
    export_month DATE NOT NULL,
    file_path VARCHAR(255),
    record_count INT,
    bureau_name VARCHAR(50), -- EQUIFAX, TRANSUNION, EXPERIAN
    sent_date TIMESTAMP,
    confirmation_received BOOLEAN,
    error_message TEXT,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE risk_assessment_rules (
    id BIGSERIAL PRIMARY KEY,
    rule_name VARCHAR(100),
    score_component VARCHAR(50), -- account_age, payment_history, etc.
    min_value DECIMAL(10,2),
    max_value DECIMAL(10,2),
    score_points INT,
    active BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_installment_plans_customer ON installment_plans(customer_id);
CREATE INDEX idx_installment_plans_status ON installment_plans(status);
CREATE INDEX idx_installment_schedules_due_date ON installment_schedules(due_date);
CREATE INDEX idx_installment_schedules_status ON installment_schedules(status);
CREATE INDEX idx_charge_records_plan ON charge_records(plan_id, execution_date);
CREATE INDEX idx_collections_cases_status ON collections_cases(case_status);
```

---

## Redis Data Structures

```
Key Pattern                            Type        TTL     Purpose
──────────────────────────────────────────────────────────────────
due_dates:{date}                       ZSet        1d      Installments due on date (sorted by plan_id)
charge_lock:{schedule_id}              String      10min   Prevent concurrent charge attempts
retry_queue:{date}                     List        1d      Queue of failed charges to retry
pending_charges:{plan_id}              List        ongoing List of pending installment IDs
risk_decision_cache:{customer_id}      String      24h     Cached risk score (TTL: prevent rescore within 24h)
charge_tokens:{plan_id}                String      7d      Tokenized card for idempotent retries
collections_queue:{date}               ZSet        7d      Cases escalated to collections (score = days_late)
late_fees_applied:{plan_id}            Hash        7d      Applied fees per installment
delinquency_report:{date}              String      30d     Daily delinquency snapshot (JSON)
```

---

## Kafka Event Flow

```
Topic: installment_plans_created (20 partitions, key = customer_id)
├─ Payload: { plan_id, order_id, total_amount, risk_score, decision }
└─ Consumers: AnalyticsService, CreditBureauService, NotificationService

Topic: installment_charges_due (20 partitions, key = plan_id)
├─ Published by: Scheduled job query (finds due installments)
├─ Payload: { schedule_id, plan_id, amount, due_date, attempt_number }
└─ Consumer: InstallmentChargeService (Quartz job listener)

Topic: installment_charged_success (10 partitions, key = plan_id)
├─ Published by: ChargeService
├─ Payload: { schedule_id, amount, gateway_tx_id, charged_at }
└─ Consumers: NotificationService, AnalyticsService, CreditBureauService

Topic: installment_charged_failed (15 partitions, key = plan_id)
├─ Payload: { schedule_id, amount, failure_reason, retry_date }
├─ Consumers: RetryScheduler, NotificationService
└─ Retention: 30 days (audit trail for collections)

Topic: installment_missed_payment (10 partitions, key = plan_id)
├─ Published by: ChargeService (after 3rd failed attempt)
├─ Payload: { schedule_id, plan_id, late_fee_cents }
└─ Consumers: CollectionsService, NotificationService, CreditBureauService

Topic: installment_paid_off (10 partitions, key = plan_id)
├─ Published by: EarlyPayoffService
├─ Payload: { plan_id, payoff_amount, discount }
└─ Consumer: NotificationService, CreditBureauService

Topic: credit_bureau_reports (1 partition)
├─ Published by: CreditBureauReporter
├─ Payload: { export_id, month, record_count, bureau_names }
└─ Consumer: ReportingService (tracking)
```

---

## Implementation Code

### 1. InstallmentRiskAssessor

```java
package com.payment.installment.service;

import org.springframework.stereotype.Service;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;
import java.util.*;

@Service
public class InstallmentRiskAssessor {

    private final CustomerRepository customerRepository;
    private final OrderRepository orderRepository;
    private final ChargeRecordRepository chargeRecordRepository;
    private final RedisTemplate<String, String> redis;
    private final RiskAssessmentRuleRepository ruleRepository;

    private static final int APPROVAL_THRESHOLD = 60;
    private static final int MANUAL_REVIEW_THRESHOLD_LOW = 50;
    private static final int MANUAL_REVIEW_THRESHOLD_HIGH = 59;

    public RiskDecision assessRisk(long customerId, long planAmountCents, LocalDate firstChargeDate) {
        // Check cache first
        String cached = redis.opsForValue().get("risk_decision_cache:" + customerId);
        if (cached != null) {
            return RiskDecision.fromJson(cached);
        }

        Customer customer = customerRepository.findById(customerId)
                .orElseThrow(() -> new IllegalArgumentException("Customer not found"));

        RiskScore score = new RiskScore();

        // Component 1: Account Age (0-20 points)
        long accountAgeDays = ChronoUnit.DAYS.between(customer.getCreatedAt().toLocalDate(), LocalDate.now());
        score.accountAgeScore = calculateAccountAgeScore(accountAgeDays);

        // Component 2: Payment History (0-25 points)
        List<ChargeRecord> history = chargeRecordRepository.findByCustomerIdOrderByCreatedAtDesc(
                customerId,
                PageRequest.of(0, 100)
        ).getContent();
        long successCount = history.stream().filter(c -> ChargeStatus.SUCCESS.equals(c.getStatus())).count();
        double successRate = history.isEmpty() ? 0 : (successCount / (double) history.size());
        score.paymentHistoryScore = calculatePaymentHistoryScore(successRate);

        // Component 3: Return Rate (0-15 points)
        long totalOrders = orderRepository.countByCustomerId(customerId);
        long returnOrders = orderRepository.countByCustomerIdAndStatusIn(
                customerId,
                Arrays.asList(OrderStatus.RETURNED, OrderStatus.REFUNDED)
        );
        double returnRate = totalOrders == 0 ? 0 : (returnOrders / (double) totalOrders);
        score.returnRateScore = calculateReturnRateScore(returnRate);

        // Component 4: Financial Health (0-20 points)
        long outstandingBalance = calculateOutstandingBalance(customerId);
        score.financialHealthScore = calculateFinancialHealthScore(outstandingBalance);

        // Component 5: Behavioral Signals (0-20 points)
        score.behavioralScore = assessBehavioralRisks(customerId, planAmountCents, firstChargeDate);

        // Calculate final score
        int finalScore = Math.min(100,
                score.accountAgeScore + score.paymentHistoryScore +
                score.returnRateScore + score.financialHealthScore +
                score.behavioralScore
        );

        // Determine decision
        RiskDecision decision;
        if (finalScore >= APPROVAL_THRESHOLD) {
            decision = new RiskDecision(
                    RiskDecision.Status.APPROVED,
                    finalScore,
                    "Customer meets risk criteria"
            );
        } else if (finalScore >= MANUAL_REVIEW_THRESHOLD_LOW && finalScore <= MANUAL_REVIEW_THRESHOLD_HIGH) {
            decision = new RiskDecision(
                    RiskDecision.Status.MANUAL_REVIEW,
                    finalScore,
                    "Borderline risk; requires manual review"
            );
        } else {
            decision = new RiskDecision(
                    RiskDecision.Status.DENIED,
                    finalScore,
                    "Risk score below approval threshold"
            );
        }

        // Cache decision for 24 hours
        redis.opsForValue().set(
                "risk_decision_cache:" + customerId,
                decision.toJson(),
                Duration.ofHours(24)
        );

        return decision;
    }

    private int calculateAccountAgeScore(long ageInDays) {
        if (ageInDays < 30) return 0;
        if (ageInDays < 90) return 5;
        if (ageInDays < 180) return 10;
        if (ageInDays < 365) return 15;
        return 20;
    }

    private int calculatePaymentHistoryScore(double successRate) {
        if (successRate >= 1.0) return 25;
        if (successRate >= 0.99) return 20;
        if (successRate >= 0.90) return 15;
        if (successRate >= 0.75) return 10;
        if (successRate >= 0.50) return 5;
        return 0;
    }

    private int calculateReturnRateScore(double returnRate) {
        if (returnRate > 0.10) return 0;
        if (returnRate > 0.05) return 5;
        if (returnRate > 0.02) return 10;
        return 15;
    }

    private int calculateFinancialHealthScore(long outstandingCents) {
        if (outstandingCents > 500_000) return 0; // >$5K
        if (outstandingCents > 200_000) return 5; // >$2K
        if (outstandingCents > 50_000) return 10; // >$500
        return 15; // Low outstanding
    }

    private int assessBehavioralRisks(long customerId, long planAmount, LocalDate chargeDate) {
        int score = 20; // Start at max

        // Check for velocity (multiple applications in same day)
        long applicationsToday = orderRepository.countByCustomerIdAndCreatedAtAfter(
                customerId,
                LocalDateTime.now().minusHours(24)
        );
        if (applicationsToday > 3) {
            score -= 20; // Red flag
        }

        // Device/IP consistency check (mock)
        // In production: integrate with fraud detection service
        // if (deviceMismatch) score -= 5;

        // Known fraud signals
        boolean isBlacklisted = fraudServiceClient.isBlacklisted(customerId);
        if (isBlacklisted) {
            score -= 20;
        }

        return Math.max(0, score);
    }

    private long calculateOutstandingBalance(long customerId) {
        List<InstallmentPlan> activePlans = installmentPlanRepository.findByCustomerIdAndStatus(
                customerId,
                InstallmentStatus.ACTIVE
        );

        return activePlans.stream()
                .flatMap(plan -> installmentScheduleRepository.findByPlanIdAndStatusNotIn(
                        plan.getId(),
                        Arrays.asList(InstallmentStatus.PAID)
                ).stream())
                .mapToLong(schedule -> schedule.getAmountCents())
                .sum();
    }
}

class RiskScore {
    int accountAgeScore;
    int paymentHistoryScore;
    int returnRateScore;
    int financialHealthScore;
    int behavioralScore;
}

class RiskDecision {
    enum Status { APPROVED, DENIED, MANUAL_REVIEW }

    Status status;
    int score;
    String reason;

    RiskDecision(Status status, int score, String reason) {
        this.status = status;
        this.score = score;
        this.reason = reason;
    }

    String toJson() {
        return String.format("{\"status\":\"%s\",\"score\":%d,\"reason\":\"%s\"}", status, score, reason);
    }

    static RiskDecision fromJson(String json) {
        // Parse JSON
        return new RiskDecision(Status.APPROVED, 75, "Cached");
    }
}
```

### 2. InstallmentChargeJob (Quartz Scheduler)

```java
package com.payment.installment.batch;

import org.quartz.*;
import org.springframework.stereotype.Component;
import org.springframework.beans.factory.annotation.Autowired;
import java.time.LocalDate;
import java.util.*;

@Component
public class InstallmentChargeJob implements Job {

    @Autowired
    private InstallmentScheduleRepository scheduleRepository;

    @Autowired
    private PaymentChargeService chargeService;

    @Autowired
    private KafkaTemplate<String, InstallmentEvent> kafkaTemplate;

    @Autowired
    private InstallmentMetrics metrics;

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        logger.info("InstallmentChargeJob started");
        LocalDate today = LocalDate.now();

        try {
            // Find all installments due today
            List<InstallmentSchedule> dueInstallments = scheduleRepository
                    .findByDueDateAndStatusIn(
                            today,
                            Arrays.asList(
                                    InstallmentStatus.PENDING,
                                    InstallmentStatus.FAILED_ATTEMPT_1,
                                    InstallmentStatus.FAILED_ATTEMPT_2
                            )
                    );

            logger.info("Found " + dueInstallments.size() + " due installments");

            // Process in parallel batches
            dueInstallments.parallelStream()
                    .forEach(schedule -> processInstallment(schedule));

            logger.info("InstallmentChargeJob completed successfully");

        } catch (Exception e) {
            logger.error("InstallmentChargeJob failed", e);
            throw new JobExecutionException(e);
        }
    }

    private void processInstallment(InstallmentSchedule schedule) {
        try {
            // Optimistic lock check
            long versionBefore = schedule.getVersion();
            schedule = scheduleRepository.findByIdForUpdate(schedule.getId())
                    .orElseThrow();

            if (schedule.getVersion() != versionBefore) {
                logger.warn("Schedule version mismatch; skipping: " + schedule.getId());
                return;
            }

            // Increment attempt count
            int attemptNumber = schedule.getAttemptCount() + 1;
            schedule.setAttemptCount(attemptNumber);
            schedule.setVersion(schedule.getVersion() + 1);

            // Charge payment gateway
            PaymentChargeResult result = chargeService.chargeInstallment(
                    schedule.getPlanId(),
                    schedule.getId(),
                    schedule.getAmountCents() + schedule.getLateFeesCents(),
                    attemptNumber
            );

            // Record charge
            ChargeRecord record = new ChargeRecord();
            record.setScheduleId(schedule.getId());
            record.setPlanId(schedule.getPlanId());
            record.setAttemptNumber(attemptNumber);
            record.setExecutionDate(LocalDate.now());
            record.setAmountCents(schedule.getAmountCents() + schedule.getLateFeesCents());
            record.setGatewayTransactionId(result.getTransactionId());

            if (result.isSuccess()) {
                schedule.setStatus(InstallmentStatus.PAID);
                schedule.setChargedAt(LocalDateTime.now());
                record.setStatus(ChargeStatus.SUCCESS);

                // Publish success event
                kafkaTemplate.send(
                        "installment_charged_success",
                        String.valueOf(schedule.getPlanId()),
                        new InstallmentEvent(schedule.getId(), InstallmentEvent.Type.CHARGED_SUCCESS)
                );

                metrics.recordChargeSuccess(schedule.getAmountCents());

            } else {
                record.setStatus(ChargeStatus.FAILED);
                record.setFailureMessage(result.getFailureMessage());
                record.setLastFailedAt(LocalDateTime.now());

                // Determine next status and retry date
                if (attemptNumber == 1) {
                    schedule.setStatus(InstallmentStatus.FAILED_ATTEMPT_1);
                    record.setRetryScheduledDate(LocalDate.now().plusDays(1)); // D+1
                } else if (attemptNumber == 2) {
                    schedule.setStatus(InstallmentStatus.FAILED_ATTEMPT_2);
                    record.setRetryScheduledDate(LocalDate.now().plusDays(3)); // D+3
                } else if (attemptNumber == 3) {
                    schedule.setStatus(InstallmentStatus.MISSED);
                    schedule.setLastFailedAt(LocalDateTime.now());

                    // Apply late fee
                    applyLateFee(schedule);

                    // Publish missed payment event
                    kafkaTemplate.send(
                            "installment_missed_payment",
                            String.valueOf(schedule.getPlanId()),
                            new InstallmentEvent(schedule.getId(), InstallmentEvent.Type.MISSED_PAYMENT)
                    );

                    metrics.recordMissedPayment();
                } else {
                    // Escalate to collections after 3 retries
                    escalateToCollections(schedule.getPlanId());
                }

                metrics.recordChargeFailed(result.getFailureReason());
            }

            chargeRecordRepository.save(record);
            scheduleRepository.save(schedule);

        } catch (Exception e) {
            logger.error("Failed to process installment: " + schedule.getId(), e);
            metrics.recordChargeError();
        }
    }

    private void applyLateFee(InstallmentSchedule schedule) {
        long feeCents = Math.min(
                700, // $7 flat
                (schedule.getAmountCents() * 5) / 100 // 5% of installment
        );

        if (feeCents > 0) {
            LateFee lateFee = new LateFee();
            lateFee.setScheduleId(schedule.getId());
            lateFee.setFeeCents(feeCents);
            lateFee.setFeeType(LateFeeType.LATE_FEE);
            lateFee.setAppliedDate(LocalDate.now());

            lateFeeRepository.save(lateFee);
            schedule.setLateFeesCents(feeCents);
        }
    }

    private void escalateToCollections(long planId) {
        CollectionsCase case_ = new CollectionsCase();
        case_.setPlanId(planId);
        case_.setCaseStatus(CollectionsCaseStatus.OPEN);
        case_.setEscalatedDate(LocalDate.now());

        collectionsCaseRepository.save(case_);

        kafkaTemplate.send(
                "installment_collections_escalated",
                String.valueOf(planId),
                new InstallmentEvent(planId, InstallmentEvent.Type.ESCALATED_TO_COLLECTIONS)
        );
    }
}

// Quartz Configuration
@Configuration
public class QuartzConfig {

    @Bean
    public JobDetail installmentChargeJobDetail() {
        return JobBuilder.newJob(InstallmentChargeJob.class)
                .withIdentity("installmentChargeJob")
                .storeDurably()
                .build();
    }

    @Bean
    public Trigger installmentChargeTrigger() {
        return TriggerBuilder.newTrigger()
                .forJob(installmentChargeJobDetail())
                .withIdentity("installmentChargeTrigger")
                .withSchedule(CronScheduleBuilder.cronSchedule("0 0 0,6,12,18 * * *")) // Every 6 hours
                .build();
    }
}
```

### 3. EarlyPayoffService

```java
package com.payment.installment.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

@Service
public class EarlyPayoffService {

    private final InstallmentPlanRepository planRepository;
    private final InstallmentScheduleRepository scheduleRepository;
    private final PaymentChargeService chargeService;
    private final KafkaTemplate<String, InstallmentEvent> kafkaTemplate;

    @Transactional
    public EarlyPayoffQuote getPayoffQuote(long planId) {
        InstallmentPlan plan = planRepository.findById(planId)
                .orElseThrow(() -> new IllegalArgumentException("Plan not found"));

        List<InstallmentSchedule> unpaid = scheduleRepository.findByPlanIdAndStatusNotIn(
                planId,
                Arrays.asList(InstallmentStatus.PAID)
        );

        long remainingBalance = unpaid.stream()
                .mapToLong(s -> s.getAmountCents())
                .sum();

        // Calculate prepayment discount: 0.5% per month
        long monthsRemaining = 0;
        for (InstallmentSchedule schedule : unpaid) {
            long daysUntilDue = ChronoUnit.DAYS.between(LocalDate.now(), schedule.getDueDate());
            long monthsUntil = (daysUntilDue / 30) + 1;
            monthsRemaining = Math.max(monthsRemaining, monthsUntil);
        }

        long discountAmount = (remainingBalance * monthsRemaining * 5) / 1000; // 0.5% per month
        long payoffAmount = remainingBalance - discountAmount;

        return new EarlyPayoffQuote(
                planId,
                remainingBalance,
                discountAmount,
                payoffAmount,
                monthsRemaining
        );
    }

    @Transactional
    public void executeEarlyPayoff(long planId, String idempotencyKey) {
        InstallmentPlan plan = planRepository.findById(planId)
                .orElseThrow();

        EarlyPayoffQuote quote = getPayoffQuote(planId);

        // Charge payoff amount
        PaymentChargeResult result = chargeService.chargeInstallment(
                planId,
                -1, // Special schedule ID for early payoff
                quote.getPayoffAmount(),
                999 // Special attempt number
        );

        if (result.isSuccess()) {
            // Mark all unpaid installments as PAID
            List<InstallmentSchedule> unpaid = scheduleRepository.findByPlanIdAndStatusNotIn(
                    planId,
                    Arrays.asList(InstallmentStatus.PAID)
            );

            for (InstallmentSchedule schedule : unpaid) {
                schedule.setStatus(InstallmentStatus.PAID);
                schedule.setChargedAt(LocalDateTime.now());
                scheduleRepository.save(schedule);
            }

            // Mark plan as PAID_OFF
            plan.setStatus(InstallmentStatus.PAID_OFF);
            planRepository.save(plan);

            // Publish event
            kafkaTemplate.send(
                    "installment_paid_off",
                    String.valueOf(planId),
                    new InstallmentEvent(planId, InstallmentEvent.Type.PAID_OFF,
                            "discount_cents", String.valueOf(quote.getDiscountAmount()))
            );

        } else {
            throw new PaymentException("Early payoff charge failed: " + result.getFailureMessage());
        }
    }
}

class EarlyPayoffQuote {
    long planId;
    long remainingBalance;
    long discountAmount;
    long payoffAmount;
    long monthsRemaining;

    // Constructor, getters
}
```

### 4. CreditBureauReporter

```java
package com.payment.installment.reporting;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import java.io.*;
import java.time.YearMonth;

@Service
public class CreditBureauReporter {

    private final InstallmentPlanRepository planRepository;
    private final InstallmentScheduleRepository scheduleRepository;
    private final CreditBureauExportRepository exportRepository;
    private final SFTPClient sftpClient;

    @Scheduled(cron = "0 0 1 * * *") // 1st of month at 00:00
    public void generateAndSubmitMetro2() {
        YearMonth month = YearMonth.now().minusMonths(1); // Previous month
        logger.info("Generating Metro 2 export for " + month);

        // Fetch all plans for the month
        List<InstallmentPlan> plans = planRepository.findByCreatedAtBetween(
                month.atDay(1).atStartOfDay(),
                month.atEndOfMonth().atTime(23, 59, 59)
        );

        try {
            // Generate Metro 2 file
            String metro2Content = generateMetro2File(plans);
            String filename = "metro2_" + month.getYear() + "_" + month.getMonthValue() + ".txt";

            // Upload to SFTP
            submitToSFTP(filename, metro2Content, "equifax");
            submitToSFTP(filename, metro2Content, "transunion");
            submitToSFTP(filename, metro2Content, "experian");

            // Log export
            CreditBureauExport export = new CreditBureauExport();
            export.setExportMonth(month.atDay(1).toLocalDate());
            export.setFilePath("s3://credit-bureau-exports/" + filename);
            export.setRecordCount(plans.size());
            export.setBureauName("EQUIFAX,TRANSUNION,EXPERIAN");
            export.setSentDate(LocalDateTime.now());
            exportRepository.save(export);

            logger.info("Metro 2 export completed for " + month);

        } catch (Exception e) {
            logger.error("Metro 2 export failed for " + month, e);
            throw new RuntimeException(e);
        }
    }

    private String generateMetro2File(List<InstallmentPlan> plans) {
        StringBuilder sb = new StringBuilder();

        // Metro 2 Header
        sb.append(String.format("%06d%-40s%010d%8s%1s%1s%4s\n",
                0, // Record ID (0 = header)
                "PAY_IN_4_INC",
                plans.size(),
                LocalDate.now().toString(),
                "0", // Blank
                "1", // Version
                ""   // Filler
        ));

        // Data records (one per plan)
        for (InstallmentPlan plan : plans) {
            List<InstallmentSchedule> schedules = scheduleRepository.findByPlanId(plan.getId());

            // Metro 2 Account Record
            String metro2Record = formatMetro2Record(plan, schedules);
            sb.append(metro2Record);
        }

        // Trailer record
        long totalBalance = plans.stream()
                .mapToLong(InstallmentPlan::getTotalAmountCents)
                .sum();

        sb.append(String.format("%06d%-40s%010d%8s\n",
                999, // Record ID (999 = trailer)
                "PAY_IN_4_INC",
                plans.size(),
                LocalDate.now().toString()
        ));

        return sb.toString();
    }

    private String formatMetro2Record(InstallmentPlan plan, List<InstallmentSchedule> schedules) {
        // Metro 2 format per CREDIT_REPORTING_AGENCY_ALLIANCE guidelines
        // Account Type: I (Installment)
        // Payment Status Codes: 0 (Current), 1 (30 days late), etc.

        InstallmentStatus currentStatus = plan.getStatus();
        String statusCode = mapInstallmentStatusToMetro2(currentStatus);

        long remainingBalance = schedules.stream()
                .filter(s -> !InstallmentStatus.PAID.equals(s.getStatus()))
                .mapToLong(InstallmentSchedule::getAmountCents)
                .sum();

        return String.format(
                "%06d%-30s%-1s%010d%8s%-1s%1s\n",
                1, // Data record
                plan.getOrderId(),
                "I", // Installment account
                remainingBalance,
                plan.getCreatedAt().toLocalDate().toString(),
                statusCode,
                ""  // Filler
        );
    }

    private String mapInstallmentStatusToMetro2(InstallmentStatus status) {
        return switch (status) {
            case ACTIVE -> "0"; // Current
            case MISSED -> "1"; // 30 days late
            case COLLECTIONS -> "3"; // 60 days late
            case PAID, PAID_OFF -> "0"; // Current (settled)
            default -> "9"; // Unknown
        };
    }

    private void submitToSFTP(String filename, String content, String bureau) throws Exception {
        String remotePath = "/incoming/" + bureau + "/" + filename;
        sftpClient.uploadFile(
                content.getBytes(),
                remotePath,
                "sftp://" + sftpConfig.getHost(bureau)
        );
        logger.info("Submitted Metro 2 to " + bureau);
    }
}
```

---

## Failure Scenarios

| Scenario | Cause | Mitigation |
|---|---|---|
| **Payment charge fails (soft)** | Insufficient funds | Retry D+1, D+3, D+7; notify customer |
| **Payment charge fails (hard)** | Card closed, fraud | Move to COLLECTIONS immediately; don't retry |
| **Quartz job hangs** | Database lock contention | Add job timeout; use job listener for cleanup |
| **Credit bureau file rejected** | Format error | Validate Metro 2 before SFTP; alert ops |
| **Duplicate charges** | Job executed twice | Idempotency key (schedule_id + attempt_number) |
| **Risk score inconsistency** | Cache stale | Nightly cache refresh; 24h TTL |
| **Late fee not applied** | Job skip | Nightly audit job verifies late fees |

---

## Scaling Strategy

| Component | Scale-out Method | Bottleneck |
|---|---|---|
| **Installment Plan Service** | Horizontal (load balancer) | Risk assessment (3rd-party APIs) |
| **Quartz Charge Job** | Database-driven partitioning (by date range) | Payment gateway rate limits (500/sec) |
| **Credit Bureau Reporter** | Batch monthly; separate scheduler | SFTP bandwidth (100 MB/file) |
| **Collections Service** | Async processing; Kafka-driven | External agency integrations |

---

## Monitoring & Alerting

```java
package com.payment.installment.monitoring;

import io.micrometer.core.instrument.*;

@Service
public class InstallmentMetrics {

    private final MeterRegistry meterRegistry;

    public void recordPlanCreated(RiskDecision.Status decision, int riskScore) {
        meterRegistry.counter("installment.plan.created", "decision", decision.toString())
                .increment();
        meterRegistry.gauge("installment.risk.score", (double) riskScore);
    }

    public void recordChargeSuccess(long amountCents) {
        meterRegistry.counter("installment.charge.success")
                .increment();
        meterRegistry.timer("installment.charge.duration")
                .record(Duration.ofMillis(150)); // Mock
    }

    public void recordChargeFailed(String reason) {
        meterRegistry.counter("installment.charge.failed", "reason", reason)
                .increment();
    }

    public void recordMissedPayment() {
        meterRegistry.counter("installment.missed_payment")
                .increment();
    }

    public void recordDelinquency(int days_late) {
        meterRegistry.gauge("installment.delinquency.days", (double) days_late);
    }
}
```

---

## Summary Cheat Sheet

| Concept | Key Takeaway |
|---|---|
| **Risk Scoring** | Account age + payment history + return rate + financial health + behavioral (0-100); ≥60 approval |
| **Payment Schedule** | 4 installments: 25% each; due at weeks 0, 2, 4, 6 |
| **Charge Retries** | Max 3 attempts: D+1 (soft), D+3, D+7; hard decline → COLLECTIONS immediately |
| **Late Fees** | MIN($7 flat, 5% of installment); max $50; applied at D+1 of missed payment |
| **Early Payoff** | Remaining balance - (months_remaining × 0.5% discount); marked PAID_OFF |
| **Credit Bureau** | Monthly Metro 2 format; status codes (0=current, 1=30d late, 3=60d late); SFTP to 3 bureaus |
| **Collections** | After 3 failed retries; state COLLECTIONS; escalate to external agency after D+30 |
| **Idempotency** | Unique(schedule_id, attempt_number, execution_date) prevents duplicate charges |
| **Quartz Job** | Cron trigger every 6 hours; parallel processing; 100K charges/batch |
| **Scaling** | Horizontal service instances; database partitioning by date range; async collections |

