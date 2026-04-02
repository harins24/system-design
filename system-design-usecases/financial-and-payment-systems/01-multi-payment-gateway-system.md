---
title: Multi-Payment Gateway System
layout: default
---

# Multi-Payment Gateway System — Deep Dive Design

> **Scenario:** Design a payment processing system supporting credit/debit cards (Stripe, PayPal), digital wallets (Apple Pay, Google Pay), Buy Now Pay Later (Affirm, Klarna), gift cards, multiple currencies (USD, EUR, GBP), automatic failover, PCI compliance, and 100K transactions per day.
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

- **Multiple Gateway Support:** Stripe, PayPal, Affirm, Klarna, Apple Pay, Google Pay, gift cards.
- **Currency Support:** USD, EUR, GBP with real-time FX conversion.
- **Partial Payments:** Single transaction split across multiple payment methods (e.g., $70 credit card + $30 gift card).
- **Automatic Failover:** If primary gateway fails, automatically retry with secondary gateway.
- **PCI Compliance:** Never store raw card numbers; tokenize at payment entry; store only tokens and last-4.
- **Idempotency:** Prevent duplicate charges on network retries or client retries.
- **Reconciliation:** Daily batch reconciliation of transactions vs gateway settlement files.
- **Payment History:** Full audit trail per transaction (who, what, when, gateway, status).

### Non-Functional Requirements

- **Throughput:** 100K transactions per day (~1.16 TPS nominal, 5–10 TPS peak).
- **Latency:** Payment authorization under 3 seconds, capture under 2 seconds (p99).
- **Availability:** 99.9% uptime SLA; automatic failover within 5 seconds.
- **Consistency:** At-most-once payment charging; strong consistency on idempotency checks.
- **Data Retention:** 7 years for PCI audit; sensitive card data purged after 2 years.

### Out of Scope

- Fraud detection algorithms (handled by payment gateways).
- Advanced chargeback management (separate service, Q23).
- Accounting integration (separate consumer service).
- Mobile app push notifications (handled by notification service).

---

## 2. Capacity Estimation

```
CAPACITY PLANNING TABLE

┌─────────────────────────────────────────────────────────────────┐
│ Metric                          │ Calculation       │ Value       │
├─────────────────────────────────────────────────────────────────┤
│ Transactions per day            │ Given            │ 100,000     │
│ TPS (nominal)                   │ 100K / 86400s    │ 1.16 TPS    │
│ TPS (peak, 10x burst)           │ 1.16 * 10        │ 11.6 TPS    │
│ Avg transaction value           │ Estimate         │ $75         │
│ Daily gross volume              │ 100K * $75       │ $7.5M       │
│                                 │                  │             │
│ Transaction DB rows/year        │ 100K * 365       │ 36.5M rows  │
│ Avg row size (transactions)     │ Estimate (LOBs)  │ 2 KB        │
│ PostgreSQL storage/year         │ 36.5M * 2KB      │ 73 GB       │
│ With indices (2x)               │ 73GB * 2         │ 146 GB      │
│                                 │                  │             │
│ Payment method tokens stored    │ Daily unique 10% │ 10,000      │
│ Each token record size          │ Estimate         │ 500 B       │
│ Total token storage (annual)    │ 10K * 365 * 500B │ 1.8 GB      │
│                                 │                  │             │
│ Gateway response cache (Redis)  │ 24h TTL, 100K    │ 100K keys   │
│ Cache entry size               │ ~1 KB per entry  │ 100 MB      │
│                                 │                  │             │
│ Avg outbound bandwidth/sec      │ 11.6 TPS * 2KB   │ 23 KB/s     │
│ Peak bandwidth                  │ 23 KB/s * 10     │ 230 KB/s    │
│                                 │                  │             │
│ Kafka message volume/day        │ Payment events   │ 200K msgs   │
│ Avg Kafka event size            │ Estimate         │ 500 B       │
│ Kafka daily storage             │ 200K * 500B      │ 100 MB      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 3. High-Level Architecture

```
┌────────────────────────────────────────────────────────────────────────┐
│                          CLIENT / MOBILE APP                            │
│                    (Uses Stripe.js, PayPal SDK)                         │
└────────────────────────┬─────────────────────────────────────────────────┘
                         │ Payment token + amount
                         ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    API GATEWAY (Spring Cloud Gateway)                   │
│              Rate limiting, request validation, auth                    │
└────────────────────────┬─────────────────────────────────────────────────┘
                         │
         ┌───────────────┴───────────────┐
         ▼                               ▼
    ┌─────────────┐              ┌──────────────────┐
    │ Order       │              │ Payment Service  │
    │ Service     │              │ (Core)           │
    │             │              │                  │
    │ Validates   │              │ • Routes to      │
    │ order       │              │   gateways       │
    │ Publishes   │              │ • Handles        │
    │ event       │              │   failover       │
    └─────┬───────┘              │ • Reconciles     │
          │                      └────────┬──────────┘
          │ Order.Placed event             │
          ▼                                ▼
        ┌──────────────────────────────────────────┐
        │          Kafka (Event Bus)                │
        │  Topics: payment.requested                │
        │          payment.completed                │
        │          payment.failed                   │
        │          gateway.health.updated           │
        └──────────┬───────────┬────────────────────┘
                   │           │
        ┌──────────┘           └──────────┐
        ▼                                  ▼
    ┌────────────────┐         ┌──────────────────────┐
    │ Reconciliation │         │ Gateway Health       │
    │ Job            │         │ Monitor              │
    │                │         │                      │
    │ Batch matches  │         │ Pings gateways      │
    │ txns vs        │         │ Updates health      │
    │ settlements    │         │ scores in Redis     │
    └────────────────┘         └──────────────────────┘
        ▼                                  ▼
    ┌────────────────┐         ┌──────────────────────┐
    │   PostgreSQL   │         │     Redis            │
    │                │         │                      │
    │ • Transactions │         │ • Health scores      │
    │ • Authorizations          │ • Rate limits        │
    │ • Reconciliation│         │ • FX rates cache     │
    │   records      │         │ • Token blacklist    │
    └────────────────┘         └──────────────────────┘
                                        ▲
                                        │
        ┌───────────────────────────────┘
        │
        ▼
    ┌──────────────────────────────────────────┐
    │  Payment Gateway Adapters                 │
    │  ┌─────────────────────────────────────┐  │
    │  │ StripeGatewayAdapter                │  │
    │  │ • POST /v1/payment_intents           │  │
    │  │ • Confirm intent                     │  │
    │  └─────────────────────────────────────┘  │
    │  ┌─────────────────────────────────────┐  │
    │  │ PayPalGatewayAdapter                │  │
    │  │ • POST /v2/payments                  │  │
    │  │ • Execute payment                    │  │
    │  └─────────────────────────────────────┘  │
    │  ┌─────────────────────────────────────┐  │
    │  │ AffirmGatewayAdapter                │  │
    │  │ • POST /v2/charges                   │  │
    │  │ • Authorize & charge                 │  │
    │  └─────────────────────────────────────┘  │
    │  ┌─────────────────────────────────────┐  │
    │  │ GiftCardGatewayAdapter              │  │
    │  │ • Local DB check                     │  │
    │  │ • Decrement balance                  │  │
    │  └─────────────────────────────────────┘  │
    └──────────────────────────────────────────┘
```

---

## 4. Core Design Questions Answered

### Q1: How do you abstract multiple payment gateways behind a unified interface?

**Design Pattern: Strategy + Adapter**

Define a `PaymentGateway` interface that all gateway implementations satisfy. Each concrete implementation (Stripe, PayPal, Affirm) adapts the gateway's proprietary API to our unified contract.

```java
public interface PaymentGateway {
    PaymentResult authorize(AuthorizeRequest request) throws PaymentException;
    PaymentResult capture(CaptureRequest request) throws PaymentException;
    RefundResult refund(RefundRequest request) throws PaymentException;
    GatewayStatus healthCheck() throws PaymentException;
}

public record AuthorizeRequest(
    String idempotencyKey,
    String token,           // Tokenized card/wallet
    BigDecimal amount,
    String currency,
    String customerId,
    Map<String, String> metadata
) {}

