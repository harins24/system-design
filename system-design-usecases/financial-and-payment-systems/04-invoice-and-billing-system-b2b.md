---
title: Invoice and Billing System for B2B — Deep Dive Design
layout: default
---

# Invoice and Billing System for B2B — Deep Dive Design

> **Scenario:** Corporate customers get NET-30 payment terms. Generate invoices monthly for all orders. Support purchase orders (PO). Credit limits per customer. Aging reports (30/60/90 days overdue). Automatic dunning (payment reminders). Early payment discounts (2% if paid within 10 days).
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
- Customers assigned credit limits (e.g., $50K); enforce at order placement
- Monthly invoice generation: aggregate all orders for customer → PDF → email
- Support Purchase Order (PO) numbers: link PO to orders
- Payment terms: NET-30 (30 days from invoice date)
- Dunning: automated payment reminders at 7-days, 14-days, 21-days overdue; escalate to collections
- Partial payments: invoice can receive multiple payments until balance = $0
- Early payment discount: 2% off if paid within 10 days of invoice date
- Aging reports: 0-30 days, 31-60 days, 61-90 days, 90+ days overdue
- Accounting integration: adapt for QuickBooks, SAP, NetSuite (pluggable)

### Non-Functional Requirements
- **Scale:** 5,000 customers, ~50K orders/month, ~10K invoices/month
- **Latency:** Credit check <100ms; invoice generation <5 seconds per customer
- **Availability:** 99.9% uptime
- **Consistency:** Credit limit checks must be atomic; invoice generation must be idempotent
- **Audit:** all payment transactions logged; immutable audit trail

---

## Capacity Estimation

| Metric | Volume | Notes |
|--------|--------|-------|
| Customers | 5,000 | B2B corporate accounts |
| Orders/Month | 50,000 | ~10 orders per customer |
| Invoices/Month | 10,000 | ~2 invoices per customer (monthly + adjustments) |
| Invoice Payments/Month | 20,000 | partial payments, early payments |
| Storage (1 year) | ~100 GB | invoices (PDF), payment records, audit logs |
| Redis Memory | ~500 MB | credit limits + locks (100 bytes per customer) |
| Database Size (1 year) | ~50 GB PostgreSQL | invoices, orders, payments, customers |

---

## High-Level Architecture

```
                     ┌─────────────────────────────────────────┐
                     │  Order Service (Existing)                │
                     │  - Create Order                          │
                     │  - Emit: OrderCreated (Kafka)            │
                     └────────────────────┬────────────────────┘
                                          │
                                          v
                     ┌─────────────────────────────────────────┐
                     │  Credit Limit Service                     │
                     │  - Check credit availability             │
                     │  - Redis: credit:{customerId}            │
                     │  - INCRBY to track usage                 │
                     └────────────────────┬────────────────────┘
                                          │
                     ┌────────────────────v────────────────────┐
                     │  Kafka Topics                             │
                     │  - OrderCreated                           │
                     │  - InvoiceGenerated                       │
                     │  - PaymentProcessed                       │
                     │  - DunningTriggered                       │
                     └─────────┬──────────┬──────────┬──────────┘
                               │          │          │
             ┌─────────────────v─┐ ┌──────v──────────v──────┐
             │ Invoice Service   │ │ Dunning Service        │
             │ - Monthly batch   │ │ - Scheduled checks     │
             │ - Generate PDF    │ │ - Send reminders       │
             │ - Email via       │ │ - Escalation logic     │
             │   SendGrid        │ │                        │
             └─────────────────┬─┘ └──────┬─────────────────┘
                               │          │
             ┌─────────────────v──────────v────────────────┐
             │  Payment Service                             │
             │  - Record payments                           │
             │  - Partial payment handling                  │
             │  - Early discount calculation                │
             │  - Update invoice status                     │
             └─────────────────┬────────────────────────────┘
                               │
             ┌─────────────────v────────────────────────────┐
             │  Accounting Adapter                           │
             │  - QuickBooksAdapter                          │
             │  - SAPAdapter                                 │
             │  - Sync invoice + payment                     │
             └────────────────────────────────────────────┘

          ┌─────────────────────────────────────┐
          │  PostgreSQL                          │
          │  - Customers, Credit Limits          │
          │  - Orders, Invoices, Payments        │
          │  - Dunning Status, Audit Log         │
          └─────────────────────────────────────┘

          ┌─────────────────────────────────────┐
          │  MongoDB                             │
          │  - Invoice PDFs (binary)             │
          │  - PO Documents                      │
          │  - Audit Trail (high volume)         │
          └─────────────────────────────────────┘

          ┌─────────────────────────────────────┐
          │  Redis                               │
          │  - Credit limits (real-time)         │
          │  - Dunning state cache               │
          │  - Session locks                     │
          └─────────────────────────────────────┘
```

---

## Core Design Questions Answered

### 1. How do you track credit limits in real-time during order placement?

**Answer:**
- Store `credit:{customerId}` in Redis as a JSON: `{ "total_limit": 50000, "current_used": 12000, "available": 38000 }`
- On order creation: call `CreditLimitService.validateAndReserve(customerId, orderAmount)`
  - Atomically: `INCRBY credit:{customerId}:used orderAmount`
  - If resulting used > limit → reject order (REDIS MULTI/EXEC for atomicity)
- Post-transaction: update PostgreSQL customer record to reflect latest used amount
- If Redis fails → fallback to PostgreSQL (slower but consistent)

**Implementation Pattern:**
```java
@RedisLock(key = "credit:{customerId}", timeout = 100)
public boolean validateAndReserve(String customerId, BigDecimal amount) {
    String creditKey = "credit:" + customerId;
    Long newUsed = redisTemplate.opsForValue()
        .increment(creditKey + ":used", amount.longValue());
    Long limit = redisTemplate.opsForValue()
        .get(creditKey + ":limit");
    if (newUsed > limit) {
        redisTemplate.opsForValue()
            .decrement(creditKey + ":used", amount.longValue());
        return false;
    }
    return true;
}
```

### 2. How do you generate and send invoices?

