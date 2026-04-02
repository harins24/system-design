---
title: Auction and Bidding System
layout: default
---

# Auction and Bidding System — Deep Dive Design

> **Scenario:** Sellers list items with starting bid and reserve price. Buyers place bids. Highest bidder wins when auction ends. Proxy bidding (auto-bid up to max). Sniping prevention (extend auction if bid in last 2 minutes). Payment processing for winners. 100 concurrent auctions, 10K active bidders.
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
- Sellers create auctions with starting bid, reserve price, and duration
- Buyers place bids; system validates bid > current highest bid
- Proxy bidding: auto-bid up to bidder's max, charge second-highest + increment
- Sniping prevention: extend auction by 2 min if bid placed in last 2 min (max 10 extensions)
- Real-time bid notifications to all watching bidders (WebSocket)
- Auction automatically closes at end time; highest bidder wins
- Winner payment: system captures payment method on file
- Non-payment recovery: second-chance offer to next highest bidder
- View auction history and bid increments

### Non-Functional Requirements
- 100 concurrent auctions, 10K active bidders
- 1K bids/sec during peak hours
- Bid acceptance latency: <200 ms (p99)
- WebSocket broadcast to 100+ watchers per auction: <500 ms propagation
- Consistency: prevent two bidders winning same item
- Data retention: 5 years for dispute resolution

### Constraints
- No overbidding (validation on each bid)
- Auction cannot be modified after first bid placed
- Payment processing must be PCI-DSS compliant
- Sniping extensions capped at 10 per auction
- Bid increment rules per auction tier

---

## Capacity Estimation

| Metric | Value |
|--------|-------|
| Concurrent Auctions | 100 |
| Active Bidders | 10,000 |
| Bids per Second (peak) | 1,000 |
| Bids per Second (avg) | 200 |
| Average Bid Watchers per Auction | 50 |
| WebSocket Connections (peak) | 500K |
| Auction Duration (avg) | 7 days |
| Total Bids per Day | ~17M |
| Bid History Storage/Day | ~300 GB (ID, bidder, amount, timestamp) |
| PostgreSQL DB Size (YoY) | ~5 TB |
| MongoDB Document Volume | ~50M (auction metadata, bid history) |
| Redis Memory (peak) | ~50 GB (auction state, bid locks, watchers) |
| Bid Latency (p99) | <200 ms |
| WebSocket Broadcast Latency (p95) | <500 ms |

---

## High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                       Auction & Bidding System                          │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│              Frontend Layer (React + WebSocket Client)                   │
│  - Auction Listing   - Bid Placement   - Auction Timer   - Live Feed    │
└──────────────────────────────────────┬──────────────────────────────────┘
                                       │
┌──────────────────────────────────────┴──────────────────────────────────┐
│                  API Gateway + WebSocket Handler                         │
│           - Authentication (JWT)                                        │
│           - Rate Limiting (1000 bids/sec per user)                      │
│           - WebSocket upgrade + routing                                 │
└──────────────────────────────────────┬──────────────────────────────────┘
         ┌─────────────────────────────┼────────────────────────────┐
         │                             │                            │
    ┌────▼─────┐         ┌────────────▼──────┐        ┌───────────▼────┐
    │  Auction │         │  Bid Processing   │        │  Payment       │
    │  Service │         │  Service          │        │  Service       │
    │          │         │                   │        │                │
    └────┬─────┘         └────────┬──────────┘        └────────┬───────┘
         │                        │                           │
    ┌────▼────────────────────────▼──────────────────────────▼────────┐
    │              Kafka (Event Bus)                                   │
    │  - auction.created                                              │
    │  - bid.placed                                                   │
    │  - bid.proxy_executed                                           │
    │  - auction.closed                                               │
    │  - winner.determined                                            │
    │  - payment.captured                                             │
    └────────────────────────┬─────────────────────────────────────────┘
         ┌────────────────────┼────────────────┬──────────────────┐
    ┌────▼──┐  ┌──────────┐  ┌▼──────────┐  ┌─▼────────┐  ┌─────▼──┐
    │ Proxy │  │ Sniping  │  │ Notif     │  │ Payment  │  │Second  │
    │Engine │  │Prevention│  │Service    │  │Gateway   │  │Chance  │
    └───────┘  └──────────┘  └───────────┘  └──────────┘  └────────┘

┌─────────────────────────────────────────────────────────────────────────┐
│                          Data Layer                                     │
│  ┌──────────────────┐  ┌──────────┐  ┌────────┐  ┌──────────────────┐ │
│  │   PostgreSQL     │  │ MongoDB  │  │ Redis  │  │  Message Queue   │ │
│  │ - Auctions       │  │ - Bid    │  │ - Locks│  │  (Rabbitmq/SQS) │ │
│  │ - Bids           │  │   History│  │ - State│  │  - Payment retry │ │
│  │ - Payment Methods│  │ - Auction│  │ - Cache│  │                  │ │
│  │ - Winners        │  │   Metadata│ │        │  │                  │ │
│  └──────────────────┘  └──────────┘  └────────┘  └──────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Core Design Questions Answered

### 1. How do you handle concurrent bids on the same item?

**Answer:** Redis distributed lock + serial processing + optimistic locking fallback.

- Auction ID is lock key: `SET NX EX {auction_id} {token} EX 5` (Redis atomic set-if-not-exists)
- Only one bid processor holds lock; others queue in Kafka partition queue
- If lock acquisition fails: retry with exponential backoff (10ms, 20ms, 50ms)
- Alternative: optimistic locking in PostgreSQL (version column) for fallback if Redis down
- Guarantee: bid order is linearized; no race conditions

### 2. How do you implement proxy bidding?

**Answer:** ProxyBidEngine computes winning price as second_highest + minimum_increment.

- On bid received: check bidder's max_bid in Redis
- If bidder already has proxy bid: update max_bid
- ProxyBidEngine logic:
  - Current leading bid: B₁ by bidder A
  - New bid from bidder B: max_bid = M
  - If M > B₁: B becomes leader at price B₁ + increment
  - Notify A of outbid; A's proxy can auto-bid if max ≥ (B₁ + increment)
  - Continue auto-bid chain until someone's max is exceeded