public record PaymentResult(
    String gatewayTransactionId,
    PaymentStatus status,   // PENDING, AUTHORIZED, FAILED
    String message,
    Instant createdAt
) {}
```

**Adapter Implementation Example:**

```java
@Component
public class StripeGatewayAdapter implements PaymentGateway {

    private final StripeClient stripeClient;
    private final Logger log = LoggerFactory.getLogger(StripeGatewayAdapter.class);

    @Override
    public PaymentResult authorize(AuthorizeRequest request) throws PaymentException {
        try {
            var params = new StripePaymentIntentParams()
                .setAmount(request.amount().multiply(BigDecimal.valueOf(100)).longValue())
                .setCurrency(request.currency().toLowerCase())
                .setPaymentMethod(request.token())
                .setCustomerId(request.customerId())
                .setConfirm(true)
                .setReturnUrl("https://myapp.com/checkout/return");

            PaymentIntent intent = stripeClient.createPaymentIntent(params);

            return new PaymentResult(
                intent.id(),
                mapStripeStatus(intent.status()),
                intent.chargeError() != null ? intent.chargeError().message() : "Success",
                Instant.now()
            );
        } catch (StripeException e) {
            log.error("Stripe authorization failed: {}", e.getMessage());
            throw new PaymentException("Stripe auth failed", e);
        }
    }

    @Override
    public GatewayStatus healthCheck() throws PaymentException {
        try {
            // Ping Stripe API
            stripeClient.ping();
            return new GatewayStatus(true, 0, Instant.now());
        } catch (Exception e) {
            return new GatewayStatus(false, 1, Instant.now());
        }
    }

    private PaymentStatus mapStripeStatus(String stripeStatus) {
        return switch (stripeStatus) {
            case "requires_payment_method" -> PaymentStatus.PENDING;
            case "succeeded" -> PaymentStatus.AUTHORIZED;
            case "processing" -> PaymentStatus.PENDING;
            default -> PaymentStatus.FAILED;
        };
    }
}
```

---

### Q2: How do you implement automatic failover between gateways?

**Design: Gateway Router with Health Score Tracking**

Maintain a priority list of gateways for each payment type. Track health scores in Redis (updated every 30 seconds). When a payment fails, try the next healthy gateway in priority order.

```java
@Service
public class PaymentGatewayRouter {

    private final Map<String, PaymentGateway> gatewayRegistry;
    private final RedisTemplate<String, Object> redis;
    private final PaymentRepository paymentRepo;
    private final Logger log = LoggerFactory.getLogger(PaymentGatewayRouter.class);

    // Gateway priority list: Stripe preferred, PayPal fallback, etc.
    private static final List<String> GATEWAY_PRIORITY = List.of("stripe", "paypal", "affirm");
    private static final String GATEWAY_HEALTH_KEY_PREFIX = "gateway:health:";

    @Autowired
    public PaymentGatewayRouter(
            StripeGatewayAdapter stripe,
            PayPalGatewayAdapter paypal,
            AffirmGatewayAdapter affirm) {
        this.gatewayRegistry = Map.of(
            "stripe", stripe,
            "paypal", paypal,
            "affirm", affirm
        );
    }

    public PaymentResult processPaymentWithFailover(AuthorizeRequest request)
            throws PaymentException {

        PaymentException lastException = null;

        for (String gatewayName : GATEWAY_PRIORITY) {
            try {
                if (!isGatewayHealthy(gatewayName)) {
                    log.warn("Gateway {} is unhealthy, skipping", gatewayName);
                    continue;
                }

                PaymentGateway gateway = gatewayRegistry.get(gatewayName);
                log.info("Attempting payment with gateway: {}", gatewayName);

                PaymentResult result = gateway.authorize(request);

                if (result.status() == PaymentStatus.AUTHORIZED) {
                    // Record successful attempt
                    recordPaymentAttempt(request, gatewayName, true);
                    return result;
                } else {
                    lastException = new PaymentException(result.message());
                }
            } catch (PaymentException e) {
                log.error("Payment failed with gateway {}: {}", gatewayName, e.getMessage());
                lastException = e;
                recordPaymentAttempt(request, gatewayName, false);
                // Continue to next gateway
            }
        }

        throw new PaymentException("All gateways failed", lastException);
    }

    private boolean isGatewayHealthy(String gatewayName) {
        String key = GATEWAY_HEALTH_KEY_PREFIX + gatewayName;
        Object scoreObj = redis.opsForValue().get(key);
        if (scoreObj == null) return true; // Default healthy

        int score = Integer.parseInt((String) scoreObj);
        return score > 50; // Health score 0-100, >50 = healthy
    }

    private void recordPaymentAttempt(AuthorizeRequest request, String gatewayName, boolean success) {
        // Store attempt in PostgreSQL for audit trail
        // Also update gateway health score based on success/failure
    }
}
```

---

### Q3: How do you ensure PCI compliance (not storing card numbers)?

**Design: Tokenization + Last-4 Storage**

Never accept or store raw card numbers. Instead:
1. Client sends card details to payment gateway's tokenization service (e.g., Stripe.js).
2. Gateway returns a token (e.g., `pm_1234567`).
3. Client sends token + last-4 to your server.
4. You store only the token + last-4 + gateway name in PostgreSQL.

```java
// Database entity: only stores safe data
@Entity
@Table(name = "payment_tokens")
public class PaymentToken {
    @Id
    private String id;

    private String customerId;
    private String gatewayToken;      // e.g., "pm_1234567"
    private String gatewayName;       // "stripe", "paypal"
    private String lastFourDigits;    // "4242"
    private String cardBrand;         // "visa", "mastercard"
    private String expiryMonth;       // "12"
    private String expiryYear;        // "2026"
    private Boolean isDefault;
    private Instant createdAt;
    private Instant deletedAt;
}

// API endpoint: accepts only token, never card number
@RestController
@RequestMapping("/api/payments")
public class PaymentController {

    private final PaymentGatewayRouter gatewayRouter;

    @PostMapping("/authorize")
    public ResponseEntity<PaymentResponse> authorizePayment(
            @RequestBody AuthorizePaymentRequest request) {

        // request contains: token (from client-side tokenization), amount, currency
        // NEVER accept: cardNumber, cvv, etc.

        if (request.token() == null || request.token().isEmpty()) {
            return ResponseEntity.badRequest().build();
        }

        try {
            var authRequest = new AuthorizeRequest(
                UUID.randomUUID().toString(),  // idempotencyKey
                request.token(),               // Token only, never raw card
                request.amount(),
                request.currency(),
                request.customerId(),
                Map.of()
            );

            PaymentResult result = gatewayRouter.processPaymentWithFailover(authRequest);
            return ResponseEntity.ok(new PaymentResponse(result.gatewayTransactionId(), result.status()));
        } catch (PaymentException e) {
            return ResponseEntity.status(402).body(
                new PaymentResponse(null, PaymentStatus.FAILED)
            );
        }
    }
}
```

---

### Q4: How do you handle currency conversion?

**Design: FX Rate Service with Redis Caching**

Maintain a currency conversion service that fetches daily rates from an external provider (e.g., ECB, Fixer.io) and caches them in Redis with 24-hour TTL.

```java
@Service
public class CurrencyConversionService {

    private final RestTemplate restTemplate;
    private final RedisTemplate<String, Object> redis;
    private final Logger log = LoggerFactory.getLogger(CurrencyConversionService.class);

    private static final String FX_RATE_KEY_PREFIX = "fx:rate:";
    private static final long FX_RATE_TTL_SECONDS = 86400; // 24 hours

    public BigDecimal convertToSettlementCurrency(
            BigDecimal amount,
            String sourceCurrency,
            String settlementCurrency) {

        if (sourceCurrency.equals(settlementCurrency)) {
            return amount;
        }

        BigDecimal rate = getExchangeRate(sourceCurrency, settlementCurrency);
        return amount.multiply(rate).setScale(2, RoundingMode.HALF_UP);
    }

    private BigDecimal getExchangeRate(String from, String to) {
        // Try cache first
        String cacheKey = FX_RATE_KEY_PREFIX + from + ":" + to;
        Object cached = redis.opsForValue().get(cacheKey);
        if (cached != null) {
            return new BigDecimal((String) cached);
        }

        // Fetch from external provider
        try {
            BigDecimal rate = fetchRateFromExternalProvider(from, to);
            redis.opsForValue().set(cacheKey, rate.toString(),
                Duration.ofSeconds(FX_RATE_TTL_SECONDS));
            return rate;
        } catch (Exception e) {
            log.error("Failed to fetch FX rate {}/{}: {}", from, to, e.getMessage());
            // Fallback to cached rate or throw
            throw new PaymentException("FX rate unavailable", e);
        }
    }