**Answer:**
- Monthly Spring Batch job runs at 1 AM UTC on 1st of each month
- Job step 1: Query all customers → for each customer, aggregate orders placed in previous month
- Job step 2: Generate PDF via iTextPDF or LibreOffice (include lineitem, terms, PO ref, discount eligibility)
- Job step 3: Store PDF in MongoDB with metadata (customerId, invoiceId, generatedAt)
- Job step 4: Email via SendGrid with PDF attachment
- Job step 5: Create InvoiceRecord in PostgreSQL; emit InvoiceGenerated event to Kafka
- Idempotency: check if invoice already exists for customer + month before generating

**Key Fields in Invoice:**
```
- Invoice #: format: INV-{YYYYMM}-{customerId}-{seq}
- Invoice Date: first day of current month
- Due Date: invoice date + 30 days
- Early Payment Date: invoice date + 10 days (for 2% discount)
- Items: all orders placed in previous month
- Net Amount: sum of order totals
- Early Payment Discount: 2% if paid by early payment date
- Total Due: net amount - early discount (if applicable)
```

### 3. How do you implement payment terms and dunning?

**Answer:**
- **Payment Terms:** NET-30 encoded in invoice. Due date = invoice date + 30 days.
- **Dunning State Machine:**
  ```
  INVOICE_CREATED
    → check at +7 days: if unpaid → DUNNING_7_DAYS → send email
    → check at +14 days: if unpaid → DUNNING_14_DAYS → send email + SMS
    → check at +21 days: if unpaid → DUNNING_21_DAYS → send email + phone call
    → check at +30 days: if unpaid → OVERDUE_30_PLUS → escalate to collections
    → (if payment received) → PAID or PARTIALLY_PAID → exit dunning
  ```
- Dunning job runs daily; queries all invoices with status in DUNNING_*; checks days overdue; transitions state
- Reminder emails use templates with payment link (hosted secure form)
- Store dunning state in PostgreSQL `invoice_dunning_state` table

### 4. How do you handle partial payments on invoices?

**Answer:**
- Invoice has column `amount_due` (initially = net amount). `amount_paid` = 0 initially.
- Each payment creates `InvoicePayment` record: `(invoiceId, paymentAmount, paymentDate, method, earlyDiscount, ...)`
- After payment: recalculate `amount_due = net_amount - sum(all invoice_payment.amount) + sum(all early_discounts_applied)`
- Invoice status:
  - `UNPAID` if amount_due > 0 and amount_paid = 0
  - `PARTIALLY_PAID` if amount_due > 0 and amount_paid > 0
  - `PAID` if amount_due = 0
  - `OVERPAID` (edge case) if amount_paid > net_amount; refund difference
- Payments recorded via PaymentService.recordPayment(invoiceId, amount, paymentMethod)

### 5. How do you calculate early payment discounts?

**Answer:**
- At payment time: call `EarlyPaymentDiscountCalculator.calculateDiscount(invoiceId, paymentDate)`
- Logic:
  ```
  if (paymentDate <= invoiceDate + 10 days) {
      discount = invoiceNetAmount * 0.02  // 2% discount
      amountApplied = paymentAmount + discount
      return discount
  }
  return 0
  ```
- Discount recorded in InvoicePayment record; applied to amount_due calculation
- Can be applied multiple times if payment is split across dates (rare but handled)

### 6. How do you integrate with accounting systems (QuickBooks, SAP)?

**Answer:**
- **Adapter Pattern:** `AccountingAdapter` interface with methods:
  ```java
  void syncInvoice(Invoice invoice);
  void syncPayment(InvoicePayment payment);
  void syncCredit(Customer customer);
  ```
- **Concrete Adapters:**
  - `QuickBooksAdapter`: REST calls to QB API; map Invoice → QB Invoice; PaymentPayload → QB Payment
  - `SAPAdapter`: SOAP/RFC calls to SAP; batch integration via SAP IDocs
- **Event-Driven Sync:** Kafka consumers listen to:
  - `InvoiceGenerated` → call adapter.syncInvoice()
  - `PaymentProcessed` → call adapter.syncPayment()
- **Audit Trail:** Every adapter call logged in PostgreSQL with status (success/failed); retry mechanism for failed syncs
- **Configuration:** Adapter selection via feature flag or customer profile

---

## Microservices Breakdown

| Microservice | Port | Responsibilities | Key Dependencies |
|--------------|------|------------------|------------------|
| Credit Limit Service | 8001 | Validate/reserve credit; track usage | Redis, PostgreSQL |
| Invoice Service | 8002 | Generate monthly invoices; store PDFs; email | MongoDB, SendGrid, PostgreSQL |
| Payment Service | 8003 | Record payments; partial payment logic; early discounts | PostgreSQL, Kafka |
| Dunning Service | 8004 | Scheduled dunning checks; send reminders; escalation | PostgreSQL, Email Service, Kafka |
| Accounting Adapter | 8005 | Sync to QuickBooks/SAP; retry logic | PostgreSQL, 3rd-party APIs |
| Reporting Service | 8006 | Aging reports; customer statements; dashboards | PostgreSQL, MongoDB |

---

## Database Design (DDL)

### PostgreSQL Schema