- Store proxy bids in Redis: `auction:{id}:proxy_bids = {bidder_id -> max_amount}`

### 3. How do you prevent auction sniping?

**Answer:** Automatic extension on late bids + cap on extensions.

- Track auction end time in Redis with TTL
- On bid received: if (now - end_time) < 2 minutes → extend by 2 minutes
- Increment extension_count in Redis atomic counter
- If extension_count >= 10 (max) → ignore further extensions
- Set new TTL: `EXPIRE auction:{id}:end_time 120` (2 minutes)
- Notify all watchers: "Auction extended to {new_end_time}"

### 4. How do you notify bidders when outbid?

**Answer:** WebSocket push + push notifications + email for delayed notifications.

- Bid placed → emit Kafka event `bid.placed`
- Notification service consumes: checks if bidder A is outbid
- Push outbid notification via WebSocket to bidder A (if connected)
- For offline bidders: send push notification (mobile) or email
- Include new leading bid amount and remaining time
- Latency target: <500 ms from bid placement to notification broadcast

### 5. How do you process payment for auction winners?

**Answer:** Capture pre-authorized payment; fallback to new authorization if needed.

- Auction closes → WinnerPaymentService queries winning bid
- Check payment method on file (credit card, PayPal, etc.)
- If card stored: attempt capture of auth hold
- If capture fails (expired card, insufficient funds): attempt new authorization
- 3 retries with exponential backoff
- If all retries fail: trigger second-chance offer to next bidder
- Idempotency: payment capture request has unique reference; prevents double-charge

### 6. How do you handle scenarios where winner doesn't pay?

**Answer:** Timeout → Second-chance offer → Negative feedback → Suspension.

- Winner has 48 hours to complete payment
- After 48h with no payment: mark as `NON_PAYMENT`
- Automatically offer item to second-highest bidder at their max bid amount
- Send 2nd-highest bidder: "Would you like this item for $X?" (24h to accept)
- If accepted: close second auction; charge 2nd bidder
- If rejected or timeout: offer to 3rd bidder (and so on)
- Mark original winner with negative feedback score
- After 3 non-payments: auto-suspend bidder (no new bids allowed)

---

## Microservices Breakdown

| Service | Responsibility | Key Classes | Dependencies |
|---------|-----------------|------------|--------------|
| **Auction Service** | Create auctions, manage lifecycle, enforce rules | `AuctionController`, `AuctionService`, `AuctionRepository`, `AuctionValidator` | PostgreSQL, Redis, Kafka |
| **Bid Processing Service** | Validate bids, apply proxy logic, enforce incrementing | `BidProcessorService`, `BidValidator`, `ProxyBidEngine`, `BidLockManager` | PostgreSQL, Redis, Kafka |
| **Sniper Prevention** | Detect late bids, extend auction, cap extensions | `SniperPreventionService`, `AuctionTimerManager` | Redis, Kafka |
| **Notification Service** | WebSocket push, email/SMS for outbid alerts | `NotificationService`, `WebSocketHandler`, `NotificationQueue` | Redis, Kafka, WebSocket |
| **Payment Service** | Capture payment, handle failures, retry logic | `PaymentCaptureService`, `PaymentGatewayAdapter`, `PaymentRetryQueue` | Payment Gateway, Kafka |
| **Second Chance Offer** | Auto-offer to next bidders on non-payment | `SecondChanceOfferService`, `OfferQueue` | PostgreSQL, Kafka |

---

## Database Design

### PostgreSQL Schema

```sql
-- Auctions Table
CREATE TABLE auctions (
    id BIGSERIAL PRIMARY KEY,
    seller_id UUID NOT NULL,
    title VARCHAR(255) NOT NULL,
    description TEXT,
    starting_bid DECIMAL(10, 2) NOT NULL,
    reserve_price DECIMAL(10, 2) NOT NULL,
    current_highest_bid DECIMAL(10, 2),
    current_highest_bidder_id UUID,
    auction_start_time TIMESTAMP NOT NULL,
    auction_end_time TIMESTAMP NOT NULL,
    actual_end_time TIMESTAMP,
    status VARCHAR(50) DEFAULT 'ACTIVE',
    view_count BIGINT DEFAULT 0,
    bid_count BIGINT DEFAULT 0,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_status_end_time (status, auction_end_time),
    INDEX idx_seller_id (seller_id),
    INDEX idx_winning_bidder (current_highest_bidder_id)
);

-- Bids Table (Time-series optimized)
CREATE TABLE bids (
    id BIGSERIAL PRIMARY KEY,
    auction_id BIGINT NOT NULL REFERENCES auctions(id),
    bidder_id UUID NOT NULL,
    max_bid_amount DECIMAL(10, 2) NOT NULL,
    current_bid_amount DECIMAL(10, 2) NOT NULL,
    is_proxy_bid BOOLEAN DEFAULT FALSE,
    bid_timestamp TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_auction_timestamp (auction_id, bid_timestamp),
    INDEX idx_bidder_id (bidder_id),
    INDEX idx_current_highest (auction_id, current_bid_amount DESC)
);

-- Auction Winners (Denormalized for fast lookup)
CREATE TABLE auction_winners (
    id BIGSERIAL PRIMARY KEY,
    auction_id BIGINT NOT NULL REFERENCES auctions(id),
    winner_id UUID NOT NULL,
    final_price DECIMAL(10, 2) NOT NULL,
    payment_status VARCHAR(50) DEFAULT 'PENDING',
    payment_timestamp TIMESTAMP,
    transaction_id VARCHAR(255),
    won_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(auction_id),
    INDEX idx_winner_payment_status (winner_id, payment_status)
);

-- Bidder Feedback & Reputation
CREATE TABLE bidder_reputation (
    id BIGSERIAL PRIMARY KEY,
    bidder_id UUID NOT NULL UNIQUE,
    total_auctions_won BIGINT DEFAULT 0,
    total_auctions_lost BIGINT DEFAULT 0,
    non_payment_count INT DEFAULT 0,
    average_feedback_score DECIMAL(3, 2),
    suspension_status VARCHAR(50) DEFAULT 'ACTIVE',
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_bidder_suspension (bidder_id, suspension_status)
);

-- Auction Extensions (Sniping prevention)
CREATE TABLE auction_extensions (
    id BIGSERIAL PRIMARY KEY,
    auction_id BIGINT NOT NULL REFERENCES auctions(id),
    extension_count INT DEFAULT 0,
    original_end_time TIMESTAMP NOT NULL,
    extended_end_time TIMESTAMP NOT NULL,
    last_extension_time TIMESTAMP,
    INDEX idx_auction_id (auction_id)
);

-- Payment Methods (Encrypted PCI-DSS compliant)
CREATE TABLE payment_methods (
    id BIGSERIAL PRIMARY KEY,
    bidder_id UUID NOT NULL,
    payment_token VARCHAR(512) NOT NULL,
    payment_type VARCHAR(50) NOT NULL,
    last_four_digits VARCHAR(4),
    expiry_month INT,
    expiry_year INT,
    is_default BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_bidder_id (bidder_id)
);

-- Second Chance Offers
CREATE TABLE second_chance_offers (
    id BIGSERIAL PRIMARY KEY,
    auction_id BIGINT NOT NULL REFERENCES auctions(id),
    offered_to_bidder_id UUID NOT NULL,
    offered_price DECIMAL(10, 2) NOT NULL,
    offer_status VARCHAR(50) DEFAULT 'PENDING',
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    accepted_at TIMESTAMP,
    INDEX idx_offered_to (offered_to_bidder_id, offer_status),
    INDEX idx_auction_id (auction_id)
);
```

