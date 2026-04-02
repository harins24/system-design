---
title: Wallet and Store Credit System
layout: default
---

# Wallet and Store Credit System — Deep Dive Design

> **Scenario:** Customers load money into wallet, use balance for purchases, receive refunds to wallet, transfer balance to bank, earn cashback/rewards, track transaction history, enforce limits (max balance $5K, max transaction $2K). **5 million active wallets.**
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
| **Active Wallets** | 5 million customers |
| **Peak TPS** | 10K transactions/sec (wallet operations) |
| **Latency SLA** | <100ms for debit/credit; <200ms for withdrawal |
| **Consistency** | Strong (ACID); no double-spending; eventual consistency for audit |
| **Max Balance** | $5,000 per wallet |
| **Max Transaction** | $2,000 per transaction |
| **Concurrent Ops** | Same wallet: serialized; different wallets: parallel |
| **Refunds** | Auto-credit to wallet; no expiry |
| **Withdrawal** | Bank transfer; ACH (1–3 days); fee $0.50 or 1% (whichever lower) |
| **Cashback** | 1–5% earn on spend; credited within 24 hours |
| **Dispute Period** | 30 days; freezes wallet; provisional credit while investigating |
| **Audit** | Nightly reconciliation; immutable ledger |
| **Compliance** | KYC verified; AML screening; PCI-DSS storage |

---

## Capacity Estimation

| Metric | Calculation | Result |
|---|---|---|
| **Daily Active Wallets** | 5M × 30% | 1.5M wallets |
| **Daily Transactions** | 1.5M × 2.5 avg tx/day | 3.75M tx/day |
| **Peak TPS** | 3.75M ÷ 86400 × 3 (peak factor) | ~130 tx/sec (planned: 10K) |
| **Ledger Entries** | 3.75M tx/day × 365 × 2 years | ~2.7B ledger rows |
| **Storage (PostgreSQL)** | 2.7B × 500 bytes/row | ~1.35 TB |
| **Redis Cache** | 5M wallets × 100 bytes (id + balance) | ~500 MB |
| **Kafka Topics** | wallet_events partitions | 20–30 partitions |
| **Withdrawal Queue** | 1–2% of tx = 37.5K–75K/day | Peak: ~5 withdrawals/sec |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       WALLET & STORE CREDIT SYSTEM                          │
└─────────────────────────────────────────────────────────────────────────────┘

CLIENT LAYER
├─ Mobile App          → Load Wallet (Card/Bank)
├─ Web App             → Transfer to Bank
├─ Merchant POS        → Debit Wallet
└─ Admin Dashboard     → Dispute Management

                              ↓

┌──────────────────────────────────────────────────────────────────────────────┐
│                    WALLET SERVICE (Spring Boot, Multi-Instance)              │
├──────────────────────────────────────────────────────────────────────────────┤
│  API Endpoints:                                                              │
│  · POST /wallet/load              → LoadWalletService                        │
│  · POST /wallet/debit             → DebitWalletService (Advisory Lock)       │
│  · POST /wallet/credit            → CreditWalletService                      │
│  · POST /wallet/withdraw          → WithdrawalService (Async)                │
│  · GET  /wallet/balance           → CacheFirst (Redis → PostgreSQL)          │
│  · GET  /wallet/history           → LedgerRepository                         │
│  · POST /wallet/dispute           → DisputeService (Freeze)                  │
└──────────────────────────────────────────────────────────────────────────────┘
                                    ↓
        ┌───────────────────────────────────────────────────────┐
        │           LOCKING & CONCURRENCY LAYER                 │
        ├───────────────────────────────────────────────────────┤
        │ High Contention Wallets: PostgreSQL SERIALIZABLE      │
        │ Normal Wallets: Advisory Locks (pg_advisoryxact_lock) │
        │ Very Low Contention: Optimistic Locking (version++)   │
        └───────────────────────────────────────────────────────┘
                                    ↓
        ┌───────────────────────────────────────────────────────┐
        │         LEDGER POSTING (Event Sourcing)               │
        ├───────────────────────────────────────────────────────┤
        │ Idempotent: Dedup on (wallet_id, idempotency_key)     │
        │ Types: LOAD, DEBIT, CREDIT, REFUND, WITHDRAWAL, FEE   │
        │ Immutable: INSERT only (no UPDATE)                    │
        │ Balance = SUM(all ledger entries for wallet)           │
        └───────────────────────────────────────────────────────┘
                                    ↓
                    ┌──────────────────────────────┐
                    │    KAFKA EVENT STREAM        │
                    ├──────────────────────────────┤
                    │ wallet_transactions_created  │
                    │ wallet_loads_initiated       │
                    │ wallet_withdrawals_started   │
                    │ wallet_disputes_filed        │
                    │ wallet_balances_updated      │
                    └──────────────────────────────┘
                                    ↓
        ┌───────────────────────────────────────────────────────┐
        │        ASYNC BACKGROUND SERVICES                      │
        ├───────────────────────────────────────────────────────┤
        │ · Withdrawal Processor → Bank API → Settlement         │
        │ · Cashback Accrual → 1–5% calc → Post to ledger        │
        │ · Dispute Processor → Evidence collection → Settlement  │
        │ · Balance Auditor (nightly) → Reconcile balance cache   │
        └───────────────────────────────────────────────────────┘
                                    ↓
        ┌───────────────────────────────────────────────────────┐
        │      DATA LAYER (PostgreSQL + MongoDB + Redis)        │
        ├───────────────────────────────────────────────────────┤
        │ PostgreSQL:         │ MongoDB:              │ Redis:    │
        │ · wallets           │ · dispute_cases       │ · balance │
        │ · wallet_ledger     │ · dispute_evidence    │ · locks   │
        │ · wallet_limits     │ · audit_logs          │ · hints   │
        │ · withdrawals       │                       │           │
        │ · cashback_rules    │                       │           │
        └───────────────────────────────────────────────────────┘

