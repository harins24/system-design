---
title: Payment Reconciliation System
layout: default
---

# Payment Reconciliation System — Deep Dive Design

> **Scenario:** Match payment gateway transactions vs internal orders, bank deposits vs gateway settlements, refunds processed vs received from gateway. Identify discrepancies (missing transactions, amount mismatches). Daily reconciliation reports. Alert on anomalies. **100K transactions/day.**
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
| **Throughput** | 100K transactions/day (1.16 tx/sec avg, but bursty) |
| **Matching Accuracy** | 99.99% match rate; <0.01% false positives |
| **Latency** | Reconciliation completes within 24 hours; discrepancy alerts within 2 hours |
| **Data Sources** | Payment gateway (Stripe, Adyen), bank statements, internal order DB |
| **Matching Logic** | Three-way match: internal order ↔ gateway ↔ bank; fuzzy match on amount + date |
| **Timing Buffer** | T+1 to T+2 day lookback; gateway delays up to 48 hours |
| **Discrepancy Types** | AMOUNT_MISMATCH, MISSING_INTERNAL, MISSING_GATEWAY, DUPLICATE, TIMING_VARIANCE |
| **Auto-Resolution** | Known delays auto-clear; rounding errors auto-approved; manual review for anomalies |
| **Audit Trail** | Immutable ledger; credit bureau compliance |
| **Data Retention** | 7 years for financial records (compliance) |

---

## Capacity Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Daily Transactions** | 100K/day | 100,000 records |
| **Hourly (peak)** | 100K ÷ 24 × 2.5 (burst) | ~10,400 tx/hour |
| **Reconciliation Batch Size** | Process in 2-4 hour windows | 5K–20K tx per batch |
| **Discrepancy Rate** | Assume 0.5% require investigation | 500 discrepancies/day |
| **Storage (PostgreSQL)** | 100K tx/day × 365 days × 2 years × 2KB/tx | ~146 GB |
| **Redis Cache** | In-flight batches + discrepancy index | ~50 MB |
| **Kafka Topic Partitions** | 10–15 (distributed by tx ID hash) | 15 partitions |
| **Monthly Reconciliation Reports** | PDF + CSV export | ~30 files/month |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         PAYMENT RECONCILIATION SYSTEM                        │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────────┐
│                          DATA INGESTION LAYER                                 │
├──────────────────────────────────────────────────────────────────────────────┤
│  Stripe API / Adyen API          Bank Statement Processor       Internal DB   │
│  ↓                               ↓                              ↓             │
│  GatewayTransactionConsumer      BankStatementFileParser        OrderEventSub │
│  (Scheduled Daily 00:00)         (Scheduled Daily 02:00)        (Real-time)   │
└──────────────────────────────────────────────────────────────────────────────┘
                                    ↓
                        ┌───────────────────────────┐
                        │   KAFKA TOPICS (15x)      │
                        ├───────────────────────────┤
                        │ transactions.gateway      │
                        │ transactions.bank         │
                        │ transactions.internal     │
                        │ discrepancies.detected    │
                        └───────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────────┐
│                       RECONCILIATION ENGINE (Spring Batch)                    │
├──────────────────────────────────────────────────────────────────────────────┤
│  ReconciliationJob (Scheduled Daily T+1)                                     │
│  ├─ Step 1: Read gateway + bank + internal (Partitioned Reader)              │
│  ├─ Step 2: TransactionMatcher (Fuzzy match algorithm)                       │
│  ├─ Step 3: DiscrepancyDetector (Classification)                             │
│  ├─ Step 4: AutoResolutionService (Resolve known patterns)                   │
│  └─ Step 5: Write matched + unmatched to PostgreSQL                          │
└──────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────────┐
│                   ANALYSIS & ALERTING (Real-time Stream)                      │
├──────────────────────────────────────────────────────────────────────────────┤
│  DiscrepancyAnalyzer (Kafka Streams)                                         │
│  ├─ Anomaly detection (one-off missing vs pattern)                           │
│  ├─ Amount threshold alerts (>$5K variance)                                  │
│  ├─ Missing transaction alerts (>10 consecutive)                             │
│  └─ Send to alerting.critical topic                                          │
└──────────────────────────────────────────────────────────────────────────────┘
                                    ↓
┌──────────────────────────────────────────────────────────────────────────────┐
│              REPORTING & INVESTIGATION LAYER (MongoDB + REST API)             │
├──────────────────────────────────────────────────────────────────────────────┤
│  ReconciliationReportGenerator    │  DiscrepancyInvestigationService         │
│  ├─ Daily summary PDF             │  ├─ Case assignment to ops team          │
│  ├─ Weekly exception report       │  ├─ Evidence collection                  │
│  ├─ Monthly audit ledger          │  └─ Status tracking (OPEN/RESOLVED)      │
│  └─ Export to accounting system   │                                          │
└──────────────────────────────────────────────────────────────────────────────┘
                                    ↓
                    ┌─────────────────────────────┐
                    │  PostgreSQL + MongoDB       │
                    ├─────────────────────────────┤
                    │ · matched_transactions      │
                    │ · discrepancy_records       │
                    │ · reconciliation_batches    │
                    │ · reconciliation_reports    │
                    │ · auto_resolution_rules     │
                    └─────────────────────────────┘