### MongoDB Collections

```javascript
// Auction Metadata & Analytics
db.createCollection("auction_metadata", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      required: ["auction_id", "created_at"],
      properties: {
        _id: { bsonType: "objectId" },
        auction_id: { bsonType: "long" },
        category: { bsonType: "string" },
        tags: { bsonType: "array" },
        images: { bsonType: "array" },
        condition: { bsonType: "string" },
        shipping_details: { bsonType: "object" },
        created_at: { bsonType: "date" }
      }
    }
  }
});

// Detailed Bid History (Time-series)
db.createCollection("bid_history", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        _id: { bsonType: "objectId" },
        auction_id: { bsonType: "long" },
        bidder_id: { bsonType: "string" },
        max_bid: { bsonType: "decimal" },
        current_bid: { bsonType: "decimal" },
        is_proxy: { bsonType: "bool" },
        timestamp: { bsonType: "date" },
        ip_address: { bsonType: "string" }
      }
    }
  }
});

// Auction Events (Audit Trail)
db.createCollection("auction_events", {
  validator: {
    $jsonSchema: {
      bsonType: "object",
      properties: {
        _id: { bsonType: "objectId" },
        auction_id: { bsonType: "long" },
        event_type: { bsonType: "string" },
        event_data: { bsonType: "object" },
        timestamp: { bsonType: "date" }
      }
    }
  }
});

// Create TTL index to auto-expire old bid history
db.bid_history.createIndex({ timestamp: 1 }, { expireAfterSeconds: 157680000 }); // 5 years
```

---

## Redis Data Structures

```yaml
# Auction State & Live Data
auction:{auction_id}:state
  Type: HASH
  Fields: {current_highest_bidder, current_bid, bid_count, view_count}
  TTL: None
  Usage: Fast read of current auction state

# Auction End Time + Sniping Prevention
auction:{auction_id}:end_time
  Type: STRING
  Value: Unix timestamp (milliseconds)
  TTL: Dynamic (updated on extension)
  Usage: Track when auction ends; trigger closure

# Extension Counter (Sniping Prevention)
auction:{auction_id}:extensions
  Type: STRING
  Value: Integer (current extension count)
  TTL: None
  Usage: Cap sniping extensions at 10 per auction

# Bid Lock (Distributed Lock)
bid:lock:{auction_id}
  Type: STRING
  Value: Lock token (UUID)
  TTL: 5 seconds
  Usage: Ensure single concurrent bid processor

# Proxy Bids Cache
auction:{auction_id}:proxy_bids
  Type: HASH
  Fields: {bidder_id -> max_bid_amount}
  TTL: None
  Usage: Track bidder max bids for proxy execution

# Active Auction Watchers (WebSocket Connections)
auction:{auction_id}:watchers
  Type: SET
  Members: connection_id (WebSocket session ID)
  TTL: None (managed on connect/disconnect)
  Usage: Broadcast bid updates to all watching users

# Bidder's Outbid Notifications Queue
bidder:{bidder_id}:outbid_queue
  Type: LIST
  Members: {auction_id, new_highest_bid, timestamp}
  TTL: 86400 seconds (1 day)
  Usage: Queue outbid notifications for offline users

# Current Highest Bid (Optimized lookup)
auction:{auction_id}:highest_bid
  Type: STRING
  Value: Decimal amount
  TTL: None
  Usage: Fast validation that new bid exceeds current

# Bidder Reputation Cache
bidder:{bidder_id}:reputation
  Type: HASH
  Fields: {auctions_won, non_payment_count, suspension_status}
  TTL: 3600 seconds (1 hour)
  Usage: Quick reputation check during bid validation

# Rate Limit Counters
ratelimit:bid:{user_id}:{minute}
  Type: STRING
  Value: Integer (bid count in current minute)
  TTL: 60 seconds
  Usage: Enforce 100 bids/min per user
```

---

## Kafka Event Flow

