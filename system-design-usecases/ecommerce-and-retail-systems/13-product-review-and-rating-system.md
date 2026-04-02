---
title: Product Review and Rating System
layout: default
---

# Product Review and Rating System — Deep Dive Design

## Scenario

Design a product review and rating system for an e-commerce platform where only verified purchasers can post reviews with ratings (1-5 stars), text, photos, and videos. The system must support helpful votes, flag inappropriate reviews for moderation, calculate real-time aggregate ratings with distributions, support sorting by recent/helpful/rating, and handle 1 million reviews per day at peak.

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
- Only verified purchasers can post reviews (verified via order history)
- Reviews include: rating (1-5), text (max 5000 chars), photos (max 10), videos (max 3)
- Each customer can review a product at most once (prevent duplicates)
- Helpful votes: upvote/downvote reviews; prevent duplicate votes per customer
- Flag reviews for moderation: inappropriate content, spam, fake reviews
- Moderation queue: flagged reviews routed to human reviewers
- Aggregate ratings: average rating, distribution (1-star %, 2-star %, etc.), count
- Sort reviews by: most recent, most helpful (votes), highest/lowest rating
- Search reviews by: product, rating, keyword, date range
- Real-time aggregate stats updated as new reviews come in

### Non-Functional Requirements
- **Throughput:** 1M reviews/day = ~12 reviews/second
- **Peak throughput:** 50 reviews/second (peak hours)
- **Read latency:** aggregate ratings <100ms, review list <200ms
- **Write latency:** post review <500ms
- **Availability:** 99.95% SLA (reviews are not critical path)
- **Data durability:** all reviews persisted; eventual consistency acceptable for aggregates
- **Audit:** all reviews immutable; deletions soft-delete

---

## 2. Capacity Estimation

```
Reviews Per Day: 1,000,000
Reviews Per Second (avg): ~12
Reviews Per Second (peak): ~50
Helpful Votes Per Day: ~5,000,000 (5 votes per review on average)
Helpful Votes Per Second: ~58

MongoDB Reviews Storage (1 year):
365 days × 1M reviews/day × 2 KB/review = 730 GB

PostgreSQL Ratings Aggregation:
10,000 products × 1000 average reviews = 10M ratings = ~500 MB

Redis Cache:
10,000 products × 500 bytes (ratings data) = 5 MB
1,000 recent reviews per product × 10K products × 500 bytes = 5 GB

Kafka Throughput:
- reviews.created: 50/sec peak
- reviews.flagged: ~1/sec (2% flag rate)
- reviews.helpful_vote: 290/sec peak (5 votes per review avg)

Daily API Calls:
- POST /products/{id}/reviews: 1M calls
- GET /products/{id}/reviews: 50M calls (discovery, feeds, comparisons)
- PATCH /reviews/{id}/helpful: 5M calls
```

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────┐
│           Customer Web/Mobile                       │
└──────────────┬──────────────────────────────────────┘
               │ REST APIs
        ┌──────┴────────────────────┐
        │                           │
┌───────▼──────────────┐  ┌────────▼──────────┐
│ Review API Service   │  │ Helpful Vote      │
│ (create, list,       │  │ Service           │
│  search, filter)     │  │ (upvote/downvote) │
└───────┬──────────────┘  └─────────┬──────────┘
        │                           │
        │ Events                    │ Events
        ▼                           ▼
┌──────────────────────────────────────────────────────┐
│         Kafka Topics                                 │
│  - reviews.created → Review Processor                │
│  - reviews.flagged → Moderation Service              │
│  - reviews.helpful_vote → Rating Aggregator          │
└──────────┬──────────────────────────────────────────┘
           │
    ┌──────┴─────────┬──────────────────┬──────────────┐
    │                │                  │              │
┌───▼────────┐  ┌───▼──────────┐  ┌───▼──────┐  ┌──▼────────────┐
│ Review     │  │ Rating       │  │ Fake     │  │ Moderation    │
│ Processor  │  │ Aggregator   │  │ Review   │  │ Dashboard     │
│ (verify    │  │ (compute avg,│  │ Detector │  │ (human review)│
│ purchase & │  │ distribution)│  │ (ML)     │  │               │
│ index)     │  │              │  │          │  │               │
└────────────┘  └──────────────┘  └──────────┘  └────────────────┘
                        │
                  ┌─────▼──────────────┐
                  │ Redis Cache        │
                  │ - ratings:{id}     │
                  │ - reviews cache    │
                  └────────────────────┘

        ┌──────────────────────────────────┐
        │ MongoDB                          │
        │ - reviews (flexible schema)      │
        │ - reviews_metadata               │
        │ - helpful_votes                  │
        │ - flagged_reviews                │
        └──────────────────────────────────┘

        ┌──────────────────────────────────┐
        │ PostgreSQL                       │
        │ - rating_aggregates (denorm)     │
        │ - product_ratings (cache)        │
        │ - helpful_vote_aggregate (count) │
        └──────────────────────────────────┘