CACHING LAYER (Redis)
├─ in_flight:{batchId} → Batch metadata (TTL 24h)
├─ discrepancies:unresolved:{date} → Sorted set (score=timestamp)
├─ transaction_index:{txId} → Fast lookup (TTL 7d)
└─ reconciliation_metrics:{date} → Match rate, error counts
```

---

## Core Design Questions Answered

### 1. How do you match transactions across systems with different IDs?

**Three-level matching hierarchy:**

- **Level 1 (Exact):** Match by gateway transaction ID if internal system stored it during payment
- **Level 2 (Primary Key):** Match by amount + date ± 2-day window + merchant reference
- **Level 3 (Fuzzy):** Jaro-Winkler similarity on merchant name; amount tolerance ±$0.99

Example:
```
Internal: order_123, $149.99, 2024-03-15, "john.doe@example.com"
Gateway:  txn_456, $149.99, 2024-03-15, gateway_ref="ORDER123"
Bank:     Stripe Settlement, $149.99, 2024-03-15

Result: All 3 MATCHED (confidence: 99.5%)
```

### 2. How do you handle timing differences (gateway reports delayed)?

**Temporal buffering strategy:**

- Reconciliation runs at T+1 (next day at 00:00)
- Lookback window: 2 days (T-2 to T)
- Missing gateway transactions after 48h marked UNMATCHED_48H
- Bank deposits lag by 1–3 days; reconciliation re-runs T+3 for catch-up
- Known delays per gateway stored in `auto_resolution_rules` table

### 3. How do you identify and investigate discrepancies?

**Classification engine:**

| Discrepancy Type | Detection | Action |
|---|---|---|
| AMOUNT_MISMATCH | abs(internal - gateway) > $0.01 | Manual review + screenshot |
| MISSING_INTERNAL | Gateway present, internal absent | Check order DB query; escalate to order service |
| MISSING_GATEWAY | Internal present, gateway absent | Retry gateway API; check failed payment logs |
| DUPLICATE | 2+ internal orders match same gateway tx | Investigate double-charge risk; refund if needed |
| TIMING_VARIANCE | Date diff >3 days | Auto-clear if within known gateway SLA |

### 4. How do you automate the reconciliation process?

**Spring Batch job with idempotent design:**

- Job: `ReconciliationJob` runs daily at 00:30 UTC
- Partitioned readers: 10 partitions (by date range)
- Processor chain: Match → Classify → AutoResolve → Score
- Output: `matched_transactions` + `discrepancy_records` tables
- Idempotency: Job name = "RECONCILIATION_" + date; runs once/day

### 5. How do you generate audit reports for accounting?

**Report generation:**

- **Daily Summary:** Total matched, discrepancies by type, auto-resolved count
- **Weekly Exception Report:** High-value discrepancies, missing gateways, refund anomalies
- **Monthly Ledger:** All transactions grouped by gateway + settlement period; exported to accounting system
- **Format:** PDF (visual), CSV (data), JSON (API)

### 6. How do you handle partial refunds and split payments?

**Refund matching:**

- Refund linked to original payment via `original_transaction_id`
- Partial refunds: Match by refund amount ≤ original; create separate refund record
- Split payments (e.g., wallet + card): Internal order split into 2 payment records; both matched independently
- Net settlement = sum of charged amounts - sum of refund amounts

---

## Microservices Breakdown

| Service | Responsibility | Tech |
|---|---|---|
| **Gateway Consumer** | Fetch transactions from Stripe/Adyen APIs; publish to Kafka | Spring Boot + REST client |
| **Bank Statement Processor** | Parse bank feed (SFTP CSV/XML); validate; publish to Kafka | Spring Boot + File I/O |
| **Reconciliation Engine** | Spring Batch job; match; classify; resolve | Spring Boot + Spring Batch |
| **Discrepancy Service** | Store + retrieve discrepancy records; assign to teams | Spring Boot + REST API |
| **Reporting Service** | Generate PDF/CSV reports; export to accounting | Spring Boot + iText/FreeMarker |
| **Alerting Service** | Monitor Kafka stream; fire alerts on anomalies | Spring Boot + Kafka Streams |

---

## Database Design (DDL)

```sql
-- PostgreSQL Schema

CREATE TABLE transactions_internal (
    id BIGSERIAL PRIMARY KEY,
    order_id VARCHAR(100) NOT NULL UNIQUE,
    customer_id BIGINT NOT NULL,
    amount_cents BIGINT NOT NULL,
    currency VARCHAR(3),
    status VARCHAR(20),
    payment_method VARCHAR(50),
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP,
    CONSTRAINT positive_amount CHECK (amount_cents > 0)
);

CREATE TABLE transactions_gateway (
    id BIGSERIAL PRIMARY KEY,
    gateway_transaction_id VARCHAR(100) UNIQUE,
    gateway_name VARCHAR(50) NOT NULL,
    amount_cents BIGINT NOT NULL,
    currency VARCHAR(3),
    status VARCHAR(20),
    merchant_reference VARCHAR(255),
    received_at TIMESTAMP NOT NULL,
    processed_at TIMESTAMP,
    CONSTRAINT positive_amount CHECK (amount_cents > 0)
);