```
Topic: auction.created
  Schema: { auction_id, seller_id, title, starting_bid, duration }
  Producers: AuctionService
  Consumers: NotificationService, SearchIndexer
  Partitions: 10 (by auction_id)
  Retention: 365 days

Topic: bid.placed
  Schema: { auction_id, bidder_id, bid_amount, is_proxy, timestamp }
  Producers: BidProcessorService
  Consumers: ProxyBidEngine, NotificationService, AuctionMetricsService
  Partitions: 20 (by auction_id % 20)
  Retention: 365 days (audit trail)

Topic: bid.proxy_executed
  Schema: { auction_id, bidder_id, max_bid, current_price, outbid_bidder }
  Producers: ProxyBidEngine
  Consumers: NotificationService, AuctionService
  Partitions: 10
  Retention: 365 days

Topic: auction.extended
  Schema: { auction_id, extension_count, new_end_time, extended_by_bidder_id }
  Producers: SniperPreventionService
  Consumers: NotificationService, AuctionService
  Partitions: 10
  Retention: 365 days

Topic: auction.closed
  Schema: { auction_id, status, final_highest_bid, winning_bidder_id }
  Producers: AuctionTimerService
  Consumers: WinnerDeterminationService, NotificationService
  Partitions: 1 (single partition for ordered closure)
  Retention: 3650 days (7+ years)

Topic: winner.determined
  Schema: { auction_id, winner_id, final_price, reserve_met }
  Producers: WinnerDeterminationService
  Consumers: PaymentService, NotificationService
  Partitions: 10
  Retention: 3650 days

Topic: payment.captured
  Schema: { auction_id, winner_id, transaction_id, amount, status }
  Producers: PaymentService
  Consumers: AuctionService (update status), NotificationService
  Partitions: 10
  Retention: 3650 days (compliance)

Topic: payment.failed
  Schema: { auction_id, winner_id, failure_reason, retry_count }
  Producers: PaymentService
  Consumers: SecondChanceOfferService, NotificationService
  Partitions: 10
  Retention: 3650 days

Topic: second_chance.offered
  Schema: { auction_id, offered_to_bidder_id, price, expires_at }
  Producers: SecondChanceOfferService
  Consumers: NotificationService, SecondChanceOfferService (expiry handling)
  Partitions: 10
  Retention: 365 days
```

---

## Implementation Code

### 1. AuctionBidService (Concurrent Bid Processing)

```java
package com.auction.service;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.auction.domain.Auction;
import com.auction.domain.Bid;
import com.auction.repository.AuctionRepository;
import com.auction.repository.BidRepository;
import com.auction.event.BidPlacedEvent;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class AuctionBidService {

    private final AuctionRepository auctionRepository;
    private final BidRepository bidRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final ProxyBidEngine proxyBidEngine;
    private final BidValidator bidValidator;

    private static final long LOCK_TIMEOUT_SECONDS = 5;
    private static final String LOCK_PREFIX = "bid:lock:";
    private static final int MAX_RETRIES = 3;
    private static final int BASE_BACKOFF_MS = 10;

    /**
     * Place a bid on an auction with distributed locking.
     * Ensures serialization of concurrent bids on same auction.
     */
    @Transactional
    public BidResult placeBid(Long auctionId, UUID bidderId, BigDecimal bidAmount) {
        String lockKey = LOCK_PREFIX + auctionId;
        String lockToken = UUID.randomUUID().toString();

        for (int attempt = 0; attempt < MAX_RETRIES; attempt++) {
            try {
                // Attempt to acquire distributed lock
                Boolean acquired = redisTemplate.opsForValue().setIfAbsent(
                    lockKey, lockToken, LOCK_TIMEOUT_SECONDS, TimeUnit.SECONDS);

                if (!acquired) {
                    // Exponential backoff: 10ms, 20ms, 40ms
                    long backoffMs = BASE_BACKOFF_MS * (long) Math.pow(2, attempt);
                    Thread.sleep(backoffMs);
                    continue;
                }

                try {
                    // Lock acquired; process bid
                    return processBidWithLock(auctionId, bidderId, bidAmount);
                } finally {
                    // Release lock only if we still own it
                    String currentToken = redisTemplate.opsForValue().get(lockKey);
                    if (lockToken.equals(currentToken)) {
                        redisTemplate.delete(lockKey);
                    }
                }

            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new BidProcessingException("Bid processing interrupted", e);
            }
        }

        throw new BidProcessingException("Failed to acquire lock after " + MAX_RETRIES + " attempts");
    }

    private BidResult processBidWithLock(Long auctionId, UUID bidderId, BigDecimal bidAmount) {
        // Fetch auction under lock
        Auction auction = auctionRepository.findById(auctionId)
            .orElseThrow(() -> new AuctionNotFoundException(auctionId));

        // Validate bid
        BidValidationError validationError = bidValidator.validateBid(auction, bidderId, bidAmount);
        if (validationError != null) {
            return BidResult.failure(validationError);
        }

        // Check if auction is still active
        if (!auction.isActive()) {
            return BidResult.failure(new BidValidationError("AUCTION_CLOSED", "Auction has ended"));
        }

        // Create bid record
        Bid bid = Bid.builder()
            .auctionId(auctionId)
            .bidderId(bidderId)
            .maxBidAmount(bidAmount)
            .currentBidAmount(bidAmount)
            .isProxyBid(false)
            .bidTimestamp(LocalDateTime.now())
            .build();

        // Apply proxy bidding logic
        BigDecimal chargeAmount = proxyBidEngine.executeProxyBidding(auction, bidderId, bidAmount);

        bid.setCurrentBidAmount(chargeAmount);
        bidRepository.save(bid);

        // Update auction state
        auction.setCurrentHighestBid(chargeAmount);
        auction.setCurrentHighestBidderId(bidderId);
        auction.setBidCount(auction.getBidCount() + 1);
        auction.setUpdatedAt(LocalDateTime.now());
        auctionRepository.save(auction);

        // Update Redis cache for fast reads
        updateAuctionStateCache(auctionId, chargeAmount, bidderId);

        // Emit event for downstream processing
        kafkaTemplate.send("bid.placed", auctionId.toString(),
            new BidPlacedEvent(auctionId, bidderId, chargeAmount, LocalDateTime.now()));

        log.info("Bid placed on auction {} by {} for ${}", auctionId, bidderId, chargeAmount);

        return BidResult.success(chargeAmount, bidAmount);
    }

    private void updateAuctionStateCache(Long auctionId, BigDecimal currentBid, UUID bidderId) {
        String stateKey = "auction:" + auctionId + ":state";
        redisTemplate.opsForHash().putAll(stateKey, Map.of(
            "current_highest_bidder", bidderId.toString(),
            "current_bid", currentBid.toString()
        ));
    }
}

@lombok.Data
class BidResult {
    private boolean success;
    private BigDecimal chargeAmount;
    private BigDecimal maxBidAmount;
    private String errorMessage;

    public static BidResult success(BigDecimal chargeAmount, BigDecimal maxBidAmount) {
        BidResult result = new BidResult();
        result.success = true;
        result.chargeAmount = chargeAmount;
        result.maxBidAmount = maxBidAmount;
        return result;
    }

    public static BidResult failure(BidValidationError error) {
        BidResult result = new BidResult();
        result.success = false;
        result.errorMessage = error.getErrorCode() + ": " + error.getErrorMessage();
        return result;
    }
}
```

