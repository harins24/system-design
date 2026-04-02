---
title: Payment Authorization and Capture
layout: default
---

# Payment Authorization and Capture — Deep Dive Design

> **Scenario:** Two-phase payment — Phase 1: Authorize (hold amount) when order placed; Phase 2: Capture (charge) when order ships. Authorization expires after 7 days. Support partial captures ($100 order → ship $60 now, $40 later). Support cancellations. Handle auth failures and retries.
>
> **Tech Stack:** Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka

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

- **Two-Phase Payment:** Phase 1 (authorize) when order is placed; Phase 2 (capture) when order ships.
- **Authorization Validity:** Authorizations expire after 7 days if not captured.
- **Partial Captures:** A single $100 authorization can be captured in multiple installments ($60, then $40).
- **Cancellations:** Cancel authorization before expiry (refund hold, release reserved funds).
- **Automatic Re-Authorization:** If authorization expires before shipping, re-authorize automatically.
- **Idempotent Captures:** Same capture request twice should not double-charge.
- **Capture Failure Handling:** If capture fails post-shipment, retry 3x then escalate to ops.
- **State Transitions:** AUTHORIZED → PARTIALLY_CAPTURED → FULLY_CAPTURED (or EXPIRED, CANCELLED, FAILED).
- **Audit Trail:** Track all authorization and capture events with timestamps and metadata.

### Non-Functional Requirements

- **Latency:** Authorization under 3s (p99); capture under 2s (p99).
- **Throughput:** 100K authorizations/day; 80K captures/day (80% of orders ship within 7 days).
- **Consistency:** Strong consistency on authorization state; at-most-once capture.
- **Availability:** 99.9% SLA; tolerate temporary gateway failures.
- **Data Retention:** 7 years for PCI audit; soft-delete authorizations.

### Out of Scope

- Fraud detection (handled separately).
- Advanced refund logic (Q23 covers refunds).
- Multi-currency settlement (Q21 covers FX).
- PCI compliance mechanisms beyond token handling.

---

## 2. Capacity Estimation

```
CAPACITY PLANNING TABLE

┌─────────────────────────────────────────────────────────────────┐
│ Metric                          │ Calculation       │ Value       │
├─────────────────────────────────────────────────────────────────┤
│ Authorizations per day          │ Given (100K)     │ 100,000     │
│ TPS (nominal)                   │ 100K / 86400s    │ 1.16 TPS    │
│ TPS (peak, 10x)                 │ 1.16 * 10        │ 11.6 TPS    │
│ Avg authorization hold time     │ Order → ship     │ 3 days      │
│ Captured orders (% auth)        │ Estimate         │ 80%         │
│ Captures per day                │ 100K * 0.8       │ 80,000      │
│                                 │                  │             │
│ Auth DB rows/year               │ 100K * 365       │ 36.5M rows  │
│ Auth row size (includes LOBs)   │ Estimate         │ 2 KB        │
│ PostgreSQL storage/year         │ 36.5M * 2KB      │ 73 GB       │
│ With indices (2x)               │ 73GB * 2         │ 146 GB      │
│                                 │                  │             │
│ Active authorizations (3-day avg)  │ 100K * 3 days  │ 300,000     │
│ Redis hash entry size (auth)    │ Estimate         │ 500 B       │
│ Total Redis for active auths    │ 300K * 500B      │ 150 MB      │
│                                 │                  │             │
│ Capture records per auth        │ Avg 1.5 captures │ 150K        │
│ Per year capture records        │ 36.5M * 1.5      │ 54.75M rows │
│                                 │                  │             │
│ Re-authorization retries (5%)   │ 100K * 0.05      │ 5,000/day   │
│ Kafka events (all transitions)  │ Auths+Captures   │ 180K/day    │
│ Avg event size                  │ Estimate         │ 1 KB        │
│ Daily Kafka volume              │ 180K * 1KB       │ 180 MB      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                      CHECKOUT / ORDER SERVICE                     │
│          (User places order, triggers authorization)              │
└──────────────┬───────────────────────────────────────────────────┘
               │ Order.Placed event
               ▼
┌──────────────────────────────────────────────────────────────────┐
│                    AUTHORIZATION SERVICE                          │
│                   (Core: Two-Phase Payment)                       │
│                                                                    │
│  ┌──────────────────┐        ┌──────────────┐                    │
│  │ AuthorizeHandler │───────▶│ GatewayRouter│                    │
│  │ (Phase 1)        │        │ (Charge now) │                    │
│  └──────────────────┘        └──────────────┘                    │
│           ▼                                                       │
│  ┌──────────────────────────────┐                                │
│  │ PaymentAuthorization Entity  │                                │
│  │ Status: AUTHORIZED           │                                │
│  │ Expires: +7 days             │                                │
│  └──────────────────────────────┘                                │
└──────────────┬───────────────────────────────────────────────────┘
               │ payment.authorized event
               ▼
        ┌──────────────────┐
        │   Kafka Topic    │
        │payment.authorize │
        │payment.captured  │
        │payment.expired   │
        │payment.cancelled │
        └─────┬────────────┘
              │
     ┌────────┴──────────┬───────────────┐
     ▼                   ▼               ▼
  ┌────────┐       ┌──────────┐    ┌────────────┐
  │ Fulfil-│       │Authorization  │ Expiry Job │
  │ment    │       │Health Monitor │            │
  │Service │       │(Auto-ReAuth)   │ (7-day)   │
  │        │       │                │ Scans &   │
  │ Trigger│       │ Every 6 days   │ alerts    │
  │ Capture│       │ check if auth  │           │
  │        │       │ expires soon   │           │
  └────────┘       └──────────────┘ └────────────┘
     │                    │
     ▼                    ▼
  ┌────────────────────────────────────────────┐
  │           CAPTURE SERVICE                   │
  │        (Phase 2: Charge when ship)          │
  │                                             │
  │  ┌────────────────────────────────────┐    │
  │  │ CaptureHandler                     │    │
  │  │ • Check authorization still valid  │    │
  │  │ • Validate amount <= authorized    │    │
  │  │ • Idempotency check                │    │
  │  │ • Call gateway to capture          │    │
  │  │ • Deduct from authorized amount    │    │
  │  └────────────────────────────────────┘    │
  └────────────────────────────────────────────┘
            │         │         │
     ┌──────┘         │         └──────┐
     ▼                ▼                 ▼
  SUCCESS        RETRY(3x)          FAIL
     │                │                 │
     ▼                ▼                 ▼
  Publish        Publish Kafka   Create
  captured       retry event     Manual
  event                          Review
                                 Ticket
                                    │
                                    ▼
                              Alert ops
                              team on
                              Slack
     │
     └─────────────────────────┐
                               ▼
                    ┌────────────────────────┐
                    │    Accounting Service   │
                    │ (Consume capture evts) │
                    │ Post to ledger         │
                    └────────────────────────┘
     │
     └─────────────────────────┐
                               ▼
                    ┌────────────────────────┐
                    │   PostgreSQL           │
                    │ • Authorizations       │
                    │ • Captures             │
                    │ • Audit log            │
                    └────────────────────────┘
```

---

## 4. Core Design Questions Answered

### Q1: How do you track authorization status?

**Design: State Machine with Strong Consistency**

Use a PostgreSQL `PaymentAuthorization` entity with a status column that follows a state machine. Transitions are atomic updates guarded by optimistic locking (version field) or pessimistic row locks.

```java
@Entity
@Table(name = "payment_authorizations")
public class PaymentAuthorization {
    @Id
    private UUID id;

    @Column(nullable = false)
    private UUID orderId;

    @Column(nullable = false)
    private String status;  // AUTHORIZED, PARTIALLY_CAPTURED, FULLY_CAPTURED, EXPIRED, CANCELLED, FAILED

    @Column(nullable = false)
    private BigDecimal authorizedAmount;

    @Column(nullable = false)
    private BigDecimal capturedAmount = BigDecimal.ZERO;

    @Column(nullable = false)
    private String currency;

    @Column(nullable = false)
    private String gatewayName;

    @Column(nullable = false)
    private String gatewayAuthorizationId;  // e.g., "pi_1234567"

    @Column(nullable = false)
    private Instant authorizedAt;

    @Column(nullable = false)
    private Instant expiresAt;  // +7 days from authorized

    @Column(nullable = true)
    private Instant fullyCaptureddAt;

    @Column(nullable = true)
    private Instant cancelledAt;

    @Column(nullable = true)
    private Instant reauthorizedAt;

    @Version
    private Long version;  // Optimistic locking

    @Column(nullable = false, updatable = false)
    private Instant createdAt = Instant.now();

    @Column(nullable = false)
    private Instant updatedAt = Instant.now();

    @OneToMany(mappedBy = "authorization", cascade = CascadeType.ALL)
    private List<CaptureRecord> captures;

    // Status transition guards
    public void transitionToPartiallyCaptured() {
        if (this.status.equals("AUTHORIZED") || this.status.equals("PARTIALLY_CAPTURED")) {
            this.status = "PARTIALLY_CAPTURED";
        } else {
            throw new IllegalStateException("Cannot capture from status: " + this.status);
        }
    }

    public void transitionToFullyCaptured() {
        if (this.status.equals("AUTHORIZED") || this.status.equals("PARTIALLY_CAPTURED")) {
            this.status = "FULLY_CAPTURED";
            this.fullyCaptureddAt = Instant.now();
        } else {
            throw new IllegalStateException("Cannot capture from status: " + this.status);
        }
    }

    public void transitionToExpired() {
        if (!this.status.equals("FULLY_CAPTURED") && !this.status.equals("CANCELLED")) {
            this.status = "EXPIRED";
        }
    }

    public void transitionToCancelled() {
        if (!this.status.equals("FULLY_CAPTURED")) {
            this.status = "CANCELLED";
            this.cancelledAt = Instant.now();
        } else {
            throw new IllegalStateException("Cannot cancel fully captured authorization");
        }
    }

    public BigDecimal getRemainingAuthority() {
        return this.authorizedAmount.subtract(this.capturedAmount);
    }
}

@Entity
@Table(name = "capture_records")
public class CaptureRecord {
    @Id
    private UUID id;

    @ManyToOne
    @JoinColumn(name = "authorization_id")
    private PaymentAuthorization authorization;

    @Column(nullable = false)
    private BigDecimal captureAmount;

    @Column(nullable = false)
    private String status;  // PENDING, COMPLETED, FAILED, RETRYING

    @Column(nullable = false)
    private UUID shipmentId;  // Tied to specific shipment

    @Column(nullable = true)
    private String gatewayTransactionId;

    @Column(nullable = false)
    private Integer retryCount = 0;

    @Column(nullable = false)
    private Instant createdAt = Instant.now();

    @Column(nullable = true)
    private Instant completedAt;

    @Column(nullable = true)
    private String errorMessage;
}
```

