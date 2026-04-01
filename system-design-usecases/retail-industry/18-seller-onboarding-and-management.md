---
title: Q18 - Seller Onboarding and Management Platform
layout: default
---

# Q18 - Seller Onboarding and Management Platform — Deep Dive Design

> **Scenario**: Build a seller onboarding and management platform for a marketplace. Sellers register, submit business documents, upload product catalogs, manage pricing/inventory, track orders, receive payouts, view analytics, and resolve disputes. Support 100K sellers, 1M products, 50K orders/day.
>
> **Tech Stack**: Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka

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
- **Seller Registration**: Multi-step onboarding with identity/business document verification
- **Product Management**: Bulk CSV uploads (100K+ products), real-time pricing updates, inventory sync
- **Order Fulfillment**: Track fulfillment from order to delivery
- **Payment Settlement**: Weekly automated payouts with detailed breakdowns
- **Analytics Dashboard**: Real-time sales, traffic, conversion metrics per seller
- **Dispute Resolution**: Handle customer-seller disputes with evidence tracking
- **Support Tickets**: Category-based ticket system for seller support

### Non-Functional Requirements
- **Availability**: 99.9% uptime
- **Latency**: P99 < 500ms for API calls
- **Throughput**: 50K orders/day, 100 CSV uploads/day, 1000s concurrent sellers
- **Data Consistency**: Strong consistency for payments, eventual consistency for analytics
- **Security**: PCI-DSS compliance, seller data isolation, document encryption
- **Scalability**: Support growth to 500K sellers, 10M products

### Constraints
- Multi-tenant isolation required (tenant_id on all tables)
- Regulatory compliance (tax reporting, KYC/AML)
- Document verification must be auditable
- Payout processing tied to business hours
- Seller analytics must not impact transactional system

---

## 2. Capacity Estimation

| Metric | Value | Notes |
|--------|-------|-------|
| Active Sellers | 100,000 | Unique seller accounts |
| Total Products | 1,000,000 | Across all sellers (avg 10 per seller) |
| Daily Orders | 50,000 | Orders going to sellers |
| CSV Uploads/Day | 100 | Bulk product uploads |
| Avg Products/CSV | 10,000 | Range: 1K–100K |
| Concurrent Sellers | 5,000 | Peak simultaneous dashboard users |
| Payouts/Week | 50,000 | One per order (simplified) |
| Tickets/Month | 50,000 | 1,667 per day |
| Daily Analytics Events | 500,000 | Page views, clicks, conversions |

### Storage Estimates
- **PostgreSQL**: Sellers (~1GB), Products (~50GB), Orders (~100GB/year), Transactions (~50GB/year)
- **MongoDB**: Analytics snapshots (~10GB/year), Tickets (~5GB/year)
- **Redis**: Hot product cache (~5GB), Inventory cache (~2GB), Session store (~1GB)
- **S3**: Documents (~100GB), Product images (~500GB), Analytics exports (~100GB/year)

### Network Bandwidth
- Outbound: ~500 Mbps average, 2 Gbps peak (during CSV uploads)
- Database queries: ~100K req/s average, 500K req/s peak

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Client Layer                                  │
│  (Web/Mobile Apps, Admin Dashboard, Seller Portal)                  │
└────────────────────────┬────────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────────┐
│                    API Gateway Layer                                 │
│  (Rate Limiting, Auth, Request Validation)                          │
└────────────────────────┬────────────────────────────────────────────┘
                         │
        ┌────────────────┼────────────────────┐
        │                │                    │
┌───────▼──────┐ ┌──────▼────────┐ ┌────────▼─────────┐
│ Seller Svc   │ │ Product Svc   │ │ Order/Fulfil Svc │
│ (Onboarding) │ │ (Catalog Mgmt)│ │ (Order Tracking) │
└────┬─────────┘ └──────┬────────┘ └────────┬─────────┘
     │                  │                    │
     └──────────────────┼────────────────────┘
                        │
                   ┌────▼──────────────┐
                   │  Kafka Broker     │
                   │  (Event Bus)      │
                   └─────┬──────┬──────┘
                         │      │
        ┌────────────────┘      └──────────────────┐
        │                                          │
┌───────▼────────────┐               ┌────────────▼────────┐
│ Payout Service     │               │ Analytics Service   │
│ (Settlement)       │               │ (CQRS Read Model)   │
└───────┬────────────┘               └────────────┬────────┘
        │                                         │
┌───────▼──────────────────────────────────────┬─▼────────┐
│         Persistent Data Layer                 │ Pipeline │
├───────────────┬───────────────┬───────────────┼──────────┤
│  PostgreSQL   │   MongoDB     │    Redis      │   S3     │
│  (OLTP)       │   (Analytics) │   (Cache)     │ (Docs)   │
└───────────────┴───────────────┴───────────────┴──────────┘
```

---

## 4. Core Design Questions Answered

### Q1: How do you verify seller identity and business legitimacy?

**Answer**: Multi-stage async verification with document upload, background checks, and manual review.

**Design**:
- **Stage 1 (Auto-Verify)**: Validate document format, metadata, OCR for basic fields
- **Stage 2 (Async Background Check)**: Send documents to third-party verification API (Stripe, Jumio)
- **Stage 3 (Manual Review)**: For high-risk sellers, dispatch to ops team via ticket system
- **State Machine**: `PENDING → DOCUMENTS_SUBMITTED → UNDER_REVIEW → APPROVED/REJECTED`

**Data Flow**:
```
SellerRegistration API
    ↓
Save seller in PENDING state (PostgreSQL)
    ↓
Upload documents to S3
    ↓
Publish SellerDocumentsSubmittedEvent → Kafka
    ↓
DocumentVerificationConsumer polls Kafka
    ↓
Call third-party API (async)
    ↓
Update seller status in PostgreSQL
    ↓
