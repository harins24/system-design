---
title: Expense Management and Reimbursement System
layout: default
---

# Expense Management and Reimbursement System — Deep Dive Design

> **Scenario:** 10K employees submit expense reports with receipts. Approval workflow (manager → finance). Reimbursement processing. Integration with corporate cards. Policy enforcement (spending limits, eligible categories). Receipt OCR and auto-categorization. Tax compliance and reporting.
>
> **Tech Stack:** Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka

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
10. [Failure Scenarios](#failure-scenarios)
11. [Scaling Strategy](#scaling-strategy)
12. [Monitoring & Alerts](#monitoring--alerts)
13. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Requirements & Constraints

### Functional Requirements
- Employees submit expense reports with line items, receipts, and descriptions
- Multi-level approval workflow: DRAFT → SUBMITTED → MANAGER_REVIEW → FINANCE_REVIEW → APPROVED/REJECTED
- Auto-categorize expenses using OCR on receipt images
- Match expense reports to corporate card transactions (fuzzy matching)
- Enforce spending policies (category limits, vendor restrictions, receipt requirements)
- Process reimbursements via bank transfer (ACH) weekly
- Generate tax reports (W-2 supplements for taxable benefits)
- Support expense amendments and dispute resolution

### Non-Functional Requirements
- 10K employees, ~30 expense reports/day per employee = 300K reports/month
- Peak: 50 reports/minute during month-end
- OCR processing: async, latency <5 min (95th percentile)
- Approval workflow: <2 sec latency for state transitions
- Policy enforcement: real-time validation
- Data retention: 7 years for tax compliance

### Constraints
- All receipts encrypted at rest and in transit
- PII handling compliant (SSN, bank details in finance module only)
- Audit trail for every approval/rejection/amendment
- Idempotent reimbursement processing (prevent duplicate payments)

---

## Capacity Estimation

| Metric | Value |
|--------|-------|
| Active Employees | 10,000 |
| Expense Reports/Month | ~300,000 |
| Avg. Line Items per Report | 5 |
| Peak Reports/Min | 50 |
| Receipt Images/Report | 4–6 (avg. 50 KB each) |
| Receipt Storage/Month | ~60–75 GB |
| OCR Processing Latency (p95) | <5 min |
| Approval Workflow State Transitions/Day | ~100,000 |
| Policy Rule Evaluations/Day | ~1.5M |
| Corporate Card Transaction Matching/Day | ~50,000 |
| Weekly Reimbursement Batches | 1 (10K–15K transactions) |
| PostgreSQL DB Size (YoY) | ~500 GB |
| MongoDB Document Volume | ~3M (receipt metadata, OCR results) |
| Redis Memory (peak) | ~10 GB (approval queues, locks, cache) |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Expense Management System                      │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                         Frontend Layer (React/Vue)                       │
│  - Expense Report Form   - Receipt Upload   - Approval Dashboard        │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       │
┌──────────────────────────────────────┴──────────────────────────────────┐
│                      API Gateway (Spring Cloud Gateway)                  │
│                        - Auth (OAuth2/JWT)                              │
│                        - Rate Limiting                                   │
│                        - Request Validation                              │
└──────────────────────────────────────┬──────────────────────────────────┘
                    ┌───────────────────┼───────────────────┐
                    │                   │                   │
        ┌───────────▼─────────┐ ┌──────▼──────┐ ┌──────────▼───────┐
        │  Expense Service    │ │ OCR Service │ │ Approval Service │
        │  - CRUD            │ │ - Async     │ │ - Workflow       │
        │  - Validation      │ │ - Tesseract │ │ - State Machine  │
        │  - Policy Check    │ │ - AWS Text. │ │ - Notifications  │
        └─────────┬──────────┘ └──────┬──────┘ └────────┬─────────┘
                  │                   │                 │
        ┌─────────▼──────────────────▼────────────────▼─────────┐
        │               Kafka (Event Bus)                       │
        │  - expense.submitted                                  │
        │  - receipt.uploaded                                   │
        │  - expense.approved                                   │
        │  - expense.rejected                                   │
        └──────────────────────┬────────────────────────────────┘
                  ┌────────────┼────────────────┬────────────┐
        ┌─────────▼──────┐ ┌──▼──┐ ┌──────────▼──┐ ┌────────▼──┐
        │ Card Matching  │ │ Tax │ │ Reimburse   │ │ Reporting │
        │ Service        │ │ Report  │ Service     │ │ Service   │
        │                │ │ Gen │ │             │ │           │
        └────────────────┘ └─────┘ └─────────────┘ └───────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                          Data Layer                                      │
│  ┌─────────────────────┐  ┌──────────┐  ┌────────┐  ┌──────────────┐  │
│  │   PostgreSQL        │  │ MongoDB  │  │ Redis  │  │ S3 (Receipts)│  │
│  │  - Expenses         │  │ - OCR    │  │ - Locks│  │ - Encrypted  │  │
│  │  - Approvals        │  │ - Metadata│ │ - Cache│  │ - Versioned  │  │
│  │  - Policy Rules     │  │          │  │        │  │              │  │
│  │  - Card Txns        │  │          │  │        │  │              │  │
│  └─────────────────────┘  └──────────┘  └────────┘  └──────────────┘  │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Core Design Questions Answered

### 1. How do you implement multi-level approval workflows?

**Answer:** State machine with Kafka-driven transitions.

- Store approval state in PostgreSQL: `DRAFT` → `SUBMITTED` → `MANAGER_REVIEW` → `FINANCE_REVIEW` → `APPROVED`/`REJECTED`
- Each transition emits a Kafka event (e.g., `expense.manager_approved`)
- Listeners consume events and trigger downstream actions (notifications, reimbursement batch)
- Redis caching for current approval queues per manager
- Temporal/Airflow for complex conditional logic (escalations, deadline reminders)

### 2. How do you match corporate card transactions to expense reports?

**Answer:** Fuzzy matching using Elasticsearch and business rules.

- Index corporate card transactions: (amount, merchant_name, transaction_date, employee_id)
- When expense submitted: search for matches on:
  - Amount within ±$1
  - Merchant name similarity (Levenshtein distance <2)
  - Date within ±3 days
  - Same employee
- Return top 3 candidates; employee manually selects or auto-match if confidence >95%
- Store match in PostgreSQL; prevent duplicate reimbursement

### 3. How do you enforce spending policies automatically?

**Answer:** PolicyEnforcementEngine service with rule engine (Drools or custom).

- Rules stored in PostgreSQL/MongoDB:
  - "Travel: max $500/day"
  - "Meals: max $75 per meal"
  - "Office supplies: no Apple products"
  - "Receipt required for expenses >$25"
- On expense submission: evaluate each line item against rules
- Reject line items failing validation; return user-friendly errors
- Cache rules in Redis (TTL 1 day); invalidate on policy change

### 4. How do you process reimbursements at scale?

**Answer:** Weekly batch job with idempotency and reconciliation.

- Scheduled job: run every Friday 2 AM
- Query all `APPROVED` expenses not yet reimbursed
- Group by employee and payment method
- Generate batch file (ACH format): employee bank account, amount, reference ID
- Send to bank/payroll system via secure API (mTLS)
- Monitor for failures; retry failed transactions
- Mark as `REIMBURSED` + store batch ID in PostgreSQL
- Idempotency: each batch has unique ID; bank rejects duplicate transmissions

### 5. How do you extract data from receipts using OCR?

**Answer:** Async pipeline with Tesseract or AWS Textract.

- On receipt upload: emit Kafka event `receipt.uploaded` (receipt_id, S3 path, expense_id)
- OCR Service consumer:
  - Download image from S3
  - Call Tesseract (on-prem) or AWS Textract (cloud)
  - Extract: merchant, amount, date, line items
  - Store raw OCR text in MongoDB
  - Return structured JSON: `{merchant, amount, date, confidence}`
- Auto-fill expense form if confidence >80%
- Store OCR metadata linked to receipt_id
- Handle failures: retry with exponential backoff; fallback to manual entry

### 6. How do you generate tax reports for accounting?

**Answer:** Annual aggregation by employee and expense category.

- Monthly job (or query-on-demand): aggregate approved expenses by employee
- Classify as taxable vs non-taxable per IRS rules (meals/entertainment, home office, etc.)
- Generate W-2 supplement form for taxable amounts
- Export to CSV for payroll/accounting integration
- Store in MongoDB for audit and historical access

---

## Microservices Breakdown

| Service | Responsibility | Key Classes | Dependencies |
|---------|-----------------|------------|--------------|
| **Expense Service** | CRUD expenses, validation, policy check | `ExpenseController`, `ExpenseService`, `ExpenseRepository`, `PolicyEnforcementEngine` | PostgreSQL, Redis, Kafka |
| **Approval Service** | Workflow state machine, approvals, notifications | `ApprovalWorkflowService`, `ApprovalEventListener`, `NotificationService` | PostgreSQL, Redis, Kafka |
| **OCR Service** | Receipt image processing, data extraction | `OcrProcessor`, `TesseractClient`, `TextractAdapter`, `OcrResultRepository` | MongoDB, S3, Kafka |
| **Card Matching** | Corporate card transaction matching | `CardMatchingService`, `FuzzyMatcher`, `ElasticsearchClient` | PostgreSQL, Elasticsearch, Kafka |
| **Reimbursement** | Batch processing, bank transfers, reconciliation | `ReimbursementBatchService`, `AchFileGenerator`, `BankApiClient`, `ReimbursementReconciliation` | PostgreSQL, Kafka |
| **Tax Reporting** | Annual tax calculations, W-2 supplements | `TaxReportGenerator`, `ExpenseAggregator`, `TaxClassifier` | PostgreSQL, MongoDB |

---

## Database Design

### PostgreSQL Schema

```sql
-- Expenses Table
CREATE TABLE expenses (
    id BIGSERIAL PRIMARY KEY,
    employee_id UUID NOT NULL,
    report_title VARCHAR(255) NOT NULL,
    submission_date TIMESTAMP NOT NULL,
    approval_status VARCHAR(50) NOT NULL DEFAULT 'DRAFT',
    total_amount DECIMAL(10, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_employee_status (employee_id, approval_status),
    INDEX idx_submission_date (submission_date)
);

-- Expense Line Items
CREATE TABLE expense_line_items (
    id BIGSERIAL PRIMARY KEY,
    expense_id BIGINT NOT NULL REFERENCES expenses(id),
    category VARCHAR(50) NOT NULL,
    merchant VARCHAR(255) NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    expense_date DATE NOT NULL,
    description TEXT,
    receipt_id UUID,
    policy_violation_flag BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_expense_id (expense_id)
);

-- Approvals Audit Trail
CREATE TABLE approvals (
    id BIGSERIAL PRIMARY KEY,
    expense_id BIGINT NOT NULL REFERENCES expenses(id),
    approver_id UUID NOT NULL,
    approval_level VARCHAR(50) NOT NULL,
    action VARCHAR(20) NOT NULL,
    comments TEXT,
    approved_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_expense_id (expense_id)
);

-- Receipts Metadata
CREATE TABLE receipts (
    id UUID PRIMARY KEY,
    expense_id BIGINT NOT NULL REFERENCES expenses(id),
    s3_path VARCHAR(512) NOT NULL,
    file_size BIGINT,
    upload_timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    ocr_status VARCHAR(50) DEFAULT 'PENDING',
    INDEX idx_expense_id (expense_id)
);

-- Corporate Card Transactions
CREATE TABLE corporate_card_transactions (
    id BIGSERIAL PRIMARY KEY,
    employee_id UUID NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    merchant_name VARCHAR(255) NOT NULL,
    transaction_date TIMESTAMP NOT NULL,
    matched_expense_id BIGINT REFERENCES expenses(id),
    matched_at TIMESTAMP,
    INDEX idx_employee_date (employee_id, transaction_date),
    INDEX idx_unmatched (matched_expense_id)
);

-- Spending Policies
CREATE TABLE spending_policies (
    id BIGSERIAL PRIMARY KEY,
    policy_name VARCHAR(255) NOT NULL,
    category VARCHAR(50) NOT NULL,
    daily_limit DECIMAL(10, 2),
    monthly_limit DECIMAL(10, 2),
    require_receipt_above DECIMAL(10, 2),
    active BOOLEAN DEFAULT TRUE,
    effective_date DATE NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_category_active (category, active)
);

-- Reimbursements
CREATE TABLE reimbursements (
    id BIGSERIAL PRIMARY KEY,
    expense_id BIGINT NOT NULL REFERENCES expenses(id),
    batch_id UUID NOT NULL,
    employee_id UUID NOT NULL,
    amount DECIMAL(10, 2) NOT NULL,
    status VARCHAR(50) DEFAULT 'PENDING',
    transferred_at TIMESTAMP,
    transaction_reference VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_batch_id (batch_id),
    INDEX idx_employee_id (employee_id)
);
```

### MongoDB Collections

```javascript
// OCR Results Collection
db.createCollection("ocr_results", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["receipt_id", "merchant", "amount", "extraction_date"],
      properties: {
        _id: { bsonType: "objectId" },
        receipt_id: { bsonType: "string" },
        expense_id: { bsonType: "long" },
        merchant: { bsonType: "string" },
        amount: { bsonType: "decimal" },
        transaction_date: { bsonType: "date" },
        confidence: { bsonType: "double" },
        raw_text: { bsonType: "string" },
        line_items: { bsonType: "array" },
        extraction_date: { bsonType: "date" },
        processing_time_ms: { bsonType: "int" }
      }
    }
  }
});

// Tax Reports Collection
db.createCollection("tax_reports", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        _id: { bsonType: "objectId" },
        employee_id: { bsonType: "string" },
        report_year: { bsonType: "int" },
        taxable_amount: { bsonType: "decimal" },
        non_taxable_amount: { bsonType: "decimal" },
        breakdown: { bsonType: "object" },
        generated_at: { bsonType: "date" }
      }
    }
  }
});
```

---

## Redis Data Structures

```yaml
# Approval Queue (per manager)
manager:approval:queue:{manager_id}
  Type: ZSET
  Members: expense_id (score = submission_timestamp)
  TTL: None
  Usage: Track pending approvals sorted by submission date

# Distributed Lock (concurrent expense updates)
expense:lock:{expense_id}
  Type: STRING
  Value: "lock_token_{uuid}"
  TTL: 30 seconds
  Usage: Prevent concurrent modifications to same expense

# Policy Rules Cache
policy:rules:{category}
  Type: HASH
  Fields: {daily_limit, monthly_limit, require_receipt}
  TTL: 86400 seconds (1 day)
  Usage: Fast policy validation without DB lookup

# Receipt OCR Status
receipt:ocr:status:{receipt_id}
  Type: STRING
  Value: "PENDING" | "PROCESSING" | "COMPLETED" | "FAILED"
  TTL: 604800 seconds (7 days)
  Usage: Track OCR processing state

# Card Transaction Match Cache
card:match:{employee_id}:{month}
  Type: ZSET
  Members: txn_id (score = match_confidence)
  TTL: 86400 seconds
  Usage: Cache recent card transaction matches

# Session/User Cache
user:approval:context:{user_id}
  Type: HASH
  Fields: {role, approval_level, assigned_department}
  TTL: 3600 seconds (1 hour)
  Usage: Fast user context lookup during approvals
```

---

## Kafka Event Flow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     Kafka Topics & Event Flow                            │
└──────────────────────────────────────────────────────────────────────────┘

Topic: expense.submitted
  Schema: { expense_id, employee_id, total_amount, submission_date }
  Producers: ExpenseService.submit()
  Consumers: ApprovalService, PolicyEnforcementEngine, NotificationService
  Partitions: 10 (by expense_id)
  Retention: 30 days

Topic: receipt.uploaded
  Schema: { receipt_id, expense_id, s3_path, file_size, upload_timestamp }
  Producers: ReceiptUploadController
  Consumers: OcrProcessor (async consumer group)
  Partitions: 5
  Retention: 7 days

Topic: receipt.ocr_completed
  Schema: { receipt_id, merchant, amount, date, confidence, line_items }
  Producers: OcrProcessor
  Consumers: ExpenseService (auto-fill), ApprovalService (if not yet submitted)
  Partitions: 5
  Retention: 30 days

Topic: expense.approval_requested
  Schema: { expense_id, approval_level, approver_id, action_required_by }
  Producers: ApprovalService
  Consumers: NotificationService, ApprovalDashboard
  Partitions: 10
  Retention: 90 days (audit trail)

Topic: expense.approved
  Schema: { expense_id, approval_level, approver_id, approved_at }
  Producers: ApprovalService
  Consumers: ReimbursementService, TaxReportService, NotificationService
  Partitions: 10
  Retention: 7 years (compliance)

Topic: expense.rejected
  Schema: { expense_id, approver_id, rejection_reason, rejected_at }
  Producers: ApprovalService
  Consumers: NotificationService, ExpenseService (revert to DRAFT)
  Partitions: 10
  Retention: 7 years (compliance)

Topic: card.transaction_matched
  Schema: { expense_id, card_txn_id, confidence_score, matched_at }
  Producers: CardMatchingService
  Consumers: ExpenseService, ReimbursementService
  Partitions: 5
  Retention: 30 days

Topic: reimbursement.batch_created
  Schema: { batch_id, expense_ids, total_amount, file_reference, created_at }
  Producers: ReimbursementBatchService
  Consumers: BankApiClient, ReimbursementReconciliation, AuditLogger
  Partitions: 1 (ordered processing)
  Retention: 7 years (compliance)

Topic: reimbursement.completed
  Schema: { batch_id, transaction_references, status, completed_at }
  Producers: BankApiClient
  Consumers: ExpenseService, NotificationService, TaxReportService
  Partitions: 1
  Retention: 7 years (compliance)

Topic: policy.violation_detected
  Schema: { expense_id, line_item_id, policy_name, violation_reason }
  Producers: PolicyEnforcementEngine
  Consumers: ApprovalService (escalate), NotificationService (alert)
  Partitions: 5
  Retention: 7 years (compliance audit)
```

---

## Implementation Code

### 1. ExpenseApprovalWorkflow Service

```java
package com.expense.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.expense.domain.Expense;
import com.expense.domain.ApprovalAction;
import com.expense.domain.ApprovalLevel;
import com.expense.repository.ExpenseRepository;
import com.expense.repository.ApprovalRepository;
import com.expense.event.ExpenseApprovedEvent;
import com.expense.event.ExpenseRejectedEvent;
import java.time.LocalDateTime;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class ExpenseApprovalWorkflow {

    private final ExpenseRepository expenseRepository;
    private final ApprovalRepository approvalRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final NotificationService notificationService;
    private final RedisLockService redisLockService;

    private static final String LOCK_PREFIX = "expense:lock:";
    private static final long LOCK_TIMEOUT_SECONDS = 30;

    /**
     * Transition expense approval state through workflow.
     * Prevents concurrent modifications via distributed lock.
     */
    @Transactional
    public void approveExpense(Long expenseId, UUID approverId, ApprovalLevel level, String comments) {
        String lockKey = LOCK_PREFIX + expenseId;
        String lockToken = UUID.randomUUID().toString();

        if (!redisLockService.acquireLock(lockKey, lockToken, LOCK_TIMEOUT_SECONDS)) {
            throw new ConcurrentApprovalException("Another approval is in progress");
        }

        try {
            Expense expense = expenseRepository.findById(expenseId)
                .orElseThrow(() -> new ExpenseNotFoundException(expenseId));

            // Validate workflow transition
            validateApprovalTransition(expense, level);

            // Record approval in audit trail
            var approval = Approval.builder()
                .expenseId(expenseId)
                .approverId(approverId)
                .approvalLevel(level)
                .action(ApprovalAction.APPROVED)
                .comments(comments)
                .approvedAt(LocalDateTime.now())
                .build();
            approvalRepository.save(approval);

            // Transition state
            ApprovalLevel nextLevel = getNextApprovalLevel(level);
            if (nextLevel == null) {
                // Final approval
                expense.setApprovalStatus("APPROVED");
                kafkaTemplate.send("expense.approved", expenseId.toString(),
                    new ExpenseApprovedEvent(expenseId, approverId, LocalDateTime.now()));
                log.info("Expense {} fully approved by {}", expenseId, approverId);
            } else {
                // Intermediate approval
                expense.setApprovalStatus(level.name() + "_APPROVED");
                kafkaTemplate.send("expense.approval_requested", expenseId.toString(),
                    new ApprovalRequestedEvent(expenseId, nextLevel, LocalDateTime.now()));
                log.info("Expense {} approved at level {}, next: {}", expenseId, level, nextLevel);
            }

            expense.setUpdatedAt(LocalDateTime.now());
            expenseRepository.save(expense);

            // Async notification
            notificationService.notifyApprovalComplete(expenseId, level, true);

        } finally {
            redisLockService.releaseLock(lockKey, lockToken);
        }
    }

    @Transactional
    public void rejectExpense(Long expenseId, UUID approverId, ApprovalLevel level, String rejectionReason) {
        String lockKey = LOCK_PREFIX + expenseId;
        String lockToken = UUID.randomUUID().toString();

        if (!redisLockService.acquireLock(lockKey, lockToken, LOCK_TIMEOUT_SECONDS)) {
            throw new ConcurrentApprovalException("Another approval is in progress");
        }

        try {
            Expense expense = expenseRepository.findById(expenseId)
                .orElseThrow(() -> new ExpenseNotFoundException(expenseId));

            var approval = Approval.builder()
                .expenseId(expenseId)
                .approverId(approverId)
                .approvalLevel(level)
                .action(ApprovalAction.REJECTED)
                .comments(rejectionReason)
                .approvedAt(LocalDateTime.now())
                .build();
            approvalRepository.save(approval);

            expense.setApprovalStatus("REJECTED");
            expense.setUpdatedAt(LocalDateTime.now());
            expenseRepository.save(expense);

            kafkaTemplate.send("expense.rejected", expenseId.toString(),
                new ExpenseRejectedEvent(expenseId, rejectionReason, LocalDateTime.now()));

            notificationService.notifyApprovalComplete(expenseId, level, false);
            log.info("Expense {} rejected at level {} by {}", expenseId, level, approverId);

        } finally {
            redisLockService.releaseLock(lockKey, lockToken);
        }
    }

    private void validateApprovalTransition(Expense expense, ApprovalLevel level) {
        String currentStatus = expense.getApprovalStatus();
        // Implement state machine validation (DRAFT -> MANAGER_REVIEW -> FINANCE_REVIEW, etc.)
        if (!isValidTransition(currentStatus, level)) {
            throw new InvalidApprovalTransitionException(currentStatus, level);
        }
    }

    private ApprovalLevel getNextApprovalLevel(ApprovalLevel current) {
        return switch (current) {
            case MANAGER_REVIEW -> ApprovalLevel.FINANCE_REVIEW;
            case FINANCE_REVIEW -> null; // Final approval
            default -> null;
        };
    }

    private boolean isValidTransition(String currentStatus, ApprovalLevel nextLevel) {
        // Business logic for valid state transitions
        return true; // Simplified
    }
}
```

### 2. PolicyEnforcementEngine

```java
package com.expense.policy;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import com.expense.domain.Expense;
import com.expense.domain.ExpenseLineItem;
import com.expense.repository.SpendingPolicyRepository;
import java.util.ArrayList;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class PolicyEnforcementEngine {

    private final SpendingPolicyRepository policyRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    private static final String POLICY_CACHE_PREFIX = "policy:rules:";
    private static final long CACHE_TTL_SECONDS = 86400; // 1 day

    /**
     * Validate expense line items against spending policies.
     * Returns list of violations found.
     */
    public List<PolicyViolation> validateExpense(Expense expense) {
        List<PolicyViolation> violations = new ArrayList<>();

        for (ExpenseLineItem item : expense.getLineItems()) {
            SpendingPolicy policy = getPolicy(item.getCategory());
            if (policy == null) continue;

            // Check receipt requirement
            if (policy.getRequireReceiptAbove() != null &&
                item.getAmount().compareTo(policy.getRequireReceiptAbove()) > 0 &&
                item.getReceiptId() == null) {
                violations.add(PolicyViolation.builder()
                    .expenseId(expense.getId())
                    .lineItemId(item.getId())
                    .policyName("RECEIPT_REQUIRED")
                    .violationReason("Receipt required for expenses > " + policy.getRequireReceiptAbove())
                    .build());
                item.setPolicyViolationFlag(true);
            }

            // Check daily limit (aggregate by category + date)
            if (policy.getDailyLimit() != null) {
                var dailyTotal = calculateDailyTotal(expense.getEmployeeId(),
                    item.getCategory(), item.getExpenseDate());
                if (dailyTotal.add(item.getAmount()).compareTo(policy.getDailyLimit()) > 0) {
                    violations.add(PolicyViolation.builder()
                        .expenseId(expense.getId())
                        .lineItemId(item.getId())
                        .policyName("DAILY_LIMIT_EXCEEDED")
                        .violationReason(String.format("Daily limit for %s is %s, total would be %s",
                            item.getCategory(), policy.getDailyLimit(), dailyTotal.add(item.getAmount())))
                        .build());
                    item.setPolicyViolationFlag(true);
                }
            }

            // Check monthly limit
            if (policy.getMonthlyLimit() != null) {
                var monthlyTotal = calculateMonthlyTotal(expense.getEmployeeId(),
                    item.getCategory(), item.getExpenseDate());
                if (monthlyTotal.add(item.getAmount()).compareTo(policy.getMonthlyLimit()) > 0) {
                    violations.add(PolicyViolation.builder()
                        .expenseId(expense.getId())
                        .lineItemId(item.getId())
                        .policyName("MONTHLY_LIMIT_EXCEEDED")
                        .violationReason(String.format("Monthly limit for %s is %s, total would be %s",
                            item.getCategory(), policy.getMonthlyLimit(), monthlyTotal.add(item.getAmount())))
                        .build());
                    item.setPolicyViolationFlag(true);
                }
            }
        }

        // Emit policy violation event if any violations found
        if (!violations.isEmpty()) {
            violations.forEach(v -> kafkaTemplate.send("policy.violation_detected",
                expense.getId().toString(), v));
            log.warn("Found {} policy violations in expense {}", violations.size(), expense.getId());
        }

        return violations;
    }

    /**
     * Get spending policy from cache or database.
     */
    private SpendingPolicy getPolicy(String category) {
        String cacheKey = POLICY_CACHE_PREFIX + category;
        // Try cache first
        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return deserializePolicy(cached);
        }

        // Fetch from DB and cache
        var policy = policyRepository.findByCategoryAndActive(category, true).orElse(null);
        if (policy != null) {
            redisTemplate.opsForValue().set(cacheKey, serializePolicy(policy),
                java.time.Duration.ofSeconds(CACHE_TTL_SECONDS));
        }
        return policy;
    }

    private java.math.BigDecimal calculateDailyTotal(java.util.UUID employeeId,
            String category, java.time.LocalDate date) {
        // Query DB: sum of expenses for employee, category, date (status = APPROVED)
        return java.math.BigDecimal.ZERO; // Simplified
    }

    private java.math.BigDecimal calculateMonthlyTotal(java.util.UUID employeeId,
            String category, java.time.LocalDate date) {
        // Query DB: sum of expenses for employee, category, month (status = APPROVED)
        return java.math.BigDecimal.ZERO; // Simplified
    }

    private String serializePolicy(SpendingPolicy policy) {
        // JSON serialization
        return "";
    }

    private SpendingPolicy deserializePolicy(String json) {
        // JSON deserialization
        return null;
    }
}

@lombok.Data
class PolicyViolation {
    private Long expenseId;
    private Long lineItemId;
    private String policyName;
    private String violationReason;

    @lombok.Builder
    public PolicyViolation(Long expenseId, Long lineItemId, String policyName, String violationReason) {
        this.expenseId = expenseId;
        this.lineItemId = lineItemId;
        this.policyName = policyName;
        this.violationReason = violationReason;
    }
}
```

### 3. ReceiptOcrProcessor

```java
package com.expense.ocr;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.textract.TextractClient;
import software.amazon.awssdk.services.textract.model.DocumentLocation;
import software.amazon.awssdk.services.textract.model.AnalyzeDocumentRequest;
import software.amazon.awssdk.services.textract.model.AnalyzeDocumentResponse;
import com.expense.event.ReceiptUploadedEvent;
import com.expense.event.ReceiptOcrCompletedEvent;
import com.expense.repository.OcrResultRepository;
import java.io.IOException;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class ReceiptOcrProcessor {

    private final TextractClient textractClient;
    private final OcrResultRepository ocrResultRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final S3Client s3Client;
    private final RedisTemplate<String, String> redisTemplate;

    /**
     * Process uploaded receipts asynchronously via Kafka.
     */
    @KafkaListener(topics = "receipt.uploaded", groupId = "ocr-processor-group")
    public void processReceipt(ReceiptUploadedEvent event) {
        String receiptId = event.getReceiptId();
        String statusKey = "receipt:ocr:status:" + receiptId;

        try {
            // Mark as processing
            redisTemplate.opsForValue().set(statusKey, "PROCESSING",
                java.time.Duration.ofDays(7));

            log.info("Starting OCR processing for receipt {}", receiptId);
            long startTime = System.currentTimeMillis();

            // Download image from S3
            byte[] imageData = s3Client.downloadFile(event.getS3Path());

            // Call AWS Textract
            OcrResult ocrResult = extractTextFromReceipt(imageData, receiptId);

            // Save to MongoDB
            ocrResultRepository.save(OcrResult.builder()
                .receiptId(receiptId)
                .expenseId(event.getExpenseId())
                .merchant(ocrResult.getMerchant())
                .amount(ocrResult.getAmount())
                .transactionDate(ocrResult.getTransactionDate())
                .confidence(ocrResult.getConfidence())
                .rawText(ocrResult.getRawText())
                .processingTimeMs(System.currentTimeMillis() - startTime)
                .extractionDate(LocalDateTime.now())
                .build());

            // Emit completion event
            kafkaTemplate.send("receipt.ocr_completed", receiptId,
                new ReceiptOcrCompletedEvent(receiptId, event.getExpenseId(),
                    ocrResult.getMerchant(), ocrResult.getAmount(),
                    ocrResult.getTransactionDate(), ocrResult.getConfidence()));

            redisTemplate.opsForValue().set(statusKey, "COMPLETED",
                java.time.Duration.ofDays(7));
            log.info("OCR processing completed for receipt {} in {} ms", receiptId,
                System.currentTimeMillis() - startTime);

        } catch (Exception e) {
            log.error("OCR processing failed for receipt {}", receiptId, e);
            redisTemplate.opsForValue().set(statusKey, "FAILED",
                java.time.Duration.ofDays(7));
            // Retry logic via Kafka dead-letter topic
            kafkaTemplate.send("receipt.ocr_failed", receiptId,
                new ReceiptOcrFailedEvent(receiptId, e.getMessage()));
        }
    }

    /**
     * Extract text and structured data from receipt using AWS Textract.
     */
    private OcrResult extractTextFromReceipt(byte[] imageData, String receiptId) {
        try {
            // Call Textract API
            AnalyzeDocumentRequest request = AnalyzeDocumentRequest.builder()
                .document(software.amazon.awssdk.services.textract.model.Document.builder()
                    .bytes(software.amazon.awssdk.core.SdkBytes.fromByteArray(imageData))
                    .build())
                .featureTypes(software.amazon.awssdk.services.textract.model.FeatureType.FORMS,
                    software.amazon.awssdk.services.textract.model.FeatureType.TABLES)
                .build();

            AnalyzeDocumentResponse response = textractClient.analyzeDocument(request);

            // Parse response blocks
            Map<String, String> extractedData = parseTextractResponse(response);

            // Extract merchant, amount, date with confidence scoring
            String merchant = extractedData.getOrDefault("merchant", "Unknown");
            BigDecimal amount = extractAmount(extractedData.get("amount"));
            LocalDate date = extractDate(extractedData.get("date"));
            double confidence = calculateConfidence(extractedData);

            return OcrResult.builder()
                .merchant(merchant)
                .amount(amount)
                .transactionDate(date)
                .confidence(confidence)
                .rawText(extractedData.toString())
                .build();

        } catch (Exception e) {
            throw new OcrProcessingException("Failed to process receipt with Textract", e);
        }
    }

    private Map<String, String> parseTextractResponse(AnalyzeDocumentResponse response) {
        Map<String, String> data = new HashMap<>();
        // Iterate through blocks, extract key-value pairs
        // For brevity, simplified here
        return data;
    }

    private BigDecimal extractAmount(String amountStr) {
        if (amountStr == null) return null;
        // Remove currency symbols, parse to BigDecimal
        String cleaned = amountStr.replaceAll("[^0-9.]", "");
        try {
            return new BigDecimal(cleaned);
        } catch (NumberFormatException e) {
            return null;
        }
    }

    private LocalDate extractDate(String dateStr) {
        if (dateStr == null) return null;
        try {
            // Try multiple date formats
            return LocalDate.parse(dateStr);
        } catch (Exception e) {
            return null;
        }
    }

    private double calculateConfidence(Map<String, String> data) {
        // Simple confidence: percentage of fields successfully extracted
        int fieldsExpected = 3; // merchant, amount, date
        int fieldsFound = 0;
        if (data.containsKey("merchant")) fieldsFound++;
        if (data.containsKey("amount")) fieldsFound++;
        if (data.containsKey("date")) fieldsFound++;
        return (double) fieldsFound / fieldsExpected;
    }
}
```

### 4. CorporateCardMatchingService

```java
package com.expense.matching;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.elasticsearch.core.ElasticsearchOperations;
import org.springframework.data.elasticsearch.core.query.Query;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import com.expense.domain.Expense;
import com.expense.domain.CorporateCardTransaction;
import com.expense.repository.CardTransactionRepository;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.List;
import java.util.stream.Collectors;

@Slf4j
@Service
@RequiredArgsConstructor
public class CorporateCardMatchingService {

    private final ElasticsearchOperations elasticsearchOps;
    private final CardTransactionRepository cardTxnRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    /**
     * Find matching corporate card transactions for an expense.
     * Uses fuzzy matching: amount (±$1), merchant name similarity, date (±3 days).
     */
    public List<CardMatchCandidate> findMatches(Expense expense) {
        List<CardMatchCandidate> candidates = new ArrayList<>();

        for (ExpenseLineItem item : expense.getLineItems()) {
            List<CardMatchCandidate> itemMatches =
                searchCardTransactions(expense.getEmployeeId(), item);
            candidates.addAll(itemMatches);
        }

        return candidates.stream()
            .sorted((a, b) -> Double.compare(b.getConfidenceScore(), a.getConfidenceScore()))
            .limit(5)
            .collect(Collectors.toList());
    }

    private List<CardMatchCandidate> searchCardTransactions(
            java.util.UUID employeeId, ExpenseLineItem item) {

        BigDecimal minAmount = item.getAmount().subtract(new BigDecimal("1"));
        BigDecimal maxAmount = item.getAmount().add(new BigDecimal("1"));
        LocalDate minDate = item.getExpenseDate().minusDays(3);
        LocalDate maxDate = item.getExpenseDate().plusDays(3);

        // Elasticsearch query: range on amount and date, fuzzy match on merchant
        List<CorporateCardTransaction> transactions = cardTxnRepository
            .findByEmployeeIdAndAmountBetweenAndTransactionDateBetween(
                employeeId, minAmount, maxAmount, minDate.atStartOfDay(), maxDate.plusDays(1).atStartOfDay());

        return transactions.stream()
            .map(txn -> {
                double merchantSimilarity = calculateMerchantSimilarity(
                    item.getMerchant(), txn.getMerchantName());
                double amountSimilarity = calculateAmountSimilarity(
                    item.getAmount(), txn.getAmount());
                double dateSimilarity = calculateDateSimilarity(
                    item.getExpenseDate(), txn.getTransactionDate().toLocalDate());

                double confidenceScore =
                    (merchantSimilarity * 0.5) + (amountSimilarity * 0.3) + (dateSimilarity * 0.2);

                return CardMatchCandidate.builder()
                    .transactionId(txn.getId())
                    .merchant(txn.getMerchantName())
                    .amount(txn.getAmount())
                    .transactionDate(txn.getTransactionDate().toLocalDate())
                    .confidenceScore(confidenceScore)
                    .merchantSimilarity(merchantSimilarity)
                    .build();
            })
            .filter(c -> c.getConfidenceScore() > 0.5) // Minimum threshold
            .collect(Collectors.toList());
    }

    /**
     * Match expense to card transaction. Auto-match if confidence > 0.95.
     */
    public void matchExpenseToCard(Long expenseId, Long cardTxnId,
            double confidenceScore) {

        CorporateCardTransaction txn = cardTxnRepository.findById(cardTxnId)
            .orElseThrow(() -> new TransactionNotFoundException(cardTxnId));

        txn.setMatchedExpenseId(expenseId);
        txn.setMatchedAt(java.time.LocalDateTime.now());
        cardTxnRepository.save(txn);

        kafkaTemplate.send("card.transaction_matched", expenseId.toString(),
            new CardTransactionMatchedEvent(expenseId, cardTxnId, confidenceScore));

        log.info("Matched expense {} to card transaction {} with confidence {}",
            expenseId, cardTxnId, confidenceScore);
    }

    /**
     * Levenshtein distance-based merchant name similarity (0.0 - 1.0).
     */
    private double calculateMerchantSimilarity(String name1, String name2) {
        String n1 = name1.toLowerCase().replaceAll("[^a-z0-9]", "");
        String n2 = name2.toLowerCase().replaceAll("[^a-z0-9]", "");

        int distance = levenshteinDistance(n1, n2);
        int maxLen = Math.max(n1.length(), n2.length());
        return maxLen == 0 ? 1.0 : 1.0 - ((double) distance / maxLen);
    }

    private double calculateAmountSimilarity(BigDecimal exp, BigDecimal txn) {
        BigDecimal diff = exp.subtract(txn).abs();
        // Score: 1.0 if exact, 0.5 if ±$1, 0.0 if >$1 difference
        if (diff.compareTo(BigDecimal.ZERO) == 0) return 1.0;
        if (diff.compareTo(new BigDecimal("1")) <= 0) return 0.5;
        return 0.0;
    }

    private double calculateDateSimilarity(LocalDate exp, LocalDate txn) {
        long daysDiff = Math.abs(java.time.temporal.ChronoUnit.DAYS.between(exp, txn));
        // Score: 1.0 if same day, 0.7 if ±1 day, 0.3 if ±3 days, 0.0 otherwise
        if (daysDiff == 0) return 1.0;
        if (daysDiff <= 1) return 0.7;
        if (daysDiff <= 3) return 0.3;
        return 0.0;
    }

    private int levenshteinDistance(String a, String b) {
        int[][] dp = new int[a.length() + 1][b.length() + 1];
        for (int i = 0; i <= a.length(); i++) dp[i][0] = i;
        for (int j = 0; j <= b.length(); j++) dp[0][j] = j;

        for (int i = 1; i <= a.length(); i++) {
            for (int j = 1; j <= b.length(); j++) {
                int cost = a.charAt(i - 1) == b.charAt(j - 1) ? 0 : 1;
                dp[i][j] = Math.min(Math.min(dp[i-1][j] + 1, dp[i][j-1] + 1),
                    dp[i-1][j-1] + cost);
            }
        }
        return dp[a.length()][b.length()];
    }
}
```

### 5. ReimbursementBatchService

```java
package com.expense.reimbursement;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.batch.core.Job;
import org.springframework.batch.core.JobParameters;
import org.springframework.batch.core.JobParametersBuilder;
import org.springframework.batch.core.launch.JobLauncher;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.expense.domain.Expense;
import com.expense.domain.Reimbursement;
import com.expense.repository.ExpenseRepository;
import com.expense.repository.ReimbursementRepository;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class ReimbursementBatchService {

    private final ExpenseRepository expenseRepository;
    private final ReimbursementRepository reimbursementRepository;
    private final AchFileGenerator achFileGenerator;
    private final BankApiClient bankApiClient;
    private final JobLauncher jobLauncher;
    private final Job reimbursementBatchJob;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    /**
     * Weekly scheduled job: process all approved, non-reimbursed expenses.
     */
    @Scheduled(cron = "0 2 ? * FRI") // Friday 2 AM
    public void createWeeklyReimbursementBatch() {
        log.info("Starting weekly reimbursement batch processing");

        try {
            JobParameters jobParams = new JobParametersBuilder()
                .addLong("timestamp", System.currentTimeMillis())
                .toJobParameters();

            jobLauncher.run(reimbursementBatchJob, jobParams);
            log.info("Weekly reimbursement batch job completed");

        } catch (Exception e) {
            log.error("Failed to launch reimbursement batch job", e);
            // Alert monitoring system
        }
    }

    /**
     * Create reimbursement records for approved expenses.
     * Group by employee bank account for ACH batch transmission.
     */
    @Transactional
    public void processBatch(List<Expense> approvedExpenses) {
        UUID batchId = UUID.randomUUID();
        BigDecimal totalAmount = BigDecimal.ZERO;

        // Group expenses by employee
        var byEmployee = approvedExpenses.stream()
            .collect(java.util.stream.Collectors.groupingBy(Expense::getEmployeeId));

        List<Reimbursement> reimbursements = new ArrayList<>();

        for (var entry : byEmployee.entrySet()) {
            UUID employeeId = entry.getKey();
            List<Expense> expenses = entry.getValue();

            BigDecimal employeeTotal = expenses.stream()
                .map(Expense::getTotalAmount)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

            Reimbursement reimbursement = Reimbursement.builder()
                .batchId(batchId)
                .employeeId(employeeId)
                .amount(employeeTotal)
                .status("PENDING")
                .build();
            reimbursements.add(reimbursement);
            totalAmount = totalAmount.add(employeeTotal);

            // Mark expenses as reimbursed
            expenses.forEach(e -> {
                e.setApprovalStatus("REIMBURSED");
                e.setUpdatedAt(LocalDateTime.now());
            });
        }

        reimbursementRepository.saveAll(reimbursements);
        expenseRepository.saveAll(approvedExpenses);

        // Generate ACH batch file
        String achBatchContent = achFileGenerator.generateBatch(batchId, reimbursements);

        // Transmit to bank via secure API
        try {
            String transactionRef = bankApiClient.submitAchBatch(achBatchContent, batchId);
            log.info("ACH batch {} submitted with {} reimbursements, total ${}",
                batchId, reimbursements.size(), totalAmount);

            // Emit event for downstream processing
            kafkaTemplate.send("reimbursement.batch_created", batchId.toString(),
                new ReimbursementBatchCreatedEvent(batchId, reimbursements.size(),
                    totalAmount, transactionRef, LocalDateTime.now()));

        } catch (Exception e) {
            log.error("Failed to submit ACH batch {}", batchId, e);
            // Retry logic or manual intervention
            reimbursements.forEach(r -> r.setStatus("FAILED"));
            reimbursementRepository.saveAll(reimbursements);
        }
    }
}
```

### 6. TaxReportGenerator

```java
package com.expense.tax;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.mongodb.core.MongoTemplate;
import org.springframework.data.mongodb.core.query.Criteria;
import org.springframework.data.mongodb.core.query.Query;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import com.expense.repository.ExpenseRepository;
import com.expense.domain.Expense;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.Year;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class TaxReportGenerator {

    private final ExpenseRepository expenseRepository;
    private final MongoTemplate mongoTemplate;
    private final TaxClassificationService taxClassificationService;

    /**
     * Generate annual tax reports for all employees (scheduled for Jan 1 of next year).
     */
    @Scheduled(cron = "0 3 1 1 *") // Jan 1, 3 AM
    public void generateAnnualTaxReports() {
        int previousYear = Year.now().getValue() - 1;
        log.info("Starting tax report generation for year {}", previousYear);

        // Get all employees with approved expenses in previous year
        List<UUID> employeeIds = expenseRepository.findAllEmployeesWithApprovedExpenses(previousYear);

        for (UUID employeeId : employeeIds) {
            generateTaxReportForEmployee(employeeId, previousYear);
        }

        log.info("Tax report generation completed for {}", previousYear);
    }

    /**
     * Generate W-2 supplement for a single employee-year.
     * Segregate taxable vs non-taxable expenses per IRS rules.
     */
    @Transactional
    public void generateTaxReportForEmployee(UUID employeeId, int year) {
        List<Expense> expenses = expenseRepository.findApprovedExpensesByEmployeeAndYear(
            employeeId, year);

        BigDecimal taxableAmount = BigDecimal.ZERO;
        BigDecimal nonTaxableAmount = BigDecimal.ZERO;
        Map<String, BigDecimal> breakdownByCategory = new HashMap<>();

        for (Expense expense : expenses) {
            for (ExpenseLineItem item : expense.getLineItems()) {
                TaxClassification classification =
                    taxClassificationService.classify(item.getCategory());

                if (classification.isTaxable()) {
                    taxableAmount = taxableAmount.add(item.getAmount());
                } else {
                    nonTaxableAmount = nonTaxableAmount.add(item.getAmount());
                }

                breakdownByCategory.merge(item.getCategory(), item.getAmount(), BigDecimal::add);
            }
        }

        TaxReport taxReport = TaxReport.builder()
            .employeeId(employeeId)
            .reportYear(year)
            .taxableAmount(taxableAmount)
            .nonTaxableAmount(nonTaxableAmount)
            .breakdown(breakdownByCategory)
            .generatedAt(LocalDateTime.now())
            .build();

        mongoTemplate.insert(taxReport, "tax_reports");
        log.info("Tax report generated for employee {} year {}: taxable ${}, non-taxable ${}",
            employeeId, year, taxableAmount, nonTaxableAmount);
    }

    /**
     * Export tax reports as W-2 supplement CSV for payroll integration.
     */
    public String exportW2Supplement(int year) {
        Query query = new Query(Criteria.where("reportYear").is(year));
        List<TaxReport> reports = mongoTemplate.find(query, TaxReport.class, "tax_reports");

        StringBuilder csv = new StringBuilder();
        csv.append("EmployeeID,TaxableAmount,NonTaxableAmount,ReportYear\n");

        for (TaxReport report : reports) {
            csv.append(String.format("%s,%.2f,%.2f,%d\n",
                report.getEmployeeId(), report.getTaxableAmount(),
                report.getNonTaxableAmount(), year));
        }

        return csv.toString();
    }
}

/**
 * Service to classify expense categories as taxable/non-taxable per IRS rules.
 */
@Service
class TaxClassificationService {

    private static final Map<String, Boolean> TAXABILITY_MAP = Map.ofEntries(
        Map.entry("MEALS_ENTERTAINMENT", true),      // Subject to Section 274
        Map.entry("HOME_OFFICE", false),             // Deductible but not taxable to employee
        Map.entry("TRAVEL", false),                  // Deductible but not taxable
        Map.entry("OFFICE_SUPPLIES", false),         // Non-taxable
        Map.entry("PROFESSIONAL_DEVELOPMENT", false),// Non-taxable
        Map.entry("CELL_PHONE", true),               // Partially taxable
        Map.entry("VEHICLE", true)                   // Taxable (excess mileage)
    );

    public TaxClassification classify(String category) {
        Boolean taxable = TAXABILITY_MAP.getOrDefault(category, false);
        return TaxClassification.builder()
            .category(category)
            .taxable(taxable)
            .build();
    }
}
```

---

## Failure Scenarios

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| Approval service crashes mid-transition | Expense state corrupted, lost events | Distributed lock + transactional write; replay from Kafka |
| Receipt OCR times out (>5 min) | User blocked from submitting expense | Async processing; return immediately, OCR in background; notify user |
| Corporate card match fails (DB unavailable) | Can't auto-match expenses | Fall back to manual employee matching; retry with exponential backoff |
| Policy rule cache stale | Violations missed or false positives | TTL 1 day; event-driven invalidation on rule change; DB as source of truth |
| Reimbursement batch corrupted | Duplicate/missing payments | Idempotent batch ID; reconciliation report; manual audit before ACH transmission |
| Kafka broker down | Events lost, no audit trail | Multiple brokers (replication factor 3); event store backup in PostgreSQL |
| S3 receipt unavailable | OCR blocked | Retry with exponential backoff; cache recently uploaded receipts in local SSD |
| Tax report calculation error | Incorrect W-2 supplements | Regression tests on historical data; manual spot checks; flag anomalies (>2x avg) |

---

## Scaling Strategy

### Horizontal Scaling

**Expense Service:**
- Stateless; deploy multiple instances behind load balancer
- Scale based on QPS (target: 500 QPS per instance)
- Database: read replicas for report queries; primary for writes

**OCR Service:**
- Kafka consumer group with auto-scaling
- Scale based on lag (target: <5 min lag)
- Batch OCR processing in worker pool (Kubernetes job queue)

**Approval Service:**
- Stateless; multiple instances
- Redis for distributed locks ensures serial approval processing

**Card Matching Service:**
- Elasticsearch index sharding by employee_id (10 shards)
- Bulk import of card transactions via Logstash

### Vertical Scaling

**PostgreSQL:**
- Connection pooling: HikariCP (max 100 per instance)
- Index optimization: btree on (employee_id, status), (status, submission_date)
- Partitioning: range partition expenses by year
- Archival: move >3 years to cold storage (S3)

**MongoDB:**
- Sharding key: receipt_id (even distribution)
- Indexes: {receipt_id: 1}, {employee_id: 1, year: 1}

**Redis:**
- Replication: master-slave for cache
- Cluster mode for >64 GB data
- Eviction policy: LRU, max memory 10 GB

### Database Optimization

```sql
-- Expense query optimization
CREATE INDEX idx_expenses_employee_status ON expenses(employee_id, approval_status);
CREATE INDEX idx_expenses_submission_date ON expenses(submission_date DESC);

-- Partition by year
ALTER TABLE expenses PARTITION BY RANGE (YEAR(submission_date)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025),
    PARTITION p_future VALUES LESS THAN MAXVALUE
);

-- Materialized view for reporting
CREATE MATERIALIZED VIEW expense_summary AS
SELECT
    employee_id,
    EXTRACT(YEAR_MONTH FROM submission_date) AS month,
    COUNT(*) AS count,
    SUM(total_amount) AS total
FROM expenses
WHERE approval_status = 'APPROVED'
GROUP BY employee_id, EXTRACT(YEAR_MONTH FROM submission_date);
```

---

## Monitoring & Alerts

### Key Metrics

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| Expense submission latency (p95) | <2 sec | >5 sec |
| Approval state transition latency (p95) | <1 sec | >3 sec |
| OCR processing latency (p95) | <5 min | >10 min |
| Policy validation latency (p99) | <500 ms | >1 sec |
| Reimbursement batch success rate | >99.5% | <99% |
| Kafka lag (approval events) | <100 msgs | >1000 msgs |
| Card match confidence (avg) | >0.7 | <0.5 |
| Redis lock contention (p95 wait) | <100 ms | >500 ms |
| PostgreSQL query latency (p95) | <500 ms | >2 sec |
| Exception rate | <0.1% | >0.5% |

### Alert Rules (Prometheus)

```yaml
groups:
  - name: expense_system
    rules:
      - alert: HighExpenseSubmissionLatency
        expr: histogram_quantile(0.95, expense_submission_latency_seconds) > 5
        for: 5m
        annotations:
          summary: "Expense submission latency high (p95: {{ $value }}s)"

      - alert: OcrProcessingBacklog
        expr: increase(receipt_ocr_pending_total[5m]) > 1000
        for: 10m
        annotations:
          summary: "OCR processing backlog exceeded threshold"

      - alert: ReimbursementBatchFailure
        expr: rate(reimbursement_batch_failures_total[5m]) > 0.01
        for: 5m
        annotations:
          summary: "Reimbursement batch failure rate elevated"

      - alert: PolicyEnforcementErrors
        expr: rate(policy_validation_errors_total[5m]) > 0.005
        for: 5m
        annotations:
          summary: "Policy validation error rate high"
```

---

## Summary Cheat Sheet

### Architecture Components
- **Expense Service**: CRUD, validation, policy checks
- **Approval Workflow**: State machine (DRAFT → SUBMITTED → MANAGER → FINANCE → APPROVED)
- **OCR Processor**: Async receipt → JSON extraction (Tesseract/Textract)
- **Card Matching**: Fuzzy match on (amount ±$1, merchant name, date ±3 days)
- **Reimbursement**: Weekly batch → ACH → bank transfer
- **Tax Reports**: Annual aggregation by employee/category → W-2 supplement

### Key Technologies
- **PostgreSQL**: Expenses, approvals, policies, card transactions (primary OLTP)
- **MongoDB**: OCR results, tax reports (document storage)
- **Redis**: Approval queues, policy cache, distributed locks, receipt OCR status
- **Kafka**: Event-driven workflow (13 topics)
- **Elasticsearch**: Fuzzy card transaction search
- **AWS S3**: Encrypted receipt storage
- **Textract/Tesseract**: OCR processing

### Approval Workflow State Machine
```
DRAFT
  → [Submit]
    SUBMITTED
      → [Mgr Approve]
        MANAGER_REVIEW_APPROVED
          → [Finance Approve]
            APPROVED ─→ [Weekly Batch] → REIMBURSED
          → [Finance Reject]
            REJECTED ─→ [Revise] → DRAFT
      → [Mgr Reject]
        REJECTED ─→ [Revise] → DRAFT
```

### Performance Targets
- 10K employees, 300K reports/month
- Peak: 50 reports/min
- Approval transition: <2 sec
- OCR latency (p95): <5 min
- Reimbursement success rate: >99.5%

### Idempotency & Compliance
- Reimbursement batch ID ensures no duplicates
- 7-year retention for tax compliance
- All approvals audit-logged in PostgreSQL
- Kafka event stream as immutable ledger