**Service to Track Status:**

```java
@Service
public class AuthorizationStatusService {

    private final AuthorizationRepository authRepo;
    private final CaptureRecordRepository captureRepo;
    private final RedisTemplate<String, Object> redis;
    private final Logger log = LoggerFactory.getLogger(AuthorizationStatusService.class);

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public PaymentAuthorization getAuthorizationStatus(UUID authId) {
        return authRepo.findById(authId)
            .orElseThrow(() -> new EntityNotFoundException("Authorization not found"));
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void updateAuthorizationStatus(UUID authId, String newStatus) {
        PaymentAuthorization auth = authRepo.findByIdWithLock(authId);  // Pessimistic lock

        if (auth == null) {
            throw new EntityNotFoundException("Authorization not found");
        }

        // Validate state transition
        String oldStatus = auth.getStatus();
        if (!isValidTransition(oldStatus, newStatus)) {
            throw new IllegalStateException(
                String.format("Invalid transition: %s -> %s", oldStatus, newStatus)
            );
        }

        auth.setStatus(newStatus);
        auth.setUpdatedAt(Instant.now());
        authRepo.save(auth);

        // Update Redis cache for fast lookup
        String cacheKey = "auth:status:" + authId;
        redis.opsForValue().set(cacheKey, newStatus, Duration.ofHours(24));

        log.info("Authorization {} status updated: {} -> {}", authId, oldStatus, newStatus);
    }

    private boolean isValidTransition(String from, String to) {
        return switch (from) {
            case "AUTHORIZED" -> to.matches("PARTIALLY_CAPTURED|FULLY_CAPTURED|CANCELLED|EXPIRED");
            case "PARTIALLY_CAPTURED" -> to.matches("FULLY_CAPTURED|CANCELLED|EXPIRED");
            case "FULLY_CAPTURED" -> to.equals("EXPIRED");  // No more transitions
            case "CANCELLED", "EXPIRED", "FAILED" -> false;  // Terminal states
            default -> false;
        };
    }
}
```

---

### Q2: How do you handle authorization expiration?

**Design: Scheduled Expiry Job + Auto-Re-Authorization**

A scheduled job runs daily (e.g., 1 AM) to scan for authorizations expiring within the next 24 hours. If the order hasn't shipped, re-authorize. If it has shipped but capture is still pending, prioritize capture.

```java
@Service
public class AuthorizationExpiryJob {

    private final AuthorizationRepository authRepo;
    private final PaymentGatewayRouter gatewayRouter;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final Logger log = LoggerFactory.getLogger(AuthorizationExpiryJob.class);

    @Scheduled(cron = "0 1 * * *")  // 1 AM daily
    public void scanAndHandleExpiringAuthorizations() {
        Instant tomorrow = Instant.now().plus(Duration.ofDays(1));
        List<PaymentAuthorization> expiringAuths =
            authRepo.findByStatusAndExpiresAtBefore("AUTHORIZED", tomorrow);

        log.info("Found {} authorizations expiring within 24h", expiringAuths.size());

        for (PaymentAuthorization auth : expiringAuths) {
            try {
                handleExpiringAuthorization(auth);
            } catch (Exception e) {
                log.error("Error handling expiring authorization {}: {}",
                    auth.getId(), e.getMessage());
                // Create manual review ticket for ops
                createManualReviewTicket(auth, "Expiry handling failed");
            }
        }
    }

    private void handleExpiringAuthorization(PaymentAuthorization auth) throws PaymentException {
        Instant now = Instant.now();

        // Check if order has already shipped
        boolean orderShipped = isOrderShipped(auth.getOrderId());

        if (orderShipped) {
            // Priority: ensure capture happens before expiry
            log.info("Order {} already shipped, prioritizing capture for auth {}",
                auth.getOrderId(), auth.getId());

            // Trigger immediate capture if not already pending
            List<CaptureRecord> pendingCaptures =
                captureRepo.findByAuthorizationIdAndStatus(auth.getId(), "PENDING");

            if (pendingCaptures.isEmpty()) {
                // No pending capture; order shipped without capture = error
                log.warn("Order shipped but no pending capture for auth {}", auth.getId());
                auth.setStatus("EXPIRED");
                authRepo.save(auth);
                publishAuthExpiredEvent(auth);
            } else {
                // Capture is pending; extend auth 1 more day
                auth.setExpiresAt(auth.getExpiresAt().plus(Duration.ofDays(1)));
                authRepo.save(auth);
                log.info("Extended authorization {} expiry by 1 day", auth.getId());
            }
        } else {
            // Order hasn't shipped yet; re-authorize
            log.info("Order {} not shipped, re-authorizing", auth.getOrderId());
            reauthorizePayment(auth);
        }
    }

    private void reauthorizePayment(PaymentAuthorization auth) throws PaymentException {
        try {
            var authRequest = new AuthorizeRequest(
                auth.getId() + ":reauth:" + Instant.now().toEpochMilli(),
                "use_existing_token",  // Stripe's saved payment method
                auth.getAuthorizedAmount(),
                auth.getCurrency(),
                auth.getCustomerId(),
                Map.of("original_auth_id", auth.getId().toString())
            );

            PaymentResult result = gatewayRouter.processPaymentWithFailover(authRequest);

            if (result.status() == PaymentStatus.AUTHORIZED) {
                // Create new authorization record
                var newAuth = new PaymentAuthorization();
                newAuth.setOrderId(auth.getOrderId());
                newAuth.setAuthorizedAmount(auth.getAuthorizedAmount());
                newAuth.setCurrency(auth.getCurrency());
                newAuth.setGatewayName(auth.getGatewayName());
                newAuth.setGatewayAuthorizationId(result.gatewayTransactionId());
                newAuth.setAuthorizedAt(Instant.now());
                newAuth.setExpiresAt(Instant.now().plus(Duration.ofDays(7)));
                newAuth.setStatus("AUTHORIZED");
                newAuth.setReauthorizedAt(Instant.now());

                authRepo.save(newAuth);
                log.info("Re-authorization succeeded for order {}", auth.getOrderId());

                // Mark old auth as replaced
                auth.setStatus("EXPIRED");
                authRepo.save(auth);

                publishReauthorizedEvent(newAuth);
            } else {
                log.error("Re-authorization failed for order {}: {}",
                    auth.getOrderId(), result.message());
                createManualReviewTicket(auth, "Re-authorization failed: " + result.message());
            }
        } catch (Exception e) {
            log.error("Re-authorization exception for order {}", auth.getOrderId(), e);
            createManualReviewTicket(auth, "Re-authorization exception: " + e.getMessage());
        }
    }

    private boolean isOrderShipped(UUID orderId) {
        // Query fulfillment service or event store
        return fulfillmentService.isOrderShipped(orderId);
    }

    private void createManualReviewTicket(PaymentAuthorization auth, String reason) {
        var ticket = new ManualReviewTicket();
        ticket.setAuthorizationId(auth.getId());
        ticket.setOrderId(auth.getOrderId());
        ticket.setReason(reason);
        ticket.setStatus("OPEN");
        ticket.setCreatedAt(Instant.now());
        manualReviewTicketRepo.save(ticket);

        // Notify ops via Slack
        slackNotifier.notify("Payment Auth Issue: " + reason);
    }

    private void publishAuthExpiredEvent(PaymentAuthorization auth) {
        kafkaTemplate.send("payment.expired",
            auth.getId().toString(),
            new PaymentExpiredEvent(auth.getId(), auth.getOrderId(), Instant.now()));
    }

    private void publishReauthorizedEvent(PaymentAuthorization auth) {
        kafkaTemplate.send("payment.reauthorized",
            auth.getId().toString(),
            new PaymentReauthorizedEvent(auth.getId(), auth.getOrderId(), Instant.now()));
    }
}
```

---

### Q3: How do you implement partial captures?

**Design: Capture Record Tied to Shipment + Running Balance**