```sql
-- Customers Table
CREATE TABLE customers (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    credit_limit DECIMAL(15, 2) NOT NULL DEFAULT 0,
    credit_used DECIMAL(15, 2) NOT NULL DEFAULT 0,
    payment_terms INT DEFAULT 30,  -- days
    tier VARCHAR(50) DEFAULT 'standard',  -- standard, premium, enterprise
    accounting_system VARCHAR(50),  -- quickbooks, sap, netsuite
    accounting_id VARCHAR(255),  -- external ID in 3rd-party system
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_customers_tier ON customers(tier);
CREATE INDEX idx_customers_accounting_system ON customers(accounting_system);

-- Purchase Orders Table
CREATE TABLE purchase_orders (
    id UUID PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES customers(id),
    po_number VARCHAR(100) NOT NULL,
    po_amount DECIMAL(15, 2) NOT NULL,
    po_date DATE NOT NULL,
    expiry_date DATE,
    status VARCHAR(50) DEFAULT 'active',  -- active, expired, closed
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(customer_id, po_number)
);

CREATE INDEX idx_po_customer ON purchase_orders(customer_id);
CREATE INDEX idx_po_status ON purchase_orders(status);

-- Orders Table (linked from Order Service, simplified here)
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES customers(id),
    po_id UUID REFERENCES purchase_orders(id),
    order_amount DECIMAL(15, 2) NOT NULL,
    order_date TIMESTAMP NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Invoices Table
CREATE TABLE invoices (
    id UUID PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES customers(id),
    invoice_number VARCHAR(100) NOT NULL UNIQUE,
    invoice_date DATE NOT NULL,
    due_date DATE NOT NULL,
    early_payment_date DATE NOT NULL,  -- invoice_date + 10 days
    net_amount DECIMAL(15, 2) NOT NULL,
    amount_paid DECIMAL(15, 2) DEFAULT 0,
    amount_due DECIMAL(15, 2) NOT NULL,
    status VARCHAR(50) DEFAULT 'unpaid',  -- unpaid, partially_paid, paid, overdue
    accounting_id VARCHAR(255),  -- QuickBooks/SAP invoice ID
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_invoices_customer_date ON invoices(customer_id, invoice_date);
CREATE INDEX idx_invoices_status ON invoices(status);
CREATE INDEX idx_invoices_due_date ON invoices(due_date);

-- Invoice Line Items
CREATE TABLE invoice_line_items (
    id UUID PRIMARY KEY,
    invoice_id UUID NOT NULL REFERENCES invoices(id) ON DELETE CASCADE,
    order_id UUID REFERENCES orders(id),
    description VARCHAR(500) NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(15, 2) NOT NULL,
    line_amount DECIMAL(15, 2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_line_items_invoice ON invoice_line_items(invoice_id);

-- Invoice Payments
CREATE TABLE invoice_payments (
    id UUID PRIMARY KEY,
    invoice_id UUID NOT NULL REFERENCES invoices(id),
    payment_amount DECIMAL(15, 2) NOT NULL,
    early_discount DECIMAL(15, 2) DEFAULT 0,
    payment_method VARCHAR(50) NOT NULL,  -- credit_card, bank_transfer, check
    payment_date TIMESTAMP NOT NULL,
    reference_number VARCHAR(255),
    status VARCHAR(50) DEFAULT 'processed',  -- processed, failed, reversed
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payments_invoice_date ON invoice_payments(invoice_id, payment_date);

-- Dunning State
CREATE TABLE invoice_dunning_state (
    id UUID PRIMARY KEY,
    invoice_id UUID NOT NULL UNIQUE REFERENCES invoices(id),
    state VARCHAR(50) DEFAULT 'unpaid',  -- unpaid, dunning_7, dunning_14, dunning_21, overdue_30, paid
    last_dunning_sent_at TIMESTAMP,
    dunning_count INT DEFAULT 0,
    next_check_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_dunning_state ON invoice_dunning_state(state, next_check_date);

-- Audit Log
CREATE TABLE audit_log (
    id UUID PRIMARY KEY,
    entity_type VARCHAR(100),  -- invoice, payment, customer
    entity_id UUID,
    action VARCHAR(100),  -- created, updated, paid, dunning_sent
    user_id VARCHAR(255),
    details JSONB,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_audit_entity ON audit_log(entity_type, entity_id);
CREATE INDEX idx_audit_created_at ON audit_log(created_at);
```

### MongoDB Schema

```json
{
  "_id": "ObjectId",
  "invoiceId": "UUID (same as PostgreSQL)",
  "customerId": "UUID",
  "invoiceNumber": "INV-202401-cust001-001",
  "pdfUrl": "s3://bucket/invoices/INV-202401-cust001-001.pdf",
  "pdfContent": "BinData (actual PDF file)",
  "poDocument": "BinData (PO scanned image, optional)",
  "metadata": {
    "generatedAt": "2024-01-01T08:00:00Z",
    "generatedBy": "InvoiceGenerationJob",
    "emailSentAt": "2024-01-01T08:05:00Z",
    "emailStatus": "sent"
  },
  "createdAt": "2024-01-01T08:00:00Z"
}
```

---

## Redis Data Structures

```
# Credit Limit Hash (per customer)
credit:{customerId}
  ├─ total_limit: 50000
  ├─ current_used: 12000
  ├─ available: 38000
  └─ updated_at: 1704067200

# Dunning State Cache
dunning:{invoiceId}
  ├─ state: dunning_7
  ├─ last_sent_at: 1704067200
  ├─ send_count: 1
  └─ next_check: 1704240000

# Invoice Lock (during payment processing)
invoice:lock:{invoiceId}
  └─ value: timestamp (expires after 30 seconds)

# Early Payment Window (bloom filter for quick lookup)
early_payment:invoices:{date}
  └─ set of invoiceIds eligible for early payment discount on this date
```

---

## Kafka Event Flow

```
Topic: invoice-events
Partition Key: customerId

Event 1: InvoiceGenerated
{
  "eventId": "uuid",
  "invoiceId": "uuid",
  "customerId": "uuid",
  "invoiceNumber": "INV-202401-cust001-001",
  "invoiceDate": "2024-01-01",
  "dueDate": "2024-01-31",
  "netAmount": 10000.00,
  "timestamp": "2024-01-01T08:00:00Z"
}
Consumers:
  - AccountingAdapterService (sync to QuickBooks/SAP)
  - DunningService (initialize dunning state)
  - ReportingService (aggregate for dashboards)

Event 2: PaymentProcessed
{
  "eventId": "uuid",
  "invoiceId": "uuid",
  "customerId": "uuid",
  "paymentAmount": 10000.00,
  "earlyDiscount": 200.00,
  "paymentMethod": "bank_transfer",
  "paymentDate": "2024-01-10",
  "timestamp": "2024-01-10T14:30:00Z"
}
Consumers:
  - DunningService (clear dunning state if fully paid)
  - AccountingAdapterService (sync payment to 3rd-party)
  - ReportingService (update AR aging report)
  - CreditLimitService (release reserved credit)

Event 3: DunningTriggered
{
  "eventId": "uuid",
  "invoiceId": "uuid",
  "customerId": "uuid",
  "dunningState": "dunning_7",
  "daysOverdue": 7,
  "timestamp": "2024-02-07T06:00:00Z"
}
Consumers:
  - EmailService (send payment reminder)
  - ReportingService (track dunning metrics)

Event 4: EarlyPaymentDiscount Applied
{
  "eventId": "uuid",
  "invoiceId": "uuid",
  "customerId": "uuid",
  "discountAmount": 200.00,
  "discountPercentage": 2.0,
  "timestamp": "2024-01-10T14:30:00Z"
}
Consumers:
  - ReportingService (track discount trends)
  - FinanceService (record journal entry)
```