DATA FLOW: Debit Wallet
────────────────────────
1. Client calls POST /wallet/debit {amount, merchant, idempotency_key}
2. WalletService acquires lock (advisory or SERIALIZABLE)
3. Read current balance from PostgreSQL
4. Validate: amount ≤ balance, amount ≤ max_tx, balance - amount ≥ 0
5. Insert DEBIT entry into wallet_ledger (with idempotency_key)
6. Update Redis balance cache (TTL 1 hour)
7. Publish wallet_transaction_created event to Kafka
8. Release lock; return 200 OK with new balance
9. If fails: Rollback; return 400 Bad Request

EDGE CASES:
───────────
· Double debit (duplicate request): Caught by unique(wallet_id, idempotency_key)
· Concurrent debits: Locked wallet serializes; other threads wait
· Balance boundary: Checked before INSERT; prevents overdraft
· Withdrawal limits: Max 5 outstanding; Max $10K/day
```

---

## Core Design Questions Answered

### 1. How do you structure the wallet ledger for accuracy?

**Event Sourcing Ledger Model:**

- **Immutable**: All entries are `INSERT` only; no `UPDATE` or `DELETE`
- **Double-entry**: Each DEBIT has a corresponding merchant/bank CREDIT (balanced)
- **Idempotency**: Primary key: `(wallet_id, idempotency_key, entry_type)`
- **Balance Computation**: `balance = SUM(amount) WHERE wallet_id = ? AND deleted_at IS NULL`
- **Retention**: 7 years; older entries archived to cold storage
- **Timestamps**: microsecond precision for ordering within same millisecond

```
wallet_ledger:
├─ id (BIGSERIAL)
├─ wallet_id (FK)
├─ entry_type (LOAD, DEBIT, CREDIT, REFUND, WITHDRAWAL, FEE, CASHBACK)
├─ amount_cents (positive or negative)
├─ currency
├─ reference_id (order_id, refund_id, withdrawal_id, cashback_id)
├─ idempotency_key (UUID; ensures no double-posting)
├─ description
├─ created_at (NOW() at insert)
├─ deleted_at (NULL unless reversed)
└─ metadata (JSONB: merchant_id, reason, correlation_id)

Index: wallet_id, created_at (for efficient balance calculation)
```

### 2. How do you prevent double-spending from wallet?

**Multi-layer prevention:**

1. **SQL Constraint**: `CHECK (amount_cents >= 0)` on balance transfer
2. **Pessimistic Lock**: `SELECT FOR UPDATE` on wallet row before debit
3. **Idempotency Key**: `UNIQUE(wallet_id, idempotency_key)` prevents duplicate ledger entries
4. **Atomic Balance Check**: Read balance + subtract + check ≥ 0 in single transaction
5. **Version Check**: Optional optimistic lock on wallet.version for non-critical wallets

```
Debit Operation (Serializable Isolation):
────────────────────────────────────────
BEGIN TRANSACTION;
  SELECT balance, version FROM wallets WHERE id = ? FOR UPDATE;
  -- balance = 500, version = 5

  IF balance >= amount AND amount ≤ max_transaction THEN
    INSERT INTO wallet_ledger (..., amount = -amount, idempotency_key = ?)
      ON CONFLICT DO NOTHING; -- Idempotent

    UPDATE wallets SET balance = balance - amount, version = version + 1
      WHERE id = ? AND version = 5;

    IF rows_affected = 0 THEN ROLLBACK; -- Concurrent update; retry
    ELSE COMMIT;
  ELSE
    ROLLBACK; -- Insufficient balance
  END IF;
END TRANSACTION;
```

### 3. How do you handle concurrent wallet transactions?

**Tiered locking strategy:**

| Wallet Type | Contention | Lock Method | Latency |
|---|---|---|---|
| High-traffic (>100 tx/hour) | High | PostgreSQL SERIALIZABLE | 50–100ms |
| Normal (1–100 tx/hour) | Medium | Advisory lock (pg_advisoryxact_lock) | 20–50ms |
| Low-traffic (<1 tx/hour) | Low | Optimistic version check | <10ms |

**Advisory Locks** (for medium contention):
```sql
SELECT pg_advisory_xact_lock(wallet_id::bigint);
-- Holds lock until transaction end; auto-released

-- Multiple wallets: Lock in order to avoid deadlock
SELECT pg_advisory_xact_lock(wallet_id1), pg_advisory_xact_lock(wallet_id2);
```

**Handling Deadlocks:**
- Retry logic with exponential backoff (3 retries)
- If deadlock after 3 retries: Return 503 Service Unavailable
- Alert ops; investigate lock contention

### 4. How do you implement wallet withdrawal to bank?

**Async withdrawal flow with state machine:**

```
State Transitions:
──────────────────
PENDING_VERIFICATION → VERIFICATION_PASSED → PENDING_ACH →
  ACH_SENT → ACH_COMPLETED → FINISHED

  OR