Each capture is a separate record tied to a shipment ID. Track the remaining authorized amount as the sum of captures is deducted.

```java
@Entity
@Table(name = "capture_records")
public class CaptureRecord {
    @Id
    private UUID id;

    @ManyToOne
    @JoinColumn(name = "authorization_id", nullable = false)
    private PaymentAuthorization authorization;

    @Column(nullable = false)
    private UUID shipmentId;  // Each shipment = one capture

    @Column(nullable = false)
    private BigDecimal captureAmount;

    @Column(nullable = false)
    private String status;  // PENDING, COMPLETED, FAILED

    @Column(nullable = true)
    private String gatewayTransactionId;  // Result from gateway

    @Column(nullable = false)
    private Instant createdAt = Instant.now();

    @Column(nullable = true)
    private Instant completedAt;
}

@Service
public class PartialCaptureHandler {

    private final AuthorizationRepository authRepo;
    private final CaptureRecordRepository captureRepo;
    private final PaymentGatewayRouter gatewayRouter;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final RedisTemplate<String, Object> redis;
    private final Logger log = LoggerFactory.getLogger(PartialCaptureHandler.class);

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public CaptureRecord capturePayment(UUID authorizationId, UUID shipmentId,
            BigDecimal captureAmount) throws PaymentException {

        // Check idempotency: if capture already exists for this shipment, return cached result
        String idempotencyKey = authorizationId + ":" + shipmentId;
        CaptureRecord existing = (CaptureRecord) redis.opsForValue()
            .get("capture:idempotency:" + idempotencyKey);
        if (existing != null && existing.getStatus().equals("COMPLETED")) {
            log.info("Returning cached capture result for {}", idempotencyKey);
            return existing;
        }

        // Fetch authorization with pessimistic lock
        PaymentAuthorization auth = authRepo.findByIdWithLock(authorizationId);
        if (auth == null) {
            throw new PaymentException("Authorization not found");
        }

        // Validate authorization state
        if (!auth.getStatus().matches("AUTHORIZED|PARTIALLY_CAPTURED")) {
            throw new PaymentException(
                "Cannot capture from status: " + auth.getStatus());
        }

        // Validate expiry
        if (auth.getExpiresAt().isBefore(Instant.now())) {
            auth.setStatus("EXPIRED");
            authRepo.save(auth);
            throw new PaymentException("Authorization has expired");
        }

        // Validate amount
        BigDecimal remaining = auth.getRemainingAuthority();
        if (captureAmount.compareTo(remaining) > 0) {
            throw new PaymentException(
                String.format("Capture amount %.2f exceeds remaining %.2f",
                    captureAmount, remaining));
        }

        // Create capture record
        var capture = new CaptureRecord();
        capture.setAuthorizationId(authorizationId);
        capture.setShipmentId(shipmentId);
        capture.setCaptureAmount(captureAmount);
        capture.setStatus("PENDING");
        capture.setCreatedAt(Instant.now());
        captureRepo.save(capture);

        // Call gateway to capture
        try {
            var captureRequest = new CaptureRequest(
                auth.getGatewayAuthorizationId(),
                captureAmount,
                auth.getCurrency(),
                Map.of("shipment_id", shipmentId.toString())
            );

            PaymentResult result = gatewayRouter.capturePayment(captureRequest);

            if (result.status() == PaymentStatus.CAPTURED) {
                // Update capture record
                capture.setStatus("COMPLETED");
                capture.setGatewayTransactionId(result.gatewayTransactionId());
                capture.setCompletedAt(Instant.now());
                captureRepo.save(capture);

                // Update authorization: deduct from authorized amount
                auth.setCapturedAmount(auth.getCapturedAmount().add(captureAmount));

                // Check if fully captured
                if (auth.getCapturedAmount().compareTo(auth.getAuthorizedAmount()) >= 0) {
                    auth.transitionToFullyCaptured();
                } else {
                    auth.transitionToPartiallyCaptured();
                }
                authRepo.save(auth);

                // Cache result for idempotency
                redis.opsForValue().set("capture:idempotency:" + idempotencyKey, capture,
                    Duration.ofHours(24));

                // Publish event
                publishCaptureCompletedEvent(auth, capture);

                log.info("Capture completed: auth={}, shipment={}, amount={}",
                    authorizationId, shipmentId, captureAmount);

                return capture;
            } else {
                throw new PaymentException("Gateway capture failed: " + result.message());
            }
        } catch (PaymentException e) {
            log.error("Capture failed: {}", e.getMessage());
            capture.setStatus("FAILED");
            capture.setErrorMessage(e.getMessage());
            captureRepo.save(capture);
            throw e;
        }
    }

    private void publishCaptureCompletedEvent(PaymentAuthorization auth, CaptureRecord capture) {
        var event = new PaymentCapturedEvent(
            auth.getId(), capture.getId(), capture.getCaptureAmount(),
            auth.getCurrency(), capture.getShipmentId(), Instant.now()
        );
        kafkaTemplate.send("payment.captured", auth.getId().toString(), event);
    }
}
```

---

### Q4: What happens if capture fails after item already shipped?

**Design: Retry Service + Manual Review Escalation**

If capture fails but the shipment has already been sent to the customer (irreversible), retry 3 times with exponential backoff. On final failure, escalate to manual review and notify ops.

```java
@Service
public class CaptureRetryService {

    private final CaptureRecordRepository captureRepo;
    private final AuthorizationRepository authRepo;
    private final PaymentGatewayRouter gatewayRouter;
    private final ManualReviewTicketRepository reviewTicketRepo;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final Logger log = LoggerFactory.getLogger(CaptureRetryService.class);

    private static final int MAX_RETRIES = 3;
    private static final long[] BACKOFF_MILLIS = {1000, 5000, 30000};  // 1s, 5s, 30s

    @Transactional
    public void retryFailedCapture(UUID captureId) throws PaymentException {
        CaptureRecord capture = captureRepo.findById(captureId)
            .orElseThrow(() -> new EntityNotFoundException("Capture not found"));

        if (!capture.getStatus().equals("FAILED")) {
            throw new IllegalStateException("Capture is not in FAILED status");
        }

        if (capture.getRetryCount() >= MAX_RETRIES) {
            log.error("Capture {} exceeded max retries, escalating to manual review", captureId);
            escalateToManualReview(capture);
            return;
        }

        PaymentAuthorization auth = authRepo.findById(capture.getAuthorizationId())
            .orElseThrow();

        try {
            var captureRequest = new CaptureRequest(
                auth.getGatewayAuthorizationId(),
                capture.getCaptureAmount(),
                auth.getCurrency(),
                Map.of("retry_count", String.valueOf(capture.getRetryCount()))
            );

            PaymentResult result = gatewayRouter.capturePayment(captureRequest);

            if (result.status() == PaymentStatus.CAPTURED) {
                capture.setStatus("COMPLETED");
                capture.setGatewayTransactionId(result.gatewayTransactionId());
                capture.setCompletedAt(Instant.now());
                captureRepo.save(capture);

                auth.setCapturedAmount(auth.getCapturedAmount().add(capture.getCaptureAmount()));
                authRepo.save(auth);

                log.info("Retry succeeded for capture {}", captureId);
            } else {
                throw new PaymentException("Retry failed: " + result.message());
            }
        } catch (PaymentException e) {
            capture.setRetryCount(capture.getRetryCount() + 1);
            if (capture.getRetryCount() < MAX_RETRIES) {
                // Schedule next retry
                capture.setStatus("RETRYING");
                captureRepo.save(capture);

                long delayMillis = BACKOFF_MILLIS[capture.getRetryCount() - 1];
                scheduleRetry(captureId, delayMillis);
                log.info("Scheduled retry {} for capture {} in {}ms",
                    capture.getRetryCount(), captureId, delayMillis);
            } else {
                escalateToManualReview(capture);
            }
        }
    }

    @Scheduled(fixedDelay = 1000)
    public void processRetryQueue() {
        // Query Redis for pending retries, execute them
        // This is handled via scheduled tasks or message queue in production
    }

    private void escalateToManualReview(CaptureRecord capture) {
        var ticket = new ManualReviewTicket();
        ticket.setCaptureId(capture.getId());
        ticket.setAuthorizationId(capture.getAuthorizationId());
        ticket.setShipmentId(capture.getShipmentId());
        ticket.setAmount(capture.getCaptureAmount());
        ticket.setReason("Capture failed after " + MAX_RETRIES + " retries");
        ticket.setStatus("OPEN");
        ticket.setPriority("HIGH");
        ticket.setCreatedAt(Instant.now());
        reviewTicketRepo.save(ticket);

        // Notify ops team
        slackNotifier.notifyWithPriority(
            "CRITICAL: Capture failed for shipment " + capture.getShipmentId() +
            ". Customer was charged via shipment but payment not captured. " +
            "Manual intervention required.",
            "high"
        );

        // Publish event for downstream processing
        kafkaTemplate.send("payment.capture.manual_review_required",
            capture.getId().toString(),
            new CaptureManualReviewEvent(capture.getId(), capture.getAuthorizationId(),
                capture.getCaptureAmount(), Instant.now()));
    }

    private void scheduleRetry(UUID captureId, long delayMillis) {
        // Use Kafka delayed queue or Spring Task Scheduler
        // For simplicity, use @Scheduled with Redis queue
        String queueKey = "retry:queue:capture:" + captureId;
        redis.opsForValue().set(queueKey, "pending",
            Duration.ofMillis(delayMillis));
    }
}
```