### 2. ProxyBidEngine

```java
package com.auction.bidding;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import com.auction.domain.Auction;
import com.auction.repository.BidRepository;
import com.auction.event.BidProxyExecutedEvent;
import java.math.BigDecimal;
import java.util.Map;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class ProxyBidEngine {

    private final BidRepository bidRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    private static final BigDecimal INCREMENT = new BigDecimal("0.50");

    /**
     * Execute proxy bidding logic.
     * New bid (from bidder B, amount M) vs existing leading bid (bidder A, amount B₁):
     * - If M > B₁: B becomes leader at B₁ + INCREMENT
     * - A's proxy can auto-bid if A's max >= (B₁ + INCREMENT)
     * - Continue chain until max is exceeded
     */
    public BigDecimal executeProxyBidding(Auction auction, UUID newBidderId, BigDecimal newMaxBid) {
        UUID currentLeaderId = auction.getCurrentHighestBidderId();
        BigDecimal currentLeadingBid = auction.getCurrentHighestBid() != null
            ? auction.getCurrentHighestBid() : auction.getStartingBid();

        // Case 1: New bid does not exceed current leading bid
        if (newMaxBid.compareTo(currentLeadingBid) <= 0) {
            // Outbid current leader
            return currentLeadingBid.add(INCREMENT);
        }

        // Case 2: New bid exceeds current leading bid; new bidder becomes leader
        // Charge: current leading bid + increment (don't charge full max)
        BigDecimal chargeAmount = currentLeadingBid.add(INCREMENT);

        // Case 3: Check if current leader has proxy bid that can auto-counter
        BigDecimal currentLeaderMaxBid = getProxyBidMax(auction.getId(), currentLeaderId);
        if (currentLeaderMaxBid != null && currentLeaderMaxBid.compareTo(chargeAmount) > 0) {
            // Current leader's proxy can auto-bid; new bid becomes leader at chargeAmount
            chargeAmount = currentLeaderMaxBid.add(INCREMENT);
            newBidderId = currentLeaderId;
            // Recursively check if new bidder can be outbid...
        }

        return chargeAmount;
    }

    /**
     * Get bidder's maximum bid amount from proxy storage.
     */
    private BigDecimal getProxyBidMax(Long auctionId, UUID bidderId) {
        String proxyBidsKey = "auction:" + auctionId + ":proxy_bids";
        String maxBidStr = (String) redisTemplate.opsForHash().get(proxyBidsKey, bidderId.toString());
        return maxBidStr != null ? new BigDecimal(maxBidStr) : null;
    }

    /**
     * Store bidder's max proxy bid for future auto-bidding.
     */
    public void storeProxyBid(Long auctionId, UUID bidderId, BigDecimal maxBid) {
        String proxyBidsKey = "auction:" + auctionId + ":proxy_bids";
        redisTemplate.opsForHash().put(proxyBidsKey, bidderId.toString(), maxBid.toString());
    }

    /**
     * Execute proxy auto-bids in chain until max exceeded.
     */
    public void executeProxyChain(Long auctionId, UUID outbiddingBidderId, BigDecimal chargeAmount) {
        // Find all proxy bids above chargeAmount
        String proxyBidsKey = "auction:" + auctionId + ":proxy_bids";
        Map<Object, Object> proxyBids = redisTemplate.opsForHash().entries(proxyBidsKey);

        for (var entry : proxyBids.entrySet()) {
            UUID bidderIdObj = UUID.fromString((String) entry.getKey());
            BigDecimal maxBid = new BigDecimal((String) entry.getValue());

            if (maxBid.compareTo(chargeAmount) > 0 && !bidderIdObj.equals(outbiddingBidderId)) {
                // This bidder can auto-bid
                BigDecimal newChargeAmount = chargeAmount.add(INCREMENT);
                // Emit proxy execution event
                kafkaTemplate.send("bid.proxy_executed", auctionId.toString(),
                    new BidProxyExecutedEvent(auctionId, bidderIdObj, maxBid, newChargeAmount));
            }
        }
    }
}
```

### 3. SniperPreventionService

