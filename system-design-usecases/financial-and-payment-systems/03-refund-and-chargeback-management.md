---
title: Refund and Chargeback Management
layout: default
---

# Refund and Chargeback Management — Deep Dive Design

> **Scenario:** Full and partial refunds, refunds to original payment method, handle chargebacks from card companies, chargeback dispute workflow, track refund status, reconciliation with accounting system.
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

- **Full and Partial Refunds:** Refund entire order or specific items.
- **Refund to Original Payment Method:** Refund credit card → credit card; gift card → gift card; wallet → wallet.
- **Partial Refunds on Split Payments:** If order paid via $70 credit card + $30 gift card, refund $20 → pro-rata refund ($14 CC, $6 GC).
- **Duplicate Refund Prevention:** Same refund request twice should not process twice.
- **Chargeback Workflow:** Receive chargeback notification → collect evidence → submit dispute → track outcome (WON/LOST).
- **Refund State Machine:** PENDING → PROCESSING → COMPLETED / FAILED → RETRYING → MANUAL_REVIEW.
- **Accounting Integration:** Post refund journal entries to ledger via Kafka; track GL accounts (e.g., 4110 Refunds).
- **Reconciliation:** Match internal refunds vs external processor settlement files.
- **Audit Trail:** Full history of who, what, when, why for every refund.
- **Dispute Evidence Collection:** Attach order details, shipping proof, customer communication to chargeback case.

### Non-Functional Requirements

- **Latency:** Refund initiation < 2s (p99); refund processing < 30 minutes.
- **Throughput:** 20% of 100K authorizations = ~20K refunds/day (~0.23 TPS).
- **Consistency:** At-most-once refund processing; strong consistency on duplicate checks.
- **Availability:** 99.9% SLA; tolerate gateway failures with retries.
- **Data Retention:** 7 years for PCI audit; chargeback evidence retention varies by card network (typically 2–3 years).

### Out of Scope

- Fraud prevention on refund requests (risk team reviews).
- Advanced chargeback prediction models (ML team).
- PCI compliance mechanisms beyond token handling.

---

## 2. Capacity Estimation

```
CAPACITY PLANNING TABLE

┌─────────────────────────────────────────────────────────────────┐
│ Metric                          │ Calculation       │ Value       │
├─────────────────────────────────────────────────────────────────┤
│ Authorizations per day          │ Given            │ 100,000     │
│ Refund rate (%)                 │ Typical e-comm   │ 20%         │
│ Refunds per day                 │ 100K * 0.2       │ 20,000      │
│ TPS (nominal)                   │ 20K / 86400s     │ 0.23 TPS    │
│ TPS (peak, 10x)                 │ 0.23 * 10        │ 2.3 TPS     │
│                                 │                  │             │
│ Refund DB rows/year             │ 20K * 365        │ 7.3M rows   │
│ Refund row size                 │ Estimate         │ 1.5 KB      │
│ PostgreSQL storage/year         │ 7.3M * 1.5KB     │ 11 GB       │
│ With indices (2x)               │ 11GB * 2         │ 22 GB       │
│                                 │                  │             │
│ Chargeback rate (% of txns)     │ Typical Visa     │ 0.1%        │
│ Chargebacks per day             │ 100K * 0.001     │ 100/day     │
│ Chargeback cases/year           │ 100 * 365        │ 36,500      │
│ Per case size (attachments)     │ Estimate         │ 5 MB        │
│ Chargeback evidence storage     │ 36.5K * 5MB      │ 182.5 GB    │
│                                 │                  │             │
│ Refund idempotency cache (7d)   │ 20K * 7 days     │ 140K keys   │
│ Cache entry size               │ ~500B per entry  │ 70 MB       │
│                                 │                  │             │
│ Kafka event volume/day          │ Refunds + disputes│ 40K msgs    │
│ Avg event size                 │ Estimate         │ 1 KB        │
│ Daily Kafka volume              │ 40K * 1KB        │ 40 MB       │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                    ORDER / FULFILLMENT SERVICE                      │
│        (Customer returns item, requests refund, initiates RMA)       │
└────────────────┬─────────────────────────────────────────────────────┘
                 │ Refund.Requested event
                 ▼
┌────────────────────────────────────────────────────────────────────┐
│                     REFUND SERVICE (Core)                           │
│                                                                      │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ RefundHandler                                                │  │
│  │ • Validate refund amount vs original payment                 │  │
│  │ • Check for duplicates (idempotency)                         │  │
│  │ • Determine refund strategy (CC / GC / Wallet)              │  │
│  │ • Create Refund entity in PENDING state                      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│           │                                                          │
│           ▼                                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ RefundStrategyFactory                                        │  │
│  │ ├─ CreditCardRefundStrategy (call Stripe.refund)            │  │
│  │ ├─ GiftCardRefundStrategy (restore balance)                 │  │
│  │ └─ DigitalWalletRefundStrategy (Apple Pay, Google Pay)      │  │
│  └──────────────────────────────────────────────────────────────┘  │
│           │                                                          │
│           ▼                                                          │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │ Refund State Machine                                         │  │
│  │ PENDING → PROCESSING → COMPLETED (or FAILED)                │  │
│  │           ↓                                                  │  │
│  │         RETRYING → MANUAL_REVIEW                            │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────┬─────────────────────────────────────────────────────┘
                 │ Publish: refund.initiated
                 ▼
        ┌──────────────────┐
        │   Kafka Topics   │
        │ refund.initiated │
        │ refund.completed │
        │ refund.failed    │
        │ chargeback.*     │
        └─────┬────────────┘
              │
     ┌────────┴──────────────┬─────────────────┐
     ▼                       ▼                 ▼
  ┌───────────────┐  ┌──────────────────┐ ┌──────────────────┐
  │ Accounting    │  │ Reconciliation   │ │ Manual Review    │
  │ Service       │  │ Service          │ │ Service          │
  │ (Consumer)    │  │                  │ │                  │
  │ Posts to GL   │  │ Match internal   │ │ Track failed     │
  │ Account 4110  │  │ vs processor     │ │ refunds, alerts  │
  └───────────────┘  └──────────────────┘ └──────────────────┘
       │
       ▼
  ┌─────────────────────────────────────────────┐
  │    PostgreSQL + MongoDB                      │
  │ • Refund records                             │
  │ • Chargeback cases                           │
  │ • Dispute evidence                           │
  │ • Audit trail                                │
  └─────────────────────────────────────────────┘

CHARGEBACK FLOW (separate from refunds):

┌────────────────────────────────────────────┐
│  Card Network (Visa, MasterCard)            │
│  Sends chargeback notification              │
└────────────────┬─────────────────────────────┘
                 │ Webhook: chargeback.received
                 ▼
┌────────────────────────────────────────────┐
│  Chargeback Dispute Service                 │
│                                             │
│  ┌────────────────────────────────────────┐│
│  │ ChargebackHandler                      ││
│  │ • Create DisputeCase (RECEIVED)        ││
│  │ • Collect evidence:                    ││
│  │   - Order details, shipping proof      ││
│  │   - Customer communication             ││
│  │   - Refund proof (if applicable)       ││
│  │ • Transition to EVIDENCE_COLLECTED     ││
│  └────────────────────────────────────────┘│
│           │                                 │
│           ▼                                 │
│  ┌────────────────────────────────────────┐│
│  │ DisputeSubmitter (manual or auto)      ││
│  │ Upload to card network portal          ││
│  │ Transition to DISPUTE_SUBMITTED        ││
│  └────────────────────────────────────────┘│
│           │                                 │
│           ▼                                 │
│  ┌────────────────────────────────────────┐│
│  │ Webhook listener: chargeback.resolved  ││
│  │ Transition to WON or LOST              ││
│  │ Publish outcome event                  ││
│  └────────────────────────────────────────┘│
└────────────────┬─────────────────────────────┘
                 │ Kafka: chargeback.won / chargeback.lost
                 ▼
           Alert ops team
```