---

### Q5: How do you prevent double-capture?

**Design: Idempotency Key + Redis Cache**

Each capture request includes an idempotency key (e.g., authId + shipmentId). Check Redis cache first; if found and successful, return cached result. If not found, execute and cache.

```java
@Service
public class IdempotentCaptureGuard {

    private final RedisTemplate<String, Object> redis;
    private final CaptureRecordRepository captureRepo;
    private final Logger log = LoggerFactory.getLogger(IdempotentCaptureGuard.class);

    private static final String IDEMPOTENCY_KEY_PREFIX = "capture:idempotency:";
    private static final long CACHE_TTL_SECONDS = 86400;  // 24 hours

    public CaptureRecord executeIdempotentCapture(UUID authId, UUID shipmentId,
            CaptureExecutor executor) throws PaymentException {

        String idempotencyKey = generateIdempotencyKey(authId, shipmentId);

        // Check cache first
        String cachedResultJson = (String) redis.opsForValue()
            .get(IDEMPOTENCY_KEY_PREFIX + idempotencyKey);

        if (cachedResultJson != null) {
            log.info("Returning cached capture result for idempotency key: {}", idempotencyKey);
            return deserializeCapture(cachedResultJson);
        }

        // Execute capture
        CaptureRecord result = executor.execute();

        // Cache result
        if (result.getStatus().equals("COMPLETED")) {
            redis.opsForValue().set(
                IDEMPOTENCY_KEY_PREFIX + idempotencyKey,
                serializeCapture(result),
                Duration.ofSeconds(CACHE_TTL_SECONDS)
            );
        }

        return result;
    }

    // Also enforce DB-level idempotency: unique constraint on (authId, shipmentId)
    // If same shipment tries to capture twice, DB unique constraint prevents it

    private String generateIdempotencyKey(UUID authId, UUID shipmentId) {
        return authId + ":" + shipmentId;
    }

    private String serializeCapture(CaptureRecord capture) {
        ObjectMapper mapper = new ObjectMapper();
        return mapper.writeValueAsString(capture);
    }

    private CaptureRecord deserializeCapture(String json) {
        ObjectMapper mapper = new ObjectMapper();
        return mapper.readValue(json, CaptureRecord.class);
    }

    @FunctionalInterface
    public interface CaptureExecutor {
        CaptureRecord execute() throws PaymentException;
    }
}
```

---

### Q6: How do you handle split shipments with split captures?

**Design: One Authorization → Many Shipments → Many Captures**

Each shipment has its own capture record. They all draw from the same authorization pool. The remaining authorized amount decreases with each capture.

```java
@Service
public class SplitShipmentCaptureCoordinator {

    private final AuthorizationRepository authRepo;
    private final PartialCaptureHandler captureHandler;
    private final Logger log = LoggerFactory.getLogger(SplitShipmentCaptureCoordinator.class);

    // Example: Order for $100, split into 3 shipments: $40, $50, $10
    @Transactional
    public List<CaptureRecord> captureMultipleShipments(UUID authorizationId,
            List<ShipmentCaptureRequest> shipments) throws PaymentException {

        List<CaptureRecord> results = new ArrayList<>();

        for (ShipmentCaptureRequest shipment : shipments) {
            try {
                log.info("Capturing shipment {}: amount={}", shipment.shipmentId(),
                    shipment.amount());

                CaptureRecord capture = captureHandler.capturePayment(
                    authorizationId,
                    shipment.shipmentId(),
                    shipment.amount()
                );

                results.add(capture);
            } catch (PaymentException e) {
                log.error("Capture failed for shipment {}: {}",
                    shipment.shipmentId(), e.getMessage());

                // Decide: fail-fast or continue with next shipment?
                if (shipment.isCritical()) {
                    throw e;  // Fail-fast for critical shipments
                } else {
                    // Log and continue
                    results.add(new CaptureRecord(shipment.shipmentId(),
                        PaymentStatus.FAILED, e.getMessage()));
                }
            }
        }

        return results;
    }

    // Track which shipments are captured and pending
    public CaptureStatus getShipmentCaptureStatus(UUID authorizationId) {
        PaymentAuthorization auth = authRepo.findById(authorizationId).orElseThrow();
        List<CaptureRecord> captures = auth.getCaptures();

        BigDecimal capturedTotal = captures.stream()
            .filter(c -> c.getStatus().equals("COMPLETED"))
            .map(CaptureRecord::getCaptureAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal pendingTotal = captures.stream()
            .filter(c -> c.getStatus().equals("PENDING"))
            .map(CaptureRecord::getCaptureAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal failedTotal = captures.stream()
            .filter(c -> c.getStatus().equals("FAILED"))
            .map(CaptureRecord::getCaptureAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        return new CaptureStatus(
            auth.getAuthorizedAmount(),
            capturedTotal,
            pendingTotal,
            failedTotal,
            auth.getRemainingAuthority()
        );
    }

    public record ShipmentCaptureRequest(UUID shipmentId, BigDecimal amount, boolean isCritical) {}
    public record CaptureStatus(BigDecimal authorized, BigDecimal captured, BigDecimal pending,
                                 BigDecimal failed, BigDecimal remaining) {}
}
```

---

## 5. Microservices Breakdown

| Service | Responsibility | Tech Stack | Key APIs |
|---------|-----------------|------------|----------|
| **Authorization Service** | Authorize on order placement, manage auth state, expiry | Java 17 + Spring Boot 3, PostgreSQL | POST /authorize, GET /auth/{id}, PUT /auth/{id}/cancel |
| **Capture Service** | Capture payment when order ships, partial captures, retries | Java 17 + Spring Boot 3, PostgreSQL, Redis | POST /capture, GET /capture/{id}, POST /capture/{id}/retry |
| **Authorization Health Monitor** | Scan for expiring authorizations, auto-reauth, extend expiry | Java 17 + Spring Boot 3, PostgreSQL, Kafka | Scheduled job (no REST API) |
| **Manual Review Service** | Track failed captures needing human intervention | Java 17 + Spring Boot 3, PostgreSQL | GET /tickets, POST /tickets/{id}/resolve |
| **Event Consumer Service** | Consume Kafka events (payment.authorized, payment.captured) | Java 17 + Spring Boot 3, Kafka | Kafka consumer groups |

---

## 6. Database Design

### PostgreSQL Schema

```sql
-- Authorization entity (Phase 1)
CREATE TABLE payment_authorizations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL UNIQUE,
    customer_id UUID NOT NULL,
    status VARCHAR(50) NOT NULL,             -- AUTHORIZED, PARTIALLY_CAPTURED, etc.
    authorized_amount NUMERIC(12, 2) NOT NULL,
    captured_amount NUMERIC(12, 2) DEFAULT 0,
    currency VARCHAR(3) NOT NULL,
    gateway_name VARCHAR(50) NOT NULL,
    gateway_authorization_id VARCHAR(255) NOT NULL,
    authorized_at TIMESTAMP NOT NULL DEFAULT NOW(),
    expires_at TIMESTAMP NOT NULL,           -- +7 days
    fully_captured_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    reauthorized_at TIMESTAMP,
    version BIGINT DEFAULT 0,                -- Optimistic locking
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    CONSTRAINT authorized_amount_positive CHECK (authorized_amount > 0),
    CONSTRAINT captured_le_authorized CHECK (captured_amount <= authorized_amount)
);

CREATE INDEX idx_auth_order ON payment_authorizations(order_id);
CREATE INDEX idx_auth_expires_at ON payment_authorizations(expires_at) WHERE status != 'FULLY_CAPTURED';
CREATE INDEX idx_auth_status ON payment_authorizations(status);

-- Capture records (Phase 2, can be multiple per authorization)
CREATE TABLE capture_records (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    authorization_id UUID NOT NULL REFERENCES payment_authorizations(id),
    shipment_id UUID NOT NULL,
    capture_amount NUMERIC(12, 2) NOT NULL,
    status VARCHAR(50) NOT NULL,             -- PENDING, COMPLETED, FAILED, RETRYING
    gateway_transaction_id VARCHAR(255),
    retry_count INT DEFAULT 0,
    error_message TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    UNIQUE(authorization_id, shipment_id),  -- Prevent duplicate captures per shipment
    CONSTRAINT capture_amount_positive CHECK (capture_amount > 0)
);

CREATE INDEX idx_capture_auth ON capture_records(authorization_id);
CREATE INDEX idx_capture_shipment ON capture_records(shipment_id);
CREATE INDEX idx_capture_status ON capture_records(status) WHERE status != 'COMPLETED';

-- Audit trail
CREATE TABLE payment_authorization_audit (
    id BIGSERIAL PRIMARY KEY,
    authorization_id UUID NOT NULL,
    event_type VARCHAR(100),                 -- AUTHORIZED, PARTIALLY_CAPTURED, EXPIRED, etc.
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    details JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_audit_auth ON payment_authorization_audit(authorization_id);

-- Manual review tickets for failed captures
CREATE TABLE manual_review_tickets (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    authorization_id UUID REFERENCES payment_authorizations(id),
    capture_id UUID REFERENCES capture_records(id),
    shipment_id UUID,
    reason TEXT NOT NULL,
    status VARCHAR(50) NOT NULL,             -- OPEN, IN_PROGRESS, RESOLVED
    priority VARCHAR(20),                    -- LOW, MEDIUM, HIGH
    assigned_to VARCHAR(100),
    notes TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    resolved_at TIMESTAMP
);

CREATE INDEX idx_tickets_status ON manual_review_tickets(status);
CREATE INDEX idx_tickets_priority ON manual_review_tickets(priority) WHERE status = 'OPEN';
```