If approved: publish SellerApprovedEvent
```

**Security**: All documents encrypted at rest (AES-256), access logs tracked, IP restrictions on seller verification endpoints.

---

### Q2: How do you handle bulk product uploads (100K products)?

**Answer**: Spring Batch job with chunked processing, validation, deduplication, and async indexing.

**Design**:
- **Batch Job**: Read CSV in chunks of 1K rows, validate each chunk, bulk upsert to PostgreSQL
- **Validation**: Product name, SKU, price, inventory fields; deduplicate by seller_id + SKU
- **Commit Strategy**: Commit every 1K rows to avoid transaction bloat
- **Async Indexing**: After DB insert, publish ProductIndexedEvent → Kafka → Elasticsearch consumer

**Error Handling**:
- Invalid rows logged to `product_upload_errors` table with row number, error message
- Partial upload recovery: if batch fails mid-stream, resume from last committed chunk
- Report returned to seller with summary: "9,990 products added, 10 errors" + downloadable error log

**Processing Time**: 100K products = ~2–5 minutes depending on validation complexity.

---

### Q3: How do you isolate seller data for security?

**Answer**: Row-level security with `tenant_id` on all tables, Spring Security annotations, and database views.

**Design**:
- **Multi-Tenancy**: Every table has `tenant_id` (seller_id), foreign key constraints
- **Spring Security**: Custom `@PreAuthorize` annotation leverages seller context from JWT
  ```java
  @PreAuthorize("@sellerAuth.owns(#sellerId)")
  public SellerProfile getSellerProfile(@PathVariable Long sellerId) { ... }
  ```
- **Database Views**: Create seller-specific views (e.g., `seller_123_orders`)
- **Query Filtering**: SellerRepository adds `.where(SELLER_ID.eq(tenantId))` to all queries automatically
- **Auditing**: Track all data access via triggers → `audit_logs` table

**Encryption**: Sensitive fields (business license number, tax ID) encrypted with seller-specific keys (key stored in HashiCorp Vault).

---

### Q4: How do you calculate and process seller payouts?

**Answer**: Weekly batch aggregation of settled orders, fee deduction, and payment gateway integration.

**Design**:
- **Payout Trigger**: Weekly job (Monday 2 AM) aggregates settled orders from previous week
- **Settlement Rules**:
  - Include only DELIVERED orders (3+ days old to handle returns)
  - Deduct platform fees (15%), payment processing (2.9%), refunds
  - Minimum payout threshold ($100) to reduce transaction costs
- **Payout Record**: Create `seller_payout` record with breakdown (sales, fees, taxes, net amount)
- **Payment Processing**: Call Stripe/PayPal API → record transaction ID → webhook updates payout status
- **Dispute Escrow**: If open disputes, hold amount in escrow until resolved

**State Machine**: `PENDING → PROCESSING → COMPLETED/FAILED`

---

### Q5: How do you provide real-time analytics without impacting transactional systems?

**Answer**: CQRS pattern with async aggregation. Write orders to PostgreSQL, async Kafka consumer publishes to MongoDB analytics store.

**Design**:
- **Write Side**: Order created → saved to PostgreSQL (OLTP optimized)
- **Event Side**: `OrderConfirmedEvent` published to Kafka topic `seller-analytics`
- **Read Side Consumer**: Subscribes to `seller-analytics`, aggregates metrics every 1 hour:
  - **Metrics**: Sales revenue, order count, traffic (page views from separate event stream), conversion rate
  - **Storage**: MongoDB document per seller per day: `{ seller_id, date, sales, orders, traffic, conversions }`
- **Query API**: `/sellers/{id}/analytics` reads from MongoDB (much faster than aggregating PostgreSQL)
- **Real-time Dashboard**: Use Redis cache for last 24 hours (Redis sorted set with hourly buckets), MongoDB for historical

**Latency Guarantees**: MongoDB reads P99 < 50ms, hourly aggregation delay acceptable for dashboard.

---

### Q6: How do you handle disputes between customers and sellers?

**Answer**: Ticketing system with evidence tracking, escalation, and arbitration workflow.

**Design**:
- **Dispute Initiation**: Customer creates ticket from order detail page
- **Evidence Submission**: Both customer and seller can upload evidence (photos, messages, receipts)
- **Triage**: Automatically categorized (quality, missing items, damaged, etc.) → routed to dispute resolution team
- **Resolution Options**:
  1. **Automatic**: Low-value disputes (< $50) with clear evidence → auto-refund with ML rules
  2. **Manual Review**: Ops team reviews evidence, decides refund/reject
  3. **Escalation**: Unresolved after 14 days → escalated to management
- **Appeal**: Either party can appeal decision within 7 days
- **Data**: Disputes stored in MongoDB (documents, evidence), linked to PostgreSQL orders table

---

## 5. Microservices Breakdown

| Service | Responsibility | Tech | Scaling |
|---------|-----------------|------|---------|
| **Seller Service** | Onboarding, profiles, verification | Spring Boot + PostgreSQL | 3–10 instances |
| **Product Service** | Catalog management, bulk uploads, indexing | Spring Boot + PostgreSQL + Elasticsearch | 5–20 instances |
| **Order Service** | Order creation, tracking, fulfillment | Spring Boot + PostgreSQL + Kafka | 5–15 instances |
| **Payout Service** | Settlement calculations, payment gateway | Spring Boot + PostgreSQL + Redis | 2–5 instances |
| **Analytics Service** | Aggregation, reporting, dashboards | Spring Boot + MongoDB + Kafka | 3–10 instances |
| **Dispute Service** | Ticket management, evidence tracking | Spring Boot + MongoDB | 2–8 instances |
| **Document Service** | Document storage, verification, OCR | Spring Boot + S3 + Redis | 2–5 instances |

---

## 6. Database Design

### PostgreSQL Schema (OLTP)

```sql
-- Sellers (multi-tenant root)
CREATE TABLE sellers (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    business_name VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL, -- PENDING, UNDER_REVIEW, APPROVED, REJECTED, SUSPENDED
    verification_status VARCHAR(50), -- PENDING, APPROVED, REJECTED
    tax_id VARCHAR(100) ENCRYPTED,
    bank_account_id VARCHAR(100) ENCRYPTED,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    INDEX idx_status (status),
    INDEX idx_email (email)
);

-- Seller Documents
CREATE TABLE seller_documents (
    id BIGSERIAL PRIMARY KEY,
    seller_id BIGINT NOT NULL REFERENCES sellers(id) ON DELETE CASCADE,
    document_type VARCHAR(50), -- BUSINESS_LICENSE, TAX_ID, IDENTITY, BANK_STATEMENT
    s3_key VARCHAR(500) NOT NULL,
    document_url VARCHAR(500),
    verification_status VARCHAR(50), -- PENDING, VERIFIED, REJECTED
    rejection_reason TEXT,
    created_at TIMESTAMP NOT NULL,
    UNIQUE(seller_id, document_type),
    INDEX idx_seller_id (seller_id)
);

-- Products (heavily indexed for search)
CREATE TABLE products (
    id BIGSERIAL PRIMARY KEY,
    seller_id BIGINT NOT NULL REFERENCES sellers(id) ON DELETE CASCADE,
    sku VARCHAR(100) NOT NULL,
    name VARCHAR(500) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    cost DECIMAL(10, 2),
    inventory INT NOT NULL DEFAULT 0,
    status VARCHAR(50), -- ACTIVE, INACTIVE, DELISTED
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    UNIQUE(seller_id, sku),
    INDEX idx_seller_id (seller_id),
    INDEX idx_sku (sku),
    INDEX idx_status (status)
);

