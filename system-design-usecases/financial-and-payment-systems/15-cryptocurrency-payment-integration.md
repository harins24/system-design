---
title: Cryptocurrency Payment Integration
layout: default
---

# Cryptocurrency Payment Integration — Deep Dive Design

> **Scenario:** Accept Bitcoin + Ethereum for purchases. Convert crypto to fiat immediately (avoid volatility). Handle confirmation delays (10-60 min). Refund in crypto or fiat. AML/KYC compliance. Secure wallet and private key management.
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
- Accept Bitcoin (BTC) and Ethereum (ETH) as payment methods
- Display real-time crypto prices with 15-minute lock window
- Handle blockchain confirmation delays (BTC: 3 confirmations, ETH: 12 confirmations)
- Immediately convert confirmed crypto to USD (avoid volatility)
- Process orders in `PENDING_CRYPTO_CONFIRMATION` state until blockchain confirms
- Support crypto or fiat refunds (at merchant's choice)
- AML/KYC: screen wallet addresses against OFAC/Chainalysis blacklists
- Require KYC for transactions >$3,000
- Secure wallet management: HSM or MPC wallets (no private keys in application)
- Generate audit trail for regulatory compliance

### Non-Functional Requirements
- Payment processor integration: CoinGate, BitPay, or Coinbase Commerce
- Confirmation checking: polling every 10 sec or webhook-based
- Price quote validity: 15 minutes (re-quote if unconfirmed after 15 min)
- Order confirmation rate: >99.5% (minimal failed conversions)
- KYC verification latency: <30 sec for automated checks; <24 hours for manual review
- AML transaction screening: <2 sec
- Refund processing: <24 hours for fiat, <1 hour for crypto transfer initiation
- Data retention: 7 years for regulatory audit trail

### Constraints
- Blockchain confirmation times: 10-60 minutes (variable network conditions)
- Network fees: variable; merchant absorbs or passes to customer
- Price volatility: lock quote for 15 min; re-quote if unconfirmed
- KYC/AML: SOX/GDPR compliant; limit data retention
- Private key security: never expose in logs or memory
- Wallet address validation: ensure checksum correctness

---

## Capacity Estimation

| Metric | Value |
|--------|-------|
| Crypto Transactions/Day | ~5,000 |
| Peak Transactions/Minute | 100 |
| BTC Transactions | 60% of total |
| ETH Transactions | 40% of total |
| Avg Transaction Value | $500 |
| High-Value Txns (>$3K requiring KYC) | 20% |
| Price Quote Requests/Day | ~50K |
| Blockchain Poll Cycles (avg 20 min duration) | ~100K |
| Confirmation Check Latency (p95) | <2 sec |
| KYC Verification API Calls/Day | ~1K |
| Refund Requests/Day | ~100 |
| PostgreSQL DB Size (YoY) | ~200 GB |
| MongoDB Document Volume | ~10M (transaction history, KYC data) |
| Redis Memory (peak) | ~5 GB (price quotes, confirmation state, locks) |
| Crypto to Fiat Conversion Rate | 99.5% |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│              Cryptocurrency Payment Integration System                   │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                    Frontend (Checkout Page)                              │
│  - Crypto Payment Option   - QR Code (BTC/ETH Address)   - Price Display│
└──────────────────────────────────────────────────────────────────────────┘
                                   │
┌──────────────────────────────────┴──────────────────────────────────────┐
│                         API Gateway                                      │
│              - Authentication, Rate Limiting, Validation                │
└──────────────────────────────────┬──────────────────────────────────────┘
                    ┌───────────────┼───────────────┐
                    │               │               │
        ┌───────────▼─────┐  ┌──────▼──────┐  ┌───▼──────┐
        │ Payment Service │  │ KYC/AML     │  │ Confirmation
        │ - Create order  │  │ Service     │  │ Checker
        │ - Price quote   │  │ - Screening │  │ - Polling
        │ - Receive addr  │  │ - Verification│ │ - Webhooks
        └─────────┬───────┘  └──────┬──────┘  └────┬─────┘
                  │                 │              │
        ┌─────────▼──────────────────▼──────────────▼─────────┐
        │              Kafka (Event Bus)                      │
        │  - payment.received_quote                          │
        │  - payment.crypto_confirmed                         │
        │  - payment.fiat_converted                           │
        │  - payment.refund_initiated                         │
        │  - kyc.verification_required                        │
        │  - aml.screening_passed                             │
        └──────────────┬────────────────────────────────────┘
                       │
    ┌──────────────────┼─────────────────┬─────────────┐
    │                  │                 │             │
 ┌──▼──┐     ┌────────▼────┐  ┌─────────▼──┐  ┌──────▼─┐
 │Crypto│     │  Fiat       │  │ Wallet     │  │Refund  │
 │Adapter    │  Conversion  │  │ Management │  │Service │
 │(Coingate) │  Service     │  │ Service    │  │        │
 └──────┘     └─────────────┘  └────────────┘  └────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│                          Data & External Services                        │
│  ┌────────────────┐  ┌──────────┐  ┌────────┐  ┌──────────────────┐   │
│  │   PostgreSQL   │  │ MongoDB  │  │ Redis  │  │ External APIs    │   │
│  │ - Crypto TX    │  │ - Wallet │  │ - Quote│  │ - Chainanalysis  │   │
│  │ - Orders       │  │   Addrs  │  │ - State│  │ - OFAC Screening │   │
│  │ - KYC Data     │  │ - Events │  │ - Locks│  │ - CoinGate/BitPay│   │
│  │ - Refunds      │  │          │  │        │  │ - Price Feed     │   │
│  │ - Audit Log    │  │          │  │        │  │ - Blockchain RPC │   │
│  └────────────────┘  └──────────┘  └────────┘  └──────────────────┘   │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Core Design Questions Answered

### 1. How do you integrate with crypto payment processors?

**Answer:** Adapter pattern with fallback support.

- Define `CryptoPaymentGateway` interface:
  - `getAddress(orderID, currency)` → returns unique deposit address
  - `getPrice()` → returns current BTC/ETH price
  - `checkConfirmation(txHash, minConfirmations)` → polls blockchain or uses webhook
- Implement adapters for CoinGate, BitPay, Coinbase Commerce
- Primary: CoinGate; Secondary: BitPay (fallback if primary down)
- Store adapter config in PostgreSQL; switch via feature flag
- Use circuit breaker pattern (Hystrix/Resilience4j) for fault tolerance

### 2. How do you handle confirmation delays without blocking orders?

**Answer:** Async state machine with polling/webhooks.

- Order states: `PENDING_CRYPTO_CONFIRMATION` → `CRYPTO_CONFIRMED` → `FIAT_CONVERTED` → `FULFILLED`
- On payment received: store in Redis: `order:{id}:crypto_state = {tx_hash, amount, confirmations}`
- Confirmation checker: background job (Kafka consumer) polling every 10 seconds
- On N confirmations reached: emit `payment.crypto_confirmed` event
- Downstream: convert to fiat, capture payment, fulfil order
- Latency: user sees "Awaiting confirmation" with progress indicator (N/12 for ETH)

### 3. How do you manage exchange rate volatility?

**Answer:** Lock price for 15-minute window; re-quote if unconfirmed.

- On quote request: fetch current BTC/ETH price from price feed (Coinbase, Kraken)
- Store in Redis: `quote:{order_id} = {price, timestamp, currency, fiat_amount}`
- TTL: 15 minutes (quote expires after 15 min)
- If customer sends crypto after quote expires:
  - Re-fetch price; compare to locked price
  - If difference >2%: send email "Price changed, confirm new amount?"
  - Otherwise: auto-accept new price
- Store price history in MongoDB for audit trail

### 4. How do you refund crypto transactions?

**Answer:** Fiat refund via standard gateway; crypto refund via wallet transfer.

- Refund initiated: check customer preference (fiat or crypto)
- **Fiat refund**: reverse charge via payment gateway API; process within 24 hours
- **Crypto refund**:
  - Retrieve original sender's wallet address from blockchain transaction
  - Initiate transfer from merchant wallet (via payment processor API)
  - Cover network fees (miner + slippage): add 5% to refund amount
  - Allow 1-24 hours for transaction confirmation
- Store refund request in PostgreSQL with status: `INITIATED` → `CONFIRMED`

### 5. How do you secure private keys and wallets?

**Answer:** HSM or MPC wallets; never expose keys in application memory.

- **Option 1 (Recommended): Hardware Security Module (HSM)**
  - Thales/YubiHSM stores private keys in tamper-resistant hardware
  - Application calls HSM API to sign transactions (never receives key)
  - Network: private, secured with mTLS
  - High availability: clustered HSM with replication

- **Option 2: Multi-Party Computation (MPC) Wallets**
  - Use service like Fireblocks or Coincover
  - Private key split into shares (n-of-m threshold)
  - No single entity holds full key
  - Application sends signing requests to MPC service
  - Requires cryptographic approval from multiple parties

- **Option 3: Cloud Custodian (Lowest control)**
  - BitGo, Anchorage: hold keys, provide signing service
  - Application makes REST calls to sign/transfer
  - Good for managed compliance

- **Best Practice**: Combination:
  - Hot wallet (5% of funds): HSM-protected for fast transfers
  - Cold wallet (95% of funds): offline multisig (Coinbase Custody)
  - No private key material ever in application logs

### 6. How do you comply with regulatory requirements?

**Answer:** AML/KYC screening + transaction monitoring + audit trail.

- **AML Screening (on wallet address)**:
  - Call Chainalysis/Sanction Scanner API on deposit address
  - Check OFAC list: if flagged → `AML_BLOCKED`; notify customer
  - Also screen USDT/stablecoin transfers for patterns
  - Latency: <2 sec; cache results for 24 hours

- **KYC Verification (on transaction >$3K)**:
  - Auto-verify: match transaction sender's IP geolocation, device fingerprint
  - If KYC required: email customer link to automated verification (ID scan + liveness)
  - Store KYC data in MongoDB; encrypted with key rotation
  - Retention: 7 years for regulatory audit

- **Transaction Monitoring**:
  - Flag suspicious patterns: rapid multiple txns, round-number amounts, known pools
  - Generate SAR (Suspicious Activity Report) if needed
  - Audit trail: all decisions logged in PostgreSQL

- **Audit Trail**:
  - Every transaction: timestamp, wallet address, IP, KYC status, AML result
  - Store in MongoDB (immutable log); copy to cold storage (S3)
  - Compliance query: "All txns for address X from date Y-Z"

---

## Microservices Breakdown

| Service | Responsibility | Key Classes | Dependencies |
|---------|-----------------|------------|--------------|
| **Crypto Payment Service** | Order creation, address generation, quote management | `CryptoPaymentController`, `PaymentOrderService`, `CryptoAdapterFactory`, `PriceQuoteService` | PostgreSQL, Redis, Kafka |
| **KYC/AML Service** | Customer verification, wallet screening, compliance | `KycVerificationService`, `AmlScreeningService`, `WalletAddressScreener`, `ComplianceDecisionEngine` | PostgreSQL, MongoDB, Kafka, External APIs |
| **Confirmation Checker** | Poll blockchain, track confirmations, emit events | `BlockchainConfirmationListener`, `Web3jClient`, `ConfirmationPoller`, `WebhookHandler` | PostgreSQL, Redis, Kafka |
| **Crypto to Fiat Converter** | Execute instant conversion, price settlement | `CryptoToFiatConverter`, `PriceAggregator`, `FiatPaymentCapture` | Coinbase API, PostgreSQL, Kafka |
| **Wallet Management** | Secure key storage, address management, transfers | `WalletService`, `HsmClient`, `MpcWalletAdapter`, `TransactionSigner` | HSM/MPC service, MongoDB |
| **Refund Service** | Process refunds in fiat or crypto | `RefundService`, `CryptoTransferInitiator`, `RefundReconciliation` | PostgreSQL, Wallet Service, Kafka |

---

## Database Design

### PostgreSQL Schema

```sql
-- Cryptocurrency Payment Orders
CREATE TABLE crypto_payment_orders (
    id BIGSERIAL PRIMARY KEY,
    order_id UUID NOT NULL UNIQUE,
    customer_id UUID NOT NULL,
    merchant_id UUID NOT NULL,
    currency VARCHAR(10) NOT NULL, -- BTC, ETH
    crypto_amount DECIMAL(20, 8) NOT NULL,
    usd_amount DECIMAL(10, 2) NOT NULL,
    deposit_address VARCHAR(512) NOT NULL,
    transaction_hash VARCHAR(512),
    confirmations_received INT DEFAULT 0,
    confirmations_required INT NOT NULL,
    quote_timestamp TIMESTAMP NOT NULL,
    quote_expiry_timestamp TIMESTAMP NOT NULL,
    confirmation_status VARCHAR(50) DEFAULT 'PENDING_CONFIRMATION',
    fiat_conversion_status VARCHAR(50),
    converted_usd_amount DECIMAL(10, 2),
    conversion_timestamp TIMESTAMP,
    order_status VARCHAR(50) DEFAULT 'PENDING_CRYPTO_CONFIRMATION',
    kyc_required BOOLEAN DEFAULT FALSE,
    kyc_status VARCHAR(50),
    aml_screening_status VARCHAR(50),
    network_fee_estimate DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_order_id (order_id),
    INDEX idx_customer_status (customer_id, order_status),
    INDEX idx_address_tx (deposit_address, transaction_hash),
    INDEX idx_confirmation_status (confirmation_status, confirmations_received)
);

-- Price Quotes (Short-lived)
CREATE TABLE price_quotes (
    id BIGSERIAL PRIMARY KEY,
    order_id UUID NOT NULL REFERENCES crypto_payment_orders(order_id),
    currency VARCHAR(10) NOT NULL,
    btc_price DECIMAL(15, 2),
    eth_price DECIMAL(15, 2),
    crypto_amount_requested DECIMAL(20, 8),
    fiat_amount_usd DECIMAL(10, 2),
    quote_timestamp TIMESTAMP NOT NULL,
    quote_expires_at TIMESTAMP NOT NULL,
    price_source VARCHAR(50) NOT NULL, -- COINBASE, KRAKEN, COINGECKO
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_order_expires (order_id, quote_expires_at)
);

-- KYC Data (PII - Encrypted)
CREATE TABLE kyc_verification (
    id BIGSERIAL PRIMARY KEY,
    customer_id UUID NOT NULL UNIQUE,
    full_name VARCHAR(255) NOT NULL ENCRYPTED,
    date_of_birth DATE NOT NULL ENCRYPTED,
    country VARCHAR(100),
    identity_document_type VARCHAR(50),
    identity_document_hash VARCHAR(512),
    verification_status VARCHAR(50),
    verified_at TIMESTAMP,
    auto_verification_score DECIMAL(3, 2),
    manual_review_required BOOLEAN,
    reviewer_id UUID,
    reviewed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_customer_status (customer_id, verification_status)
);

-- AML Screening Results
CREATE TABLE aml_screening_results (
    id BIGSERIAL PRIMARY KEY,
    order_id UUID NOT NULL REFERENCES crypto_payment_orders(order_id),
    wallet_address VARCHAR(512) NOT NULL,
    screening_provider VARCHAR(50) NOT NULL, -- CHAINALYSIS, SANCTION_SCANNER
    risk_score DECIMAL(3, 2),
    risk_level VARCHAR(50), -- LOW, MEDIUM, HIGH, BLOCKED
    findings TEXT,
    screening_timestamp TIMESTAMP NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_order_status (order_id, risk_level),
    INDEX idx_address_screening (wallet_address)
);

-- Refund Transactions
CREATE TABLE crypto_refunds (
    id BIGSERIAL PRIMARY KEY,
    order_id UUID NOT NULL REFERENCES crypto_payment_orders(order_id),
    refund_type VARCHAR(50) NOT NULL, -- FIAT, CRYPTO
    refund_amount DECIMAL(10, 2) NOT NULL,
    destination_address VARCHAR(512),
    refund_tx_hash VARCHAR(512),
    refund_status VARCHAR(50) DEFAULT 'INITIATED',
    network_fee_paid DECIMAL(10, 2),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    confirmed_at TIMESTAMP,
    INDEX idx_order_refund (order_id, refund_status)
);

-- Transaction Audit Trail
CREATE TABLE crypto_transaction_audit (
    id BIGSERIAL PRIMARY KEY,
    order_id UUID NOT NULL,
    event_type VARCHAR(100) NOT NULL,
    event_data JSONB NOT NULL,
    ip_address VARCHAR(45),
    user_agent VARCHAR(512),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_order_audit (order_id, created_at)
);

-- Wallet Addresses (Mapping to Orders)
CREATE TABLE wallet_addresses (
    id BIGSERIAL PRIMARY KEY,
    order_id UUID NOT NULL REFERENCES crypto_payment_orders(order_id),
    currency VARCHAR(10) NOT NULL,
    address VARCHAR(512) NOT NULL UNIQUE,
    address_label VARCHAR(255),
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_address (address)
);
```

### MongoDB Collections

```javascript
// Extended transaction metadata & history
db.createCollection("crypto_transactions", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["order_id", "timestamp"],
      properties: {
        _id: { bsonType: "objectId" },
        order_id: { bsonType: "string" },
        tx_hash: { bsonType: "string" },
        wallet_address: { bsonType: "string" },
        currency: { bsonType: "string" },
        amount: { bsonType: "decimal" },
        confirmations: { bsonType: "int" },
        block_number: { bsonType: "long" },
        block_timestamp: { bsonType: "date" },
        network_fee: { bsonType: "decimal" },
        conversion_rate: { bsonType: "decimal" },
        ip_address: { bsonType: "string" },
        device_fingerprint: { bsonType: "string" },
        timestamp: { bsonType: "date" }
      }
    }
  }
});

// KYC audit trail
db.createCollection("kyc_audit_trail", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        _id: { bsonType: "objectId" },
        customer_id: { bsonType: "string" },
        verification_round: { bsonType: "int" },
        auto_score: { bsonType: "decimal" },
        auto_decision: { bsonType: "string" },
        documents_submitted: { bsonType: "array" },
        manual_reviewer: { bsonType: "string" },
        final_decision: { bsonType: "string" },
        decision_reason: { bsonType: "string" },
        timestamp: { bsonType: "date" }
      }
    }
  }
});

// Price history for auditing
db.createCollection("price_history", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        _id: { bsonType: "objectId" },
        currency: { bsonType: "string" },
        price_usd: { bsonType: "decimal" },
        source: { bsonType: "string" },
        timestamp: { bsonType: "date" }
      }
    }
  }
});

// Create TTL index for price history (keep 2 years)
db.price_history.createIndex({ timestamp: 1 }, { expireAfterSeconds: 63072000 });
```

---

## Redis Data Structures

```yaml
# Active Price Quotes (15-min validity window)
quote:{order_id}
  Type: HASH
  Fields: {btc_price, eth_price, crypto_amount, fiat_amount, timestamp}
  TTL: 900 seconds (15 minutes)
  Usage: Lock price for customer during checkout

# Confirmation State (Tracking blockchain progress)
order:{order_id}:confirmation
  Type: HASH
  Fields: {tx_hash, confirmations, required_confirmations, first_seen_timestamp}
  TTL: 86400 seconds (24 hours, then move to PostgreSQL)
  Usage: Fast lookup of current confirmation count

# Polling Checkpoint (Prevent duplicate confirmation checks)
order:{order_id}:last_poll
  Type: STRING
  Value: Timestamp (milliseconds) of last poll
  TTL: 3600 seconds (1 hour)
  Usage: Throttle polling to once per 10 seconds

# Wallet Address Allocation (Uniqueness guarantee)
address:{wallet_address}
  Type: STRING
  Value: order_id
  TTL: 3600 seconds (1 hour after address generation)
  Usage: Prevent re-using address if order cancelled

# KYC Verification Status (Fast lookup)
kyc:customer:{customer_id}
  Type: HASH
  Fields: {status, score, decision, timestamp}
  TTL: 3600 seconds (1 hour)
  Usage: Cache KYC decision during transaction window

# AML Screening Cache (24-hour validity)
aml:address:{wallet_address}
  Type: HASH
  Fields: {risk_score, risk_level, provider, timestamp}
  TTL: 86400 seconds (24 hours)
  Usage: Avoid re-screening same address within 24 hours

# Price Feed Cache (5-minute validity)
price:feed:{currency}
  Type: STRING
  Value: Current price in USD
  TTL: 300 seconds (5 minutes)
  Usage: Fast price display on checkout page

# Rate Limit Counters (Per-customer)
ratelimit:payment:{customer_id}:{hour}
  Type: STRING
  Value: Integer (payment count in current hour)
  TTL: 3600 seconds (1 hour)
  Usage: Enforce max 10 transactions/hour per customer

# Transaction Lock (Prevent duplicate processing)
txlock:{tx_hash}
  Type: STRING
  Value: order_id
  TTL: 3600 seconds
  Usage: Ensure single processing of blockchain transaction
```

---

## Kafka Event Flow

```
Topic: payment.quote_requested
  Schema: { order_id, customer_id, currency, amount_usd }
  Producers: PaymentController
  Consumers: PriceQuoteService
  Partitions: 5
  Retention: 7 days

Topic: payment.deposit_address_created
  Schema: { order_id, currency, deposit_address, quote_expires_at }
  Producers: PaymentOrderService
  Consumers: NotificationService, AmlScreeningService
  Partitions: 10 (by order_id)
  Retention: 3650 days (7 years)

Topic: payment.crypto_received
  Schema: { order_id, tx_hash, currency, amount, wallet_address, network, timestamp }
  Producers: BlockchainWebhookHandler
  Consumers: ConfirmationChecker, AuditLogger
  Partitions: 10
  Retention: 3650 days

Topic: payment.crypto_confirmed
  Schema: { order_id, tx_hash, confirmations, currency, fiat_equivalent }
  Producers: ConfirmationChecker
  Consumers: CryptoToFiatConverter, NotificationService
  Partitions: 10
  Retention: 3650 days

Topic: payment.fiat_converted
  Schema: { order_id, crypto_amount, fiat_amount, rate, conversion_timestamp }
  Producers: CryptoToFiatConverter
  Consumers: OrderFulfillmentService, PaymentService
  Partitions: 10
  Retention: 3650 days

Topic: kyc.verification_required
  Schema: { customer_id, order_id, transaction_amount, required_by_timestamp }
  Producers: PaymentOrderService
  Consumers: KycService, NotificationService
  Partitions: 5
  Retention: 3650 days (compliance)

Topic: kyc.verification_completed
  Schema: { customer_id, verification_status, auto_score, manual_reviewer }
  Producers: KycService
  Consumers: PaymentOrderService, ComplianceService
  Partitions: 5
  Retention: 3650 days

Topic: aml.screening_completed
  Schema: { order_id, wallet_address, risk_level, screening_provider }
  Producers: AmlScreeningService
  Consumers: PaymentOrderService, ComplianceService
  Partitions: 10
  Retention: 3650 days

Topic: payment.refund_initiated
  Schema: { order_id, refund_type, amount, destination }
  Producers: RefundService
  Consumers: FiatGateway, CryptoTransferService, AuditLogger
  Partitions: 5
  Retention: 3650 days

Topic: payment.refund_confirmed
  Schema: { refund_id, order_id, refund_tx_hash, confirmed_at }
  Producers: RefundService
  Consumers: NotificationService, OrderService
  Partitions: 5
  Retention: 3650 days
```

---

## Implementation Code

### 1. CryptoPaymentAdapter (Strategy Pattern)

```java
package com.crypto.payment.adapter;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.math.BigDecimal;
import java.util.UUID;

/**
 * Adapter interface for crypto payment processors.
 * Allows switching between CoinGate, BitPay, Coinbase Commerce.
 */
public interface CryptoPaymentGateway {
    /**
     * Generate unique deposit address for order.
     */
    String generateDepositAddress(UUID orderId, String currency) throws PaymentGatewayException;

    /**
     * Get current price quote for currency.
     */
    PriceQuote getPriceQuote(String currency) throws PaymentGatewayException;

    /**
     * Check blockchain confirmation status.
     */
    ConfirmationStatus checkConfirmation(String txHash, String currency) throws PaymentGatewayException;

    /**
     * Register webhook for transaction notifications.
     */
    void registerWebhook(String orderId, String webhookUrl) throws PaymentGatewayException;
}

@Slf4j
@Service
@RequiredArgsConstructor
public class CoinGateAdapter implements CryptoPaymentGateway {

    private final CoinGateApiClient apiClient;
    private final RestTemplate restTemplate;
    private final String coinGateApiKey;

    @Override
    public String generateDepositAddress(UUID orderId, String currency) throws PaymentGatewayException {
        try {
            CoinGateCreateOrderRequest request = CoinGateCreateOrderRequest.builder()
                .priceAmount(1000) // Example: $1000
                .priceCurrency("USD")
                .receiveCurrency(currency.toLowerCase())
                .orderId(orderId.toString())
                .title("Order " + orderId)
                .description("Payment for order " + orderId)
                .callbackUrl("https://merchant.com/crypto/callback")
                .successUrl("https://merchant.com/success")
                .cancelUrl("https://merchant.com/cancel")
                .build();

            CoinGateCreateOrderResponse response = apiClient.createOrder(request, coinGateApiKey);
            return response.getPaymentAddress();

        } catch (HttpClientErrorException e) {
            log.error("CoinGate API error for order {}: {}", orderId, e.getMessage());
            throw new PaymentGatewayException("Failed to generate address", e);
        }
    }

    @Override
    public PriceQuote getPriceQuote(String currency) throws PaymentGatewayException {
        try {
            CoinGatePriceResponse response = apiClient.getPrice(currency, "USD", coinGateApiKey);
            return PriceQuote.builder()
                .currency(currency)
                .priceUsd(response.getPriceUsd())
                .source("COINGATE")
                .timestamp(java.time.Instant.now())
                .build();

        } catch (Exception e) {
            throw new PaymentGatewayException("Failed to fetch price", e);
        }
    }

    @Override
    public ConfirmationStatus checkConfirmation(String txHash, String currency) throws PaymentGatewayException {
        // Call blockchain API (Infura for ETH, Blockchain.com for BTC)
        // Return confirmations count
        return null; // Simplified
    }

    @Override
    public void registerWebhook(String orderId, String webhookUrl) throws PaymentGatewayException {
        // Register webhook with CoinGate for transaction notifications
    }
}

@Slf4j
@Service
public class CryptoPaymentAdapterFactory {

    private final CoinGateAdapter coinGateAdapter;
    private final BitPayAdapter bitPayAdapter;

    public CryptoPaymentGateway getAdapter(String adapterType) {
        return switch (adapterType) {
            case "COINGRATE" -> coinGateAdapter;
            case "BITPAY" -> bitPayAdapter;
            default -> throw new IllegalArgumentException("Unknown adapter: " + adapterType);
        };
    }

    /**
     * Get primary adapter with fallback.
     */
    public CryptoPaymentGateway getPrimaryAdapterWithFallback() {
        try {
            // Check if primary adapter is healthy
            return coinGateAdapter;
        } catch (Exception e) {
            log.warn("Primary adapter (CoinGate) unavailable; falling back to BitPay");
            return bitPayAdapter;
        }
    }
}

@lombok.Data
@lombok.Builder
class PriceQuote {
    private String currency;
    private BigDecimal priceUsd;
    private String source;
    private java.time.Instant timestamp;
}

@lombok.Data
@lombok.Builder
class ConfirmationStatus {
    private String txHash;
    private int confirmations;
    private int requiredConfirmations;
    private String status; // PENDING, CONFIRMED
}
```

### 2. BlockchainConfirmationListener (Polling-based)

```java
package com.crypto.confirmation;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.crypto.domain.CryptoPaymentOrder;
import com.crypto.repository.CryptoPaymentOrderRepository;
import com.crypto.event.PaymentCryptoConfirmedEvent;
import web3j.protocol.Web3j;
import web3j.protocol.core.methods.response.EthBlock;
import java.io.IOException;
import java.math.BigInteger;
import java.util.List;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class BlockchainConfirmationListener {

    private final CryptoPaymentOrderRepository orderRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final Web3j web3j; // Infura or Alchemy client for Ethereum
    private final BitcoinRpcClient bitcoinClient; // For Bitcoin RPC

    private static final int ETH_CONFIRMATIONS_REQUIRED = 12;
    private static final int BTC_CONFIRMATIONS_REQUIRED = 3;
    private static final long POLL_INTERVAL_SECONDS = 10;

    /**
     * Poll blockchain for confirmation updates every 10 seconds.
     * Check orders in PENDING_CRYPTO_CONFIRMATION status.
     */
    @Scheduled(fixedDelay = POLL_INTERVAL_SECONDS, timeUnit = TimeUnit.SECONDS)
    @Transactional
    public void pollBlockchainConfirmations() {
        try {
            // Find pending orders
            List<CryptoPaymentOrder> pendingOrders = orderRepository
                .findByConfirmationStatusOrderByCreatedAtAsc("PENDING_CONFIRMATION", 100);

            for (CryptoPaymentOrder order : pendingOrders) {
                // Rate limit: check at most once per 10 seconds per order
                String lastPollKey = "order:" + order.getOrderId() + ":last_poll";
                String lastPollStr = redisTemplate.opsForValue().get(lastPollKey);

                if (lastPollStr != null) {
                    long lastPoll = Long.parseLong(lastPollStr);
                    if (System.currentTimeMillis() - lastPoll < POLL_INTERVAL_SECONDS * 1000) {
                        continue; // Skip; too soon to poll again
                    }
                }

                try {
                    checkAndUpdateConfirmation(order);
                    redisTemplate.opsForValue().set(lastPollKey,
                        String.valueOf(System.currentTimeMillis()),
                        Duration.ofMinutes(30)); // Expire old entries

                } catch (Exception e) {
                    log.error("Failed to check confirmation for order {}", order.getOrderId(), e);
                }
            }

        } catch (Exception e) {
            log.error("Blockchain polling cycle failed", e);
        }
    }

    private void checkAndUpdateConfirmation(CryptoPaymentOrder order) throws IOException {
        if ("ETH".equalsIgnoreCase(order.getCurrency())) {
            checkEthConfirmation(order);
        } else if ("BTC".equalsIgnoreCase(order.getCurrency())) {
            checkBtcConfirmation(order);
        }
    }

    private void checkEthConfirmation(CryptoPaymentOrder order) throws IOException {
        // Get transaction receipt
        var txReceipt = web3j.ethGetTransactionReceipt(order.getTransactionHash())
            .send();

        if (txReceipt.getResult() == null) {
            // Transaction not yet mined
            log.debug("ETH transaction {} not yet mined for order {}",
                order.getTransactionHash(), order.getOrderId());
            return;
        }

        // Calculate confirmations
        EthBlock.Block currentBlock = web3j.ethGetBlockByNumber(
            org.web3j.protocol.core.DefaultBlockParameterName.LATEST, false)
            .send()
            .getBlock();

        BigInteger txBlockNumber = txReceipt.getResult().getBlockNumber();
        BigInteger confirmations = currentBlock.getNumber().subtract(txBlockNumber);

        log.debug("ETH transaction {} has {} confirmations",
            order.getTransactionHash(), confirmations.longValue());

        // Update confirmation count
        order.setConfirmationsReceived(confirmations.intValue());

        // Check if reached required confirmations
        if (confirmations.intValue() >= ETH_CONFIRMATIONS_REQUIRED) {
            onConfirmationComplete(order);
        } else {
            orderRepository.save(order);
        }
    }

    private void checkBtcConfirmation(CryptoPaymentOrder order) throws IOException {
        try {
            // Call Bitcoin RPC to get transaction
            var txInfo = bitcoinClient.getTransaction(order.getTransactionHash());
            int confirmations = txInfo.getConfirmations();

            log.debug("BTC transaction {} has {} confirmations",
                order.getTransactionHash(), confirmations);

            order.setConfirmationsReceived(confirmations);

            if (confirmations >= BTC_CONFIRMATIONS_REQUIRED) {
                onConfirmationComplete(order);
            } else {
                orderRepository.save(order);
            }

        } catch (Exception e) {
            log.warn("Failed to fetch BTC transaction {}", order.getTransactionHash(), e);
        }
    }

    /**
     * Transaction reached required confirmations; emit event for downstream processing.
     */
    private void onConfirmationComplete(CryptoPaymentOrder order) {
        order.setConfirmationStatus("CONFIRMED");
        order.setUpdatedAt(java.time.LocalDateTime.now());
        orderRepository.save(order);

        kafkaTemplate.send("payment.crypto_confirmed", order.getOrderId().toString(),
            new PaymentCryptoConfirmedEvent(
                order.getOrderId(),
                order.getTransactionHash(),
                order.getConfirmationsReceived(),
                order.getCurrency(),
                order.getUsdAmount()));

        log.info("Order {} confirmed on blockchain with {} confirmations",
            order.getOrderId(), order.getConfirmationsReceived());
    }
}
```

### 3. CryptoPriceQuoteService

```java
package com.crypto.pricing;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.crypto.domain.PriceQuote;
import com.crypto.repository.PriceQuoteRepository;
import com.crypto.event.PaymentQuoteRequestedEvent;
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.time.LocalDateTime;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class CryptoPriceQuoteService {

    private final PriceQuoteRepository priceQuoteRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final ExternalPriceProvider priceProvider; // Coinbase, Kraken, etc.

    private static final long QUOTE_VALIDITY_SECONDS = 900; // 15 minutes
    private static final long PRICE_CACHE_SECONDS = 60; // 1 minute

    /**
     * Generate price quote for customer. Lock price for 15 minutes.
     */
    @Transactional
    public PriceQuote generateQuote(UUID orderId, String currency, BigDecimal usdAmount) {
        // Fetch current price (from cache if <1 min old)
        BigDecimal currentPrice = getCurrentPrice(currency);

        // Calculate crypto amount
        BigDecimal cryptoAmount = usdAmount.divide(currentPrice, 8, RoundingMode.HALF_UP);

        // Create quote record
        PriceQuote quote = PriceQuote.builder()
            .orderId(orderId)
            .currency(currency)
            .priceUsd(currentPrice)
            .cryptoAmountRequested(cryptoAmount)
            .fiatAmountUsd(usdAmount)
            .quoteTimestamp(LocalDateTime.now())
            .quoteExpiresAt(LocalDateTime.now().plusSeconds(QUOTE_VALIDITY_SECONDS))
            .priceSource("COINBASE")
            .build();

        priceQuoteRepository.save(quote);

        // Store in Redis for fast lookup (during checkout)
        String quoteKey = "quote:" + orderId;
        redisTemplate.opsForHash().putAll(quoteKey, Map.of(
            "btc_price", currentPrice.toString(),
            "currency", currency,
            "crypto_amount", cryptoAmount.toString(),
            "fiat_amount", usdAmount.toString(),
            "timestamp", String.valueOf(System.currentTimeMillis())
        ));
        redisTemplate.expire(quoteKey, java.time.Duration.ofSeconds(QUOTE_VALIDITY_SECONDS));

        // Emit event for analytics
        kafkaTemplate.send("payment.quote_requested", orderId.toString(),
            new PaymentQuoteRequestedEvent(orderId, currency, currentPrice));

        log.info("Generated price quote for order {}: {} {} = ${}",
            orderId, cryptoAmount, currency, usdAmount);

        return quote;
    }

    /**
     * Get current price with caching.
     */
    private BigDecimal getCurrentPrice(String currency) {
        String cacheKey = "price:feed:" + currency;

        // Try cache first
        String cachedPrice = redisTemplate.opsForValue().get(cacheKey);
        if (cachedPrice != null) {
            return new BigDecimal(cachedPrice);
        }

        // Fetch from external provider
        BigDecimal price = priceProvider.getPrice(currency, "USD");

        // Cache for 60 seconds
        redisTemplate.opsForValue().set(cacheKey, price.toString(),
            java.time.Duration.ofSeconds(PRICE_CACHE_SECONDS));

        // Store in MongoDB for history
        storePriceHistory(currency, price);

        return price;
    }

    /**
     * Check if quote has expired and generate new one if needed.
     */
    public boolean validateQuote(UUID orderId, BigDecimal currentPrice, BigDecimal quotedPrice) {
        // If price difference >2%, return false (quote stale)
        BigDecimal priceDiff = currentPrice.subtract(quotedPrice).abs();
        BigDecimal percentDiff = priceDiff.divide(quotedPrice, 2, RoundingMode.HALF_UP)
            .multiply(new BigDecimal(100));

        return percentDiff.compareTo(new BigDecimal("2")) <= 0;
    }

    private void storePriceHistory(String currency, BigDecimal price) {
        // Store in MongoDB for audit trail
        // Implementation: insert into price_history collection
    }
}
```

### 4. WalletAddressScreeningService (AML/KYC)

```java
package com.crypto.compliance;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.crypto.domain.AmlScreeningResult;
import com.crypto.repository.AmlScreeningResultRepository;
import com.crypto.event.AmlScreeningCompletedEvent;
import java.time.LocalDateTime;
import java.time.Duration;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class WalletAddressScreeningService {

    private final AmlScreeningResultRepository screeningResultRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final ChainanalysisApiClient chainanalysisClient;
    private final SanctionScannerApiClient sanctionScannerClient;
    private final OfacListService ofacListService;

    private static final long SCREENING_CACHE_SECONDS = 86400; // 24 hours
    private static final String[] RISK_BLOCKED_INDICATORS = {"SANCTIONED", "RANSOMWARE", "STOLEN"};

    /**
     * Screen wallet address against AML/KYC databases.
     * Returns risk level: LOW, MEDIUM, HIGH, BLOCKED.
     */
    @Transactional
    public AmlScreeningResult screenWalletAddress(UUID orderId, String walletAddress, String currency) {
        // Check cache first (24-hour validity)
        AmlScreeningResult cachedResult = getCachedScreeningResult(walletAddress);
        if (cachedResult != null) {
            log.info("Using cached AML screening for address {} (age: <24h)", walletAddress);
            return cachedResult;
        }

        try {
            // Primary screening: Chainalysis
            AmlScreeningResult chainanalysisResult = screenWithChainanalysis(walletAddress, currency);

            // Store result
            chainanalysisResult.setOrderId(orderId);
            chainanalysisResult.setScreeningTimestamp(LocalDateTime.now());
            screeningResultRepository.save(chainanalysisResult);

            // Cache result
            cacheScreeningResult(walletAddress, chainanalysisResult);

            // Emit event
            kafkaTemplate.send("aml.screening_completed", orderId.toString(),
                new AmlScreeningCompletedEvent(orderId, walletAddress,
                    chainanalysisResult.getRiskLevel(), "CHAINALYSIS"));

            log.info("AML screening completed for order {}: risk_level={}",
                orderId, chainanalysisResult.getRiskLevel());

            return chainanalysisResult;

        } catch (Exception e) {
            log.error("AML screening failed for order {}, address {}", orderId, walletAddress, e);

            // Fallback: manual review required
            AmlScreeningResult failureResult = AmlScreeningResult.builder()
                .orderId(orderId)
                .walletAddress(walletAddress)
                .riskLevel("MANUAL_REVIEW")
                .findings("Screening service unavailable; manual review required")
                .screeningProvider("UNKNOWN")
                .screeningTimestamp(LocalDateTime.now())
                .build();

            screeningResultRepository.save(failureResult);
            return failureResult;
        }
    }

    private AmlScreeningResult screenWithChainanalysis(String walletAddress, String currency) {
        try {
            var response = chainanalysisClient.screenAddress(walletAddress);

            // Parse response
            String riskLevel = response.getRiskLevel(); // LOW, MEDIUM, HIGH
            double riskScore = response.getRiskScore(); // 0.0 to 1.0

            // Check if sanctioned
            if (isAddressSanctioned(walletAddress)) {
                riskLevel = "BLOCKED";
            }

            AmlScreeningResult result = AmlScreeningResult.builder()
                .walletAddress(walletAddress)
                .screeningProvider("CHAINALYSIS")
                .riskScore(java.math.BigDecimal.valueOf(riskScore))
                .riskLevel(riskLevel)
                .findings(response.getFindings())
                .build();

            return result;

        } catch (Exception e) {
            throw new AmlScreeningException("Chainalysis screening failed", e);
        }
    }

    private boolean isAddressSanctioned(String walletAddress) {
        // Check against OFAC list
        return ofacListService.isAddressOnList(walletAddress);
    }

    private AmlScreeningResult getCachedScreeningResult(String walletAddress) {
        String cacheKey = "aml:address:" + walletAddress;
        String cached = redisTemplate.opsForValue().get(cacheKey);

        if (cached != null) {
            // Deserialize from JSON
            return deserializeResult(cached);
        }
        return null;
    }

    private void cacheScreeningResult(String walletAddress, AmlScreeningResult result) {
        String cacheKey = "aml:address:" + walletAddress;
        redisTemplate.opsForValue().set(cacheKey, serializeResult(result),
            Duration.ofSeconds(SCREENING_CACHE_SECONDS));
    }

    private String serializeResult(AmlScreeningResult result) {
        // JSON serialization
        return "";
    }

    private AmlScreeningResult deserializeResult(String json) {
        // JSON deserialization
        return null;
    }
}

/**
 * KYC Verification Service for high-value transactions.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class KycVerificationService {

    private final KycVerificationRepository kycRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    private static final BigDecimal KYC_THRESHOLD = new BigDecimal("3000");

    /**
     * Trigger KYC if transaction >$3,000.
     */
    public void requireKycIfHighValue(UUID orderId, UUID customerId, BigDecimal amount) {
        if (amount.compareTo(KYC_THRESHOLD) > 0) {
            // Check if customer already KYC'd
            var existing = kycRepository.findByCustomerId(customerId);

            if (existing == null || !existing.getVerificationStatus().equals("VERIFIED")) {
                log.info("KYC required for customer {} (amount: ${})", customerId, amount);

                // Emit event to trigger KYC workflow
                kafkaTemplate.send("kyc.verification_required", customerId.toString(),
                    new KycVerificationRequiredEvent(customerId, orderId, amount));
            }
        }
    }
}
```

### 5. CryptoToFiatConverter

```java
package com.crypto.conversion;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.crypto.domain.CryptoPaymentOrder;
import com.crypto.repository.CryptoPaymentOrderRepository;
import com.crypto.event.PaymentCryptoConfirmedEvent;
import com.crypto.event.PaymentFiatConvertedEvent;
import com.payment.gateway.PaymentGatewayAdapter;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Slf4j
@Service
@RequiredArgsConstructor
public class CryptoToFiatConverter {

    private final CryptoPaymentOrderRepository orderRepository;
    private final PaymentGatewayAdapter paymentGateway;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final PriceAggregator priceAggregator;

    /**
     * Listen for confirmed crypto transactions; convert to fiat immediately.
     */
    @KafkaListener(topics = "payment.crypto_confirmed", groupId = "fiat-converter")
    @Transactional
    public void onCryptoConfirmed(PaymentCryptoConfirmedEvent event) {
        CryptoPaymentOrder order = orderRepository.findByOrderId(event.getOrderId())
            .orElseThrow(() -> new OrderNotFoundException(event.getOrderId()));

        try {
            // Get current conversion rate
            BigDecimal conversionRate = priceAggregator.getCurrentPrice(order.getCurrency(), "USD");

            // Calculate fiat equivalent
            BigDecimal fiatAmount = order.getCryptoAmount()
                .multiply(conversionRate)
                .setScale(2, java.math.RoundingMode.HALF_UP);

            // Convert via payment processor (Coinbase Instant Convert)
            var conversionResult = paymentGateway.convertToFiat(
                order.getTransactionHash(),
                order.getCurrency(),
                "USD",
                order.getCryptoAmount()
            );

            if (conversionResult.isSuccess()) {
                order.setFiatConversionStatus("COMPLETED");
                order.setConvertedUsdAmount(conversionResult.getFiatAmount());
                order.setConversionTimestamp(LocalDateTime.now());
                orderRepository.save(order);

                // Emit conversion event
                kafkaTemplate.send("payment.fiat_converted", order.getOrderId().toString(),
                    new PaymentFiatConvertedEvent(
                        order.getOrderId(),
                        order.getCryptoAmount(),
                        conversionResult.getFiatAmount(),
                        conversionRate,
                        LocalDateTime.now()));

                log.info("Crypto converted for order {}: {} {} = ${}",
                    order.getOrderId(), order.getCryptoAmount(), order.getCurrency(),
                    conversionResult.getFiatAmount());

            } else {
                log.error("Fiat conversion failed for order {}: {}",
                    order.getOrderId(), conversionResult.getErrorMessage());
                order.setFiatConversionStatus("FAILED");
                orderRepository.save(order);
            }

        } catch (Exception e) {
            log.error("Crypto to fiat conversion failed for order {}", order.getOrderId(), e);
            order.setFiatConversionStatus("FAILED");
            orderRepository.save(order);
        }
    }
}
```

### 6. CryptoRefundService

```java
package com.crypto.refund;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.crypto.domain.CryptoPaymentOrder;
import com.crypto.domain.CryptoRefund;
import com.crypto.repository.CryptoPaymentOrderRepository;
import com.crypto.repository.CryptoRefundRepository;
import com.crypto.event.PaymentRefundInitiatedEvent;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class CryptoRefundService {

    private final CryptoPaymentOrderRepository orderRepository;
    private final CryptoRefundRepository refundRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final WalletService walletService;
    private final PaymentGatewayAdapter paymentGateway;

    /**
     * Process refund: fiat via payment gateway, crypto via blockchain transfer.
     */
    @Transactional
    public void processRefund(UUID orderId, String refundType) {
        CryptoPaymentOrder order = orderRepository.findByOrderId(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        if ("FIAT".equalsIgnoreCase(refundType)) {
            processFiatRefund(order);
        } else if ("CRYPTO".equalsIgnoreCase(refundType)) {
            processCryptoRefund(order);
        }
    }

    private void processFiatRefund(CryptoPaymentOrder order) {
        try {
            // Refund via payment gateway
            var refundResult = paymentGateway.refundTransaction(
                order.getTransactionHash(), order.getConvertedUsdAmount());

            if (refundResult.isSuccess()) {
                CryptoRefund refund = CryptoRefund.builder()
                    .orderId(order.getOrderId())
                    .refundType("FIAT")
                    .refundAmount(order.getConvertedUsdAmount())
                    .refundStatus("INITIATED")
                    .build();
                refundRepository.save(refund);

                kafkaTemplate.send("payment.refund_initiated", order.getOrderId().toString(),
                    new PaymentRefundInitiatedEvent(order.getOrderId(), "FIAT",
                        order.getConvertedUsdAmount(), null));

                log.info("Fiat refund initiated for order {}: ${}",
                    order.getOrderId(), order.getConvertedUsdAmount());

            } else {
                log.error("Fiat refund failed for order {}", order.getOrderId());
            }

        } catch (Exception e) {
            log.error("Fiat refund error for order {}", order.getOrderId(), e);
        }
    }

    private void processCryptoRefund(CryptoPaymentOrder order) {
        try {
            // Get original sender's address from blockchain
            String senderAddress = retrieveSenderAddressFromBlockchain(
                order.getTransactionHash(), order.getCurrency());

            // Calculate refund: crypto amount + network fees (add 5% for slippage)
            BigDecimal refundAmount = order.getCryptoAmount()
                .multiply(new BigDecimal("1.05"));

            // Initiate transfer from merchant wallet
            String refundTxHash = walletService.initiateTransfer(
                order.getCurrency(),
                senderAddress,
                refundAmount);

            CryptoRefund refund = CryptoRefund.builder()
                .orderId(order.getOrderId())
                .refundType("CRYPTO")
                .refundAmount(order.getCryptoAmount())
                .destinationAddress(senderAddress)
                .refundTxHash(refundTxHash)
                .refundStatus("INITIATED")
                .networkFeePaid(refundAmount.subtract(order.getCryptoAmount()))
                .build();
            refundRepository.save(refund);

            kafkaTemplate.send("payment.refund_initiated", order.getOrderId().toString(),
                new PaymentRefundInitiatedEvent(order.getOrderId(), "CRYPTO",
                    order.getCryptoAmount(), refundTxHash));

            log.info("Crypto refund initiated for order {}: {} {} tx={}",
                order.getOrderId(), refundAmount, order.getCurrency(), refundTxHash);

        } catch (Exception e) {
            log.error("Crypto refund error for order {}", order.getOrderId(), e);
        }
    }

    private String retrieveSenderAddressFromBlockchain(String txHash, String currency) {
        // Query blockchain RPC for transaction sender
        // Implementation depends on currency (ETH vs BTC)
        return "";
    }
}
```

---

## Failure Scenarios

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| Confirmation checker stuck (Infura down) | Orders stuck in PENDING_CONFIRMATION | Fallback to multiple RPC providers; queue to DLQ for manual review |
| Price feed unavailable | Cannot generate quotes | Cache price for 1 hour; disable checkout if cache unavailable |
| AML screening API down | Cannot block sanctioned addresses | Require manual review; block address temporarily (retry in 1 hour) |
| KYC verification timeout | Customer cannot complete payment | Cache auto-verification decision; allow retry after 24 hours |
| Wallet transfer fails (insufficient funds) | Cannot refund customer | Queue refund to retry queue; alert operations team |
| Double-confirmation (duplicate polling) | Same transaction processed twice | Use Redis lock per tx_hash; store confirmation idempotency key |
| Price quote expired but unconfirmed | Customer payment at stale rate | Re-quote if unconfirmed after 15 min; require explicit acceptance |
| HSM unavailable (private key storage) | Cannot sign refund transactions | Cold wallet multisig backup; manual signing process |
| Blockchain reorganization (reorg) | Confirmations roll back | Monitor chain depth; revert orders if reorg >2 blocks |

---

## Scaling Strategy

### Horizontal Scaling

**Payment Service:**
- Stateless; multiple instances behind load balancer
- Scale based on quote requests and order creation (target: 100 orders/min per instance)

**Confirmation Checker:**
- Kafka consumer group with auto-scaling
- Scale based on lag (target: <5 min lag on pending orders)
- Batch polling (100 orders per cycle) for efficiency

**AML/KYC Service:**
- Async processing; cache results 24 hours
- Screening API calls: max 10/sec per provider; queue excess requests

**Crypto to Fiat Converter:**
- Kafka consumer; process confirmed transactions
- Parallel processing: 50 conversions simultaneously

### Vertical Scaling

**PostgreSQL:**
- Partitioning by order_id (MOD sharding)
- Indexes: (confirmation_status, updated_at), (customer_id, created_at)
- Read replicas for analytics queries

**MongoDB:**
- Sharding key: order_id
- Indexes: {order_id: 1}, {wallet_address: 1}, {timestamp: 1}

**Redis:**
- Master-slave replication for high availability
- Cluster mode if >10 GB data
- TTL policies: auto-expire quotes (15 min), screening cache (24 hours)

### Database Optimization

```sql
-- Indexes for common queries
CREATE INDEX idx_orders_status ON crypto_payment_orders(confirmation_status, updated_at);
CREATE INDEX idx_orders_customer ON crypto_payment_orders(customer_id, created_at DESC);
CREATE INDEX idx_address ON wallet_addresses(address) UNIQUE;
CREATE INDEX idx_kyc_customer ON kyc_verification(customer_id);
CREATE INDEX idx_aml_address ON aml_screening_results(wallet_address);
CREATE INDEX idx_refunds_status ON crypto_refunds(order_id, refund_status);

-- Partition for time-series data
ALTER TABLE crypto_transaction_audit PARTITION BY RANGE (YEAR(created_at)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);
```

---

## Monitoring & Alerts

### Key Metrics

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| Quote generation latency (p95) | <500 ms | >2000 ms |
| Confirmation check latency (p95) | <2 sec | >5 sec |
| AML screening latency (p95) | <2 sec | >10 sec |
| KYC verification completion | <30 min (auto), <24 h (manual) | >48 h |
| Crypto-to-fiat conversion success rate | >99% | <98% |
| Fiat refund success rate | >99.5% | <99% |
| Crypto refund initiation latency | <1 min | >5 min |
| Confirmation polling lag | <5 min | >30 min |
| Price feed availability | 99.9% | <99% |
| AML blacklist match rate | <0.1% | >1% |

### Alert Rules (Prometheus)

```yaml
groups:
  - name: crypto_payments
    rules:
      - alert: HighConfirmationCheckLatency
        expr: histogram_quantile(0.95, confirmation_check_latency_seconds) > 5
        for: 10m
        annotations:
          summary: "Confirmation check latency high (p95: {{ $value }}s)"

      - alert: AmlScreeningServiceDown
        expr: rate(aml_screening_errors_total[5m]) > 0.1
        for: 5m
        annotations:
          summary: "AML screening service experiencing errors"

      - alert: FiatConversionFailures
        expr: rate(fiat_conversion_failures_total[5m]) > 0.01
        for: 10m
        annotations:
          summary: "Fiat conversion failure rate elevated ({{ $value }})"

      - alert: ConfirmationPollingBacklog
        expr: pending_confirmation_orders_count > 5000
        for: 15m
        annotations:
          summary: "Large backlog of unconfirmed orders ({{ $value }})"
```

---

## Summary Cheat Sheet

### Architecture Components
- **Payment Service**: Quote generation, deposit address creation, order state machine
- **Confirmation Checker**: Poll blockchain (BTC/ETH), track confirmations, emit events
- **AML/KYC Service**: Screen wallet addresses, verify customers for high-value txns
- **Crypto-to-Fiat Converter**: Convert confirmed crypto to USD via CoinGate/BitPay
- **Wallet Manager**: Secure key storage (HSM/MPC), sign refund transactions
- **Refund Service**: Fiat refunds via gateway, crypto refunds via blockchain transfer

### Key Technologies
- **Payment Processors**: CoinGate, BitPay, Coinbase Commerce (adapter pattern)
- **Blockchain Nodes**: Infura/Alchemy (Ethereum), Blockchain.com (Bitcoin)
- **Wallet Security**: HSM (Thales/YubiHSM) or MPC (Fireblocks, Coincover)
- **AML Screening**: Chainalysis, Sanction Scanner, OFAC list
- **PostgreSQL**: Orders, quotes, KYC, refunds, audit trail
- **MongoDB**: Transaction history, wallet metadata, events
- **Redis**: Price quotes (15 min TTL), confirmation state, AML cache (24 h TTL)
- **Kafka**: 9 topics for event streaming

### Confirmation Flow
```
BTC: 3 confirmations (≈30 min)
ETH: 12 confirmations (≈3 min)
Check every 10 sec via polling or webhook
Emit event when confirmed
Trigger fiat conversion immediately
```

### Price Quote Validity
- Lock price for 15 minutes
- If unconfirmed after 15 min: re-quote
- Require customer acceptance if >2% price change

### KYC/AML Flow
```
Tx >$3K → Require KYC
Screen address → OFAC/Chainalysis (24-hour cache)
Risk: LOW, MEDIUM, HIGH, BLOCKED
Manual review if BLOCKED or screening unavailable
```

### Refund Options
- **Fiat**: 24-hour processing via standard payment gateway
- **Crypto**: 1-hour initiation, 10-60 min blockchain confirmation
- Network fees covered by merchant (add 5% to crypto refund)

### Data Retention
- Audit trail: 7 years (compliance)
- KYC data: 7 years (regulatory requirement)
- Price history: 2 years (MongoDB TTL)
- Orders: 1 year hot, then archive

### Performance Targets
- 5K crypto transactions/day, 100 peak/min
- Quote latency p95: <500 ms
- Confirmation check p95: <2 sec
- AML screening p95: <2 sec
- Crypto-to-fiat success: >99%