PENDING_VERIFICATION → VERIFICATION_FAILED → CANCELLED
ACH_SENT → ACH_FAILED → RETRY → ACH_SENT → ...
```

**Withdrawal Service Process:**

1. User initiates withdrawal via `/wallet/withdraw {amount, bank_account_id}`
2. Freeze amount from balance (soft hold; not deducted from ledger)
3. Verify bank account (Plaid or internal vault)
4. Calculate fee: `fee = max(50_cents, amount * 0.01)`
5. Create withdrawal record in `PENDING_VERIFICATION` state
6. Publish `wallet_withdrawal_initiated` event to Kafka
7. Background job: ACH provider (e.g., Dwolla) processes; updates state
8. On success: Post WITHDRAWAL + FEE entries to ledger
9. On failure: Unfreeze amount; notify user; allow retry

### 5. How do you audit wallet balances for accuracy?

**Nightly reconciliation job:**

```
Audit Schedule: Daily 02:00 UTC
───────────────────────────────
FOR each wallet_id IN wallets:
  1. computed_balance = SUM(amount) FROM wallet_ledger WHERE wallet_id = ?
  2. cached_balance = GET balance:{wallet_id} FROM Redis
  3. actual_balance = SELECT balance FROM wallets WHERE id = ?

  IF computed_balance != actual_balance:
    → Create audit_discrepancy record
    → Alert ops (critical if variance > $10)
    → Propose correction: UPDATE balance to computed_balance

  IF cached_balance != computed_balance:
    → Log cache miss; update Redis
```

### 6. How do you handle wallet disputes?

**Dispute flow with evidence collection:**

```
States: OPEN → INVESTIGATING → RESOLVED (or APPROVED/DENIED)
──────────────────────────────────────────────────────────────

1. Customer files dispute: POST /wallet/disputes {transaction_id, reason}
2. Wallet is FROZEN (no debits allowed)
3. Provisional credit posted to ledger (reversible)
4. Evidence collected:
   - Transaction receipt / merchant proof
   - Device fingerprint
   - Location mismatch check
5. Investigator reviews (ops team)
6. Decision: APPROVED (provisional credit becomes permanent) or DENIED (reversed)
7. Notify customer
8. Unfreeze wallet
```

---

## Microservices Breakdown

| Service | Responsibility | Tech | Deployment |
|---|---|---|---|
| **Wallet Core Service** | Load, debit, credit, balance query | Spring Boot + Kubernetes | 10 instances |
| **Withdrawal Service** | Process ACH, update state, retry logic | Spring Boot + Scheduled | 3 instances |
| **Dispute Service** | Case management, evidence collection | Spring Boot + MongoDB | 2 instances |
| **Cashback Service** | Calculate and post cashback earnings | Spring Boot + Kafka Streams | 2 instances |
| **Audit Service** | Nightly balance reconciliation | Spring Boot + Quartz | 1 instance |
| **Notification Service** | Send SMS/email on transactions, disputes | Spring Boot + SQS | 3 instances |

---

## Database Design (DDL)

```sql
-- PostgreSQL

CREATE TABLE wallets (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL UNIQUE,
    balance_cents BIGINT NOT NULL DEFAULT 0,
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(20) DEFAULT 'ACTIVE', -- ACTIVE, FROZEN, CLOSED
    max_balance_cents BIGINT DEFAULT 500000, -- $5000
    max_transaction_cents BIGINT DEFAULT 200000, -- $2000
    version BIGINT DEFAULT 1, -- For optimistic locking
    kyc_verified BOOLEAN DEFAULT FALSE,
    aml_screened BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP,
    CONSTRAINT positive_balance CHECK (balance_cents >= 0),
    CONSTRAINT balance_under_max CHECK (balance_cents <= max_balance_cents)
);

CREATE TABLE wallet_ledger (
    id BIGSERIAL PRIMARY KEY,
    wallet_id BIGINT NOT NULL REFERENCES wallets(id) ON DELETE RESTRICT,
    entry_type VARCHAR(30) NOT NULL, -- LOAD, DEBIT, CREDIT, REFUND, WITHDRAWAL, FEE, CASHBACK
    amount_cents BIGINT NOT NULL, -- Can be negative for debits/fees
    currency VARCHAR(3),
    reference_id VARCHAR(100), -- order_id, refund_id, withdrawal_id
    idempotency_key UUID NOT NULL,
    description TEXT,
    merchant_id VARCHAR(100),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    deleted_at TIMESTAMP, -- Soft delete for reversal
    metadata JSONB,
    UNIQUE(wallet_id, idempotency_key)
);

CREATE TABLE wallet_limits (
    id BIGSERIAL PRIMARY KEY,
    wallet_id BIGINT NOT NULL UNIQUE REFERENCES wallets(id),
    max_daily_withdrawal_cents BIGINT DEFAULT 1000000, -- $10K/day
    max_outstanding_withdrawals INT DEFAULT 5,
    withdrawal_count_today INT DEFAULT 0,
    withdrawal_total_today_cents BIGINT DEFAULT 0,
    reset_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP + INTERVAL '1 day'
);

CREATE TABLE withdrawals (
    id BIGSERIAL PRIMARY KEY,
    wallet_id BIGINT NOT NULL REFERENCES wallets(id),
    amount_cents BIGINT NOT NULL,
    fee_cents BIGINT NOT NULL,
    bank_account_id BIGINT NOT NULL,
    status VARCHAR(30) DEFAULT 'PENDING_VERIFICATION', -- PENDING_VERIFICATION, VERIFICATION_PASSED, PENDING_ACH, ACH_SENT, ACH_COMPLETED, FAILED
    ach_reference_id VARCHAR(100), -- From ACH provider (Dwolla, etc.)
    error_message TEXT,
    initiated_at TIMESTAMP NOT NULL DEFAULT NOW(),
    completed_at TIMESTAMP,
    retry_count INT DEFAULT 0
);