-- Orders (order event stream)
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    seller_id BIGINT NOT NULL REFERENCES sellers(id),
    customer_id BIGINT NOT NULL,
    status VARCHAR(50) NOT NULL, -- CONFIRMED, PROCESSING, SHIPPED, DELIVERED
    total_amount DECIMAL(12, 2) NOT NULL,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    INDEX idx_seller_id (seller_id),
    INDEX idx_customer_id (customer_id),
    INDEX idx_status (status),
    INDEX idx_created_at (created_at)
);

-- Order Items
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id BIGINT NOT NULL REFERENCES products(id),
    seller_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    subtotal DECIMAL(12, 2) NOT NULL,
    INDEX idx_order_id (order_id),
    INDEX idx_product_id (product_id)
);

-- Seller Payouts
CREATE TABLE seller_payouts (
    id BIGSERIAL PRIMARY KEY,
    seller_id BIGINT NOT NULL REFERENCES sellers(id),
    payout_period_start DATE NOT NULL,
    payout_period_end DATE NOT NULL,
    gross_sales DECIMAL(12, 2) NOT NULL,
    platform_fees DECIMAL(12, 2) NOT NULL,
    payment_processing_fees DECIMAL(12, 2) NOT NULL,
    refunds DECIMAL(12, 2) DEFAULT 0,
    net_amount DECIMAL(12, 2) NOT NULL,
    status VARCHAR(50), -- PENDING, PROCESSING, COMPLETED, FAILED
    transaction_id VARCHAR(100),
    created_at TIMESTAMP NOT NULL,
    completed_at TIMESTAMP,
    INDEX idx_seller_id (seller_id),
    INDEX idx_status (status),
    INDEX idx_payout_period_start (payout_period_start)
);

-- Product Upload Errors (for tracking bulk import issues)
CREATE TABLE product_upload_errors (
    id BIGSERIAL PRIMARY KEY,
    seller_id BIGINT NOT NULL REFERENCES sellers(id),
    upload_batch_id VARCHAR(100) NOT NULL,
    row_number INT,
    error_message TEXT,
    error_data JSONB,
    created_at TIMESTAMP NOT NULL,
    INDEX idx_seller_id (seller_id),
    INDEX idx_batch_id (upload_batch_id)
);

-- Audit Logs (immutable, append-only)
CREATE TABLE audit_logs (
    id BIGSERIAL PRIMARY KEY,
    seller_id BIGINT,
    action VARCHAR(100),
    entity_type VARCHAR(50),
    entity_id BIGINT,
    old_values JSONB,
    new_values JSONB,
    ip_address VARCHAR(50),
    created_at TIMESTAMP NOT NULL,
    INDEX idx_seller_id (seller_id),
    INDEX idx_created_at (created_at)
);
```

### MongoDB Collections (Analytics & Tickets)

```json
// seller_daily_analytics
{
    "_id": "seller_123_2024-04-01",
    "seller_id": 123,
    "date": ISODate("2024-04-01"),
    "sales_revenue": 15000.50,
    "order_count": 250,
    "page_views": 5000,
    "conversions": 300,
    "conversion_rate": 6.0,
    "avg_order_value": 60.00,
    "refund_amount": 500.00,
    "top_products": [
        { "product_id": 1, "name": "Widget A", "revenue": 5000, "units": 100 }
    ]
}

// seller_disputes (also called tickets)
{
    "_id": ObjectId(),
    "seller_id": 123,
    "customer_id": 456,
    "order_id": 789,
    "ticket_id": "DISP-20240401-001",
    "category": "QUALITY_ISSUE",
    "subject": "Product arrived damaged",
    "description": "...",
    "status": "OPEN", // OPEN, ASSIGNED, IN_REVIEW, RESOLVED, CLOSED
    "priority": "HIGH",
    "evidence": [
        { "type": "IMAGE", "url": "s3://...", "uploaded_by": "customer", "uploaded_at": ISODate() }
    ],
    "resolution": {
        "outcome": "REFUND",
        "amount": 50.00,
        "decided_by": "ops_agent_123"
    },
    "created_at": ISODate(),
    "updated_at": ISODate()
}
```

---

## 7. Redis Data Structures

```redis
# Product Cache (hot products, 24-hour TTL)
seller:123:products:{product_id} = HASH {
    "id": "456",
    "name": "Widget A",
    "price": "29.99",
    "inventory": "150",
    "updated_at": "1712099999"
}
TTL: 86400 seconds (24 hours)

# Seller Session
session:{session_id} = HASH {
    "seller_id": "123",
    "email": "seller@example.com",
    "login_at": "1712099999",
    "last_activity": "1712101000"
}
TTL: 3600 seconds (1 hour, refreshed on activity)

# Inventory Cache (real-time, 5-minute TTL)
inventory:seller:123:product:456 = STRING "150"
TTL: 300 seconds

# Analytics Hourly Buckets (for real-time dashboard)
seller:123:analytics:2024-04-01:13 = HASH {
    "sales": "2500.00",
    "orders": "45",
    "page_views": "600"
}
TTL: 2592000 seconds (30 days)

# Seller Rating Cache (24-hour TTL)
seller:123:rating = HASH {
    "avg_rating": "4.7",
    "total_reviews": "1250",
    "quality": "4.6",
    "shipping_speed": "4.8"
}
TTL: 86400 seconds

# Payout Lock (distribute payout processing across workers)
payout:lock:{seller_id}:{week_start} = STRING "worker_node_3"
TTL: 300 seconds (during processing)

# Rate Limiting (per seller per endpoint)
ratelimit:seller:123:bulk-upload = STRING "5"
TTL: 3600 seconds (reset hourly)
```

---

## 8. Kafka Event Flow

### Topics & Events

| Topic | Events | Partition Key | Retention |
|-------|--------|---------------|-----------|
| `seller-onboarding` | SellerRegisteredEvent, SellerDocumentsSubmittedEvent, SellerApprovedEvent | seller_id | 30 days |
| `product-catalog` | ProductUploadStartedEvent, ProductAddedEvent, ProductDeletedEvent, ProductUpdatedEvent | seller_id | 30 days |
| `seller-orders` | OrderConfirmedEvent, OrderShippedEvent, OrderDeliveredEvent | seller_id | 90 days |
| `seller-analytics` | OrderConfirmedEvent, PageViewEvent, CheckoutEvent | seller_id | 365 days |
| `seller-documents` | DocumentUploadedEvent, DocumentVerifiedEvent, DocumentRejectedEvent | seller_id | 90 days |
| `seller-payouts` | PayoutCalculatedEvent, PayoutProcessedEvent, PayoutFailedEvent | seller_id | 365 days |
| `seller-disputes` | DisputeCreatedEvent, DisputeEvidenceSubmittedEvent, DisputeResolvedEvent | seller_id | 365 days |

### Event Examples

```json
// SellerApprovedEvent
{
    "event_id": "evt-123456",
    "event_type": "SELLER_APPROVED",
    "seller_id": 123,
    "timestamp": "2024-04-01T10:30:00Z",
    "verified_at": "2024-04-01T10:29:00Z",
    "verification_method": "AUTOMATED"
}