CREATE TABLE transactions_bank (
    id BIGSERIAL PRIMARY KEY,
    bank_settlement_id VARCHAR(100) UNIQUE,
    amount_cents BIGINT NOT NULL,
    currency VARCHAR(3),
    bank_name VARCHAR(100),
    settlement_date DATE NOT NULL,
    transaction_date DATE NOT NULL,
    received_at TIMESTAMP NOT NULL,
    CONSTRAINT positive_amount CHECK (amount_cents > 0)
);

CREATE TABLE matched_transactions (
    id BIGSERIAL PRIMARY KEY,
    internal_tx_id BIGINT NOT NULL REFERENCES transactions_internal(id),
    gateway_tx_id BIGINT REFERENCES transactions_gateway(id),
    bank_tx_id BIGINT REFERENCES transactions_bank(id),
    match_confidence DECIMAL(5,2),
    match_level INT,
    matched_at TIMESTAMP NOT NULL,
    UNIQUE(internal_tx_id, gateway_tx_id, bank_tx_id)
);

CREATE TABLE discrepancy_records (
    id BIGSERIAL PRIMARY KEY,
    batch_date DATE NOT NULL,
    discrepancy_type VARCHAR(50) NOT NULL,
    internal_tx_id BIGINT REFERENCES transactions_internal(id),
    gateway_tx_id BIGINT REFERENCES transactions_gateway(id),
    bank_tx_id BIGINT REFERENCES transactions_bank(id),
    variance_amount_cents BIGINT,
    description TEXT,
    status VARCHAR(30),
    assigned_to VARCHAR(100),
    resolution_notes TEXT,
    resolved_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP
);

CREATE TABLE reconciliation_batches (
    id BIGSERIAL PRIMARY KEY,
    batch_date DATE NOT NULL UNIQUE,
    batch_status VARCHAR(30),
    total_internal_count INT,
    total_gateway_count INT,
    total_bank_count INT,
    matched_count INT,
    discrepancy_count INT,
    auto_resolved_count INT,
    started_at TIMESTAMP,
    completed_at TIMESTAMP
);

CREATE TABLE auto_resolution_rules (
    id BIGSERIAL PRIMARY KEY,
    rule_name VARCHAR(100) NOT NULL,
    gateway_name VARCHAR(50),
    max_delay_hours INT,
    tolerance_cents INT,
    auto_approve BOOLEAN,
    created_at TIMESTAMP
);

CREATE INDEX idx_transactions_internal_created ON transactions_internal(created_at);
CREATE INDEX idx_transactions_gateway_received ON transactions_gateway(received_at);
CREATE INDEX idx_transactions_bank_settlement_date ON transactions_bank(settlement_date);
CREATE INDEX idx_discrepancy_batch_date ON discrepancy_records(batch_date);
CREATE INDEX idx_discrepancy_status ON discrepancy_records(status);
```

---

## Redis Data Structures

```
Key Pattern                         Type        TTL     Purpose
─────────────────────────────────────────────────────────────────
in_flight:{batchId}                Hash        24h     Batch progress (started_count, matched_count)
discrepancies:unresolved:{date}     ZSet        7d      Sorted by timestamp; fast retrieval by date range
transaction_index:{txId}            String      7d      JSON of matched transaction (quick lookup)
reconciliation_metrics:{date}       Hash        30d     match_rate, error_counts, duration_ms
gateway_delays:{gateway}            Hash        ongoing Known delays per gateway (max_delay_hours)
pending_alerts:{severity}           List        24h     Queue of alerts pending notification
```

---

## Kafka Event Flow

```
Topic: transactions.gateway (15 partitions, key = gateway_transaction_id)
├─ Partition 0–14: One message per gateway transaction
├─ Payload: { gateway_tx_id, amount, currency, merchant_ref, status, received_at }
└─ Consumer: ReconciliationEngine (lag < 1 hour)

Topic: transactions.bank (10 partitions, key = settlement_date + bank_name)
├─ Payload: { bank_settlement_id, amount, settlement_date, transaction_date }
└─ Consumer: ReconciliationEngine

Topic: transactions.internal (15 partitions, key = order_id)
├─ Payload: { order_id, customer_id, amount, payment_method, created_at }
└─ Consumer: ReconciliationEngine

Topic: discrepancies.detected (8 partitions, key = discrepancy_type)
├─ Published by: ReconciliationEngine
├─ Payload: { discrepancy_id, type, variance_amount, severity }
└─ Consumers: DiscrepancyAnalyzer, AlertingService

Topic: alerts.critical (3 partitions, key = severity)
├─ Published by: AlertingService
├─ Payload: { alert_id, severity, message, timestamp }
└─ Consumer: NotificationService (→ PagerDuty, Slack)