```

---

## 4. Core Design Questions Answered

### Q1: How do you verify that reviewer actually purchased the product?

**Answer:** When a customer submits a review, the `ReviewService` queries the `Order` service (via REST or event DB cache) to check if the customer has a completed order with that product SKU. Store the verified `order_id` with the review. Implement this as an async pre-check: validate on review submission and block if not verified. Cache purchase eligibility in Redis (ttl 1 day) keyed by `customer_id:product_id` to reduce order service load. If order service is unavailable, allow submission but mark as `PENDING_VERIFICATION` and verify later from cached event stream.

### Q2: How do you store and serve reviews efficiently?

**Answer:** Store reviews in MongoDB with flexible schema supporting text, arrays (photos), embedded objects (metadata). Index on `product_id`, `created_at`, `rating` for common queries. Use pagination (cursor-based or offset) to serve reviews. For hot products with high review velocity, implement a read-through cache in Redis: store top 100 recent reviews per product (TTL 1 hour). Use Elasticsearch or similar for full-text search on review text and comments (optional, if search is critical path).

### Q3: How do you calculate aggregate ratings in real-time?

**Answer:** Maintain two data structures: (1) **PostgreSQL denormalized table** `product_ratings` with columns for `average_rating`, `total_count`, `one_star_count`, `two_star_count`, ... `five_star_count`. Update this table via Kafka consumer (`RatingAggregator`) on each `reviews.created` event using atomic SQL update. (2) **Redis cache** `ratings:{product_id}` with same data, updated synchronously in the Kafka consumer. This allows <100ms latency for reads. Re-compute from scratch daily during off-peak to correct any inconsistencies.

### Q4: How do you implement "helpful votes" with high read/write volume?

**Answer:** Use Redis counter: `review:{review_id}:helpful_votes` for fast increments (O(1)). Store actual vote records in MongoDB for audit trail with unique index `(review_id, customer_id)` to prevent duplicate votes. Kafka topic `reviews.helpful_vote` publishes vote events asynchronously. Consume these events to update both Redis counters and PostgreSQL aggregate counts. Eventual consistency acceptable: Redis shows latest, DB updated within seconds. Use sorted set to rank reviews by helpfulness: `product:{product_id}:reviews:by_helpful` with score = helpful_votes.

### Q5: How do you detect and prevent fake reviews?

**Answer:** Implement ML-based `FakeReviewDetector` service with signals: (1) account age (new accounts more likely fake), (2) review velocity (multiple reviews in short time), (3) IP clustering (many accounts from same IP), (4) linguistic analysis (bot-like language), (5) rating inconsistency (accounts with 5-star reviews only), (6) verified purchase ratio. Each review gets a fake_score (0-100). Reviews with score >75 auto-flagged for moderation. Use Kafka to stream review events to ML pipeline for scoring. Store scores in MongoDB for historical analysis.

### Q6: How do you moderate reviews at scale?

**Answer:** Implement moderation queue: flagged reviews published to Kafka topic `reviews.flagged` with UUID, reason, fake_score. Moderation service consumes events and exposes REST API + dashboard for human reviewers. Dashboard shows: review text, photos, videos, customer history, similar reviews, aggregate flags. Reviewer can approve (publish), reject (delete), or request changes. Rejected reviews marked as deleted (soft-delete) with reason. Implement approval SLA: 80% within 24h. Auto-publish reviews with low flag counts after 48h if not reviewed. Escalate high-severity cases (fraud patterns) to legal/security team.

---

## 5. Microservices Breakdown

| Service | Responsibility | Tech | Scaling |
|---------|---|---|---|
| Review API | Create, read, list, search reviews | Spring Boot REST | Horizontal, stateless |
| Helpful Vote Service | Track helpful/unhelpful votes | Spring Boot | Horizontal, stateless |
| Rating Aggregator | Compute aggregate ratings | Kafka Consumer | 3-5 instances, order guarantee |
| Fake Review Detector | ML-based fraud detection | ML Pipeline | Batch + streaming |
| Moderation Service | Queue & manage flagged reviews | Spring Boot REST | Horizontal, stateless |
| Purchase Verifier | Verify customer purchase history | Spring Boot | Horizontal, cached |
| Review Indexer | Index reviews in Elasticsearch | Kafka Consumer | Horizontal, auto-scale |
| Analytics | Compute review trends, insights | Spark/Analytics Engine | Batch daily |

---

## 6. Database Design

### PostgreSQL Schema

```sql
CREATE TABLE product_ratings (
    product_id UUID PRIMARY KEY,
    total_reviews INT NOT NULL DEFAULT 0,
    average_rating DECIMAL(3,2) NOT NULL DEFAULT 0,
    one_star_count INT DEFAULT 0,
    two_star_count INT DEFAULT 0,
    three_star_count INT DEFAULT 0,
    four_star_count INT DEFAULT 0,
    five_star_count INT DEFAULT 0,
    helpful_votes_total INT DEFAULT 0,
    last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_average_rating (average_rating)
);

CREATE TABLE helpful_votes_aggregate (
    review_id UUID NOT NULL,
    helpful_count INT DEFAULT 0,
    unhelpful_count INT DEFAULT 0,
    net_helpful INT DEFAULT 0,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    PRIMARY KEY (review_id),
    INDEX idx_net_helpful (net_helpful)
);

CREATE TABLE reviews_audit (
    id UUID PRIMARY KEY,
    review_id UUID NOT NULL,
    action VARCHAR(20) NOT NULL CHECK (action IN ('CREATED', 'EDITED', 'FLAGGED', 'DELETED', 'APPROVED')),
    actor_id UUID,
    reason VARCHAR(500),
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_review_id (review_id),
    INDEX idx_timestamp (timestamp)
);
```

### MongoDB Collections

```json
// reviews collection
{
  "_id": ObjectId,
  "review_id": UUID,
  "product_id": UUID,
  "customer_id": UUID,
  "order_id": UUID,           // Verified purchase
  "rating": 4,                // 1-5
  "title": "Great product!",
  "text": "This is a wonderful product. Highly recommend to everyone...",
  "photos": [
    {
      "id": UUID,
      "url": "s3://...",
      "caption": "Product unboxing"
    }
  ],
  "videos": [
    {
      "id": UUID,
      "url": "youtube/...",
      "duration_secs": 60
    }
  ],
  "verified_purchase": true,
  "fake_score": 5,            // ML-based fraud detection (0-100)
  "is_flagged": false,
  "flag_reason": null,
  "moderation_status": "APPROVED",  // PENDING, APPROVED, REJECTED
  "helpful_votes": {
    "helpful": 245,
    "unhelpful": 12,
    "net": 233
  },
  "created_at": ISODate,
  "updated_at": ISODate,
  "deleted_at": null,
  "deleted_reason": null,
  "version": 1
}