```java
package com.auction.sniper;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import com.auction.domain.Auction;
import com.auction.repository.AuctionRepository;
import com.auction.repository.AuctionExtensionRepository;
import com.auction.domain.AuctionExtension;
import com.auction.event.AuctionExtendedEvent;
import java.time.LocalDateTime;
import java.util.concurrent.TimeUnit;

@Slf4j
@Service
@RequiredArgsConstructor
public class SniperPreventionService {

    private final AuctionRepository auctionRepository;
    private final AuctionExtensionRepository extensionRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    private static final long EXTENSION_DURATION_MINUTES = 2;
    private static final int MAX_EXTENSIONS = 10;
    private static final long SNIPE_WINDOW_MINUTES = 2;

    /**
     * Check if bid was placed in final 2 minutes; extend auction if so.
     */
    public void checkAndExtendAuction(Long auctionId, LocalDateTime bidTime) {
        Auction auction = auctionRepository.findById(auctionId)
            .orElseThrow(() -> new AuctionNotFoundException(auctionId));

        // Check if bid was placed in final 2 minutes
        long secondsUntilEnd = java.time.temporal.ChronoUnit.SECONDS.between(
            bidTime, auction.getAuctionEndTime());

        if (secondsUntilEnd > 0 && secondsUntilEnd <= (SNIPE_WINDOW_MINUTES * 60)) {
            // Bid within sniping window; check extension count
            String extensionCountKey = "auction:" + auctionId + ":extensions";
            String countStr = redisTemplate.opsForValue().get(extensionCountKey);
            int currentExtensions = countStr != null ? Integer.parseInt(countStr) : 0;

            if (currentExtensions < MAX_EXTENSIONS) {
                extendAuction(auction, currentExtensions);
            } else {
                log.info("Auction {} reached max extensions limit", auctionId);
            }
        }
    }

    private void extendAuction(Auction auction, int currentExtensions) {
        LocalDateTime newEndTime = auction.getAuctionEndTime()
            .plusMinutes(EXTENSION_DURATION_MINUTES);

        // Update auction
        auction.setAuctionEndTime(newEndTime);
        auction.setUpdatedAt(LocalDateTime.now());
        auctionRepository.save(auction);

        // Record extension in DB
        AuctionExtension extension = AuctionExtension.builder()
            .auctionId(auction.getId())
            .extensionCount(currentExtensions + 1)
            .originalEndTime(auction.getAuctionEndTime())
            .extendedEndTime(newEndTime)
            .lastExtensionTime(LocalDateTime.now())
            .build();
        extensionRepository.save(extension);

        // Update Redis counter
        String extensionCountKey = "auction:" + auction.getId() + ":extensions";
        redisTemplate.opsForValue().set(extensionCountKey, String.valueOf(currentExtensions + 1));

        // Update TTL for auction end time key
        String endTimeKey = "auction:" + auction.getId() + ":end_time";
        redisTemplate.opsForValue().set(endTimeKey, String.valueOf(newEndTime.toInstant().toEpochMilli()),
            EXTENSION_DURATION_MINUTES, TimeUnit.MINUTES);

        // Emit event
        kafkaTemplate.send("auction.extended", auction.getId().toString(),
            new AuctionExtendedEvent(auction.getId(), currentExtensions + 1, newEndTime));

        log.info("Auction {} extended to {} (extension #{})", auction.getId(), newEndTime, currentExtensions + 1);
    }
}
```

### 4. BidNotificationService (WebSocket Broadcasting)

```java
package com.auction.notification;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.stereotype.Service;
import com.auction.event.BidPlacedEvent;
import com.auction.event.AuctionExtendedEvent;
import com.fasterxml.jackson.databind.ObjectMapper;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
public class BidNotificationService {

    private final SimpMessagingTemplate messagingTemplate;
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;

    /**
     * Listen for bid placed events and broadcast to all watchers via WebSocket.
     */
    @KafkaListener(topics = "bid.placed", groupId = "notification-service")
    public void onBidPlaced(BidPlacedEvent event) {
        Long auctionId = event.getAuctionId();

        // Fetch all watchers for this auction
        String watchersKey = "auction:" + auctionId + ":watchers";
        var watchers = redisTemplate.opsForSet().members(watchersKey);

        if (watchers == null || watchers.isEmpty()) {
            log.debug("No watchers for auction {}", auctionId);
            return;
        }

        // Construct notification payload
        var notification = Map.of(
            "auctionId", auctionId,
            "bidderId", event.getBidderId().toString(),
            "currentBid", event.getBidAmount().toString(),
            "timestamp", event.getTimestamp().toString()
        );

        // Broadcast to all watchers
        for (Object watcher : watchers) {
            String connectionId = (String) watcher;
            try {
                messagingTemplate.convertAndSendToUser(
                    connectionId,
                    "/queue/auction/" + auctionId + "/bids",
                    notification);
                log.debug("Sent bid notification to watcher {} for auction {}", connectionId, auctionId);
            } catch (Exception e) {
                log.warn("Failed to send notification to {}", connectionId, e);
                // Remove stale watcher
                redisTemplate.opsForSet().remove(watchersKey, watcher);
            }
        }
    }

    /**
     * Notify outbid bidders.
     */
    @KafkaListener(topics = "bid.placed", groupId = "outbid-notification-service")
    public void notifyOutbidBidders(BidPlacedEvent event) {
        // Implementation similar to onBidPlaced
        // Send outbid notification to previous leading bidder
    }

    /**
     * Broadcast auction extension to all watchers.
     */
    @KafkaListener(topics = "auction.extended", groupId = "notification-service")
    public void onAuctionExtended(AuctionExtendedEvent event) {
        String watchersKey = "auction:" + event.getAuctionId() + ":watchers";
        var watchers = redisTemplate.opsForSet().members(watchersKey);

        if (watchers == null) return;

        var notification = Map.of(
            "auctionId", event.getAuctionId(),
            "newEndTime", event.getNewEndTime().toString(),
            "extensionCount", event.getExtensionCount()
        );

        for (Object watcher : watchers) {
            try {
                messagingTemplate.convertAndSendToUser(
                    (String) watcher,
                    "/queue/auction/" + event.getAuctionId() + "/extension",
                    notification);
            } catch (Exception e) {
                log.warn("Failed to send extension notification", e);
            }
        }
    }
}
```

### 5. AuctionPaymentService