---

## 4. Core Design Questions Answered

### Q1: How do you process refunds to different payment types?

**Design: Strategy Pattern + Polymorphic Refund Execution**

Define a `RefundStrategy` interface with implementations for each payment method. The factory selects the correct strategy based on original payment type.

```java
public interface RefundStrategy {
    RefundResult executeRefund(RefundRequest request) throws PaymentException;
    boolean supports(PaymentMethodType type);
}

@Service
public class CreditCardRefundStrategy implements RefundStrategy {

    private final StripeClient stripeClient;
    private final Logger log = LoggerFactory.getLogger(CreditCardRefundStrategy.class);

    @Override
    public RefundResult executeRefund(RefundRequest request) throws PaymentException {
        try {
            var params = new StripeRefundParams()
                .setCharge(request.gatewayTransactionId())
                .setAmount(request.amount().multiply(BigDecimal.valueOf(100)).longValue())
                .setReason("requested_by_customer");

            Refund refund = stripeClient.createRefund(params);

            return new RefundResult(
                refund.id(),
                RefundStatus.COMPLETED,
                "Refund processed successfully",
                Instant.now()
            );
        } catch (StripeException e) {
            log.error("Stripe refund failed: {}", e.getMessage());
            throw new PaymentException("Credit card refund failed", e);
        }
    }

    @Override
    public boolean supports(PaymentMethodType type) {
        return type == PaymentMethodType.CREDIT_CARD;
    }
}

@Service
public class GiftCardRefundStrategy implements RefundStrategy {

    private final GiftCardService giftCardService;
    private final Logger log = LoggerFactory.getLogger(GiftCardRefundStrategy.class);

    @Override
    public RefundResult executeRefund(RefundRequest request) throws PaymentException {
        try {
            GiftCard card = giftCardService.findCard(request.giftCardNumber());

            if (card.getStatus().equals("REVOKED") || card.getStatus().equals("EXPIRED")) {
                throw new PaymentException("Gift card is " + card.getStatus());
            }

            // Restore balance
            card.setCurrentBalance(card.getCurrentBalance().add(request.amount()));
            giftCardService.saveCard(card);

            return new RefundResult(
                "gc_" + card.getId(),
                RefundStatus.COMPLETED,
                "Gift card balance restored",
                Instant.now()
            );
        } catch (Exception e) {
            log.error("Gift card refund failed: {}", e.getMessage());
            throw new PaymentException("Gift card refund failed", e);
        }
    }

    @Override
    public boolean supports(PaymentMethodType type) {
        return type == PaymentMethodType.GIFT_CARD;
    }
}

@Service
public class RefundStrategyFactory {

    private final List<RefundStrategy> strategies;

    @Autowired
    public RefundStrategyFactory(List<RefundStrategy> strategies) {
        this.strategies = strategies;
    }

    public RefundStrategy getStrategy(PaymentMethodType type) {
        return strategies.stream()
            .filter(s -> s.supports(type))
            .findFirst()
            .orElseThrow(() -> new PaymentException("No strategy for type: " + type));
    }
}
```

---

### Q2: How do you handle partial refunds across split payments?

**Design: Pro-Rata Refund Distribution**

If order paid via $70 credit card + $30 gift card, and customer requests $50 refund:
- Refund $35 to credit card (70% of $50)
- Refund $15 to gift card (30% of $50)

```java
@Service
public class PartialRefundCalculator {

    private final PaymentRepository paymentRepo;
    private final Logger log = LoggerFactory.getLogger(PartialRefundCalculator.class);

    public List<RefundAllocation> calculateProRataRefund(UUID orderId, BigDecimal refundAmount)
            throws PaymentException {

        // Fetch all payments for this order
        List<Payment> payments = paymentRepo.findByOrderId(orderId);

        if (payments.isEmpty()) {
            throw new PaymentException("No payments found for order");
        }

        // Calculate total authorized amount
        BigDecimal totalAmount = payments.stream()
            .map(Payment::getAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        if (refundAmount.compareTo(totalAmount) > 0) {
            throw new PaymentException("Refund amount exceeds total order amount");
        }

        List<RefundAllocation> allocations = new ArrayList<>();

        for (Payment payment : payments) {
            // Pro-rata: (payment amount / total amount) * refund amount
            BigDecimal proportion = payment.getAmount()
                .divide(totalAmount, 4, RoundingMode.HALF_UP);

            BigDecimal allocationAmount = refundAmount
                .multiply(proportion)
                .setScale(2, RoundingMode.HALF_UP);

            allocations.add(new RefundAllocation(
                payment.getId(),
                payment.getPaymentMethodType(),
                payment.getGatewayTransactionId(),
                allocationAmount
            ));

            log.info("Pro-rata allocation: type={}, amount={}, proportion={}%",
                payment.getPaymentMethodType(), allocationAmount,
                proportion.multiply(BigDecimal.valueOf(100)));
        }

        return allocations;
    }

    public record RefundAllocation(
        UUID paymentId,
        PaymentMethodType type,
        String gatewayTransactionId,
        BigDecimal amount
    ) {}
}

@Service
public class PartialRefundHandler {

    private final PartialRefundCalculator calculator;
    private final RefundStrategyFactory strategyFactory;
    private final RefundRepository refundRepo;
    private final Logger log = LoggerFactory.getLogger(PartialRefundHandler.class);

    @Transactional
    public List<RefundRecord> executePartialRefund(UUID orderId, BigDecimal refundAmount)
            throws PaymentException {

        List<RefundAllocation> allocations = calculator.calculateProRataRefund(orderId, refundAmount);
        List<RefundRecord> results = new ArrayList<>();

        for (RefundAllocation allocation : allocations) {
            try {
                RefundStrategy strategy = strategyFactory.getStrategy(allocation.type());

                var refundRequest = new RefundRequest(
                    allocation.paymentId(),
                    allocation.gatewayTransactionId(),
                    allocation.amount(),
                    allocation.type()
                );

                RefundResult result = strategy.executeRefund(refundRequest);

                var refund = new RefundRecord();
                refund.setOrderId(orderId);
                refund.setPaymentId(allocation.paymentId());
                refund.setAmount(allocation.amount());
                refund.setStatus(result.status().toString());
                refund.setGatewayRefundId(result.gatewayRefundId());
                refund.setCreatedAt(Instant.now());
                refundRepo.save(refund);

                results.add(refund);
                log.info("Refund processed: {} → ${}", allocation.type(), allocation.amount());

            } catch (PaymentException e) {
                log.error("Refund failed for payment {}: {}", allocation.paymentId(), e.getMessage());
                throw e;  // Fail-fast: if any method fails, abort all
            }
        }

        return results;
    }
}
```

---

### Q3: How do you implement the chargeback dispute workflow?

**Design: State Machine + Evidence Tracking**

A chargeback case transitions through states: RECEIVED → EVIDENCE_COLLECTED → DISPUTE_SUBMITTED → WON/LOST.