### MongoDB Collections

```javascript
// Authorization events (for analytics and event sourcing)
db.createCollection("authorization_events", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            properties: {
                _id: { bsonType: "objectId" },
                authorization_id: { bsonType: "string" },
                order_id: { bsonType: "string" },
                event_type: { bsonType: "string" },  // authorized, expired, reauthorized
                old_status: { bsonType: "string" },
                new_status: { bsonType: "string" },
                amount: { bsonType: "decimal" },
                currency: { bsonType: "string" },
                gateway: { bsonType: "string" },
                timestamp: { bsonType: "date" }
            }
        }
    }
});

db.authorization_events.createIndex({ timestamp: -1 });
db.authorization_events.createIndex({ authorization_id: 1 });
db.authorization_events.createIndex({ event_type: 1, timestamp: -1 });

// Capture events
db.createCollection("capture_events", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            properties: {
                _id: { bsonType: "objectId" },
                capture_id: { bsonType: "string" },
                authorization_id: { bsonType: "string" },
                shipment_id: { bsonType: "string" },
                event_type: { bsonType: "string" },  // captured, capture_failed, retry
                amount: { bsonType: "decimal" },
                status: { bsonType: "string" },
                retry_count: { bsonType: "int" },
                timestamp: { bsonType: "date" }
            }
        }
    }
});

db.capture_events.createIndex({ timestamp: -1 });
db.capture_events.createIndex({ capture_id: 1 });
```

---

## 7. Redis Data Structures

```java
// Authorization status cache (for quick lookups)
HSET auth:status:auth_123 status "PARTIALLY_CAPTURED" updated_at "2026-04-01T10:00:00Z"
EXPIRE auth:status:auth_123 86400

// Remaining authorized amount (for quick checks before capture)
SET auth:remaining:auth_123 "25.00" EX 86400

// Capture idempotency cache (24h TTL)
SET capture:idempotency:auth_123:shipment_456 "{\"id\":\"cap_789\",\"status\":\"COMPLETED\"}" EX 86400

// Manual review tickets queue
LPUSH manual_review:queue:high "ticket_001 ticket_002"

// Retry scheduled captures
ZADD capture:retry:queue 1700000000 "cap_123:retry:1"  # Sorted set with timestamps
ZADD capture:retry:queue 1700000300 "cap_124:retry:1"

// Authorization expiry watch list (for job scans)
ZADD auth:expiry:watch 1700086400 "auth_001"
ZADD auth:expiry:watch 1700172800 "auth_002"
```

**Key Commands:**

```java
// Check remaining authorized amount
Long remaining = redis.opsForValue().get("auth:remaining:" + authId);

// Cache idempotent capture result
redis.opsForValue().set(
    "capture:idempotency:" + authId + ":" + shipmentId,
    captureResult,
    Duration.ofHours(24)
);

// Schedule retry
redis.opsForZSet().add(
    "capture:retry:queue",
    captureId,
    System.currentTimeMillis() + delayMillis
);
```

---

## 8. Kafka Event Flow

### Topics

```
payment.authorization.requested
  ├─ AuthorizationRequestedEvent
  │   - order_id, customer_id, amount, currency
  │   - Consumed by: Authorization Service, Audit Service

payment.authorization.succeeded
  ├─ AuthorizationSucceededEvent
  │   - authorization_id, gateway_transaction_id, expires_at
  │   - Consumed by: Order Service, Audit Service, Analytics

payment.authorization.failed
  ├─ AuthorizationFailedEvent
  │   - order_id, reason, gateway_error_code
  │   - Consumed by: Order Service, Retry Service

payment.authorization.expired
  ├─ AuthorizationExpiredEvent
  │   - authorization_id, order_id, reason (no shipment / no capture)
  │   - Consumed by: Audit Service, Billing Service

payment.authorization.reauthorized
  ├─ AuthorizationReauthorizedEvent
  │   - original_auth_id, new_auth_id, expires_at
  │   - Consumed by: Order Service, Audit Service

payment.captured
  ├─ PaymentCapturedEvent
  │   - authorization_id, capture_id, shipment_id, amount
  │   - Consumed by: Accounting Service, Fulfillment Service, Audit Service

payment.capture.failed
  ├─ PaymentCaptureFailedEvent
  │   - capture_id, authorization_id, reason, retry_count
  │   - Consumed by: Retry Service, Manual Review Service

payment.capture.manual_review_required
  ├─ CaptureManualReviewEvent
  │   - capture_id, authorization_id, amount, reason
  │   - Consumed by: Manual Review Service, Alert Service
```

### Payload Examples

```json
// Event: payment.authorization.succeeded
{
  "event_id": "evt_auth_success_001",
  "authorization_id": "auth_123",
  "order_id": "order_456",
  "customer_id": "cust_789",
  "gateway_transaction_id": "pi_1234567abcdef",
  "authorized_amount": 99.99,
  "currency": "USD",
  "authorized_at": "2026-04-01T10:00:00Z",
  "expires_at": "2026-04-08T10:00:00Z",
  "gateway_name": "stripe"
}

// Event: payment.captured
{
  "event_id": "evt_capture_001",
  "capture_id": "cap_123",
  "authorization_id": "auth_123",
  "shipment_id": "ship_456",
  "captured_amount": 60.00,
  "currency": "USD",
  "gateway_transaction_id": "ch_1abcdef123",
  "captured_at": "2026-04-02T14:30:00Z"
}

// Event: payment.authorization.expired
{
  "event_id": "evt_auth_expired_001",
  "authorization_id": "auth_123",
  "order_id": "order_456",
  "reason": "No shipment initiated within 7 days",
  "expired_at": "2026-04-08T10:00:00Z"
}
```

---

## 9. Implementation Code

### Class 1: AuthorizationService

```java
package com.payment.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;
import com.payment.model.*;
import com.payment.repository.*;
import com.payment.event.*;

import java.math.BigDecimal;
import java.time.Instant;
import java.time.Duration;
import java.util.UUID;

@Service
public class AuthorizationService {

    private final AuthorizationRepository authRepo;
    private final PaymentGatewayRouter gatewayRouter;
    private final RedisTemplate<String, Object> redis;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final Logger log = LoggerFactory.getLogger(AuthorizationService.class);

    @Autowired
    public AuthorizationService(
            AuthorizationRepository authRepo,
            PaymentGatewayRouter gatewayRouter,
            RedisTemplate<String, Object> redis,
            KafkaTemplate<String, Object> kafkaTemplate) {
        this.authRepo = authRepo;
        this.gatewayRouter = gatewayRouter;
        this.redis = redis;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public PaymentAuthorization authorizePayment(AuthorizePaymentRequest request)
            throws PaymentException {

        String idempotencyKey = UUID.randomUUID().toString();

        try {
            // Call gateway to authorize
            var authRequest = new AuthorizeRequest(
                idempotencyKey,
                request.token(),
                request.amount(),
                request.currency(),
                request.customerId(),
                Map.of()
            );

            PaymentResult result = gatewayRouter.processPaymentWithFailover(authRequest);

            if (result.status() != PaymentStatus.AUTHORIZED) {
                publishAuthFailedEvent(request.orderId(), result.message());
                throw new PaymentException("Authorization failed: " + result.message());
            }

            // Create authorization record
            var auth = new PaymentAuthorization();
            auth.setOrderId(request.orderId());
            auth.setCustomerId(request.customerId());
            auth.setStatus("AUTHORIZED");
            auth.setAuthorizedAmount(request.amount());
            auth.setCapturedAmount(BigDecimal.ZERO);
            auth.setCurrency(request.currency());
            auth.setGatewayName(result.gatewayName());
            auth.setGatewayAuthorizationId(result.gatewayTransactionId());
            auth.setAuthorizedAt(Instant.now());
            auth.setExpiresAt(Instant.now().plus(Duration.ofDays(7)));
            auth.setCreatedAt(Instant.now());

            authRepo.save(auth);

            // Cache status for quick lookup
            redis.opsForValue().set("auth:status:" + auth.getId(), "AUTHORIZED",
                Duration.ofDays(7));
            redis.opsForValue().set("auth:remaining:" + auth.getId(), auth.getAuthorizedAmount().toString(),
                Duration.ofDays(7));

            // Publish success event
            publishAuthSucceededEvent(auth);

            log.info("Authorization created: id={}, amount={}, expires={}",
                auth.getId(), auth.getAuthorizedAmount(), auth.getExpiresAt());

            return auth;

        } catch (PaymentException e) {
            log.error("Authorization failed: {}", e.getMessage());
            throw e;
        }
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public void cancelAuthorization(UUID authorizationId) throws PaymentException {
        PaymentAuthorization auth = authRepo.findByIdWithLock(authorizationId);

        if (auth == null) {
            throw new EntityNotFoundException("Authorization not found");
        }

        if (auth.getStatus().equals("FULLY_CAPTURED")) {
            throw new IllegalStateException("Cannot cancel fully captured authorization");
        }

        auth.transitionToCancelled();
        auth.setCancelledAt(Instant.now());
        authRepo.save(auth);

        // Invalidate cache
        redis.delete("auth:status:" + authorizationId);
        redis.delete("auth:remaining:" + authorizationId);

        publishAuthCancelledEvent(auth);

        log.info("Authorization cancelled: id={}", authorizationId);
    }

    private void publishAuthSucceededEvent(PaymentAuthorization auth) {
        var event = new AuthorizationSucceededEvent(
            auth.getId(),
            auth.getOrderId(),
            auth.getAuthorizedAmount(),
            auth.getCurrency(),
            auth.getGatewayAuthorizationId(),
            auth.getExpiresAt(),
            Instant.now()
        );
        kafkaTemplate.send("payment.authorization.succeeded",
            auth.getId().toString(), event);
    }

    private void publishAuthFailedEvent(UUID orderId, String reason) {
        var event = new AuthorizationFailedEvent(
            orderId, reason, Instant.now()
        );
        kafkaTemplate.send("payment.authorization.failed",
            orderId.toString(), event);
    }

    private void publishAuthCancelledEvent(PaymentAuthorization auth) {
        var event = new AuthorizationCancelledEvent(
            auth.getId(), auth.getOrderId(), Instant.now()
        );
        kafkaTemplate.send("payment.authorization.cancelled",
            auth.getId().toString(), event);
    }
}
```