Topic: reconciliation.completed (1 partition)
├─ Published by: ReconciliationEngine (once per day)
├─ Payload: { batch_date, metrics, report_id }
└─ Consumer: ReportingService
```

---

## Implementation Code

### 1. ReconciliationJob (Spring Batch Configuration)

```java
package com.payment.reconciliation.batch;

import org.springframework.batch.core.Job;
import org.springframework.batch.core.Step;
import org.springframework.batch.core.configuration.annotation.JobBuilderFactory;
import org.springframework.batch.core.configuration.annotation.StepBuilderFactory;
import org.springframework.batch.core.launch.support.RunIdIncrementer;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemReader;
import org.springframework.batch.item.ItemWriter;
import org.springframework.batch.item.database.builder.JdbcBatchItemWriterBuilder;
import org.springframework.batch.item.file.FlatFileItemReader;
import org.springframework.batch.item.file.builder.FlatFileItemReaderBuilder;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.FileSystemResource;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;

import java.io.File;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;

@Configuration
@EnableScheduling
public class ReconciliationBatchConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final TransactionMatcher transactionMatcher;
    private final DiscrepancyDetector discrepancyDetector;
    private final AutoResolutionService autoResolutionService;

    public ReconciliationBatchConfiguration(
            JobBuilderFactory jobBuilderFactory,
            StepBuilderFactory stepBuilderFactory,
            TransactionMatcher transactionMatcher,
            DiscrepancyDetector discrepancyDetector,
            AutoResolutionService autoResolutionService) {
        this.jobBuilderFactory = jobBuilderFactory;
        this.stepBuilderFactory = stepBuilderFactory;
        this.transactionMatcher = transactionMatcher;
        this.discrepancyDetector = discrepancyDetector;
        this.autoResolutionService = autoResolutionService;
    }

    @Bean
    public Job reconciliationJob(
            Step readAndMatchStep,
            Step discrepancyStep,
            Step reportStep) {
        return jobBuilderFactory.get("reconciliationJob")
                .incrementer(new RunIdIncrementer())
                .flow(readAndMatchStep)
                .next(discrepancyStep)
                .next(reportStep)
                .end()
                .build();
    }

    @Bean
    public Step readAndMatchStep(
            ItemReader<TransactionBatch> batchReader,
            ItemProcessor<TransactionBatch, MatchedTransactionDTO> matchProcessor,
            ItemWriter<MatchedTransactionDTO> matchWriter) {
        return stepBuilderFactory.get("readAndMatchStep")
                .<TransactionBatch, MatchedTransactionDTO>chunk(1000)
                .reader(batchReader)
                .processor(matchProcessor)
                .writer(matchWriter)
                .faultTolerant()
                .skipLimit(10)
                .skip(Exception.class)
                .build();
    }

    @Bean
    public Step discrepancyStep(
            ItemReader<UnmatchedTransaction> unmatchedReader,
            ItemProcessor<UnmatchedTransaction, DiscrepancyRecord> discrepancyProcessor,
            ItemWriter<DiscrepancyRecord> discrepancyWriter) {
        return stepBuilderFactory.get("discrepancyStep")
                .<UnmatchedTransaction, DiscrepancyRecord>chunk(500)
                .reader(unmatchedReader)
                .processor(discrepancyProcessor)
                .writer(discrepancyWriter)
                .build();
    }

    @Bean
    public ItemProcessor<TransactionBatch, MatchedTransactionDTO> matchProcessor() {
        return batch -> {
            MatchResult result = transactionMatcher.matchTransactions(
                    batch.getInternalTx(),
                    batch.getGatewayTx(),
                    batch.getBankTx()
            );
            if (result.isMatched()) {
                return new MatchedTransactionDTO(
                        batch.getInternalTx().getId(),
                        batch.getGatewayTx().getId(),
                        batch.getBankTx().getId(),
                        result.getConfidence(),
                        result.getMatchLevel()
                );
            }
            return null;
        };
    }

    @Bean
    public ItemProcessor<UnmatchedTransaction, DiscrepancyRecord> discrepancyProcessor() {
        return unmatched -> {
            DiscrepancyType type = discrepancyDetector.classify(unmatched);
            boolean autoResolved = autoResolutionService.canAutoResolve(unmatched, type);
            return new DiscrepancyRecord(
                    LocalDate.now(),
                    type,
                    unmatched,
                    autoResolved ? DiscrepancyStatus.AUTO_RESOLVED : DiscrepancyStatus.PENDING_REVIEW
            );
        };
    }

    @Scheduled(cron = "0 30 0 * * *") // Daily at 00:30 UTC
    public void triggerReconciliationJob(JobLauncher jobLauncher) throws Exception {
        jobLauncher.run(reconciliationJob(null, null, null), new JobParameters());
    }
}
```

### 2. TransactionMatcher (Matching Algorithm)

```java
package com.payment.reconciliation.service;

import org.springframework.stereotype.Service;
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.*;

@Service
public class TransactionMatcher {

    private static final long AMOUNT_TOLERANCE_CENTS = 1; // $0.01
    private static final int MAX_DATE_VARIANCE_DAYS = 2;
    private static final double EXACT_MATCH_THRESHOLD = 0.99;
    private static final double FUZZY_MATCH_THRESHOLD = 0.85;