---

## Implementation Code

### CreditLimitService

```java
package com.billing.service.credit;

import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
public class CreditLimitService {

    private final RedisTemplate<String, Object> redisTemplate;
    private final CustomerRepository customerRepository;
    private static final String CREDIT_KEY_PREFIX = "credit:";

    public CreditLimitService(RedisTemplate<String, Object> redisTemplate,
                               CustomerRepository customerRepository) {
        this.redisTemplate = redisTemplate;
        this.customerRepository = customerRepository;
    }

    /**
     * Validate and reserve credit for an order.
     * Returns true if reservation successful, false if insufficient credit.
     */
    @Transactional
    public boolean validateAndReserveCredit(UUID customerId, BigDecimal orderAmount) {
        String creditKey = CREDIT_KEY_PREFIX + customerId.toString();

        try {
            // Get current limit and used amount atomically
            Long limit = (Long) redisTemplate.opsForValue().get(creditKey + ":limit");
            Long currentUsed = (Long) redisTemplate.opsForValue().get(creditKey + ":used");

            if (limit == null || currentUsed == null) {
                // Cache miss, load from PostgreSQL
                syncCreditFromDatabase(customerId);
                return validateAndReserveCredit(customerId, orderAmount);
            }

            long orderAmountLong = orderAmount.setScale(2).longValue() * 100; // convert to cents
            long newUsed = currentUsed + orderAmountLong;

            if (newUsed > limit) {
                log.warn("Insufficient credit for customer {}: limit={}, used={}, requested={}",
                         customerId, limit, currentUsed, orderAmountLong);
                return false;
            }

            // Atomically increment used amount
            Boolean result = (Boolean) redisTemplate.execute(connection -> {
                byte[] key = (creditKey + ":used").getBytes();
                connection.incr(key, orderAmountLong);
                return true;
            });

            // Schedule async sync to PostgreSQL
            updateCreditInDatabase(customerId, orderAmount);

            log.info("Credit reserved for customer {}: amount={}", customerId, orderAmount);
            return true;

        } catch (Exception e) {
            log.error("Error validating credit for customer {}", customerId, e);
            // Fallback to database check
            return validateCreditFromDatabase(customerId, orderAmount);
        }
    }

    /**
     * Release previously reserved credit (e.g., order cancelled).
     */
    @Transactional
    public void releaseCredit(UUID customerId, BigDecimal orderAmount) {
        String creditKey = CREDIT_KEY_PREFIX + customerId.toString();
        long orderAmountLong = orderAmount.setScale(2).longValue() * 100;

        try {
            redisTemplate.opsForValue().decrement(creditKey + ":used", orderAmountLong);
            updateCreditInDatabase(customerId, orderAmount.negate());
        } catch (Exception e) {
            log.error("Error releasing credit for customer {}", customerId, e);
        }
    }

    /**
     * Sync credit limit from PostgreSQL to Redis.
     */
    public void syncCreditFromDatabase(UUID customerId) {
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new IllegalArgumentException("Customer not found"));

        String creditKey = CREDIT_KEY_PREFIX + customerId.toString();
        long limitCents = customer.getCreditLimit().setScale(2).longValue() * 100;
        long usedCents = customer.getCreditUsed().setScale(2).longValue() * 100;

        redisTemplate.opsForValue().set(creditKey + ":limit", limitCents, 1, TimeUnit.HOURS);
        redisTemplate.opsForValue().set(creditKey + ":used", usedCents, 1, TimeUnit.HOURS);
    }

    /**
     * Update credit used in PostgreSQL asynchronously.
     */
    private void updateCreditInDatabase(UUID customerId, BigDecimal amountDelta) {
        // Fire-and-forget async update
        java.util.concurrent.ForkJoinPool.commonPool().execute(() -> {
            try {
                Customer customer = customerRepository.findById(customerId)
                    .orElseReturn();
                customer.setCreditUsed(customer.getCreditUsed().add(amountDelta));
                customerRepository.save(customer);
            } catch (Exception e) {
                log.error("Error updating credit in database for customer {}", customerId, e);
            }
        });
    }

    /**
     * Fallback: validate credit from PostgreSQL (slower).
     */
    private boolean validateCreditFromDatabase(UUID customerId, BigDecimal orderAmount) {
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new IllegalArgumentException("Customer not found"));

        BigDecimal available = customer.getCreditLimit().subtract(customer.getCreditUsed());
        if (available.compareTo(orderAmount) < 0) {
            return false;
        }

        customer.setCreditUsed(customer.getCreditUsed().add(orderAmount));
        customerRepository.save(customer);
        return true;
    }
}
```

### InvoiceGenerationJob