```java
package com.auction.payment;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.auction.domain.AuctionWinner;
import com.auction.domain.PaymentMethod;
import com.auction.repository.AuctionWinnerRepository;
import com.auction.repository.PaymentMethodRepository;
import com.auction.event.WinnerDeterminedEvent;
import com.auction.event.PaymentCapturedEvent;
import com.auction.event.PaymentFailedEvent;
import com.payment.gateway.PaymentGatewayAdapter;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;

@Slf4j
@Service
@RequiredArgsConstructor
public class AuctionPaymentService {

    private final AuctionWinnerRepository winnerRepository;
    private final PaymentMethodRepository paymentMethodRepository;
    private final PaymentGatewayAdapter paymentGateway;
    private final KafkaTemplate<String, Object> kafkaTemplate;
    private final RetryQueue paymentRetryQueue;

    private static final int MAX_PAYMENT_RETRIES = 3;
    private static final long RETRY_DELAY_MS = 60000; // 1 minute

    /**
     * Listen for winner determination; attempt to capture payment.
     */
    @KafkaListener(topics = "winner.determined", groupId = "payment-service")
    @Transactional
    public void onWinnerDetermined(WinnerDeterminedEvent event) {
        AuctionWinner winner = AuctionWinner.builder()
            .auctionId(event.getAuctionId())
            .winnerId(event.getWinnerId())
            .finalPrice(event.getFinalPrice())
            .paymentStatus("PENDING")
            .build();
        winnerRepository.save(winner);

        // Attempt payment capture
        captureWinnerPayment(winner);
    }

    /**
     * Capture payment from winner's stored payment method.
     */
    @Transactional
    public void captureWinnerPayment(AuctionWinner winner) {
        PaymentMethod method = paymentMethodRepository.findDefaultByBidderId(winner.getWinnerId())
            .orElseThrow(() -> new PaymentMethodNotFoundException(winner.getWinnerId()));

        try {
            // Attempt payment capture
            PaymentResult result = paymentGateway.capturePayment(
                method.getPaymentToken(),
                winner.getFinalPrice(),
                "Auction " + winner.getAuctionId()
            );

            if (result.isSuccess()) {
                // Mark as paid
                winner.setPaymentStatus("COMPLETED");
                winner.setPaymentTimestamp(LocalDateTime.now());
                winner.setTransactionId(result.getTransactionId());
                winnerRepository.save(winner);

                kafkaTemplate.send("payment.captured", winner.getAuctionId().toString(),
                    new PaymentCapturedEvent(winner.getAuctionId(), winner.getWinnerId(),
                        result.getTransactionId(), winner.getFinalPrice()));

                log.info("Payment captured for auction {} winner {}: ${}",
                    winner.getAuctionId(), winner.getWinnerId(), winner.getFinalPrice());

            } else {
                // Payment failed; queue for retry
                handlePaymentFailure(winner, result.getFailureReason(), 0);
            }

        } catch (Exception e) {
            log.error("Payment capture failed for auction {}", winner.getAuctionId(), e);
            handlePaymentFailure(winner, e.getMessage(), 0);
        }
    }

    private void handlePaymentFailure(AuctionWinner winner, String failureReason, int retryCount) {
        if (retryCount < MAX_PAYMENT_RETRIES) {
            // Queue retry
            paymentRetryQueue.enqueue(new PaymentRetryTask(winner.getAuctionId(),
                winner.getWinnerId(), retryCount + 1), RETRY_DELAY_MS);
            log.warn("Payment capture failed for auction {}; queued retry attempt {}",
                winner.getAuctionId(), retryCount + 1);
        } else {
            // Max retries exceeded; trigger second-chance offer
            winner.setPaymentStatus("FAILED");
            winnerRepository.save(winner);

            kafkaTemplate.send("payment.failed", winner.getAuctionId().toString(),
                new PaymentFailedEvent(winner.getAuctionId(), winner.getWinnerId(),
                    failureReason, retryCount));

            log.error("Payment capture failed for auction {} after {} retries; initiating second-chance",
                winner.getAuctionId(), MAX_PAYMENT_RETRIES);
        }
    }
}
```

### 6. SecondChanceOfferService

```java
package com.auction.recovery;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import com.auction.domain.SecondChanceOffer;
import com.auction.repository.SecondChanceOfferRepository;
import com.auction.repository.BidRepository;
import com.auction.event.PaymentFailedEvent;
import com.auction.event.SecondChanceOfferedEvent;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class SecondChanceOfferService {

    private final SecondChanceOfferRepository offerRepository;
    private final BidRepository bidRepository;
    private final KafkaTemplate<String, Object> kafkaTemplate;

    private static final long OFFER_VALIDITY_HOURS = 24;

    /**
     * Listen for payment failures; offer item to next highest bidder.
     */
    @KafkaListener(topics = "payment.failed", groupId = "second-chance-service")
    @Transactional
    public void onPaymentFailed(PaymentFailedEvent event) {
        // Get next highest bid
        List<Bid> bidders = bidRepository.findTopBiddersForAuction(event.getAuctionId(), 5);

        // Find first bidder who didn't win and doesn't already have an offer
        for (Bid bid : bidders) {
            if (bid.getBidderId().equals(event.getWinnerId())) continue;
            if (offerRepository.existsByAuctionIdAndOfferedToBidderId(event.getAuctionId(), bid.getBidderId())) {
                continue;
            }

            // Create second-chance offer
            LocalDateTime expiresAt = LocalDateTime.now().plusHours(OFFER_VALIDITY_HOURS);
            SecondChanceOffer offer = SecondChanceOffer.builder()
                .auctionId(event.getAuctionId())
                .offeredToBidderId(bid.getBidderId())
                .offeredPrice(bid.getMaxBidAmount())
                .offerStatus("PENDING")
                .createdAt(LocalDateTime.now())
                .expiresAt(expiresAt)
                .build();
            offerRepository.save(offer);

            kafkaTemplate.send("second_chance.offered", event.getAuctionId().toString(),
                new SecondChanceOfferedEvent(event.getAuctionId(), bid.getBidderId(),
                    bid.getMaxBidAmount(), expiresAt));

            log.info("Second-chance offer created for auction {} to bidder {} for ${}",
                event.getAuctionId(), bid.getBidderId(), bid.getMaxBidAmount());
            break;
        }
    }

    /**
     * Expire pending second-chance offers after 24 hours.
     */
    @Scheduled(fixedDelay = 600000) // Every 10 minutes
    public void expirePendingOffers() {
        List<SecondChanceOffer> expiredOffers = offerRepository
            .findByOfferStatusAndExpiresAtBefore("PENDING", LocalDateTime.now());

        for (SecondChanceOffer offer : expiredOffers) {
            offer.setOfferStatus("EXPIRED");
            offerRepository.save(offer);
            log.info("Second-chance offer {} expired", offer.getId());
        }
    }
}

@lombok.Data
@lombok.Builder
class Bid {
    private Long id;
    private Long auctionId;
    private java.util.UUID bidderId;
    private BigDecimal maxBidAmount;
    private BigDecimal currentBidAmount;
}
```

---