// ProductAddedEvent
{
    "event_id": "evt-123457",
    "event_type": "PRODUCT_ADDED",
    "seller_id": 123,
    "product_id": 456,
    "product_name": "Widget A",
    "sku": "WIDGET-001",
    "price": 29.99,
    "timestamp": "2024-04-01T10:35:00Z"
}

// OrderConfirmedEvent
{
    "event_id": "evt-123458",
    "event_type": "ORDER_CONFIRMED",
    "order_id": 789,
    "seller_id": 123,
    "customer_id": 456,
    "total_amount": 150.00,
    "timestamp": "2024-04-01T10:40:00Z"
}
```

---

## 9. Implementation Code

### 9.1 SellerOnboardingService

```java
@Service
@RequiredArgsConstructor
public class SellerOnboardingService {
    private final SellerRepository sellerRepository;
    private final SellerDocumentRepository documentRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final SellerAuthService authService;
    private final AuditService auditService;
    private final ObjectMapper objectMapper;
    private static final Logger logger = LoggerFactory.getLogger(SellerOnboardingService.class);

    public SellerRegistrationResponse registerSeller(SellerRegistrationRequest request) {
        // Validate email uniqueness
        if (sellerRepository.existsByEmail(request.getEmail())) {
            throw new BusinessException("Email already registered");
        }

        // Create seller in PENDING state
        Seller seller = new Seller();
        seller.setEmail(request.getEmail());
        seller.setBusinessName(request.getBusinessName());
        seller.setStatus(SellerStatus.PENDING);
        seller.setVerificationStatus(VerificationStatus.PENDING);
        seller.setCreatedAt(LocalDateTime.now());

        Seller savedSeller = sellerRepository.save(seller);
        auditService.logAction(savedSeller.getId(), "SELLER_REGISTERED", null,
            objectMapper.valueToTree(savedSeller));

        // Publish event
        publishSellerRegisteredEvent(savedSeller);

        // Generate auth credentials
        String token = authService.generateSellerToken(savedSeller);

        logger.info("Seller {} registered successfully", savedSeller.getId());
        return new SellerRegistrationResponse(savedSeller.getId(), token, SellerStatus.PENDING);
    }

    public void submitDocuments(Long sellerId, DocumentSubmissionRequest request) {
        Seller seller = sellerRepository.findById(sellerId)
            .orElseThrow(() -> new SellerNotFoundException(sellerId));

        if (!seller.getStatus().equals(SellerStatus.PENDING)) {
            throw new BusinessException("Seller cannot submit documents in current status: " + seller.getStatus());
        }

        // Save each document
        for (DocumentSubmissionRequest.DocumentData docData : request.getDocuments()) {
            SellerDocument doc = new SellerDocument();
            doc.setSellerId(sellerId);
            doc.setDocumentType(docData.getType());
            doc.setS3Key("sellers/" + sellerId + "/" + UUID.randomUUID() + ".pdf");
            doc.setVerificationStatus(VerificationStatus.PENDING);
            doc.setCreatedAt(LocalDateTime.now());

            documentRepository.save(doc);

            // Upload to S3 (async, not shown)
            uploadDocumentToS3(doc.getS3Key(), docData.getFileContent());
        }

        // Update seller status
        seller.setStatus(SellerStatus.DOCUMENTS_SUBMITTED);
        sellerRepository.save(seller);

        // Publish event for async verification
        publishSellerDocumentsSubmittedEvent(seller);

        logger.info("Seller {} submitted {} documents", sellerId, request.getDocuments().size());
    }

    private void publishSellerRegisteredEvent(Seller seller) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "SELLER_REGISTERED");
            event.put("seller_id", seller.getId());
            event.put("email", seller.getEmail());
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("seller-onboarding", seller.getId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish SellerRegisteredEvent for seller {}", seller.getId(), e);
        }
    }

    private void publishSellerDocumentsSubmittedEvent(Seller seller) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "SELLER_DOCUMENTS_SUBMITTED");
            event.put("seller_id", seller.getId());
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("seller-onboarding", seller.getId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish SellerDocumentsSubmittedEvent for seller {}", seller.getId(), e);
        }
    }

    private void uploadDocumentToS3(String s3Key, byte[] content) {
        // S3 upload logic (stubbed)
    }
}

@Data
class SellerRegistrationRequest {
    private String email;
    private String businessName;
    private String phone;
}

@Data
class SellerRegistrationResponse {
    private Long sellerId;
    private String authToken;
    private SellerStatus status;

    public SellerRegistrationResponse(Long sellerId, String authToken, SellerStatus status) {
        this.sellerId = sellerId;
        this.authToken = authToken;
        this.status = status;
    }
}
```

### 9.2 BulkProductUploadJob (Spring Batch)

```java
@Configuration
@RequiredArgsConstructor
public class BulkProductUploadJobConfig {
    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    private final ProductRepository productRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;
    private static final int CHUNK_SIZE = 1000;

    @Bean
    public Job bulkProductUploadJob(Step uploadStep) {
        return jobBuilderFactory.get("bulkProductUploadJob")
            .start(uploadStep)
            .build();
    }

    @Bean
    public Step uploadStep(ItemReader<ProductCSVRow> csvReader,
                          ItemProcessor<ProductCSVRow, Product> processor,
                          ItemWriter<Product> writer) {
        return stepBuilderFactory.get("uploadStep")
            .<ProductCSVRow, Product>chunk(CHUNK_SIZE)
            .reader(csvReader)
            .processor(processor)
            .writer(writer)
            .faultTolerant()
            .skipLimit(100)
            .skip(InvalidProductException.class)
            .listener(new ChunkListener(kafkaTemplate, objectMapper))
            .build();
    }

    @Bean
    @StepScope
    public ItemReader<ProductCSVRow> csvFileItemReader(
            @Value("#{jobParameters['filePath']}") String filePath) {
        FlatFileItemReader<ProductCSVRow> reader = new FlatFileItemReader<>();
        reader.setResource(new FileSystemResource(filePath));
        reader.setLineMapper(new DefaultLineMapper<ProductCSVRow>() {{
            setLineTokenizer(new DelimitedLineTokenizer() {{
                setNames("sku", "name", "description", "price", "cost", "inventory");
            }});
            setFieldSetMapper(fieldSet -> {
                ProductCSVRow row = new ProductCSVRow();
                row.setSku(fieldSet.readString("sku"));
                row.setName(fieldSet.readString("name"));
                row.setPrice(fieldSet.readBigDecimal("price"));
                row.setInventory(fieldSet.readInt("inventory"));
                return row;
            });
        }});
        return reader;
    }