    private BigDecimal fetchRateFromExternalProvider(String from, String to) {
        // Call ECB API or similar
        String url = String.format("https://api.exchangerate-api.com/v4/latest/%s", from);
        try {
            var response = restTemplate.getForObject(url, Map.class);
            var rates = (Map<String, Object>) response.get("rates");
            return new BigDecimal(rates.get(to).toString());
        } catch (Exception e) {
            throw new PaymentException("FX provider error", e);
        }
    }

    @Scheduled(fixedDelay = 3600000) // Every hour
    public void refreshFXRates() {
        List<String> currencies = List.of("USD", "EUR", "GBP", "JPY", "CAD");
        for (int i = 0; i < currencies.size(); i++) {
            for (int j = 0; j < currencies.size(); j++) {
                if (i != j) {
                    String from = currencies.get(i);
                    String to = currencies.get(j);
                    try {
                        BigDecimal rate = fetchRateFromExternalProvider(from, to);
                        String cacheKey = FX_RATE_KEY_PREFIX + from + ":" + to;
                        redis.opsForValue().set(cacheKey, rate.toString(),
                            Duration.ofSeconds(FX_RATE_TTL_SECONDS));
                    } catch (Exception e) {
                        log.error("Failed to refresh rate {}/{}", from, to);
                    }
                }
            }
        }
    }
}
```

---

### Q5: How do you reconcile payments across multiple gateways?

**Design: Daily Batch Reconciliation Job**

Each gateway provides a settlement file (CSV/API) with all transactions settled that day. Match them against our internal transaction log.

```java
@Service
public class PaymentReconciliationJob {

    private final PaymentRepository paymentRepo;
    private final GatewaySettlementFetcher settlementFetcher;
    private final ReconciliationReportRepository reportRepo;
    private final Logger log = LoggerFactory.getLogger(PaymentReconciliationJob.class);

    @Scheduled(cron = "0 2 * * *") // Run at 2 AM daily
    public void reconcilePayments() {
        LocalDate yesterday = LocalDate.now().minusDays(1);
        log.info("Starting reconciliation for {}", yesterday);

        var report = new ReconciliationReport(yesterday, Instant.now());

        for (String gatewayName : List.of("stripe", "paypal", "affirm")) {
            try {
                reconcileGateway(gatewayName, yesterday, report);
            } catch (Exception e) {
                log.error("Reconciliation failed for gateway {}", gatewayName, e);
                report.markGatewayFailed(gatewayName, e.getMessage());
            }
        }

        reportRepo.save(report);
        notifyOnDiscrepancies(report);
    }

    private void reconcileGateway(String gatewayName, LocalDate date,
            ReconciliationReport report) throws Exception {

        // Fetch settlement file from gateway
        List<GatewaySettlementRecord> gatewayRecords =
            settlementFetcher.fetchSettlements(gatewayName, date);

        // Fetch our internal records for same date
        List<Payment> internalRecords = paymentRepo.findByGatewayAndDate(gatewayName, date);

        Map<String, Payment> internalMap = internalRecords.stream()
            .collect(Collectors.toMap(Payment::getGatewayTransactionId, p -> p));

        Set<String> matched = new HashSet<>();
        BigDecimal discrepancy = BigDecimal.ZERO;

        for (GatewaySettlementRecord gatewayRecord : gatewayRecords) {
            Payment payment = internalMap.get(gatewayRecord.transactionId());

            if (payment == null) {
                // Transaction in gateway but not in our DB (possible lag)
                log.warn("Missing transaction in DB: {}", gatewayRecord.transactionId());
                report.addMissing(gatewayRecord);
            } else if (payment.getAmount().equals(gatewayRecord.amount())) {
                matched.add(gatewayRecord.transactionId());
                report.incrementMatched();
            } else {
                // Amount mismatch (currency conversion issue, refund, etc.)
                log.warn("Amount mismatch for {}: internal={}, gateway={}",
                    gatewayRecord.transactionId(), payment.getAmount(), gatewayRecord.amount());
                discrepancy = discrepancy.add(payment.getAmount().subtract(gatewayRecord.amount()));
                report.addDiscrepancy(payment, gatewayRecord);
            }
        }

        // Find payments in our DB but not in gateway (possible pending)
        for (Payment payment : internalRecords) {
            if (!matched.contains(payment.getGatewayTransactionId())) {
                log.warn("Extra transaction in DB: {}", payment.getGatewayTransactionId());
                report.addExtra(payment);
            }
        }

        report.setGatewayTotalAmount(gatewayRecords.stream()
            .map(GatewaySettlementRecord::amount)
            .reduce(BigDecimal.ZERO, BigDecimal::add));
    }

    private void notifyOnDiscrepancies(ReconciliationReport report) {
        if (!report.getDiscrepancies().isEmpty() || !report.getMissing().isEmpty()) {
            // Send alert to operations team
            // Post to Slack, email, or internal dashboard
        }
    }
}
```

---

### Q6: How do you handle partial payments (gift card + credit card)?

**Design: PaymentSplit Transaction with Atomic Execution**

A single order may be paid using multiple methods. Create a `PaymentSplit` record that atomically processes each payment method in sequence, with compensation on partial failure.

```java
@Service
public class PartialPaymentOrchestrator {

    private final PaymentGatewayRouter gatewayRouter;
    private final GiftCardService giftCardService;
    private final PaymentRepository paymentRepo;
    private final PaymentSplitRepository splitRepo;
    private final Logger log = LoggerFactory.getLogger(PartialPaymentOrchestrator.class);

    public PaymentSplitResult processPartialPayment(PartialPaymentRequest request)
            throws PaymentException {

        // request contains: List<PaymentMethodRequest> with amount for each method

        var split = new PaymentSplit(
            request.orderId(),
            request.totalAmount(),
            PaymentSplitStatus.PENDING,
            Instant.now()
        );
        splitRepo.save(split);

        List<PaymentMethodResult> results = new ArrayList<>();
        BigDecimal totalChargedSoFar = BigDecimal.ZERO;

        try {
            for (PaymentMethodRequest method : request.paymentMethods()) {
                try {
                    PaymentMethodResult result = processPaymentMethod(method, split);
                    results.add(result);
                    totalChargedSoFar = totalChargedSoFar.add(result.chargedAmount());

                    if (totalChargedSoFar.compareTo(request.totalAmount()) >= 0) {
                        break; // All payment obtained
                    }
                } catch (PaymentException e) {
                    log.error("Payment method failed: {}", method.paymentMethodId(), e);
                    results.add(new PaymentMethodResult(method.paymentMethodId(),
                        BigDecimal.ZERO, PaymentStatus.FAILED, e.getMessage()));

                    // Check if we're at a critical failure point
                    if (results.stream().filter(r -> r.status() == PaymentStatus.FAILED).count()
                        >= request.paymentMethods().size() / 2) {
                        throw new PaymentException("Too many payment methods failed");
                    }
                }
            }

            if (totalChargedSoFar.compareTo(request.totalAmount()) < 0) {
                throw new PaymentException(String.format(
                    "Insufficient payment: charged %.2f, needed %.2f",
                    totalChargedSoFar, request.totalAmount()
                ));
            }

            split.setStatus(PaymentSplitStatus.COMPLETED);
            split.setTotalCharged(totalChargedSoFar);
            splitRepo.save(split);

            return new PaymentSplitResult(split.id(), results, PaymentSplitStatus.COMPLETED);

        } catch (PaymentException e) {
            // Compensation: refund all charged amounts
            compensatePartialPayment(results, split);
            split.setStatus(PaymentSplitStatus.FAILED);
            splitRepo.save(split);
            throw e;
        }
    }

    private PaymentMethodResult processPaymentMethod(PaymentMethodRequest method, PaymentSplit split)
            throws PaymentException {

        if (method.type() == PaymentMethodType.GIFT_CARD) {
            return processGiftCardPayment(method, split);
        } else if (method.type() == PaymentMethodType.CREDIT_CARD) {
            return processCreditCardPayment(method, split);
        } else if (method.type() == PaymentMethodType.WALLET) {
            return processWalletPayment(method, split);
        }

        throw new PaymentException("Unknown payment method type");
    }

