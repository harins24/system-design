---
title: Subscription Billing System — System Design Deep Dive
layout: default
---

# Subscription Billing System — Deep Dive Design

> **Scenario:** Handle 100K active subscriptions with monthly/annual cycles, prorated billing on plan changes, metered billing (pay per API call), dunning for failed payments (3 retries over 2 weeks), auto plan upgrades/downgrades, billing cycles (calendar month vs anniversary dates).
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
- Support monthly and annual subscription plans with prorated charges
- Metered billing: accumulate usage events, aggregate at cycle end
- Handle plan changes (upgrade/downgrade) mid-cycle with prorations
- Implement dunning: retry failed payments 3x over 14 days
- Auto-upgrade/downgrade based on usage thresholds
- Generate invoices and payment receipts
- Prevent double-charging and missing charges
- Support calendar-month and anniversary-date billing cycles
- 100K active subscriptions, 50K new signups/month

### Non-Functional Requirements
- Billing must be **idempotent**: re-running failed jobs must produce same result
- Guaranteed delivery: no missed invoices or double charges
- Sub-100ms invoice lookup latency
- Real-time usage aggregation within 5 minutes of event ingestion
- Horizontal scaling of billing scheduler across 10+ nodes
- Audit trail for all billing events (immutable log)

---

## Capacity Estimation

| Metric | Calculation | Value |
|--------|-------------|-------|
| **Active Subscriptions** | Given | 100,000 |
| **Billing Jobs/Day** | 100K ÷ 30 (avg cycle) + 50K new | ~3,333 + 1,667 = 5,000 jobs/day |
| **Billing Jobs/Second** | 5,000 ÷ 86,400 | ~0.06 jobs/sec (bursty: 1–2 jobs/sec peaks) |
| **Usage Events/Day** | 100K subs × 10 events/sub (avg) | 1,000,000 events/day |
| **Usage Events/Second** | 1,000,000 ÷ 86,400 | ~11.6 events/sec |
| **Invoice Records/Month** | 100K subscriptions | 100,000 invoices/month |
| **Dunning Retries/Month** | 5% failure rate × 3 retries × 100K | ~15,000 retries/month |
| **Storage: PostgreSQL (annual)** | 100K subs × 12 months × (subscription + invoice + log) | ~2–3 GB |
| **Storage: MongoDB (usage)** | 1M events/day × 365 × 100 bytes | ~36 GB/year |
| **Redis Memory (active)** | Usage buffers + rate limits + locks | ~500 MB |

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                     Client Applications                       │
│         (Web, Mobile, Admin Portal)                           │
└──────────────────────┬───────────────────────────────────────┘
                       │
     ┌─────────────────┼────────────────────┐
     │                 │                    │
┌────▼────────┐  ┌────▼──────────┐  ┌─────▼─────────┐
│ API Gateway │  │ Usage Tracker  │  │ Billing Admin │
│  (Plan mgmt)│  │  (Events→      │  │   (Manual     │
│             │  │   Kafka)       │  │   adjusts)    │
└────┬────────┘  └────┬──────────┘  └─────┬─────────┘
     │                │                    │
     └────────────────┼────────────────────┘
                      │
        ┌─────────────┴──────────────┐
        │                            │
    ┌───▼──────────────────┐   ┌────▼───────────────────┐
    │  Kafka Cluster       │   │  PostgreSQL Primary    │
    ├──────────────────────┤   ├───────────────────────┤
    │ usage-events topic   │   │ subscriptions table    │
    │ plan-change topic    │   │ invoices table         │
    │ billing-result topic │   │ usage_records table    │
    │ dunning-event topic  │   │ billing_log table      │
    │ payment-update topic │   │ dunning_attempts table │
    └──────┬───────────────┘   └────────┬──────────────┘
           │                           │
     ┌─────┴───────────┐       ┌───────┴────────┐
     │                 │       │                │
┌────▼──────────┐  ┌──▼──────────┐  ┌──────────▼──┐
│ Billing       │  │  MongoDB    │  │   Redis    │
│ Scheduler Pod │  │  (Analytics)│  │ (Cache +   │
│ (Partitioned)│  │             │  │  Locks)    │
└─────┬────────┘  └─────────────┘  └────────────┘
      │
  ┌───┴──────────────────────────────────────┐
  │  Quartz Job Scheduler                    │
  │  - 10 nodes (subscription ID % 10)       │
  │  - Each job: BillingService.execute()    │
  │  - Idempotent via unique constraint      │
  └────────────────────────────────────────┘