### Class 2: CaptureService

```java
package com.payment.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Isolation;
import org.springframework.transaction.annotation.Transactional;
import com.payment.model.*;
import com.payment.repository.*;
import com.payment.event.*;

import java.math.BigDecimal;
import java.time.Instant;
import java.time.Duration;
import java.util.UUID;

@Service
public class CaptureService {

    private final AuthorizationRepository authRepo;
    private final CaptureRecordRepository captureRepo;
    private final PaymentGatewayRouter gatewayRouter;
    private final RedisTemplate<String, Object> redis;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final Logger log = LoggerFactory.getLogger(CaptureService.class);

    @Autowired
    public CaptureService(
            AuthorizationRepository authRepo,
            CaptureRecordRepository captureRepo,
            PaymentGatewayRouter gatewayRouter,
            RedisTemplate<String, Object> redis,
            KafkaTemplate<String, Object> kafkaTemplate) {
        this.authRepo = authRepo;
        this.captureRepo = captureRepo;
        this.gatewayRouter = gatewayRouter;
        this.redis = redis;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public CaptureRecord capturePayment(UUID authorizationId, UUID shipmentId,
            BigDecimal captureAmount) throws PaymentException {

        // Check idempotency
        String idempotencyKey = authorizationId + ":" + shipmentId;
        String cachedKey = "capture:idempotency:" + idempotencyKey;
        Object cached = redis.opsForValue().get(cachedKey);
        if (cached != null) {
            log.info("Returning cached capture for {}", idempotencyKey);
            return deserializeCapture((String) cached);
        }

        // Fetch authorization with pessimistic lock
        PaymentAuthorization auth = authRepo.findByIdWithLock(authorizationId);
        if (auth == null) {
            throw new EntityNotFoundException("Authorization not found");
        }

        // Validate state
        if (!auth.getStatus().matches("AUTHORIZED|PARTIALLY_CAPTURED")) {
            throw new PaymentException("Cannot capture from status: " + auth.getStatus());
        }

        // Validate expiry
        if (auth.getExpiresAt().isBefore(Instant.now())) {
            auth.setStatus("EXPIRED");
            authRepo.save(auth);
            throw new PaymentException("Authorization has expired");
        }

        // Validate amount
        BigDecimal remaining = auth.getRemainingAuthority();
        if (captureAmount.compareTo(remaining) > 0) {
            throw new PaymentException(String.format(
                "Capture amount %.2f exceeds remaining %.2f", captureAmount, remaining));
        }

        // Create capture record
        var capture = new CaptureRecord();
        capture.setAuthorizationId(authorizationId);
        capture.setShipmentId(shipmentId);
        capture.setCaptureAmount(captureAmount);
        capture.setStatus("PENDING");
        capture.setCreatedAt(Instant.now());
        captureRepo.save(capture);

        try {
            // Call gateway to capture
            var captureRequest = new CaptureRequest(
                auth.getGatewayAuthorizationId(),
                captureAmount,
                auth.getCurrency(),
                Map.of("shipment_id", shipmentId.toString())
            );

            PaymentResult result = gatewayRouter.capturePayment(captureRequest);

            if (result.status() != PaymentStatus.CAPTURED) {
                throw new PaymentException("Gateway capture failed: " + result.message());
            }

            // Update capture record
            capture.setStatus("COMPLETED");
            capture.setGatewayTransactionId(result.gatewayTransactionId());
            capture.setCompletedAt(Instant.now());
            captureRepo.save(capture);

            // Update authorization
            auth.setCapturedAmount(auth.getCapturedAmount().add(captureAmount));

            if (auth.getCapturedAmount().compareTo(auth.getAuthorizedAmount()) >= 0) {
                auth.setStatus("FULLY_CAPTURED");
                auth.setFullyCaptureddAt(Instant.now());
            } else {
                auth.setStatus("PARTIALLY_CAPTURED");
            }
            authRepo.save(auth);

            // Cache result
            redis.opsForValue().set(cachedKey, serializeCapture(capture),
                Duration.ofHours(24));

            // Update Redis cache
            BigDecimal newRemaining = auth.getRemainingAuthority();
            if (newRemaining.compareTo(BigDecimal.ZERO) > 0) {
                redis.opsForValue().set("auth:remaining:" + authorizationId,
                    newRemaining.toString(), Duration.ofDays(7));
            }

            // Publish event
            publishCaptureCompletedEvent(auth, capture);

            log.info("Capture completed: auth={}, shipment={}, amount={}",
                authorizationId, shipmentId, captureAmount);

            return capture;

        } catch (PaymentException e) {
            log.error("Capture failed: {}", e.getMessage());
            capture.setStatus("FAILED");
            capture.setErrorMessage(e.getMessage());
            captureRepo.save(capture);

            publishCaptureFailedEvent(capture, auth);
            throw e;
        }
    }

    private void publishCaptureCompletedEvent(PaymentAuthorization auth, CaptureRecord capture) {
        var event = new PaymentCapturedEvent(
            capture.getId(),
            auth.getId(),
            capture.getShipmentId(),
            capture.getCaptureAmount(),
            auth.getCurrency(),
            capture.getGatewayTransactionId(),
            Instant.now()
        );
        kafkaTemplate.send("payment.captured", capture.getId().toString(), event);
    }

    private void publishCaptureFailedEvent(CaptureRecord capture, PaymentAuthorization auth) {
        var event = new PaymentCaptureFailedEvent(
            capture.getId(),
            auth.getId(),
            capture.getErrorMessage(),
            capture.getRetryCount(),
            Instant.now()
        );
        kafkaTemplate.send("payment.capture.failed", capture.getId().toString(), event);
    }

    private String serializeCapture(CaptureRecord capture) {
        ObjectMapper mapper = new ObjectMapper();
        return mapper.writeValueAsString(capture);
    }

    private CaptureRecord deserializeCapture(String json) {
        ObjectMapper mapper = new ObjectMapper();
        return mapper.readValue(json, CaptureRecord.class);
    }
}
```

### Class 3: AuthorizationExpiryJob (Scheduled)

