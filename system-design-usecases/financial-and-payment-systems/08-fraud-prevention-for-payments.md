---
title: Fraud Prevention for Payments — System Design Deep Dive
layout: default
---

# Fraud Prevention for Payments — Deep Dive Design

> **Scenario:** Detect card testing (stolen cards with small amounts), velocity abuse (same card 100x in 1 hour), geographic anomalies (US card used in Russia 30 minutes later), device fingerprinting, 3D Secure for high-risk transactions, block known fraudulent BINs, make real-time decisions <100ms.
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
10. [Failure Scenarios & Recovery](#failure-scenarios--recovery)
11. [Scaling Strategy](#scaling-strategy)
12. [Monitoring & Alerting](#monitoring--alerting)
13. [Summary Cheat Sheet](#summary-cheat-sheet)

---

## Requirements & Constraints

### Functional Requirements
- Detect card testing: flag transactions with 3+ small (<$5) attempts in 1 hour
- Detect velocity abuse: flag card if 10+ transactions within 60 seconds
- Detect geographic anomalies: flag if card used in 2+ countries within 15 minutes
- Device fingerprinting: link device to user; flag if new device after multiple charges
- 3D Secure challenge: require authentication for risk_score > 50
- Block known fraudulent BINs (first 6 digits of card number)
- Adaptive rules: allow-list for trusted customers, block-list for known fraud
- False positive handling: manual review queue with override option
- Real-time decisions: <100ms latency (p99)
- 10M+ transactions/day at peak

### Non-Functional Requirements
- Sub-100ms decision latency (decision service SLA)
- 99.99% availability (fraud detection cannot be single point of failure)
- Graceful degradation: if fraud service down, fall back to simple threshold rules
- Audit trail: all decisions logged immutably
- Chargeback feedback loop: integrate with payment gateway chargebacks
- Hot reload: rules updated without service restart

---

## Capacity Estimation

| Metric | Calculation | Value |
|--------|-------------|-------|
| **Transactions/Day** | Given | 10,000,000 |
| **Transactions/Second** | 10M ÷ 86,400 | ~116 txns/sec |
| **Peak (4x average)** | 116 × 4 | ~465 txns/sec |
| **Fraud Events/Day** | 0.5% of txns | ~50,000 fraud events/day |
| **Velocity Check Queries/Sec** | 465 peak × 2 queries per txn | ~930 queries/sec |
| **Redis Memory (1h sliding window)** | 465 × 3600 × 200 bytes | ~335 MB |
| **Chargeback Feedback/Day** | 0.1% of txns | ~10,000 chargebacks/day |
| **Device Fingerprints (unique)** | 5% of transactions | ~500K unique devices |
| **BIN Blocklist Size** | 50K–100K fraudulent BINs | ~100 KB (bloom filter) |
| **MongoDB Fraud Decision Log** | 10M txns/day × 200 bytes | ~2 GB/month |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│              Payment Processing Gateway                      │
│         (Stripe, Square, or custom PSP)                      │
└─────────────────┬───────────────────────────────────────────┘
                  │
        ┌─────────┴──────────┐
        │                    │
    ┌───▼─────────┐      ┌──▼──────────────┐
    │ Checkout    │      │ API Payment     │
    │ Form        │      │ Endpoint        │
    └───┬─────────┘      └──┬──────────────┘
        │                   │
        └───────────┬───────┘
                    │
        ┌───────────▼─────────────────┐
        │  Fraud Detection Service    │
        │  (Decision Engine)          │
        │  <100ms SLA                 │
        └───┬──────────────┬──────────┘
            │              │
    ┌───────┴─┐    ┌──────▼──────┐
    │ Risk    │    │ Device      │
    │ Scorer  │    │ Fingerprint │
    │         │    │ Service     │
    └────┬────┘    └──────┬──────┘
         │                │
    ┌────┴────────────────┴──────────────┐
    │      Redis Cluster                 │
    │ (Velocity checks, BIN blocklist)   │
    └────────────┬─────────────────────┘
                 │
    ┌────────────┴────────┐
    │                     │
┌───▼────────┐  ┌────────▼──────┐
│PostgreSQL  │  │  MongoDB       │
│(User/Card  │  │ (Fraud Log,    │
│ profiles,  │  │  Decisions)    │
│ chargeback)│  │                │
└────────────┘  └────────┬───────┘
                         │
        ┌────────────────┴──────────────┐
        │   Kafka Topics               │
        │ - fraud-event               │
        │ - chargeback-feedback       │
        │ - manual-review             │
        │ - rule-update               │
        └────────────────┬─────────────┘
                         │
        ┌────────────────▼───────────┐
        │  Analytics & Monitoring    │
        │  (Kibana, Grafana)         │
        └────────────────────────────┘
```

---

## Core Design Questions Answered

### Q1: How do you detect card testing attacks in real-time?

**Answer:** Card testing = attacker uses stolen card with multiple small amounts (<$5) to test if card is active before large fraud. Detection strategy:

1. **Redis sliding window counter** keyed by `cardHash` (first 6 + last 4 digits)
2. **Window:** Last 1 hour
3. **Threshold:** If count > 3 for small amounts (<$5), flag as high risk
4. **Action:** If risk > 50, require 3D Secure; if risk > 80, decline

**Pseudo-code:**
```
cardHash = SHA256(card_first_6 + card_last_4)
key = "fraud:card:{cardHash}:small_txns:1h"
count = redis.INCR(key)
redis.EXPIRE(key, 3600)
if count > 3 and amount < 500_cents:
  risk_score += 30
```

---

### Q2: How do you implement velocity checks across distributed systems?

**Answer:** Use Redis sorted set with timestamp scores. Multiple checks per transaction:

1. **Per-card velocity:** Same card, multiple txns in short window
2. **Per-device velocity:** Same device, multiple txns in short window
3. **Per-IP velocity:** Same IP, multiple txns in short window

**Distributed setup:** Redis cluster with key sharding by cardHash ensures consistent velocity counts across nodes.

**Pseudo-code:**
```java
// Per-card: txns in last 60 seconds
key = "fraud:velocity:card:{cardHash}:60s"
redis.ZADD(key, System.currentTimeMillis(), txn_id)
count = redis.ZCOUNT(key, min_timestamp, now)
redis.EXPIRE(key, 60)
if count > 10:
  risk_score += 50 // High velocity
```

---

### Q3: How do you integrate 3D Secure without adding friction?

**Answer:** Risk-based 3DS integration:

- **Risk < 30:** Skip 3DS (low friction)
- **Risk 30–50:** Use frictionless 3DS (device data only, often approved instantly)
- **Risk > 50:** Full 3DS challenge (user enters password)
- **Risk > 80:** Decline immediately (don't even offer 3DS)

**Flow:**
```
Payment Gateway Decision
  ↓
Risk Score Calculation
  ↓
  [if risk > 50]
    → Request 3DS Authentication from Payment Processor
    → Wait for ACS (Access Control Server) response
    → Proceed if authenticated
  [if risk <= 50]
    → Proceed without 3DS
```

---

### Q4: How do you maintain and update fraud rules?

**Answer:** Rule engine loaded from database; hot-reloaded via Kafka config events:

1. **Rules stored in PostgreSQL:** `rule_id, name, condition, action, enabled, version`
2. **Drools-style DSL:** Rules written as:
   ```
   rule "Card Testing Detection"
     when
       $fraud: FraudContext(smallTransactionCount > 3, amount < 500)
     then
       $fraud.addRiskScore(30);
   end
   ```
3. **Hot reload:** When rule changes, publish `rule-update` event → Kafka → all fraud services reload rules immediately
4. **Versioning:** Each rule has version number; track which version executed for each decision (audit trail)

---

### Q5: How do you handle false positives gracefully?

**Answer:**
1. **Allow-list by customer history:** If customer has 100+ clean transactions, lower threshold for flag
2. **Allow-list by payment amount:** If amount matches recent patterns, don't flag
3. **Manual review queue:** Medium-risk txns (40–70 score) sent to queue for human review
4. **Customer override:** Let customer confirm "Yes, this is me" via email/SMS; update allow-list
5. **Machine learning feedback:** Log false positives; retrain model to avoid similar flags

---

### Q6: How do you collaborate with payment gateways for chargeback data?

**Answer:**
1. **Daily chargeback feed:** Payment gateway (Stripe, Square) provides CSV/API feed of chargebacks
2. **Ingest to PostgreSQL:** Store `chargeback_id, card_hash, reason, dispute_date`
3. **Update BIN blocklist:** If chargeback rate for BIN > 5%, add BIN to blocklist in Redis
4. **Update ML model features:** Feed chargeback data as negative labels to retrain risk model
5. **Notify team:** Alert if chargeback rate exceeds SLA

---

## Microservices Breakdown

| Service | Responsibility | Tech | Throughput |
|---------|-----------------|------|-----------|
| **Fraud Decision Engine** | Calculate risk score, make accept/reject/challenge decision | Spring Boot + Drools | 465 txns/sec peak |
| **Device Fingerprinting Service** | Generate device fingerprint, detect new devices | Spring Boot + Crypto | 465 requests/sec |
| **Velocity Checker** | Check velocity across Redis, return count | Spring Boot + Jedis | 930 checks/sec |
| **BIN Blocklist Service** | Maintain bloom filter, check BIN against list | Spring Boot + Redis | 465 checks/sec |
| **3DS Orchestrator** | Request 3DS challenge from payment gateway, poll for result | Spring Boot + HTTP | 100 challenges/sec |
| **Manual Review Handler** | Queue medium-risk transactions for human review | Spring Boot + Queue | 50 events/sec |
| **Chargeback Feedback Consumer** | Ingest chargebacks, update BIN blocklist | Kafka Consumer | 20 chargebacks/sec |

---

## Database Design (DDL)

```sql
-- PostgreSQL Schema

CREATE TABLE fraud_rules (
    id BIGSERIAL PRIMARY KEY,
    rule_name VARCHAR(255) NOT NULL UNIQUE,
    rule_type VARCHAR(50) NOT NULL, -- velocity, card_testing, geographic, device
    condition_dsl TEXT NOT NULL, -- Drools DSL or JSON condition
    action_dsl TEXT NOT NULL, -- Risk score increment, decline, challenge
    enabled BOOLEAN DEFAULT TRUE,
    version INT NOT NULL DEFAULT 1,
    created_by VARCHAR(100),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE fraud_decisions (
    id BIGSERIAL PRIMARY KEY,
    transaction_id VARCHAR(100) UNIQUE NOT NULL,
    card_hash VARCHAR(64) NOT NULL,
    device_fingerprint VARCHAR(64),
    ip_address INET NOT NULL,
    amount_cents BIGINT NOT NULL,
    currency VARCHAR(3),
    risk_score INT NOT NULL, -- 0-100
    decision VARCHAR(50) NOT NULL, -- ACCEPT, DECLINE, CHALLENGE_3DS, MANUAL_REVIEW
    reasons TEXT, -- JSON array of rule violations
    rules_version INT,
    customer_id BIGINT,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_fraud_decisions_transaction ON fraud_decisions(transaction_id);
CREATE INDEX idx_fraud_decisions_card_hash ON fraud_decisions(card_hash, timestamp);
CREATE INDEX idx_fraud_decisions_decision ON fraud_decisions(decision);

CREATE TABLE fraud_feedback (
    id BIGSERIAL PRIMARY KEY,
    fraud_decision_id BIGSERIAL REFERENCES fraud_decisions(id),
    feedback_type VARCHAR(50) NOT NULL, -- CONFIRMED_FRAUD, FALSE_POSITIVE, CHARGEBACK
    reported_by VARCHAR(100),
    customer_comment TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE chargebacks (
    id BIGSERIAL PRIMARY KEY,
    chargeback_id VARCHAR(100) UNIQUE NOT NULL,
    transaction_id VARCHAR(100),
    card_hash VARCHAR(64),
    amount_cents BIGINT,
    reason_code VARCHAR(20), -- 4855, 4863, etc.
    reason_description TEXT,
    dispute_date DATE,
    received_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_chargebacks_card_hash ON chargebacks(card_hash);
CREATE INDEX idx_chargebacks_received_at ON chargebacks(received_at);

CREATE TABLE manual_review_queue (
    id BIGSERIAL PRIMARY KEY,
    fraud_decision_id BIGINT NOT NULL REFERENCES fraud_decisions(id),
    transaction_id VARCHAR(100),
    risk_score INT,
    assigned_to VARCHAR(100),
    review_status VARCHAR(50) DEFAULT 'PENDING', -- PENDING, APPROVED, DECLINED, ESCALATED
    reviewer_comment TEXT,
    reviewed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_manual_review_status ON manual_review_queue(review_status);

CREATE TABLE device_fingerprints (
    id BIGSERIAL PRIMARY KEY,
    device_fingerprint VARCHAR(64) UNIQUE NOT NULL,
    customer_id BIGINT,
    user_agent VARCHAR(255),
    browser_language VARCHAR(10),
    accept_language VARCHAR(50),
    screen_resolution VARCHAR(20),
    timezone VARCHAR(50),
    first_seen TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_seen TIMESTAMP,
    transaction_count INT DEFAULT 1,
    is_trusted BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
CREATE INDEX idx_device_fingerprints_customer ON device_fingerprints(customer_id);
CREATE INDEX idx_device_fingerprints_last_seen ON device_fingerprints(last_seen);

CREATE TABLE bin_blocklist (
    id BIGSERIAL PRIMARY KEY,
    bin VARCHAR(6) UNIQUE NOT NULL, -- First 6 digits of card
    reason VARCHAR(100),
    chargeback_count INT DEFAULT 0,
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP, -- NULL = permanent
    is_active BOOLEAN DEFAULT TRUE
);
CREATE INDEX idx_bin_blocklist_expires ON bin_blocklist(expires_at);

CREATE TABLE customer_allowlist (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    card_hash VARCHAR(64),
    device_fingerprint VARCHAR(64),
    status VARCHAR(50) DEFAULT 'ACTIVE', -- ACTIVE, REMOVED, EXPIRED
    clean_transaction_count INT DEFAULT 0,
    added_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(customer_id, card_hash)
);
CREATE INDEX idx_allowlist_customer ON customer_allowlist(customer_id);
```

---

## Redis Data Structures

```python
# Velocity Checks (Sorted Set: timestamp scores)
fraud:velocity:card:{cardHash}:60s → ZSET
  Members: {transaction_id}, Score: {timestamp_ms}
  Query: ZCOUNT key min_timestamp now → count
  Used to detect "10 txns in 60 seconds"

fraud:velocity:device:{fingerprint}:60s → ZSET
  Members: {transaction_id}, Score: {timestamp_ms}

fraud:velocity:ip:{ip_address}:60s → ZSET
  Members: {transaction_id}, Score: {timestamp_ms}

# Card Testing Detection (Key-Value: hour sliding window)
fraud:card:{cardHash}:small_txns:1h → INT (count)
  Incremented on each small (<$5) transaction
  TTL: 3600 seconds
  Threshold: count > 3 → flag

# Geographic Anomalies (Sorted Set: recent countries)
fraud:geo:card:{cardHash}:locations → ZSET
  Members: {country_code}, Score: {timestamp_ms}
  Query: ZCOUNT for txns in last 15 minutes
  If 2+ countries in 15 min: flag

# BIN Blocklist (Bloom Filter or Set)
fraud:bin:blocklist → SET
  Members: {bin} (first 6 digits of card)
  Lookup: SISMEMBER key bin
  Populated from PostgreSQL chargebacks table

# Rate Cards (Hash: pricing lookup)
fraud:rules:{rules_version} → HASH
  Fields: {rule_name}, Value: {rule_dsl_json}
  TTL: 24 hours (or manual invalidation on update)

# Device Reputation (Hash: device → score)
fraud:device:{fingerprint}:reputation → HASH
  Fields: {clean_count, total_count, last_fraud_time}
  TTL: 30 days
```

---

## Kafka Event Flow

```
┌──────────────────────────────────────────────────────┐
│ Kafka Topics & Event Flow                            │
└──────────────────────────────────────────────────────┘

1. payment-request (source: payment API)
   └─ Payload: { transaction_id, card_hash, amount, ip, device_fingerprint }
   └─ Consumers: FraudDecisionEngine, Analytics

2. fraud-event (result: fraud decision engine)
   └─ Payload: { transaction_id, risk_score, decision, reasons, rules_version }
   └─ Consumers: PaymentGateway, ManualReviewService, Analytics
   └─ Partitioning: by transaction_id

3. chargeback-feedback (source: payment gateway daily feed)
   └─ Payload: { chargeback_id, transaction_id, card_hash, reason, date }
   └─ Consumers: ChargebackProcessor (update BIN blocklist), ML model trainer
   └─ Partitioning: by card_hash

4. manual-review (medium-risk txns needing human review)
   └─ Payload: { transaction_id, risk_score, fraud_decision_id, customer_id }
   └─ Consumers: ManualReviewService, NotificationService
   └─ Partitioning: by customer_id

5. rule-update (rule changes from DB)
   └─ Payload: { rule_name, version, condition_dsl, action_dsl, enabled }
   └─ Consumers: All FraudDecisionEngine instances (hot reload)
   └─ Partitioning: broadcast topic

6. fraud-feedback (customer dispute or fraud confirmation)
   └─ Payload: { fraud_decision_id, feedback_type, customer_comment }
   └─ Consumers: ML model trainer, Rule optimizer
   └─ Partitioning: by feedback_type

Event Example (payment-request):
{
  "transaction_id": "txn-20240101-00001",
  "card_hash": "abc123def456",
  "amount_cents": 2999,
  "currency": "USD",
  "ip_address": "192.168.1.100",
  "device_fingerprint": "dev-fp-xyz789",
  "customer_id": 12345,
  "timestamp": 1704067200000
}
```

---

## Implementation Code

### 1. FraudDecisionEngine — Real-time Risk Scoring

```java
package com.payment.fraud.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.kie.api.runtime.KieSession;
import com.payment.fraud.model.*;
import java.util.ArrayList;
import java.util.List;

@Service
public class FraudDecisionEngine {

    @Autowired private VelocityChecker velocityChecker;
    @Autowired private BINBlocklistService binBlocklistService;
    @Autowired private DeviceFingerprintService deviceFingerprintService;
    @Autowired private GeographicAnomalyChecker geoChecker;
    @Autowired private KieSession kieSession; // Drools rules engine
    @Autowired private FraudDecisionRepository fraudDecisionRepo;
    @Autowired private KafkaTemplate<String, FraudEventPayload> kafkaTemplate;
    @Autowired private ThreeDSecureService threeDSecureService;

    private static final long DECISION_TIMEOUT_MS = 100;

    /**
     * Main entry point: make fraud decision in <100ms
     */
    public FraudDecision evaluateTransaction(PaymentRequest request) {
        long startTime = System.currentTimeMillis();

        FraudContext context = buildFraudContext(request);

        // Execute Drools rules in parallel
        executeFraudRules(context);

        int riskScore = context.getRiskScore();
        String decision = determineDecision(riskScore);

        FraudDecision fraudDecision = new FraudDecision();
        fraudDecision.setTransactionId(request.getTransactionId());
        fraudDecision.setCardHash(request.getCardHash());
        fraudDecision.setDeviceFingerprint(context.getDeviceFingerprint());
        fraudDecision.setIpAddress(request.getIpAddress());
        fraudDecision.setAmountCents(request.getAmountCents());
        fraudDecision.setRiskScore(riskScore);
        fraudDecision.setDecision(decision);
        fraudDecision.setReasons(context.getViolatedRules());

        long elapsedMs = System.currentTimeMillis() - startTime;
        if (elapsedMs > DECISION_TIMEOUT_MS) {
            // Log but don't fail; fraud service must be fast
            System.err.println("Fraud decision exceeded SLA: " + elapsedMs + "ms");
        }

        // Persist decision to PostgreSQL (async)
        fraudDecisionRepo.save(fraudDecision);

        // Publish fraud event to Kafka
        publishFraudEvent(fraudDecision);

        return fraudDecision;
    }

    private FraudContext buildFraudContext(PaymentRequest request) {
        FraudContext context = new FraudContext();
        context.setTransactionId(request.getTransactionId());
        context.setCardHash(request.getCardHash());
        context.setAmountCents(request.getAmountCents());
        context.setIpAddress(request.getIpAddress());
        context.setCustomerId(request.getCustomerId());

        // Device fingerprinting
        String deviceFingerprint = deviceFingerprintService.generateFingerprint(request);
        context.setDeviceFingerprint(deviceFingerprint);

        // Velocity checks
        int cardVelocity = velocityChecker.checkCardVelocity(request.getCardHash(), 60); // 60 sec window
        int deviceVelocity = velocityChecker.checkDeviceVelocity(deviceFingerprint, 60);
        int ipVelocity = velocityChecker.checkIPVelocity(request.getIpAddress(), 60);

        context.setCardVelocity(cardVelocity);
        context.setDeviceVelocity(deviceVelocity);
        context.setIpVelocity(ipVelocity);

        // BIN check
        boolean isBINBlocked = binBlocklistService.isBINBlocked(request.getCardHash().substring(0, 6));
        context.setBinBlocked(isBINBlocked);

        // Geographic check
        String previousCountry = geoChecker.getPreviousCountry(request.getCardHash());
        boolean isGeoAnomaly = geoChecker.isAnomalousLocation(previousCountry, request.getCountry(), 15); // 15 min window
        context.setGeoAnomaly(isGeoAnomaly);
        context.setCurrentCountry(request.getCountry());
        context.setPreviousCountry(previousCountry);

        // Device reputation
        var deviceReputation = deviceFingerprintService.getReputation(deviceFingerprint);
        context.setDeviceCleanCount(deviceReputation.getOrDefault("clean_count", 0));
        context.setDeviceTotalCount(deviceReputation.getOrDefault("total_count", 0));

        return context;
    }

    private void executeFraudRules(FraudContext context) {
        // Insert facts into Drools session
        kieSession.insert(context);

        // Fire all matching rules
        kieSession.fireAllRules();

        // Note: Rules in Drools will modify context.riskScore directly
        // Example rule:
        // rule "Velocity High"
        //   when $ctx: FraudContext(cardVelocity > 10)
        //   then $ctx.addRiskScore(30);
        // end
    }

    private String determineDecision(int riskScore) {
        if (riskScore > 80) {
            return "DECLINE";
        } else if (riskScore > 50) {
            return "CHALLENGE_3DS";
        } else if (riskScore > 30) {
            return "MANUAL_REVIEW";
        } else {
            return "ACCEPT";
        }
    }

    private void publishFraudEvent(FraudDecision fraudDecision) {
        FraudEventPayload event = new FraudEventPayload(
            fraudDecision.getTransactionId(),
            fraudDecision.getRiskScore(),
            fraudDecision.getDecision(),
            fraudDecision.getReasons()
        );
        kafkaTemplate.send("fraud-event", fraudDecision.getTransactionId(), event);
    }
}

// Drools Rule File (fraud-rules.drl)
/*
package com.payment.fraud.rules;

import com.payment.fraud.model.FraudContext;

rule "Card Testing Detection"
  when
    $ctx: FraudContext(
      amountCents < 500,
      cardVelocity > 3
    )
  then
    $ctx.addRiskScore(30);
    $ctx.addViolatedRule("Card testing detected");
end

rule "High Velocity Card"
  when
    $ctx: FraudContext(cardVelocity > 10)
  then
    $ctx.addRiskScore(50);
    $ctx.addViolatedRule("High velocity: " + $ctx.getCardVelocity() + " txns/min");
end

rule "High Velocity IP"
  when
    $ctx: FraudContext(ipVelocity > 20)
  then
    $ctx.addRiskScore(25);
    $ctx.addViolatedRule("High IP velocity");
end

rule "Blocked BIN"
  when
    $ctx: FraudContext(binBlocked == true)
  then
    $ctx.addRiskScore(100); // Automatic decline
    $ctx.addViolatedRule("BIN is blocked");
end

rule "Geographic Anomaly"
  when
    $ctx: FraudContext(geoAnomaly == true)
  then
    $ctx.addRiskScore(40);
    $ctx.addViolatedRule("Card used in " + $ctx.getCurrentCountry() +
                        " after recent use in " + $ctx.getPreviousCountry());
end

rule "New Device High Transaction"
  when
    $ctx: FraudContext(
      deviceCleanCount < 5,
      amountCents > 10000
    )
  then
    $ctx.addRiskScore(35);
    $ctx.addViolatedRule("New device with high transaction amount");
end
*/
```

### 2. VelocityChecker — Distributed Velocity Checks

```java
package com.payment.fraud.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import java.util.Set;
import java.util.concurrent.TimeUnit;

@Service
public class VelocityChecker {

    @Autowired private RedisTemplate<String, String> redisTemplate;

    /**
     * Check how many transactions for this card in the last N seconds
     */
    public int checkCardVelocity(String cardHash, int windowSeconds) {
        String key = "fraud:velocity:card:" + cardHash + ":" + windowSeconds + "s";

        Long count = redisTemplate.opsForZSet().count(key,
            System.currentTimeMillis() - (windowSeconds * 1000L),
            System.currentTimeMillis()
        );

        // Increment counter
        redisTemplate.opsForZSet().add(key, java.util.UUID.randomUUID().toString(),
            System.currentTimeMillis());

        // Set expiration
        redisTemplate.expire(key, windowSeconds + 10, TimeUnit.SECONDS);

        return count != null ? count.intValue() : 0;
    }

    /**
     * Check device velocity
     */
    public int checkDeviceVelocity(String deviceFingerprint, int windowSeconds) {
        String key = "fraud:velocity:device:" + deviceFingerprint + ":" + windowSeconds + "s";

        Long count = redisTemplate.opsForZSet().count(key,
            System.currentTimeMillis() - (windowSeconds * 1000L),
            System.currentTimeMillis()
        );

        redisTemplate.opsForZSet().add(key, java.util.UUID.randomUUID().toString(),
            System.currentTimeMillis());
        redisTemplate.expire(key, windowSeconds + 10, TimeUnit.SECONDS);

        return count != null ? count.intValue() : 0;
    }

    /**
     * Check IP velocity
     */
    public int checkIPVelocity(String ipAddress, int windowSeconds) {
        String key = "fraud:velocity:ip:" + ipAddress + ":" + windowSeconds + "s";

        Long count = redisTemplate.opsForZSet().count(key,
            System.currentTimeMillis() - (windowSeconds * 1000L),
            System.currentTimeMillis()
        );

        redisTemplate.opsForZSet().add(key, java.util.UUID.randomUUID().toString(),
            System.currentTimeMillis());
        redisTemplate.expire(key, windowSeconds + 10, TimeUnit.SECONDS);

        return count != null ? count.intValue() : 0;
    }
}
```

### 3. BINBlocklistService — Bloom Filter BIN Lookup

```java
package com.payment.fraud.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;
import org.springframework.scheduling.annotation.Scheduled;
import java.util.HashSet;
import java.util.Set;

@Service
public class BINBlocklistService {

    @Autowired private RedisTemplate<String, String> redisTemplate;
    @Autowired private ChargebackRepository chargebackRepo;

    private static final String BIN_BLOCKLIST_KEY = "fraud:bin:blocklist";
    private static final double CHARGEBACK_RATE_THRESHOLD = 0.05; // 5%

    /**
     * Check if a BIN (first 6 digits) is blocked
     */
    public boolean isBINBlocked(String bin) {
        Boolean isMember = redisTemplate.opsForSet().isMember(BIN_BLOCKLIST_KEY, bin);
        return isMember != null && isMember;
    }

    /**
     * Add BIN to blocklist (called when chargeback rate exceeds threshold)
     */
    public void blockBIN(String bin, String reason) {
        redisTemplate.opsForSet().add(BIN_BLOCKLIST_KEY, bin);
        // Also store in PostgreSQL for audit
    }

    /**
     * Scheduled job to refresh blocklist from chargebacks
     */
    @Scheduled(fixedRate = 3600000) // Every hour
    public void refreshBINBlocklistFromChargebacks() {
        // Query chargebacks in last 30 days
        var chargebacks = chargebackRepo.findByReceivedAtGreaterThan(
            java.time.LocalDateTime.now().minusDays(30));

        // Group by BIN and calculate chargeback rate
        var binStats = new java.util.HashMap<String, BINChargebackStats>();

        for (Chargeback cb : chargebacks) {
            String bin = cb.getCardHash().substring(0, 6);
            binStats.putIfAbsent(bin, new BINChargebackStats());

            BINChargebackStats stats = binStats.get(bin);
            stats.addChargeback();
        }

        // Add BINs with high chargeback rate to blocklist
        for (var entry : binStats.entrySet()) {
            String bin = entry.getKey();
            BINChargebackStats stats = entry.getValue();

            double chargebackRate = (double) stats.getChargebackCount() / stats.getTotalTransactions();
            if (chargebackRate > CHARGEBACK_RATE_THRESHOLD) {
                blockBIN(bin, "High chargeback rate: " + chargebackRate);
            }
        }
    }

    /**
     * Kafka consumer for real-time chargeback feedback
     */
    @KafkaListener(topics = "chargeback-feedback")
    public void handleChargebackFeedback(ChargebackFeedbackEvent event) {
        String bin = event.getCardHash().substring(0, 6);

        // Increment chargeback count for BIN
        String countKey = "fraud:bin:" + bin + ":chargeback_count";
        redisTemplate.opsForValue().increment(countKey);

        // Check if rate exceeded; if so, block
        var chargebacks = chargebackRepo.findByCardHashStartsWith(bin);
        double rate = (double) chargebacks.size() / 1000; // Estimate
        if (rate > CHARGEBACK_RATE_THRESHOLD) {
            blockBIN(bin, "Real-time chargeback threshold exceeded");
        }
    }

    private static class BINChargebackStats {
        int chargebackCount = 0;
        int totalTransactions = 1000; // Estimated

        void addChargeback() {
            chargebackCount++;
        }

        int getChargebackCount() { return chargebackCount; }
        int getTotalTransactions() { return totalTransactions; }
    }
}
```

### 4. DeviceFingerprintService — Device Tracking & Reputation

```java
package com.payment.fraud.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import com.payment.fraud.model.PaymentRequest;
import java.security.MessageDigest;
import java.util.HashMap;
import java.util.Map;

@Service
public class DeviceFingerprintService {

    @Autowired private RedisTemplate<String, String> redisTemplate;
    @Autowired private DeviceFingerprintRepository deviceFingerprintRepo;

    /**
     * Generate device fingerprint from request headers
     */
    public String generateFingerprint(PaymentRequest request) {
        // Combine multiple signals
        String fingerprintInput = String.format(
            "%s|%s|%s|%s|%s",
            request.getUserAgent(),
            request.getAcceptLanguage(),
            request.getScreenResolution(),
            request.getTimezone(),
            request.getBrowserLanguage()
        );

        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(fingerprintInput.getBytes());
            return bytesToHex(hash);
        } catch (Exception e) {
            throw new RuntimeException("Failed to generate device fingerprint", e);
        }
    }

    /**
     * Get device reputation (clean txn count, total txn count, last fraud time)
     */
    public Map<String, Integer> getReputation(String deviceFingerprint) {
        String key = "fraud:device:" + deviceFingerprint + ":reputation";

        Map<String, String> reputationMap = redisTemplate.opsForHash().getAll(key);

        Map<String, Integer> reputation = new HashMap<>();
        reputation.put("clean_count",
            reputationMap.containsKey("clean_count") ? Integer.parseInt((String) reputationMap.get("clean_count")) : 0);
        reputation.put("total_count",
            reputationMap.containsKey("total_count") ? Integer.parseInt((String) reputationMap.get("total_count")) : 0);

        return reputation;
    }

    /**
     * Update device reputation after transaction
     */
    public void updateReputation(String deviceFingerprint, boolean isClean) {
        String key = "fraud:device:" + deviceFingerprint + ":reputation";

        if (isClean) {
            redisTemplate.opsForHash().increment(key, "clean_count", 1);
        }
        redisTemplate.opsForHash().increment(key, "total_count", 1);
        redisTemplate.opsForHash().put(key, "last_seen", String.valueOf(System.currentTimeMillis()));

        // TTL: 30 days
        redisTemplate.expire(key, 30 * 24 * 3600, java.util.concurrent.TimeUnit.SECONDS);

        // Also persist to PostgreSQL
        var deviceFp = deviceFingerprintRepo.findByDeviceFingerprint(deviceFingerprint)
            .orElse(new DeviceFingerprint());
        if (isClean) {
            deviceFp.setCleanTransactionCount(deviceFp.getCleanTransactionCount() + 1);
        }
        deviceFp.setTransactionCount(deviceFp.getTransactionCount() + 1);
        deviceFp.setLastSeen(java.time.LocalDateTime.now());
        deviceFingerprintRepo.save(deviceFp);
    }

    private String bytesToHex(byte[] bytes) {
        StringBuilder sb = new StringBuilder();
        for (byte b : bytes) {
            sb.append(String.format("%02x", b));
        }
        return sb.toString();
    }
}
```

### 5. ThreeDSecureService — 3DS Orchestration

```java
package com.payment.fraud.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpEntity;
import org.springframework.http.HttpHeaders;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestTemplate;

@Service
public class ThreeDSecureService {

    @Autowired private RestTemplate restTemplate;
    @Autowired private PaymentGatewayConfig paymentConfig;

    private static final int CHALLENGE_TIMEOUT_MS = 30000; // 30 second timeout

    /**
     * Request 3DS authentication from payment gateway
     * Returns auth URL or result
     */
    public ThreeDSecureResult initiate3DSChallenge(String transactionId, String cardHash, long amountCents) {
        // Call to Stripe/Square API
        String apiUrl = paymentConfig.getPaymentGatewayUrl() + "/3ds/initiate";

        HttpHeaders headers = new HttpHeaders();
        headers.set("Authorization", "Bearer " + paymentConfig.getApiKey());
        headers.set("Content-Type", "application/json");

        String requestBody = String.format(
            "{\"transaction_id\": \"%s\", \"amount\": %d, \"currency\": \"USD\"}",
            transactionId, amountCents
        );

        try {
            ResponseEntity<String> response = restTemplate.postForEntity(
                apiUrl,
                new HttpEntity<>(requestBody, headers),
                String.class
            );

            // Parse response (usually JSON with auth URL or status)
            var result = new ThreeDSecureResult();
            result.setStatus("INITIATED");
            result.setAuthUrl(parseAuthUrl(response.getBody()));
            result.setAcsTransactionId(parseAcsTransactionId(response.getBody()));
            return result;
        } catch (Exception e) {
            throw new RuntimeException("3DS initiation failed", e);
        }
    }

    /**
     * Poll for 3DS authentication result
     */
    public ThreeDSecureResult poll3DSResult(String acsTransactionId, int maxWaitMs) {
        long startTime = System.currentTimeMillis();

        while (System.currentTimeMillis() - startTime < maxWaitMs) {
            String apiUrl = paymentConfig.getPaymentGatewayUrl() + "/3ds/status/" + acsTransactionId;

            try {
                ResponseEntity<String> response = restTemplate.getForEntity(apiUrl, String.class);
                var result = new ThreeDSecureResult();

                // Parse status from response
                if (response.getBody().contains("\"authenticated\": true")) {
                    result.setStatus("AUTHENTICATED");
                    result.setAuthenticated(true);
                    return result;
                } else if (response.getBody().contains("\"authenticated\": false")) {
                    result.setStatus("FAILED");
                    result.setAuthenticated(false);
                    return result;
                }

                // Still pending; wait and retry
                Thread.sleep(1000);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                break;
            }
        }

        // Timeout
        var result = new ThreeDSecureResult();
        result.setStatus("TIMEOUT");
        return result;
    }

    private String parseAuthUrl(String jsonResponse) {
        // Simple JSON parsing; use Jackson ObjectMapper in production
        int start = jsonResponse.indexOf("\"auth_url\": \"") + 13;
        int end = jsonResponse.indexOf("\"", start);
        return jsonResponse.substring(start, end);
    }

    private String parseAcsTransactionId(String jsonResponse) {
        int start = jsonResponse.indexOf("\"acs_transaction_id\": \"") + 23;
        int end = jsonResponse.indexOf("\"", start);
        return jsonResponse.substring(start, end);
    }
}

public class ThreeDSecureResult {
    private String status; // INITIATED, AUTHENTICATED, FAILED, TIMEOUT
    private String authUrl;
    private String acsTransactionId;
    private boolean authenticated;

    // Getters, Setters
}
```

### 6. GeographicAnomalyChecker — Location-based Fraud Detection

```java
package com.payment.fraud.service;

import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;
import java.util.concurrent.TimeUnit;

@Service
public class GeographicAnomalyChecker {

    @Autowired private RedisTemplate<String, String> redisTemplate;

    /**
     * Get the most recent country for a card
     */
    public String getPreviousCountry(String cardHash) {
        String key = "fraud:geo:card:" + cardHash + ":last_country";
        return redisTemplate.opsForValue().get(key);
    }

    /**
     * Check if location change is anomalous
     * Anomaly: 2+ countries in windowSeconds (e.g., 15 minutes)
     */
    public boolean isAnomalousLocation(String previousCountry, String currentCountry, int windowSeconds) {
        // Same country = no anomaly
        if (previousCountry != null && previousCountry.equals(currentCountry)) {
            return false;
        }

        // Different country: check if it's possible (geolocation distance)
        if (previousCountry != null && !previousCountry.equals(currentCountry)) {
            // Simple heuristic: allow country change only if >4 hours have passed
            // In production, use GeoIP distance calculation
            return true; // Flag as anomaly if different country
        }

        return false;
    }

    /**
     * Record current transaction location
     */
    public void recordLocation(String cardHash, String country) {
        String key = "fraud:geo:card:" + cardHash + ":last_country";
        redisTemplate.opsForValue().set(key, country);

        // Also record in sorted set for historical tracking
        String historyKey = "fraud:geo:card:" + cardHash + ":locations";
        redisTemplate.opsForZSet().add(historyKey, country + "_" + System.currentTimeMillis(),
            System.currentTimeMillis());

        // Keep only last 24 hours
        long cutoff = System.currentTimeMillis() - (24 * 3600 * 1000);
        redisTemplate.opsForZSet().removeRangeByScore(historyKey, 0, cutoff);

        redisTemplate.expire(historyKey, 24, TimeUnit.HOURS);
    }
}
```

---

## Failure Scenarios & Recovery

| Scenario | Detection | Recovery |
|----------|-----------|----------|
| **Fraud service timeout** | Request > 100ms | Fall back to simple threshold rule (amount > $10K = manual review) |
| **Redis unavailable** | Connection refused | Circuit breaker; use local in-memory cache with TTL; alert ops |
| **Rules engine crash** | Exception in Drools | Revert to previous rules version from DB; alert |
| **3DS gateway down** | HTTP timeout to payment processor | Mark txn as PENDING; retry 3DS check later; don't auto-decline |
| **BIN blocklist stale** | Chargeback feedback delayed | Refresh blocklist more frequently (every 15 min instead of hourly) |
| **False positive spam** | Customer complaints > 100/hour | Temporarily disable rule; investigate; publish emergency pause via Kafka |

---

## Scaling Strategy

### Horizontal Scaling
1. **Fraud Decision Service:** Stateless; scale horizontally across 10+ nodes via load balancer
2. **Redis Cluster:** 16 shards for velocity checks; auto-failover with Sentinel
3. **Kafka Consumer Groups:** 10+ workers per consumer group for chargeback feedback and rule updates
4. **Manual Review Queue:** Async processing; queue workers scale independently

### Database Scaling
- **PostgreSQL:** Sharding by card_hash for fraud_decisions table; read replicas for analytics
- **MongoDB:** Not used in fraud detection (low write volume)

---

## Monitoring & Alerting

### Key Metrics
```
# Throughput
fraud.decisions.total (counter)
fraud.decision_latency_ms (histogram, p99 < 100ms)
fraud.challenges.3ds.count (counter)
fraud.declines.count (counter)

# Reliability
fraud.engine.errors.total (counter)
fraud.redis.connection_failures.total (counter)
fraud.rules.reload.total (counter)

# Business Metrics
fraud.risk_score.distribution (histogram)
fraud.true_positive_rate (gauge) = confirmed_fraud / total_decisions
fraud.false_positive_rate (gauge) = manual_review_overrides / manual_reviews
chargebacks.count.daily (gauge)
```

### Alert Rules
```
ALERT FraudDecisionLatencyHigh
  IF fraud.decision_latency_ms{quantile="0.99"} > 100
  FOR 5m

ALERT FraudServiceErrors
  IF rate(fraud.engine.errors.total[5m]) > 0.1
  FOR 5m

ALERT ChargebackRateHigh
  IF rate(chargebacks.count.daily[1d]) > 0.001
  FOR 1h
```

---

## Summary Cheat Sheet

| Component | Pattern | Key Code |
|-----------|---------|----------|
| **Real-time Risk Scoring** | Drools rules engine + context facts | `FraudDecisionEngine.evaluateTransaction()` |
| **Velocity Detection** | Redis sorted set with timestamp scores | `VelocityChecker.checkCardVelocity(cardHash, windowSeconds)` |
| **BIN Blocklist** | Redis set for O(1) lookup | `BINBlocklistService.isBINBlocked(bin)` |
| **Card Testing** | Redis counter for small txns in 1h window | `fraud:card:{cardHash}:small_txns:1h` |
| **Geographic Anomaly** | Redis geo data + time-based checks | `GeographicAnomalyChecker.isAnomalousLocation()` |
| **Device Fingerprinting** | SHA-256 hash of browser/device signals | `DeviceFingerprintService.generateFingerprint()` |
| **3DS Integration** | Risk-based challenge threshold | `if risk > 50: challenge_3ds` |
| **Manual Review Queue** | Async processing for 30–70 risk scores | Risk 30–50 → manual_review topic |
| **Chargeback Feedback Loop** | Daily Kafka feed → update BIN blocklist | ChargebackProcessor consumes feed |
| **Hot Reload Rules** | Rule update event on Kafka broadcast topic | All instances reload immediately |