```java
package com.billing.service.invoice;

import org.springframework.batch.core.*;
import org.springframework.batch.core.job.builder.JobBuilder;
import org.springframework.batch.core.repository.JobRepository;
import org.springframework.batch.core.step.builder.StepBuilder;
import org.springframework.batch.item.database.JdbcPagingItemReader;
import org.springframework.batch.item.ItemProcessor;
import org.springframework.batch.item.ItemWriter;
import org.springframework.stereotype.Component;
import org.springframework.transaction.PlatformTransactionManager;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.YearMonth;
import java.util.*;
import java.util.stream.Collectors;
import com.itextpdf.kernel.pdf.PdfWriter;
import com.itextpdf.layout.Document;

@Slf4j
@Component
public class InvoiceGenerationJob {

    private final JobRepository jobRepository;
    private final PlatformTransactionManager transactionManager;
    private final CustomerRepository customerRepository;
    private final OrderRepository orderRepository;
    private final InvoiceRepository invoiceRepository;
    private final InvoicePaymentRepository invoicePaymentRepository;
    private final MongoTemplate mongoTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final SendGridEmailService emailService;

    public InvoiceGenerationJob(JobRepository jobRepository,
                                 PlatformTransactionManager transactionManager,
                                 CustomerRepository customerRepository,
                                 OrderRepository orderRepository,
                                 InvoiceRepository invoiceRepository,
                                 InvoicePaymentRepository invoicePaymentRepository,
                                 MongoTemplate mongoTemplate,
                                 KafkaTemplate<String, Object> kafkaTemplate,
                                 SendGridEmailService emailService) {
        this.jobRepository = jobRepository;
        this.transactionManager = transactionManager;
        this.customerRepository = customerRepository;
        this.orderRepository = orderRepository;
        this.invoiceRepository = invoiceRepository;
        this.invoicePaymentRepository = invoicePaymentRepository;
        this.mongoTemplate = mongoTemplate;
        this.kafkaTemplate = kafkaTemplate;
        this.emailService = emailService;
    }

    public Job invoiceGenerationJob() {
        return new JobBuilder("invoiceGenerationJob", jobRepository)
            .start(invoiceGenerationStep())
            .build();
    }

    private Step invoiceGenerationStep() {
        return new StepBuilder("invoiceGenerationStep", jobRepository)
            .<Customer, Invoice>chunk(100, transactionManager)
            .reader(customerReader())
            .processor(invoiceProcessor())
            .writer(invoiceWriter())
            .build();
    }

    private JdbcPagingItemReader<Customer> customerReader() {
        JdbcPagingItemReader<Customer> reader = new JdbcPagingItemReader<>();
        reader.setDataSource(dataSource);
        reader.setPageSize(100);
        reader.setRowMapper((rs, rowNum) -> {
            Customer c = new Customer();
            c.setId(UUID.fromString(rs.getString("id")));
            c.setName(rs.getString("name"));
            c.setEmail(rs.getString("email"));
            return c;
        });

        SqlPagingQueryProviderFactoryBean queryProvider = new SqlPagingQueryProviderFactoryBean();
        queryProvider.setDataSource(dataSource);
        queryProvider.setSelectClause("SELECT id, name, email, credit_limit");
        queryProvider.setFromClause("FROM customers");
        queryProvider.setWhereClause("WHERE active = true");
        queryProvider.setSortKey("id");
        reader.setQueryProvider(queryProvider.getObject());
        return reader;
    }

    private ItemProcessor<Customer, Invoice> invoiceProcessor() {
        return customer -> {
            YearMonth previousMonth = YearMonth.now().minusMonths(1);
            LocalDate monthStart = previousMonth.atDay(1);
            LocalDate monthEnd = previousMonth.atEndOfMonth();

            // Check idempotency: invoice already generated for this customer + month?
            Optional<Invoice> existingInvoice = invoiceRepository
                .findByCustomerIdAndInvoiceDateBetween(customer.getId(), monthStart, monthEnd);
            if (existingInvoice.isPresent()) {
                log.info("Invoice already generated for customer {} for {}", customer.getId(), previousMonth);
                return null; // Skip
            }

            // Fetch orders for previous month
            List<Order> orders = orderRepository
                .findByCustomerIdAndOrderDateBetween(customer.getId(), monthStart.atStartOfDay(), monthEnd.atTime(23, 59, 59));

            if (orders.isEmpty()) {
                log.info("No orders for customer {} in {}", customer.getId(), previousMonth);
                return null; // No invoice needed
            }

            // Create invoice
            Invoice invoice = new Invoice();
            invoice.setId(UUID.randomUUID());
            invoice.setCustomerId(customer.getId());
            invoice.setInvoiceNumber(generateInvoiceNumber(customer.getId()));
            invoice.setInvoiceDate(LocalDate.now());
            invoice.setDueDate(LocalDate.now().plusDays(30)); // NET-30
            invoice.setEarlyPaymentDate(LocalDate.now().plusDays(10));

            BigDecimal netAmount = orders.stream()
                .map(Order::getOrderAmount)
                .reduce(BigDecimal.ZERO, BigDecimal::add);

            invoice.setNetAmount(netAmount);
            invoice.setAmountPaid(BigDecimal.ZERO);
            invoice.setAmountDue(netAmount);
            invoice.setStatus("unpaid");

            // Create line items
            List<InvoiceLineItem> lineItems = orders.stream()
                .map(order -> {
                    InvoiceLineItem item = new InvoiceLineItem();
                    item.setId(UUID.randomUUID());
                    item.setInvoiceId(invoice.getId());
                    item.setOrderId(order.getId());
                    item.setDescription("Order #" + order.getId());
                    item.setLineAmount(order.getOrderAmount());
                    return item;
                })
                .collect(Collectors.toList());

            invoice.setLineItems(lineItems);
            return invoice;
        };
    }

    private ItemWriter<Invoice> invoiceWriter() {
        return invoices -> {
            for (Invoice invoice : invoices) {
                try {
                    // Save to PostgreSQL
                    invoiceRepository.save(invoice);

                    // Generate PDF
                    byte[] pdfContent = generateInvoicePdf(invoice);

                    // Store PDF in MongoDB
                    InvoicePdfDocument pdfDoc = new InvoicePdfDocument();
                    pdfDoc.setInvoiceId(invoice.getId().toString());
                    pdfDoc.setCustomerId(invoice.getCustomerId().toString());
                    pdfDoc.setInvoiceNumber(invoice.getInvoiceNumber());
                    pdfDoc.setPdfContent(pdfContent);
                    mongoTemplate.save(pdfDoc);

                    // Email invoice
                    Customer customer = customerRepository.findById(invoice.getCustomerId())
                        .orElseThrow();
                    emailService.sendInvoice(customer.getEmail(), invoice, pdfContent);

                    // Emit event
                    InvoiceGeneratedEvent event = new InvoiceGeneratedEvent(
                        UUID.randomUUID(),
                        invoice.getId(),
                        invoice.getCustomerId(),
                        invoice.getInvoiceNumber(),
                        invoice.getNetAmount(),
                        Instant.now()
                    );
                    kafkaTemplate.send("invoice-events", event.getCustomerId().toString(), event);

                    log.info("Invoice generated: {}", invoice.getInvoiceNumber());

                } catch (Exception e) {
                    log.error("Error generating invoice: {}", invoice.getId(), e);
                    throw new ItemWriterException("Failed to generate invoice", e);
                }
            }
        };
    }

    private byte[] generateInvoicePdf(Invoice invoice) throws Exception {
        ByteArrayOutputStream baos = new ByteArrayOutputStream();
        PdfWriter writer = new PdfWriter(baos);
        Document document = new Document(new PdfDocument(writer));

        // Add invoice header
        document.add(new Paragraph("INVOICE")
            .setFontSize(18)
            .setBold());
        document.add(new Paragraph("Invoice #: " + invoice.getInvoiceNumber()));
        document.add(new Paragraph("Invoice Date: " + invoice.getInvoiceDate()));
        document.add(new Paragraph("Due Date: " + invoice.getDueDate()));
        document.add(new Paragraph("Early Payment Date: " + invoice.getEarlyPaymentDate() + " (2% discount)"));

        // Add line items table
        Table table = new Table(UnitValue.createPercentArray(3)).useAllAvailableWidth();
        table.addHeaderCell("Description");
        table.addHeaderCell("Quantity");
        table.addHeaderCell("Amount");

        for (InvoiceLineItem item : invoice.getLineItems()) {
            table.addCell(item.getDescription());
            table.addCell(String.valueOf(item.getQuantity()));
            table.addCell("$" + item.getLineAmount());
        }
        document.add(table);

        // Add totals
        document.add(new Paragraph("\n"));
        document.add(new Paragraph("Net Amount: $" + invoice.getNetAmount())
            .setTextAlignment(TextAlignment.RIGHT));
        document.add(new Paragraph("Early Payment Discount (if paid by " + invoice.getEarlyPaymentDate() + "): 2%")
            .setTextAlignment(TextAlignment.RIGHT));
        document.add(new Paragraph("Total Due: $" + invoice.getAmountDue())
            .setTextAlignment(TextAlignment.RIGHT)
            .setBold());

        document.close();
        return baos.toByteArray();
    }

    private String generateInvoiceNumber(UUID customerId) {
        YearMonth month = YearMonth.now();
        int seq = invoiceRepository.countByCustomerIdAndInvoiceDateBetween(
            customerId, month.atDay(1), month.atEndOfMonth()) + 1;
        return String.format("INV-%s-%s-%03d", month, customerId.toString().substring(0, 8), seq);
    }
}
```