    private PaymentMethodResult processGiftCardPayment(PaymentMethodRequest method, PaymentSplit split)
            throws PaymentException {

        try {
            var chargeResult = giftCardService.chargeCard(
                method.giftCardNumber(),
                method.amount()
            );

            var payment = new Payment(
                split.getId(),
                PaymentMethodType.GIFT_CARD,
                method.giftCardNumber(),
                method.amount(),
                "GIFT_CARD",
                chargeResult.transactionId(),
                PaymentStatus.AUTHORIZED,
                Instant.now()
            );
            paymentRepo.save(payment);

            return new PaymentMethodResult(method.paymentMethodId(), method.amount(),
                PaymentStatus.AUTHORIZED, "Gift card charged");
        } catch (Exception e) {
            throw new PaymentException("Gift card charge failed", e);
        }
    }

    private PaymentMethodResult processCreditCardPayment(PaymentMethodRequest method, PaymentSplit split)
            throws PaymentException {

        var authRequest = new AuthorizeRequest(
            split.getId() + ":" + method.paymentMethodId(),
            method.token(),
            method.amount(),
            split.getCurrency(),
            split.getCustomerId(),
            Map.of("splitId", split.getId().toString())
        );

        PaymentResult result = gatewayRouter.processPaymentWithFailover(authRequest);

        var payment = new Payment(
            split.getId(),
            PaymentMethodType.CREDIT_CARD,
            "****",
            method.amount(),
            method.gatewayName(),
            result.gatewayTransactionId(),
            result.status(),
            Instant.now()
        );
        paymentRepo.save(payment);

        if (result.status() != PaymentStatus.AUTHORIZED) {
            throw new PaymentException(result.message());
        }

        return new PaymentMethodResult(method.paymentMethodId(), method.amount(),
            result.status(), result.message());
    }

    private PaymentMethodResult processWalletPayment(PaymentMethodRequest method, PaymentSplit split)
            throws PaymentException {
        // Similar to credit card, but uses wallet token (Apple Pay, Google Pay)
        return null;
    }

    private void compensatePartialPayment(List<PaymentMethodResult> results, PaymentSplit split) {
        for (PaymentMethodResult result : results) {
            if (result.status() == PaymentStatus.AUTHORIZED) {
                try {
                    // Refund this payment
                    log.info("Compensating payment method {} for split {}",
                        result.paymentMethodId(), split.getId());
                    // Call appropriate refund service
                } catch (Exception e) {
                    log.error("Compensation failed for payment method {}",
                        result.paymentMethodId(), e);
                }
            }
        }
    }
}
```

---

## 5. Microservices Breakdown

| Service | Responsibility | Tech Stack | Key APIs |
|---------|-----------------|------------|----------|
| **Payment Gateway Service** | Route payments to gateways, handle failover, orchestrate authorization/capture | Java 17 + Spring Boot 3, Kafka | POST /authorize, POST /capture, GET /status |
| **Currency Service** | Manage FX rates, convert amounts, audit conversions | Java 17 + Spring Boot 3, Redis | GET /rate, POST /convert |
| **Gift Card Service** | Manage gift card balances, charge/refund, audit | Java 17 + Spring Boot 3, PostgreSQL | POST /charge, POST /refund, GET /balance |
| **Token Management Service** | Manage payment tokens, rotate, revoke | Java 17 + Spring Boot 3, PostgreSQL | POST /tokenize, DELETE /token/{id} |
| **Reconciliation Service** | Daily batch reconciliation, discrepancy reporting | Java 17 + Spring Boot 3, PostgreSQL, Kafka | GET /report, POST /reconcile |
| **Health Monitor Service** | Ping gateways, update health scores, alert on failures | Java 17 + Spring Boot 3, Redis, Kafka | GET /health/{gateway} |
| **Audit & Analytics Service** | Log all payment events, generate reports | Java 17 + Spring Boot 3, MongoDB, Kafka | GET /logs, POST /events |

---

## 6. Database Design

### PostgreSQL Schema

```sql
-- Core payment transaction
CREATE TABLE payments (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL,
    payment_method_type VARCHAR(50) NOT NULL,  -- CREDIT_CARD, GIFT_CARD, WALLET, BNPL
    gateway_name VARCHAR(50) NOT NULL,          -- stripe, paypal, affirm
    gateway_transaction_id VARCHAR(255) UNIQUE NOT NULL,
    amount NUMERIC(12, 2) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(50) NOT NULL,                -- AUTHORIZED, CAPTURED, FAILED, REFUNDED
    idempotency_key VARCHAR(255) UNIQUE,
    customer_id UUID NOT NULL,
    metadata JSONB,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW(),
    CONSTRAINT amount_positive CHECK (amount > 0)
);

CREATE INDEX idx_payments_order ON payments(order_id);
CREATE INDEX idx_payments_gateway_txn ON payments(gateway_name, gateway_transaction_id);
CREATE INDEX idx_payments_idempotency ON payments(idempotency_key);
CREATE INDEX idx_payments_created ON payments(created_at DESC);

-- Payment method tokens (PCI-safe)
CREATE TABLE payment_tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    customer_id UUID NOT NULL,
    gateway_name VARCHAR(50) NOT NULL,
    gateway_token VARCHAR(255) NOT NULL,     -- e.g., pm_1234567
    payment_method_type VARCHAR(50) NOT NULL,
    last_four_digits VARCHAR(4),
    card_brand VARCHAR(50),
    expiry_month VARCHAR(2),
    expiry_year VARCHAR(4),
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT NOW(),
    deleted_at TIMESTAMP,
    UNIQUE(customer_id, gateway_token),
    CONSTRAINT no_raw_card_data CHECK (gateway_token IS NOT NULL)
);

CREATE INDEX idx_tokens_customer ON payment_tokens(customer_id, deleted_at);

-- Partial payment splits
CREATE TABLE payment_splits (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    order_id UUID NOT NULL UNIQUE,
    total_amount NUMERIC(12, 2) NOT NULL,
    total_charged NUMERIC(12, 2),
    currency VARCHAR(3) NOT NULL DEFAULT 'USD',
    customer_id UUID NOT NULL,
    status VARCHAR(50) NOT NULL,         -- PENDING, COMPLETED, FAILED
    created_at TIMESTAMP DEFAULT NOW(),
    completed_at TIMESTAMP
);

CREATE TABLE payment_split_items (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    split_id UUID NOT NULL REFERENCES payment_splits(id),
    payment_id UUID REFERENCES payments(id),
    payment_method_type VARCHAR(50) NOT NULL,
    amount NUMERIC(12, 2) NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Reconciliation records
CREATE TABLE reconciliation_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    reconciliation_date DATE NOT NULL,
    gateway_name VARCHAR(50) NOT NULL,
    total_transactions_internal INT,
    total_transactions_gateway INT,
    matched_count INT,
    discrepancy_count INT,
    missing_count INT,
    total_discrepancy_amount NUMERIC(12, 2),
    status VARCHAR(50),                  -- COMPLETED, DISCREPANCIES_FOUND
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(reconciliation_date, gateway_name)
);

-- Gift card balances
CREATE TABLE gift_cards (
    id VARCHAR(50) PRIMARY KEY,          -- gift card number
    original_balance NUMERIC(12, 2) NOT NULL,
    current_balance NUMERIC(12, 2) NOT NULL,
    currency VARCHAR(3) DEFAULT 'USD',
    status VARCHAR(50),                  -- ACTIVE, EXPIRED, REVOKED
    expires_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Audit trail
CREATE TABLE payment_audit_log (
    id BIGSERIAL PRIMARY KEY,
    payment_id UUID NOT NULL,
    event_type VARCHAR(100),             -- CREATED, AUTHORIZED, CAPTURED, REFUNDED, FAILED
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    details JSONB,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_audit_payment ON payment_audit_log(payment_id);
```

### MongoDB Collections (Analytics & Logs)

```javascript
// Payment event log (for analytics)
db.createCollection("payment_events", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            properties: {
                _id: { bsonType: "objectId" },
                payment_id: { bsonType: "string" },
                order_id: { bsonType: "string" },
                event_type: { bsonType: "string" },  // authorized, captured, failed
                gateway: { bsonType: "string" },
                amount: { bsonType: "decimal" },
                currency: { bsonType: "string" },
                status: { bsonType: "string" },
                latency_ms: { bsonType: "int" },
                timestamp: { bsonType: "date" }
            }
        }
    }
});