    public MatchResult matchTransactions(
            InternalTransaction internal,
            GatewayTransaction gateway,
            BankTransaction bank) {

        // Level 1: Exact match by gateway transaction ID
        if (gateway != null && internal.getGatewayTxId() != null &&
                gateway.getId().equals(internal.getGatewayTxId())) {
            return MatchResult.exact(internal, gateway, bank, 0.99);
        }

        // Level 2: Match by amount + date + merchant reference
        List<MatchCandidate> candidates = new ArrayList<>();

        if (gateway != null && amountMatch(internal.getAmountCents(), gateway.getAmountCents()) &&
                dateWithinWindow(internal.getCreatedAt(), gateway.getReceivedAt(), MAX_DATE_VARIANCE_DAYS)) {
            candidates.add(new MatchCandidate(gateway, 0.95, 2));
        }

        if (bank != null && amountMatch(internal.getAmountCents(), bank.getAmountCents()) &&
                dateWithinWindow(internal.getCreatedAt(), bank.getTransactionDate(), MAX_DATE_VARIANCE_DAYS)) {
            candidates.add(new MatchCandidate(bank, 0.92, 2));
        }

        if (!candidates.isEmpty()) {
            MatchCandidate best = candidates.stream()
                    .max(Comparator.comparingDouble(MatchCandidate::getConfidence))
                    .orElseThrow();
            return new MatchResult(true, best.getConfidence(), best.getMatchLevel(), internal, gateway, bank);
        }

        // Level 3: Fuzzy match on merchant name + amount tolerance
        double similarityScore = calculateSimilarity(internal, gateway);
        if (similarityScore >= FUZZY_MATCH_THRESHOLD &&
                amountTolerance(internal.getAmountCents(), gateway.getAmountCents(), 100)) { // $1.00 tolerance
            return new MatchResult(true, similarityScore, 3, internal, gateway, bank);
        }

        return MatchResult.noMatch();
    }

    private boolean amountMatch(long internalCents, long gatewayCents) {
        return Math.abs(internalCents - gatewayCents) <= AMOUNT_TOLERANCE_CENTS;
    }

    private boolean amountTolerance(long internalCents, long gatewayCents, long toleranceCents) {
        return Math.abs(internalCents - gatewayCents) <= toleranceCents;
    }

    private boolean dateWithinWindow(LocalDateTime internal, LocalDateTime gateway, int days) {
        long diff = ChronoUnit.DAYS.between(gateway, internal);
        return Math.abs(diff) <= days;
    }

    private double calculateSimilarity(InternalTransaction internal, GatewayTransaction gateway) {
        if (gateway.getMerchantReference() == null || internal.getMerchantRef() == null) {
            return 0.0;
        }
        return jaroWinklerSimilarity(
                internal.getMerchantRef().toLowerCase(),
                gateway.getMerchantReference().toLowerCase()
        );
    }

    private double jaroWinklerSimilarity(String s1, String s2) {
        if (s1.equals(s2)) return 1.0;

        int jaro = (int) (jaroSimilarity(s1, s2) * 100);
        if (jaro < 70) return jaro / 100.0;

        int prefixLen = 0;
        for (int i = 0; i < Math.min(s1.length(), s2.length()) && i < 4; i++) {
            if (s1.charAt(i) == s2.charAt(i)) {
                prefixLen++;
            } else {
                break;
            }
        }

        return (jaro / 100.0) + (prefixLen * 0.1 * (1.0 - jaro / 100.0));
    }

    private double jaroSimilarity(String s1, String s2) {
        int len1 = s1.length();
        int len2 = s2.length();
        if (len1 == 0 && len2 == 0) return 1.0;
        if (len1 == 0 || len2 == 0) return 0.0;

        int matchDistance = Math.max(len1, len2) / 2 - 1;
        boolean[] s1Matches = new boolean[len1];
        boolean[] s2Matches = new boolean[len2];

        int matches = 0;
        for (int i = 0; i < len1; i++) {
            int start = Math.max(0, i - matchDistance);
            int end = Math.min(i + matchDistance + 1, len2);
            for (int j = start; j < end; j++) {
                if (s2Matches[j] || s1.charAt(i) != s2.charAt(j)) continue;
                s1Matches[i] = true;
                s2Matches[j] = true;
                matches++;
                break;
            }
        }

        if (matches == 0) return 0.0;

        int transpositions = 0;
        int k = 0;
        for (int i = 0; i < len1; i++) {
            if (!s1Matches[i]) continue;
            while (!s2Matches[k]) k++;
            if (s1.charAt(i) != s2.charAt(k)) transpositions++;
            k++;
        }

        return (matches / (double) len1 +
                matches / (double) len2 +
                (matches - transpositions / 2.0) / matches) / 3.0;
    }
}

class MatchResult {
    private final boolean matched;
    private final double confidence;
    private final int matchLevel;
    private final InternalTransaction internal;
    private final GatewayTransaction gateway;
    private final BankTransaction bank;

