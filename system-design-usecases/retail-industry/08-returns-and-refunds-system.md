---
title: Returns and Refunds System
layout: default
---

# Returns and Refunds System — Deep Dive Design

> **Scenario:** Design a returns management system where:
> - Customer initiates return within 30 days
> - System generates return label
> - Customer ships item back
> - Warehouse receives and inspects item
> - If approved → Issue refund, restock inventory
> - If rejected → Ship back to customer, charge restocking fee
> - Support partial returns (ordered 3 items, return 2)
> - Handle different return windows for different product categories
>
> **Tech Stack:** Java 17, Spring Boot 3, PostgreSQL, MongoDB, Redis, Kafka

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
10. [Failure Scenarios & Mitigations](#failure-scenarios--mitigations)
11. [Scaling Strategy](#scaling-strategy)
12. [Monitoring & Observability](#monitoring--observability)
13. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Requirements & Constraints

### Functional Requirements
- Accept returns within category-specific windows (30, 60, 90 days)
- Generate return labels (different per carrier/destination)
- Track return item status end-to-end
- Partial returns at line-item level with pro-rata refund calculation
- Approve/reject based on condition inspection at warehouse
- Support multiple refund methods (original card, store credit, different card)
- Charge restocking fees for non-defective items (10-20%)
- Fraud detection (serial returners, abuse patterns)
- Compensation logic if reverse logistics fails
- Integration with payment processors for refunds

### Non-Functional Requirements
- Return state transitions within 5 minutes (async)
- Refund processing SLA: 3-5 business days (payment processor dependent)
- Availability: 99.5% (async workflows tolerate eventual consistency)
- Data durability: Immutable event log of all return state changes
- Scalability: Handle 5% return rate (100K returns/day avg)
- Support distributed saga pattern for refund coordination

---

## Capacity Estimation

| Metric | Calculation | Value |
|--------|-----------|-------|
| Daily orders | Typical e-commerce | 2M orders |
| Return rate | Industry average | 5% |
| Daily returns | 2M × 5% | 100,000 |
| Partial returns (% of returns) | Return 1-2 items of multi-item order | 30% |
| Return state changes/return | INITIATED → LABEL_GEN → IN_TRANSIT → RECEIVED → INSPECTING → APPROVED → REFUNDED | ~8 |
| Total events/day | 100K returns × 8 transitions | 800K events |
| Return records (2 years) | 100K/day × 365 × 2 | 73M records |
| Return label images/PDFs | 100K/day × 2 (pickup + delivery) | 200K images |
| Refund compensation (failed) | ~1% failure rate | 1,000 refunds/day |
| Revenue tied up in refunds (7 days) | 100K × avg order value $100 × 7 days | $70M |
| Fraud flags per month | ~0.5% high-risk returns | 15,000 flags |
| Payment processor webhooks | 100K returns × 3 events | 300K webhooks/day |
| Kafka topics | return-events, refund-saga, fraud-alerts, label-generated | 4 topics |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│              Customer Portal                        │
│        (Initiate Return Request)                    │
└────────────────────┬────────────────────────────────┘
                     │
        ┌────────────▼─────────────┐
        │   ReturnService          │
        │   (initiation, eligibility)
        └────────────┬──────────────┘
                     │
    ┌────────────────┼────────────────┐
    │                │                │
┌───▼────┐  ┌────────▼──────┐  ┌──────▼──────┐
│ Fraud  │  │ Return Label  │  │ State       │
│ Check  │  │ Generator     │  │ Machine     │
└───┬────┘  └────────┬──────┘  └──────┬──────┘
    │               │                 │
    └───────────────┼─────────────────┘
                    │
        ┌───────────▼──────────┐
        │  Kafka: return-events│
        │  (state transitions) │
        └───────┬──────────────┘
                │
    ┌───────────┼───────────┐
    │           │           │
┌───▼──────┐  ┌─▼────────┐ │
│ Warehouse│  │ Refund   │ │
│ Receiver │  │ Saga     │ │
└───┬──────┘  │Orchestrator
    │         └─┬────────┘
    │           │
┌───▼──────────▼──────────────┐
│  PostgreSQL                 │
│  (return, refund, audit log)│
└──────────────┬──────────────┘
               │
    ┌──────────┼──────────┐
    │          │          │
┌───▼──┐ ┌────▼────┐ ┌───▼──┐
│Redis │ │MongoDB  │ │Kafka │
│(cache│ │(rules,  │ │(saga │
│status│ │fraud)   │ │track)│
└──────┘ └─────────┘ └──────┘
```

---

## Core Design Questions Answered

### 1. What microservices are involved?

| Service | Responsibility | Tech |
|---------|-----------------|------|
| **ReturnService** | Initiate returns, eligibility check | Spring Boot, PostgreSQL |
| **ReturnLabelGenerator** | Create return labels (carrier integration) | Spring Boot, 3rd-party APIs |
| **WarehouseReceiver** | Track inbound RMA, item inspection | Spring Boot, Kafka Consumer |
| **FraudCheckService** | Detect abuse patterns, flag serial returners | Spring Boot, MongoDB, ML |
| **RefundSagaOrchestrator** | Coordinate refund payment + inventory restock | Spring Boot, Kafka, PostgreSQL |
| **InventoryRestockProducer** | Publish restock events after approval | Kafka Producer |
| **PaymentProcessor** | Interface with Stripe/PayPal for refunds | Spring Boot, REST clients |
| **ReturnAnalyticsService** | Track return rates, fraud metrics | Kafka Consumer, PostgreSQL |

### 2. How do you track return status through the workflow?

**State Machine (Immutable Event Log):**
```
INITIATED
  ↓
LABEL_GENERATED (return label PDF created)
  ↓
IN_TRANSIT (customer ships, tracking received)
  ↓
RECEIVED (at warehouse RMA bay)
  ↓
INSPECTING (quality check, condition assessment)
  ↓
APPROVED ←→ REJECTED
  ↓
REFUNDED (or RESTOCKING_FEE_CHARGED if rejected)
```

**Storage:** Each state transition = event in PostgreSQL + Kafka topic

**Recovery:** If any step fails, captured in Dead Letter Queue (DLQ) for manual review

### 3. How do you handle refunds to different payment methods?

**Refund Processor Challenges:**
- Original card may be expired/canceled
- Different payment methods (card, PayPal, gift card) need different handling
- Payment processors have daily limits
- Some refunds are instant; others take 5-7 days

**Strategy:**
1. **Optimistic:** Try refund to original payment method
2. **Fallback 1:** If card declined → ask customer for alternate card
3. **Fallback 2:** If customer unavailable → store credit (immediate)
4. **Fallback 3:** If store credit depleted → manual payment review

**Kafka Flow:**
```
refund-approved event
  → PaymentRefundService
    → Try original method (async)
      → Success: publish refund-completed
      → Decline: publish refund-retry-needed
        → (wait for customer response)
        → Apply store credit fallback
```

### 4. How do you prevent return fraud?

**Fraud Detection Layers:**

| Signal | Action | Threshold |
|--------|--------|-----------|
| Serial returner (>50% return rate) | Flag, review manually | 50% |
| Expensive item returned post-use | Flag, inspect condition | items > $1000 |
| Multiple returns same customer same day | Delay processing | > 3 returns |
| Return reason mismatch (says broken but photo clean) | Reject, assess damage | Manual |
| Return initiated outside warranty but claims defect | Investigate, escalate | Case-by-case |

**Implementation:**
```java
// Check serial returner pattern
int totalReturns = returnRepo.countByUserId(userId);
int approvedReturns = returnRepo.countByUserIdAndStatusApproved(userId);
double returnRate = approvedReturns / (double) totalReturns;

if (returnRate > 0.50) {
    fraudCheckService.flagReturn(return, "SERIAL_RETURNER", returnRate);
}
```

### 5. How do you coordinate inventory restocking with warehouse system?

**Async Coordination (Saga Pattern):**

1. **Return Approved** → Publish event
2. **Inventory Service** listens → reserves stock
3. **Warehouse System** listens → updates bin location, stock count
4. **If Inventory fails** → DLQ → retry or manual reconciliation
5. **Compensation** → If restock fails but refund succeeded → flag for audit

**Idempotent Operations:**
- Restock request includes return-id (idempotency key)
- Warehouse deduplicates by return-id
- If retry occurs → no double-counting

### 6. What happens if the original order service is down when processing return?

**Resilience Strategy:**

| Order Service Status | Action |
|---------------------|--------|
| **Available** | Fetch order details in real-time |
| **Down** | Use cached order (Redis, 24h TTL) |
| **Cache miss** | Reject with "order not found", notify customer |
| **Slow** | Timeout after 2s, use cache if available |

**Code:**
```java
Order order = getOrderWithFallback(orderId);

private Order getOrderWithFallback(Long orderId) {
    try {
        return orderService.getOrder(orderId); // timeout: 2s
    } catch (TimeoutException e) {
        // Try cache
        Order cached = redisTemplate.opsForValue()
            .get("order:" + orderId);
        if (cached != null) {
            log.warn("Order service timeout, using cached order: {}", orderId);
            return cached;
        }
        throw new OrderNotAvailableException();
    }
}
```

---

## Microservices Breakdown

### ReturnService (Core Domain)
```
POST /api/returns/initiate
{
  "orderId": 12345,
  "lineItemIds": [1, 2],  // partial return
  "reason": "WRONG_SIZE|DEFECTIVE|CHANGED_MIND",
  "notes": "Size ran small"
}

Response: 201 Created
{
  "returnId": "RET-2024-001234",
  "status": "INITIATED",
  "eligibilityCheck": {
    "withinWindow": true,
    "daysUntilExpiry": 15,
    "category": "APPAREL"
  },
  "estimatedRefund": 89.99,
  "nextSteps": "Awaiting label generation"
}
```

### ReturnLabelGenerator
```
POST /api/returns/{returnId}/labels/generate
Response: 200 OK
{
  "returnLabel": "RET-2024-001234.pdf",
  "carrier": "UPS",
  "trackingNumber": "1Z999AA...",
  "pickupAddress": {...},
  "dropoffAddress": {...},
  "expiresAt": "2024-02-15"
}
```

### WarehouseReceiver
```
POST /api/returns/{returnId}/receive
{
  "trackingNumber": "1Z999AA...",
  "itemsReceived": [
    {
      "lineItemId": 1,
      "condition": "LIKE_NEW|GOOD|FAIR|POOR",
      "defects": ["missing_tag"],
      "notes": "Condition excellent"
    }
  ]
}
```

### RefundSagaOrchestrator
```
State machine:
- ApprovedReturn event
  - Step 1: Refund payment (async)
  - Step 2: Restock inventory (async)
  - Step 3: Update return status (async)
- Compensation (if any step fails):
  - Refund completed → don't undo
  - Restock failed → retry + alert
```

---

## Database Design

### PostgreSQL Schema

```sql
-- Returns main table
CREATE TABLE returns (
    id BIGSERIAL PRIMARY KEY,
    return_id VARCHAR(50) UNIQUE NOT NULL,
    order_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'INITIATED',
    -- INITIATED, LABEL_GENERATED, IN_TRANSIT, RECEIVED, INSPECTING, APPROVED, REJECTED, REFUNDED
    return_reason VARCHAR(100) NOT NULL,
    return_notes TEXT,
    return_initiated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    label_generated_at TIMESTAMP,
    received_at_warehouse TIMESTAMP,
    approved_at TIMESTAMP,
    refunded_at TIMESTAMP,
    refund_amount DECIMAL(10,2),
    restocking_fee DECIMAL(10,2),
    net_refund DECIMAL(10,2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_user_id (user_id),
    INDEX idx_order_id (order_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);

-- Return line items (partial returns)
CREATE TABLE return_line_items (
    id BIGSERIAL PRIMARY KEY,
    return_id BIGINT NOT NULL,
    order_line_item_id BIGINT NOT NULL,
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    item_condition VARCHAR(50), -- LIKE_NEW, GOOD, FAIR, POOR
    defects TEXT[], -- array of defect descriptions
    inspection_notes TEXT,
    approved BOOLEAN,
    restocking_fee DECIMAL(10,2),
    refund_amount DECIMAL(10,2), -- pro-rata
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (return_id) REFERENCES returns(id),
    INDEX idx_return_id (return_id)
);

-- Return labels (shipping labels)
CREATE TABLE return_labels (
    id BIGSERIAL PRIMARY KEY,
    return_id BIGINT NOT NULL,
    carrier VARCHAR(50) NOT NULL, -- UPS, FedEx, USPS
    tracking_number VARCHAR(100) UNIQUE,
    label_url VARCHAR(500),
    pickup_address JSONB,
    dropoff_address JSONB,
    generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    FOREIGN KEY (return_id) REFERENCES returns(id),
    UNIQUE(return_id, carrier)
);

-- Refund saga tracking (distributed transaction)
CREATE TABLE refund_sagas (
    id BIGSERIAL PRIMARY KEY,
    return_id BIGINT NOT NULL,
    saga_id VARCHAR(100) UNIQUE NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING',
    -- PENDING, REFUNDING, RESTOCKING, COMPLETED, FAILED, COMPENSATING
    payment_processor VARCHAR(50), -- STRIPE, PAYPAL, etc
    refund_transaction_id VARCHAR(100),
    refund_status VARCHAR(50), -- PENDING, SUCCEEDED, FAILED, EXPIRED
    refund_amount DECIMAL(10,2),
    refund_method VARCHAR(50), -- ORIGINAL_CARD, ALTERNATE_CARD, STORE_CREDIT
    refund_failure_reason TEXT,
    inventory_update_status VARCHAR(50), -- PENDING, SUCCEEDED, FAILED
    compensated_at TIMESTAMP,
    completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_return_id (return_id),
    INDEX idx_saga_id (saga_id),
    INDEX idx_status (status)
);

-- Fraud flags (audit trail)
CREATE TABLE fraud_flags (
    id BIGSERIAL PRIMARY KEY,
    return_id BIGINT NOT NULL,
    user_id BIGINT NOT NULL,
    flag_type VARCHAR(100), -- SERIAL_RETURNER, EXPENSIVE_ITEM, etc
    risk_score DECIMAL(3,2), -- 0.0 to 1.0
    reason TEXT,
    action_taken VARCHAR(50), -- FLAGGED, REVIEWED, APPROVED, REJECTED
    reviewed_by VARCHAR(100),
    reviewed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (return_id) REFERENCES returns(id),
    INDEX idx_user_id (user_id),
    INDEX idx_created_at (created_at)
);

-- Return events (immutable event log)
CREATE TABLE return_events (
    id BIGSERIAL PRIMARY KEY,
    return_id BIGINT NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    -- INITIATED, LABEL_GENERATED, IN_TRANSIT, etc
    event_data JSONB,
    published_to_kafka BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (return_id) REFERENCES returns(id),
    INDEX idx_return_id_created (return_id, created_at)
);

-- Refund payment attempts (retry tracking)
CREATE TABLE refund_payment_attempts (
    id BIGSERIAL PRIMARY KEY,
    refund_saga_id VARCHAR(100) NOT NULL,
    attempt_number INT NOT NULL,
    payment_method VARCHAR(50),
    status VARCHAR(50), -- PENDING, SUCCEEDED, FAILED
    response_code VARCHAR(50),
    response_message TEXT,
    attempted_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (refund_saga_id) REFERENCES refund_sagas(saga_id),
    INDEX idx_saga_id_attempt (refund_saga_id, attempt_number)
);
```

### MongoDB Collections

```javascript
// Return policies by product category
db.return_policies.createIndex({
    "productCategory": 1
});

db.return_policies.insertOne({
    _id: ObjectId(),
    productCategory: "APPAREL",
    returnWindowDays: 30,
    restockingFeePct: 10,
    conditionRequirements: {
        acceptedConditions: ["LIKE_NEW", "GOOD", "FAIR"],
        rejectedConditions: ["POOR"]
    },
    defectAllowances: {
        "missing_tag": "ACCEPTED",
        "signs_of_wear": "ACCEPTED",
        "stained": "REJECTED"
    },
    reasonBlacklist: ["USED_EXTENSIVELY", "DAMAGED_BY_USER"]
});

// Fraud rules engine (ML model)
db.fraud_rules.insertOne({
    _id: "fraud-model-v2",
    rules: [
        {
            name: "serial_returner",
            threshold: 0.50,
            lookbackDays: 90,
            riskScore: 0.8,
            action: "REVIEW"
        },
        {
            name: "expensive_high_return",
            itemPriceThreshold: 1000,
            riskScore: 0.7,
            action: "INSPECT_CONDITION"
        },
        {
            name: "bulk_return_same_day",
            bulkThreshold: 5,
            riskScore: 0.6,
            action: "DELAY_PROCESSING"
        }
    ],
    updatedAt: ISODate("2024-01-15T00:00:00Z")
});

// Return carrier integrations (shipping label providers)
db.carrier_configs.insertOne({
    _id: ObjectId(),
    carrierName: "UPS",
    apiKey: "***", // encrypted
    dropoffAddress: {
        street: "123 Returns Warehouse Ave",
        city: "Los Angeles",
        state: "CA",
        zip: "90001"
    },
    labelFormat: "PDF",
    maxLabelsPerDay: 50000,
    costPerLabel: 0.50
});
```

---

## Redis Data Structures

### Return Status Cache
```
KEY: return:{returnId}
TYPE: Hash
TTL: 24 hours
VALUE:
{
  "status": "IN_TRANSIT",
  "orderId": "12345",
  "userId": "user-789",
  "estimatedRefund": "89.99",
  "lastUpdated": "1711939200000"
}

Example:
HSET return:RET-2024-001234 status IN_TRANSIT orderId 12345 estimatedRefund 89.99
EXPIRE return:RET-2024-001234 86400
```

### Return Eligibility Cache
```
KEY: return-eligible:{orderId}
TYPE: Hash
TTL: 6 hours
VALUE:
{
  "withinWindow": "true",
  "category": "APPAREL",
  "windowDays": "30",
  "daysUntilExpiry": "15",
  "maxReturnAmount": "399.99"
}
```

### User Return History (for fraud detection)
```
KEY: user-return-history:{userId}
TYPE: Set (capped at 100 entries)
TTL: 90 days
VALUE: returnId:timestamp:approvalStatus
[
  "RET-2024-001234:1711939200000:APPROVED",
  "RET-2024-001233:1711852800000:APPROVED",
  "RET-2024-001232:1711766400000:REJECTED"
]

Example:
SADD user-return-history:user-789 "RET-2024-001234:1711939200000:APPROVED"
EXPIRE user-return-history:user-789 7776000  // 90 days
```

### Fraud Risk Score (ephemeral)
```
KEY: fraud-risk:{returnId}
TYPE: String (JSON)
TTL: 12 hours
VALUE:
{
  "score": 0.75,
  "reasons": ["serial_returner:0.80", "expensive_item:0.60"],
  "action": "REVIEW"
}
```

### Refund Saga Status (distributed transaction tracking)
```
KEY: refund-saga:{sagaId}
TYPE: Hash
TTL: 7 days (until refund settles)
VALUE:
{
  "status": "REFUNDING",
  "paymentRefundStatus": "PENDING",
  "inventoryRestockStatus": "COMPLETED",
  "lastUpdated": "1711939200000"
}
```

---

## Kafka Event Flow

### Topic: `return-events`
```
Partition: returnId % num_partitions (order per return)
Retention: 90 days (compliance)
Replication: 3

Message (per state transition):
{
  "returnId": "RET-2024-001234",
  "eventType": "INITIATED|LABEL_GENERATED|IN_TRANSIT|RECEIVED|INSPECTING|APPROVED|REJECTED|REFUNDED",
  "orderId": 12345,
  "userId": 789,
  "eventData": {
    "status": "IN_TRANSIT",
    "trackingNumber": "1Z999AA...",
    "timestamp": 1711939200000
  },
  "causedBy": "customer|warehouse|system",
  "timestamp": 1711939200000
}
```

### Topic: `refund-saga-commands`
```
Message (orchestration):
{
  "sagaId": "saga-2024-001234",
  "returnId": "RET-2024-001234",
  "command": "PROCESS_REFUND",
  "refundAmount": 89.99,
  "refundMethod": "ORIGINAL_CARD",
  "lineItems": [
    {
      "lineItemId": 1,
      "quantity": 1,
      "refundAmount": 49.99
    }
  ],
  "timestamp": 1711939200000
}
```

### Topic: `refund-saga-events`
```
Message (status updates):
{
  "sagaId": "saga-2024-001234",
  "stepName": "REFUND_PAYMENT|RESTOCK_INVENTORY|UPDATE_RETURN_STATUS",
  "status": "STARTED|SUCCEEDED|FAILED",
  "stepData": { /* step-specific data */ },
  "timestamp": 1711939200000
}
```

### Topic: `fraud-alerts`
```
Message:
{
  "returnId": "RET-2024-001234",
  "userId": 789,
  "flagType": "SERIAL_RETURNER|EXPENSIVE_ITEM|BULK_RETURN",
  "riskScore": 0.85,
  "reason": "User has 60% return rate in last 90 days",
  "recommendedAction": "MANUAL_REVIEW",
  "timestamp": 1711939200000
}
```

---

## Implementation Code

### 1. ReturnService - Initiation & Eligibility

```java
package com.returns.service;

import org.springframework.stereotype.Service;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.transaction.annotation.Transactional;
import com.returns.repository.ReturnRepository;
import com.returns.repository.ReturnLineItemRepository;
import com.returns.entity.Return;
import com.returns.entity.ReturnLineItem;
import com.returns.dto.InitiateReturnRequest;
import com.returns.dto.ReturnEligibilityResult;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.util.*;

@Slf4j
@Service
public class ReturnService {

    private final ReturnRepository returnRepo;
    private final ReturnLineItemRepository lineItemRepo;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final OrderService orderService;
    private final ReturnPolicyService policyService;
    private final ObjectMapper objectMapper;

    @Transactional
    public Return initiateReturn(InitiateReturnRequest request) {
        long startTime = System.currentTimeMillis();

        try {
            // 1. Fetch order
            Order order = orderService.getOrder(request.getOrderId());
            if (order == null) {
                throw new OrderNotFoundException(request.getOrderId());
            }

            // 2. Check eligibility
            ReturnEligibilityResult eligibility = checkEligibility(
                order,
                request.getLineItemIds()
            );

            if (!eligibility.isEligible()) {
                throw new ReturnNotEligibleException(eligibility.getReason());
            }

            // 3. Create Return entity
            String returnId = generateReturnId();
            Return returnRecord = Return.builder()
                .returnId(returnId)
                .orderId(order.getId())
                .userId(order.getUserId())
                .status("INITIATED")
                .returnReason(request.getReason())
                .returnNotes(request.getNotes())
                .returnInitiatedAt(new Date())
                .build();

            returnRepo.save(returnRecord);

            // 4. Create Return Line Items (partial returns)
            BigDecimal totalRefund = BigDecimal.ZERO;

            for (Long lineItemId : request.getLineItemIds()) {
                OrderLineItem orderItem = order.getLineItem(lineItemId);
                if (orderItem == null) {
                    throw new LineItemNotFoundException(lineItemId);
                }

                // Pro-rata refund calculation
                BigDecimal itemRefund = orderItem.getUnitPrice()
                    .multiply(BigDecimal.valueOf(orderItem.getQuantity()));

                ReturnLineItem returnItem = ReturnLineItem.builder()
                    .returnId(returnRecord.getId())
                    .orderLineItemId(lineItemId)
                    .productId(orderItem.getProductId())
                    .quantity(orderItem.getQuantity())
                    .unitPrice(orderItem.getUnitPrice())
                    .itemCondition(null) // Set after inspection
                    .refundAmount(itemRefund)
                    .build();

                lineItemRepo.save(returnItem);
                totalRefund = totalRefund.add(itemRefund);
            }

            returnRecord.setRefundAmount(totalRefund);
            returnRepo.save(returnRecord);

            // 5. Publish INITIATED event
            publishReturnEvent(returnRecord, "INITIATED", null);

            // 6. Cache eligibility result
            cacheEligibilityResult(request.getOrderId(), eligibility);

            log.info("Return initiated [returnId={}, orderId={}, refundAmount={}, " +
                    "durationMs={}]",
                returnId, request.getOrderId(), totalRefund,
                System.currentTimeMillis() - startTime);

            return returnRecord;

        } catch (Exception e) {
            log.error("Error initiating return", e);
            throw new ReturnInitiationException(e.getMessage(), e);
        }
    }

    private ReturnEligibilityResult checkEligibility(
            Order order,
            List<Long> lineItemIds) {

        // 1. Check order is not already returned
        if ("CANCELLED".equals(order.getStatus()) ||
            "REFUNDED".equals(order.getStatus())) {
            return ReturnEligibilityResult.notEligible(
                "Order already cancelled or refunded");
        }

        // 2. Check within return window (category-specific)
        long daysSincePurchase = getDaysSincePurchase(order);
        String category = order.getLineItems().get(0).getCategory();
        int returnWindowDays = policyService.getReturnWindowDays(category);

        if (daysSincePurchase > returnWindowDays) {
            return ReturnEligibilityResult.notEligible(
                String.format("Return window expired (%d days)", returnWindowDays));
        }

        // 3. Check reason is not blacklisted
        List<String> blacklistedReasons = policyService
            .getBlacklistedReasons(category);
        // (would check request.getReason() here in full implementation)

        // 4. Check line items exist and are valid
        for (Long lineItemId : lineItemIds) {
            OrderLineItem item = order.getLineItem(lineItemId);
            if (item == null) {
                return ReturnEligibilityResult.notEligible(
                    "Line item not found: " + lineItemId);
            }

            // Can't return already-returned items
            if ("RETURNED".equals(item.getStatus())) {
                return ReturnEligibilityResult.notEligible(
                    "Item already returned");
            }
        }

        // 5. Calculate refund
        BigDecimal totalOrderValue = order.getLineItems().stream()
            .map(li -> li.getUnitPrice()
                .multiply(BigDecimal.valueOf(li.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        return ReturnEligibilityResult.eligible()
            .setDaysUntilExpiry(returnWindowDays - (int) daysSincePurchase)
            .setCategory(category)
            .setWindowDays(returnWindowDays)
            .setMaxReturnAmount(totalOrderValue);
    }

    private void publishReturnEvent(Return returnRecord, String eventType, Object eventData) {
        try {
            String message = String.format(
                "{\"returnId\":\"%s\",\"eventType\":\"%s\",\"orderId\":%d," +
                "\"userId\":%d,\"timestamp\":%d}",
                returnRecord.getReturnId(),
                eventType,
                returnRecord.getOrderId(),
                returnRecord.getUserId(),
                System.currentTimeMillis()
            );

            kafkaTemplate.send("return-events",
                returnRecord.getReturnId(),
                message);

        } catch (Exception e) {
            log.error("Failed to publish return event", e);
        }
    }

    private void cacheEligibilityResult(Long orderId, ReturnEligibilityResult result) {
        // Cache in Redis for fast lookups
        // (implementation skipped for brevity)
    }

    private long getDaysSincePurchase(Order order) {
        long diffMs = System.currentTimeMillis() -
                     order.getPurchaseDate().getTime();
        return diffMs / (1000 * 60 * 60 * 24);
    }

    private String generateReturnId() {
        // RET-YYYY-XXXXXX format
        return "RET-" + System.currentTimeMillis() / 1000;
    }
}

@Data
class ReturnEligibilityResult {
    private boolean eligible;
    private String reason;
    private int daysUntilExpiry;
    private String category;
    private int windowDays;
    private BigDecimal maxReturnAmount;

    public static ReturnEligibilityResult eligible() {
        return new ReturnEligibilityResult().setEligible(true);
    }

    public static ReturnEligibilityResult notEligible(String reason) {
        return new ReturnEligibilityResult().setEligible(false).setReason(reason);
    }
}
```

### 2. RefundSagaOrchestrator - Distributed Transaction

```java
package com.returns.service;

import org.springframework.stereotype.Service;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.kafka.annotation.KafkaListener;
import com.returns.repository.RefundSagaRepository;
import com.returns.entity.RefundSaga;
import com.returns.dto.RefundSagaCommand;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.util.*;

@Slf4j
@Service
public class RefundSagaOrchestrator {

    private final RefundSagaRepository sagaRepo;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final PaymentService paymentService;
    private final InventoryService inventoryService;
    private final ObjectMapper objectMapper;

    /**
     * Listen for APPROVED returns and start refund saga
     */
    @KafkaListener(topics = "return-events", groupId = "refund-saga-orchestrator")
    public void onReturnApproved(String returnEventMessage) {
        try {
            Map<String, Object> event = objectMapper.readValue(
                returnEventMessage, Map.class);

            String eventType = (String) event.get("eventType");
            if (!"APPROVED".equals(eventType)) {
                return;
            }

            String returnId = (String) event.get("returnId");
            Long orderId = ((Number) event.get("orderId")).longValue();
            Long userId = ((Number) event.get("userId")).longValue();

            // Create and start refund saga
            startRefundSaga(returnId, orderId, userId);

        } catch (Exception e) {
            log.error("Error processing approved return event", e);
        }
    }

    private void startRefundSaga(String returnId, Long orderId, Long userId) {
        String sagaId = generateSagaId();

        RefundSaga saga = RefundSaga.builder()
            .returnId(returnId)
            .sagaId(sagaId)
            .status("PENDING")
            .paymentRefundStatus("PENDING")
            .inventoryRestockStatus("PENDING")
            .createdAt(new Date())
            .build();

        sagaRepo.save(saga);

        log.info("Refund saga started [sagaId={}, returnId={}]", sagaId, returnId);

        // Step 1: Process refund payment
        processRefundPayment(saga);
    }

    /**
     * Step 1: Process payment refund
     * Retry logic: 3 attempts with exponential backoff
     */
    private void processRefundPayment(RefundSaga saga) {
        try {
            saga.setStatus("REFUNDING");
            saga.setPaymentRefundStatus("PENDING");
            sagaRepo.save(saga);

            log.info("Processing refund payment [sagaId={}]", saga.getSagaId());

            // Get return details
            Return returnRecord = returnRepo.findByReturnId(saga.getReturnId());
            BigDecimal refundAmount = returnRecord.getNetRefund();

            // Call payment processor
            String refundTxnId = paymentService.processRefund(
                returnRecord.getOrderId(),
                refundAmount,
                "ORIGINAL_CARD"  // Try original card first
            );

            saga.setRefundStatus("SUCCEEDED");
            saga.setRefundTransactionId(refundTxnId);
            saga.setPaymentRefundStatus("COMPLETED");
            sagaRepo.save(saga);

            // Step 2: Restock inventory
            processInventoryRestock(saga);

        } catch (PaymentFailedException e) {
            saga.setRefundStatus("FAILED");
            saga.setRefundFailureReason(e.getMessage());
            saga.setPaymentRefundStatus("FAILED");
            sagaRepo.save(saga);

            // Retry with fallback payment method or escalate
            handleRefundPaymentFailure(saga, e);
        }
    }

    /**
     * Step 2: Restock inventory (async)
     * Failure compensation: don't undo refund (already completed)
     * Instead: alert ops for manual reconciliation
     */
    private void processInventoryRestock(RefundSaga saga) {
        try {
            saga.setInventoryRestockStatus("PENDING");
            sagaRepo.save(saga);

            log.info("Processing inventory restock [sagaId={}]", saga.getSagaId());

            Return returnRecord = returnRepo.findByReturnId(saga.getReturnId());

            // Publish restock command (async)
            RestockCommand restockCmd = RestockCommand.builder()
                .sagaId(saga.getSagaId())
                .returnId(returnRecord.getReturnId())
                .items(returnRecord.getLineItems())
                .build();

            kafkaTemplate.send("inventory-restock-commands",
                saga.getSagaId(),
                objectMapper.writeValueAsString(restockCmd));

            // Wait for async response (via event listener)
        } catch (Exception e) {
            log.error("Error processing inventory restock", e);
            saga.setInventoryRestockStatus("FAILED");
            sagaRepo.save(saga);
            // Compensate: refund already succeeded, so just alert
        }
    }

    /**
     * Listen for inventory restock completion
     */
    @KafkaListener(topics = "inventory-restock-completed", groupId = "refund-saga")
    public void onInventoryRestockCompleted(String restockCompletedMessage) {
        try {
            Map<String, Object> event = objectMapper.readValue(
                restockCompletedMessage, Map.class);

            String sagaId = (String) event.get("sagaId");
            String status = (String) event.get("status");

            RefundSaga saga = sagaRepo.findBySagaId(sagaId);
            saga.setInventoryRestockStatus(status);

            if ("COMPLETED".equals(status)) {
                saga.setStatus("COMPLETED");
                saga.setCompletedAt(new Date());
                log.info("Refund saga completed [sagaId={}]", sagaId);
            } else {
                saga.setStatus("FAILED");
                log.error("Inventory restock failed [sagaId={}]", sagaId);
            }

            sagaRepo.save(saga);

        } catch (Exception e) {
            log.error("Error processing restock completion event", e);
        }
    }

    private void handleRefundPaymentFailure(RefundSaga saga, PaymentFailedException e) {
        log.warn("Refund payment failed [sagaId={}, reason={}]",
            saga.getSagaId(), e.getMessage());

        // Attempt 1: Try alternate payment method
        // Attempt 2: Use store credit
        // Attempt 3: Manual payment review

        // For now, just flag for manual review
        saga.setStatus("PENDING_MANUAL_REVIEW");
        sagaRepo.save(saga);
    }

    private String generateSagaId() {
        return "saga-" + UUID.randomUUID().toString();
    }
}

@Data
@Builder
class RefundSagaCommand {
    private String sagaId;
    private String returnId;
    private BigDecimal refundAmount;
    private String refundMethod;
    private List<LineItemRefund> lineItems;
    private Long timestamp;
}

@Data
@Builder
class RestockCommand {
    private String sagaId;
    private String returnId;
    private List<ReturnLineItem> items;
    private Long timestamp;
}
```

### 3. FraudCheckService - Fraud Detection

```java
package com.returns.service;

import org.springframework.stereotype.Service;
import org.springframework.data.mongodb.repository.MongoRepository;
import com.returns.repository.ReturnRepository;
import com.returns.repository.FraudFlagRepository;
import com.returns.entity.FraudFlag;
import com.returns.entity.Return;
import com.mongodb.client.MongoCollection;
import org.bson.Document;
import lombok.extern.slf4j.Slf4j;
import java.util.*;

@Slf4j
@Service
public class FraudCheckService {

    private final ReturnRepository returnRepo;
    private final FraudFlagRepository fraudFlagRepo;
    private final MongoCollection<Document> fraudRulesCollection;

    public void checkFraud(Return returnRecord) {
        try {
            double riskScore = 0.0;
            List<String> flagReasons = new ArrayList<>();

            // Rule 1: Serial returner
            Double serialReturnerRisk = checkSerialReturner(returnRecord.getUserId());
            if (serialReturnerRisk > 0) {
                riskScore = Math.max(riskScore, serialReturnerRisk);
                flagReasons.add("SERIAL_RETURNER:" + serialReturnerRisk);
            }

            // Rule 2: Expensive item high return rate
            Double expensiveItemRisk = checkExpensiveItemReturn(returnRecord);
            if (expensiveItemRisk > 0) {
                riskScore = Math.max(riskScore, expensiveItemRisk);
                flagReasons.add("EXPENSIVE_ITEM:" + expensiveItemRisk);
            }

            // Rule 3: Bulk returns same day
            Double bulkReturnRisk = checkBulkReturnSameDay(returnRecord.getUserId());
            if (bulkReturnRisk > 0) {
                riskScore = Math.max(riskScore, bulkReturnRisk);
                flagReasons.add("BULK_RETURN:" + bulkReturnRisk);
            }

            // Rule 4: Return reason mismatch (condition vs reason)
            // (Would be checked at warehouse inspection time)

            // Flag if risk score > threshold
            if (riskScore > 0.5) {
                FraudFlag flag = FraudFlag.builder()
                    .returnId(returnRecord.getId())
                    .userId(returnRecord.getUserId())
                    .flagType("COMPOSITE")
                    .riskScore(riskScore)
                    .reason(String.join(", ", flagReasons))
                    .actionTaken("FLAGGED")
                    .createdAt(new Date())
                    .build();

                fraudFlagRepo.save(flag);

                log.warn("Return flagged for fraud review [returnId={}, riskScore={}, " +
                        "reasons={}]",
                    returnRecord.getReturnId(), riskScore, flagReasons);
            }

        } catch (Exception e) {
            log.error("Error checking fraud for return: {}", returnRecord.getId(), e);
        }
    }

    private Double checkSerialReturner(Long userId) {
        // Get user's return history (last 90 days)
        List<Return> userReturns = returnRepo.findByUserIdAndReturnInitiatedAtAfter(
            userId,
            new Date(System.currentTimeMillis() - 90 * 24 * 60 * 60 * 1000)
        );

        if (userReturns.isEmpty()) {
            return 0.0;
        }

        // Calculate return rate
        int approvedCount = (int) userReturns.stream()
            .filter(r -> "APPROVED".equals(r.getStatus()))
            .count();

        double returnRate = approvedCount / (double) userReturns.size();

        log.debug("Serial returner check [userId={}, returnRate={}, count={}]",
            userId, returnRate, userReturns.size());

        // Return rate > 50% = high risk
        if (returnRate > 0.50) {
            return 0.8;  // 80% risk
        } else if (returnRate > 0.40) {
            return 0.6;
        }

        return 0.0;
    }

    private Double checkExpensiveItemReturn(Return returnRecord) {
        // Get average item value
        Double avgItemValue = returnRecord.getLineItems().stream()
            .mapToDouble(li -> li.getUnitPrice().doubleValue())
            .average()
            .orElse(0.0);

        log.debug("Expensive item check [returnId={}, avgValue={}]",
            returnRecord.getReturnId(), avgItemValue);

        // Items > $1000 with limited purchase history = risky
        if (avgItemValue > 1000) {
            int purchaseCount = returnRepo.countByUserIdAndStatusApproved(
                returnRecord.getUserId());

            if (purchaseCount < 5) {
                return 0.7;  // 70% risk for new customers buying expensive items
            } else {
                return 0.4;  // Lower risk for established customers
            }
        }

        return 0.0;
    }

    private Double checkBulkReturnSameDay(Long userId) {
        // Check if user initiating >3 returns same day
        Date today = new Date();
        List<Return> todayReturns = returnRepo.findByUserIdAndReturnInitiatedAtAfter(
            userId,
            today
        );

        log.debug("Bulk return check [userId={}, countToday={}]",
            userId, todayReturns.size());

        if (todayReturns.size() > 3) {
            return 0.6;  // 60% risk
        }

        return 0.0;
    }
}
```

### 4. WarehouseReceiver - Inspection & Status Update

```java
package com.returns.service;

import org.springframework.stereotype.Service;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.transaction.annotation.Transactional;
import com.returns.repository.ReturnRepository;
import com.returns.entity.Return;
import com.returns.dto.WarehouseReceiveRequest;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.extern.slf4j.Slf4j;
import java.math.BigDecimal;
import java.util.*;

@Slf4j
@Service
public class WarehouseReceiver {

    private final ReturnRepository returnRepo;
    private final ReturnLineItemRepository lineItemRepo;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ReturnPolicyService policyService;

    @Transactional
    public void receiveReturn(WarehouseReceiveRequest request) {
        try {
            Return returnRecord = returnRepo.findByReturnId(request.getReturnId());
            if (returnRecord == null) {
                throw new ReturnNotFoundException(request.getReturnId());
            }

            // 1. Update return status
            returnRecord.setStatus("RECEIVED");
            returnRecord.setReceivedAtWarehouse(new Date());
            returnRepo.save(returnRecord);

            // 2. Inspect each line item
            inspectLineItems(returnRecord, request.getItemsReceived());

            // 3. Determine approval/rejection
            determineApproval(returnRecord);

            log.info("Return received and inspected [returnId={}]",
                request.getReturnId());

        } catch (Exception e) {
            log.error("Error receiving return", e);
            throw new WarehouseOperationException(e);
        }
    }

    private void inspectLineItems(
            Return returnRecord,
            List<InspectedItem> inspectedItems) {

        for (InspectedItem inspected : inspectedItems) {
            ReturnLineItem lineItem = lineItemRepo.findById(
                inspected.getLineItemId()).orElse(null);

            if (lineItem == null) continue;

            // Update condition and defects
            lineItem.setItemCondition(inspected.getCondition());
            lineItem.setDefects(String.join(",", inspected.getDefects()));
            lineItem.setInspectionNotes(inspected.getNotes());

            // Calculate restocking fee based on condition
            BigDecimal restockingFee = calculateRestockingFee(
                lineItem,
                inspected.getCondition()
            );
            lineItem.setRestockingFee(restockingFee);

            // Calculate net refund (original price - restocking fee)
            BigDecimal netRefund = lineItem.getUnitPrice()
                .multiply(BigDecimal.valueOf(lineItem.getQuantity()))
                .subtract(restockingFee);
            lineItem.setRefundAmount(netRefund);

            lineItemRepo.save(lineItem);

            log.debug("Item inspected [lineItemId={}, condition={}, restockFee={}]",
                inspected.getLineItemId(), inspected.getCondition(), restockingFee);
        }
    }

    private void determineApproval(Return returnRecord) {
        boolean approve = true;
        String rejectionReason = null;

        // Check each line item condition
        for (ReturnLineItem lineItem : returnRecord.getLineItems()) {
            String condition = lineItem.getItemCondition();

            // POOR condition = reject
            if ("POOR".equals(condition)) {
                approve = false;
                rejectionReason = "Item in poor condition: " + lineItem.getDefects();
                break;
            }

            // Check if defects match expected vs reasonable wear
            if (hasUnexpectedDamage(lineItem)) {
                approve = false;
                rejectionReason = "Damage inconsistent with stated reason";
                break;
            }
        }

        // Update return status
        String newStatus = approve ? "APPROVED" : "REJECTED";
        returnRecord.setStatus(newStatus);
        returnRecord.setApprovedAt(new Date());

        if (!approve && rejectionReason != null) {
            log.info("Return rejected [returnId={}, reason={}]",
                returnRecord.getReturnId(), rejectionReason);
        }

        returnRepo.save(returnRecord);

        // Publish event
        publishReturnEvent(returnRecord, newStatus);

        // If approved, trigger refund saga
        if (approve) {
            triggerRefundSaga(returnRecord);
        }
    }

    private BigDecimal calculateRestockingFee(
            ReturnLineItem lineItem,
            String condition) {

        BigDecimal basePrice = lineItem.getUnitPrice()
            .multiply(BigDecimal.valueOf(lineItem.getQuantity()));

        double feePct = 0.0;

        switch (condition) {
            case "LIKE_NEW":
                feePct = 5.0;  // 5% restocking fee
                break;
            case "GOOD":
                feePct = 10.0; // 10% restocking fee
                break;
            case "FAIR":
                feePct = 20.0; // 20% restocking fee
                break;
            case "POOR":
                feePct = 50.0; // 50% (effectively reject)
                break;
        }

        return basePrice.multiply(
            BigDecimal.valueOf(feePct / 100.0)
        );
    }

    private boolean hasUnexpectedDamage(ReturnLineItem lineItem) {
        // Check if defects match the stated reason
        // e.g., if customer said "wrong size" but item is stained, reject
        List<String> defects = Arrays.asList(
            lineItem.getDefects().split(","));

        // Allowed defects for different return reasons
        if ("WRONG_SIZE".equals(lineItem.getReturn().getReturnReason())) {
            // Only minor wear acceptable
            return defects.stream()
                .anyMatch(d -> d.contains("STAIN") || d.contains("RIP"));
        }

        return false;
    }

    private void publishReturnEvent(Return returnRecord, String status) {
        try {
            String message = String.format(
                "{\"returnId\":\"%s\",\"eventType\":\"%s\",\"timestamp\":%d}",
                returnRecord.getReturnId(),
                status,
                System.currentTimeMillis()
            );

            kafkaTemplate.send("return-events",
                returnRecord.getReturnId(),
                message);

        } catch (Exception e) {
            log.error("Failed to publish return event", e);
        }
    }

    private void triggerRefundSaga(Return returnRecord) {
        // Publish APPROVED event → RefundSagaOrchestrator listens
        publishReturnEvent(returnRecord, "APPROVED");
    }
}

@Data
class InspectedItem {
    private Long lineItemId;
    private String condition;  // LIKE_NEW, GOOD, FAIR, POOR
    private List<String> defects;
    private String notes;
}

@Data
class WarehouseReceiveRequest {
    private String returnId;
    private String trackingNumber;
    private List<InspectedItem> itemsReceived;
}
```

---

## Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| Order service down when initiating return | Can't fetch order details | Cache order (24h TTL), show cached view |
| Refund payment fails (card declined) | Customer doesn't get refund | Retry with alt payment method, store credit, manual review |
| Inventory restock fails | Item not back in stock | Compensation: still refund, flag for manual ops reconciliation |
| Kafka broker down | Can't track state changes | DLQ captures messages, manual replay after recovery |
| Warehouse inspection delay | Customer waits for refund | Email status updates; handle optimistically |
| Return label API down | Can't generate labels | Use fallback carrier (USPS), manual label generation |
| Fraud flag delays processing | Legitimate customer waits | Auto-approve after 48h if no manual review |

---

## Scaling Strategy

### Horizontal
- **ReturnService:** Stateless, scale behind load balancer
- **Kafka:** 1000+ partitions (one per return type/category)
- **PostgreSQL:** Read replicas for queries, writes to primary

### Vertical
- Increase PostgreSQL IOPS for high volume of writes
- Increase Kafka retention (90 days compliance)
- Increase Redis memory for return status cache

### Caching
- Redis for return status (24h TTL)
- Redis for order cache (24h TTL)
- PostgreSQL for immutable audit/event log

### Rate Limiting
- Limit returns initiation per user per day (prevent abuse)
- Rate limit fraud checks (avoid timeout)

---

## Monitoring & Observability

### Key Metrics
```
return_initiation_latency_ms (histogram)
  - p50, p95, p99

return_approval_rate (gauge)
  - approved / total returns

fraud_flag_rate (gauge)
  - flagged returns / total

refund_saga_completion_time_days (histogram)
  - time from approval to refund completion

inventory_restock_failure_rate (gauge)
  - failed restocks / total approved

payment_processor_latency_ms (histogram)
  - refund processing time per provider
```

### Alerts
- Return approval latency > 5 days (SLA)
- Fraud flag rate > 2% (unusually high)
- Refund saga failure > 1%
- Kafka lag > 1 hour on any consumer group
- PostgreSQL audit log insertion latency > 1 second

---

## Summary Cheat Sheet

| Component | Tech | Key Details |
|-----------|------|------------|
| **Return Initiation** | Spring Boot | Eligibility check, line-item level |
| **State Machine** | PostgreSQL | Immutable event log, 7 transitions |
| **Partial Returns** | PostgreSQL | Pro-rata refund calculation |
| **Fraud Detection** | MongoDB + ML | Serial returner, expensive item, bulk |
| **Refund Saga** | Kafka orchestration | 2-phase: payment + inventory |
| **Compensation** | Idempotent ops | If restock fails, still refund |
| **Price Lock** | Redis | 24h TTL, version-checked validation |
| **Return Policies** | MongoDB | Category-specific windows, fees |
| **Audit Trail** | PostgreSQL | 7-year retention, compliance |
| **Propagation** | Kafka | State changes, <5min SLA |

---

**End of Deep Dive**