CREATE TABLE disputes (
    id BIGSERIAL PRIMARY KEY,
    wallet_id BIGINT NOT NULL REFERENCES wallets(id),
    ledger_entry_id BIGINT NOT NULL REFERENCES wallet_ledger(id),
    reason VARCHAR(255),
    description TEXT,
    status VARCHAR(30) DEFAULT 'OPEN', -- OPEN, INVESTIGATING, RESOLVED
    decision VARCHAR(20), -- APPROVED, DENIED
    provisional_credit_cents BIGINT DEFAULT 0,
    resolution_notes TEXT,
    assigned_to VARCHAR(100),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    resolved_at TIMESTAMP
);

CREATE TABLE cashback_rules (
    id BIGSERIAL PRIMARY KEY,
    merchant_category VARCHAR(50),
    earn_rate DECIMAL(5,2), -- 1.00 = 1%, 5.00 = 5%
    cap_cents BIGINT DEFAULT 500000, -- $5K cap per year
    active BOOLEAN DEFAULT TRUE
);

CREATE INDEX idx_wallet_ledger_wallet_created ON wallet_ledger(wallet_id, created_at);
CREATE INDEX idx_wallet_ledger_idempotency ON wallet_ledger(wallet_id, idempotency_key);
CREATE INDEX idx_withdrawals_wallet_status ON withdrawals(wallet_id, status);
CREATE INDEX idx_disputes_wallet_status ON disputes(wallet_id, status);
```

---

## Redis Data Structures

```
Key Pattern                      Type        TTL    Purpose
─────────────────────────────────────────────────────────────
balance:{wallet_id}              String      1h     Current balance (JSON: {cents, version, updated_at})
lock:{wallet_id}                 String      5min   Distributed lock (value = request_id)
withdrawal_limits:{wallet_id}    Hash        24h    withdrawal_count, withdrawal_total, reset_at
dispute_cases:{wallet_id}        ZSet        7d     Active disputes (score = created_at)
cashback_earned:{wallet_id}:{m}  String      0      Cashback earned this month (monthly rotation)
pending_withdrawals:{wallet_id}  List        ongoing Queued withdrawals (FIFO)
audit_discrepancies              HyperLogLog 30d    Wallets with balance mismatches
```

---

## Kafka Event Flow

```
Topic: wallet_transactions_created (30 partitions, key = wallet_id)
├─ Payload: { wallet_id, entry_type, amount, reference_id, created_at }
├─ Consumers: CashbackService, NotificationService, AnalyticsService
└─ Retention: 30 days

Topic: wallet_loads_initiated (15 partitions, key = wallet_id)
├─ Published by: WalletService
├─ Payload: { wallet_id, amount, payment_method, reference_id }
└─ Consumer: PaymentGatewayService (charge card/bank)

Topic: wallet_withdrawals_started (10 partitions, key = wallet_id)
├─ Published by: WithdrawalService
├─ Payload: { withdrawal_id, wallet_id, amount, bank_account_id }
└─ Consumer: ACHProcessorService (send to Dwolla)

Topic: wallet_disputes_filed (8 partitions, key = wallet_id)
├─ Published by: DisputeService
├─ Payload: { dispute_id, wallet_id, ledger_entry_id, reason }
└─ Consumers: NotificationService, DisputeInvestigationQueue

Topic: wallet_balances_updated (10 partitions, key = wallet_id)
├─ Published by: WalletService (every significant operation)
├─ Payload: { wallet_id, new_balance, operation_type }
└─ Consumer: AnalyticsService, CacheInvalidationService
```

---

## Implementation Code

### 1. WalletService (Core Operations)

```java
package com.payment.wallet.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.dao.DataIntegrityViolationException;
import java.util.*;
import java.time.Instant;

@Service
public class WalletService {

    private final WalletRepository walletRepository;
    private final WalletLedgerRepository ledgerRepository;
    private final WalletLockManager lockManager;
    private final RedisTemplate<String, String> redis;
    private final KafkaTemplate<String, WalletEvent> kafkaTemplate;