```java
@Entity
@Table(name = "chargeback_disputes")
public class ChargebackDispute {
    @Id
    private UUID id;

    @Column(nullable = false)
    private UUID orderId;

    @Column(nullable = false)
    private UUID paymentId;

    @Column(nullable = false)
    private String cardNetwork;  // VISA, MASTERCARD, AMEX

    @Column(nullable = false)
    private String chargebackCaseId;  // From card network

    @Column(nullable = false)
    private String chargebackCode;  // e.g., "4855" for goods not received

    @Column(nullable = false)
    private BigDecimal chargebackAmount;

    @Column(nullable = false)
    private String status;  // RECEIVED, EVIDENCE_COLLECTED, DISPUTE_SUBMITTED, WON, LOST

    @Column(nullable = false)
    private Instant chargebackDate;

    @Column(nullable = true)
    private Instant dueDate;  // Deadline to respond to chargeback

    @Column(nullable = true)
    private Instant submissionDate;

    @Column(nullable = true)
    private Instant resolutionDate;

    @Column(nullable = true)
    private String outcome;  // WON, LOST

    @OneToMany(mappedBy = "dispute", cascade = CascadeType.ALL)
    private List<DisputeEvidence> evidenceItems;

    @OneToMany(mappedBy = "dispute", cascade = CascadeType.ALL)
    private List<DisputeAuditLog> auditLog;

    @Version
    private Long version;

    @Column(nullable = false, updatable = false)
    private Instant createdAt = Instant.now();

    @Column(nullable = false)
    private Instant updatedAt = Instant.now();
}

@Entity
@Table(name = "dispute_evidence")
public class DisputeEvidence {
    @Id
    private UUID id;

    @ManyToOne
    @JoinColumn(name = "dispute_id", nullable = false)
    private ChargebackDispute dispute;

    @Column(nullable = false)
    private String evidenceType;  // ORDER_CONFIRMATION, SHIPPING_PROOF, CUSTOMER_COMMUNICATION, REFUND_PROOF

    @Column(nullable = false)
    private String title;

    @Column(nullable = true, columnDefinition = "TEXT")
    private String description;

    @Column(nullable = true)
    private String documentUrl;  // S3 or CDN URL

    @Column(nullable = false)
    private Instant uploadedAt = Instant.now();
}

@Service
public class ChargebackDisputeService {

    private final ChargebackDisputeRepository disputeRepo;
    private final DisputeEvidenceRepository evidenceRepo;
    private final OrderService orderService;
    private final PaymentRepository paymentRepo;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final S3FileUploader s3Uploader;
    private final Logger log = LoggerFactory.getLogger(ChargebackDisputeService.class);

    @Transactional
    public ChargebackDispute receiveChargebackNotification(ChargebackNotification notification)
            throws PaymentException {

        // Check for duplicate chargeback
        ChargebackDispute existing = disputeRepo.findByChargebackCaseId(notification.caseId());
        if (existing != null) {
            log.warn("Duplicate chargeback case: {}", notification.caseId());
            return existing;
        }

        var dispute = new ChargebackDispute();
        dispute.setOrderId(notification.orderId());
        dispute.setPaymentId(notification.paymentId());
        dispute.setCardNetwork(notification.cardNetwork());
        dispute.setChargebackCaseId(notification.caseId());
        dispute.setChargebackCode(notification.code());
        dispute.setChargebackAmount(notification.amount());
        dispute.setStatus("RECEIVED");
        dispute.setChargebackDate(notification.chargebackDate());
        dispute.setDueDate(notification.dueDate());
        dispute.setCreatedAt(Instant.now());

        disputeRepo.save(dispute);

        // Publish event for immediate alerting
        publishChargebackEvent("chargeback.received", dispute);

        // Schedule automatic evidence collection
        scheduleEvidenceCollection(dispute.getId(), notification.dueDate());

        log.info("Chargeback case created: id={}, amount={}, code={}",
            dispute.getId(), dispute.getChargebackAmount(), dispute.getChargebackCode());

        return dispute;
    }

    @Transactional
    public void collectEvidence(UUID disputeId, List<EvidenceSubmission> submissions)
            throws PaymentException {

        ChargebackDispute dispute = disputeRepo.findByIdWithLock(disputeId);
        if (dispute == null) {
            throw new EntityNotFoundException("Dispute not found");
        }

        if (!dispute.getStatus().equals("RECEIVED")) {
            throw new IllegalStateException("Cannot collect evidence from status: " + dispute.getStatus());
        }

        // Auto-collect standard evidence
        addOrderConfirmationEvidence(dispute);
        addShippingProofEvidence(dispute);
        addCustomerCommunicationEvidence(dispute);

        // Add custom evidence from submissions
        for (EvidenceSubmission submission : submissions) {
            try {
                String documentUrl = s3Uploader.uploadFile(
                    "chargeback/" + dispute.getId() + "/" + submission.filename(),
                    submission.fileContent()
                );

                var evidence = new DisputeEvidence();
                evidence.setDispute(dispute);
                evidence.setEvidenceType(submission.type());
                evidence.setTitle(submission.title());
                evidence.setDescription(submission.description());
                evidence.setDocumentUrl(documentUrl);
                evidenceRepo.save(evidence);

                log.info("Evidence uploaded: type={}, url={}", submission.type(), documentUrl);
            } catch (Exception e) {
                log.error("Failed to upload evidence: {}", e.getMessage());
                throw new PaymentException("Evidence upload failed", e);
            }
        }

        dispute.setStatus("EVIDENCE_COLLECTED");
        disputeRepo.save(dispute);

        publishChargebackEvent("chargeback.evidence_collected", dispute);
        log.info("Evidence collection completed for dispute {}", disputeId);
    }

    @Transactional
    public void submitDispute(UUID disputeId) throws PaymentException {
        ChargebackDispute dispute = disputeRepo.findByIdWithLock(disputeId);
        if (dispute == null) {
            throw new EntityNotFoundException("Dispute not found");
        }

        if (!dispute.getStatus().equals("EVIDENCE_COLLECTED")) {
            throw new IllegalStateException("Cannot submit dispute from status: " + dispute.getStatus());
        }

        try {
            // Upload evidence to card network portal (Visa, MasterCard, etc.)
            submitToCardNetwork(dispute);

            dispute.setStatus("DISPUTE_SUBMITTED");
            dispute.setSubmissionDate(Instant.now());
            disputeRepo.save(dispute);

            publishChargebackEvent("chargeback.dispute_submitted", dispute);

            log.info("Dispute submitted to card network: id={}, case={}",
                disputeId, dispute.getChargebackCaseId());

        } catch (Exception e) {
            log.error("Failed to submit dispute: {}", e.getMessage());
            throw new PaymentException("Dispute submission failed", e);
        }
    }

    @Transactional
    public void resolveChargeback(UUID disputeId, String outcome) throws PaymentException {
        ChargebackDispute dispute = disputeRepo.findByIdWithLock(disputeId);
        if (dispute == null) {
            throw new EntityNotFoundException("Dispute not found");
        }

        if (!dispute.getStatus().equals("DISPUTE_SUBMITTED")) {
            throw new IllegalStateException("Cannot resolve dispute from status: " + dispute.getStatus());
        }

        if (!outcome.matches("WON|LOST")) {
            throw new IllegalArgumentException("Invalid outcome: " + outcome);
        }

        dispute.setStatus(outcome);
        dispute.setOutcome(outcome);
        dispute.setResolutionDate(Instant.now());
        disputeRepo.save(dispute);

        publishChargebackEvent("chargeback." + outcome.toLowerCase(), dispute);

        // If lost, create manual review ticket
        if (outcome.equals("LOST")) {
            createLostChargebackTicket(dispute);
        }

        log.info("Chargeback resolved: id={}, outcome={}", disputeId, outcome);
    }

    private void addOrderConfirmationEvidence(ChargebackDispute dispute) {
        Order order = orderService.getOrder(dispute.getOrderId());

        var evidence = new DisputeEvidence();
        evidence.setDispute(dispute);
        evidence.setEvidenceType("ORDER_CONFIRMATION");
        evidence.setTitle("Order Confirmation Email");
        evidence.setDescription(String.format(
            "Order %s placed on %s for customer %s",
            order.getId(), order.getCreatedAt(), order.getCustomerEmail()
        ));
        evidence.setDocumentUrl(uploadOrderConfirmation(order));
        evidenceRepo.save(evidence);
    }

    private void addShippingProofEvidence(ChargebackDispute dispute) {
        Order order = orderService.getOrder(dispute.getOrderId());
        Shipment shipment = fulfillmentService.getShipment(order.getId());

        if (shipment != null && shipment.getTrackingNumber() != null) {
            var evidence = new DisputeEvidence();
            evidence.setDispute(dispute);
            evidence.setEvidenceType("SHIPPING_PROOF");
            evidence.setTitle("Proof of Shipment");
            evidence.setDescription(String.format(
                "Item shipped via %s, tracking: %s on %s",
                shipment.getCarrier(), shipment.getTrackingNumber(), shipment.getShippedAt()
            ));
            evidence.setDocumentUrl(uploadShippingLabel(shipment));
            evidenceRepo.save(evidence);
        }
    }

    private void addCustomerCommunicationEvidence(ChargebackDispute dispute) {
        // Fetch customer support tickets related to this order
        List<SupportTicket> tickets = supportService.getTicketsByOrder(dispute.getOrderId());

        for (SupportTicket ticket : tickets.stream().limit(3).toList()) {  // Limit to 3 recent
            var evidence = new DisputeEvidence();
            evidence.setDispute(dispute);
            evidence.setEvidenceType("CUSTOMER_COMMUNICATION");
            evidence.setTitle("Support Ticket: " + ticket.getSubject());
            evidence.setDescription(ticket.getDescription());
            evidence.setDocumentUrl(uploadSupportTicket(ticket));
            evidenceRepo.save(evidence);
        }
    }

    private void submitToCardNetwork(ChargebackDispute dispute) {
        // Call card network API (Visa, MasterCard) to submit dispute
        // Include all evidence items
        // Implementation varies by card network
    }

    private void createLostChargebackTicket(ChargebackDispute dispute) {
        var ticket = new ManualReviewTicket();
        ticket.setDisputeId(dispute.getId());
        ticket.setOrderId(dispute.getOrderId());
        ticket.setReason("Chargeback case lost: " + dispute.getChargebackCode());
        ticket.setStatus("OPEN");
        ticket.setPriority("HIGH");
        ticket.setCreatedAt(Instant.now());
        manualReviewTicketRepo.save(ticket);

        slackNotifier.notifyWithPriority(
            "CHARGEBACK LOST: Order " + dispute.getOrderId() + " for $" +
            dispute.getChargebackAmount(), "high");
    }

    private void publishChargebackEvent(String topic, ChargebackDispute dispute) {
        var event = new ChargebackEvent(
            dispute.getId(),
            dispute.getOrderId(),
            dispute.getChargebackCaseId(),
            dispute.getStatus(),
            dispute.getChargebackAmount(),
            Instant.now()
        );
        kafkaTemplate.send(topic, dispute.getId().toString(), event);
    }

    private void scheduleEvidenceCollection(UUID disputeId, Instant dueDate) {
        // Schedule a job to auto-collect evidence 1 week before due date
        long delayMs = dueDate.toEpochMilli() - System.currentTimeMillis() - (7 * 24 * 60 * 60 * 1000);
        if (delayMs > 0) {
            taskScheduler.schedule(
                () -> autoCollectEvidence(disputeId),
                Instant.now().plus(Duration.ofMillis(delayMs))
            );
        }
    }
}
```