// helpful_votes collection
{
  "_id": ObjectId,
  "review_id": UUID,
  "customer_id": UUID,
  "vote_type": "HELPFUL" | "UNHELPFUL",
  "created_at": ISODate,

  "UNIQUE INDEX": (review_id, customer_id)
}

// flagged_reviews collection
{
  "_id": ObjectId,
  "review_id": UUID,
  "product_id": UUID,
  "flag_reason": "SPAM" | "INAPPROPRIATE" | "FAKE" | "DUPLICATE" | "OTHER",
  "user_flag_count": 3,       // How many users flagged
  "fake_score": 78,
  "priority": "HIGH" | "MEDIUM" | "LOW",
  "status": "PENDING" | "REVIEWED" | "APPROVED" | "REJECTED",
  "reviewer_id": UUID,
  "reviewer_comment": "Clearly spam account",
  "created_at": ISODate,
  "reviewed_at": ISODate,
  "resolution_at": ISODate
}

// review_metadata collection (denormalized for analytics)
{
  "_id": ObjectId,
  "review_id": UUID,
  "product_id": UUID,
  "product_name": "Blue T-Shirt",
  "category": "Apparel",
  "customer_id": UUID,
  "customer_age_group": "25-34",
  "rating": 4,
  "helpful_votes": 233,
  "comment_count": 12,
  "created_at": ISODate,
  "text_length": 250,
  "has_photos": true,
  "has_videos": false,
  "verified_purchase": true
}
```

---

## 7. Redis Data Structures

```
# Product ratings cache (TTL 1 hour, updated on every review)
ratings:{product_id} → Hash {
    "average_rating": "4.25",
    "total_reviews": 1523,
    "one_star_count": "45",
    "two_star_count": "89",
    "three_star_count": "234",
    "four_star_count": "567",
    "five_star_count": "588",
    "helpful_votes_total": "3567",
    "last_updated": "2026-04-01T10:30:00Z"
}

# Review helpful votes counter (ephemeral)
review:{review_id}:helpful_votes → Integer (incremented on each vote)

# Recent reviews by product (TTL 1 hour)
product:{product_id}:reviews:recent → Sorted Set {
    score = created_at_timestamp,
    member = review_id
} → Max 100 members

# Reviews ranked by helpfulness (TTL 1 hour)
product:{product_id}:reviews:helpful → Sorted Set {
    score = net_helpful_votes,
    member = review_id
} → Max 100 members

# Customer purchase verification cache (TTL 24 hours)
customer:{customer_id}:product:{product_id}:purchased → "true"

# Helpful vote tracking per customer (TTL 30 days)
customer:{customer_id}:voted_reviews → Set [review_id1, review_id2, ...]

# Moderation queue length (real-time)
moderation:queue:length → Integer

# Rate limiting - reviews per customer (TTL 24 hours)
customer:{customer_id}:reviews_posted_today → Integer
```

---

## 8. Kafka Event Flow

### Topics & Events

```
Topic: reviews (Retention: 90 days)
├─ review.created
│  └─ {review_id, product_id, customer_id, order_id, rating, text, photos, videos}
├─ review.flagged
│  └─ {review_id, flag_reason, flagged_by_user_count}
├─ review.moderated
│  └─ {review_id, moderation_status, reviewer_id, reviewer_comment}
└─ review.deleted
   └─ {review_id, delete_reason, deleted_by}

Topic: helpful_votes (Retention: 30 days)
├─ helpful_vote.recorded
│  └─ {review_id, customer_id, vote_type: HELPFUL|UNHELPFUL}
├─ helpful_vote.withdrawn
│  └─ {review_id, customer_id}
└─ [consumed by Rating Aggregator, vote counting service]

Topic: ratings (Retention: 30 days, compacted)
├─ product_rating.updated
│  └─ {product_id, average_rating, total_reviews, distribution}
└─ [consumed by caching layer, analytics]

Topic: moderation (Retention: 90 days)
├─ moderation.flagged
│  └─ {review_id, flag_reason, fake_score, priority}
├─ moderation.approved
│  └─ {review_id, reviewer_id}
└─ moderation.rejected
   └─ {review_id, reviewer_id, rejection_reason}

Topic: fraud (Retention: 90 days)
└─ fraud_detection.score_updated
   └─ {review_id, fake_score, signals: {account_age, review_velocity, ip_cluster, ...}}
```

---

## 9. Implementation Code

### 9.1 Review Entity & Repository

```java
package com.retail.review.entity;