    @Bean
    public ItemProcessor<ProductCSVRow, Product> productProcessor(
            @Value("#{jobParameters['sellerId']}") Long sellerId) {
        return row -> {
            // Validate row
            if (row.getSku() == null || row.getSku().isEmpty()) {
                throw new InvalidProductException("SKU is required");
            }
            if (row.getPrice().compareTo(BigDecimal.ZERO) <= 0) {
                throw new InvalidProductException("Price must be positive");
            }

            // Check for duplicate SKU
            boolean exists = productRepository.existsBySellerIdAndSku(sellerId, row.getSku());
            if (exists) {
                // Update existing product
                Product product = productRepository.findBySellerIdAndSku(sellerId, row.getSku())
                    .orElseThrow();
                product.setName(row.getName());
                product.setPrice(row.getPrice());
                product.setInventory(row.getInventory());
                product.setUpdatedAt(LocalDateTime.now());
                return product;
            }

            // Create new product
            Product product = new Product();
            product.setSellerId(sellerId);
            product.setSku(row.getSku());
            product.setName(row.getName());
            product.setPrice(row.getPrice());
            product.setInventory(row.getInventory());
            product.setStatus(ProductStatus.ACTIVE);
            product.setCreatedAt(LocalDateTime.now());
            return product;
        };
    }

    @Bean
    public ItemWriter<Product> productWriter(ProductRepository productRepository) {
        return items -> {
            productRepository.saveAll(items);
            logger.info("Batch wrote {} products", items.size());
        };
    }
}

@Component
class ChunkListener extends ChunkListener {
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    public ChunkListener(KafkaTemplate<String, String> kafkaTemplate, ObjectMapper objectMapper) {
        this.kafkaTemplate = kafkaTemplate;
        this.objectMapper = objectMapper;
    }

    @Override
    public void afterChunk(ChunkContext context) {
        // Publish event after every chunk is written
        StepExecution stepExecution = context.getStepContext().getStepExecution();
        Long sellerId = (Long) stepExecution.getJobParameters().getParameter("sellerId");
        int readCount = stepExecution.getReadCount();
        int writeCount = stepExecution.getWriteCount();

        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "PRODUCTS_BATCH_WRITTEN");
            event.put("seller_id", sellerId);
            event.put("batch_size", writeCount);
            event.put("total_processed", readCount);
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("product-catalog", sellerId.toString(), eventJson);
        } catch (Exception e) {
            logger.error("Failed to publish batch event", e);
        }
    }
}

@Data
class ProductCSVRow {
    private String sku;
    private String name;
    private String description;
    private BigDecimal price;
    private BigDecimal cost;
    private Integer inventory;
}

class InvalidProductException extends Exception {
    public InvalidProductException(String message) {
        super(message);
    }
}
```

### 9.3 SellerPayoutService

```java
@Service
@RequiredArgsConstructor
public class SellerPayoutService {
    private final SellerPayoutRepository payoutRepository;
    private final OrderRepository orderRepository;
    private final SellerRepository sellerRepository;
    private final PaymentGatewayClient paymentGateway;
    private final RedisTemplate<String, Object> redisTemplate;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;
    private static final Logger logger = LoggerFactory.getLogger(SellerPayoutService.class);
    private static final BigDecimal PLATFORM_FEE_RATE = new BigDecimal("0.15"); // 15%
    private static final BigDecimal PAYMENT_PROCESSING_RATE = new BigDecimal("0.029"); // 2.9%
    private static final BigDecimal MIN_PAYOUT_AMOUNT = new BigDecimal("100.00");

    @Scheduled(cron = "0 2 * * MON", zone = "UTC") // Every Monday at 2 AM UTC
    public void processWeeklyPayouts() {
        logger.info("Starting weekly payout processing");

        LocalDate now = LocalDate.now();
        LocalDate weekStart = now.minusDays(now.getDayOfWeek().getValue() % 7).minusDays(7);
        LocalDate weekEnd = weekStart.plusDays(6);

        List<Seller> approvedSellers = sellerRepository.findByStatus(SellerStatus.APPROVED);

        for (Seller seller : approvedSellers) {
            String lockKey = "payout:lock:" + seller.getId() + ":" + weekStart;

            // Use Redis lock to distribute processing (avoid duplicate processing)
            Boolean acquired = redisTemplate.opsForValue().setIfAbsent(lockKey, "processing",
                Duration.ofMinutes(5));
            if (!acquired) {
                logger.debug("Payout lock already held for seller {}", seller.getId());
                continue;
            }

            try {
                processPayout(seller, weekStart, weekEnd);
            } catch (Exception e) {
                logger.error("Failed to process payout for seller {}", seller.getId(), e);
                publishPayoutFailedEvent(seller, e.getMessage());
            } finally {
                redisTemplate.delete(lockKey);
            }
        }

        logger.info("Completed weekly payout processing");
    }