db.payment_events.createIndex({ timestamp: -1 });
db.payment_events.createIndex({ payment_id: 1 });
db.payment_events.createIndex({ gateway: 1, timestamp: -1 });
```

---

## 7. Redis Data Structures

```java
// Gateway health scores (updated every 30 seconds)
HSET gateway:health stripe 85
HSET gateway:health paypal 92
HSET gateway:health affirm 45

// FX rate cache (24h TTL)
SET fx:rate:USD:EUR "0.92" EX 86400
SET fx:rate:EUR:GBP "0.86" EX 86400

// Token blacklist (24h TTL after deletion)
SET token:blacklist:pm_1234567 "revoked" EX 86400

// Rate limiting (per customer, per gateway)
INCR rate:limit:customer:123:stripe:requests
EXPIRE rate:limit:customer:123:stripe:requests 60

// Idempotency cache (24h TTL to prevent double-charge)
SET idempotency:order_123:split_1 "{\"status\":\"AUTHORIZED\",\"txn_id\":\"pm_abc\"}" EX 86400

// Ongoing transaction locks
SET lock:payment:order_123 "processing" EX 30

// Payment state machine cache
HSET payment:state:pay_123 status "PENDING" created_at "2026-04-01T10:00:00Z"
EXPIRE payment:state:pay_123 7200
```

**Key Commands:**

```java
// Health score update
redis.opsForHash().put("gateway:health", "stripe", "88");

// FX rate fetch with fallback
BigDecimal rate = redis.opsForValue().get("fx:rate:USD:EUR");

// Idempotency check
String cached = redis.opsForValue().getAndDelete("idempotency:" + idempotencyKey);
if (cached != null) { return cached; }

// Lock for mutual exclusion
Boolean locked = redis.opsForValue().setIfAbsent("lock:payment:" + orderId,
    "processing", Duration.ofSeconds(30));
if (!locked) { throw new PaymentException("Payment already in progress"); }
```

---

## 8. Kafka Event Flow

### Topics

```
payment.authorization.requested
  ├─ PaymentAuthorizationRequestedEvent
  │   - payment_id, amount, currency, gateway_name
  │   - Consumed by: Gateway Router, Audit Service, Analytics Service

payment.authorization.succeeded
  ├─ PaymentAuthorizedEvent
  │   - payment_id, gateway_transaction_id, authorized_amount
  │   - Consumed by: Order Service, Reconciliation Service, Audit Service

payment.authorization.failed
  ├─ PaymentAuthorizationFailedEvent
  │   - payment_id, reason, gateway_error_code
  │   - Consumed by: Order Service, Retry Service, Audit Service

payment.captured
  ├─ PaymentCapturedEvent
  │   - payment_id, captured_amount, settlement_amount
  │   - Consumed by: Accounting Service, Reconciliation Service

gateway.health.updated
  ├─ GatewayHealthUpdatedEvent
  │   - gateway_name, is_healthy, health_score, latency_ms
  │   - Consumed by: Health Monitor, Alerting Service

reconciliation.completed
  ├─ ReconciliationCompletedEvent
  │   - gateway_name, total_transactions, matched_count, discrepancies
  │   - Consumed by: Alert Service, Reporting Service
```

### Payload Examples

```json
// Event: payment.authorization.requested
{
  "event_id": "evt_pay_auth_001",
  "payment_id": "pay_123",
  "order_id": "order_456",
  "customer_id": "cust_789",
  "gateway_name": "stripe",
  "token": "pm_1234567",
  "amount": 99.99,
  "currency": "USD",
  "idempotency_key": "order_456:attempt_1",
  "metadata": {
    "order_items": [123, 124, 125],
    "user_agent": "Mozilla/5.0"
  },
  "timestamp": "2026-04-01T10:15:30Z"
}

// Event: payment.authorization.succeeded
{
  "event_id": "evt_pay_auth_success_001",
  "payment_id": "pay_123",
  "gateway_transaction_id": "pi_1234567abcdef",
  "gateway_name": "stripe",
  "authorized_amount": 99.99,
  "authorized_currency": "USD",
  "settlement_amount": 91.99,
  "settlement_currency": "EUR",
  "fx_rate": 0.92,
  "latency_ms": 850,
  "timestamp": "2026-04-01T10:15:31Z"
}

// Event: gateway.health.updated
{
  "event_id": "evt_health_stripe_001",
  "gateway_name": "stripe",
  "is_healthy": true,
  "health_score": 88,
  "error_rate_percent": 0.5,
  "latency_p99_ms": 2100,
  "last_success_count": 1098,
  "last_failure_count": 6,
  "timestamp": "2026-04-01T10:16:00Z"
}
```

---

## 9. Implementation Code

### Class 1: PaymentGatewayRouter (Main Orchestrator)

```java
package com.payment.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import com.payment.model.*;
import com.payment.repository.PaymentRepository;
import com.payment.event.PaymentAuthorizationRequestedEvent;
import com.payment.event.PaymentAuthorizedEvent;
import com.payment.event.PaymentAuthorizationFailedEvent;

import java.time.Instant;
import java.util.*;
import java.util.concurrent.TimeUnit;

@Service
public class PaymentGatewayRouter {

    private final Map<String, PaymentGateway> gatewayRegistry;
    private final PaymentRepository paymentRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private static final List<String> GATEWAY_PRIORITY =
        List.of("stripe", "paypal", "affirm");

    private final Logger log = LoggerFactory.getLogger(PaymentGatewayRouter.class);

    @Autowired
    public PaymentGatewayRouter(
            StripeGatewayAdapter stripe,
            PayPalGatewayAdapter paypal,
            AffirmGatewayAdapter affirm,
            PaymentRepository paymentRepository,
            RedisTemplate<String, Object> redisTemplate,
            KafkaTemplate<String, Object> kafkaTemplate) {

        this.gatewayRegistry = Map.of(
            "stripe", stripe,
            "paypal", paypal,
            "affirm", affirm
        );
        this.paymentRepository = paymentRepository;
        this.redisTemplate = redisTemplate;
        this.kafkaTemplate = kafkaTemplate;
    }

    public PaymentResult authorizePaymentWithFailover(AuthorizeRequest request)
            throws PaymentException {

        // Check idempotency
        String idempotencyKey = request.idempotencyKey();
        String cachedResult = (String) redisTemplate.opsForValue()
            .getAndDelete("idempotency:" + idempotencyKey);
        if (cachedResult != null) {
            log.info("Returning cached result for idempotency key: {}", idempotencyKey);
            return deserializePaymentResult(cachedResult);
        }

        // Publish event: authorization requested
        publishEvent("payment.authorization.requested",
            new PaymentAuthorizationRequestedEvent(
                idempotencyKey, request.token(), request.amount(),
                request.currency(), Instant.now()
            ));

        PaymentException lastException = null;

        for (String gatewayName : GATEWAY_PRIORITY) {
            try {
                if (!isGatewayHealthy(gatewayName)) {
                    log.warn("Gateway {} is unhealthy, skipping", gatewayName);
                    continue;
                }

                PaymentGateway gateway = gatewayRegistry.get(gatewayName);
                log.info("Attempting authorization with gateway: {}", gatewayName);

                long startTime = System.currentTimeMillis();
                PaymentResult result = gateway.authorize(request);
                long latency = System.currentTimeMillis() - startTime;

                if (result.status() == PaymentStatus.AUTHORIZED) {
                    // Cache for idempotency
                    redisTemplate.opsForValue().set(
                        "idempotency:" + idempotencyKey,
                        serializePaymentResult(result),
                        24,
                        TimeUnit.HOURS
                    );

                    // Persist to DB
                    savePayment(request, gatewayName, result);

                    // Publish success event
                    publishEvent("payment.authorization.succeeded",
                        new PaymentAuthorizedEvent(
                            result.gatewayTransactionId(), request.amount(),
                            request.currency(), latency, Instant.now()
                        ));

                    return result;
                } else {
                    lastException = new PaymentException(
                        "Authorization failed: " + result.message());
                }
            } catch (PaymentException e) {
                log.error("Payment failed with gateway {}: {}", gatewayName,
                    e.getMessage());
                lastException = e;
            }
        }

        // All gateways failed
        publishEvent("payment.authorization.failed",
            new PaymentAuthorizationFailedEvent(
                idempotencyKey, lastException.getMessage(), Instant.now()
            ));

        throw new PaymentException("All gateways failed", lastException);
    }