import org.springframework.data.annotation.Id;
import org.springframework.data.mongodb.core.mapping.Document;
import org.springframework.data.mongodb.core.index.Indexed;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Document(collection = "reviews")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Review {

    @Id
    private String mongoId;

    @Indexed
    private UUID reviewId;

    @Indexed
    private UUID productId;

    @Indexed
    private UUID customerId;

    private UUID orderId;

    @Indexed
    private Integer rating; // 1-5

    private String title;

    private String text;

    private List<Photo> photos;

    private List<Video> videos;

    private boolean verifiedPurchase;

    private Integer fakeScore; // 0-100, higher = more likely fake

    private boolean isFlagged;

    private String flagReason;

    private ModerationStatus moderationStatus;

    private HelpfulVotesInfo helpfulVotes;

    @Indexed
    private LocalDateTime createdAt;

    private LocalDateTime updatedAt;

    private LocalDateTime deletedAt;

    private String deletedReason;

    private Integer version;

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class Photo {
        private UUID id;
        private String url;
        private String caption;
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class Video {
        private UUID id;
        private String url;
        private Integer durationSecs;
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    @Builder
    public static class HelpfulVotesInfo {
        private Integer helpful;
        private Integer unhelpful;
        private Integer net;
    }
}

public enum ModerationStatus {
    PENDING, APPROVED, REJECTED
}
```

### 9.2 Review Service with Purchase Verification

```java
package com.retail.review.service;

import com.retail.review.entity.Review;
import com.retail.review.event.ReviewEvent;
import com.retail.review.repository.ReviewRepository;
import com.retail.review.client.OrderServiceClient;
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class ReviewService {

    private final ReviewRepository reviewRepository;
    private final OrderServiceClient orderServiceClient;
    private final KafkaTemplate<String, ReviewEvent> kafkaTemplate;
    private final RedisTemplate<String, String> redisTemplate;
    private final MeterRegistry meterRegistry;
    private final FakeReviewDetector fakeReviewDetector;

    public ReviewService(
        ReviewRepository reviewRepository,
        OrderServiceClient orderServiceClient,
        KafkaTemplate<String, ReviewEvent> kafkaTemplate,
        RedisTemplate<String, String> redisTemplate,
        MeterRegistry meterRegistry,
        FakeReviewDetector fakeReviewDetector
    ) {
        this.reviewRepository = reviewRepository;
        this.orderServiceClient = orderServiceClient;
        this.kafkaTemplate = kafkaTemplate;
        this.redisTemplate = redisTemplate;
        this.meterRegistry = meterRegistry;
        this.fakeReviewDetector = fakeReviewDetector;
    }

    public Review createReview(CreateReviewRequest request, UUID customerId) throws Exception {
        // Step 1: Verify customer has already reviewed this product
        long existingReviewCount = reviewRepository.countByProductIdAndCustomerIdAndDeletedAtIsNull(
            request.productId(), customerId
        );

        if (existingReviewCount > 0) {
            throw new DuplicateReviewException("Customer already reviewed this product");
        }

        // Step 2: Verify purchase - check cache first
        String purchaseCacheKey = "customer:" + customerId + ":product:" + request.productId() + ":purchased";
        String cached = redisTemplate.opsForValue().get(purchaseCacheKey);

        UUID orderId = null;
        boolean verifiedPurchase = false;

        if (cached != null && cached.equals("true")) {
            verifiedPurchase = true;
        } else {
            // Verify with Order Service
            try {
                OrderVerificationResponse orderVerification = orderServiceClient.verifyPurchase(
                    customerId, request.productId()
                );

                if (orderVerification.isPurchased()) {
                    verifiedPurchase = true;
                    orderId = orderVerification.getOrderId();
                    // Cache for 24 hours
                    redisTemplate.opsForValue().set(
                        purchaseCacheKey, "true", 24, TimeUnit.HOURS
                    );
                }
            } catch (Exception e) {
                log.warn("Order service call failed, marking review as PENDING_VERIFICATION", e);
                // Allow submission but mark as pending verification
            }
        }

        if (!verifiedPurchase) {
            throw new UnverifiedPurchaseException("Customer has not purchased this product");
        }

        // Step 3: Create review document
        Review review = Review.builder()
            .reviewId(UUID.randomUUID())
            .productId(request.productId())
            .customerId(customerId)
            .orderId(orderId)
            .rating(request.rating())
            .title(request.title())
            .text(request.text())
            .photos(request.photos())
            .videos(request.videos())
            .verifiedPurchase(verifiedPurchase)
            .fakeScore(0) // Will be computed asynchronously
            .isFlagged(false)
            .moderationStatus(ModerationStatus.PENDING)
            .helpfulVotes(new Review.HelpfulVotesInfo(0, 0, 0))
            .createdAt(LocalDateTime.now())
            .updatedAt(LocalDateTime.now())
            .version(1)
            .build();

        review = reviewRepository.save(review);

        // Step 4: Publish event for async processing
        ReviewEvent event = ReviewEvent.created(review);
        kafkaTemplate.send("reviews", review.getReviewId().toString(), event);

        // Step 5: Queue for fraud detection
        fakeReviewDetector.analyzeAsync(review);

        // Metrics
        Counter.builder("review.created")
            .tag("product_id", request.productId().toString())
            .tag("rating", String.valueOf(request.rating()))
            .register(meterRegistry)
            .increment();

        log.info("Review created: id={}, product={}, customer={}",
            review.getReviewId(), request.productId(), customerId);

        return review;
    }

    public Review getReview(UUID reviewId) {
        return reviewRepository.findByReviewId(reviewId)
            .orElseThrow(() -> new ReviewNotFoundException(reviewId));
    }

    public void flagReview(UUID reviewId, String reason) {
        Review review = getReview(reviewId);

        review.setFlagged(true);
        review.setFlagReason(reason);
        review.setUpdatedAt(LocalDateTime.now());
        reviewRepository.save(review);

        ReviewEvent event = ReviewEvent.flagged(reviewId, reason);
        kafkaTemplate.send("reviews", reviewId.toString(), event);

        Counter.builder("review.flagged")
            .tag("reason", reason)
            .register(meterRegistry)
            .increment();

        log.info("Review flagged: id={}, reason={}", reviewId, reason);
    }

    public void approveReview(UUID reviewId) {
        Review review = getReview(reviewId);

        review.setModerationStatus(ModerationStatus.APPROVED);
        review.setUpdatedAt(LocalDateTime.now());
        reviewRepository.save(review);

        ReviewEvent event = ReviewEvent.moderated(reviewId, ModerationStatus.APPROVED);
        kafkaTemplate.send("reviews", reviewId.toString(), event);

        // Trigger rating aggregation
        kafkaTemplate.send("ratings", review.getProductId().toString(),
            new RatingUpdateEvent(review.getProductId()));

        Counter.builder("review.approved").register(meterRegistry).increment();
        log.info("Review approved: id={}", reviewId);
    }

    public void rejectReview(UUID reviewId, String reason) {
        Review review = getReview(reviewId);

        review.setModerationStatus(ModerationStatus.REJECTED);
        review.setDeletedAt(LocalDateTime.now());
        review.setDeletedReason(reason);
        review.setUpdatedAt(LocalDateTime.now());
        reviewRepository.save(review);

        ReviewEvent event = ReviewEvent.deleted(reviewId, reason);
        kafkaTemplate.send("reviews", reviewId.toString(), event);

        Counter.builder("review.rejected").register(meterRegistry).increment();
        log.info("Review rejected: id={}, reason={}", reviewId, reason);
    }
}

public interface ReviewRepository extends MongoRepository<Review, String> {
    Review findByReviewId(UUID reviewId);
    long countByProductIdAndCustomerIdAndDeletedAtIsNull(UUID productId, UUID customerId);
    List<Review> findByProductIdAndDeletedAtIsNullOrderByCreatedAtDesc(
        UUID productId, Pageable pageable
    );
}
```

### 9.3 Helpful Votes Service

```java
package com.retail.review.service;

import com.retail.review.entity.Review;
import com.retail.review.entity.HelpfulVote;
import com.retail.review.event.HelpfulVoteEvent;
import com.retail.review.repository.HelpfulVoteRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.Optional;
import java.util.UUID;

@Service
@Slf4j
public class HelpfulVoteService {

    private final HelpfulVoteRepository helpfulVoteRepository;
    private final ReviewService reviewService;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, HelpfulVoteEvent> kafkaTemplate;

    public HelpfulVoteService(
        HelpfulVoteRepository helpfulVoteRepository,
        ReviewService reviewService,
        RedisTemplate<String, String> redisTemplate,
        KafkaTemplate<String, HelpfulVoteEvent> kafkaTemplate
    ) {
        this.helpfulVoteRepository = helpfulVoteRepository;
        this.reviewService = reviewService;
        this.redisTemplate = redisTemplate;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Transactional
    public void voteHelpful(UUID reviewId, UUID customerId, VoteType voteType) {
        Review review = reviewService.getReview(reviewId);

        // Check if customer already voted
        Optional<HelpfulVote> existingVote = helpfulVoteRepository
            .findByReviewIdAndCustomerId(reviewId, customerId);

        if (existingVote.isPresent()) {
            HelpfulVote vote = existingVote.get();
            // Allow vote change: from HELPFUL to UNHELPFUL or vice versa
            if (vote.getVoteType() == voteType) {
                // Same vote, withdraw it
                withdrawVote(vote);
                return;
            } else {
                // Change vote
                vote.setVoteType(voteType);
                vote.setUpdatedAt(LocalDateTime.now());
                helpfulVoteRepository.save(vote);
            }
        } else {
            // New vote
            HelpfulVote vote = HelpfulVote.builder()
                .reviewId(reviewId)
                .customerId(customerId)
                .voteType(voteType)
                .createdAt(LocalDateTime.now())
                .build();
            helpfulVoteRepository.save(vote);
        }

        // Update Redis counter (atomic)
        String redisKey = "review:" + reviewId + ":helpful_votes";
        redisTemplate.opsForValue().increment(redisKey);

        // Cache customer's voted reviews for deduplication
        String customerVoteKey = "customer:" + customerId + ":voted_reviews";
        redisTemplate.opsForSet().add(customerVoteKey, reviewId.toString());
        redisTemplate.expire(customerVoteKey, 30, java.util.concurrent.TimeUnit.DAYS);

        // Publish event
        HelpfulVoteEvent event = HelpfulVoteEvent.recorded(reviewId, customerId, voteType);
        kafkaTemplate.send("helpful_votes", reviewId.toString(), event);

        log.info("Helpful vote recorded: review={}, customer={}, vote={}",
            reviewId, customerId, voteType);
    }

    private void withdrawVote(HelpfulVote vote) {
        helpfulVoteRepository.delete(vote);

        String redisKey = "review:" + vote.getReviewId() + ":helpful_votes";
        redisTemplate.opsForValue().decrement(redisKey);

        HelpfulVoteEvent event = HelpfulVoteEvent.withdrawn(
            vote.getReviewId(), vote.getCustomerId()
        );
        kafkaTemplate.send("helpful_votes", vote.getReviewId().toString(), event);

        log.info("Helpful vote withdrawn: review={}, customer={}",
            vote.getReviewId(), vote.getCustomerId());
    }

    public Integer getHelpfulVoteCount(UUID reviewId) {
        String redisKey = "review:" + reviewId + ":helpful_votes";
        String count = redisTemplate.opsForValue().get(redisKey);
        if (count != null) {
            return Integer.parseInt(count);
        }
        // Fallback to database
        return helpfulVoteRepository.countByReviewIdAndVoteType(reviewId, VoteType.HELPFUL);
    }
}

public enum VoteType {
    HELPFUL, UNHELPFUL
}
```

### 9.4 Rating Aggregator (Kafka Consumer)

```java
package com.retail.review.aggregation;

import com.retail.review.entity.ProductRating;
import com.retail.review.event.ReviewEvent;
import com.retail.review.repository.ReviewRepository;
import com.retail.review.repository.ProductRatingRepository;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class RatingAggregatorService {

    private final ReviewRepository reviewRepository;
    private final ProductRatingRepository productRatingRepository;
    private final JdbcTemplate jdbcTemplate;
    private final RedisTemplate<String, String> redisTemplate;

    public RatingAggregatorService(
        ReviewRepository reviewRepository,
        ProductRatingRepository productRatingRepository,
        JdbcTemplate jdbcTemplate,
        RedisTemplate<String, String> redisTemplate
    ) {
        this.reviewRepository = reviewRepository;
        this.productRatingRepository = productRatingRepository;
        this.jdbcTemplate = jdbcTemplate;
        this.redisTemplate = redisTemplate;
    }

    @KafkaListener(topics = "reviews", groupId = "rating-aggregator-group")
    public void processReviewEvent(ReviewEvent event) {
        if (event.getEventType() == ReviewEventType.CREATED
            || event.getEventType() == ReviewEventType.MODERATED) {

            log.info("Aggregating ratings for product: {}", event.getProductId());
            aggregateRatings(event.getProductId());
        }
    }

    public void aggregateRatings(UUID productId) {
        // Query all approved reviews for product
        List<Review> reviews = reviewRepository
            .findByProductIdAndModerationStatusAndDeletedAtIsNull(
                productId, ModerationStatus.APPROVED
            );

        int totalReviews = reviews.size();

        if (totalReviews == 0) {
            return; // No reviews yet
        }

        // Calculate distribution
        Map<Integer, Long> distribution = reviews.stream()
            .collect(Collectors.groupingBy(
                Review::getRating,
                Collectors.counting()
            ));

        // Calculate average
        double averageRating = reviews.stream()
            .mapToInt(Review::getRating)
            .average()
            .orElse(0.0);

        // Update PostgreSQL (atomic)
        String updateSql = """
            INSERT INTO product_ratings
            (product_id, total_reviews, average_rating, one_star_count, two_star_count,
             three_star_count, four_star_count, five_star_count, updated_at)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, CURRENT_TIMESTAMP)
            ON CONFLICT (product_id) DO UPDATE SET
                total_reviews = ?,
                average_rating = ?,
                one_star_count = ?,
                two_star_count = ?,
                three_star_count = ?,
                four_star_count = ?,
                five_star_count = ?,
                updated_at = CURRENT_TIMESTAMP
            """;

        jdbcTemplate.update(updateSql,
            productId, totalReviews, averageRating,
            distribution.getOrDefault(1, 0L),
            distribution.getOrDefault(2, 0L),
            distribution.getOrDefault(3, 0L),
            distribution.getOrDefault(4, 0L),
            distribution.getOrDefault(5, 0L),
            totalReviews, averageRating,
            distribution.getOrDefault(1, 0L),
            distribution.getOrDefault(2, 0L),
            distribution.getOrDefault(3, 0L),
            distribution.getOrDefault(4, 0L),
            distribution.getOrDefault(5, 0L)
        );

        // Update Redis cache (TTL 1 hour)
        String redisKey = "ratings:" + productId;
        redisTemplate.opsForHash().putAll(redisKey, Map.ofEntries(
            Map.entry("average_rating", String.format("%.2f", averageRating)),
            Map.entry("total_reviews", String.valueOf(totalReviews)),
            Map.entry("one_star_count", String.valueOf(distribution.getOrDefault(1, 0L))),
            Map.entry("two_star_count", String.valueOf(distribution.getOrDefault(2, 0L))),
            Map.entry("three_star_count", String.valueOf(distribution.getOrDefault(3, 0L))),
            Map.entry("four_star_count", String.valueOf(distribution.getOrDefault(4, 0L))),
            Map.entry("five_star_count", String.valueOf(distribution.getOrDefault(5, 0L))),
            Map.entry("last_updated", LocalDateTime.now().toString())
        ));
        redisTemplate.expire(redisKey, 1, TimeUnit.HOURS);

        log.info("Ratings aggregated: product={}, avgRating={}, totalReviews={}",
            productId, averageRating, totalReviews);
    }
}
```

### 9.5 Fake Review Detector (ML-Based)

```java
package com.retail.review.fraud;

import com.retail.review.entity.Review;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;

@Service
@Slf4j
public class FakeReviewDetector {

    private final ReviewRepository reviewRepository;
    private final CustomerService customerService;
    private final KafkaTemplate<String, FraudDetectionEvent> kafkaTemplate;

    public FakeReviewDetector(
        ReviewRepository reviewRepository,
        CustomerService customerService,
        KafkaTemplate<String, FraudDetectionEvent> kafkaTemplate
    ) {
        this.reviewRepository = reviewRepository;
        this.customerService = customerService;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Async
    public void analyzeAsync(Review review) {
        try {
            Integer fakeScore = computeFakeScore(review);
            review.setFakeScore(fakeScore);
            reviewRepository.save(review);

            // If fake score is high, flag for moderation
            if (fakeScore >= 75) {
                review.setFlagged(true);
                review.setFlagReason("POTENTIAL_FAKE_REVIEW");
                reviewRepository.save(review);
            }

            // Publish event for analytics
            FraudDetectionEvent event = FraudDetectionEvent.builder()
                .reviewId(review.getReviewId())
                .productId(review.getProductId())
                .fakeScore(fakeScore)
                .timestamp(LocalDateTime.now())
                .build();

            kafkaTemplate.send("fraud", review.getReviewId().toString(), event);

            log.info("Fraud analysis complete: review={}, score={}", review.getReviewId(), fakeScore);

        } catch (Exception e) {
            log.error("Error analyzing review for fraud", e);
        }
    }

    private Integer computeFakeScore(Review review) {
        int score = 0;

        // Signal 1: Account age
        Customer customer = customerService.getCustomer(review.getCustomerId());
        LocalDateTime accountCreatedAt = customer.getCreatedAt();
        long accountAgeDays = java.time.temporal.ChronoUnit.DAYS
            .between(accountCreatedAt, LocalDateTime.now());

        if (accountAgeDays < 7) {
            score += 25; // New account is suspicious
        } else if (accountAgeDays < 30) {
            score += 10;
        }

        // Signal 2: Review velocity (multiple reviews quickly)
        long recentReviewCount = reviewRepository.countByCustomerIdAndCreatedAtAfter(
            review.getCustomerId(),
            LocalDateTime.now().minusDays(1)
        );

        if (recentReviewCount > 5) {
            score += 20; // Lots of reviews in short time
        }

        // Signal 3: Rating pattern (all 5-star from new accounts)
        if (review.getRating() == 5 && accountAgeDays < 30) {
            score += 15;
        }

        // Signal 4: Text analysis (bot-like language)
        String text = review.getText().toLowerCase();
        if (text.contains("you wont regret")
            || text.contains("amazing product")
            || text.contains("perfect")
            || text.contains("highly recommend")) {
            score += 10; // Generic language
        }

        // Signal 5: No verified purchase
        if (!review.isVerifiedPurchase()) {
            score += 25;
        }

        // Signal 6: Very short reviews
        if (review.getText() == null || review.getText().length() < 20) {
            score += 10;
        }

        return Math.min(score, 100); // Cap at 100
    }
}
```

### 9.6 Review API Controller

```java
package com.retail.review.controller;

import com.retail.review.entity.Review;
import com.retail.review.service.ReviewService;
import com.retail.review.service.HelpfulVoteService;
import com.retail.review.service.RatingService;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.core.Authentication;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/products/{productId}/reviews")
@Slf4j
public class ReviewController {

    private final ReviewService reviewService;
    private final HelpfulVoteService helpfulVoteService;
    private final RatingService ratingService;
    private final MeterRegistry meterRegistry;

    public ReviewController(
        ReviewService reviewService,
        HelpfulVoteService helpfulVoteService,
        RatingService ratingService,
        MeterRegistry meterRegistry
    ) {
        this.reviewService = reviewService;
        this.helpfulVoteService = helpfulVoteService;
        this.ratingService = ratingService;
        this.meterRegistry = meterRegistry;
    }

    @PostMapping
    public ResponseEntity<Review> createReview(
        @PathVariable UUID productId,
        @RequestBody CreateReviewRequest request,
        Authentication authentication
    ) {
        UUID customerId = UUID.fromString(authentication.getName());

        try {
            Review review = reviewService.createReview(request, customerId);
            return ResponseEntity.status(HttpStatus.CREATED).body(review);
        } catch (DuplicateReviewException e) {
            return ResponseEntity.status(HttpStatus.CONFLICT).build();
        } catch (UnverifiedPurchaseException e) {
            return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
        }
    }

    @GetMapping
    public ResponseEntity<Page<Review>> listReviews(
        @PathVariable UUID productId,
        @RequestParam(defaultValue = "RECENT") SortBy sortBy,
        Pageable pageable
    ) {
        Page<Review> reviews = reviewService.listReviewsForProduct(productId, sortBy, pageable);
        return ResponseEntity.ok(reviews);
    }

    @GetMapping("/{reviewId}")
    public ResponseEntity<Review> getReview(
        @PathVariable UUID productId,
        @PathVariable UUID reviewId
    ) {
        Review review = reviewService.getReview(reviewId);

        if (!review.getProductId().equals(productId)) {
            return ResponseEntity.notFound().build();
        }

        return ResponseEntity.ok(review);
    }

    @PostMapping("/{reviewId}/helpful")
    public ResponseEntity<Void> markHelpful(
        @PathVariable UUID productId,
        @PathVariable UUID reviewId,
        @RequestParam VoteType voteType,
        Authentication authentication
    ) {
        UUID customerId = UUID.fromString(authentication.getName());

        helpfulVoteService.voteHelpful(reviewId, customerId, voteType);
        return ResponseEntity.status(HttpStatus.NO_CONTENT).build();
    }

    @GetMapping("/{reviewId}/helpful-count")
    public ResponseEntity<Integer> getHelpfulCount(
        @PathVariable UUID productId,
        @PathVariable UUID reviewId
    ) {
        Integer count = helpfulVoteService.getHelpfulVoteCount(reviewId);
        return ResponseEntity.ok(count);
    }

    @GetMapping("/product-rating")
    public ResponseEntity<ProductRatingDto> getProductRating(
        @PathVariable UUID productId
    ) {
        ProductRatingDto rating = ratingService.getProductRating(productId);
        return ResponseEntity.ok(rating);
    }

    @PostMapping("/{reviewId}/flag")
    public ResponseEntity<Void> flagReview(
        @PathVariable UUID productId,
        @PathVariable UUID reviewId,
        @RequestBody FlagReviewRequest request
    ) {
        reviewService.flagReview(reviewId, request.reason());
        return ResponseEntity.status(HttpStatus.NO_CONTENT).build();
    }
}

public enum SortBy {
    RECENT, HELPFUL, RATING_HIGH, RATING_LOW
}
```

---

## 10. Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| Order service down | Can't verify purchase | Allow review with PENDING_VERIFICATION, cache last 24h |
| Rating aggregation lag | Stale aggregate ratings | Redis cache with computed values, async Kafka processing |
| Kafka message loss | Lost review events | Kafka retention 90 days, replay on gaps detected |
| MongoDB replica lag | View unverified reviews | Read from primary for critical queries, eventual consistency for UI |
| Redis cache eviction | Recompute ratings on demand | Fall back to PostgreSQL, cache miss cost acceptable |
| Fraudulent reviews published | Bad data pollutes ratings | Moderation queue, human review, easy report mechanism |
| High volume of flags | Moderation backlog | Auto-flag high fake_score, priority queue, escalation |
| Email service down | Notifications not sent | Queue in Kafka, retry with backoff |

---

## 11. Scaling Strategy

### Horizontal Scaling
- **Review API:** Stateless Spring Boot; scale based on RPS
- **Helpful Vote Service:** Stateless; can be colocated with review API
- **Moderation Service:** Stateless REST; scale horizontally
- **Kafka Consumers:** Partition topics; scale consumer groups independently

### Database Optimization
- Index `(product_id, created_at)` for recent reviews query
- Index `(product_id, rating)` for rating-filtered queries
- Partition review collection by `product_id` in MongoDB (if needed)
- Archive old reviews to separate collection (older than 1 year)

### Redis Caching
- Product ratings: TTL 1 hour, refresh on new review
- Recent reviews per product: TTL 1 hour, max 100 cached
- Customer purchase cache: TTL 24 hours
- Helpful votes counters: ephemeral, no TTL needed

### Kafka Partitioning
- `reviews`: partition by `product_id` (group reviews by product)
- `helpful_votes`: partition by `review_id` (preserve order per review)
- `fraud`: partition by `product_id` (analyze fraud patterns per product)
- `moderation`: partition by `priority` (separate queue for urgent flags)

---

## 12. Monitoring & Observability

### Key Metrics
```
review.created (counter, tag: rating, verified_purchase)
review.flagged (counter, tag: reason)
review.approved (counter)
review.rejected (counter)
helpful_vote.recorded (counter, tag: vote_type)
helpful_vote.withdrawn (counter)
rating.aggregation_latency (timer, tag: product_id)
fake_review.score_computed (distribution, tag: fake_score_range)
moderation.queue_depth (gauge)
moderation.review_approval_latency (timer)
```

### Alerting
- If fake_review detection latency > 10s: investigate ML pipeline
- If moderation.queue_depth > 1000: escalate to team
- If helpful_vote processing latency p99 > 1s: scale consumer
- If rating.aggregation_latency > 5s: investigate Kafka lag
- If review.created rate drop by 50% in 5min: check API health

### Distributed Tracing
All Kafka consumers and API endpoints instrumented with Spring Cloud Sleuth + Zipkin for request tracing.

---

## 13. Summary Cheat Sheet

### Critical Patterns
- **Purchase Verification:** Order service query with Redis cache (24h TTL)
- **Helpful Votes:** Redis counter + eventual DB persistence
- **Rating Aggregation:** Kafka consumer batch processing with atomic DB + cache update
- **Fraud Detection:** Async ML scoring with Kafka event streaming
- **Moderation Queue:** Kafka-backed async task queue with human dashboard

### Key Data Structures

| Entity | Storage | Key Index | TTL |
|--------|---------|-----------|-----|
| Review | MongoDB | product_id, created_at | N/A |
| Rating | PostgreSQL | product_id (primary) | N/A |
| Helpful Votes | MongoDB | (review_id, customer_id) | N/A |
| Purchase Cache | Redis | customer:product | 24 hours |
| Rating Cache | Redis | ratings:product_id | 1 hour |
| Recent Reviews | Redis | product:reviews:recent | 1 hour |

### API Endpoints
```
POST /products/{id}/reviews                      → Create review (verified purchaser)
GET /products/{id}/reviews?sort=RECENT|HELPFUL   → List reviews
GET /products/{id}/reviews/{id}                  → Get review detail
POST /products/{id}/reviews/{id}/helpful         → Vote helpful/unhelpful
POST /products/{id}/reviews/{id}/flag            → Flag review for moderation
GET /products/{id}/rating                        → Get aggregate rating
```

### Kafka Topics
```
reviews → review CRUD events
helpful_votes → helpful/unhelpful vote events
ratings → product rating aggregation events
fraud → fraud detection scores
moderation → flagged review moderation events
```

### Best Practices
1. **Verify purchase first:** Always validate before allowing review creation
2. **Async fraud detection:** ML scoring doesn't block review creation
3. **Moderation SLA:** 80% of flags reviewed within 24 hours
4. **Cache strategy:** Hot data (ratings) in Redis, cold data in DB
5. **Idempotency:** Customer can only have 1 review per product (enforced at app layer)