    public WalletService(
            WalletRepository walletRepository,
            WalletLedgerRepository ledgerRepository,
            WalletLockManager lockManager,
            RedisTemplate<String, String> redis,
            KafkaTemplate<String, WalletEvent> kafkaTemplate) {
        this.walletRepository = walletRepository;
        this.ledgerRepository = ledgerRepository;
        this.lockManager = lockManager;
        this.redis = redis;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public WalletBalance debitWallet(
            long walletId,
            long amountCents,
            String merchantId,
            String orderId,
            UUID idempotencyKey) throws WalletException {

        // Acquire distributed lock
        String lockToken = lockManager.acquireLock(walletId, Duration.ofSeconds(10));
        if (lockToken == null) {
            throw new WalletException("Wallet locked; another operation in progress");
        }

        try {
            // Fetch wallet with lock
            Wallet wallet = walletRepository.findByIdForUpdate(walletId)
                    .orElseThrow(() -> new WalletException("Wallet not found"));

            // Validations
            if (!WalletStatus.ACTIVE.equals(wallet.getStatus())) {
                throw new WalletException("Wallet is " + wallet.getStatus());
            }
            if (wallet.getBalanceCents() < amountCents) {
                throw new InsufficientBalanceException(
                        "Balance: " + wallet.getBalanceCents() + ", required: " + amountCents);
            }
            if (amountCents > wallet.getMaxTransactionCents()) {
                throw new WalletException("Amount exceeds max transaction limit");
            }

            // Insert ledger entry (idempotent)
            WalletLedgerEntry entry = new WalletLedgerEntry();
            entry.setWalletId(walletId);
            entry.setEntryType(EntryType.DEBIT);
            entry.setAmountCents(-amountCents);
            entry.setReferenceId(orderId);
            entry.setIdempotencyKey(idempotencyKey);
            entry.setDescription("Purchase from " + merchantId);
            entry.setMetadata(Map.of("merchant_id", merchantId));

            try {
                ledgerRepository.save(entry);
            } catch (DataIntegrityViolationException e) {
                // Duplicate idempotency key; return existing balance
                WalletBalance existingBalance = getWalletBalance(walletId);
                return existingBalance;
            }

            // Update wallet balance and version
            wallet.setBalanceCents(wallet.getBalanceCents() - amountCents);
            wallet.setVersion(wallet.getVersion() + 1);
            walletRepository.save(wallet);

            // Update Redis cache
            redis.opsForValue().set(
                    "balance:" + walletId,
                    new WalletBalance(wallet.getBalanceCents(), wallet.getVersion(), Instant.now()).toJson(),
                    Duration.ofHours(1)
            );

            // Publish event
            publishEvent(new WalletEvent(
                    walletId,
                    "DEBIT",
                    amountCents,
                    orderId,
                    Instant.now()
            ));

            return new WalletBalance(wallet.getBalanceCents(), wallet.getVersion(), Instant.now());

        } finally {
            lockManager.releaseLock(walletId, lockToken);
        }
    }

    @Transactional
    public WalletBalance creditWallet(
            long walletId,
            long amountCents,
            String reason,
            String referenceId,
            UUID idempotencyKey) throws WalletException {

        Wallet wallet = walletRepository.findById(walletId)
                .orElseThrow(() -> new WalletException("Wallet not found"));

        if (wallet.getBalanceCents() + amountCents > wallet.getMaxBalanceCents()) {
            throw new WalletException("Credit would exceed max balance limit");
        }

        // Insert ledger entry
        WalletLedgerEntry entry = new WalletLedgerEntry();
        entry.setWalletId(walletId);
        entry.setEntryType(EntryType.CREDIT);
        entry.setAmountCents(amountCents);
        entry.setReferenceId(referenceId);
        entry.setIdempotencyKey(idempotencyKey);
        entry.setDescription(reason);

        try {
            ledgerRepository.save(entry);
        } catch (DataIntegrityViolationException e) {
            return getWalletBalance(walletId);
        }

        // Update wallet
        wallet.setBalanceCents(wallet.getBalanceCents() + amountCents);
        wallet.setVersion(wallet.getVersion() + 1);
        walletRepository.save(wallet);

        // Invalidate Redis cache
        redis.delete("balance:" + walletId);

        // Publish event
        publishEvent(new WalletEvent(
                walletId,
                "CREDIT",
                amountCents,
                referenceId,
                Instant.now()
        ));

        return new WalletBalance(wallet.getBalanceCents(), wallet.getVersion(), Instant.now());
    }

    public WalletBalance getWalletBalance(long walletId) {
        // Try Redis first
        String cached = redis.opsForValue().get("balance:" + walletId);
        if (cached != null) {
            return WalletBalance.fromJson(cached);
        }

        // Compute from ledger
        long computedBalance = ledgerRepository.computeBalance(walletId);
        Wallet wallet = walletRepository.findById(walletId)
                .orElseThrow(() -> new WalletException("Wallet not found"));

        WalletBalance balance = new WalletBalance(computedBalance, wallet.getVersion(), Instant.now());

        // Cache for 1 hour
        redis.opsForValue().set(
                "balance:" + walletId,
                balance.toJson(),
                Duration.ofHours(1)
        );

        return balance;
    }

    @Transactional(readOnly = true)
    public List<WalletTransaction> getTransactionHistory(long walletId, int limit) {
        return ledgerRepository.findByWalletIdOrderByCreatedAtDesc(walletId, PageRequest.of(0, limit))
                .stream()
                .map(entry -> new WalletTransaction(
                        entry.getId(),
                        entry.getEntryType(),
                        entry.getAmountCents(),
                        entry.getReferenceId(),
                        entry.getCreatedAt()
                ))
                .collect(Collectors.toList());
    }

    private void publishEvent(WalletEvent event) {
        try {
            kafkaTemplate.send("wallet_transactions_created", String.valueOf(event.getWalletId()), event);
        } catch (Exception e) {
            logger.error("Failed to publish wallet event: " + e.getMessage());
            // Non-blocking; don't fail the transaction
        }
    }
}
```

### 2. WalletLockManager (Concurrency Control)

```java
package com.payment.wallet.service;

import org.springframework.stereotype.Component;
import java.util.*;
import java.util.concurrent.TimeUnit;

@Component
public class WalletLockManager {

    private final JdbcTemplate jdbcTemplate;
    private final RedisTemplate<String, String> redis;
    private static final String LOCK_KEY_PREFIX = "lock:";

    public WalletLockManager(JdbcTemplate jdbcTemplate, RedisTemplate<String, String> redis) {
        this.jdbcTemplate = jdbcTemplate;
        this.redis = redis;
    }