### DunningScheduler

```java
package com.billing.service.dunning;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.kafka.core.KafkaTemplate;
import lombok.extern.slf4j.Slf4j;
import java.time.LocalDate;
import java.util.List;
import java.util.UUID;

@Slf4j
@Service
public class DunningScheduler {

    private final InvoiceRepository invoiceRepository;
    private final InvoiceDunningStateRepository dunningStateRepository;
    private final EmailService emailService;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    public DunningScheduler(InvoiceRepository invoiceRepository,
                             InvoiceDunningStateRepository dunningStateRepository,
                             EmailService emailService,
                             KafkaTemplate<String, Object> kafkaTemplate) {
        this.invoiceRepository = invoiceRepository;
        this.dunningStateRepository = dunningStateRepository;
        this.emailService = emailService;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Scheduled(cron = "0 6 * * *")  // Daily at 6 AM
    public void processDunning() {
        log.info("Starting dunning process");
        LocalDate today = LocalDate.now();

        // Get all invoices needing dunning checks
        List<InvoiceDunningState> dunningStates = dunningStateRepository
            .findByStateInAndNextCheckDateLessThanEqual(
                List.of("unpaid", "dunning_7", "dunning_14", "dunning_21"),
                today
            );

        for (InvoiceDunningState state : dunningStates) {
            Invoice invoice = invoiceRepository.findById(state.getInvoiceId())
                .orElse(null);

            if (invoice == null || "paid".equals(invoice.getStatus())) {
                continue;
            }

            LocalDate dueDate = invoice.getDueDate();
            long daysOverdue = java.time.temporal.ChronoUnit.DAYS.between(dueDate, today);

            String newState = determineNewState(daysOverdue, state.getState());

            if (!newState.equals(state.getState())) {
                // Transition to new state
                state.setState(newState);
                state.setLastDunningSentAt(java.time.Instant.now());
                state.setDunningCount(state.getDunningCount() + 1);
                state.setNextCheckDate(calculateNextCheckDate(newState, today));
                dunningStateRepository.save(state);

                // Send email
                Customer customer = invoice.getCustomer();
                String template = getEmailTemplate(newState);
                emailService.sendDunningEmail(customer.getEmail(), invoice, template, daysOverdue);

                // Emit event
                DunningTriggeredEvent event = new DunningTriggeredEvent(
                    UUID.randomUUID(),
                    invoice.getId(),
                    invoice.getCustomerId(),
                    newState,
                    daysOverdue,
                    java.time.Instant.now()
                );
                kafkaTemplate.send("dunning-events", invoice.getCustomerId().toString(), event);

                log.info("Dunning sent for invoice {}: state={}, daysOverdue={}",
                         invoice.getInvoiceNumber(), newState, daysOverdue);
            }
        }
    }

    private String determineNewState(long daysOverdue, String currentState) {
        if (daysOverdue < 7) {
            return "unpaid";
        } else if (daysOverdue < 14) {
            return "dunning_7";
        } else if (daysOverdue < 21) {
            return "dunning_14";
        } else if (daysOverdue < 30) {
            return "dunning_21";
        } else {
            return "overdue_30";
        }
    }

    private LocalDate calculateNextCheckDate(String state, LocalDate today) {
        return switch (state) {
            case "unpaid" -> today.plusDays(7);
            case "dunning_7" -> today.plusDays(7);
            case "dunning_14" -> today.plusDays(7);
            case "dunning_21" -> today.plusDays(9);
            case "overdue_30" -> today.plusDays(30);
            default -> today.plusDays(1);
        };
    }

    private String getEmailTemplate(String state) {
        return switch (state) {
            case "dunning_7" -> "reminder_7_days_overdue";
            case "dunning_14" -> "reminder_14_days_overdue_urgent";
            case "dunning_21" -> "reminder_21_days_overdue_critical";
            case "overdue_30" -> "escalation_to_collections";
            default -> "payment_reminder";
        };
    }
}
```

### EarlyPaymentDiscountCalculator