---

### Q4: How do you prevent duplicate refunds?

**Design: Idempotency Key + Redis Cache + DB Constraint**

Each refund request includes an idempotency key (e.g., orderId + refund reason + amount hash). Check Redis cache first; if found and successful, return cached result.

```java
@Service
public class RefundIdempotencyGuard {

    private final RedisTemplate<String, Object> redis;
    private final RefundRepository refundRepo;
    private final Logger log = LoggerFactory.getLogger(RefundIdempotencyGuard.class);

    private static final String IDEMPOTENCY_KEY_PREFIX = "refund:idempotency:";
    private static final long CACHE_TTL_SECONDS = 604800;  // 7 days

    @Transactional
    public RefundRecord executeIdempotentRefund(UUID orderId, String refundReason,
            BigDecimal refundAmount, RefundExecutor executor) throws PaymentException {

        // Generate idempotency key
        String idempotencyKey = generateIdempotencyKey(orderId, refundReason, refundAmount);

        // Check Redis cache
        String cachedRefundJson = (String) redis.opsForValue()
            .get(IDEMPOTENCY_KEY_PREFIX + idempotencyKey);

        if (cachedRefundJson != null) {
            log.info("Returning cached refund for idempotency key: {}", idempotencyKey);
            return deserializeRefund(cachedRefundJson);
        }

        // Check DB for existing refund with same idempotency signature
        List<RefundRecord> existingRefunds = refundRepo.findByOrderIdAndReasonAndAmount(
            orderId, refundReason, refundAmount
        );

        if (!existingRefunds.isEmpty()) {
            RefundRecord existing = existingRefunds.get(0);
            if (existing.getStatus().equals("COMPLETED")) {
                log.info("Found existing completed refund, returning it");
                return existing;
            }
        }

        // Execute refund
        RefundRecord result = executor.execute();

        // Cache result
        if (result.getStatus().equals("COMPLETED")) {
            redis.opsForValue().set(
                IDEMPOTENCY_KEY_PREFIX + idempotencyKey,
                serializeRefund(result),
                Duration.ofSeconds(CACHE_TTL_SECONDS)
            );
        }

        return result;
    }

    private String generateIdempotencyKey(UUID orderId, String reason, BigDecimal amount) {
        String composite = orderId + ":" + reason + ":" + amount.toString();
        return Hashing.sha256()
            .hashString(composite, StandardCharsets.UTF_8)
            .toString();
    }

    private String serializeRefund(RefundRecord refund) {
        ObjectMapper mapper = new ObjectMapper();
        return mapper.writeValueAsString(refund);
    }

    private RefundRecord deserializeRefund(String json) {
        ObjectMapper mapper = new ObjectMapper();
        return mapper.readValue(json, RefundRecord.class);
    }

    @FunctionalInterface
    public interface RefundExecutor {
        RefundRecord execute() throws PaymentException;
    }
}
```

---

### Q5: How do you reconcile refunds with the accounting system?

**Design: Kafka Event Producer + Accounting Consumer**

Each refund completion publishes a `RefundJournalEntry` event to Kafka. An accounting consumer service listens and posts GL entries.