    public String acquireLock(long walletId, Duration timeout) {
        // Attempt PostgreSQL advisory lock first (preferred for high contention)
        try {
            boolean locked = jdbcTemplate.query(
                    "SELECT pg_advisory_xact_lock(?)",
                    new Object[]{walletId},
                    rs -> true
            );
            if (locked) {
                return "pg_lock_" + walletId;
            }
        } catch (Exception e) {
            // Fall back to Redis
        }

        // Redis fallback
        String lockToken = UUID.randomUUID().toString();
        String lockKey = LOCK_KEY_PREFIX + walletId;

        boolean acquired = redis.opsForValue().setIfAbsent(
                lockKey,
                lockToken,
                timeout
        );

        return acquired ? lockToken : null;
    }

    public void releaseLock(long walletId, String lockToken) {
        String lockKey = LOCK_KEY_PREFIX + walletId;
        String currentToken = redis.opsForValue().get(lockKey);

        if (lockToken.equals(currentToken)) {
            redis.delete(lockKey);
        }
    }

    // Advisory lock for high-contention wallets (called from batch process)
    public void lockWalletsInOrder(List<Long> walletIds) {
        List<Long> sorted = walletIds.stream().sorted().collect(Collectors.toList());

        for (Long walletId : sorted) {
            jdbcTemplate.query(
                    "SELECT pg_advisory_xact_lock(?)",
                    new Object[]{walletId},
                    rs -> true
            );
        }
    }
}
```

### 3. WithdrawalService (Async Withdrawal Processing)

```java
package com.payment.wallet.service;

import org.springframework.stereotype.Service;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.transaction.annotation.Transactional;
import java.util.*;

@Service
public class WithdrawalService {

    private final WithdrawalRepository withdrawalRepository;
    private final WalletService walletService;
    private final ACHProviderClient achProviderClient;
    private final BankAccountRepository bankAccountRepository;
    private final KafkaTemplate<String, WithdrawalEvent> kafkaTemplate;

    public WithdrawalService(
            WithdrawalRepository withdrawalRepository,
            WalletService walletService,
            ACHProviderClient achProviderClient,
            BankAccountRepository bankAccountRepository,
            KafkaTemplate<String, WithdrawalEvent> kafkaTemplate) {
        this.withdrawalRepository = withdrawalRepository;
        this.walletService = walletService;
        this.achProviderClient = achProviderClient;
        this.bankAccountRepository = bankAccountRepository;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Transactional
    public WithdrawalRecord initiateWithdrawal(
            long walletId,
            long amountCents,
            long bankAccountId) throws WithdrawalException {

        // Verify bank account ownership
        BankAccount bankAccount = bankAccountRepository.findById(bankAccountId)
                .orElseThrow(() -> new WithdrawalException("Bank account not found"));

        if (!bankAccount.getWalletId().equals(walletId)) {
            throw new WithdrawalException("Bank account does not belong to wallet");
        }

        // Validate withdrawal limits
        WalletLimits limits = walletLimitRepository.findByWalletId(walletId)
                .orElseThrow();

        if (limits.getWithdrawalCountToday() >= limits.getMaxOutstandingWithdrawals()) {
            throw new WithdrawalException("Daily withdrawal limit reached");
        }
        if (limits.getWithdrawalTotalTodayCents() + amountCents > limits.getMaxDailyWithdrawalCents()) {
            throw new WithdrawalException("Daily limit exceeded");
        }

        // Calculate fee
        long feeCents = Math.max(50, (long) (amountCents * 0.01));

        // Create withdrawal record
        Withdrawal withdrawal = new Withdrawal();
        withdrawal.setWalletId(walletId);
        withdrawal.setAmountCents(amountCents);
        withdrawal.setFeeCents(feeCents);
        withdrawal.setBankAccountId(bankAccountId);
        withdrawal.setStatus(WithdrawalStatus.PENDING_VERIFICATION);
        withdrawal.setInitiatedAt(LocalDateTime.now());

        withdrawal = withdrawalRepository.save(withdrawal);

        // Freeze amount (no ledger entry yet; balance check only)
        // In production: soft-hold in wallet_limits.outstanding_withdrawal_cents

        // Publish event
        kafkaTemplate.send("wallet_withdrawals_started",
                String.valueOf(walletId),
                new WithdrawalEvent(withdrawal.getId(), walletId, amountCents));

        return withdrawal;
    }

    @Scheduled(fixedDelay = 60000) // Every 60 seconds
    @Transactional
    public void processWithdrawals() {
        List<Withdrawal> pending = withdrawalRepository.findByStatus(
                WithdrawalStatus.PENDING_VERIFICATION,
                PageRequest.of(0, 100)
        ).getContent();

        for (Withdrawal w : pending) {
            try {
                // Verify bank account via Plaid
                boolean verified = verifyBankAccount(w.getBankAccountId());
                if (!verified) {
                    w.setStatus(WithdrawalStatus.VERIFICATION_FAILED);
                    w.setErrorMessage("Bank account verification failed");
                    withdrawalRepository.save(w);
                    continue;
                }

                w.setStatus(WithdrawalStatus.VERIFICATION_PASSED);
                withdrawalRepository.save(w);

            } catch (Exception e) {
                logger.error("Withdrawal verification failed for ID: " + w.getId(), e);
            }
        }

        // Send to ACH provider
        List<Withdrawal> verifiedPending = withdrawalRepository.findByStatus(
                WithdrawalStatus.VERIFICATION_PASSED,
                PageRequest.of(0, 50)
        ).getContent();

        for (Withdrawal w : verifiedPending) {
            try {
                ACHTransferRequest request = new ACHTransferRequest(
                        w.getAmountCents(),
                        w.getBankAccountId(),
                        "Wallet withdrawal"
                );

                ACHTransferResponse response = achProviderClient.initiateTransfer(request);

                w.setStatus(WithdrawalStatus.ACH_SENT);
                w.setAchReferenceId(response.getTransferId());
                withdrawalRepository.save(w);

            } catch (Exception e) {
                w.setRetryCount(w.getRetryCount() + 1);
                if (w.getRetryCount() >= 3) {
                    w.setStatus(WithdrawalStatus.FAILED);
                    w.setErrorMessage("Max retries exceeded: " + e.getMessage());
                }
                withdrawalRepository.save(w);
            }
        }

        // Poll for completion
        List<Withdrawal> inProgress = withdrawalRepository.findByStatus(
                WithdrawalStatus.ACH_SENT,
                PageRequest.of(0, 100)
        ).getContent();

        for (Withdrawal w : inProgress) {
            try {
                ACHStatus status = achProviderClient.checkTransferStatus(w.getAchReferenceId());

                if (status == ACHStatus.COMPLETED) {
                    // Post WITHDRAWAL + FEE entries to ledger
                    walletService.creditWallet(
                            w.getWalletId(),
                            -(w.getAmountCents() + w.getFeeCents()),
                            "Withdrawal to bank",
                            "WITHDRAWAL_" + w.getId(),
                            UUID.randomUUID()
                    );

                    w.setStatus(WithdrawalStatus.ACH_COMPLETED);
                    w.setCompletedAt(LocalDateTime.now());
                    withdrawalRepository.save(w);

                    // Notify customer
                    notificationService.sendWithdrawalComplete(w.getWalletId());

                } else if (status == ACHStatus.FAILED) {
                    w.setStatus(WithdrawalStatus.FAILED);
                    w.setErrorMessage("ACH transfer failed");
                    withdrawalRepository.save(w);
                    notificationService.sendWithdrawalFailed(w.getWalletId());
                }

            } catch (Exception e) {
                logger.error("Failed to poll ACH status for withdrawal: " + w.getId(), e);
            }
        }
    }

    private boolean verifyBankAccount(long bankAccountId) {
        // Integrate with Plaid API
        BankAccount account = bankAccountRepository.findById(bankAccountId).orElseThrow();
        PlaidClient client = new PlaidClient(plaidConfig);
        return client.verifyAccount(account.getPlaidAccountToken());
    }
}
```

### 4. WalletAuditJob (Nightly Reconciliation)

```java
package com.payment.wallet.batch;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.*;

@Service
public class WalletAuditJob {

