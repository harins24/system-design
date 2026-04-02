---
title: Subscription and Recurring Orders
layout: default
---

# Subscription and Recurring Orders — Deep Dive Design

## Scenario

Implement a subscription and recurring orders system for an e-commerce platform where customers can subscribe to products on monthly, quarterly, or custom cadences. The system must auto-charge payment on schedule, create orders and arrange shipments, allow customers to pause, resume, or cancel anytime, support frequency changes, handle payment failures with automatic retries, and send reminder emails before deliveries.

**Tech Stack:** Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka

---

## Table of Contents

1. [Requirements & Constraints](#1-requirements--constraints)
2. [Capacity Estimation](#2-capacity-estimation)
3. [High-Level Architecture](#3-high-level-architecture)
4. [Core Design Questions Answered](#4-core-design-questions-answered)
5. [Microservices Breakdown](#5-microservices-breakdown)
6. [Database Design](#6-database-design)
7. [Redis Data Structures](#7-redis-data-structures)
8. [Kafka Event Flow](#8-kafka-event-flow)
9. [Implementation Code](#9-implementation-code)
10. [Failure Scenarios & Mitigations](#10-failure-scenarios--mitigations)
11. [Scaling Strategy](#11-scaling-strategy)
12. [Monitoring & Observability](#12-monitoring--observability)
13. [Summary Cheat Sheet](#13-summary-cheat-sheet)

---

## 1. Requirements & Constraints

### Functional Requirements
- Customers create subscriptions with configurable frequency (monthly, quarterly, custom days)
- Auto-charge on billing date; retry failed payments 3 times with exponential backoff
- Create order + shipment automatically on successful charge
- Pause/resume/cancel subscriptions with validation
- Change delivery frequency anytime
- Price lock: existing subscribers honor locked price until renewal
- Pre-billing inventory check 24 hours before charge
- Reminder emails 7 days, 1 day, and on-billing-date
- Modifiable fields: frequency, address, card details

### Non-Functional Requirements
- **Throughput:** 10,000 active subscriptions, 1,000 billing events/day
- **Latency:** subscription operations <200ms
- **Availability:** 99.9% SLA; no duplicate charges even on network failures
- **Idempotency:** payment operations must be fully idempotent
- **Audit:** all state changes logged with timestamps and user context
- **Recovery:** resume after 48-hour system downtime without duplicate charges

---

## 2. Capacity Estimation

```
Active Subscriptions: 100,000 (across platform)
Billing Events/Day: ~3,300 (100K / 30 days avg frequency)
Peak Billing Events/Hour: ~500 (during 9-10 AM spike)
Order Creation Rate: 3,300/day = 38/sec
Payment Retry Attempts: 3,300 × 2.5 avg retries = 8,250/day
Emails/Day: 3,300 (reminders) + 3,300 (confirmations) = 6,600
Redis Cache Size: 100K subscriptions × 2KB = ~200 MB
PostgreSQL Rows/Month: Subscriptions + Billing Logs + Audit = ~5 GB/month
MongoDB Subscription Events: ~100K/day × 30 days = 3M documents = ~6 GB/month
```

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     Customer Web/Mobile                         │
└──────────────────────┬──────────────────────────────────────────┘
                       │ REST APIs
        ┌──────────────┴───────────────┐
        │                              │
┌───────▼─────────┐         ┌──────────▼──────────┐
│ Subscription    │         │ Subscription        │
│ API Service     │◄────────┤ State Manager       │
│ (create/pause/  │         │ (validates & saves) │
│  resume/cancel) │         └─────────┬───────────┘
└────────┬────────┘                   │
         │                            │
         └──────────────┬─────────────┘
                        │ Events
                   ┌────▼────────────────────────────────┐
                   │  Kafka Topic: subscriptions          │
                   │  - subscription.created             │
                   │  - subscription.state_changed       │
                   └────┬───────────────┬────────────────┘
                        │               │
        ┌───────────────┘               └──────────────┬──────────────┐
        │                                              │              │
┌───────▼──────────────┐  ┌──────────────────┐  ┌─────▼──────┐  ┌───▼────────┐
│ Billing Scheduler    │  │ Reminder Email   │  │ Inventory  │  │ Moderation │
│ (Quartz Distributed) │  │ Service          │  │ Service    │  │ Dashboard  │
│ - billing.scheduled  │  │ (24h, 1d, 0d)    │  │ (24h       │  │            │
│ - triggers charge    │  │                  │  │  before)   │  │            │
└───────┬──────────────┘  └──────────────────┘  └────────────┘  └────────────┘
        │
        │ Publishes
        ▼
┌──────────────────────────────────────────┐
│ Kafka Topic: billing.scheduled           │
└──────────────────┬───────────────────────┘
                   │
        ┌──────────┴──────────┐
        │                     │
┌───────▼──────────────┐  ┌──▼──────────────────────┐
│ Payment Processor    │  │ Recurring Order         │
│ Service             │  │ Processor               │
│ - charge payment    │  │ - create order/shipment │
│ - retry logic       │  │ - update subscription   │
│ - Kafka events      │  │   last_charged_date     │
└──────────┬───────────┘  └───────────┬──────────────┘
           │                          │
           └──────────────┬───────────┘
                          │ Events
                    ┌─────▼────────────────┐
                    │ PostgreSQL           │
                    │ - subscription       │
                    │ - billing_log        │
                    │ - audit_log          │
                    └──────────────────────┘

                    ┌──────────────────────┐
                    │ MongoDB              │
                    │ - subscription_events│
                    │ - payment_history    │
                    └──────────────────────┘

                    ┌──────────────────────┐
                    │ Redis                │
                    │ - subscription cache │
                    │ - billing state      │
                    │ - locks              │
                    └──────────────────────┘
```

---

## 4. Core Design Questions Answered

### Q1: How do you schedule recurring orders reliably?

**Answer:** Use Quartz Scheduler (distributed, clustered) with idempotent billing jobs. Each subscription has a `next_billing_date` stored in PostgreSQL. A scheduled job runs every 6 hours, scans for subscriptions where `next_billing_date <= now()` and status is `ACTIVE`, publishes `billing.scheduled` events to Kafka. Each billing event is immutable and idempotent (subscription_id + billing_period = unique key). If Quartz crashes, the job resumes on restart and reprocesses pending billings without duplicates because of the idempotency key.

### Q2: How do you handle payment failures and retries?

**Answer:** Payment failures trigger exponential backoff retries (1st: immediate, 2nd: +5min, 3rd: +15min). Use a dedicated `PaymentRetryHandler` that publishes `payment.retry` events to Kafka with retry_count incremented. After 3 failed attempts, transition subscription to `SUSPENDED` state and notify customer via email and push. Retry logic is completely independent from the original billing trigger. Use `payment_id` (UUID) for idempotency.

### Q3: How do you prevent duplicate orders if system has downtime?

**Answer:** Combine idempotency key (`subscription_id:billing_period`) with database unique constraint. Store billing metadata in PostgreSQL: `(subscription_id, billing_period, status)` with unique index. When processing billing event, do `INSERT ... ON CONFLICT DO UPDATE` to make it idempotent. Also use distributed lock (Redis with key `billing:{subscription_id}:lock`, TTL 5 min) to serialize billing operations for same subscription.

### Q4: How do you allow customers to modify subscriptions?

**Answer:** Implement a `SubscriptionModificationService` that validates each change. Allow modifications only in specific states (ACTIVE, PAUSED). Each modification operation: (1) validates new state (e.g., frequency >= 7 days), (2) publishes `subscription.modified` event, (3) updates subscription in PostgreSQL, (4) publishes audit log. Price changes don't apply retroactively; use `price_locked_until` field to honor old price until next billing cycle.

### Q5: How do you ensure inventory is available for scheduled orders?

**Answer:** Implement inventory pre-check job that runs 24 hours before each billing date. Query inventory service for each subscription's SKU. If insufficient stock, publish `subscription.inventory_alert` event → notify customer with option to delay billing by 7 days. If customer confirms delay, update `next_billing_date`. Use Redis cache for fast inventory lookups with 1-hour TTL.

### Q6: How do you handle price changes for existing subscriptions?

**Answer:** Implement price versioning. Store `price_version` with each subscription. New subscription gets current version. Price changes create new version in `product_pricing_history` table. Existing subscriptions continue with old price until `price_locked_until` date (usually end of current billing cycle). New billings apply new price. Publish `subscription.pricing_changed` event for analytics/notifications.

---

## 5. Microservices Breakdown

| Service | Responsibility | Tech | Scaling |
|---------|---|---|---|
| Subscription API | Create, read, pause, resume, cancel | Spring Boot REST | Horizontal, stateless |
| Subscription State Manager | Validates state transitions | Spring Boot | Horizontal, stateless |
| Billing Scheduler | Triggers billing jobs on schedule | Quartz in Cluster mode | 2-3 nodes, distributed |
| Payment Processor | Charges payment, handles retries | Spring Boot + Payment Gateway | Horizontal, stateless |
| Recurring Order Processor | Creates orders/shipments | Spring Boot + Order Service | Horizontal, stateless |
| Reminder Email Service | Sends reminder emails | Spring Boot + Email Service | Horizontal, stateless |
| Inventory Pre-Check | Checks stock 24h before billing | Spring Boot Job | Single, scheduled |
| Audit Service | Logs all state changes | Spring Boot + MongoDB | Horizontal, stateless |

---

## 6. Database Design

### PostgreSQL Schema

```sql
CREATE TABLE subscriptions (
    id UUID PRIMARY KEY,
    customer_id UUID NOT NULL,
    product_id UUID NOT NULL,
    quantity INT NOT NULL,
    billing_frequency_days INT NOT NULL CHECK (billing_frequency_days >= 7),
    status VARCHAR(20) NOT NULL DEFAULT 'ACTIVE'
        CHECK (status IN ('ACTIVE', 'PAUSED', 'SUSPENDED', 'CANCELLED')),
    subscription_price DECIMAL(10,2) NOT NULL,
    price_version INT NOT NULL DEFAULT 1,
    price_locked_until TIMESTAMP,
    next_billing_date TIMESTAMP NOT NULL,
    last_charged_date TIMESTAMP,
    delivery_address_id UUID,
    payment_method_id UUID NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    cancelled_at TIMESTAMP,
    cancellation_reason VARCHAR(255),

    UNIQUE (customer_id, product_id, status) WHERE status != 'CANCELLED',
    INDEX idx_next_billing_date (next_billing_date),
    INDEX idx_customer_id (customer_id),
    INDEX idx_status (status)
);

CREATE TABLE billing_logs (
    id UUID PRIMARY KEY,
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    billing_period DATE NOT NULL,
    charge_amount DECIMAL(10,2) NOT NULL,
    payment_status VARCHAR(20) NOT NULL
        CHECK (payment_status IN ('PENDING', 'SUCCESS', 'FAILED', 'RETRY')),
    retry_count INT DEFAULT 0,
    payment_id UUID,
    charged_at TIMESTAMP,
    next_retry_at TIMESTAMP,
    error_message VARCHAR(500),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE (subscription_id, billing_period),
    INDEX idx_subscription_id (subscription_id),
    INDEX idx_billing_period (billing_period),
    INDEX idx_payment_status (payment_status)
);

CREATE TABLE subscription_modifications (
    id UUID PRIMARY KEY,
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    modification_type VARCHAR(50) NOT NULL
        CHECK (modification_type IN ('FREQUENCY_CHANGE', 'ADDRESS_CHANGE', 'PAYMENT_METHOD_CHANGE', 'QUANTITY_CHANGE')),
    old_value VARCHAR(500),
    new_value VARCHAR(500),
    effective_from TIMESTAMP,
    applied_at TIMESTAMP,
    requested_by UUID NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_subscription_id (subscription_id),
    INDEX idx_effective_from (effective_from)
);

CREATE TABLE audit_logs (
    id UUID PRIMARY KEY,
    subscription_id UUID NOT NULL REFERENCES subscriptions(id),
    event_type VARCHAR(100) NOT NULL,
    old_state JSONB,
    new_state JSONB,
    user_id UUID,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_subscription_id (subscription_id),
    INDEX idx_timestamp (timestamp)
);
```

### MongoDB Collections

```json
// subscriptions_events collection - event sourcing
{
  "_id": ObjectId,
  "subscription_id": UUID,
  "event_type": "SUBSCRIPTION_CREATED|BILLING_SCHEDULED|PAYMENT_CHARGED|PAYMENT_FAILED|STATE_CHANGED",
  "timestamp": ISODate,
  "event_data": {
    "subscription_id": UUID,
    "customer_id": UUID,
    "payment_amount": 49.99,
    "retry_count": 0,
    "state_before": "ACTIVE",
    "state_after": "ACTIVE"
  },
  "version": 1,
  "created_at": ISODate
}
```

---

## 7. Redis Data Structures

```
# Subscription cache (TTL 1 hour)
subscription:{subscription_id} → Hash {
    "id": UUID,
    "customer_id": UUID,
    "product_id": UUID,
    "status": "ACTIVE",
    "next_billing_date": "2026-04-15T00:00:00Z",
    "billing_frequency_days": 30,
    "subscription_price": "49.99"
}

# Billing lock (TTL 5 minutes, distributed mutual exclusion)
billing:{subscription_id}:lock → "locked"

# Billing state (tracks in-flight billing operations)
billing:{subscription_id}:state → Hash {
    "status": "PENDING|CHARGING|CHARGED|FAILED",
    "started_at": "2026-04-01T09:00:00Z",
    "payment_id": UUID,
    "retry_count": 0
}

# Idempotency cache (prevents duplicate charges)
billing:idempotency:{subscription_id}:{billing_period} → Hash {
    "status": "SUCCESS",
    "payment_id": UUID,
    "order_id": UUID,
    "charged_at": "2026-04-01T09:05:00Z"
}

# Customer subscriptions index
customer:{customer_id}:subscriptions → Set [sub_id1, sub_id2, ...]

# Pending retry queue
payment:retry:queue → Sorted Set {
    score = next_retry_timestamp,
    member = payment_id:subscription_id
}
```

---

## 8. Kafka Event Flow

### Topics & Events

```
Topic: subscriptions (Retention: 30 days)
├─ subscription.created
│  └─ {subscription_id, customer_id, product_id, frequency, price}
├─ subscription.state_changed
│  └─ {subscription_id, old_state, new_state, reason}
├─ subscription.modified
│  └─ {subscription_id, modification_type, old_value, new_value}
└─ subscription.cancelled
   └─ {subscription_id, reason, refund_info}

Topic: billing.scheduled (Retention: 7 days)
├─ billing_event
│  └─ {subscription_id, billing_period, amount, next_billing_date}
└─ [consumed by Payment Processor, Order Processor, Email Service]

Topic: payments (Retention: 90 days)
├─ payment.initiated
│  └─ {payment_id, subscription_id, amount, retry_count}
├─ payment.succeeded
│  └─ {payment_id, subscription_id, amount, transaction_id}
├─ payment.failed
│  └─ {payment_id, subscription_id, error_code, retry_count}
├─ payment.retry
│  └─ {payment_id, subscription_id, retry_count, next_retry_at}
└─ payment.suspended
   └─ {subscription_id, reason, suggested_action}

Topic: orders.recurring (Retention: 90 days)
├─ recurring_order.created
│  └─ {order_id, subscription_id, items, shipping_address}
├─ recurring_order.shipped
│  └─ {order_id, tracking_number, estimated_delivery}
└─ recurring_order.delivered
   └─ {order_id, delivered_at}

Topic: reminders (Retention: 30 days)
├─ reminder.scheduled
│  └─ {subscription_id, reminder_type: "7D_BEFORE|1D_BEFORE|BILLING_DATE", send_at}
├─ reminder.sent
│  └─ {subscription_id, reminder_type, sent_at, channel}
└─ reminder.failed
   └─ {subscription_id, reminder_type, error}

Topic: inventory (Retention: 7 days)
└─ inventory.precheck_failed
   └─ {subscription_id, sku, requested_qty, available_qty, check_date}
```

---

## 9. Implementation Code

### 9.1 Subscription Entity & Repository

```java
package com.retail.subscription.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Entity
@Table(name = "subscriptions")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Subscription {

    @Id
    @Column(columnDefinition = "UUID")
    private UUID id;

    @Column(nullable = false)
    private UUID customerId;

    @Column(nullable = false)
    private UUID productId;

    @Column(nullable = false)
    private Integer quantity;

    @Column(nullable = false)
    private Integer billingFrequencyDays;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private SubscriptionStatus status;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal subscriptionPrice;

    @Column(nullable = false)
    private Integer priceVersion;

    @Column(name = "price_locked_until")
    private LocalDateTime priceLockedUntil;

    @Column(nullable = false)
    private LocalDateTime nextBillingDate;

    @Column(name = "last_charged_date")
    private LocalDateTime lastChargedDate;

    @Column(nullable = false)
    private UUID deliveryAddressId;

    @Column(nullable = false)
    private UUID paymentMethodId;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(nullable = false)
    private LocalDateTime updatedAt;

    private LocalDateTime cancelledAt;

    private String cancellationReason;

    @PrePersist
    protected void onCreate() {
        this.id = UUID.randomUUID();
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
}

public enum SubscriptionStatus {
    ACTIVE, PAUSED, SUSPENDED, CANCELLED
}
```

### 9.2 Repository & Service Layer

```java
package com.retail.subscription.repository;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.stereotype.Repository;
import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Repository
public interface SubscriptionRepository extends JpaRepository<Subscription, UUID> {

    @Query("""
        SELECT s FROM Subscription s
        WHERE s.status = 'ACTIVE'
        AND s.nextBillingDate <= ?1
        ORDER BY s.nextBillingDate ASC
        """)
    List<Subscription> findDueForBilling(LocalDateTime now);

    List<Subscription> findByCustomerIdAndStatus(UUID customerId, SubscriptionStatus status);

    @Query("""
        SELECT COUNT(s) FROM Subscription s
        WHERE s.customerId = ?1
        AND s.productId = ?2
        AND s.status != 'CANCELLED'
        """)
    long countActiveSubscriptionsForProduct(UUID customerId, UUID productId);
}

@Repository
public interface BillingLogRepository extends JpaRepository<BillingLog, UUID> {

    List<BillingLog> findBySubscriptionIdOrderByCreatedAtDesc(UUID subscriptionId);

    @Query("""
        SELECT b FROM BillingLog b
        WHERE b.subscriptionId = ?1
        AND b.billingPeriod = ?2
        """)
    BillingLog findBySubscriptionAndBillingPeriod(UUID subscriptionId, LocalDate billingPeriod);
}
```

### 9.3 SubscriptionService

```java
package com.retail.subscription.service;

import com.retail.subscription.entity.Subscription;
import com.retail.subscription.entity.SubscriptionStatus;
import com.retail.subscription.repository.SubscriptionRepository;
import com.retail.subscription.event.SubscriptionEvent;
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class SubscriptionService {

    private final SubscriptionRepository subscriptionRepository;
    private final KafkaTemplate<String, SubscriptionEvent> kafkaTemplate;
    private final RedisTemplate<String, String> redisTemplate;
    private final MeterRegistry meterRegistry;
    private final AuditService auditService;

    public SubscriptionService(
        SubscriptionRepository subscriptionRepository,
        KafkaTemplate<String, SubscriptionEvent> kafkaTemplate,
        RedisTemplate<String, String> redisTemplate,
        MeterRegistry meterRegistry,
        AuditService auditService
    ) {
        this.subscriptionRepository = subscriptionRepository;
        this.kafkaTemplate = kafkaTemplate;
        this.redisTemplate = redisTemplate;
        this.meterRegistry = meterRegistry;
        this.auditService = auditService;
    }

    @Transactional
    public Subscription createSubscription(CreateSubscriptionRequest request, UUID customerId) {
        // Validate no duplicate subscription
        long activeCount = subscriptionRepository.countActiveSubscriptionsForProduct(
            customerId, request.productId()
        );
        if (activeCount > 0) {
            throw new DuplicateSubscriptionException(
                "Customer already has active subscription for this product"
            );
        }

        Subscription subscription = Subscription.builder()
            .customerId(customerId)
            .productId(request.productId())
            .quantity(request.quantity())
            .billingFrequencyDays(request.billingFrequencyDays())
            .status(SubscriptionStatus.ACTIVE)
            .subscriptionPrice(request.price())
            .priceVersion(1)
            .nextBillingDate(LocalDateTime.now().plusDays(request.billingFrequencyDays()))
            .deliveryAddressId(request.deliveryAddressId())
            .paymentMethodId(request.paymentMethodId())
            .build();

        subscription = subscriptionRepository.save(subscription);

        // Cache in Redis
        cacheSubscription(subscription);

        // Publish event
        SubscriptionEvent event = SubscriptionEvent.created(subscription);
        kafkaTemplate.send("subscriptions", subscription.getId().toString(), event);

        // Audit log
        auditService.logSubscriptionCreated(subscription, customerId);

        Counter.builder("subscription.created")
            .tag("product_id", request.productId().toString())
            .register(meterRegistry)
            .increment();

        log.info("Subscription created: id={}, customer={}, product={}",
            subscription.getId(), customerId, request.productId());

        return subscription;
    }

    @Transactional
    public void pauseSubscription(UUID subscriptionId, UUID customerId) {
        Subscription subscription = getSubscriptionAndValidateOwnership(subscriptionId, customerId);

        if (subscription.getStatus() != SubscriptionStatus.ACTIVE) {
            throw new InvalidStateTransitionException(
                "Can only pause ACTIVE subscriptions"
            );
        }

        subscription.setStatus(SubscriptionStatus.PAUSED);
        subscription.setUpdatedAt(LocalDateTime.now());
        subscriptionRepository.save(subscription);

        cacheSubscription(subscription);

        SubscriptionEvent event = SubscriptionEvent.stateChanged(
            subscription.getId(),
            SubscriptionStatus.ACTIVE,
            SubscriptionStatus.PAUSED,
            "Customer-initiated pause"
        );
        kafkaTemplate.send("subscriptions", subscriptionId.toString(), event);

        auditService.logStateChange(subscription, SubscriptionStatus.ACTIVE, SubscriptionStatus.PAUSED);

        Counter.builder("subscription.paused").register(meterRegistry).increment();
        log.info("Subscription paused: id={}", subscriptionId);
    }

    @Transactional
    public void resumeSubscription(UUID subscriptionId, UUID customerId) {
        Subscription subscription = getSubscriptionAndValidateOwnership(subscriptionId, customerId);

        if (subscription.getStatus() != SubscriptionStatus.PAUSED) {
            throw new InvalidStateTransitionException(
                "Can only resume PAUSED subscriptions"
            );
        }

        subscription.setStatus(SubscriptionStatus.ACTIVE);
        subscription.setNextBillingDate(LocalDateTime.now().plusDays(subscription.getBillingFrequencyDays()));
        subscription.setUpdatedAt(LocalDateTime.now());
        subscriptionRepository.save(subscription);

        cacheSubscription(subscription);

        SubscriptionEvent event = SubscriptionEvent.stateChanged(
            subscription.getId(),
            SubscriptionStatus.PAUSED,
            SubscriptionStatus.ACTIVE,
            "Customer-initiated resume"
        );
        kafkaTemplate.send("subscriptions", subscriptionId.toString(), event);

        auditService.logStateChange(subscription, SubscriptionStatus.PAUSED, SubscriptionStatus.ACTIVE);

        Counter.builder("subscription.resumed").register(meterRegistry).increment();
        log.info("Subscription resumed: id={}", subscriptionId);
    }

    @Transactional
    public void changeFrequency(UUID subscriptionId, UUID customerId, int newFrequencyDays) {
        Subscription subscription = getSubscriptionAndValidateOwnership(subscriptionId, customerId);

        if (newFrequencyDays < 7) {
            throw new InvalidFrequencyException("Frequency must be at least 7 days");
        }

        int oldFrequency = subscription.getBillingFrequencyDays();
        subscription.setBillingFrequencyDays(newFrequencyDays);
        subscription.setUpdatedAt(LocalDateTime.now());
        subscriptionRepository.save(subscription);

        cacheSubscription(subscription);

        SubscriptionEvent event = SubscriptionEvent.modified(
            subscription.getId(),
            "FREQUENCY_CHANGE",
            String.valueOf(oldFrequency),
            String.valueOf(newFrequencyDays)
        );
        kafkaTemplate.send("subscriptions", subscriptionId.toString(), event);

        auditService.logModification(subscription, "FREQUENCY_CHANGE", oldFrequency, newFrequencyDays);
        log.info("Subscription frequency changed: id={}, oldFreq={}, newFreq={}",
            subscriptionId, oldFrequency, newFrequencyDays);
    }

    @Transactional
    public void cancelSubscription(UUID subscriptionId, UUID customerId, String reason) {
        Subscription subscription = getSubscriptionAndValidateOwnership(subscriptionId, customerId);

        if (subscription.getStatus() == SubscriptionStatus.CANCELLED) {
            throw new InvalidStateTransitionException("Already cancelled");
        }

        SubscriptionStatus oldStatus = subscription.getStatus();
        subscription.setStatus(SubscriptionStatus.CANCELLED);
        subscription.setCancelledAt(LocalDateTime.now());
        subscription.setCancellationReason(reason);
        subscription.setUpdatedAt(LocalDateTime.now());
        subscriptionRepository.save(subscription);

        redisTemplate.delete("subscription:" + subscriptionId);

        SubscriptionEvent event = SubscriptionEvent.cancelled(
            subscription.getId(),
            reason
        );
        kafkaTemplate.send("subscriptions", subscriptionId.toString(), event);

        auditService.logCancellation(subscription, oldStatus, reason);
        Counter.builder("subscription.cancelled").register(meterRegistry).increment();
        log.info("Subscription cancelled: id={}, reason={}", subscriptionId, reason);
    }

    public Subscription getSubscription(UUID subscriptionId) {
        // Try cache first
        String cached = redisTemplate.opsForValue().get("subscription:" + subscriptionId);
        if (cached != null) {
            return deserializeSubscription(cached);
        }

        Subscription subscription = subscriptionRepository.findById(subscriptionId)
            .orElseThrow(() -> new SubscriptionNotFoundException(subscriptionId));

        cacheSubscription(subscription);
        return subscription;
    }

    private Subscription getSubscriptionAndValidateOwnership(UUID subscriptionId, UUID customerId) {
        Subscription subscription = getSubscription(subscriptionId);
        if (!subscription.getCustomerId().equals(customerId)) {
            throw new UnauthorizedException("Customer does not own this subscription");
        }
        return subscription;
    }

    private void cacheSubscription(Subscription subscription) {
        String key = "subscription:" + subscription.getId();
        String serialized = serializeSubscription(subscription);
        redisTemplate.opsForValue().set(key, serialized, 1, TimeUnit.HOURS);
    }

    // Serialization helpers (use ObjectMapper in real code)
    private String serializeSubscription(Subscription s) { /* ... */ return ""; }
    private Subscription deserializeSubscription(String json) { /* ... */ return null; }
}
```

### 9.4 Billing Scheduler (Quartz)

```java
package com.retail.subscription.scheduler;

import com.retail.subscription.entity.Subscription;
import com.retail.subscription.entity.SubscriptionStatus;
import com.retail.subscription.repository.SubscriptionRepository;
import com.retail.subscription.event.BillingEvent;
import lombok.extern.slf4j.Slf4j;
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.List;

@Component
@Slf4j
public class BillingSchedulerJob implements Job {

    private final SubscriptionRepository subscriptionRepository;
    private final KafkaTemplate<String, BillingEvent> kafkaTemplate;

    public BillingSchedulerJob(
        SubscriptionRepository subscriptionRepository,
        KafkaTemplate<String, BillingEvent> kafkaTemplate
    ) {
        this.subscriptionRepository = subscriptionRepository;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Override
    public void execute(JobExecutionContext context) {
        log.info("Starting billing scheduler job at {}", LocalDateTime.now());

        LocalDateTime now = LocalDateTime.now();
        List<Subscription> dueBillings = subscriptionRepository.findDueForBilling(now);

        log.info("Found {} subscriptions due for billing", dueBillings.size());

        for (Subscription subscription : dueBillings) {
            try {
                BillingEvent event = BillingEvent.builder()
                    .subscriptionId(subscription.getId())
                    .customerId(subscription.getCustomerId())
                    .productId(subscription.getProductId())
                    .billingPeriod(now.toLocalDate())
                    .chargeAmount(subscription.getSubscriptionPrice())
                    .billingDate(now)
                    .build();

                // Publish to Kafka for idempotent processing
                kafkaTemplate.send("billing.scheduled", subscription.getId().toString(), event);

                log.info("Published billing event for subscription: {}", subscription.getId());

            } catch (Exception e) {
                log.error("Error publishing billing event for subscription: {}",
                    subscription.getId(), e);
                // Continue processing other subscriptions
            }
        }

        log.info("Billing scheduler job completed at {}", LocalDateTime.now());
    }
}

// Quartz configuration
@Configuration
public class QuartzConfig {

    @Bean
    public JobDetail billingSchedulerJobDetail() {
        return JobBuilder.newJob(BillingSchedulerJob.class)
            .withIdentity("billingSchedulerJob", "billingSchedulerGroup")
            .storeDurably()
            .build();
    }

    @Bean
    public Trigger billingSchedulerTrigger(JobDetail billingSchedulerJobDetail) {
        return TriggerBuilder.newTrigger()
            .forJob(billingSchedulerJobDetail)
            .withIdentity("billingSchedulerTrigger", "billingSchedulerGroup")
            .withSchedule(CronScheduleBuilder.cronSchedule("0 0 */6 * * ?")) // Every 6 hours
            .build();
    }
}
```

### 9.5 Payment Processor & Retry Handler

```java
package com.retail.subscription.payment;

import com.retail.subscription.entity.BillingLog;
import com.retail.subscription.entity.Subscription;
import com.retail.subscription.entity.SubscriptionStatus;
import com.retail.subscription.event.PaymentEvent;
import com.retail.subscription.repository.BillingLogRepository;
import com.retail.subscription.repository.SubscriptionRepository;
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class PaymentProcessorService {

    private final PaymentGatewayClient paymentGateway;
    private final SubscriptionRepository subscriptionRepository;
    private final BillingLogRepository billingLogRepository;
    private final KafkaTemplate<String, PaymentEvent> kafkaTemplate;
    private final RedisTemplate<String, String> redisTemplate;
    private final MeterRegistry meterRegistry;

    private static final int MAX_RETRY_ATTEMPTS = 3;
    private static final int RETRY_DELAY_SECONDS = 300; // 5 minutes

    public PaymentProcessorService(
        PaymentGatewayClient paymentGateway,
        SubscriptionRepository subscriptionRepository,
        BillingLogRepository billingLogRepository,
        KafkaTemplate<String, PaymentEvent> kafkaTemplate,
        RedisTemplate<String, String> redisTemplate,
        MeterRegistry meterRegistry
    ) {
        this.paymentGateway = paymentGateway;
        this.subscriptionRepository = subscriptionRepository;
        this.billingLogRepository = billingLogRepository;
        this.kafkaTemplate = kafkaTemplate;
        this.redisTemplate = redisTemplate;
        this.meterRegistry = meterRegistry;
    }

    @KafkaListener(topics = "billing.scheduled", groupId = "payment-processor-group")
    @Transactional
    public void processBillingEvent(BillingEvent event) {
        log.info("Processing billing event: subscription={}, period={}",
            event.subscriptionId(), event.billingPeriod());

        // Acquire distributed lock
        String lockKey = "billing:" + event.subscriptionId() + ":lock";
        Boolean lockAcquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "locked", 5, TimeUnit.MINUTES);

        if (!Boolean.TRUE.equals(lockAcquired)) {
            log.warn("Could not acquire lock for subscription: {}", event.subscriptionId());
            return;
        }

        try {
            // Check idempotency
            String idempotencyKey = "billing:idempotency:" + event.subscriptionId()
                + ":" + event.billingPeriod();
            String idempotencyValue = redisTemplate.opsForValue().get(idempotencyKey);
            if (idempotencyValue != null) {
                log.info("Billing already processed (idempotent): {}", idempotencyKey);
                return;
            }

            Subscription subscription = subscriptionRepository.findById(event.subscriptionId())
                .orElseThrow();

            // Create billing log entry
            BillingLog billingLog = BillingLog.builder()
                .subscriptionId(event.subscriptionId())
                .billingPeriod(event.billingPeriod())
                .chargeAmount(event.chargeAmount())
                .paymentStatus(PaymentStatus.PENDING)
                .retryCount(0)
                .createdAt(LocalDateTime.now())
                .build();

            billingLog = billingLogRepository.save(billingLog);

            // Charge payment
            boolean paymentSuccess = chargePayment(subscription, event.chargeAmount(), billingLog);

            if (paymentSuccess) {
                billingLog.setPaymentStatus(PaymentStatus.SUCCESS);
                billingLog.setChargedAt(LocalDateTime.now());
                billingLogRepository.save(billingLog);

                // Update subscription
                subscription.setLastChargedDate(LocalDateTime.now());
                subscription.setNextBillingDate(
                    LocalDateTime.now().plusDays(subscription.getBillingFrequencyDays())
                );
                subscriptionRepository.save(subscription);

                // Record idempotency success
                redisTemplate.opsForValue().set(
                    idempotencyKey,
                    "success:" + billingLog.getPaymentId(),
                    24,
                    TimeUnit.HOURS
                );

                // Publish success event
                PaymentEvent paymentEvent = PaymentEvent.success(
                    UUID.randomUUID(),
                    event.subscriptionId(),
                    event.chargeAmount()
                );
                kafkaTemplate.send("payments", event.subscriptionId().toString(), paymentEvent);

                // Trigger order creation
                kafkaTemplate.send("orders.recurring",
                    "create-order:" + event.subscriptionId(),
                    new OrderCreationEvent(event.subscriptionId(), event.chargeAmount()));

                Counter.builder("payment.success")
                    .tag("subscription_id", event.subscriptionId().toString())
                    .register(meterRegistry)
                    .increment();

                log.info("Payment successful: subscription={}, amount={}",
                    event.subscriptionId(), event.chargeAmount());

            } else {
                handlePaymentFailure(billingLog, subscription, event, 0);
            }

        } catch (Exception e) {
            log.error("Error processing billing event", e);
            Counter.builder("payment.error")
                .register(meterRegistry)
                .increment();
        } finally {
            redisTemplate.delete(lockKey);
        }
    }

    private boolean chargePayment(Subscription subscription, BigDecimal amount, BillingLog billingLog) {
        try {
            PaymentRequest paymentRequest = PaymentRequest.builder()
                .paymentMethodId(subscription.getPaymentMethodId())
                .customerId(subscription.getCustomerId())
                .amount(amount)
                .currency("USD")
                .metadata("subscription_id", subscription.getId().toString())
                .build();

            PaymentResponse response = paymentGateway.charge(paymentRequest);

            if (response.isSuccess()) {
                billingLog.setPaymentId(response.transactionId());
                return true;
            } else {
                billingLog.setErrorMessage(response.errorMessage());
                return false;
            }

        } catch (Exception e) {
            log.error("Payment gateway error", e);
            billingLog.setErrorMessage(e.getMessage());
            return false;
        }
    }

    @Transactional
    private void handlePaymentFailure(BillingLog billingLog, Subscription subscription,
                                     BillingEvent event, int retryCount) {
        billingLog.setPaymentStatus(PaymentStatus.FAILED);
        billingLog.setRetryCount(retryCount);

        if (retryCount < MAX_RETRY_ATTEMPTS) {
            billingLog.setPaymentStatus(PaymentStatus.RETRY);
            LocalDateTime nextRetry = LocalDateTime.now().plusSeconds(
                RETRY_DELAY_SECONDS * (long) Math.pow(2, retryCount)
            );
            billingLog.setNextRetryAt(nextRetry);

            billingLogRepository.save(billingLog);

            // Publish retry event
            PaymentEvent retryEvent = PaymentEvent.retry(
                billingLog.getPaymentId(),
                event.subscriptionId(),
                event.chargeAmount(),
                retryCount + 1,
                nextRetry
            );
            kafkaTemplate.send("payments", event.subscriptionId().toString(), retryEvent);

            // Add to retry queue
            redisTemplate.opsForZSet().add(
                "payment:retry:queue",
                billingLog.getPaymentId() + ":" + event.subscriptionId(),
                nextRetry.atZone(ZoneId.systemDefault()).toInstant().toEpochMilli() / 1000.0
            );

            Counter.builder("payment.retry")
                .tag("retry_count", String.valueOf(retryCount + 1))
                .register(meterRegistry)
                .increment();

            log.info("Payment retry scheduled: subscription={}, retryCount={}, nextRetry={}",
                event.subscriptionId(), retryCount + 1, nextRetry);

        } else {
            // Max retries exceeded - suspend subscription
            billingLogRepository.save(billingLog);

            subscription.setStatus(SubscriptionStatus.SUSPENDED);
            subscription.setUpdatedAt(LocalDateTime.now());
            subscriptionRepository.save(subscription);

            PaymentEvent suspendEvent = PaymentEvent.suspended(
                event.subscriptionId(),
                "Max payment retry attempts exceeded"
            );
            kafkaTemplate.send("payments", event.subscriptionId().toString(), suspendEvent);

            Counter.builder("subscription.suspended")
                .tag("reason", "payment_failure")
                .register(meterRegistry)
                .increment();

            log.error("Subscription suspended due to payment failure: subscription={}",
                event.subscriptionId());
        }
    }

    @KafkaListener(topics = "payment.retry", groupId = "payment-retry-group")
    public void processPaymentRetry(PaymentRetryEvent event) {
        log.info("Processing payment retry: subscription={}, retryCount={}",
            event.subscriptionId(), event.retryCount());

        // Similar to processBillingEvent but for retries
        // Implementation follows same pattern
    }
}
```

### 9.6 Reminder Email Service

```java
package com.retail.subscription.reminder;

import com.retail.subscription.entity.Subscription;
import com.retail.subscription.repository.SubscriptionRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.List;

@Service
@Slf4j
public class ReminderEmailService {

    private final SubscriptionRepository subscriptionRepository;
    private final EmailService emailService;
    private final KafkaTemplate<String, ReminderEvent> kafkaTemplate;

    public ReminderEmailService(
        SubscriptionRepository subscriptionRepository,
        EmailService emailService,
        KafkaTemplate<String, ReminderEvent> kafkaTemplate
    ) {
        this.subscriptionRepository = subscriptionRepository;
        this.emailService = emailService;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Scheduled(cron = "0 0 9 * * ?") // Daily at 9 AM
    public void sendReminderEmails() {
        log.info("Starting reminder email job");

        LocalDateTime now = LocalDateTime.now();

        // 7 days before
        sendRemindersForDate(now.plusDays(7), ReminderType.SEVEN_DAYS_BEFORE);

        // 1 day before
        sendRemindersForDate(now.plusDays(1), ReminderType.ONE_DAY_BEFORE);

        // On billing date
        sendRemindersForDate(now, ReminderType.BILLING_DATE);
    }

    private void sendRemindersForDate(LocalDateTime targetDate, ReminderType reminderType) {
        LocalDateTime startOfDay = targetDate.withHour(0).withMinute(0).withSecond(0);
        LocalDateTime endOfDay = targetDate.withHour(23).withMinute(59).withSecond(59);

        // This would need a custom query to find subscriptions with next_billing_date in range
        List<Subscription> subscriptions = subscriptionRepository.findDueForBilling(endOfDay);

        for (Subscription subscription : subscriptions) {
            try {
                ReminderEvent event = ReminderEvent.builder()
                    .subscriptionId(subscription.getId())
                    .customerId(subscription.getCustomerId())
                    .reminderType(reminderType)
                    .billingDate(startOfDay)
                    .chargeAmount(subscription.getSubscriptionPrice())
                    .build();

                kafkaTemplate.send("reminders", subscription.getId().toString(), event);

                log.info("Published reminder event: subscription={}, type={}",
                    subscription.getId(), reminderType);

            } catch (Exception e) {
                log.error("Error publishing reminder for subscription: {}",
                    subscription.getId(), e);
            }
        }
    }
}

@Service
@Slf4j
public class EmailReminderProcessor {

    private final EmailService emailService;
    private final CustomerService customerService;

    @KafkaListener(topics = "reminders", groupId = "email-reminder-group")
    public void processReminder(ReminderEvent event) {
        log.info("Processing reminder: subscription={}, type={}",
            event.subscriptionId(), event.reminderType());

        try {
            Customer customer = customerService.getCustomer(event.customerId());

            String emailSubject = switch (event.reminderType()) {
                case SEVEN_DAYS_BEFORE -> "Upcoming Subscription Delivery";
                case ONE_DAY_BEFORE -> "Your Subscription Charges Tomorrow";
                case BILLING_DATE -> "Subscription Charged Today";
            };

            String emailBody = buildReminderEmailBody(event, customer);

            emailService.sendEmail(
                customer.getEmail(),
                emailSubject,
                emailBody,
                EmailTemplate.SUBSCRIPTION_REMINDER
            );

            log.info("Reminder email sent: subscription={}, customer={}",
                event.subscriptionId(), event.customerId());

        } catch (Exception e) {
            log.error("Error processing reminder", e);
        }
    }

    private String buildReminderEmailBody(ReminderEvent event, Customer customer) {
        return """
            Hi %s,

            %s

            Subscription Details:
            - Charge Amount: $%s
            - Next Billing Date: %s

            Need help? Contact support@company.com
            """.formatted(
                customer.getFirstName(),
                getReminderMessage(event.reminderType()),
                event.chargeAmount(),
                event.billingDate()
            );
    }

    private String getReminderMessage(ReminderType reminderType) {
        return switch (reminderType) {
            case SEVEN_DAYS_BEFORE -> "Your subscription will be charged in 7 days.";
            case ONE_DAY_BEFORE -> "Your subscription will be charged tomorrow.";
            case BILLING_DATE -> "Your subscription has been charged today.";
        };
    }
}
```

---

## 10. Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| Payment gateway timeout | Billing not processed | Exponential backoff retry, Kafka event replay, idempotency key |
| Scheduler server crash | Scheduled jobs miss | Quartz in cluster mode, job history in DB, resume on restart |
| Order service down | Orders not created | Kafka event buffered, dead-letter topic, manual retry |
| Duplicate charges due to network split | Customer overcharged | Unique constraint on (subscription_id, billing_period), distributed lock |
| Inventory becomes unavailable | Unfulfillable order | 24h pre-check, notify customer, auto-delay billing 7 days |
| Email service down | No reminders sent | Async with SQS/Kafka, retry with exponential backoff |
| Subscription cache stale | Wrong state | TTL 1 hour, write-through cache, versioning |
| Customer modifies subscription during billing | Race condition | Distributed lock on subscription, atomic operations |
| Redis goes down | Can't acquire lock | Fallback to database-only locking (slower but safer) |

---

## 11. Scaling Strategy

### Horizontal Scaling
- **Payment Processor:** Stateless Kafka consumer group; scale horizontally
- **Email Service:** Independent Kafka consumer; linear scaling
- **Subscription API:** Spring Boot behind load balancer; scale based on CPU
- **Quartz Scheduler:** Clustered mode with distributed lock; safe horizontal scaling

### Database Optimization
- Partition billing logs by `billing_period` (monthly)
- Index on `(subscription_id, billing_period)` for fast lookups
- Archive old audit logs to separate table
- Read replicas for reporting queries

### Redis Caching
- Subscription cache: 1 hour TTL
- Billing state cache: ephemeral, no persistence needed
- Idempotency cache: 24 hours TTL
- Distributed locks: 5 minute TTL

### Kafka Topic Partitioning
- `subscriptions`: partition by `customer_id` (even distribution)
- `billing.scheduled`: partition by `subscription_id` (ensure ordering per subscription)
- `payments`: partition by `subscription_id`
- `reminders`: partition by `customer_id`

---

## 12. Monitoring & Observability

### Key Metrics
```java
// Subscription metrics
subscription.created (counter)
subscription.paused (counter)
subscription.resumed (counter)
subscription.cancelled (counter)
subscription.suspended (counter, tag: reason)
subscription.modification (counter, tag: modification_type)

// Payment metrics
payment.success (counter)
payment.failed (counter, tag: error_code)
payment.retry (counter, tag: retry_count)
payment.latency (timer)
payment.charge_amount (distribution)

// Billing metrics
billing.scheduled (counter)
billing.processed (counter)
billing.duration (timer)
billing.queue_depth (gauge)

// Email metrics
email.reminder.sent (counter, tag: reminder_type)
email.reminder.failed (counter)
email.send_latency (timer)

// Cache metrics
cache.subscription.hit_rate (gauge)
cache.lock.acquisition_time (timer)
```

### Alerting Rules
```
- If payment.failed rate > 5% in 5 min: Page oncall
- If billing.queue_depth > 10K: Scale payment processors
- If email.reminder.failed rate > 2%: Page oncall
- If cache.hit_rate < 80%: Investigate cache invalidation
- If any Quartz node down: Alert ops
- If payment.latency p99 > 5s: Investigate payment gateway
```

### Distributed Tracing
```java
@Aspect
@Component
public class SubscriptionTracingAspect {
    private final Tracer tracer;

    @Around("execution(* com.retail.subscription.service.*.*(..))")
    public Object traceSubscriptionOperations(ProceedingJoinPoint joinPoint) throws Throwable {
        Span span = tracer.startSpan("subscription.operation")
            .setTag("method", joinPoint.getSignature().getName());
        try {
            return joinPoint.proceed();
        } finally {
            span.finish();
        }
    }
}
```

---

## 13. Summary Cheat Sheet

### Architecture Patterns
- **Saga Pattern:** Distributed transactions across services (billing → order creation)
- **Event Sourcing:** All subscription events in MongoDB for audit trail
- **CQRS:** Separate read model in Redis for fast queries
- **Idempotency:** Unique keys prevent duplicates on retries

### Key Components
| Component | Technology | Purpose |
|-----------|-----------|---------|
| Subscription API | Spring REST | CRUD operations |
| State Manager | Spring Service | Validation & transitions |
| Billing Scheduler | Quartz Cluster | Triggers scheduled jobs |
| Payment Processor | Kafka Consumer | Processes charges |
| Email Reminder | Kafka Consumer | Sends notifications |
| Audit Log | MongoDB | Event sourcing |
| Subscription Cache | Redis Hash | Fast read access |
| Distributed Lock | Redis String | Mutual exclusion |
| Idempotency | Redis + DB | Prevents duplicates |

### Database Tables
```
subscriptions (UUID, customer_id, product_id, status, next_billing_date, ...)
billing_logs (UUID, subscription_id, billing_period, status, retry_count, ...)
subscription_modifications (UUID, subscription_id, modification_type, effective_from, ...)
audit_logs (UUID, subscription_id, event_type, old_state, new_state, ...)
```

### Kafka Topics
```
subscriptions → events
billing.scheduled → events
payments → events
orders.recurring → events
reminders → events
```

### Redis Keys
```
subscription:{id} → subscription data
billing:{id}:lock → distributed lock
billing:{id}:state → billing state
billing:idempotency:{id}:{period} → idempotency check
payment:retry:queue → sorted set of pending retries
customer:{id}:subscriptions → set of subscription IDs
```

### Critical Design Decisions
1. **Idempotency first:** Every operation has unique key to prevent duplicates
2. **Distributed lock:** Serialize billing operations per subscription
3. **Async all the way:** Kafka events decouple services
4. **Cache hot data:** Subscriptions, billing state in Redis
5. **Audit everything:** MongoDB event log for compliance
6. **Fail gracefully:** Suspend subscription on payment failure, notify customer
7. **Price lock:** Existing subscribers protected from mid-cycle price changes