```java
@Service
public class RefundJournalPublisher {

    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final Logger log = LoggerFactory.getLogger(RefundJournalPublisher.class);

    public void publishRefundJournal(RefundRecord refund, OrderDetails order) {
        // Debit: Refund Expense (4110) or Contra-Revenue
        // Credit: Cash (1010) or Customer Account (Payable)

        var journalEntry = new RefundJournalEntry(
            UUID.randomUUID(),
            refund.getOrderId(),
            refund.getId(),
            refund.getAmount(),
            order.getCurrency(),
            List.of(
                new JournalLineItem("4110", "Refund Expense", refund.getAmount(), "DEBIT"),
                new JournalLineItem("1010", "Cash", refund.getAmount(), "CREDIT")
            ),
            "Refund processed for order " + refund.getOrderId(),
            Instant.now(),
            "PENDING"  // Status: PENDING → POSTED
        );

        kafkaTemplate.send("accounting.journal.entry",
            refund.getOrderId().toString(), journalEntry);

        log.info("Published journal entry for refund {}: ${}", refund.getId(), refund.getAmount());
    }

    public record RefundJournalEntry(
        UUID entryId,
        UUID orderId,
        UUID refundId,
        BigDecimal amount,
        String currency,
        List<JournalLineItem> lineItems,
        String description,
        Instant createdAt,
        String status
    ) {}

    public record JournalLineItem(
        String glAccountNumber,
        String accountName,
        BigDecimal amount,
        String debitCredit
    ) {}
}

@Service
public class AccountingJournalConsumer {

    private final AccountingLedger ledger;
    private final JournalEntryRepository journalEntryRepo;
    private final Logger log = LoggerFactory.getLogger(AccountingJournalConsumer.class);

    @KafkaListener(topics = "accounting.journal.entry", groupId = "accounting-service")
    public void consumeJournalEntry(RefundJournalEntry entry) {
        try {
            log.info("Consuming journal entry for refund {}", entry.refundId());

            // Post to GL
            for (JournalLineItem item : entry.lineItems()) {
                ledger.postTransaction(
                    item.glAccountNumber(),
                    item.debitCredit(),
                    item.amount(),
                    entry.description(),
                    entry.createdAt()
                );
            }

            // Mark entry as posted
            journalEntryRepo.updateStatus(entry.entryId(), "POSTED", Instant.now());

            log.info("Journal entry posted: id={}, refund={}", entry.entryId(), entry.refundId());

        } catch (Exception e) {
            log.error("Failed to post journal entry: {}", e.getMessage());
            // Retry logic: Kafka will retry based on listener container settings
            throw new RuntimeException("Journal posting failed", e);
        }
    }
}
```

---

### Q6: What happens if refund to credit card fails?

**Design: Retry Service + Manual Review Escalation**

Refund to credit card can fail due to:
1. Card expired/revoked
2. Gateway timeout
3. Cardholder blocked refunds

Retry 3 times with exponential backoff. On final failure, escalate to ops.

```java
@Service
public class RefundRetryService {

    private final RefundRepository refundRepo;
    private final RefundStrategyFactory strategyFactory;
    private final ManualReviewTicketRepository reviewTicketRepo;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final RedisTemplate<String, Object> redis;
    private final SlackNotifier slackNotifier;
    private final Logger log = LoggerFactory.getLogger(RefundRetryService.class);

    private static final int MAX_RETRIES = 3;
    private static final long[] BACKOFF_MILLIS = {5000, 30000, 300000};  // 5s, 30s, 5m

    @Transactional
    public void retryFailedRefund(UUID refundId) throws PaymentException {
        RefundRecord refund = refundRepo.findById(refundId)
            .orElseThrow(() -> new EntityNotFoundException("Refund not found"));

        if (!refund.getStatus().equals("FAILED")) {
            throw new IllegalStateException("Refund not in FAILED status");
        }

        if (refund.getRetryCount() >= MAX_RETRIES) {
            log.error("Refund {} exceeded max retries, escalating", refundId);
            escalateToManualReview(refund);
            return;
        }

        try {
            RefundStrategy strategy = strategyFactory.getStrategy(refund.getPaymentMethodType());

            var refundRequest = new RefundRequest(
                refund.getPaymentId(),
                refund.getGatewayTransactionId(),
                refund.getAmount(),
                refund.getPaymentMethodType()
            );

            RefundResult result = strategy.executeRefund(refundRequest);

            if (result.status() == RefundStatus.COMPLETED) {
                refund.setStatus("COMPLETED");
                refund.setGatewayRefundId(result.gatewayRefundId());
                refund.setCompletedAt(Instant.now());
                refundRepo.save(refund);

                log.info("Refund retry succeeded: id={}", refundId);
            } else {
                throw new PaymentException("Retry returned non-completed status");
            }
        } catch (PaymentException e) {
            refund.setRetryCount(refund.getRetryCount() + 1);
            refund.setErrorMessage(e.getMessage());

            if (refund.getRetryCount() < MAX_RETRIES) {
                refund.setStatus("RETRYING");
                refundRepo.save(refund);

                long delayMillis = BACKOFF_MILLIS[refund.getRetryCount() - 1];
                scheduleRetry(refundId, delayMillis);
                log.info("Scheduled refund retry {} for {} in {}ms",
                    refund.getRetryCount(), refundId, delayMillis);
            } else {
                escalateToManualReview(refund);
            }
        }
    }

    private void escalateToManualReview(RefundRecord refund) {
        var ticket = new ManualReviewTicket();
        ticket.setRefundId(refund.getId());
        ticket.setOrderId(refund.getOrderId());
        ticket.setAmount(refund.getAmount());
        ticket.setReason("Refund failed after " + MAX_RETRIES + " retries: " + refund.getErrorMessage());
        ticket.setStatus("OPEN");
        ticket.setPriority("HIGH");
        ticket.setCreatedAt(Instant.now());
        reviewTicketRepo.save(ticket);

        slackNotifier.notifyWithPriority(
            "REFUND FAILED: Order " + refund.getOrderId() + " refund $" + refund.getAmount() +
            " failed after retries. Manual review required.",
            "high"
        );

        var event = new RefundManualReviewEvent(
            refund.getId(),
            refund.getOrderId(),
            refund.getAmount(),
            refund.getErrorMessage(),
            Instant.now()
        );
        kafkaTemplate.send("payment.refund.manual_review_required",
            refund.getId().toString(), event);
    }

    private void scheduleRetry(UUID refundId, long delayMillis) {
        long retryTimestamp = System.currentTimeMillis() + delayMillis;
        redis.opsForZSet().add("refund:retry:queue", refundId.toString(), retryTimestamp);
    }
}
```

---

## 5. Microservices Breakdown

| Service | Responsibility | Tech Stack | Key APIs |
|---------|-----------------|------------|----------|
| **Refund Service** | Process refunds, state machine, strategy selection | Java 17 + Spring Boot 3, PostgreSQL, Redis | POST /refund, GET /refund/{id}, POST /refund/{id}/retry |
| **Chargeback Service** | Receive chargeback notifications, collect evidence, submit disputes | Java 17 + Spring Boot 3, PostgreSQL, S3 | POST /chargeback/webhook, POST /dispute/{id}/evidence, POST /dispute/{id}/submit |
| **Accounting Consumer** | Listen to refund events, post GL entries | Java 17 + Spring Boot 3, Kafka, GL API | Kafka consumer group |
| **Manual Review Service** | Track failed refunds/chargebacks, escalation | Java 17 + Spring Boot 3, PostgreSQL | GET /tickets, POST /tickets/{id}/resolve |

---

## 6. Database Design

### PostgreSQL Schema