```

---

## Core Design Questions Answered

### Q1: How do you schedule recurring billing for 100K subscriptions?

**Answer:** Partition subscriptions by ID modulo across 10 Quartz scheduler nodes. Each node owns 10K subscriptions. Schedule one job per subscription per billing cycle. Use database-backed Quartz scheduler (`QRTZ_TRIGGERS` table in PostgreSQL) for durability.

**Partitioning formula:**
```
node_id = subscription_id % num_nodes
```

Each job is **idempotent**: has unique constraint `UNIQUE(subscription_id, billing_period)` in `billing_log`. If job re-runs, duplicate insert fails safely; only first run executes billing logic.

---

### Q2: How do you calculate prorated amounts for plan changes?

**Answer:** Prorated amount = `(remaining_days / days_in_cycle) × (new_price - old_price)`

**Example:**
- Old plan: $100/month (30 days)
- New plan: $150/month (30 days)
- Change on day 20 of cycle (10 days remaining)
- Prorated charge = (10/30) × ($150 - $100) = $16.67

**For downgrade:** Immediate credit to account; don't charge, deduct from next invoice.
**For upgrade:** Charge immediately, add to account balance, deduct from next invoice.

---

### Q3: How do you implement metered billing?

**Answer:**
1. Usage events → Kafka topic `usage-events`
2. UsageAggregator consumes events, stores raw events in MongoDB
3. At cycle end: query MongoDB for usage in `[cycle_start, cycle_end)`, apply aggregation (SUM, MAX, AVG), multiply by rate card
4. Store aggregated usage in PostgreSQL `usage_records` table for audit

**Aggregation examples:**
- **SUM**: API calls: sum all events, multiply by $0.01 per call
- **MAX**: Concurrent users: take max seen in month, multiply by $10/user
- **AVG**: Storage: average daily GB, multiply by $0.05/GB/day × days_in_month

---

### Q4: How do you handle failed payments and retries?

**Answer:** State machine + exponential backoff:
- **Attempt 1:** Day 1 of billing cycle failure → Retry T+1 day
- **Attempt 2:** If retry fails → Retry T+3 days
- **Attempt 3:** If retry fails → Retry T+7 days
- **Final failure:** Mark as PAST_DUE after T+14; suspend service if balance > threshold

**State transitions:**
```
ACTIVE → PAST_DUE → SUSPENDED → CANCELLED
```

Store dunning attempt logs with timestamps; use exponential backoff jitter to avoid thundering herd on retry times.

---

### Q5: How do you prevent billing errors (double-charging, missing charges)?

**Answer:**
1. **Double-charge prevention:** Unique constraint `UNIQUE(subscription_id, billing_period)` in `billing_log` table. Idempotent job: if insert fails, no duplicate charge.
2. **Missing charges:** Audit trail in PostgreSQL. Daily reconciliation job queries `billing_log` for expected vs actual; alerts if gap > 0.
3. **Ledger entries:** Double-entry bookkeeping in separate `ledger_entries` table (debit/credit pairs). Ensures sum(debit) = sum(credit).

---

### Q6: How do you generate invoices and receipts?

**Answer:**
1. After billing job completes: insert into `invoices` table
2. Async trigger Kafka event `invoice-generated`
3. InvoiceGenerationService consumes event, fetches invoice + subscription + usage details
4. Render Thymeleaf HTML template → convert to PDF using iText/OpenPDF library
5. Upload PDF to S3/object storage; store URL in `invoices.pdf_url`
6. Send email to customer with PDF link

---

## Microservices Breakdown

| Service | Responsibility | Tech | Throughput |
|---------|-----------------|------|-----------|
| **Billing Scheduler** | Trigger billing jobs, partition subscriptions | Quartz + Spring Boot | 10 nodes × 500 jobs/hour = 5K jobs/hour |
| **Billing Engine** | Execute billing logic, compute charges | Spring Boot + Transactional | ~100 jobs/sec (parallelized) |
| **Usage Aggregator** | Consume usage events, compute metered charges | Kafka Consumer + MongoDB | ~50 events/sec |
| **Dunning Manager** | Manage payment retries, state transitions | Spring Boot + Scheduler | ~20 dunning events/sec |
| **Invoice Generator** | Create PDF invoices, send emails | Spring Boot + Async Task | ~50 invoices/sec |
| **Analytics Engine** | Real-time MRR, churn, revenue dashboards | Kafka Consumer + MongoDB | ~100 events/sec |

---

## Database Design (DDL)

```sql
-- PostgreSQL Schema

CREATE TABLE subscriptions (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    plan_id BIGINT NOT NULL,
    status VARCHAR(50) NOT NULL, -- ACTIVE, PAUSED, CANCELLED, PAST_DUE
    current_price_cents BIGINT NOT NULL,
    billing_cycle_type VARCHAR(20) NOT NULL, -- CALENDAR_MONTH, ANNIVERSARY
    cycle_day_of_month INT, -- 1-28 for calendar month
    billing_period_start DATE NOT NULL,
    billing_period_end DATE NOT NULL,
    next_billing_date DATE,
    auto_renew BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(customer_id, plan_id) -- Only one active plan per customer
);
CREATE INDEX idx_subscriptions_next_billing_date ON subscriptions(next_billing_date);
CREATE INDEX idx_subscriptions_status ON subscriptions(status);

CREATE TABLE billing_log (
    id BIGSERIAL PRIMARY KEY,
    subscription_id BIGINT NOT NULL REFERENCES subscriptions(id),
    billing_period_start DATE NOT NULL,
    billing_period_end DATE NOT NULL,
    base_charge_cents BIGINT NOT NULL,
    prorated_adjustment_cents BIGINT, -- Positive or negative
    metered_charge_cents BIGINT DEFAULT 0,
    credit_applied_cents BIGINT DEFAULT 0,
    total_charge_cents BIGINT NOT NULL,
    status VARCHAR(50) NOT NULL, -- PENDING, INVOICED, PAID, FAILED
    payment_method_id BIGINT,
    paid_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(subscription_id, billing_period_start) -- Idempotency key
);
CREATE INDEX idx_billing_log_subscription_period ON billing_log(subscription_id, billing_period_start);