```java
package com.billing.service.payment;

import org.springframework.stereotype.Service;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.temporal.ChronoUnit;
import java.util.UUID;

@Slf4j
@Service
public class EarlyPaymentDiscountCalculator {

    private static final BigDecimal EARLY_PAYMENT_DISCOUNT_RATE = new BigDecimal("0.02"); // 2%

    /**
     * Calculate early payment discount for a given invoice at payment time.
     * Discount applies if payment is made within 10 days of invoice date.
     */
    public EarlyPaymentDiscount calculateDiscount(Invoice invoice, LocalDate paymentDate) {
        LocalDate invoiceDate = invoice.getInvoiceDate();
        LocalDate earlyPaymentDeadline = invoice.getEarlyPaymentDate(); // invoice_date + 10 days

        EarlyPaymentDiscount discount = new EarlyPaymentDiscount();
        discount.setInvoiceId(invoice.getId());
        discount.setPaymentDate(paymentDate);
        discount.setEligible(false);
        discount.setDiscountAmount(BigDecimal.ZERO);
        discount.setDiscountPercentage(BigDecimal.ZERO);

        // Check if payment is within early payment window
        if (paymentDate.isBefore(earlyPaymentDeadline) || paymentDate.isEqual(earlyPaymentDeadline)) {
            discount.setEligible(true);
            discount.setDiscountAmount(invoice.getNetAmount().multiply(EARLY_PAYMENT_DISCOUNT_RATE));
            discount.setDiscountPercentage(EARLY_PAYMENT_DISCOUNT_RATE.multiply(new BigDecimal("100")));

            log.info("Early payment discount eligible for invoice {}: discount={}%",
                     invoice.getInvoiceNumber(), discount.getDiscountPercentage());
        } else {
            long daysLate = ChronoUnit.DAYS.between(earlyPaymentDeadline, paymentDate);
            log.info("Early payment discount NOT eligible for invoice {}: {} days late",
                     invoice.getInvoiceNumber(), daysLate);
        }

        return discount;
    }
}

record EarlyPaymentDiscount(
    UUID invoiceId,
    LocalDate paymentDate,
    boolean eligible,
    BigDecimal discountAmount,
    BigDecimal discountPercentage
) { }
```

### PartialPaymentService

```java
package com.billing.service.payment;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.time.LocalDate;
import java.util.UUID;

@Slf4j
@Service
public class PartialPaymentService {

    private final InvoiceRepository invoiceRepository;
    private final InvoicePaymentRepository invoicePaymentRepository;
    private final EarlyPaymentDiscountCalculator discountCalculator;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    @Transactional
    public InvoicePayment recordPartialPayment(UUID invoiceId, BigDecimal paymentAmount,
                                                String paymentMethod, String referenceNumber) {
        Invoice invoice = invoiceRepository.findById(invoiceId)
            .orElseThrow(() -> new IllegalArgumentException("Invoice not found"));

        LocalDate today = LocalDate.now();

        // Calculate early payment discount if eligible
        EarlyPaymentDiscount earlyDiscount = discountCalculator.calculateDiscount(invoice, today);
        BigDecimal discountToApply = earlyDiscount.eligible() ? earlyDiscount.discountAmount() : BigDecimal.ZERO;

        // Validate payment amount
        if (paymentAmount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("Payment amount must be positive");
        }

        // Create payment record
        InvoicePayment payment = new InvoicePayment();
        payment.setId(UUID.randomUUID());
        payment.setInvoiceId(invoiceId);
        payment.setPaymentAmount(paymentAmount);
        payment.setEarlyDiscount(discountToApply);
        payment.setPaymentMethod(paymentMethod);
        payment.setPaymentDate(java.time.Instant.now());
        payment.setReferenceNumber(referenceNumber);
        payment.setStatus("processed");

        // Update invoice amounts
        BigDecimal newAmountPaid = invoice.getAmountPaid().add(paymentAmount);
        BigDecimal totalAdjustment = paymentAmount.add(discountToApply);
        BigDecimal newAmountDue = invoice.getAmountDue().subtract(totalAdjustment);

        invoice.setAmountPaid(newAmountPaid);
        invoice.setAmountDue(newAmountDue.max(BigDecimal.ZERO)); // Never negative

        // Update invoice status
        if (newAmountDue.compareTo(BigDecimal.ZERO) <= 0) {
            invoice.setStatus("paid");
        } else {
            invoice.setStatus("partially_paid");
        }

        // Save records
        invoicePaymentRepository.save(payment);
        invoiceRepository.save(invoice);

        // Emit event
        PaymentProcessedEvent event = new PaymentProcessedEvent(
            UUID.randomUUID(),
            invoiceId,
            invoice.getCustomerId(),
            paymentAmount,
            discountToApply,
            paymentMethod,
            today,
            java.time.Instant.now()
        );
        kafkaTemplate.send("invoice-events", invoice.getCustomerId().toString(), event);

        log.info("Partial payment recorded: invoiceId={}, amount={}, discount={}, remaining={}",
                 invoiceId, paymentAmount, discountToApply, newAmountDue);

        return payment;
    }
}
```

### AccountingIntegrationAdapter

```java
package com.billing.service.accounting;

import org.springframework.stereotype.Service;
import lombok.extern.slf4j.Slf4j;
import java.util.UUID;

@Slf4j
@Service
public class AccountingIntegrationAdapter {

    private final QuickBooksAdapter quickBooksAdapter;
    private final SAPAdapter sapAdapter;
    private final AccountingAdapterRepository adapterRepository;

    /**
     * Route sync based on customer's configured accounting system.
     */
    public void syncInvoice(Invoice invoice) {
        Customer customer = invoice.getCustomer();

        try {
            switch (customer.getAccountingSystem()) {
                case "quickbooks":
                    quickBooksAdapter.syncInvoice(invoice);
                    break;
                case "sap":
                    sapAdapter.syncInvoice(invoice);
                    break;
                case "netsuite":
                    netsuiteAdapter.syncInvoice(invoice);
                    break;
                default:
                    log.warn("No accounting adapter for customer {}", customer.getId());
            }

            // Record successful sync
            adapterRepository.save(new AdapterSyncRecord(
                UUID.randomUUID(),
                invoice.getId(),
                customer.getAccountingSystem(),
                "invoice",
                "success",
                java.time.Instant.now()
            ));
        } catch (Exception e) {
            log.error("Error syncing invoice to {}", customer.getAccountingSystem(), e);
            adapterRepository.save(new AdapterSyncRecord(
                UUID.randomUUID(),
                invoice.getId(),
                customer.getAccountingSystem(),
                "invoice",
                "failed",
                java.time.Instant.now(),
                e.getMessage()
            ));
            throw new AccountingSyncException("Failed to sync invoice", e);
        }
    }

    public void syncPayment(InvoicePayment payment) {
        Invoice invoice = payment.getInvoice();
        Customer customer = invoice.getCustomer();

        try {
            switch (customer.getAccountingSystem()) {
                case "quickbooks":
                    quickBooksAdapter.syncPayment(payment);
                    break;
                case "sap":
                    sapAdapter.syncPayment(payment);
                    break;
                default:
                    log.warn("No accounting adapter for customer {}", customer.getId());
            }
        } catch (Exception e) {
            log.error("Error syncing payment to {}", customer.getAccountingSystem(), e);
            throw new AccountingSyncException("Failed to sync payment", e);
        }
    }
}

interface QuickBooksAdapter {
    void syncInvoice(Invoice invoice);
    void syncPayment(InvoicePayment payment);
}

interface SAPAdapter {
    void syncInvoice(Invoice invoice);
    void syncPayment(InvoicePayment payment);
}
```