```sql
-- Refund records
CREATE TABLE refunds (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    payment_id UUID NOT NULL,
    refund_reason VARCHAR(100) NOT NULL,     -- CUSTOMER_REQUEST, PRODUCT_DEFECT, DUPLICATE_CHARGE
    amount NUMERIC(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    payment_method_type VARCHAR(50) NOT NULL, -- CREDIT_CARD, GIFT_CARD, WALLET
    gateway_transaction_id VARCHAR(255),
    gateway_refund_id VARCHAR(255),
    status VARCHAR(50) NOT NULL,               -- PENDING, PROCESSING, COMPLETED, FAILED, RETRYING, MANUAL_REVIEW
    retry_count INT DEFAULT 0,
    error_message TEXT,
    idempotency_key VARCHAR(255),
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP,
    UNIQUE(order_id, idempotency_key),        -- Prevent duplicate refunds
    CONSTRAINT amount_positive CHECK (amount > 0)
);

CREATE INDEX idx_refund_order ON refunds(order_id);
CREATE INDEX idx_refund_status ON refunds(status) WHERE status != 'COMPLETED';
CREATE INDEX idx_refund_created ON refunds(created_at DESC);

-- Chargeback disputes
CREATE TABLE chargeback_disputes (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    payment_id UUID NOT NULL,
    card_network VARCHAR(50) NOT NULL,        -- VISA, MASTERCARD, AMEX
    chargeback_case_id VARCHAR(255) NOT NULL UNIQUE,
    chargeback_code VARCHAR(50) NOT NULL,     -- 4855 = Goods not received
    chargeback_amount NUMERIC(12, 2) NOT NULL,
    status VARCHAR(50) NOT NULL,               -- RECEIVED, EVIDENCE_COLLECTED, DISPUTE_SUBMITTED, WON, LOST
    chargeback_date TIMESTAMP NOT NULL,
    due_date TIMESTAMP NOT NULL,
    submission_date TIMESTAMP,
    resolution_date TIMESTAMP,
    outcome VARCHAR(20),                       -- WON, LOST
    version BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_dispute_order ON chargeback_disputes(order_id);
CREATE INDEX idx_dispute_case_id ON chargeback_disputes(chargeback_case_id);
CREATE INDEX idx_dispute_status ON chargeback_disputes(status);

-- Dispute evidence (attachments)
CREATE TABLE dispute_evidence (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    dispute_id UUID NOT NULL REFERENCES chargeback_disputes(id),
    evidence_type VARCHAR(100) NOT NULL,       -- ORDER_CONFIRMATION, SHIPPING_PROOF, etc.
    title VARCHAR(255) NOT NULL,
    description TEXT,
    document_url VARCHAR(1024),                -- S3 URL
    uploaded_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_evidence_dispute ON dispute_evidence(dispute_id);

-- Refund audit log
CREATE TABLE refund_audit_log (
    id BIGSERIAL PRIMARY KEY,
    refund_id UUID NOT NULL,
    event_type VARCHAR(100),                   -- CREATED, PROCESSING, COMPLETED, FAILED
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    details JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_audit_refund ON refund_audit_log(refund_id);
```

### MongoDB Collections

```javascript
// Refund event log (for analytics)
db.createCollection("refund_events", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            properties: {
                _id: { bsonType: "objectId" },
                refund_id: { bsonType: "string" },
                order_id: { bsonType: "string" },
                event_type: { bsonType: "string" },  // initiated, processing, completed, failed
                amount: { bsonType: "decimal" },
                currency: { bsonType: "string" },
                reason: { bsonType: "string" },
                payment_method_type: { bsonType: "string" },
                status: { bsonType: "string" },
                timestamp: { bsonType: "date" }
            }
        }
    }
});

db.refund_events.createIndex({ timestamp: -1 });
db.refund_events.createIndex({ refund_id: 1 });
db.refund_events.createIndex({ order_id: 1 });

// Chargeback timeline
db.createCollection("chargeback_events", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            properties: {
                _id: { bsonType: "objectId" },
                dispute_id: { bsonType: "string" },
                order_id: { bsonType: "string" },
                event_type: { bsonType: "string" },  // received, evidence_collected, submitted, won, lost
                case_id: { bsonType: "string" },
                code: { bsonType: "string" },
                amount: { bsonType: "decimal" },
                outcome: { bsonType: "string" },
                timestamp: { bsonType: "date" }
            }
        }
    }
});

db.chargeback_events.createIndex({ timestamp: -1 });
db.chargeback_events.createIndex({ dispute_id: 1 });
db.chargeback_events.createIndex({ case_id: 1 });
```

---

## 7. Redis Data Structures

```java
// Refund idempotency cache (7 days TTL)
SET refund:idempotency:{orderId}:{reason}:{amount_hash} "{\"id\":\"ref_123\",\"status\":\"COMPLETED\"}" EX 604800

// Refund retry queue (sorted set by retry timestamp)
ZADD refund:retry:queue 1700000000 "ref_123"
ZADD refund:retry:queue 1700000300 "ref_124"

// Chargeback evidence upload progress
HSET chargeback:upload:progress {disputeId} order_confirmation "uploaded"
HSET chargeback:upload:progress {disputeId} shipping_proof "pending"

// Chargeback due date alerts
ZADD chargeback:due:soon 1700086400 "dispute_001"  # Alert if due within 24h

// Manual review ticket alerts
LPUSH manual_review:alerts:high "ticket_001 ticket_002"
EXPIRE manual_review:alerts:high 86400
```

**Key Commands:**

```java
// Check idempotency
String cached = redis.opsForValue().get("refund:idempotency:" + key);

// Schedule refund retry
redis.opsForZSet().add("refund:retry:queue", refundId, System.currentTimeMillis() + delayMs);

// Track evidence uploads
redis.opsForHash().put("chargeback:upload:progress:" + disputeId, "order_confirmation", "uploaded");
```

---

## 8. Kafka Event Flow

### Topics

```
payment.refund.initiated
  ├─ RefundInitiatedEvent
  │   - refund_id, order_id, amount, reason
  │   - Consumed by: Refund Service, Audit Service, Analytics

payment.refund.processing
  ├─ RefundProcessingEvent
  │   - refund_id, gateway_name, payment_method_type
  │   - Consumed by: Audit Service

payment.refund.completed
  ├─ RefundCompletedEvent
  │   - refund_id, gateway_refund_id, amount, settled_to_account
  │   - Consumed by: Accounting Service, Order Service, Audit Service

payment.refund.failed
  ├─ RefundFailedEvent
  │   - refund_id, reason, error_code
  │   - Consumed by: Retry Service, Manual Review Service

payment.refund.manual_review_required
  ├─ RefundManualReviewEvent
  │   - refund_id, order_id, amount, reason
  │   - Consumed by: Manual Review Service, Alert Service

chargeback.received
  ├─ ChargebackReceivedEvent
  │   - dispute_id, case_id, code, amount, due_date
  │   - Consumed by: Chargeback Service, Alert Service

chargeback.evidence_collected
  ├─ EvidenceCollectedEvent
  │   - dispute_id, evidence_count

chargeback.dispute_submitted
  ├─ DisputeSubmittedEvent
  │   - dispute_id, case_id, submission_date

chargeback.won
  ├─ ChargebackWonEvent
  │   - dispute_id, case_id, refund_amount
  │   - Consumed by: Order Service, Reporting Service

chargeback.lost
  ├─ ChargebackLostEvent
  │   - dispute_id, case_id, lost_amount
  │   - Consumed by: Manual Review Service, Finance Team

accounting.journal.entry
  ├─ RefundJournalEntry
  │   - entry_id, refund_id, gl_accounts, amount
  │   - Consumed by: Accounting Service
```

### Payload Examples

```json
// Event: payment.refund.completed
{
  "event_id": "evt_refund_complete_001",
  "refund_id": "ref_123",
  "order_id": "order_456",
  "customer_id": "cust_789",
  "amount": 50.00,
  "currency": "USD",
  "refund_reason": "CUSTOMER_REQUEST",
  "payment_method_type": "CREDIT_CARD",
  "gateway_refund_id": "re_1abcdef123",
  "completed_at": "2026-04-02T10:30:00Z"
}

// Event: chargeback.received
{
  "event_id": "evt_chargeback_rcv_001",
  "dispute_id": "disp_001",
  "order_id": "order_456",
  "case_id": "CHG_123456789",
  "card_network": "VISA",
  "chargeback_code": "4855",
  "chargeback_amount": 99.99,
  "chargeback_reason": "Goods Not Received",
  "chargeback_date": "2026-04-01T00:00:00Z",
  "due_date": "2026-04-15T23:59:59Z",
  "received_at": "2026-04-02T10:00:00Z"
}

// Event: accounting.journal.entry
{
  "entry_id": "je_001",
  "refund_id": "ref_123",
  "order_id": "order_456",
  "amount": 50.00,
  "currency": "USD",
  "description": "Refund for order 456",
  "line_items": [
    { "gl_account": "4110", "account_name": "Refund Expense", "debit_credit": "DEBIT", "amount": 50.00 },
    { "gl_account": "1010", "account_name": "Cash", "debit_credit": "CREDIT", "amount": 50.00 }
  ],
  "created_at": "2026-04-02T10:30:00Z",
  "status": "PENDING"
}
```