    private boolean isGatewayHealthy(String gatewayName) {
        Object scoreObj = redisTemplate.opsForHash()
            .get("gateway:health", gatewayName);
        if (scoreObj == null) return true;

        int score = Integer.parseInt((String) scoreObj);
        return score > 50;
    }

    private void savePayment(AuthorizeRequest request, String gatewayName,
            PaymentResult result) {
        var payment = new Payment();
        payment.setGatewayName(gatewayName);
        payment.setGatewayTransactionId(result.gatewayTransactionId());
        payment.setAmount(request.amount());
        payment.setCurrency(request.currency());
        payment.setStatus(result.status());
        payment.setIdempotencyKey(request.idempotencyKey());
        payment.setCustomerId(request.customerId());
        payment.setCreatedAt(Instant.now());
        paymentRepository.save(payment);
    }

    private void publishEvent(String topic, Object event) {
        kafkaTemplate.send(topic, UUID.randomUUID().toString(), event);
    }

    private String serializePaymentResult(PaymentResult result) {
        // JSON serialization
        return "{\"txnId\":\"" + result.gatewayTransactionId() + "\"}";
    }

    private PaymentResult deserializePaymentResult(String json) {
        // JSON deserialization
        return new PaymentResult("txnId", PaymentStatus.AUTHORIZED, "Cached", Instant.now());
    }
}
```

### Class 2: StripeGatewayAdapter

```java
package com.payment.adapter;

import com.stripe.StripeClient;
import com.stripe.model.PaymentIntent;
import org.springframework.stereotype.Component;
import com.payment.model.*;

@Component
public class StripeGatewayAdapter implements PaymentGateway {

    private final StripeClient stripeClient;
    private final Logger log = LoggerFactory.getLogger(StripeGatewayAdapter.class);

    @Autowired
    public StripeGatewayAdapter(StripeClient stripeClient) {
        this.stripeClient = stripeClient;
    }

    @Override
    public PaymentResult authorize(AuthorizeRequest request) throws PaymentException {
        try {
            var params = new StripePaymentIntentParams()
                .setAmount(request.amount().multiply(BigDecimal.valueOf(100)).longValue())
                .setCurrency(request.currency().toLowerCase())
                .setPaymentMethod(request.token())
                .setCustomerId(request.customerId())
                .setConfirm(true)
                .setReturnUrl("https://api.example.com/stripe/return")
                .setIdempotencyKey(request.idempotencyKey());

            PaymentIntent intent = stripeClient.createPaymentIntent(params);

            PaymentStatus status = mapStripeStatus(intent.status());
            String message = intent.chargeError() != null
                ? intent.chargeError().message()
                : "Success";

            return new PaymentResult(
                intent.id(),
                status,
                message,
                Instant.now()
            );
        } catch (StripeException e) {
            log.error("Stripe API error: {}", e.getMessage());
            throw new PaymentException("Stripe authorization failed", e);
        }
    }

    @Override
    public GatewayStatus healthCheck() throws PaymentException {
        try {
            long startTime = System.currentTimeMillis();
            stripeClient.ping();
            long latency = System.currentTimeMillis() - startTime;
            return new GatewayStatus(true, latency, Instant.now());
        } catch (Exception e) {
            log.error("Stripe health check failed: {}", e.getMessage());
            return new GatewayStatus(false, 0, Instant.now());
        }
    }

    private PaymentStatus mapStripeStatus(String stripeStatus) {
        return switch (stripeStatus) {
            case "requires_payment_method" -> PaymentStatus.PENDING;
            case "succeeded" -> PaymentStatus.AUTHORIZED;
            case "processing" -> PaymentStatus.PENDING;
            default -> PaymentStatus.FAILED;
        };
    }
}
```

### Class 3: CurrencyConversionService

```java
package com.payment.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.web.client.RestTemplate;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.Duration;
import java.util.List;

@Service
public class CurrencyConversionService {

    private final RestTemplate restTemplate;
    private final RedisTemplate<String, Object> redisTemplate;
    private final Logger log = LoggerFactory.getLogger(CurrencyConversionService.class);

    private static final String FX_RATE_KEY_PREFIX = "fx:rate:";
    private static final long FX_RATE_TTL_SECONDS = 86400;

    @Autowired
    public CurrencyConversionService(RestTemplate restTemplate,
            RedisTemplate<String, Object> redisTemplate) {
        this.restTemplate = restTemplate;
        this.redisTemplate = redisTemplate;
    }

    public BigDecimal convertToSettlementCurrency(BigDecimal amount,
            String sourceCurrency, String settlementCurrency) {

        if (sourceCurrency.equals(settlementCurrency)) {
            return amount;
        }

        BigDecimal rate = getExchangeRate(sourceCurrency, settlementCurrency);
        return amount.multiply(rate).setScale(2, RoundingMode.HALF_UP);
    }

    private BigDecimal getExchangeRate(String from, String to) {
        String cacheKey = FX_RATE_KEY_PREFIX + from + ":" + to;
        Object cached = redisTemplate.opsForValue().get(cacheKey);

        if (cached != null) {
            return new BigDecimal((String) cached);
        }

        try {
            BigDecimal rate = fetchRateFromProvider(from, to);
            redisTemplate.opsForValue().set(cacheKey, rate.toString(),
                Duration.ofSeconds(FX_RATE_TTL_SECONDS));
            return rate;
        } catch (Exception e) {
            log.error("Failed to fetch FX rate {}/{}", from, to, e);
            throw new PaymentException("FX rate unavailable", e);
        }
    }

    private BigDecimal fetchRateFromProvider(String from, String to) {
        String url = String.format(
            "https://api.exchangerate-api.com/v4/latest/%s", from);

        try {
            Map<String, Object> response = restTemplate.getForObject(url, Map.class);
            @SuppressWarnings("unchecked")
            Map<String, Object> rates = (Map<String, Object>) response.get("rates");
            return new BigDecimal(rates.get(to).toString());
        } catch (Exception e) {
            throw new PaymentException("FX provider error", e);
        }
    }

    @Scheduled(fixedDelay = 3600000)
    public void refreshFXRatesScheduled() {
        log.info("Refreshing FX rates");
        List<String> currencies = List.of("USD", "EUR", "GBP", "JPY", "CAD");

        for (String from : currencies) {
            for (String to : currencies) {
                if (!from.equals(to)) {
                    try {
                        BigDecimal rate = fetchRateFromProvider(from, to);
                        String cacheKey = FX_RATE_KEY_PREFIX + from + ":" + to;
                        redisTemplate.opsForValue().set(cacheKey, rate.toString(),
                            Duration.ofSeconds(FX_RATE_TTL_SECONDS));
                    } catch (Exception e) {
                        log.error("Failed to refresh rate {}/{}", from, to);
                    }
                }
            }
        }
    }
}
```

### Class 4: GatewayHealthMonitor

```java
package com.payment.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import com.payment.model.GatewayStatus;
import com.payment.event.GatewayHealthUpdatedEvent;

import java.time.Instant;
import java.util.Map;

@Service
public class GatewayHealthMonitor {

    private final Map<String, PaymentGateway> gatewayRegistry;
    private final RedisTemplate<String, Object> redisTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final Logger log = LoggerFactory.getLogger(GatewayHealthMonitor.class);