    public MatchResult(boolean matched, double confidence, int matchLevel,
                       InternalTransaction internal, GatewayTransaction gateway, BankTransaction bank) {
        this.matched = matched;
        this.confidence = confidence;
        this.matchLevel = matchLevel;
        this.internal = internal;
        this.gateway = gateway;
        this.bank = bank;
    }

    static MatchResult exact(InternalTransaction i, GatewayTransaction g, BankTransaction b, double conf) {
        return new MatchResult(true, conf, 1, i, g, b);
    }

    static MatchResult noMatch() {
        return new MatchResult(false, 0.0, -1, null, null, null);
    }

    // Getters
    public boolean isMatched() { return matched; }
    public double getConfidence() { return confidence; }
    public int getMatchLevel() { return matchLevel; }
}
```

### 3. DiscrepancyDetector

```java
package com.payment.reconciliation.service;

import org.springframework.stereotype.Service;
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;

@Service
public class DiscrepancyDetector {

    private static final long LARGE_VARIANCE_THRESHOLD = 500_00; // $5000

    public DiscrepancyType classify(UnmatchedTransaction unmatched) {
        if (unmatched.hasGateway() && !unmatched.hasInternal()) {
            return DiscrepancyType.MISSING_INTERNAL;
        }
        if (unmatched.hasInternal() && !unmatched.hasGateway()) {
            return DiscrepancyType.MISSING_GATEWAY;
        }
        if (Math.abs(unmatched.getAmountVariance()) > 1) {
            return DiscrepancyType.AMOUNT_MISMATCH;
        }
        if (isTimingVariance(unmatched)) {
            return DiscrepancyType.TIMING_VARIANCE;
        }
        if (unmatched.isDuplicate()) {
            return DiscrepancyType.DUPLICATE;
        }
        return DiscrepancyType.UNCLASSIFIED;
    }

    private boolean isTimingVariance(UnmatchedTransaction unmatched) {
        if (unmatched.getGatewayTimestamp() == null || unmatched.getInternalTimestamp() == null) {
            return false;
        }
        long daysDiff = ChronoUnit.DAYS.between(
                unmatched.getInternalTimestamp(),
                unmatched.getGatewayTimestamp()
        );
        return daysDiff > 0 && daysDiff <= 2;
    }

    public boolean shouldAlert(UnmatchedTransaction unmatched, DiscrepancyType type) {
        return switch (type) {
            case AMOUNT_MISMATCH -> Math.abs(unmatched.getAmountVariance()) > LARGE_VARIANCE_THRESHOLD;
            case MISSING_GATEWAY, MISSING_INTERNAL -> true;
            case DUPLICATE -> true;
            default -> false;
        };
    }
}

enum DiscrepancyType {
    AMOUNT_MISMATCH, MISSING_INTERNAL, MISSING_GATEWAY, DUPLICATE, TIMING_VARIANCE, UNCLASSIFIED
}
```

### 4. AutoResolutionService

```java
package com.payment.reconciliation.service;

import org.springframework.stereotype.Service;
import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.*;

@Service
public class AutoResolutionService {

    private final AutoResolutionRuleRepository ruleRepository;
    private final Redis redisTemplate;

    public AutoResolutionService(
            AutoResolutionRuleRepository ruleRepository,
            Redis redisTemplate) {
        this.ruleRepository = ruleRepository;
        this.redisTemplate = redisTemplate;
    }

    public boolean canAutoResolve(UnmatchedTransaction unmatched, DiscrepancyType type) {
        return switch (type) {
            case TIMING_VARIANCE -> isKnownGatewayDelay(unmatched);
            case AMOUNT_MISMATCH -> isRoundingError(unmatched);
            case MISSING_GATEWAY -> isMissingDueToKnownDelay(unmatched);
            default -> false;
        };
    }

    public String autoResolve(UnmatchedTransaction unmatched, DiscrepancyType type) {
        return switch (type) {
            case TIMING_VARIANCE -> {
                String ruleApplied = "Auto-cleared: Known gateway delay";
                // Mark as resolved
                yield ruleApplied;
            }
            case AMOUNT_MISMATCH -> {
                String ruleApplied = "Auto-cleared: Rounding error ($" +
                    (unmatched.getAmountVariance() / 100.0) + ")";
                yield ruleApplied;
            }
            case MISSING_GATEWAY -> {
                String ruleApplied = "Auto-cleared: Within 48h SLA";
                yield ruleApplied;
            }
            default -> "No auto-resolution applicable";
        };
    }

    private boolean isKnownGatewayDelay(UnmatchedTransaction unmatched) {
        String gatewayName = unmatched.getGatewayName();
        AutoResolutionRule rule = ruleRepository.findByGatewayName(gatewayName)
                .orElse(null);
        if (rule == null) return false;

        long hoursDiff = ChronoUnit.HOURS.between(
                unmatched.getInternalTimestamp(),
                unmatched.getGatewayTimestamp()
        );
        return hoursDiff <= rule.getMaxDelayHours() && rule.isAutoApprove();
    }

    private boolean isRoundingError(UnmatchedTransaction unmatched) {
        return Math.abs(unmatched.getAmountVariance()) <= 1; // ≤ $0.01
    }