## Failure Scenarios

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| Bid lock timeout (Redis down) | Concurrent bids accepted, double-winning risk | Fallback to optimistic locking in PostgreSQL; detect conflicts on write |
| WebSocket connection drops | User doesn't receive bid updates | Client reconnect with last-known bid ID; request delta updates |
| Sniping extension counter corrupted | Excessive extensions allowed | Validate against DB extension count; cap at 10 hard limit |
| Auction closure race (timer fires while bid processing) | Winner not determined correctly | Use auction_end_time as source of truth; process final bids if within 1 sec of closure |
| Payment capture fails (payment gateway down) | Winner not charged; item at risk | Retry with exponential backoff; queue to SQS for async processing; manual intervention after max retries |
| Proxy bid chain infinite loop | Bid amount increases unbounded | Cap proxy execution depth; validate each proxy bid amount increases | increment |
| Kafka broker failure | Events lost; audit trail incomplete | Replication factor 3; event sourcing in PostgreSQL as backup |
| Second-chance offer accepted late | Two offers accepted for same item | Use auction_id as unique key in DB; first-write-wins |

---

## Scaling Strategy

### Horizontal Scaling

**Bid Processing Service:**
- Kafka consumer group with partition-level parallelism
- 20 partitions per auction_id → 20 concurrent bid processors
- Scale to 50-100 instances for 1K bids/sec

**Notification Service:**
- Multiple WebSocket handlers behind load balancer
- Sticky sessions for connection affinity
- Redis Pub/Sub for cross-instance broadcast

**Payment Service:**
- Async processing via SQS/Kafka
- Worker pool for payment retries
- Auto-scaling based on queue depth

### Vertical Scaling

**PostgreSQL:**
- Read replicas for bid history queries
- Partitioning by auction_id (or date range)
- Index: (auction_id, timestamp DESC)
- Archive old auctions (>1 year) to cold storage

**Redis:**
- Replication: master-slave for high availability
- Cluster mode for >100 GB
- Persistence: RDB snapshots every 10 min
- Memory limit: 50 GB; eviction: LRU

**Elasticsearch (Optional, for search):**
- Sharding: 10 shards by auction category
- Refresh interval: 30 seconds (near-real-time search)

### Database Optimization

```sql
-- Index for bid queries
CREATE INDEX idx_auction_bids ON bids(auction_id, bid_timestamp DESC);
CREATE INDEX idx_bidder_auctions ON bids(bidder_id, bid_timestamp DESC);

-- Partition auctions by year
ALTER TABLE auctions PARTITION BY RANGE (YEAR(auction_start_time)) (
    PARTITION p2022 VALUES LESS THAN (2023),
    PARTITION p2023 VALUES LESS THAN (2024),
    PARTITION p2024 VALUES LESS THAN (2025)
);

-- Materialized view for top bids
CREATE MATERIALIZED VIEW top_auction_bids AS
SELECT auction_id, bidder_id, MAX(current_bid_amount) as highest_bid
FROM bids
GROUP BY auction_id, bidder_id;
```

---

## Monitoring & Alerts

### Key Metrics

| Metric | Target | Alert Threshold |
|--------|--------|-----------------|
| Bid latency (p99) | <200 ms | >500 ms |
| WebSocket broadcast latency (p95) | <500 ms | >2000 ms |
| Lock acquisition time (p95) | <100 ms | >500 ms |
| Payment capture success rate | >98% | <95% |
| Auction closure accuracy | 100% | <99.9% |
| Sniping extension avg count | <3 | >5 |
| Second-chance acceptance rate | >30% | <20% |
| Kafka lag (bid topic) | <5 secs | >60 secs |
| Redis memory usage | <50 GB | >70 GB |
| Exception rate (bids) | <0.1% | >1% |

### Alert Rules (Prometheus)

```yaml
groups:
  - name: auction_bidding
    rules:
      - alert: HighBidLatency
        expr: histogram_quantile(0.99, bid_latency_seconds) > 0.5
        for: 5m
        annotations:
          summary: "Bid latency high (p99: {{ $value }}s)"

      - alert: PaymentCaptureFailures
        expr: rate(payment_failures_total[5m]) > 0.02
        for: 10m
        annotations:
          summary: "Payment capture failure rate elevated"

      - alert: WebSocketBroadcastBacklog
        expr: websocket_broadcast_queue_depth > 1000
        for: 5m
        annotations:
          summary: "WebSocket broadcast queue backlog detected"

      - alert: AuctionClosureDelay
        expr: histogram_quantile(0.95, auction_closure_latency_seconds) > 5
        for: 5m
        annotations:
          summary: "Auction closure delayed beyond expected time"
```

---

## Summary Cheat Sheet

### Architecture Components
- **Auction Service**: CRUD, lifecycle management, timer
- **Bid Processing**: Distributed lock per auction, serial processing
- **Proxy Bidding**: Auto-bid up to max, charge second-highest + increment
- **Sniping Prevention**: Extend by 2 min if bid in final 2 min, cap 10 extensions
- **WebSocket Notification**: Real-time push to watching bidders
- **Payment Service**: Capture on winner determination, retry with backoff
- **Second Chance**: Offer to next bidder if winner doesn't pay

### Key Technologies
- **PostgreSQL**: Auctions, bids, winners, payment methods (OLTP)
- **MongoDB**: Bid history, auction metadata, events (time-series)
- **Redis**: Auction state cache, proxy bids, locks, watchers, auction timers
- **Kafka**: Bid events, payment events, notifications (13 topics)
- **WebSocket**: Real-time bid updates to clients
- **Payment Gateway**: Stripe/Square/PayPal integration

### Concurrency Control
```
Lock Model: SET NX EX (atomic)
Retry: exponential backoff (10ms, 20ms, 40ms)
Fallback: optimistic locking in PostgreSQL
Result: linearized bid order, no race conditions
```

### Proxy Bidding Algorithm
```
Current leader: A, bid B₁
New bid: B, max M
if M <= B₁: reject as outbid
if M > B₁:
  charge_amount = B₁ + INCREMENT
  new_leader = B
  if A.max > charge_amount: A auto-bids
    recurse with A as new_leader
```

### Performance Targets
- 100 concurrent auctions, 10K active bidders
- 1K bids/sec peak capacity
- Bid latency p99: <200 ms
- WebSocket broadcast p95: <500 ms
- Payment success: >98%

### Data Retention
- Bid history: 5 years (dispute resolution)
- Auction events (Kafka): 3650 days
- Auction closures: immutable log