```java
package com.payment.job;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.payment.model.*;
import com.payment.repository.*;
import com.payment.event.*;

import java.time.Instant;
import java.time.Duration;
import java.util.List;

@Service
public class AuthorizationExpiryJob {

    private final AuthorizationRepository authRepo;
    private final FulfillmentService fulfillmentService;
    private final PaymentGatewayRouter gatewayRouter;
    private final ManualReviewTicketRepository reviewTicketRepo;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final RedisTemplate<String, Object> redis;
    private final SlackNotifier slackNotifier;
    private final Logger log = LoggerFactory.getLogger(AuthorizationExpiryJob.class);

    @Autowired
    public AuthorizationExpiryJob(
            AuthorizationRepository authRepo,
            FulfillmentService fulfillmentService,
            PaymentGatewayRouter gatewayRouter,
            ManualReviewTicketRepository reviewTicketRepo,
            KafkaTemplate<String, Object> kafkaTemplate,
            RedisTemplate<String, Object> redis,
            SlackNotifier slackNotifier) {
        this.authRepo = authRepo;
        this.fulfillmentService = fulfillmentService;
        this.gatewayRouter = gatewayRouter;
        this.reviewTicketRepo = reviewTicketRepo;
        this.kafkaTemplate = kafkaTemplate;
        this.redis = redis;
        this.slackNotifier = slackNotifier;
    }

    @Scheduled(cron = "0 1 * * *")
    public void scanAndHandleExpiringAuthorizations() {
        Instant tomorrow = Instant.now().plus(Duration.ofDays(1));
        List<PaymentAuthorization> expiringAuths =
            authRepo.findByStatusAndExpiresAtBefore("AUTHORIZED", tomorrow);

        log.info("Found {} authorizations expiring within 24h", expiringAuths.size());

        for (PaymentAuthorization auth : expiringAuths) {
            try {
                handleExpiringAuthorization(auth);
            } catch (Exception e) {
                log.error("Error handling expiring authorization {}: {}",
                    auth.getId(), e.getMessage());
                createManualReviewTicket(auth, "Expiry handling failed: " + e.getMessage());
            }
        }
    }

    @Transactional
    public void handleExpiringAuthorization(PaymentAuthorization auth) throws PaymentException {
        boolean orderShipped = fulfillmentService.isOrderShipped(auth.getOrderId());

        if (orderShipped) {
            log.info("Order {} shipped, prioritizing capture for auth {}",
                auth.getOrderId(), auth.getId());
            // Capture is assumed to be pending; extend expiry
            auth.setExpiresAt(auth.getExpiresAt().plus(Duration.ofDays(1)));
            authRepo.save(auth);
        } else {
            log.info("Order {} not shipped, re-authorizing", auth.getOrderId());
            reauthorizePayment(auth);
        }
    }

    private void reauthorizePayment(PaymentAuthorization auth) throws PaymentException {
        try {
            var authRequest = new AuthorizeRequest(
                auth.getId() + ":reauth:" + System.currentTimeMillis(),
                "use_saved_token",
                auth.getAuthorizedAmount(),
                auth.getCurrency(),
                auth.getCustomerId(),
                Map.of("original_auth_id", auth.getId().toString())
            );

            PaymentResult result = gatewayRouter.processPaymentWithFailover(authRequest);

            if (result.status() == PaymentStatus.AUTHORIZED) {
                // Mark old auth as replaced
                auth.setStatus("EXPIRED");
                authRepo.save(auth);

                // Create new authorization
                var newAuth = new PaymentAuthorization();
                newAuth.setOrderId(auth.getOrderId());
                newAuth.setAuthorizedAmount(auth.getAuthorizedAmount());
                newAuth.setCurrency(auth.getCurrency());
                newAuth.setGatewayName(auth.getGatewayName());
                newAuth.setGatewayAuthorizationId(result.gatewayTransactionId());
                newAuth.setAuthorizedAt(Instant.now());
                newAuth.setExpiresAt(Instant.now().plus(Duration.ofDays(7)));
                newAuth.setStatus("AUTHORIZED");
                newAuth.setReauthorizedAt(Instant.now());
                authRepo.save(newAuth);

                // Cache new auth
                redis.opsForValue().set("auth:status:" + newAuth.getId(), "AUTHORIZED",
                    Duration.ofDays(7));

                publishReauthorizedEvent(newAuth);

                log.info("Re-authorization succeeded for order {}", auth.getOrderId());
            } else {
                throw new PaymentException("Re-authorization failed: " + result.message());
            }
        } catch (Exception e) {
            log.error("Re-authorization exception for order {}", auth.getOrderId(), e);
            createManualReviewTicket(auth, "Re-authorization failed: " + e.getMessage());
        }
    }

    private void createManualReviewTicket(PaymentAuthorization auth, String reason) {
        var ticket = new ManualReviewTicket();
        ticket.setAuthorizationId(auth.getId());
        ticket.setOrderId(auth.getOrderId());
        ticket.setReason(reason);
        ticket.setStatus("OPEN");
        ticket.setPriority("HIGH");
        ticket.setCreatedAt(Instant.now());
        reviewTicketRepo.save(ticket);

        slackNotifier.notifyWithPriority(
            "Payment Authorization Issue: " + reason, "high");
    }

    private void publishReauthorizedEvent(PaymentAuthorization auth) {
        var event = new AuthorizationReauthorizedEvent(
            auth.getId(), auth.getOrderId(), auth.getExpiresAt(), Instant.now()
        );
        kafkaTemplate.send("payment.authorization.reauthorized",
            auth.getId().toString(), event);
    }
}
```

### Class 4: CaptureRetryService

```java
package com.payment.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.payment.model.*;
import com.payment.repository.*;
import com.payment.event.*;

import java.time.Instant;
import java.time.Duration;
import java.util.UUID;

@Service
public class CaptureRetryService {

    private final CaptureRecordRepository captureRepo;
    private final AuthorizationRepository authRepo;
    private final PaymentGatewayRouter gatewayRouter;
    private final ManualReviewTicketRepository reviewTicketRepo;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final RedisTemplate<String, Object> redis;
    private final SlackNotifier slackNotifier;
    private final Logger log = LoggerFactory.getLogger(CaptureRetryService.class);

    private static final int MAX_RETRIES = 3;
    private static final long[] BACKOFF_MILLIS = {1000, 5000, 30000};

    @Autowired
    public CaptureRetryService(
            CaptureRecordRepository captureRepo,
            AuthorizationRepository authRepo,
            PaymentGatewayRouter gatewayRouter,
            ManualReviewTicketRepository reviewTicketRepo,
            KafkaTemplate<String, Object> kafkaTemplate,
            RedisTemplate<String, Object> redis,
            SlackNotifier slackNotifier) {
        this.captureRepo = captureRepo;
        this.authRepo = authRepo;
        this.gatewayRouter = gatewayRouter;
        this.reviewTicketRepo = reviewTicketRepo;
        this.kafkaTemplate = kafkaTemplate;
        this.redis = redis;
        this.slackNotifier = slackNotifier;
    }

    @Transactional
    public void retryFailedCapture(UUID captureId) throws PaymentException {
        CaptureRecord capture = captureRepo.findById(captureId)
            .orElseThrow(() -> new EntityNotFoundException("Capture not found"));

        if (!capture.getStatus().equals("FAILED")) {
            throw new IllegalStateException("Capture not in FAILED status");
        }

        if (capture.getRetryCount() >= MAX_RETRIES) {
            log.error("Capture {} exceeded max retries, escalating", captureId);
            escalateToManualReview(capture);
            return;
        }

        PaymentAuthorization auth = authRepo.findById(capture.getAuthorizationId()).orElseThrow();

        try {
            var captureRequest = new CaptureRequest(
                auth.getGatewayAuthorizationId(),
                capture.getCaptureAmount(),
                auth.getCurrency(),
                Map.of("retry_count", String.valueOf(capture.getRetryCount()))
            );

            PaymentResult result = gatewayRouter.capturePayment(captureRequest);

            if (result.status() == PaymentStatus.CAPTURED) {
                capture.setStatus("COMPLETED");
                capture.setGatewayTransactionId(result.gatewayTransactionId());
                capture.setCompletedAt(Instant.now());
                captureRepo.save(capture);

                auth.setCapturedAmount(auth.getCapturedAmount().add(capture.getCaptureAmount()));
                authRepo.save(auth);

                log.info("Retry succeeded for capture {}", captureId);
            } else {
                throw new PaymentException("Retry failed: " + result.message());
            }
        } catch (PaymentException e) {
            capture.setRetryCount(capture.getRetryCount() + 1);

            if (capture.getRetryCount() < MAX_RETRIES) {
                capture.setStatus("RETRYING");
                captureRepo.save(capture);

                long delayMillis = BACKOFF_MILLIS[capture.getRetryCount() - 1];
                scheduleRetry(captureId, delayMillis);
                log.info("Scheduled retry {} for capture {} in {}ms",
                    capture.getRetryCount(), captureId, delayMillis);
            } else {
                escalateToManualReview(capture);
            }
        }
    }

    private void escalateToManualReview(CaptureRecord capture) {
        var ticket = new ManualReviewTicket();
        ticket.setCaptureId(capture.getId());
        ticket.setAuthorizationId(capture.getAuthorizationId());
        ticket.setShipmentId(capture.getShipmentId());
        ticket.setAmount(capture.getCaptureAmount());
        ticket.setReason("Capture failed after " + MAX_RETRIES + " retries");
        ticket.setStatus("OPEN");
        ticket.setPriority("HIGH");
        ticket.setCreatedAt(Instant.now());
        reviewTicketRepo.save(ticket);

        slackNotifier.notifyWithPriority(
            "CRITICAL: Capture failed for shipment " + capture.getShipmentId() +
            ". Customer shipped but payment not captured. Manual intervention required.",
            "high"
        );

        var event = new CaptureManualReviewEvent(
            capture.getId(), capture.getAuthorizationId(),
            capture.getCaptureAmount(), Instant.now()
        );
        kafkaTemplate.send("payment.capture.manual_review_required",
            capture.getId().toString(), event);
    }

    private void scheduleRetry(UUID captureId, long delayMillis) {
        long retryTimestamp = System.currentTimeMillis() + delayMillis;
        redis.opsForZSet().add("capture:retry:queue", captureId.toString(), retryTimestamp);
    }
}
```

### Class 5: IdempotentCaptureGuard

```java
package com.payment.service;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.time.Duration;
import java.util.UUID;

@Service
public class IdempotentCaptureGuard {

    private final RedisTemplate<String, Object> redis;
    private final Logger log = LoggerFactory.getLogger(IdempotentCaptureGuard.class);

    private static final String IDEMPOTENCY_KEY_PREFIX = "capture:idempotency:";
    private static final long CACHE_TTL_SECONDS = 86400;

    @Autowired
    public IdempotentCaptureGuard(RedisTemplate<String, Object> redis) {
        this.redis = redis;
    }

    public CaptureRecord executeIdempotentCapture(UUID authId, UUID shipmentId,
            CaptureExecutor executor) throws PaymentException {

        String idempotencyKey = generateIdempotencyKey(authId, shipmentId);
        String cachedResultJson = (String) redis.opsForValue()
            .get(IDEMPOTENCY_KEY_PREFIX + idempotencyKey);

        if (cachedResultJson != null) {
            log.info("Returning cached capture for idempotency key: {}", idempotencyKey);
            return deserializeCapture(cachedResultJson);
        }

        // Execute capture
        CaptureRecord result = executor.execute();

        // Cache result
        if (result.getStatus().equals("COMPLETED")) {
            redis.opsForValue().set(
                IDEMPOTENCY_KEY_PREFIX + idempotencyKey,
                serializeCapture(result),
                Duration.ofSeconds(CACHE_TTL_SECONDS)
            );
        }

        return result;
    }

    private String generateIdempotencyKey(UUID authId, UUID shipmentId) {
        return authId + ":" + shipmentId;
    }

    private String serializeCapture(CaptureRecord capture) {
        ObjectMapper mapper = new ObjectMapper();
        return mapper.writeValueAsString(capture);
    }

    private CaptureRecord deserializeCapture(String json) {
        ObjectMapper mapper = new ObjectMapper();
        return mapper.readValue(json, CaptureRecord.class);
    }

    @FunctionalInterface
    public interface CaptureExecutor {
        CaptureRecord execute() throws PaymentException;
    }
}
```