    @Autowired
    public GatewayHealthMonitor(
            Map<String, PaymentGateway> gatewayRegistry,
            RedisTemplate<String, Object> redisTemplate,
            KafkaTemplate<String, Object> kafkaTemplate) {
        this.gatewayRegistry = gatewayRegistry;
        this.redisTemplate = redisTemplate;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Scheduled(fixedDelay = 30000)
    public void monitorGatewayHealth() {
        for (Map.Entry<String, PaymentGateway> entry : gatewayRegistry.entrySet()) {
            String gatewayName = entry.getKey();
            PaymentGateway gateway = entry.getValue();

            try {
                GatewayStatus status = gateway.healthCheck();
                updateHealthScore(gatewayName, status);
                publishHealthEvent(gatewayName, status);
            } catch (Exception e) {
                log.error("Health check failed for gateway {}: {}",
                    gatewayName, e.getMessage());
                updateHealthScoreOnFailure(gatewayName);
            }
        }
    }

    private void updateHealthScore(String gatewayName, GatewayStatus status) {
        int newScore = status.isHealthy() ? 100 : 0;
        redisTemplate.opsForHash().put("gateway:health", gatewayName, String.valueOf(newScore));
    }

    private void updateHealthScoreOnFailure(String gatewayName) {
        Object scoreObj = redisTemplate.opsForHash().get("gateway:health", gatewayName);
        int currentScore = scoreObj != null ? Integer.parseInt((String) scoreObj) : 100;
        int newScore = Math.max(0, currentScore - 20);
        redisTemplate.opsForHash().put("gateway:health", gatewayName, String.valueOf(newScore));
    }

    private void publishHealthEvent(String gatewayName, GatewayStatus status) {
        var event = new GatewayHealthUpdatedEvent(
            gatewayName,
            status.isHealthy(),
            (int) status.latencyMs(),
            Instant.now()
        );
        kafkaTemplate.send("gateway.health.updated", gatewayName, event);
    }
}
```

### Class 5: PaymentReconciliationJob

```java
package com.payment.job;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import com.payment.repository.*;
import com.payment.model.*;

import java.time.LocalDate;
import java.util.*;

@Service
public class PaymentReconciliationJob {

    private final PaymentRepository paymentRepository;
    private final ReconciliationReportRepository reportRepository;
    private final GatewaySettlementFetcher settlementFetcher;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final Logger log = LoggerFactory.getLogger(PaymentReconciliationJob.class);

    @Autowired
    public PaymentReconciliationJob(
            PaymentRepository paymentRepository,
            ReconciliationReportRepository reportRepository,
            GatewaySettlementFetcher settlementFetcher,
            KafkaTemplate<String, Object> kafkaTemplate) {
        this.paymentRepository = paymentRepository;
        this.reportRepository = reportRepository;
        this.settlementFetcher = settlementFetcher;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Scheduled(cron = "0 2 * * *")
    public void reconcilePayments() {
        LocalDate yesterday = LocalDate.now().minusDays(1);
        log.info("Starting reconciliation for {}", yesterday);

        var report = new ReconciliationReport(yesterday);

        for (String gateway : List.of("stripe", "paypal", "affirm")) {
            try {
                reconcileGateway(gateway, yesterday, report);
            } catch (Exception e) {
                log.error("Reconciliation failed for gateway {}", gateway, e);
                report.markGatewayFailed(gateway, e.getMessage());
            }
        }

        reportRepository.save(report);
        publishReconciliationEvent(report);
    }

    private void reconcileGateway(String gatewayName, LocalDate date,
            ReconciliationReport report) throws Exception {

        List<GatewaySettlementRecord> gatewayRecords =
            settlementFetcher.fetchSettlements(gatewayName, date);

        List<Payment> internalRecords = paymentRepository
            .findByGatewayAndDate(gatewayName, date);

        Map<String, Payment> internalMap = internalRecords.stream()
            .collect(Collectors.toMap(Payment::getGatewayTransactionId, p -> p));

        for (GatewaySettlementRecord record : gatewayRecords) {
            Payment payment = internalMap.get(record.transactionId());

            if (payment == null) {
                log.warn("Missing in DB: {}", record.transactionId());
                report.addMissing(record);
            } else if (payment.getAmount().equals(record.amount())) {
                report.incrementMatched();
            } else {
                log.warn("Amount mismatch: {} vs {}",
                    payment.getAmount(), record.amount());
                report.addDiscrepancy(payment, record);
            }
        }
    }

    private void publishReconciliationEvent(ReconciliationReport report) {
        kafkaTemplate.send("reconciliation.completed",
            UUID.randomUUID().toString(), report);
    }
}
```

### Class 6: PartialPaymentOrchestrator

```java
package com.payment.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.payment.repository.*;
import com.payment.model.*;

import java.math.BigDecimal;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.TimeUnit;

@Service
public class PartialPaymentOrchestrator {

    private final PaymentGatewayRouter gatewayRouter;
    private final GiftCardService giftCardService;
    private final PaymentRepository paymentRepository;
    private final PaymentSplitRepository splitRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    private final Logger log = LoggerFactory.getLogger(PartialPaymentOrchestrator.class);

    @Autowired
    public PartialPaymentOrchestrator(
            PaymentGatewayRouter gatewayRouter,
            GiftCardService giftCardService,
            PaymentRepository paymentRepository,
            PaymentSplitRepository splitRepository,
            RedisTemplate<String, Object> redisTemplate) {
        this.gatewayRouter = gatewayRouter;
        this.giftCardService = giftCardService;
        this.paymentRepository = paymentRepository;
        this.splitRepository = splitRepository;
        this.redisTemplate = redisTemplate;
    }

    @Transactional
    public PaymentSplitResult processPartialPayment(PartialPaymentRequest request)
            throws PaymentException {

        String lockKey = "lock:split:" + request.orderId();
        Boolean locked = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "processing", 30, TimeUnit.SECONDS);

        if (!locked) {
            throw new PaymentException("Payment already in progress");
        }

        var split = new PaymentSplit(request.orderId(), request.totalAmount());
        split.setStatus(PaymentSplitStatus.PENDING);
        split.setCreatedAt(Instant.now());
        splitRepository.save(split);

        List<PaymentMethodResult> results = new ArrayList<>();
        BigDecimal totalCharged = BigDecimal.ZERO;

        try {
            for (PaymentMethodRequest method : request.paymentMethods()) {
                try {
                    PaymentMethodResult result = processMethod(method, split);
                    results.add(result);
                    totalCharged = totalCharged.add(result.chargedAmount());

                    if (totalCharged.compareTo(request.totalAmount()) >= 0) break;
                } catch (PaymentException e) {
                    log.error("Method {} failed: {}", method.paymentMethodId(), e);
                    results.add(new PaymentMethodResult(
                        method.paymentMethodId(), BigDecimal.ZERO,
                        PaymentStatus.FAILED, e.getMessage()
                    ));
                }
            }

            if (totalCharged.compareTo(request.totalAmount()) < 0) {
                throw new PaymentException("Insufficient payment");
            }

            split.setStatus(PaymentSplitStatus.COMPLETED);
            split.setTotalCharged(totalCharged);
            split.setCompletedAt(Instant.now());
            splitRepository.save(split);

            return new PaymentSplitResult(split.getId(), results,
                PaymentSplitStatus.COMPLETED);
        } catch (PaymentException e) {
            compensate(results);
            split.setStatus(PaymentSplitStatus.FAILED);
            splitRepository.save(split);
            throw e;
        } finally {
            redisTemplate.delete(lockKey);
        }
    }

    private PaymentMethodResult processMethod(PaymentMethodRequest method,
            PaymentSplit split) throws PaymentException {

        if (method.type() == PaymentMethodType.GIFT_CARD) {
            return processGiftCard(method, split);
        } else if (method.type() == PaymentMethodType.CREDIT_CARD) {
            return processCreditCard(method, split);
        }

        throw new PaymentException("Unknown method type");
    }

    private PaymentMethodResult processGiftCard(PaymentMethodRequest method,
            PaymentSplit split) throws PaymentException {

        var chargeResult = giftCardService.chargeCard(
            method.giftCardNumber(), method.amount());

        var payment = new Payment();
        payment.setPaymentMethodType(PaymentMethodType.GIFT_CARD);
        payment.setAmount(method.amount());
        payment.setStatus(PaymentStatus.AUTHORIZED);
        paymentRepository.save(payment);

        return new PaymentMethodResult(method.paymentMethodId(), method.amount(),
            PaymentStatus.AUTHORIZED, "Gift card charged");
    }