    private boolean isMissingDueToKnownDelay(UnmatchedTransaction unmatched) {
        long hoursSinceInternal = ChronoUnit.HOURS.between(
                unmatched.getInternalTimestamp(),
                LocalDateTime.now()
        );
        // Auto-clear if missing gateway after <48 hours
        return hoursSinceInternal < 48;
    }
}
```

### 5. ReconciliationReportGenerator

```java
package com.payment.reconciliation.report;

import org.springframework.stereotype.Service;
import com.itextpdf.kernel.pdf.PdfDocument;
import com.itextpdf.kernel.pdf.PdfWriter;
import com.itextpdf.layout.Document;
import com.itextpdf.layout.element.*;
import java.io.File;
import java.time.LocalDate;
import java.util.*;

@Service
public class ReconciliationReportGenerator {

    private final MatchedTransactionRepository matchedRepository;
    private final DiscrepancyRecordRepository discrepancyRepository;
    private final ReconciliationBatchRepository batchRepository;

    public ReconciliationReportGenerator(
            MatchedTransactionRepository matchedRepository,
            DiscrepancyRecordRepository discrepancyRepository,
            ReconciliationBatchRepository batchRepository) {
        this.matchedRepository = matchedRepository;
        this.discrepancyRepository = discrepancyRepository;
        this.batchRepository = batchRepository;
    }

    public String generateDailySummaryReport(LocalDate date) throws Exception {
        ReconciliationBatch batch = batchRepository.findByBatchDate(date)
                .orElseThrow(() -> new IllegalArgumentException("No batch for date: " + date));

        File outputFile = new File("/reports/reconciliation_" + date + ".pdf");
        PdfWriter writer = new PdfWriter(outputFile);
        PdfDocument pdfDoc = new PdfDocument(writer);
        Document doc = new Document(pdfDoc);

        // Header
        Paragraph title = new Paragraph("PAYMENT RECONCILIATION REPORT")
                .setBold().setFontSize(16);
        doc.add(title);
        doc.add(new Paragraph("Date: " + date));
        doc.add(new Paragraph("Generated: " + LocalDateTime.now()));
        doc.add(new Paragraph(""));

        // Summary Table
        Table summaryTable = new Table(2);
        summaryTable.addCell("Total Internal Transactions");
        summaryTable.addCell(String.valueOf(batch.getTotalInternalCount()));
        summaryTable.addCell("Total Gateway Transactions");
        summaryTable.addCell(String.valueOf(batch.getTotalGatewayCount()));
        summaryTable.addCell("Total Bank Transactions");
        summaryTable.addCell(String.valueOf(batch.getTotalBankCount()));
        summaryTable.addCell("Matched");
        summaryTable.addCell(String.valueOf(batch.getMatchedCount()));
        summaryTable.addCell("Match Rate (%)");
        double matchRate = (batch.getMatchedCount() / (double) batch.getTotalInternalCount()) * 100;
        summaryTable.addCell(String.format("%.2f%%", matchRate));
        summaryTable.addCell("Discrepancies");
        summaryTable.addCell(String.valueOf(batch.getDiscrepancyCount()));
        summaryTable.addCell("Auto-Resolved");
        summaryTable.addCell(String.valueOf(batch.getAutoResolvedCount()));

        doc.add(summaryTable);
        doc.add(new Paragraph(""));

        // Discrepancy Details
        List<DiscrepancyRecord> discrepancies = discrepancyRepository.findByBatchDate(date);
        if (!discrepancies.isEmpty()) {
            doc.add(new Paragraph("DISCREPANCIES REQUIRING REVIEW").setBold().setFontSize(12));
            Table discTable = new Table(5);
            discTable.addCell("Type");
            discTable.addCell("Count");
            discTable.addCell("Variance");
            discTable.addCell("Status");
            discTable.addCell("Assigned To");

            Map<String, Long> typeCount = discrepancies.stream()
                    .collect(java.util.stream.Collectors.groupingBy(
                            d -> d.getDiscrepancyType().toString(),
                            java.util.stream.Collectors.counting()
                    ));

            for (Map.Entry<String, Long> entry : typeCount.entrySet()) {
                discTable.addCell(entry.getKey());
                discTable.addCell(String.valueOf(entry.getValue()));
                discTable.addCell("—");
                discTable.addCell("PENDING");
                discTable.addCell("TBD");
            }

            doc.add(discTable);
        }

        doc.close();
        return outputFile.getAbsolutePath();
    }