### Class 6: SplitShipmentCaptureCoordinator

```java
package com.payment.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.payment.model.*;
import com.payment.repository.*;

import java.math.BigDecimal;
import java.util.*;

@Service
public class SplitShipmentCaptureCoordinator {

    private final AuthorizationRepository authRepo;
    private final CaptureService captureService;
    private final Logger log = LoggerFactory.getLogger(SplitShipmentCaptureCoordinator.class);

    @Autowired
    public SplitShipmentCaptureCoordinator(
            AuthorizationRepository authRepo,
            CaptureService captureService) {
        this.authRepo = authRepo;
        this.captureService = captureService;
    }

    @Transactional
    public List<CaptureRecord> captureMultipleShipments(UUID authorizationId,
            List<ShipmentCaptureRequest> shipments) throws PaymentException {

        List<CaptureRecord> results = new ArrayList<>();

        for (ShipmentCaptureRequest shipment : shipments) {
            try {
                log.info("Capturing shipment {}: amount={}",
                    shipment.shipmentId(), shipment.amount());

                CaptureRecord capture = captureService.capturePayment(
                    authorizationId,
                    shipment.shipmentId(),
                    shipment.amount()
                );

                results.add(capture);
            } catch (PaymentException e) {
                log.error("Capture failed for shipment {}: {}",
                    shipment.shipmentId(), e.getMessage());

                if (shipment.isCritical()) {
                    throw e;
                } else {
                    results.add(new CaptureRecord(shipment.shipmentId(),
                        "FAILED", e.getMessage()));
                }
            }
        }

        return results;
    }

    public CaptureStatus getShipmentCaptureStatus(UUID authorizationId) {
        PaymentAuthorization auth = authRepo.findById(authorizationId).orElseThrow();
        List<CaptureRecord> captures = auth.getCaptures();

        BigDecimal capturedTotal = captures.stream()
            .filter(c -> c.getStatus().equals("COMPLETED"))
            .map(CaptureRecord::getCaptureAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal pendingTotal = captures.stream()
            .filter(c -> c.getStatus().equals("PENDING"))
            .map(CaptureRecord::getCaptureAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal failedTotal = captures.stream()
            .filter(c -> c.getStatus().equals("FAILED"))
            .map(CaptureRecord::getCaptureAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        return new CaptureStatus(
            auth.getAuthorizedAmount(),
            capturedTotal,
            pendingTotal,
            failedTotal,
            auth.getRemainingAuthority()
        );
    }

    public record ShipmentCaptureRequest(UUID shipmentId, BigDecimal amount, boolean isCritical) {}
    public record CaptureStatus(BigDecimal authorized, BigDecimal captured, BigDecimal pending,
                                 BigDecimal failed, BigDecimal remaining) {}
}
```

---

## 10. Failure Scenarios & Mitigations

| Failure Scenario | Detection Method | Mitigation Strategy |
|---|---|---|
| **Authorization gateway timeout** | HTTP timeout (5s) | Retry with exponential backoff; fail-fast after 3 retries; return 503 to client |
| **Authorization expires before capture** | Scheduled job scans expirations | Re-authorize automatically 6 days after initial auth; extend expiry +1 day if shipment pending |
| **Capture fails after shipment sent** | Capture service receives error | Retry 3x with exponential backoff (1s, 5s, 30s); escalate to manual review on final failure |
| **Duplicate capture request (retry)** | Idempotency key + Redis cache | Check Redis cache before executing; return cached result within 24h TTL; DB unique constraint (authId, shipmentId) |
| **Authorization expires during capture** | Check expiry before capture | Extend expiry +1 day if capture pending; fail capture if already expired |
| **Partial capture exceeds authorized amount** | Validate amount vs remaining in DB | Throw exception; reject capture; suggest correct amount to client |
| **Redis cache miss for idempotency** | All cache-dependent service | Fall back to DB check (UNIQUE constraint); slow path but correct result |
| **PostgreSQL connection pool exhausted** | Max pool size reached | Implement connection pool monitoring; fail gracefully with 503; alert DevOps |
| **Kafka broker down (event publishing)** | Kafka send timeout | Log to PostgreSQL audit table as fallback; retry publishing periodically with exponential backoff |
| **Split shipment conflict (two shipments ship simultaneously)** | DB unique constraint (authId, shipmentId) | Pessimistic row lock in capture service prevents concurrent updates; second request fails, client retries |
| **Manual review ticket backlog** | Monitor ticket queue | Escalate to management; implement SLA alerts if >10 HIGH priority tickets |

---

## 11. Scaling Strategy

### Horizontal Scaling

1. **Authorization Service:** 3–5 instances behind load balancer; each handles ~3 TPS.
2. **Capture Service:** 2–3 instances; I/O-bound (calls gateway), benefits from parallelization.
3. **Authorization Expiry Job:** Single instance (distributed lock via Redis prevents concurrent runs).
4. **Capture Retry Service:** Single instance; processes Redis sorted set of retry candidates.

### Vertical Scaling

- **PostgreSQL:** Increase `max_connections` to 300+; use read replicas for status queries.
- **Redis:** Grow to 500MB+ for cache (authorizations + captures + idempotency cache).
- **Kafka:** Increase brokers to 3+ for fault tolerance; set retention 7 days.

### Database Optimization

```sql
-- Partition authorizations by month for faster scans
ALTER TABLE payment_authorizations PARTITION BY RANGE (YEAR(created_at), MONTH(created_at));

-- Archive old records to cold storage (>1 year)
CREATE TABLE authorizations_archive AS
SELECT * FROM payment_authorizations WHERE created_at < NOW() - INTERVAL 1 YEAR;
```

---

## 12. Monitoring & Observability

### Key Metrics

```
Authorization Metrics:
  - auth.authorization.latency_ms (p50, p95, p99)
  - auth.authorization.success_rate
  - auth.active_authorizations_count
  - auth.expired_authorizations_count
  - auth.reauthorization_attempts

Capture Metrics:
  - capture.capture.latency_ms
  - capture.capture.success_rate
  - capture.partial_captures_count
  - capture.retry_count
  - capture.manual_review_queue_length

System Health:
  - authorization.expiry_upcoming (24h window)
  - capture.failed_awaiting_retry
  - idempotency_cache.hit_rate
  - postgres.connection_pool.active
```

### Alerts

```yaml
AuthorizationFailureRate:
  expr: rate(auth_failures[5m]) > 0.05
  for: 5m
  action: page on-call

CaptureManualReviewBacklog:
  expr: manual_review_tickets > 5
  for: 30m
  action: alert to payment ops

AuthExpiredWithoutCapture:
  expr: authorizations_expired_no_capture > 10
  for: 1h
  action: email finance team

CaptureRetryQueueSize:
  expr: capture_retry_queue_size > 100
  for: 10m
  action: alert SRE to investigate
```

---

## 13. Summary Cheat Sheet

| Decision | Choice | Rationale |
|---|---|---|
| **Auth State Model** | State machine with pessimistic DB locks | Prevents race conditions on concurrent captures; strong consistency |
| **Authorization Expiry** | 7-day expiry + scheduled re-auth job | Matches typical fulfillment timelines; automatic re-auth for delayed shipments |
| **Partial Captures** | One auth → many captures (UNIQUE(authId, shipmentId)) | Supports split shipments; prevents duplicate captures per shipment |
| **Capture Failures** | Retry 3x + manual review escalation | Balances automation with safety; ops visibility for critical failures |
| **Idempotency** | Redis cache (24h TTL) + DB unique constraint | Prevents double-charge; cached result serves retries; falls back to DB |
| **State Transitions** | Pessimistic locks (SELECT FOR UPDATE) | Eliminates race conditions; slower but correct for payment consistency |
| **Reconciliation** | Daily batch job matching internal vs gateway records | Detects discrepancies; enables quick remediation |
| **Scaling** | 3–5 auth service instances; 2–3 capture instances; single retry/expiry jobs | Handles 100K auths/day + 80K captures/day at 99.9% uptime |
| **Database** | PostgreSQL (authorizations, captures) + MongoDB (audit logs) | ACID guarantees for payment state; flexible schema for events |
| **Caching** | Redis for status lookups, remaining amounts, idempotency cache | Sub-ms latency for authorization checks; TTL-based expiration |
| **Event Bus** | Kafka for all state transitions (auth, capture, expiry, reauth) | Decouples services; enables audit trail + downstream processing (accounting) |

---

**End of File**