    private void processPayout(Seller seller, LocalDate periodStart, LocalDate periodEnd) {
        // Fetch settled orders (DELIVERED, at least 3 days old)
        LocalDateTime threeDaysAgo = LocalDateTime.now().minusDays(3);
        List<Order> settledOrders = orderRepository.findBySellerIdAndStatusAndCreatedAtBefore(
            seller.getId(), OrderStatus.DELIVERED, threeDaysAgo);

        if (settledOrders.isEmpty()) {
            logger.info("No settled orders for seller {} in period {}-{}", seller.getId(), periodStart, periodEnd);
            return;
        }

        // Calculate totals
        BigDecimal grossSales = settledOrders.stream()
            .map(Order::getTotalAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        BigDecimal platformFees = grossSales.multiply(PLATFORM_FEE_RATE);
        BigDecimal paymentFees = grossSales.multiply(PAYMENT_PROCESSING_RATE);
        BigDecimal refunds = calculateRefunds(seller, periodStart, periodEnd);

        BigDecimal netAmount = grossSales.subtract(platformFees).subtract(paymentFees).subtract(refunds);

        // Check minimum threshold
        if (netAmount.compareTo(MIN_PAYOUT_AMOUNT) < 0) {
            logger.info("Seller {} payout amount {} below minimum {}", seller.getId(), netAmount, MIN_PAYOUT_AMOUNT);
            // Holdover to next week (not implemented here)
            return;
        }

        // Create payout record
        SellerPayout payout = new SellerPayout();
        payout.setSellerId(seller.getId());
        payout.setPayoutPeriodStart(periodStart);
        payout.setPayoutPeriodEnd(periodEnd);
        payout.setGrossSales(grossSales);
        payout.setPlatformFees(platformFees);
        payout.setPaymentProcessingFees(paymentFees);
        payout.setRefunds(refunds);
        payout.setNetAmount(netAmount);
        payout.setStatus(PayoutStatus.PENDING);
        payout.setCreatedAt(LocalDateTime.now());

        SellerPayout savedPayout = payoutRepository.save(payout);

        // Call payment gateway
        try {
            PaymentGatewayClient.PayoutRequest request = new PaymentGatewayClient.PayoutRequest();
            request.setAmount(netAmount);
            request.setAccountId(seller.getBankAccountId()); // Decrypted during fetch
            request.setPayoutId(savedPayout.getId());

            PaymentGatewayClient.PayoutResponse response = paymentGateway.createPayout(request);

            // Update payout with transaction ID
            payout.setTransactionId(response.getTransactionId());
            payout.setStatus(PayoutStatus.PROCESSING);
            payoutRepository.save(payout);

            publishPayoutProcessedEvent(payout);

            logger.info("Payout {} for seller {} created with txn {}",
                savedPayout.getId(), seller.getId(), response.getTransactionId());
        } catch (Exception e) {
            payout.setStatus(PayoutStatus.FAILED);
            payoutRepository.save(payout);
            throw e;
        }
    }

    private BigDecimal calculateRefunds(Seller seller, LocalDate periodStart, LocalDate periodEnd) {
        // Fetch refunded orders in period
        LocalDateTime periodStartTime = periodStart.atStartOfDay();
        LocalDateTime periodEndTime = periodEnd.atTime(23, 59, 59);

        List<Order> refundedOrders = orderRepository.findBySellerIdAndStatusAndUpdatedAtBetween(
            seller.getId(), OrderStatus.REFUNDED, periodStartTime, periodEndTime);

        return refundedOrders.stream()
            .map(Order::getTotalAmount)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }

    private void publishPayoutProcessedEvent(SellerPayout payout) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "PAYOUT_PROCESSED");
            event.put("payout_id", payout.getId());
            event.put("seller_id", payout.getSellerId());
            event.put("amount", payout.getNetAmount());
            event.put("transaction_id", payout.getTransactionId());
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("seller-payouts", payout.getSellerId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish PayoutProcessedEvent", e);
        }
    }

    private void publishPayoutFailedEvent(Seller seller, String errorMessage) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "PAYOUT_FAILED");
            event.put("seller_id", seller.getId());
            event.put("error", errorMessage);
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("seller-payouts", seller.getId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish PayoutFailedEvent", e);
        }
    }
}
```

### 9.4 SellerAnalyticsService

```java
@Service
@RequiredArgsConstructor
public class SellerAnalyticsService {
    private final MongoTemplate mongoTemplate;
    private final RedisTemplate<String, Object> redisTemplate;
    private static final Logger logger = LoggerFactory.getLogger(SellerAnalyticsService.class);

    public SellerDailyAnalytics getDailyAnalytics(Long sellerId, LocalDate date) {
        // Try Redis cache first (24-hour TTL)
        String cacheKey = "seller:" + sellerId + ":analytics:" + date;
        SellerDailyAnalytics cached = (SellerDailyAnalytics) redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            return cached;
        }

        // Fall back to MongoDB
        Query query = new Query(Criteria.where("seller_id").is(sellerId)
            .and("date").is(date));

        SellerDailyAnalytics analytics = mongoTemplate.findOne(query, SellerDailyAnalytics.class);
        if (analytics != null) {
            redisTemplate.opsForValue().set(cacheKey, analytics, Duration.ofHours(24));
        }

        return analytics;
    }

    public SellerAnalyticsSummary getAnalyticsSummary(Long sellerId, LocalDate startDate, LocalDate endDate) {
        // Fetch from MongoDB, aggregate across date range
        Query query = new Query(Criteria.where("seller_id").is(sellerId)
            .and("date").gte(startDate).lte(endDate));

        List<SellerDailyAnalytics> dailyAnalytics = mongoTemplate.find(query, SellerDailyAnalytics.class);

        BigDecimal totalRevenue = dailyAnalytics.stream()
            .map(SellerDailyAnalytics::getSalesRevenue)
            .reduce(BigDecimal.ZERO, BigDecimal::add);

        long totalOrders = dailyAnalytics.stream()
            .mapToLong(SellerDailyAnalytics::getOrderCount)
            .sum();

        long totalPageViews = dailyAnalytics.stream()
            .mapToLong(SellerDailyAnalytics::getPageViews)
            .sum();

        long totalConversions = dailyAnalytics.stream()
            .mapToLong(SellerDailyAnalytics::getConversions)
            .sum();

        SellerAnalyticsSummary summary = new SellerAnalyticsSummary();
        summary.setSellerI(sellerId);
        summary.setStartDate(startDate);
        summary.setEndDate(endDate);
        summary.setTotalRevenue(totalRevenue);
        summary.setTotalOrders(totalOrders);
        summary.setTotalPageViews(totalPageViews);
        summary.setConversionRate(totalPageViews > 0 ?
            (double) totalConversions / totalPageViews * 100 : 0);
        summary.setAvgOrderValue(totalOrders > 0 ?
            totalRevenue.divide(new BigDecimal(totalOrders), 2, RoundingMode.HALF_UP) : BigDecimal.ZERO);

        return summary;
    }

    public List<ProductAnalytics> getTopProducts(Long sellerId, LocalDate date, int limit) {
        Query query = new Query(Criteria.where("seller_id").is(sellerId)
            .and("date").is(date));
        query.fields().include("top_products");

        SellerDailyAnalytics analytics = mongoTemplate.findOne(query, SellerDailyAnalytics.class);
        if (analytics == null) {
            return Collections.emptyList();
        }

        return analytics.getTopProducts().stream()
            .limit(limit)
            .collect(Collectors.toList());
    }

    @KafkaListener(topics = "seller-analytics", groupId = "analytics-aggregator")
    public void handleOrderConfirmedEvent(ConsumerRecord<String, String> record) {
        try {
            String message = record.value();
            Map<String, Object> event = new ObjectMapper().readValue(message, Map.class);

            if (!event.get("event_type").equals("ORDER_CONFIRMED")) {
                return;
            }

            Long sellerId = ((Number) event.get("seller_id")).longValue();
            BigDecimal amount = new BigDecimal(event.get("total_amount").toString());
            LocalDate date = LocalDate.now();

            // Update hourly bucket in Redis
            updateHourlyBucket(sellerId, date, amount);

            // Aggregation to MongoDB happens via separate scheduled job
        } catch (Exception e) {
            logger.error("Failed to process analytics event", e);
        }
    }

    private void updateHourlyBucket(Long sellerId, LocalDate date, BigDecimal amount) {
        int hour = LocalDateTime.now().getHour();
        String hourlyKey = "seller:" + sellerId + ":analytics:" + date + ":" + hour;

        redisTemplate.opsForHash().increment(hourlyKey, "sales", amount.doubleValue());
        redisTemplate.opsForHash().increment(hourlyKey, "orders", 1);

        redisTemplate.expire(hourlyKey, Duration.ofDays(30));
    }

    @Scheduled(cron = "0 5 * * * *") // Every hour at 5 minutes past
    public void aggregateDailyAnalytics() {
        LocalDate yesterday = LocalDate.now().minusDays(1);
        LocalDateTime startOfDay = yesterday.atStartOfDay();
        LocalDateTime endOfDay = yesterday.atTime(23, 59, 59);

        // Fetch all orders from previous day
        // Then aggregate by seller and write to MongoDB
        // (Implementation depends on order service)

        logger.info("Completed daily analytics aggregation for {}", yesterday);
    }
}