---

## 9. Implementation Code

### Class 1: RefundService

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
public class RefundService {

    private final RefundRepository refundRepo;
    private final PaymentRepository paymentRepo;
    private final RefundStrategyFactory strategyFactory;
    private final RefundIdempotencyGuard idempotencyGuard;
    private final RedisTemplate<String, Object> redis;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final Logger log = LoggerFactory.getLogger(RefundService.class);

    @Autowired
    public RefundService(
            RefundRepository refundRepo,
            PaymentRepository paymentRepo,
            RefundStrategyFactory strategyFactory,
            RefundIdempotencyGuard idempotencyGuard,
            RedisTemplate<String, Object> redis,
            KafkaTemplate<String, Object> kafkaTemplate) {
        this.refundRepo = refundRepo;
        this.paymentRepo = paymentRepo;
        this.strategyFactory = strategyFactory;
        this.idempotencyGuard = idempotencyGuard;
        this.redis = redis;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Transactional(isolation = Isolation.SERIALIZABLE)
    public RefundRecord processRefund(RefundRequest request) throws PaymentException {
        return idempotencyGuard.executeIdempotentRefund(
            request.orderId(),
            request.refundReason(),
            request.amount(),
            () -> doProcessRefund(request)
        );
    }

    private RefundRecord doProcessRefund(RefundRequest request) throws PaymentException {
        // Fetch original payment
        Payment payment = paymentRepo.findById(request.paymentId())
            .orElseThrow(() -> new EntityNotFoundException("Payment not found"));

        // Validate amount
        if (request.amount().compareTo(payment.getAmount()) > 0) {
            throw new PaymentException("Refund amount exceeds original payment");
        }

        // Create refund record
        var refund = new RefundRecord();
        refund.setOrderId(request.orderId());
        refund.setPaymentId(request.paymentId());
        refund.setRefundReason(request.refundReason());
        refund.setAmount(request.amount());
        refund.setCurrency(payment.getCurrency());
        refund.setPaymentMethodType(payment.getPaymentMethodType());
        refund.setGatewayTransactionId(payment.getGatewayTransactionId());
        refund.setStatus("PENDING");
        refund.setCreatedAt(Instant.now());

        refundRepo.save(refund);

        publishRefundEvent("payment.refund.initiated",
            new RefundInitiatedEvent(
                refund.getId(), refund.getOrderId(), refund.getAmount(),
                refund.getRefundReason(), Instant.now()
            ));

        try {
            // Execute refund
            RefundStrategy strategy = strategyFactory.getStrategy(payment.getPaymentMethodType());
            RefundResult result = strategy.executeRefund(request);

            // Update refund record
            refund.setStatus("COMPLETED");
            refund.setGatewayRefundId(result.gatewayRefundId());
            refund.setCompletedAt(Instant.now());
            refundRepo.save(refund);

            // Publish success event
            publishRefundEvent("payment.refund.completed",
                new RefundCompletedEvent(
                    refund.getId(), refund.getOrderId(), refund.getAmount(),
                    refund.getPaymentMethodType(), result.gatewayRefundId(), Instant.now()
                ));

            publishAccountingJournal(refund);

            log.info("Refund processed: id={}, amount={}, method={}",
                refund.getId(), refund.getAmount(), payment.getPaymentMethodType());

            return refund;

        } catch (PaymentException e) {
            log.error("Refund processing failed: {}", e.getMessage());
            refund.setStatus("FAILED");
            refund.setErrorMessage(e.getMessage());
            refundRepo.save(refund);

            publishRefundEvent("payment.refund.failed",
                new RefundFailedEvent(
                    refund.getId(), refund.getOrderId(), e.getMessage(), Instant.now()
                ));

            throw e;
        }
    }

    private void publishRefundEvent(String topic, Object event) {
        kafkaTemplate.send(topic, UUID.randomUUID().toString(), event);
    }

    private void publishAccountingJournal(RefundRecord refund) {
        var journalEntry = new RefundJournalPublisher.RefundJournalEntry(
            UUID.randomUUID(),
            refund.getOrderId(),
            refund.getId(),
            refund.getAmount(),
            refund.getCurrency(),
            List.of(
                new RefundJournalPublisher.JournalLineItem("4110", "Refund Expense",
                    refund.getAmount(), "DEBIT"),
                new RefundJournalPublisher.JournalLineItem("1010", "Cash",
                    refund.getAmount(), "CREDIT")
            ),
            "Refund for order " + refund.getOrderId(),
            Instant.now(),
            "PENDING"
        );
        kafkaTemplate.send("accounting.journal.entry", refund.getOrderId().toString(), journalEntry);
    }
}
```

### Class 2: ChargebackDisputeService (see Q3 above)

Code from Q3 is the main implementation.

### Class 3: RefundRetryService (see Q6 above)

Code from Q6 is the main implementation.

### Class 4: RefundStrategyFactory

```java
package com.payment.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import com.payment.model.*;

import java.util.List;

@Service
public class RefundStrategyFactory {

    private final List<RefundStrategy> strategies;
    private final Logger log = LoggerFactory.getLogger(RefundStrategyFactory.class);

    @Autowired
    public RefundStrategyFactory(List<RefundStrategy> strategies) {
        this.strategies = strategies;
    }

    public RefundStrategy getStrategy(PaymentMethodType type) {
        return strategies.stream()
            .filter(s -> s.supports(type))
            .findFirst()
            .orElseThrow(() -> new PaymentException("No refund strategy for type: " + type));
    }
}

public interface RefundStrategy {
    RefundResult executeRefund(RefundRequest request) throws PaymentException;
    boolean supports(PaymentMethodType type);
}

@Service
public class CreditCardRefundStrategy implements RefundStrategy {
    // Implementation in Q1
}

@Service
public class GiftCardRefundStrategy implements RefundStrategy {
    // Implementation in Q1
}

@Service
public class DigitalWalletRefundStrategy implements RefundStrategy {

    private final ApplePayClient applePayClient;
    private final GooglePayClient googlePayClient;
    private final Logger log = LoggerFactory.getLogger(DigitalWalletRefundStrategy.class);

    @Override
    public RefundResult executeRefund(RefundRequest request) throws PaymentException {
        try {
            // Determine wallet type from token
            if (request.token().startsWith("applepay_")) {
                return refundApplePay(request);
            } else if (request.token().startsWith("googlepay_")) {
                return refundGooglePay(request);
            }
            throw new PaymentException("Unknown wallet type");
        } catch (Exception e) {
            log.error("Wallet refund failed: {}", e.getMessage());
            throw new PaymentException("Wallet refund failed", e);
        }
    }

    private RefundResult refundApplePay(RefundRequest request) {
        // Call Apple Pay API for refund
        return new RefundResult("applepay_ref_123", RefundStatus.COMPLETED,
            "Apple Pay refund processed", Instant.now());
    }

    private RefundResult refundGooglePay(RefundRequest request) {
        // Call Google Pay API for refund
        return new RefundResult("googlepay_ref_123", RefundStatus.COMPLETED,
            "Google Pay refund processed", Instant.now());
    }