CREATE TABLE invoices (
    id BIGSERIAL PRIMARY KEY,
    billing_log_id BIGINT NOT NULL REFERENCES billing_log(id),
    invoice_number VARCHAR(50) UNIQUE NOT NULL,
    customer_id BIGINT NOT NULL,
    total_amount_cents BIGINT NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    issued_date DATE NOT NULL,
    due_date DATE NOT NULL,
    pdf_url VARCHAR(500),
    status VARCHAR(50) DEFAULT 'DRAFT', -- DRAFT, ISSUED, PAID, OVERDUE, CANCELLED
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(customer_id, invoice_number)
);
CREATE INDEX idx_invoices_customer ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(status);

CREATE TABLE usage_records (
    id BIGSERIAL PRIMARY KEY,
    subscription_id BIGINT NOT NULL REFERENCES subscriptions(id),
    metric_type VARCHAR(50) NOT NULL, -- api_calls, storage_gb, concurrent_users
    billing_period_start DATE NOT NULL,
    billing_period_end DATE NOT NULL,
    aggregation_type VARCHAR(20) NOT NULL, -- SUM, MAX, AVG
    aggregated_value DECIMAL(18, 2) NOT NULL,
    unit_price_cents BIGINT NOT NULL,
    total_metered_charge_cents BIGINT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(subscription_id, metric_type, billing_period_start)
);
CREATE INDEX idx_usage_records_subscription ON usage_records(subscription_id, billing_period_start);

CREATE TABLE dunning_attempts (
    id BIGSERIAL PRIMARY KEY,
    subscription_id BIGINT NOT NULL REFERENCES subscriptions(id),
    billing_log_id BIGINT NOT NULL REFERENCES billing_log(id),
    attempt_number INT NOT NULL, -- 1, 2, 3
    scheduled_retry_date DATE NOT NULL,
    executed_at TIMESTAMP,
    status VARCHAR(50) NOT NULL, -- PENDING, SUCCEEDED, FAILED, CANCELLED
    failure_reason VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(billing_log_id, attempt_number)
);
CREATE INDEX idx_dunning_attempts_scheduled ON dunning_attempts(scheduled_retry_date);