    private final WalletRepository walletRepository;
    private final WalletLedgerRepository ledgerRepository;
    private final RedisTemplate<String, String> redis;
    private final AuditDiscrepancyRepository auditRepository;

    @Scheduled(cron = "0 0 2 * * *") // Daily 02:00 UTC
    @Transactional
    public void reconcileAllWallets() {
        logger.info("Starting wallet audit reconciliation");

        List<Wallet> allWallets = walletRepository.findAll();
        int discrepancyCount = 0;
        int criticalCount = 0;

        for (Wallet wallet : allWallets) {
            long computedBalance = ledgerRepository.computeBalance(wallet.getId());
            long storedBalance = wallet.getBalanceCents();

            if (computedBalance != storedBalance) {
                discrepancyCount++;

                long variance = Math.abs(computedBalance - storedBalance);

                // Log discrepancy
                AuditDiscrepancy disc = new AuditDiscrepancy();
                disc.setWalletId(wallet.getId());
                disc.setComputedBalance(computedBalance);
                disc.setStoredBalance(storedBalance);
                disc.setVariance(variance);
                auditRepository.save(disc);

                // Alert if significant
                if (variance > 1000) { // $10
                    criticalCount++;
                    alertingService.sendAlert(
                            Alert.builder()
                                    .severity(AlertSeverity.CRITICAL)
                                    .message("Wallet balance discrepancy: " + wallet.getId() +
                                            ", variance: $" + (variance / 100.0))
                                    .build()
                    );

                    // Auto-correct
                    wallet.setBalanceCents(computedBalance);
                    walletRepository.save(wallet);
                    redis.delete("balance:" + wallet.getId());
                }
            }
        }

        logger.info("Wallet audit complete. Discrepancies: " + discrepancyCount + ", Critical: " + criticalCount);

        // Publish metric
        meterRegistry.counter("wallet.audit.discrepancies", "severity", "all")
                .increment(discrepancyCount);
        meterRegistry.counter("wallet.audit.discrepancies", "severity", "critical")
                .increment(criticalCount);
    }
}
```

### 5. WalletDisputeService

```java
package com.payment.wallet.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.*;

@Service
public class WalletDisputeService {

    private final DisputeRepository disputeRepository;
    private final WalletService walletService;
    private final WalletRepository walletRepository;
    private final MongoTemplate mongoTemplate;