@Data
@Document(collection = "seller_daily_analytics")
class SellerDailyAnalytics {
    @Id
    private String id;
    private Long sellerId;
    private LocalDate date;
    private BigDecimal salesRevenue;
    private long orderCount;
    private long pageViews;
    private long conversions;
    private double conversionRate;
    private BigDecimal avgOrderValue;
    private BigDecimal refundAmount;
    private List<ProductAnalytics> topProducts;
}

@Data
class SellerAnalyticsSummary {
    private Long sellerI;
    private LocalDate startDate;
    private LocalDate endDate;
    private BigDecimal totalRevenue;
    private long totalOrders;
    private long totalPageViews;
    private double conversionRate;
    private BigDecimal avgOrderValue;
}

@Data
class ProductAnalytics {
    private Long productId;
    private String name;
    private BigDecimal revenue;
    private long units;
}
```

### 9.5 DisputeResolutionService

```java
@Service
@RequiredArgsConstructor
public class DisputeResolutionService {
    private final MongoTemplate mongoTemplate;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;
    private static final Logger logger = LoggerFactory.getLogger(DisputeResolutionService.class);

    public SellerDispute createDispute(DisputeCreationRequest request) {
        SellerDispute dispute = new SellerDispute();
        dispute.setSellerId(request.getSellerId());
        dispute.setCustomerId(request.getCustomerId());
        dispute.setOrderId(request.getOrderId());
        dispute.setTicketId("DISP-" + System.currentTimeMillis());
        dispute.setCategory(request.getCategory());
        dispute.setSubject(request.getSubject());
        dispute.setDescription(request.getDescription());
        dispute.setStatus(DisputeStatus.OPEN);
        dispute.setPriority(determinePriority(request));
        dispute.setCreatedAt(LocalDateTime.now());
        dispute.setUpdatedAt(LocalDateTime.now());

        Document savedDispute = mongoTemplate.save(dispute, "seller_disputes");

        publishDisputeCreatedEvent(dispute);
        logger.info("Dispute {} created for order {}", dispute.getTicketId(), request.getOrderId());

        return dispute;
    }

    public void submitEvidence(String ticketId, EvidenceSubmissionRequest request) {
        Query query = new Query(Criteria.where("ticket_id").is(ticketId));
        SellerDispute dispute = mongoTemplate.findOne(query, SellerDispute.class, "seller_disputes");

        if (dispute == null) {
            throw new DisputeNotFoundException(ticketId);
        }

        if (!request.getSubmittedBy().equals(dispute.getSellerId()) &&
            !request.getSubmittedBy().equals(dispute.getCustomerId())) {
            throw new UnauthorizedException("User not authorized to submit evidence");
        }

        // Add evidence
        EvidenceItem evidence = new EvidenceItem();
        evidence.setType(request.getEvidenceType());
        evidence.setUrl(request.getUrl()); // S3 URL after upload
        evidence.setUploadedBy(request.getSubmittedBy());
        evidence.setUploadedAt(LocalDateTime.now());

        if (dispute.getEvidence() == null) {
            dispute.setEvidence(new ArrayList<>());
        }
        dispute.getEvidence().add(evidence);

        Update update = new Update()
            .push("evidence", evidence)
            .set("updated_at", LocalDateTime.now());

        mongoTemplate.updateFirst(query, update, "seller_disputes");

        publishDisputeEvidenceSubmittedEvent(dispute, evidence);
        logger.info("Evidence submitted for dispute {}", ticketId);
    }

    public void resolveDispute(String ticketId, DisputeResolutionRequest request) {
        Query query = new Query(Criteria.where("ticket_id").is(ticketId));
        SellerDispute dispute = mongoTemplate.findOne(query, SellerDispute.class, "seller_disputes");

        if (dispute == null) {
            throw new DisputeNotFoundException(ticketId);
        }

        // Determine outcome
        DisputeOutcome outcome = determineOutcome(dispute, request);

        DisputeResolution resolution = new DisputeResolution();
        resolution.setOutcome(outcome);
        resolution.setAmount(request.getAmount());
        resolution.setReason(request.getReason());
        resolution.setDecidedBy(request.getDecidedBy());
        resolution.setDecidedAt(LocalDateTime.now());

        Update update = new Update()
            .set("resolution", resolution)
            .set("status", DisputeStatus.RESOLVED)
            .set("updated_at", LocalDateTime.now());

        mongoTemplate.updateFirst(query, update, "seller_disputes");

        publishDisputeResolvedEvent(dispute, resolution);
        logger.info("Dispute {} resolved with outcome {}", ticketId, outcome);
    }

    private DisputeOutcome determineOutcome(SellerDispute dispute, DisputeResolutionRequest request) {
        // Simple rules (in production, use ML model)
        if (dispute.getEvidence() == null || dispute.getEvidence().isEmpty()) {
            // No evidence from either side, default to customer
            return DisputeOutcome.REFUND;
        }

        long customerEvidenceCount = dispute.getEvidence().stream()
            .filter(e -> e.getUploadedBy().equals(dispute.getCustomerId()))
            .count();

        long sellerEvidenceCount = dispute.getEvidence().stream()
            .filter(e -> e.getUploadedBy().equals(dispute.getSellerId()))
            .count();

        if (customerEvidenceCount > sellerEvidenceCount * 2) {
            return DisputeOutcome.REFUND;
        } else if (sellerEvidenceCount > customerEvidenceCount * 2) {
            return DisputeOutcome.REJECT;
        } else {
            return DisputeOutcome.PARTIAL_REFUND;
        }
    }

    private DisputePriority determinePriority(DisputeCreationRequest request) {
        if (request.getCategory().equals(DisputeCategory.MISSING_ITEMS)) {
            return DisputePriority.HIGH;
        }
        return DisputePriority.NORMAL;
    }

    private void publishDisputeCreatedEvent(SellerDispute dispute) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "DISPUTE_CREATED");
            event.put("ticket_id", dispute.getTicketId());
            event.put("seller_id", dispute.getSellerId());
            event.put("order_id", dispute.getOrderId());
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("seller-disputes", dispute.getSellerId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish DisputeCreatedEvent", e);
        }
    }

    private void publishDisputeEvidenceSubmittedEvent(SellerDispute dispute, EvidenceItem evidence) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "DISPUTE_EVIDENCE_SUBMITTED");
            event.put("ticket_id", dispute.getTicketId());
            event.put("seller_id", dispute.getSellerId());
            event.put("evidence_type", evidence.getType());
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("seller-disputes", dispute.getSellerId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish DisputeEvidenceSubmittedEvent", e);
        }
    }

    private void publishDisputeResolvedEvent(SellerDispute dispute, DisputeResolution resolution) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "DISPUTE_RESOLVED");
            event.put("ticket_id", dispute.getTicketId());
            event.put("seller_id", dispute.getSellerId());
            event.put("outcome", resolution.getOutcome());
            event.put("amount", resolution.getAmount());
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("seller-disputes", dispute.getSellerId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish DisputeResolvedEvent", e);
        }
    }
}