    private PaymentMethodResult processCreditCard(PaymentMethodRequest method,
            PaymentSplit split) throws PaymentException {

        var authRequest = new AuthorizeRequest(
            split.getId() + ":" + method.paymentMethodId(),
            method.token(),
            method.amount(),
            "USD",
            split.getCustomerId(),
            Map.of()
        );

        PaymentResult result = gatewayRouter.authorizePaymentWithFailover(authRequest);

        if (result.status() != PaymentStatus.AUTHORIZED) {
            throw new PaymentException(result.message());
        }

        return new PaymentMethodResult(method.paymentMethodId(), method.amount(),
            result.status(), result.message());
    }

    private void compensate(List<PaymentMethodResult> results) {
        for (PaymentMethodResult result : results) {
            if (result.status() == PaymentStatus.AUTHORIZED) {
                try {
                    log.info("Compensating payment method {}", result.paymentMethodId());
                    // Call refund service
                } catch (Exception e) {
                    log.error("Compensation failed", e);
                }
            }
        }
    }
}
```

---

## 10. Failure Scenarios & Mitigations

| Failure Scenario | Detection Method | Mitigation Strategy |
|---|---|---|
| **Primary gateway (Stripe) timeout** | HTTP timeout after 5s | Automatic failover to PayPal within 2s; publish `gateway.health.updated` with reduced score |
| **All gateways fail** | All priority gateways exhausted | Return HTTP 503; retry in background (max 3x); publish failure event to Kafka; notify ops via Slack |
| **Network partition to Redis** | Redis connection pool exhausted | Degrade to in-memory health cache; log warning; alert DevOps |
| **Duplicate charge (retry) after auth** | Idempotency key check before authorize | Return cached result from Redis; publish audit event |
| **Authorization expires before capture** | Scheduled job scans authorizations > 6 days old | Re-authorize or reject capture; create manual review ticket |
| **Payment success but customer refund request immediately** | Manual review within 24h | Process refund to original method; reconcile next day |
| **FX rate unavailable** | External provider API down | Use last cached rate + warn in logs; escalate if >1h stale |
| **Partial refund fails after full refund posted** | Double-check refund status before processing | Idempotent refund by (orderId, refundType); manual reconciliation |
| **Reconciliation shows missing transactions** | Daily reconciliation report | Investigate lag in gateway settlement; contact gateway support; mark for manual review |
| **Gift card balance underflow** | Balance < amount to charge | Reject charge; suggest alternative payment method |
| **PostgreSQL replication lag during split payment** | Replica read after write in split service | Enforce write-all-reads-coordinator; use strong consistency for critical paths |
| **Kafka broker down (event publishing fails)** | Kafka send timeout | Log to PostgreSQL audit table as fallback; retry Kafka publish periodically |

---

## 11. Scaling Strategy

### Horizontal Scaling

1. **Payment Gateway Router Service:**
   - Deploy behind Spring Cloud Load Balancer or Nginx.
   - Each instance handles ~5 TPS (100K/day / 86400 * 5x burst = 11.6 TPS → 3 instances nominal, 5 under burst).
   - Share Redis and PostgreSQL across instances.

2. **Reconciliation Job:**
   - Single instance (scheduled job); can be split across multiple jobs for different gateways if needed.
   - Distributed locking via Redis `SET lock:reconcile:{gateway} NX EX 3600` to prevent concurrent runs.

3. **Health Monitor Service:**
   - Deploy as single-instance service; health checks are I/O-bound, not compute-bound.
   - Can be replicated for HA; use Redis to coordinate (only one instance pings at a time).

### Vertical Scaling

1. **Increase PostgreSQL `max_connections`** from default 100 to 200+ for connection pooling.
2. **Increase Redis memory** to accommodate 24h FX rate cache + idempotency cache (estimate ~500MB for 100K/day volume).
3. **Increase Kafka broker `num.network.threads`** and `num.io.threads` for 200K events/day.

### Caching & Data Structure Optimization

- **Payment lookup:** Index on `gateway_transaction_id` + `created_at` for rapid lookups.
- **Idempotency:** TTL 24h in Redis; auto-expire to save memory.
- **Health scores:** In-memory map refreshed every 30s from Redis.

### Database Optimization

```sql
-- Partition transactions by date for faster queries
ALTER TABLE payments PARTITION BY RANGE (YEAR(created_at), MONTH(created_at)) (
    PARTITION p_202604 VALUES LESS THAN (2026, 5),
    PARTITION p_202605 VALUES LESS THAN (2026, 6)
);

-- Archive old data (>6 months) to cold storage
CREATE TABLE payments_archive AS
SELECT * FROM payments WHERE created_at < NOW() - INTERVAL 6 MONTH;
```

---

## 12. Monitoring & Observability

### Key Metrics

```
Payment Authorization Metrics:
  - payment.authorize.latency_ms (p50, p95, p99)
  - payment.authorize.success_rate
  - payment.authorize.gateway_failures (per gateway)
  - payment.authorize.idempotency_cache_hits

Payment Processing Metrics:
  - payment.capture.latency_ms
  - payment.capture.success_rate
  - payment.refund.success_rate

Gateway Health:
  - gateway.health_score (per gateway, 0-100)
  - gateway.latency_ms (p99)
  - gateway.error_rate_percent

Reconciliation Metrics:
  - reconciliation.matched_count
  - reconciliation.discrepancy_amount_usd
  - reconciliation.missing_transactions

System Health:
  - redis.connection_pool.active
  - postgres.connection_pool.active
  - kafka.producer.error_rate
  - idempotency_cache.hit_rate
```

### Alerts

```yaml
PaymentAuthorizationFailure:
  expr: rate(payment_authorize_failures[5m]) > 0.1  # > 0.1 req/sec fail rate
  for: 5m
  action: page on-call engineer

GatewayDown:
  expr: gateway_health_score < 50
  for: 2m
  action: alert to DevOps; start failover

ReconciliationDiscrepancies:
  expr: reconciliation_discrepancy_amount > 1000  # >$1000 discrepancy
  for: 1h
  action: email finance team

HighLatency:
  expr: payment_authorize_latency_p99 > 3000  # >3s
  for: 10m
  action: page SRE to investigate
```

### Tracing

Use OpenTelemetry to trace each payment request:

```java
@Service
public class PaymentGatewayRouter {
    private final Tracer tracer;

    public PaymentResult authorizePaymentWithFailover(AuthorizeRequest request) {
        try (var span = tracer.spanBuilder("authorize_payment")
                .setAttribute("order_id", request.customerId())
                .setAttribute("amount", request.amount().toString())
                .startSpan()) {

            try (var scope = span.makeCurrent()) {
                // Payment logic
            }
        }
    }
}
```

---

## 13. Summary Cheat Sheet

| Decision | Choice | Rationale |
|---|---|---|
| **Gateway Abstraction** | Strategy + Adapter pattern | Decouples core logic from gateway specifics; easy to add new gateways |
| **Failover** | Priority list + health scores in Redis | Fast failover (5s SLA); multiple gateways can be tried in sequence |
| **PCI Compliance** | Client-side tokenization + store token only | Never handle raw card numbers; meets Level 1 PCI requirements |
| **Idempotency** | Redis cache (24h TTL) + unique DB constraint | Prevents double-charges on network retries |
| **Currency Conversion** | FX rate service with Redis cache (24h TTL) | Real-time rates; cached to avoid latency; 1h refresh job |
| **Partial Payments** | PaymentSplit entity + atomic transaction | Tracks multiple payment methods; compensation on failure |
| **Reconciliation** | Daily batch job matching internal vs gateway records | Detects discrepancies; prevents leakage; posts to Kafka for accounting |
| **Event Broadcasting** | Kafka topics (payment.*, gateway.*, reconciliation.*) | Decouples services; enables audit trail and downstream processing |
| **Health Monitoring** | 30s-interval health checks; Redis hash for scores | Detects outages quickly; enables intelligent failover |
| **Scaling** | 3–5 router instances behind load balancer; single reconciliation job | Handles 100K txns/day at 99.9% uptime |
| **Database** | PostgreSQL (txns, tokens) + MongoDB (analytics logs) | ACID for payments; flexible schema for event logs |
| **Caching** | Redis for health scores, FX rates, idempotency cache | Sub-millisecond lookups; TTL-based expiration |
| **Error Handling** | Publish events + manual review tickets for edge cases | Audit trail; ops visibility; prevents silent failures |

---

**End of File**