    @Override
    public boolean supports(PaymentMethodType type) {
        return type == PaymentMethodType.WALLET;
    }
}
```

### Class 5: PartialRefundCalculator (see Q2 above)

Code from Q2 is the main implementation.

### Class 6: AccountingJournalConsumer

```java
package com.payment.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.payment.model.*;
import com.payment.repository.*;

import java.time.Instant;

@Service
public class AccountingJournalConsumer {

    private final JournalEntryRepository journalEntryRepo;
    private final GeneralLedger generalLedger;
    private final Logger log = LoggerFactory.getLogger(AccountingJournalConsumer.class);

    @Autowired
    public AccountingJournalConsumer(
            JournalEntryRepository journalEntryRepo,
            GeneralLedger generalLedger) {
        this.journalEntryRepo = journalEntryRepo;
        this.generalLedger = generalLedger;
    }

    @Transactional
    @KafkaListener(topics = "accounting.journal.entry", groupId = "accounting-service")
    public void consumeJournalEntry(RefundJournalPublisher.RefundJournalEntry entry) {
        try {
            log.info("Consuming journal entry for refund {}", entry.refundId());

            // Post each line item to GL
            for (RefundJournalPublisher.JournalLineItem item : entry.lineItems()) {
                generalLedger.post(
                    item.glAccountNumber(),
                    item.debitCredit(),
                    item.amount(),
                    entry.description(),
                    entry.createdAt()
                );

                log.debug("Posted: {} {} {}", item.debitCredit(), item.glAccountNumber(),
                    item.amount());
            }

            // Mark as posted
            journalEntryRepo.updateStatus(entry.entryId(), "POSTED", Instant.now());

            log.info("Journal entry successfully posted: {}", entry.entryId());

        } catch (Exception e) {
            log.error("Failed to post journal entry: {}", e.getMessage());
            // Kafka will retry based on listener container error handling
            throw new RuntimeException("Journal posting failed", e);
        }
    }
}
```

---

## 10. Failure Scenarios & Mitigations

| Failure Scenario | Detection Method | Mitigation Strategy |
|---|---|---|
| **Duplicate refund (retry)** | Idempotency key check in Redis + DB UNIQUE constraint | Return cached result from Redis if exists and completed; DB constraint prevents duplication |
| **Refund to expired card** | Gateway returns error (e.g., "card expired") | Retry 3x with backoff; escalate to manual review; suggest alternative refund method |
| **Refund amount exceeds original payment** | Validation before refund initiation | Reject refund; return error to client with suggested max amount |
| **Partial refund on split payment (rounding)** | Validate pro-rata calculation | Use HALF_UP rounding; ensure sum of allocations = original refund amount |
| **Chargeback received but refund already processed** | Check refund status in dispute handler | Create dispute record anyway; note refund proof in evidence; helps win dispute |
| **Evidence upload to S3 fails** | S3 API exception | Retry 3x; queue for manual upload; escalate to ops; extend due date if needed |
| **Dispute submission to card network API fails** | Network timeout or 5xx error | Retry with exponential backoff; manual queue for upload via web portal |
| **Accounting service down (Kafka consumer fails)** | Kafka consumer group lag alerts | Message persists in Kafka; auto-retry when service comes back; no ledger loss |
| **Chargeback won but funds already refunded** | Track refund proof in evidence | Chargeback still resolves favorably; no additional action needed |
| **Manual review ticket backlog** | Monitor queue size | Escalate to management; implement SLA alerts; hire more reviewers |
| **PostgreSQL deadlock on refund retry** | Transaction retry exception | Spring retry mechanism; log and alert; manual inspection if deadlock persists |

---

## 11. Scaling Strategy

### Horizontal Scaling

1. **Refund Service:** 2–3 instances behind load balancer; each handles ~0.8 TPS.
2. **Chargeback Service:** 1–2 instances; I/O-bound (S3 uploads, card network API).
3. **Accounting Consumer:** 1–2 instances; Kafka consumer group handles partitioning.

### Vertical Scaling

- **PostgreSQL:** Increase `max_connections` to 200+; use read replicas for dispute queries.
- **Redis:** Grow to 200MB+ for idempotency cache (7 days TTL) + chargeback state.
- **S3:** No scaling needed; unlimited storage; enable lifecycle policies to archive old evidence (>2 years) to Glacier.

### Database Optimization

```sql
-- Partition refunds by month
ALTER TABLE refunds PARTITION BY RANGE (YEAR(created_at), MONTH(created_at));

-- Archive old disputes (>2 years)
CREATE TABLE disputes_archive AS
SELECT * FROM chargeback_disputes WHERE created_at < NOW() - INTERVAL 2 YEAR;
```

---

## 12. Monitoring & Observability

### Key Metrics

```
Refund Metrics:
  - refund.initiation.latency_ms (p50, p95, p99)
  - refund.completion.latency_ms (time from initiation to completed)
  - refund.success_rate
  - refund.failure_rate
  - refund.retry_count
  - refund.manual_review_queue_length

Chargeback Metrics:
  - chargeback.received_count (per day, by code)
  - chargeback.evidence_collection_latency_ms
  - chargeback.won_rate
  - chargeback.lost_rate
  - dispute.due_date_approaching (24h window)
  - dispute.evidence_upload_success_rate

Accounting Metrics:
  - journal_entry.posting_latency_ms
  - journal_entry.kafka_consumer_lag
  - gl_posting.success_rate

System Health:
  - idempotency_cache.hit_rate
  - s3_upload.success_rate
  - postgres.connection_pool.active
  - manual_review.sla_breach (>24h open)
```

### Alerts

```yaml
RefundFailureRate:
  expr: rate(refund_failures[5m]) > 0.1
  for: 5m
  action: page on-call

ChargebackBacklog:
  expr: chargeback_evidence_pending > 10
  for: 1h
  action: alert to payment ops

ManualReviewSLABreach:
  expr: manual_review_tickets_open_hours > 24
  for: 30m
  action: escalate to supervisor

AccountingLag:
  expr: kafka_consumer_lag > 1000
  for: 10m
  action: alert finance team
```

---

## 13. Summary Cheat Sheet

| Decision | Choice | Rationale |
|---|---|---|
| **Refund Strategy** | Strategy pattern with factory | Decouples payment method logic; easy to add new refund types |
| **Duplicate Prevention** | Idempotency key (Redis 7d TTL) + DB UNIQUE | Fast cache lookup; DB constraint as failsafe |
| **Partial Refunds** | Pro-rata allocation across payment methods | Fair distribution; maintains payment integrity |
| **Chargeback Workflow** | State machine (RECEIVED → EVIDENCE → SUBMITTED → WON/LOST) | Clear progression; tracks deadline (due date) |
| **Evidence Collection** | Auto-collect order/shipping + manual upload | Speeds up response; allows ops to add custom evidence |
| **Retry Logic** | 3 retries with exponential backoff (5s, 30s, 5m) | Balances recovery with performance; manual review after |
| **Accounting Integration** | Kafka event → Accounting consumer → GL posting | Asynchronous; audit trail; reconcilable |
| **Scaling** | 2–3 refund instances; 1–2 chargeback instances; 1–2 accounting consumers | Handles 20K refunds/day + 100 chargebacks/day |
| **Database** | PostgreSQL (refunds, disputes) + MongoDB (audit logs) + S3 (evidence) | ACID for payments; flexible schema for events; unlimited storage for evidence |
| **Caching** | Redis for idempotency cache (7d TTL) + chargeback state | Sub-ms lookups; TTL-based expiration |
| **Event Bus** | Kafka for all refund/chargeback transitions | Decouples services; enables audit + accounting integration |

---

**End of File**