    @Transactional
    public Dispute fileDispute(
            long walletId,
            long ledgerEntryId,
            String reason,
            String description) throws DisputeException {

        // Fetch wallet and freeze
        Wallet wallet = walletRepository.findById(walletId)
                .orElseThrow(() -> new DisputeException("Wallet not found"));

        if (WalletStatus.FROZEN.equals(wallet.getStatus())) {
            throw new DisputeException("Wallet already frozen");
        }

        wallet.setStatus(WalletStatus.FROZEN);
        walletRepository.save(wallet);

        // Create dispute
        Dispute dispute = new Dispute();
        dispute.setWalletId(walletId);
        dispute.setLedgerEntryId(ledgerEntryId);
        dispute.setReason(reason);
        dispute.setDescription(description);
        dispute.setStatus(DisputeStatus.OPEN);
        dispute.setCreatedAt(LocalDateTime.now());

        dispute = disputeRepository.save(dispute);

        // Post provisional credit if amount > $50
        WalletLedgerEntry entry = ledgerRepository.findById(ledgerEntryId).orElseThrow();
        if (Math.abs(entry.getAmountCents()) > 5000) {
            long provisionalCredit = Math.abs(entry.getAmountCents());
            walletService.creditWallet(
                    walletId,
                    provisionalCredit,
                    "Dispute provisional credit",
                    "DISPUTE_" + dispute.getId(),
                    UUID.randomUUID()
            );

            dispute.setProvisionalCreditCents(provisionalCredit);
            disputeRepository.save(dispute);
        }

        return dispute;
    }

    @Transactional
    public void resolveDispute(
            long disputeId,
            DisputeDecision decision,
            String notes) throws DisputeException {

        Dispute dispute = disputeRepository.findById(disputeId)
                .orElseThrow(() -> new DisputeException("Dispute not found"));

        dispute.setStatus(DisputeStatus.RESOLVED);
        dispute.setDecision(decision);
        dispute.setResolutionNotes(notes);
        dispute.setResolvedAt(LocalDateTime.now());

        disputeRepository.save(dispute);

        // Unfreeze wallet
        Wallet wallet = walletRepository.findById(dispute.getWalletId()).orElseThrow();
        wallet.setStatus(WalletStatus.ACTIVE);
        walletRepository.save(wallet);

        // Reverse provisional credit if denied
        if (decision == DisputeDecision.DENIED) {
            walletService.creditWallet(
                    dispute.getWalletId(),
                    -dispute.getProvisionalCreditCents(),
                    "Dispute denied; reversing provisional credit",
                    "DISPUTE_REVERSAL_" + dispute.getId(),
                    UUID.randomUUID()
            );
        }
    }
}
```

---

## Failure Scenarios

| Scenario | Cause | Mitigation |
|---|---|---|
| **Double debit** | Duplicate request | Idempotency key + unique constraint prevents duplicate ledger entry |
| **Concurrent debit race** | Lock contention | Advisory locks ensure serialization; retry on deadlock |
| **Withdrawal ACH fails** | Bank account closed | Retry queue; alert user; allow retry with different account |
| **Balance cache stale** | Redis node crash | Rebuild from ledger on startup; compute fresh on cache miss |
| **Ledger sum incorrect** | Index corruption | Nightly audit catches; auto-correct wallet balance |
| **Frozen wallet thaw fails** | Dispute service crash | Manual admin override; idempotent thaw operation |

---

## Scaling Strategy

| Component | Scale-out Method | Bottleneck |
|---|---|---|
| **Wallet Service** | Horizontal (Spring Cloud load balancer) | Lock contention on hot wallets |
| **Ledger Query** | Read replicas (PostgreSQL streaming) | Large `SUM()` queries during audit |
| **Redis** | Cluster mode (6+ nodes) | High-cardinality key space |
| **Withdrawal Processing** | Increase scheduled job frequency | ACH provider rate limits (500/min) |
| **Dispute Cases** | MongoDB sharding by wallet_id | Document growth over time |

---

## Monitoring & Alerting

```java
package com.payment.wallet.monitoring;

import io.micrometer.core.instrument.*;

@Service
public class WalletMetrics {

    private final MeterRegistry meterRegistry;

    public void recordDebit(long amountCents, boolean success) {
        meterRegistry.counter("wallet.debit.count", "success", String.valueOf(success))
                .increment();
        if (success) {
            meterRegistry.timer("wallet.debit.duration")
                    .record(Duration.ofMillis(20)); // Rough estimate
        }
    }

    public void recordAuditDiscrepancy(long variance) {
        meterRegistry.gauge("wallet.audit.variance.cents", variance);
        if (variance > 1000) {
            alertingService.sendAlert(AlertSeverity.CRITICAL, "Large wallet discrepancy");
        }
    }

    public void recordWithdrawalCompletion(WithdrawalStatus status) {
        meterRegistry.counter("wallet.withdrawal.completed", "status", status.toString())
                .increment();
    }
}
```

---

## Summary Cheat Sheet

| Concept | Key Takeaway |
|---|---|
| **Ledger Model** | Event sourcing; INSERT-only; idempotency key prevents double-posting |
| **Double-Spend Prevention** | SERIALIZABLE isolation + optimistic version check + advisory locks |
| **Concurrency** | High contention: PostgreSQL SERIALIZABLE; Medium: Advisory locks; Low: Optimistic |
| **Withdrawal** | Async; state machine (PENDING → ACH_SENT → COMPLETED); 1–3 day latency |
| **Audit** | Nightly job; compute balance from ledger; alert on >$10 variance |
| **Disputes** | FREEZE wallet; provisional credit; 30-day window; RESOLVE with APPROVED/DENIED |
| **Caching** | Redis for balance (1h TTL); cold store for ledger audit; invalidate on credit/debit |
| **Scaling** | Multiple service instances; read replicas; Redis cluster; async withdrawal processing |
| **Compliance** | KYC verified; AML screened; PCI-DSS; 7-year retention |
| **Limits** | Max balance $5K; max transaction $2K; max withdrawal $10K/day; max 5 outstanding |