---

## Failure Scenarios

| Scenario | Handling |
|----------|----------|
| Order placed, credit check fails | Reject order; notify user; suggest increasing credit limit or paying down existing invoices |
| Invoice generation job fails mid-way | Idempotent: re-run marks already-generated invoices as skipped; partial failures logged for manual review |
| Email send fails | Retry via Kafka; store pending emails in DLQ; manual retry admin interface |
| Payment recorded but Kafka fails | Compensating transaction: reverse payment in database; re-emit Kafka event; audit trail captures both |
| Dunning email not sent | Scheduled retry; alert if 3 consecutive failures; fallback to SMS or phone call |
| 3rd-party accounting sync fails | Retry queue; manual review dashboard; prevent invoice from being marked as synced until success |
| Early payment discount miscalculated | Audit trail captures calculation params; refund difference if discovered; alert finance team |
| Partial refund after payout | Credit memo issued; applied against next invoice; dunning logic skips credit memo amounts |

---

## Scaling Strategy

### Horizontal Scaling

| Component | Scaling Strategy |
|-----------|------------------|
| Credit Limit Service | Stateless; scale horizontally; Redis for distributed locks |
| Invoice Generation Job | Partition by customer ID; scale Spring Batch partitions across instances |
| Payment Service | Stateless REST API; scale with load balancer; Kafka ensures order per customer |
| Dunning Service | Single instance OK (scheduled job); or use leader election if multi-instance |
| Reporting Service | Read-only; replicate PostgreSQL read replica; cache in Redis |

### Database Optimization

- **Invoices Table:** Partition by invoice_date; archive old invoices to cold storage after 7 years
- **Payments Table:** Partition by invoice_date; index on (invoice_id, payment_date)
- **Audit Log:** Write to separate fast table; archive monthly to TimescaleDB or ClickHouse for analytics
- **Redis:** Cluster mode if >100K customers; TTL on keys (1 hour) to prevent memory bloat

### Kafka Topic Configuration

- **Partitions:** Partition by customerId to maintain order per customer; ~50 partitions for 10K customers
- **Replication Factor:** 3 for high availability
- **Retention:** 7 days for invoice events; 30 days for audit trail
- **Consumer Groups:** One per service (InvoiceService, DunningService, ReportingService)

---

## Monitoring

### Key Metrics

| Metric | Target | Tool |
|--------|--------|------|
| Credit check latency | <100ms | Prometheus histogram |
| Invoice generation time/customer | <5s | Prometheus timer |
| Payment processing latency | <500ms | CloudWatch |
| Dunning email delivery rate | >95% | SendGrid webhook events |
| Accounting sync success rate | >99% | Custom metrics |
| Kafka lag (payment consumer) | <1 min | Kafka consumer lag monitor |

### Alerts

```yaml
- alert: CreditCheckLatency
  expr: histogram_quantile(0.95, http_request_duration_seconds{endpoint="/credit/check"}) > 0.1
  for: 5m
  action: page on-call

- alert: InvoiceGenerationFailure
  expr: job_failures_total{job="invoiceGenerationJob"} > 0
  for: 1m
  action: notify engineering + finance

- alert: DunningEmailSendFailure
  expr: dunning_email_send_failures_total > 10
  for: 10m
  action: page on-call

- alert: AccountingSyncFailure
  expr: accounting_sync_failures_total{status="failed"} > 5
  for: 5m
  action: notify integrations team
```

### Dashboards

- **Finance Dashboard:** total AR, aging bucket distribution, dunning status breakdown, early payment discount trends
- **Operational Dashboard:** job execution times, failure rates, Kafka lag, Redis memory usage
- **Customer Dashboard:** credit utilization by tier, payment trends, discount redemption

---

## Summary Cheat Sheet

```
CREDIT LIMITS:
  - Redis key: credit:{customerId} with :limit, :used, :available
  - Atomic INCRBY on order placement; fallback to PostgreSQL on cache miss
  - Sync to DB async to avoid blocking order creation

INVOICING:
  - Monthly Spring Batch job; idempotent (check if already generated)
  - Generate PDF via iTextPDF; store in MongoDB
  - Emit InvoiceGenerated event to Kafka (3 consumers: accounting, dunning, reporting)
  - Email via SendGrid with retry on failure

PAYMENT TERMS & DUNNING:
  - NET-30: due_date = invoice_date + 30 days
  - Dunning state machine: UNPAID → DUNNING_7 → DUNNING_14 → DUNNING_21 → OVERDUE_30
  - Daily scheduled job checks and transitions states; sends reminders
  - Dunning state stored in invoice_dunning_state table with next_check_date

PARTIAL PAYMENTS:
  - Create InvoicePayment record for each payment
  - Recalculate amount_due = net_amount - sum(all payments) + sum(early_discounts)
  - Update invoice status: UNPAID → PARTIALLY_PAID → PAID
  - Support overpayments (refund difference)

EARLY PAYMENT DISCOUNT:
  - 2% discount if paid within 10 days of invoice date
  - Check at payment time: if paymentDate <= invoiceDate + 10 → apply discount
  - Discount applied to amount_due calculation, not separate line item
  - Audited and tracked per invoice

ACCOUNTING INTEGRATION:
  - Adapter pattern: AccountingAdapter interface
  - Concrete adapters: QuickBooksAdapter, SAPAdapter, NetsuiteAdapter
  - Kafka-driven sync: InvoiceGenerated, PaymentProcessed events trigger syncs
  - Retry queue for failed syncs; audit trail of all attempts
```

---

**Last Updated:** 2024-01-15 | **Version:** 1.0 | **Owner:** Billing Team