@Data
@Document(collection = "seller_disputes")
class SellerDispute {
    @Id
    private ObjectId id;
    private Long sellerId;
    private Long customerId;
    private Long orderId;
    private String ticketId;
    private DisputeCategory category;
    private String subject;
    private String description;
    private DisputeStatus status;
    private DisputePriority priority;
    private List<EvidenceItem> evidence;
    private DisputeResolution resolution;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

@Data
class EvidenceItem {
    private String type; // IMAGE, VIDEO, MESSAGE, RECEIPT
    private String url;
    private Long uploadedBy;
    private LocalDateTime uploadedAt;
}

@Data
class DisputeResolution {
    private DisputeOutcome outcome;
    private BigDecimal amount;
    private String reason;
    private String decidedBy;
    private LocalDateTime decidedAt;
}

enum DisputeCategory {
    QUALITY_ISSUE, MISSING_ITEMS, DAMAGED, WRONG_ITEM, NOT_AS_DESCRIBED
}

enum DisputeStatus {
    OPEN, ASSIGNED, IN_REVIEW, RESOLVED, CLOSED
}

enum DisputePriority {
    LOW, NORMAL, HIGH, URGENT
}

enum DisputeOutcome {
    REFUND, REJECT, PARTIAL_REFUND
}
```

---

## 10. Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| **Document Verification API Down** | Sellers can't get approved | Queue verification requests in Kafka; replay when service recovers |
| **CSV Upload Job Fails Mid-Stream** | Partial product import | Use Spring Batch checkpoints; resume from last committed chunk |
| **Payout Processing Timeout** | Late payouts, seller frustration | Implement circuit breaker (Resilience4j), retry with exponential backoff, DLQ for failed payouts |
| **MongoDB Analytics Lag** | Stale analytics dashboard | Accept eventual consistency, display cached data with timestamp warning |
| **S3 Document Upload Fails** | Seller can't submit documents | Implement S3 retry policy (3 exponential backoffs), fallback to local storage temporarily |
| **Kafka Consumer Lag Spike** | Delayed events (orders, analytics) | Monitor consumer group lag, autoscale consumer instances, increase partition count |
| **Payment Gateway Rejection** | Payout fails, seller escalates | Validate bank account before payout, retry with different payment method, notify seller with clear reason |
| **Data Isolation Breach** | Seller sees other seller's data | SQL injection protection, parameterized queries, audit every data access, automated security scanning |

---

## 11. Scaling Strategy

### Horizontal Scaling
- **Seller Service**: Stateless microservice, scale to 10 instances (k8s HPA based on CPU)
- **Product Service**: 20 instances for batch processing, hot product cache in Redis
- **Analytics Service**: 10 instances, Kafka consumer group auto-scaling
- **Payout Service**: 3–5 instances (payout processing is batch, not high-throughput)

### Database Scaling
- **PostgreSQL**:
  - Read replicas for analytics queries
  - Sharding on `seller_id` for products table (100K sellers → 10 shards)
  - Connection pooling (HikariCP, 20 connections/instance)
- **MongoDB**: Replica set (3+ nodes), sharding on `seller_id` for analytics
- **Redis**: Cluster mode (6+ nodes), auto-failover

### Kafka Scaling
- Increase partition count per topic to match number of microservice instances
- Monitor consumer lag; scale consumer instances if lag > 10K messages

### Batch Job Scaling
- Spring Batch job runs on single node (avoid parallel processing to prevent duplicate products)
- Use external job scheduler (Quartz, APScheduler) for fault tolerance

---

## 12. Monitoring & Observability

### Key Metrics
- **Seller Registration**: Registration rate, verification success rate, avg verification time
- **Product Uploads**: CSV size, processing time, error rate per seller
- **Orders**: Order confirmation rate, fulfillment time, return rate per seller
- **Payouts**: Payout success rate, avg payout amount, processing delay
- **Analytics**: Query latency (P50, P99), MongoDB aggregation time, cache hit rate
- **Disputes**: Dispute creation rate, resolution time, customer satisfaction (CSAT)

### Logging & Tracing
```yaml
# application.yml
logging:
  level:
    root: INFO
    com.marketplace.seller: DEBUG

spring:
  application:
    name: seller-service

management:
  tracing:
    sampling.probability: 0.1  # 10% trace sampling
  endpoints.web.exposure.include: metrics,health,info,traces

resilience4j:
  circuitbreaker:
    instances:
      payment-gateway:
        register-health-indicator: true
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10000
        permitted-number-of-calls-in-half-open-state: 3
```

### Alerts
- Verification API latency > 10s → Page on-call
- Payout processing failure rate > 5% → Critical alert
- Kafka consumer lag > 100K messages → Scale consumer group
- MongoDB aggregation query > 5s → Investigate indexing

---

## 13. Summary Cheat Sheet

```
SELLER ONBOARDING (5 Steps)
1. Registration → Save seller + generate JWT
2. Submit documents → Async verification via Kafka
3. Verification → Third-party API + manual review
4. Approval → Publish SellerApprovedEvent
5. Dashboard access → View products, orders, analytics

PRODUCT MANAGEMENT
• Bulk upload: Spring Batch, 1000-row chunks, upsert by SKU
• Hot products cached in Redis (24-hour TTL)
• Elasticsearch for full-text search
• Inventory tracked in Redis (5-minute TTL)

PAYOUT PROCESSING
• Weekly job every Monday 2 AM
• Aggregate DELIVERED orders (3+ days old)
• Deduct platform fees (15%), payment processing (2.9%)
• Minimum $100 threshold
• Payment gateway integration with retry/circuit breaker

ANALYTICS
• CQRS: Write to PostgreSQL, async aggregate to MongoDB
• Hourly buckets in Redis for real-time dashboard
• Kafka consumer aggregates events
• MongoDB snapshot per seller per day

DISPUTE RESOLUTION
• Evidence-based arbitration
• ML model determines outcome (simple rules as fallback)
• Escalation after 14 days
• Appeal within 7 days
• All stored in MongoDB with audit trail

DATA ISOLATION
• tenant_id on all tables
• @PreAuthorize("@sellerAuth.owns(#sellerId)") on endpoints
• Encryption for sensitive fields (AES-256)
• Audit logs track all access
```