    public String exportToCSV(LocalDate date) {
        List<MatchedTransaction> matched = matchedRepository.findByMatchedAtDate(date);
        StringBuilder csv = new StringBuilder();
        csv.append("internal_tx_id,gateway_tx_id,bank_tx_id,match_confidence,matched_at\n");

        for (MatchedTransaction tx : matched) {
            csv.append(String.format("%d,%d,%d,%.2f,%s\n",
                    tx.getInternalTxId(),
                    tx.getGatewayTxId(),
                    tx.getBankTxId(),
                    tx.getMatchConfidence(),
                    tx.getMatchedAt()
            ));
        }

        File csvFile = new File("/reports/reconciliation_" + date + ".csv");
        java.nio.file.Files.write(csvFile.toPath(), csv.toString().getBytes());
        return csvFile.getAbsolutePath();
    }
}
```

---

## Failure Scenarios

| Scenario | Cause | Mitigation |
|---|---|---|
| **Gateway API timeout** | Stripe/Adyen API down | Retry with exponential backoff (max 3x); use cached last-known state; alert ops |
| **Bank statement file not received** | SFTP transfer failed | Check SFTP logs; requeue file retrieval; skip bank matching for day; alert |
| **Discrepancy explosion** | Batch reconciliation logic bug | Validate batch row count before/after; cap alerts at 1000/day; manual review for >2% variance |
| **Duplicate matches** | Index corruption or query bug | Idempotent writes via unique constraints; use transactions with SERIALIZABLE isolation |
| **Stale Redis cache** | Node crash | Rebuild from PostgreSQL on startup; TTL ensures freshness; skip cache if unavailable |
| **Partition rebalance** | Kafka broker failure | Replicas ensure durability; consumer group handles rebalance automatically |

---

## Scaling Strategy

| Component | Scale-out Method | Bottleneck Addressed |
|---|---|---|
| **Reconciliation Job** | Spring Batch partitions (10–20); increase partition count | Throughput: each partition processes 5K–10K tx |
| **Kafka Topics** | Increase partition count to 30+ | Consumer lag; ensure ≤1 hour lag even at 500K tx/day |
| **PostgreSQL** | Read replicas for reporting; write master for reconciliation | Query load on discrepancy reports |
| **Redis** | Cluster mode (6+ nodes) | High cardinality keys (in_flight, transaction_index) |
| **Reporting** | Async report generation (separate container) | Latency: don't block job for PDF generation |

---

## Monitoring & Alerting

```java
package com.payment.reconciliation.monitoring;

import io.micrometer.core.instrument.*;
import org.springframework.stereotype.Service;

@Service
public class ReconciliationMetrics {

    private final MeterRegistry meterRegistry;

    public ReconciliationMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }

    public void recordBatchCompletion(long internalCount, long matchedCount, long discrepancyCount) {
        meterRegistry.counter("reconciliation.total.transactions", "type", "internal")
                .increment(internalCount);
        meterRegistry.counter("reconciliation.matched.transactions")
                .increment(matchedCount);
        meterRegistry.counter("reconciliation.discrepancies", "type", "total")
                .increment(discrepancyCount);

        double matchRate = (matchedCount / (double) internalCount) * 100;
        meterRegistry.gauge("reconciliation.match.rate.percent", matchRate);

        if (matchRate < 99.5) {
            // Alert: match rate below SLA
            alertingService.sendAlert(
                    Alert.builder()
                            .severity(AlertSeverity.WARNING)
                            .message("Match rate below 99.5%: " + matchRate)
                            .build()
            );
        }
    }

    public void recordDiscrepancy(DiscrepancyType type, long varianceAmount) {
        meterRegistry.counter("reconciliation.discrepancies", "type", type.toString())
                .increment();
        if (Math.abs(varianceAmount) > 500_00) { // $5000
            alertingService.sendAlert(
                    Alert.builder()
                            .severity(AlertSeverity.CRITICAL)
                            .message("High-value discrepancy: " + type + ", variance: $" + (varianceAmount / 100.0))
                            .build()
            );
        }
    }

    public void recordMatchLatency(long durationMs) {
        meterRegistry.timer("reconciliation.match.duration.ms")
                .record(durationMs, java.util.concurrent.TimeUnit.MILLISECONDS);
    }
}
```

**Key Dashboards (Grafana):**
- Match rate (%) over time
- Discrepancy count by type
- Batch completion duration
- Kafka consumer lag (per partition)
- PostgreSQL query duration (reconciliation queries)
- Alert volume

---

## Summary Cheat Sheet

| Concept | Key Takeaway |
|---|---|
| **Matching Strategy** | 3-level hierarchy: exact ID → fuzzy match → similarity scoring |
| **Timing** | T+1 batch with 2-day lookback; known delays auto-cleared after 48h |
| **Discrepancy Types** | AMOUNT_MISMATCH, MISSING_INTERNAL/GATEWAY, DUPLICATE, TIMING_VARIANCE |
| **Auto-Resolution** | Rounding errors (<$0.01), known gateway delays, timing variance (>3 days) |
| **Data Flow** | Gateway/Bank/Internal → Kafka → Reconciliation Job → Discrepancy Detection → Reporting |
| **Spring Batch** | Partitioned job; idempotent; runs daily at T+00:30 |
| **Redis Role** | In-flight batch metadata, discrepancy index, metrics cache |
| **PostgreSQL Schema** | 4 tx tables (internal/gateway/bank/matched), discrepancy_records, reconciliation_batches |
| **Alerts** | High-value variance (>$5K), missing tx (>10), low match rate (<99.5%) |
| **Scaling** | Increase Batch partitions, Kafka partitions, add DB read replicas |
| **Compliance** | Immutable ledger, 7-year retention, daily audit reports |