CREATE TABLE ledger_entries (
    id BIGSERIAL PRIMARY KEY,
    subscription_id BIGINT NOT NULL REFERENCES subscriptions(id),
    transaction_type VARCHAR(50) NOT NULL, -- CHARGE, PAYMENT, CREDIT, REFUND
    amount_cents BIGINT NOT NULL,
    debit_account VARCHAR(50),
    credit_account VARCHAR(50),
    reference_id BIGINT, -- billing_log_id or invoice_id
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_ledger_subscription ON ledger_entries(subscription_id);
CREATE INDEX idx_ledger_created_at ON ledger_entries(created_at);
```

---

## Redis Data Structures

```python
# Rate Limits & Locks (Key-Value)
billing:lock:{subscription_id} → JSON
  { "locked_at": 1704067200, "node_id": "billing-node-3", "version": 1 }
  TTL: 5 minutes (prevents concurrent billing runs)

# Usage Buffers (Hash: per-subscription, per-metric)
usage:buffer:{subscription_id}:{metric_type} → HASH
  { "count": 1250, "first_event_ts": 1704067200, "last_event_ts": 1704067800 }
  TTL: 1 hour (flushed to MongoDB periodically)

# Rate Cards (Hash: lookup metric pricing)
rate:cards:{plan_id} → HASH
  { "api_calls_per_unit": 0.01, "storage_gb_per_unit": 0.05, "concurrent_user_per_unit": 10.00 }
  TTL: 24 hours (refreshed from DB daily)

# Dunning Retry Queues (Sorted Set: retry schedule)
dunning:retry:queue → ZSET
  Members: {billing_log_id}, Score: {next_retry_timestamp}
  Enables efficient ZRANGEBYSCORE queries for next batch of retries
```

---

## Kafka Event Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ Kafka Topics & Event Flow                                        │
└─────────────────────────────────────────────────────────────────┘

1. usage-events (source topic)
   └─ Producer: UsageTracker service
   └─ Payload: { subscription_id, metric_type, value, timestamp }
   └─ Consumers: UsageAggregator (→ MongoDB), Analytics Engine
   └─ Partitioning: by subscription_id

2. billing-scheduled (from Quartz job)
   └─ Producer: BillingScheduler
   └─ Payload: { subscription_id, billing_period_start, billing_period_end, node_id }
   └─ Consumers: BillingEngine, DunningManager
   └─ Partitioning: by subscription_id

3. billing-result (after billing execution)
   └─ Producer: BillingEngine
   └─ Payload: { subscription_id, billing_log_id, total_charge_cents, status }
   └─ Consumers: InvoiceGenerator, DunningManager, Analytics
   └─ Partitioning: by subscription_id

4. plan-change (manual or auto plan change)
   └─ Producer: API Gateway (user changes plan) or AutoScalingService
   └─ Payload: { subscription_id, old_plan_id, new_plan_id, effective_date, prorated_amount_cents }
   └─ Consumers: BillingEngine (immediate proration), ledger
   └─ Partitioning: by subscription_id

5. dunning-event (failed payment detected)
   └─ Producer: BillingEngine (on charge failure)
   └─ Payload: { billing_log_id, subscription_id, failure_reason, next_retry_date }
   └─ Consumers: DunningManager, NotificationService
   └─ Partitioning: by subscription_id

6. invoice-generated (invoice created)
   └─ Producer: BillingEngine
   └─ Payload: { invoice_id, customer_id, subscription_id, total_cents, pdf_url }
   └─ Consumers: EmailService, AnalyticsEngine
   └─ Partitioning: by customer_id

Event Example (usage-events):
{
  "event_id": "evt-20240101-00123",
  "subscription_id": 45678,
  "metric_type": "api_calls",
  "value": 1500,
  "timestamp": 1704067200000,
  "source": "api-gateway"
}
```

---

## Implementation Code

### 1. BillingScheduler — Partition-based Quartz Scheduler

```java
package com.subscription.billing.scheduler;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ApplicationRunner;
import org.springframework.context.annotation.Bean;
import org.springframework.scheduling.quartz.SchedulerFactoryBean;
import org.springframework.stereotype.Service;
import org.quartz.*;

@Service
public class BillingScheduler {

    @Autowired private SchedulerFactoryBean schedulerFactory;
    @Autowired private BillingService billingService;

    private static final int NUM_PARTITIONS = 10;

    @Bean
    public ApplicationRunner scheduleBillingJobs() {
        return args -> {
            Scheduler scheduler = schedulerFactory.getScheduler();

            // Scan subscriptions with next_billing_date <= today
            var upcomingSubscriptions = billingService.findSubscriptionsReadyForBilling();

            for (Subscription sub : upcomingSubscriptions) {
                int partitionId = Math.toIntExact(sub.getId() % NUM_PARTITIONS);
                scheduleJobForSubscription(scheduler, sub, partitionId);
            }
        };
    }

    private void scheduleJobForSubscription(Scheduler scheduler, Subscription sub, int partitionId)
            throws SchedulerException {

        // Job key includes subscription_id for uniqueness
        JobKey jobKey = new JobKey("billing_" + sub.getId(), "partition_" + partitionId);

        // Only schedule if not already scheduled
        if (!scheduler.checkExists(jobKey)) {
            JobDetail job = JobBuilder.newJob(BillingJob.class)
                .withIdentity(jobKey)
                .usingJobData("subscriptionId", sub.getId())
                .usingJobData("billingPeriodStart", sub.getBillingPeriodStart().toString())
                .usingJobData("billingPeriodEnd", sub.getBillingPeriodEnd().toString())
                .storeDurably()
                .build();

            // Schedule for today at 2 AM (staggered by partition)
            Trigger trigger = TriggerBuilder.newTrigger()
                .withIdentity("trigger_" + sub.getId(), "partition_" + partitionId)
                .forJob(jobKey)
                .startAt(calculateTriggerTime(partitionId))
                .build();

            scheduler.scheduleJob(job, trigger);
        }
    }

    private java.util.Date calculateTriggerTime(int partitionId) {
        // Stagger jobs by partition: partition 0 @ 2 AM, partition 1 @ 2:06 AM, etc.
        java.time.LocalDateTime now = java.time.LocalDateTime.now();
        java.time.LocalDateTime scheduled = now.withHour(2).withMinute(partitionId * 6);
        return java.sql.Timestamp.valueOf(scheduled);
    }
}

@Component
public class BillingJob implements Job {

    @Autowired private BillingService billingService;
    @Autowired private IdempotencyGuard idempotencyGuard;

    @Override
    public void execute(JobExecutionContext context) throws JobExecutionException {
        JobDataMap dataMap = context.getJobDetail().getJobDataMap();
        Long subscriptionId = dataMap.getLong("subscriptionId");
        String billingPeriodStart = dataMap.getString("billingPeriodStart");
        String billingPeriodEnd = dataMap.getString("billingPeriodEnd");

        try {
            // Execute billing (idempotent via database constraint)
            billingService.executeBilling(subscriptionId,
                java.time.LocalDate.parse(billingPeriodStart),
                java.time.LocalDate.parse(billingPeriodEnd));
        } catch (Exception e) {
            throw new JobExecutionException("Billing failed for subscription " + subscriptionId, e, false);
        }
    }
}
```

### 2. BillingService — Core Billing Logic (Idempotent)

```java
package com.subscription.billing.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import javax.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDate;

@Service
public class BillingService {

    @Autowired private SubscriptionRepository subscriptionRepo;
    @Autowired private BillingLogRepository billingLogRepo;
    @Autowired private UsageRecordRepository usageRecordRepo;
    @Autowired private InvoiceRepository invoiceRepo;
    @Autowired private KafkaTemplate<String, BillingResultEvent> kafkaTemplate;
    @Autowired private ProrationCalculator prorationCalculator;
    @Autowired private UsageAggregator usageAggregator;
    @Autowired private PaymentGateway paymentGateway;

    @Transactional
    public void executeBilling(Long subscriptionId, LocalDate periodStart, LocalDate periodEnd) {
        Subscription sub = subscriptionRepo.findById(subscriptionId)
            .orElseThrow(() -> new IllegalArgumentException("Subscription not found"));

        // Idempotency: check if billing_log already exists
        BillingLog existingLog = billingLogRepo
            .findBySubscriptionIdAndBillingPeriodStart(subscriptionId, periodStart);

        if (existingLog != null) {
            // Already billed; skip
            return;
        }

        // 1. Calculate base charge
        long baseChargeCents = sub.getCurrentPriceCents();

        // 2. Calculate prorated adjustments (if mid-cycle changes)
        long proratedAdjustmentCents = calculateProratedAdjustments(subscriptionId, periodStart, periodEnd);

        // 3. Calculate metered charges
        long meteredChargeCents = usageAggregator.aggregateUsageForPeriod(
            subscriptionId, periodStart, periodEnd);

        // 4. Calculate credits (manual adjustments, dunning credits)
        long creditAppliedCents = calculateAppliedCredits(subscriptionId);

        // 5. Total charge
        long totalChargeCents = baseChargeCents + proratedAdjustmentCents + meteredChargeCents - creditAppliedCents;

        // 6. Create billing log (unique constraint ensures idempotency)
        BillingLog billingLog = new BillingLog();
        billingLog.setSubscriptionId(subscriptionId);
        billingLog.setBillingPeriodStart(periodStart);
        billingLog.setBillingPeriodEnd(periodEnd);
        billingLog.setBaseChargeCents(baseChargeCents);
        billingLog.setProratedAdjustmentCents(proratedAdjustmentCents);
        billingLog.setMeteredChargeCents(meteredChargeCents);
        billingLog.setCreditAppliedCents(creditAppliedCents);
        billingLog.setTotalChargeCents(totalChargeCents);
        billingLog.setStatus("PENDING");

        BillingLog savedLog = billingLogRepo.save(billingLog);

        // 7. Attempt payment
        PaymentResult paymentResult = paymentGateway.chargePaymentMethod(
            sub.getPaymentMethodId(), totalChargeCents, sub.getCurrency());

        if (paymentResult.isSuccess()) {
            billingLog.setStatus("PAID");
            billingLog.setPaidAt(java.time.LocalDateTime.now());
        } else {
            billingLog.setStatus("FAILED");
            // Publish dunning event
            publishDunningEvent(billingLog, paymentResult.getFailureReason());
        }

        billingLogRepo.save(billingLog);

        // 8. Generate invoice
        generateInvoice(savedLog, sub);

        // 9. Publish success event
        kafkaTemplate.send("billing-result", new BillingResultEvent(
            subscriptionId, savedLog.getId(), totalChargeCents, billingLog.getStatus()));
    }

    private long calculateProratedAdjustments(Long subscriptionId, LocalDate periodStart, LocalDate periodEnd) {
        // Query plan_changes table for changes in period
        var planChanges = subscriptionRepo.findPlanChangesInPeriod(subscriptionId, periodStart, periodEnd);

        long totalProration = 0;
        for (PlanChange change : planChanges) {
            totalProration += prorationCalculator.calculateProration(
                change.getOldPlan(), change.getNewPlan(),
                change.getEffectiveDate(), periodEnd);
        }
        return totalProration;
    }

    private long calculateAppliedCredits(Long subscriptionId) {
        // Query credit_balance table
        CreditBalance balance = subscriptionRepo.findCreditBalanceBySubscription(subscriptionId);
        return balance != null ? balance.getBalanceCents() : 0L;
    }

    private void generateInvoice(BillingLog billingLog, Subscription sub) {
        String invoiceNumber = "INV-" + sub.getCustomerId() + "-" + System.currentTimeMillis();

        Invoice invoice = new Invoice();
        invoice.setBillingLogId(billingLog.getId());
        invoice.setCustomerId(sub.getCustomerId());
        invoice.setInvoiceNumber(invoiceNumber);
        invoice.setTotalAmountCents(billingLog.getTotalChargeCents());
        invoice.setCurrency(sub.getCurrency());
        invoice.setIssuedDate(LocalDate.now());
        invoice.setDueDate(LocalDate.now().plusDays(30));
        invoice.setStatus("ISSUED");

        invoiceRepo.save(invoice);

        // Publish async invoice generation event
        kafkaTemplate.send("invoice-generated", new InvoiceGeneratedEvent(
            invoice.getId(), sub.getCustomerId(), billingLog.getTotalChargeCents()));
    }

    private void publishDunningEvent(BillingLog billingLog, String failureReason) {
        DunningEvent event = new DunningEvent(
            billingLog.getId(),
            billingLog.getSubscriptionId(),
            failureReason,
            LocalDate.now().plusDays(1) // Retry in 1 day
        );
        kafkaTemplate.send("dunning-event", event);
    }

    public List<Subscription> findSubscriptionsReadyForBilling() {
        return subscriptionRepo.findByNextBillingDateLessThanEqualAndStatus(
            LocalDate.now(), "ACTIVE");
    }
}
```

### 3. ProrationCalculator — Proration Logic

```java
package com.subscription.billing.service;

import org.springframework.stereotype.Component;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;

@Component
public class ProrationCalculator {

    /**
     * Calculate prorated charge for mid-cycle plan change.
     * Prorated amount = (remaining_days / days_in_cycle) × (new_price - old_price)
     */
    public long calculateProration(Plan oldPlan, Plan newPlan,
                                  LocalDate changeDate, LocalDate cycleEnd) {

        long remainingDays = ChronoUnit.DAYS.between(changeDate, cycleEnd);
        long daysInCycle = ChronoUnit.DAYS.between(cycleEnd.minusMonths(1), cycleEnd);

        long priceDifference = newPlan.getPriceCents() - oldPlan.getPriceCents();

        BigDecimal prorationRatio = BigDecimal.valueOf(remainingDays)
            .divide(BigDecimal.valueOf(daysInCycle), 4, java.math.RoundingMode.HALF_UP);

        BigDecimal proratedAmount = prorationRatio
            .multiply(BigDecimal.valueOf(priceDifference));

        return proratedAmount.longValue();
    }

    /**
     * For downgrade: apply credit to next cycle
     */
    public void applyDowngradeCredit(Long subscriptionId, long creditCents) {
        // Insert into credit_balance or next billing_log
    }

    /**
     * For upgrade: charge immediately
     */
    public void chargeUpgradeImmediately(Long subscriptionId, long chargeCents) {
        // Publish immediate-charge event
    }
}
```

### 4. UsageAggregator — Metered Billing Aggregation

```java
package com.subscription.billing.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.stereotype.Service;
import java.time.LocalDate;
import java.util.List;

@Service
public class UsageAggregator {

    @Autowired private MongoTemplate mongoTemplate;
    @Autowired private RateCardService rateCardService;
    @Autowired private UsageRecordRepository usageRecordRepo;

    /**
     * Aggregate usage for a subscription in a billing period.
     * Returns total metered charge in cents.
     */
    public long aggregateUsageForPeriod(Long subscriptionId, LocalDate periodStart, LocalDate periodEnd) {

        List<UsageEvent> events = mongoTemplate.find(
            Query.query(Criteria.where("subscriptionId").is(subscriptionId)
                .and("timestamp").gte(periodStart).lt(periodEnd)),
            UsageEvent.class);

        // Group by metric type
        var eventsByMetric = events.stream()
            .collect(java.util.stream.Collectors.groupingBy(UsageEvent::getMetricType));

        long totalChargeCents = 0;

        for (var entry : eventsByMetric.entrySet()) {
            String metricType = entry.getKey();
            List<UsageEvent> metricEvents = entry.getValue();

            long meteredCharge = aggregateByMetricType(subscriptionId, metricType, metricEvents,
                periodStart, periodEnd);
            totalChargeCents += meteredCharge;
        }

        return totalChargeCents;
    }

    private long aggregateByMetricType(Long subscriptionId, String metricType,
                                       List<UsageEvent> events, LocalDate periodStart, LocalDate periodEnd) {

        // Determine aggregation strategy (SUM, MAX, AVG)
        String aggregationType = rateCardService.getAggregationType(metricType);

        double aggregatedValue = switch(aggregationType) {
            case "SUM" -> events.stream().mapToDouble(UsageEvent::getValue).sum();
            case "MAX" -> events.stream().mapToDouble(UsageEvent::getValue).max().orElse(0);
            case "AVG" -> events.stream().mapToDouble(UsageEvent::getValue).average().orElse(0);
            default -> 0;
        };

        // Look up unit price from rate card
        long unitPriceCents = rateCardService.getUnitPrice(metricType);

        // Calculate total charge
        long totalCharge = (long) (aggregatedValue * unitPriceCents);

        // Store in PostgreSQL for audit
        UsageRecord usageRecord = new UsageRecord();
        usageRecord.setSubscriptionId(subscriptionId);
        usageRecord.setMetricType(metricType);
        usageRecord.setBillingPeriodStart(periodStart);
        usageRecord.setBillingPeriodEnd(periodEnd);
        usageRecord.setAggregationType(aggregationType);
        usageRecord.setAggregatedValue(BigDecimal.valueOf(aggregatedValue));
        usageRecord.setUnitPriceCents(unitPriceCents);
        usageRecord.setTotalMeteredChargeCents(totalCharge);

        usageRecordRepo.save(usageRecord);

        return totalCharge;
    }
}

// MongoDB Document
@Document(collection = "usage_events")
public class UsageEvent {
    private String id;
    private Long subscriptionId;
    private String metricType; // api_calls, storage_gb, concurrent_users
    private double value;
    private long timestamp;
    private String source;

    // Getters, Setters
}
```

### 5. DunningManager — Payment Retry & State Machine

```java
package com.subscription.billing.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.LocalDate;
import java.util.List;

@Service
public class DunningManager {

    @Autowired private DunningAttemptRepository dunningRepo;
    @Autowired private BillingLogRepository billingLogRepo;
    @Autowired private SubscriptionRepository subscriptionRepo;
    @Autowired private PaymentGateway paymentGateway;

    private static final int MAX_DUNNING_ATTEMPTS = 3;
    private static final int[] RETRY_DAYS = {1, 3, 7, 14}; // T+1, T+3, T+7, T+14

    @KafkaListener(topics = "dunning-event")
    @Transactional
    public void handleDunningEvent(DunningEvent event) {
        BillingLog billingLog = billingLogRepo.findById(event.getBillingLogId())
            .orElseThrow();

        Subscription sub = subscriptionRepo.findById(event.getSubscriptionId())
            .orElseThrow();

        // Schedule first retry
        DunningAttempt attempt = new DunningAttempt();
        attempt.setSubscriptionId(event.getSubscriptionId());
        attempt.setBillingLogId(event.getBillingLogId());
        attempt.setAttemptNumber(1);
        attempt.setScheduledRetryDate(LocalDate.now().plusDays(RETRY_DAYS[0]));
        attempt.setStatus("PENDING");

        dunningRepo.save(attempt);

        // Mark subscription as PAST_DUE
        sub.setStatus("PAST_DUE");
        subscriptionRepo.save(sub);
    }

    /**
     * Scheduled job to retry failed payments
     */
    public void executeScheduledRetries() {
        List<DunningAttempt> pending = dunningRepo
            .findByStatusAndScheduledRetryDateLessThanEqual("PENDING", LocalDate.now());

        for (DunningAttempt attempt : pending) {
            retryPayment(attempt);
        }
    }

    @Transactional
    private void retryPayment(DunningAttempt attempt) {
        BillingLog billingLog = billingLogRepo.findById(attempt.getBillingLogId()).orElseThrow();
        Subscription sub = subscriptionRepo.findById(attempt.getSubscriptionId()).orElseThrow();

        // Attempt payment
        PaymentResult result = paymentGateway.chargePaymentMethod(
            sub.getPaymentMethodId(), billingLog.getTotalChargeCents(), sub.getCurrency());

        attempt.setExecutedAt(java.time.LocalDateTime.now());

        if (result.isSuccess()) {
            attempt.setStatus("SUCCEEDED");
            billingLog.setStatus("PAID");
            billingLog.setPaidAt(java.time.LocalDateTime.now());
            sub.setStatus("ACTIVE");
        } else {
            attempt.setStatus("FAILED");
            attempt.setFailureReason(result.getFailureReason());

            // Schedule next retry
            if (attempt.getAttemptNumber() < MAX_DUNNING_ATTEMPTS) {
                int daysUntilNextRetry = RETRY_DAYS[attempt.getAttemptNumber()];

                DunningAttempt nextAttempt = new DunningAttempt();
                nextAttempt.setSubscriptionId(attempt.getSubscriptionId());
                nextAttempt.setBillingLogId(attempt.getBillingLogId());
                nextAttempt.setAttemptNumber(attempt.getAttemptNumber() + 1);
                nextAttempt.setScheduledRetryDate(LocalDate.now().plusDays(daysUntilNextRetry));
                nextAttempt.setStatus("PENDING");

                dunningRepo.save(nextAttempt);
            } else {
                // All retries exhausted: SUSPENDED
                sub.setStatus("SUSPENDED");
            }
        }

        dunningRepo.save(attempt);
        subscriptionRepo.save(sub);
    }
}
```

### 6. InvoiceGenerationService — Async Invoice & PDF Generation

```java
package com.subscription.billing.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import org.thymeleaf.TemplateEngine;
import org.thymeleaf.context.Context;
import com.itextpdf.html2pdf.HtmlConverter;
import java.io.ByteArrayOutputStream;
import java.util.HashMap;
import java.util.Map;

@Service
public class InvoiceGenerationService {

    @Autowired private InvoiceRepository invoiceRepo;
    @Autowired private BillingLogRepository billingLogRepo;
    @Autowired private SubscriptionRepository subscriptionRepo;
    @Autowired private TemplateEngine templateEngine;
    @Autowired private S3Service s3Service;
    @Autowired private EmailService emailService;

    @KafkaListener(topics = "invoice-generated")
    public void generateAndSendInvoice(InvoiceGeneratedEvent event) throws Exception {
        Invoice invoice = invoiceRepo.findById(event.getInvoiceId()).orElseThrow();
        BillingLog billingLog = billingLogRepo.findById(invoice.getBillingLogId()).orElseThrow();
        Subscription sub = subscriptionRepo.findById(billingLog.getSubscriptionId()).orElseThrow();

        // Prepare template context
        Context context = new Context();
        context.setVariable("invoice", invoice);
        context.setVariable("subscription", sub);
        context.setVariable("billingLog", billingLog);
        context.setVariable("generatedDate", java.time.LocalDate.now());

        // Render Thymeleaf HTML
        String html = templateEngine.process("invoice_template", context);

        // Convert HTML to PDF
        ByteArrayOutputStream pdfOutput = new ByteArrayOutputStream();
        HtmlConverter.convertHtmlString(html, pdfOutput);
        byte[] pdfBytes = pdfOutput.toByteArray();

        // Upload to S3
        String pdfUrl = s3Service.uploadInvoice(invoice.getInvoiceNumber(), pdfBytes);
        invoice.setPdfUrl(pdfUrl);
        invoiceRepo.save(invoice);

        // Send email notification
        String recipientEmail = subscriptionRepo.findCustomerEmailBySubscriptionId(
            billingLog.getSubscriptionId());

        emailService.sendInvoiceEmail(recipientEmail, invoice.getInvoiceNumber(), pdfUrl);
    }
}

// Thymeleaf Template (invoice_template.html)
/*
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <style>
        body { font-family: Arial, sans-serif; }
        .header { text-align: center; margin-bottom: 30px; }
        .invoice-table { width: 100%; border-collapse: collapse; }
        .invoice-table td { padding: 10px; border: 1px solid #ddd; }
    </style>
</head>
<body>
    <div class="header">
        <h1>INVOICE</h1>
        <p th:text="'Invoice Number: ' + ${invoice.invoiceNumber}"></p>
        <p th:text="'Date: ' + ${generatedDate}"></p>
    </div>

    <table class="invoice-table">
        <tr>
            <td>Base Charge</td>
            <td th:text="${billingLog.baseChargeCents / 100}"></td>
        </tr>
        <tr>
            <td>Metered Usage</td>
            <td th:text="${billingLog.meteredChargeCents / 100}"></td>
        </tr>
        <tr>
            <td>Prorated Adjustment</td>
            <td th:text="${billingLog.proratedAdjustmentCents / 100}"></td>
        </tr>
        <tr style="font-weight: bold;">
            <td>Total</td>
            <td th:text="${invoice.totalAmountCents / 100}"></td>
        </tr>
    </table>
</body>
</html>
*/
```

---

## Failure Scenarios & Recovery

| Scenario | Detection | Recovery |
|----------|-----------|----------|
| **Job crash mid-billing** | Quartz recovery (persisted job state) | Re-run job; idempotency via `UNIQUE(subscription_id, billing_period)` |
| **Payment gateway timeout** | HTTP timeout after 30s | Retry with exponential backoff; publish dunning event |
| **Kafka delivery failure** | Failed ACK | Retry with dead-letter topic; alert ops |
| **Database connection lost** | Connection pool exhaustion | Circuit breaker; fallback to manual reconciliation |
| **Out-of-memory: usage cache** | RedisMemory exceeded | Flush usage buffers to MongoDB; alert |
| **Race condition: concurrent billing** | Constraint violation on insert | Caught at DB; first job succeeds, retry fails gracefully |
| **Invoice PDF generation timeout** | Thread hang > 60s | Async task timeout; retry async task later |
| **Double charge bug** | Audit query shows 2 entries with same period | Unique constraint prevents; impossible |

---

## Scaling Strategy

### Horizontal Scaling
1. **Billing Scheduler:** Add more nodes; partition count increases. Use consistent hashing or modulo for load balancing.
2. **BillingEngine:** Parallel job execution across nodes; Quartz handles distribution.
3. **UsageAggregator:** Scale Kafka consumer group; MongoDB auto-sharding by subscription_id.
4. **DunningManager:** Separate consumer group; partition by subscription_id.

### Database Scaling
- **PostgreSQL:** Sharding by customer_id (billing_log, invoices, dunning_attempts) or read replicas for analytics
- **MongoDB:** Auto-sharding by subscription_id; TTL index on usage_events (delete after 90 days)
- **Redis:** Cluster mode for high throughput; key distribution across 8–16 nodes

### Kafka Scaling
- Increase partitions: `--partitions 100` per topic
- Consumer group with 10+ workers per topic
- Monitor lag; alert if lag > 1 million messages

---

## Monitoring & Alerting

### Key Metrics
```
# Throughput
billing.jobs.executed.total (counter)
billing.jobs.duration_seconds (histogram)
usage.events.ingested.total (rate)
invoices.generated.total (rate)

# Reliability
billing.errors.total (counter, by error_type)
dunning.retry.success_rate (gauge)
payment.gateway.latency_ms (histogram)

# Business Metrics
subscriptions.active.count (gauge)
revenue.total_cents.monthly (gauge)
churn.rate.monthly (gauge)
dunning.past_due_subscriptions.count (gauge)
```

### Alert Rules
```
ALERT BillingJobHighFailureRate
  IF rate(billing.errors.total[5m]) > 0.1
  FOR 10m

ALERT PaymentGatewayLatencyHigh
  IF payment.gateway.latency_ms > 5000
  FOR 5m

ALERT DunningQueueDepthHigh
  IF kafka_consumer_lag{topic="dunning-event"} > 10000
  FOR 15m
```

---

## Summary Cheat Sheet

| Component | Pattern | Key Code |
|-----------|---------|----------|
| **Scheduling** | Quartz + partition by subscription_id % num_nodes | `jobKey = "billing_" + sub.getId()` |
| **Idempotency** | UNIQUE(subscription_id, billing_period) in billing_log | INSERT fails on re-run |
| **Proration** | `(remaining_days / days_in_cycle) × (new_price - old_price)` | `ProrationCalculator.calculate()` |
| **Metered Billing** | MongoDB buffer → aggregate at cycle end | `UsageAggregator.aggregateUsageForPeriod()` |
| **Dunning** | State machine: ACTIVE → PAST_DUE → SUSPENDED | Retry T+1, T+3, T+7, T+14 |
| **Double-Charge Prevention** | Unique constraint on insert | DB enforces; no app logic needed |
| **Invoice Generation** | Thymeleaf HTML → iText PDF → S3 + Email | `InvoiceGenerationService` |
| **Ledger** | Double-entry: debit/credit pairs | `ledger_entries` table |
| **Retry Lock** | Redis key with TTL 5 min | `billing:lock:{subscription_id}` |

