---
title: Multi-Tenant Marketplace Platform
layout: default
---

# Multi-Tenant Marketplace Platform — Deep Dive Design

## Scenario

You're building a marketplace where 100,000 sellers list 10 million products. Customers can buy from multiple sellers in a single order, with each seller managing their own inventory, pricing, and shipping. The platform takes 15% commission on each transaction and handles weekly payouts to sellers. Each seller has access to an analytics dashboard showing their sales, inventory, and customer feedback. You must ensure one seller's bad data or traffic spike doesn't affect others (tenant isolation), and support high-volume concurrent orders.

**Tech Stack:** Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka

---

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
- Sellers can list products with pricing, inventory, shipping
- Customers purchase from multiple sellers in one order
- Platform collects full payment, settles with sellers weekly
- Each seller has separate ledger and payout schedule
- Platform takes 15% commission
- Sellers access analytics dashboards
- Row-level security ensures seller sees only their data
- Inventory consistency across concurrent orders

### Non-Functional Requirements
- Support 100K sellers, 10M products, 10M customers
- 200K transactions per second (peak)
- Order split/settlement in < 1 second
- Dashboard queries return in < 2 seconds
- 99.99% availability
- Tenant isolation: no cross-tenant data leakage
- Linear scaling with seller count

### Constraints
- Payment collection is atomic across all sellers
- Seller payout is eventual consistent (settled weekly)
- One seller's inventory update doesn't lock others
- Commission calculation is immutable (audit trail)

---

## Capacity Estimation

| Metric | Value | Calculation |
|--------|-------|-------------|
| **Sellers** | 100K | Given |
| **Products** | 10M | 100 avg per seller |
| **Customers** | 10M | Given |
| **Daily Orders** | 10M | ~100 orders per second avg |
| **Peak TPS** | 200K | 10M / 50 seconds |
| **Items per Order (avg)** | 3 | 2-3 sellers per order |
| **Sub-orders/day** | 30M | 10M orders × 3 |
| **PostgreSQL Rows/Year** | 3.6B | Orders + sub-orders + ledger |
| **MongoDB Analytics Docs** | 3.65M | 100K sellers × 365 days |
| **Redis Memory** | 100GB | Inventory cache + seller sessions |

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────┐
│             API Gateway + Auth (OAuth2)              │
└────┬─────────────────────────────────────────────┬───┘
     │                                             │
┌────▼──────────────────────┐    ┌────────────────▼──┐
│   Order Service           │    │  Seller Service    │
│  (checkout, split)        │    │  (list products)   │
└────┬──────────────────────┘    └────────────────────┘
     │
     ├─ TenantContext (ThreadLocal)
     │  - seller_id / customer_id
     │  - tenant_id for all queries
     │
┌────▼─────────────────────────────────────────────────┐
│           Kafka Event Bus                            │
│  OrderCreated, SellerSubOrderCreated,               │
│  PaymentProcessed, PayoutScheduled                   │
└────┬──────────────────┬──────────────┬────────────────┘
     │                  │              │
┌────▼────┐  ┌─────────▼──┐  ┌────────▼────┐
│PostgreSQL│  │  MongoDB   │  │   Redis     │
│(OLTP)    │  │ (Analytics)│  │(Cache,Queue)│
│          │  │            │  │             │
│Orders    │  │Seller      │  │Inventory    │
│Sub-orders│  │Metrics     │  │Stock        │
│Ledger    │  │Dashboard   │  │Sessions     │
│Inventory │  │Aggregates  │  │             │
└──────────┘  └────────────┘  └─────────────┘

┌──────────────────────────────────────────┐
│   Scheduled Jobs                         │
│  - Payout calculation (weekly)           │
│  - Analytics aggregation (daily)         │
│  - Inventory sync (hourly)               │
└──────────────────────────────────────────┘
```

---

## Core Design Questions Answered

### 1. How do you structure the multi-tenant architecture (database per tenant vs shared)?

**Answer:** Use **shared database with row-level security (RLS)**. All tables have `tenant_id` column, and PostgreSQL RLS policies enforce seller sees only their own data. This minimizes operational complexity while maintaining isolation.

**Why not database-per-tenant:**
- 100K databases = unmanageable complexity
- Shared queries (global product search, admin dashboards) become complex
- Backup/recovery is expensive
- Compliance (one master ledger for audit) is easier with single database

**Implementation:**
```sql
CREATE POLICY seller_isolation ON orders
  FOR SELECT USING (
    seller_id = CURRENT_SETTING('app.current_seller_id')::BIGINT
  );

ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
```

In Java, use `TenantContextHolder` (ThreadLocal) to set `app.current_seller_id` on each request.

---

### 2. How do you handle orders with items from multiple sellers?

**Answer:** Master-detail pattern:
1. Create one **Master Order** (links customer, payment, total)
2. Create N **Seller Sub-Orders** (one per seller in the order)
3. Each sub-order has own fulfillment, tracking, and settlement
4. Payment is atomic across all sub-orders; settlement is eventual

**Atomicity guarantee:** All sub-orders created in same database transaction. If one sub-order creation fails, entire order is rolled back.

---

### 3. How do you split payments between sellers and platform?

**Answer:** **Ledger-based payment splitting** using a saga pattern:

1. **PaymentProcessed event** → Platform receives full amount in escrow account
2. **Weekly batch job** calculates seller payouts:
   - For each seller: SUM(sub-order totals) × (1 - 0.15 commission)
   - Create payout record in ledger
3. **PayoutScheduled event** → Trigger payout to seller's bank account
4. **PayoutCompleted event** → Seller balance updated

**Ledger tables:**
```sql
CREATE TABLE seller_ledger (
  id BIGSERIAL PRIMARY KEY,
  seller_id BIGINT NOT NULL,
  order_id BIGINT,
  transaction_type VARCHAR(50), -- SALE, COMMISSION_DEBIT, PAYOUT
  amount DECIMAL(12,2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_seller_id (seller_id)
);
```

---

### 4. How do you prevent one seller's bad data from affecting others?

**Answer:** Multiple layers:

1. **Row-Level Security (PostgreSQL RLS):** Seller queries filtered by `tenant_id`
2. **Resource quotas:** Limit per-seller:
   - Max products: 100K
   - Max API calls per minute: 10K
   - Max concurrent connections: 100
3. **Query isolation:** Each seller's queries run in separate connection pool
4. **Separate indices:** Create indices per high-volume table to avoid lock contention
5. **Circuit breaker:** If seller's service degrades, fail fast instead of cascading

**Implementation:**
```java
@Aspect
public class TenantIsolationAspect {
    @Before("@annotation(com.retail.tenant.TenantIsolated)")
    public void enforceTenantContext(JoinPoint joinPoint) {
        Long tenantId = TenantContextHolder.getTenantId();
        if (tenantId == null) {
            throw new TenantContextException("Tenant context not set");
        }
        // Continue with execution
    }
}
```

---

### 5. How do you provide seller-specific analytics without slow queries?

**Answer:** **Pre-aggregation strategy** with MongoDB:

1. **Streaming aggregation:** Kafka consumer listens to `OrderCreated` events
2. **Daily snapshots:** Every 24h, compute aggregates for each seller (revenue, order count, etc.)
3. **Store in MongoDB:** Pre-aggregated documents (denormalized, read-optimized)
4. **Real-time cache:** For current day metrics, use Redis counters

**MongoDB document structure:**
```json
{
  "_id": "seller_123_2026-04-01",
  "seller_id": 123,
  "date": "2026-04-01",
  "metrics": {
    "total_revenue": 12500.00,
    "total_commission": 1875.00,
    "order_count": 350,
    "unique_customers": 280,
    "average_order_value": 35.71,
    "top_products": [...],
    "customer_repeat_rate": 0.65
  },
  "inventory": {
    "total_products": 500,
    "out_of_stock_count": 25,
    "low_stock_count": 80
  }
}
```

Queries against this collection are instant (no joins, pre-computed).

---

### 6. How do you scale to support 100K sellers?

**Answer:** Multi-layered scaling:

1. **PostgreSQL sharding by seller_id mod N:** Distribute large tables across shards
2. **Read replicas:** Use replicas for analytics queries
3. **Connection pooling:** PgBouncer with max 100 connections per shard
4. **Caching hierarchy:**
   - L1: Redis (inventory, session data)
   - L2: PostgreSQL with materialized views
   - L3: MongoDB (analytics, slow-changing data)
5. **Microservices isolation:** Seller analytics service ≠ Order service
6. **Asynchronous processing:** Kafka decouples order processing from payout calculation

---

## Microservices Breakdown

| Service | Responsibility | Data Owned |
|---------|-----------------|-----------|
| **Order Service** | Create orders, split sellers, manage sub-orders | Orders, sub-orders, inventory reservations |
| **Seller Service** | Seller onboarding, product listing, inventory | Products, seller metadata, inventory |
| **Payment Service** | Charge customer, escrow, payout orchestration | Payment records, ledger, payout schedule |
| **Analytics Service** | Aggregate metrics, dashboard queries | Pre-aggregated metrics (MongoDB) |
| **Inventory Service** | Manage stock levels, reservations, allocation | Inventory ledger, reservations |
| **Settlement Service** | Weekly payout calculations | Seller ledger, payout records |

---

## Database Design

### PostgreSQL: OLTP (Transactional)

```sql
-- Master orders (one per customer transaction)
CREATE TABLE orders (
  id BIGSERIAL PRIMARY KEY,
  customer_id BIGINT NOT NULL,
  tenant_id BIGINT NOT NULL,  -- for multi-tenancy
  order_number VARCHAR(50) UNIQUE NOT NULL,
  total_amount DECIMAL(12,2) NOT NULL,
  currency VARCHAR(3) DEFAULT 'USD',
  status VARCHAR(50), -- PENDING, PAID, FULFILLED, CANCELLED
  payment_status VARCHAR(50),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_tenant_customer (tenant_id, customer_id),
  INDEX idx_status (status)
);

-- Sub-orders (one per seller per order)
CREATE TABLE seller_sub_orders (
  id BIGSERIAL PRIMARY KEY,
  order_id BIGINT NOT NULL,
  seller_id BIGINT NOT NULL,
  tenant_id BIGINT NOT NULL,
  sub_order_number VARCHAR(50) UNIQUE NOT NULL,
  sub_total DECIMAL(12,2) NOT NULL,
  commission_amount DECIMAL(12,2) NOT NULL,
  seller_payout DECIMAL(12,2) NOT NULL,
  status VARCHAR(50), -- PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
  fulfillment_status VARCHAR(50),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (order_id) REFERENCES orders(id),
  INDEX idx_tenant_seller (tenant_id, seller_id),
  INDEX idx_seller_id (seller_id)
);

-- Order items (line items per sub-order)
CREATE TABLE order_items (
  id BIGSERIAL PRIMARY KEY,
  sub_order_id BIGINT NOT NULL,
  product_id BIGINT NOT NULL,
  tenant_id BIGINT NOT NULL,
  quantity INT NOT NULL,
  unit_price DECIMAL(12,2) NOT NULL,
  subtotal DECIMAL(12,2) NOT NULL,
  FOREIGN KEY (sub_order_id) REFERENCES seller_sub_orders(id),
  INDEX idx_tenant_product (tenant_id, product_id)
);

-- Seller products
CREATE TABLE products (
  id BIGSERIAL PRIMARY KEY,
  seller_id BIGINT NOT NULL,
  tenant_id BIGINT NOT NULL,
  sku VARCHAR(100) UNIQUE NOT NULL,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  price DECIMAL(12,2) NOT NULL,
  currency VARCHAR(3) DEFAULT 'USD',
  status VARCHAR(50), -- ACTIVE, INACTIVE, DELISTED
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP,
  INDEX idx_tenant_seller (tenant_id, seller_id),
  INDEX idx_sku (sku)
);

-- Inventory (stock tracking per seller per product)
CREATE TABLE inventory (
  id BIGSERIAL PRIMARY KEY,
  product_id BIGINT NOT NULL,
  seller_id BIGINT NOT NULL,
  tenant_id BIGINT NOT NULL,
  quantity_on_hand INT NOT NULL DEFAULT 0,
  quantity_reserved INT NOT NULL DEFAULT 0,
  quantity_available INT GENERATED ALWAYS AS (quantity_on_hand - quantity_reserved),
  last_updated TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (product_id) REFERENCES products(id),
  UNIQUE(product_id, seller_id),
  INDEX idx_tenant_seller (tenant_id, seller_id)
);

-- Seller ledger (for payout tracking)
CREATE TABLE seller_ledger (
  id BIGSERIAL PRIMARY KEY,
  seller_id BIGINT NOT NULL,
  tenant_id BIGINT NOT NULL,
  sub_order_id BIGINT,
  transaction_type VARCHAR(50), -- SALE, COMMISSION_DEBIT, PAYOUT, REFUND
  amount DECIMAL(12,2) NOT NULL,
  balance DECIMAL(12,2) NOT NULL,
  payout_id BIGINT,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  INDEX idx_tenant_seller (tenant_id, seller_id),
  INDEX idx_seller_created (seller_id, created_at)
);

-- Row-Level Security policies
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;
CREATE POLICY orders_tenant_policy ON orders
  FOR ALL USING (tenant_id = CURRENT_SETTING('app.current_tenant_id')::BIGINT);

ALTER TABLE products ENABLE ROW LEVEL SECURITY;
CREATE POLICY products_seller_policy ON products
  FOR ALL USING (seller_id = CURRENT_SETTING('app.current_seller_id')::BIGINT);
```

### MongoDB: Analytics (OLAP)

```json
{
  "_id": "seller_123_2026-04-01",
  "seller_id": 123,
  "date": "2026-04-01",
  "metrics": {
    "revenue": {
      "gross": 12500.00,
      "commission": 1875.00,
      "net_payout": 10625.00
    },
    "orders": {
      "total_count": 350,
      "average_value": 35.71,
      "item_count": 1050
    },
    "customers": {
      "unique_count": 280,
      "repeat_customers": 182,
      "repeat_rate": 0.65
    },
    "inventory": {
      "total_products": 500,
      "active_products": 450,
      "out_of_stock": 25,
      "low_stock": 80
    },
    "top_products": [
      { "product_id": 1001, "name": "Widget A", "quantity_sold": 150, "revenue": 4500 },
      { "product_id": 1002, "name": "Widget B", "quantity_sold": 120, "revenue": 3600 }
    ]
  },
  "created_at": "2026-04-01T00:00:00Z"
}
```

---

## Redis Data Structures

```
# Inventory cache (real-time stock levels)
inventory:{product_id}:{seller_id} → HASH {
  quantity_on_hand: 100,
  quantity_reserved: 20,
  quantity_available: 80,
  updated_at: 2026-04-01T10:00:00Z
}

# Seller session (authenticated requests)
session:{session_id} → HASH {
  seller_id: 123,
  customer_id: 456,
  permissions: ["list_products", "view_analytics"],
  created_at: 2026-04-01T08:00:00Z
}
seller:session:{seller_id} → SET (list of active session IDs)

# Rate limiting (per seller)
rate_limit:{seller_id}:{minute} → INT (request count)
rate_limit:{seller_id}:{minute}:ttl → TTL of 60s

# Payout status (tracking)
payout:{seller_id}:{week} → HASH {
  total_amount: 10625.00,
  status: "SCHEDULED",
  scheduled_date: 2026-04-07,
  completed_date: null
}

# Current day metrics (real-time, before daily aggregation)
metrics:daily:{seller_id} → HASH {
  orders: 50,
  revenue: 1500.00,
  customers: 45
}
```

---

## Kafka Event Flow

```
Topics:
1. orders (3 partitions, key=seller_id) → OrderCreated, OrderCancelled
2. inventory (3 partitions, key=product_id) → InventoryReserved, InventoryReleased
3. payments (1 partition, ordered) → PaymentProcessed, PaymentFailed
4. settlements (1 partition, ordered) → PayoutScheduled, PayoutCompleted

Event Consumers:
- OrderService → OrderCreated (create sub-orders, reserve inventory)
- InventoryService → OrderCreated (check stock, allocate)
- PaymentService → OrderCreated (charge customer)
- AnalyticsService → OrderCreated (update daily metrics)
- SettlementService → PaymentProcessed (calculate payouts)
```

---

## Implementation Code

### 1. Tenant Context Holder

```java
package com.retail.marketplace.tenant;

public class TenantContextHolder {
    private static final ThreadLocal<Long> tenantIdHolder = new ThreadLocal<>();
    private static final ThreadLocal<Long> sellerIdHolder = new ThreadLocal<>();
    private static final ThreadLocal<Long> customerIdHolder = new ThreadLocal<>();

    public static void setTenantId(Long tenantId) {
        tenantIdHolder.set(tenantId);
    }

    public static Long getTenantId() {
        return tenantIdHolder.get();
    }

    public static void setSellerId(Long sellerId) {
        sellerIdHolder.set(sellerId);
    }

    public static Long getSellerId() {
        return sellerIdHolder.get();
    }

    public static void setCustomerId(Long customerId) {
        customerIdHolder.set(customerId);
    }

    public static Long getCustomerId() {
        return customerIdHolder.get();
    }

    public static void clear() {
        tenantIdHolder.remove();
        sellerIdHolder.remove();
        customerIdHolder.remove();
    }
}

@Component
public class TenantContextFilter implements Filter {
    @Override
    public void doFilter(ServletRequest request, ServletResponse response,
                        FilterChain chain) throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;

        try {
            // Extract tenant from JWT or header
            Long tenantId = extractTenantFromJWT(httpRequest);
            Long sellerId = extractSellerFromJWT(httpRequest);
            Long customerId = extractCustomerFromJWT(httpRequest);

            TenantContextHolder.setTenantId(tenantId);
            TenantContextHolder.setSellerId(sellerId);
            TenantContextHolder.setCustomerId(customerId);

            // Set PostgreSQL session variable for RLS
            if (tenantId != null) {
                DataSource dataSource = (DataSource) httpRequest
                    .getServletContext().getAttribute("dataSource");
                try (Connection conn = dataSource.getConnection()) {
                    conn.createStatement()
                        .execute("SET app.current_tenant_id = " + tenantId);
                }
            }

            chain.doFilter(request, response);

        } finally {
            TenantContextHolder.clear();
        }
    }

    private Long extractTenantFromJWT(HttpServletRequest request) {
        // Implementation to extract from JWT token
        return 1L;  // Default tenant
    }

    private Long extractSellerFromJWT(HttpServletRequest request) {
        // Implementation to extract from JWT token
        return null;
    }

    private Long extractCustomerFromJWT(HttpServletRequest request) {
        // Implementation to extract from JWT token
        return null;
    }
}
```

### 2. Order Service with Sub-Order Splitting

```java
package com.retail.marketplace.service;

import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.kafka.core.KafkaTemplate;
import java.math.BigDecimal;
import java.util.*;
import java.util.stream.Collectors;

@Service
public class OrderService {

    private static final Logger log = LoggerFactory.getLogger(OrderService.class);
    private static final BigDecimal COMMISSION_RATE = BigDecimal.valueOf(0.15);

    private final OrderRepository orderRepository;
    private final SellerSubOrderRepository subOrderRepository;
    private final OrderItemRepository itemRepository;
    private final InventoryService inventoryService;
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    private final ObjectMapper objectMapper;

    @Transactional
    public CreateOrderResponse createOrder(CreateOrderRequest request) {
        Long customerId = TenantContextHolder.getCustomerId();
        Long tenantId = TenantContextHolder.getTenantId();

        try {
            // 1. Create master order
            Order masterOrder = new Order();
            masterOrder.setCustomerId(customerId);
            masterOrder.setTenantId(tenantId);
            masterOrder.setOrderNumber(generateOrderNumber());
            masterOrder.setTotalAmount(BigDecimal.ZERO);
            masterOrder.setStatus("PENDING");
            masterOrder.setPaymentStatus("PENDING");

            Order savedOrder = orderRepository.save(masterOrder);

            // 2. Group items by seller
            Map<Long, List<CreateOrderRequest.CartItem>> itemsByS eller =
                request.getCartItems().stream()
                    .collect(Collectors.groupingBy(item -> item.getSellerId()));

            BigDecimal totalAmount = BigDecimal.ZERO;
            List<SellerSubOrder> subOrders = new ArrayList<>();

            // 3. Create sub-order for each seller
            for (Map.Entry<Long, List<CreateOrderRequest.CartItem>> entry :
                 itemsBySeller.entrySet()) {
                Long sellerId = entry.getKey();
                List<CreateOrderRequest.CartItem> sellerItems = entry.getValue();

                // Reserve inventory
                for (CreateOrderRequest.CartItem item : sellerItems) {
                    boolean reserved = inventoryService.reserveInventory(
                        item.getProductId(), sellerId, item.getQuantity()
                    );
                    if (!reserved) {
                        throw new InsufficientInventoryException(
                            "Product " + item.getProductId() + " out of stock"
                        );
                    }
                }

                // Calculate subtotal and commission
                BigDecimal subtotal = sellerItems.stream()
                    .map(item -> item.getUnitPrice()
                        .multiply(BigDecimal.valueOf(item.getQuantity())))
                    .reduce(BigDecimal.ZERO, BigDecimal::add);

                BigDecimal commission = subtotal.multiply(COMMISSION_RATE);
                BigDecimal sellerPayout = subtotal.subtract(commission);

                // Create sub-order
                SellerSubOrder subOrder = new SellerSubOrder();
                subOrder.setOrderId(savedOrder.getId());
                subOrder.setSellerId(sellerId);
                subOrder.setTenantId(tenantId);
                subOrder.setSubOrderNumber(generateSubOrderNumber(sellerId));
                subOrder.setSubTotal(subtotal);
                subOrder.setCommissionAmount(commission);
                subOrder.setSellerPayout(sellerPayout);
                subOrder.setStatus("PENDING");
                subOrder.setFulfillmentStatus("AWAITING_FULFILLMENT");

                SellerSubOrder savedSubOrder = subOrderRepository.save(subOrder);

                // Create order items
                for (CreateOrderRequest.CartItem item : sellerItems) {
                    OrderItem orderItem = new OrderItem();
                    orderItem.setSubOrderId(savedSubOrder.getId());
                    orderItem.setProductId(item.getProductId());
                    orderItem.setTenantId(tenantId);
                    orderItem.setQuantity(item.getQuantity());
                    orderItem.setUnitPrice(item.getUnitPrice());
                    orderItem.setSubtotal(item.getUnitPrice()
                        .multiply(BigDecimal.valueOf(item.getQuantity())));

                    itemRepository.save(orderItem);
                }

                subOrders.add(savedSubOrder);
                totalAmount = totalAmount.add(subtotal);

                // Publish OrderCreated event per seller
                OrderCreated event = new OrderCreated(
                    savedOrder.getId(),
                    savedSubOrder.getId(),
                    customerId,
                    sellerId,
                    subtotal,
                    commission
                );
                kafkaTemplate.send("orders", String.valueOf(sellerId), event);
            }

            // 4. Update master order total
            masterOrder.setTotalAmount(totalAmount);
            orderRepository.save(masterOrder);

            log.info("Created order {} with {} sub-orders for customer {}",
                     savedOrder.getId(), subOrders.size(), customerId);

            return CreateOrderResponse.success(savedOrder.getId(), subOrders.size());

        } catch (InsufficientInventoryException e) {
            log.warn("Order creation failed: {}", e.getMessage());
            // Rollback inventory reservations (transaction will handle)
            throw e;
        } catch (Exception e) {
            log.error("Error creating order", e);
            throw new OrderException("Order creation failed", e);
        }
    }

    @Transactional
    public void cancelOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        // 1. Cancel all sub-orders
        List<SellerSubOrder> subOrders = subOrderRepository.findByOrderId(orderId);
        for (SellerSubOrder subOrder : subOrders) {
            subOrder.setStatus("CANCELLED");
            subOrderRepository.save(subOrder);

            // 2. Release inventory for each sub-order
            List<OrderItem> items = itemRepository.findBySubOrderId(subOrder.getId());
            for (OrderItem item : items) {
                inventoryService.releaseInventory(
                    item.getProductId(),
                    subOrder.getSellerId(),
                    item.getQuantity()
                );
            }
        }

        // 3. Update master order
        order.setStatus("CANCELLED");
        orderRepository.save(order);

        log.info("Cancelled order {}", orderId);
    }

    private String generateOrderNumber() {
        return "ORD-" + System.currentTimeMillis();
    }

    private String generateSubOrderNumber(Long sellerId) {
        return "SUB-" + sellerId + "-" + System.currentTimeMillis();
    }
}
```

### 3. Commission Calculator

```java
package com.retail.marketplace.service;

import java.math.BigDecimal;
import java.math.RoundingMode;

@Service
public class CommissionCalculator {

    private static final BigDecimal PLATFORM_COMMISSION_RATE = BigDecimal.valueOf(0.15);
    private static final BigDecimal TAX_RATE = BigDecimal.valueOf(0.10);

    public CommissionBreakdown calculateCommission(
            Long sellerId,
            BigDecimal orderAmount,
            boolean hasPromoCoupon) {

        // Base commission
        BigDecimal commissionAmount = orderAmount.multiply(PLATFORM_COMMISSION_RATE);

        // Adjustments for premium sellers
        boolean isPremiumSeller = isPremiumSeller(sellerId);
        if (isPremiumSeller) {
            // Premium sellers get 12% commission instead of 15%
            commissionAmount = orderAmount.multiply(BigDecimal.valueOf(0.12));
        }

        // Promo discount reduces commission
        if (hasPromoCoupon) {
            commissionAmount = commissionAmount.multiply(BigDecimal.valueOf(0.9));
        }

        BigDecimal sellerPayout = orderAmount.subtract(commissionAmount);

        // Prepare breakdown with audit trail
        return CommissionBreakdown.builder()
            .orderId(orderId)
            .sellerId(sellerId)
            .grossAmount(orderAmount)
            .commissionRate(isPremiumSeller ? 0.12 : 0.15)
            .commissionAmount(commissionAmount.setScale(2, RoundingMode.HALF_UP))
            .sellerPayout(sellerPayout.setScale(2, RoundingMode.HALF_UP))
            .timestamp(LocalDateTime.now())
            .build();
    }

    private boolean isPremiumSeller(Long sellerId) {
        // Check seller tier/rating in database
        return sellerRepository.findById(sellerId)
            .map(seller -> seller.getRating() >= 4.5)
            .orElse(false);
    }
}
```

### 4. Payout Scheduler

```java
package com.retail.marketplace.job;

import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.math.BigDecimal;
import java.time.LocalDate;

@Service
public class PayoutScheduler {

    private static final Logger log = LoggerFactory.getLogger(PayoutScheduler.class);

    private final SellerSubOrderRepository subOrderRepository;
    private final SellerLedgerRepository ledgerRepository;
    private final SellerRepository sellerRepository;
    private final KafkaTemplate<String, PayoutEvent> kafkaTemplate;

    // Runs every Monday at 9 AM
    @Scheduled(cron = "0 9 * * 1")
    @Transactional
    public void scheduleWeeklyPayouts() {
        log.info("Starting weekly payout scheduling");

        try {
            // 1. Get all sellers with pending earnings
            List<Long> sellerIds = ledgerRepository.findSellersWithPendingEarnings();

            for (Long sellerId : sellerIds) {
                calculateAndSchedulePayout(sellerId);
            }

            log.info("Completed weekly payout scheduling for {} sellers", sellerIds.size());

        } catch (Exception e) {
            log.error("Error in weekly payout scheduling", e);
            // Alert operations team
            throw new PayoutException("Payout scheduling failed", e);
        }
    }

    @Transactional
    private void calculateAndSchedulePayout(Long sellerId) {
        // 1. Sum all earnings from last 7 days
        BigDecimal totalEarnings = ledgerRepository
            .sumEarningsBySeller(sellerId, LocalDate.now().minusDays(7));

        if (totalEarnings.compareTo(BigDecimal.ZERO) <= 0) {
            return;  // No earnings to payout
        }

        // 2. Deduct any chargebacks, refunds
        BigDecimal chargebacks = ledgerRepository
            .sumChargebacksBySeller(sellerId, LocalDate.now().minusDays(30));
        BigDecimal refunds = ledgerRepository
            .sumRefundsBySeller(sellerId, LocalDate.now().minusDays(7));

        BigDecimal netPayout = totalEarnings
            .subtract(chargebacks)
            .subtract(refunds);

        if (netPayout.compareTo(BigDecimal.ZERO) <= 0) {
            log.warn("Seller {} has negative or zero payout", sellerId);
            return;
        }

        // 3. Create payout record
        Payout payout = new Payout();
        payout.setSellerId(sellerId);
        payout.setPayoutDate(LocalDate.now().plusDays(7));  // Payout next week
        payout.setAmount(netPayout);
        payout.setStatus("SCHEDULED");
        payout.setPaymentMethod(getSellerPaymentMethod(sellerId));

        Payout savedPayout = payoutRepository.save(payout);

        // 4. Record in ledger
        SellerLedger ledger = new SellerLedger();
        ledger.setSellerId(sellerId);
        ledger.setTransactionType("PAYOUT");
        ledger.setAmount(netPayout.negate());  // Negative because it's an outflow
        ledger.setPayoutId(savedPayout.getId());
        ledgerRepository.save(ledger);

        // 5. Publish PayoutScheduled event
        PayoutScheduled event = new PayoutScheduled(
            savedPayout.getId(),
            sellerId,
            netPayout,
            savedPayout.getPayoutDate()
        );
        kafkaTemplate.send("settlements", String.valueOf(sellerId), event);

        log.info("Scheduled payout of {} for seller {}", netPayout, sellerId);
    }

    private String getSellerPaymentMethod(Long sellerId) {
        return sellerRepository.findById(sellerId)
            .map(Seller::getPaymentMethod)
            .orElse("BANK_TRANSFER");
    }
}
```

### 5. Seller Analytics Service

```java
package com.retail.marketplace.service;

import org.springframework.stereotype.Service;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.data.mongodb.core.MongoTemplate;
import java.time.LocalDate;

@Service
public class SellerAnalyticsService {

    private static final Logger log = LoggerFactory.getLogger(SellerAnalyticsService.class);

    private final MongoTemplate mongoTemplate;
    private final RedisTemplate<String, String> redisTemplate;
    private final OrderItemRepository orderItemRepository;

    // Consume OrderCreated events and update real-time metrics
    @KafkaListener(topics = "orders", groupId = "analytics-service")
    public void handleOrderCreated(OrderCreated event) {
        try {
            Long sellerId = event.getSellerId();
            LocalDate today = LocalDate.now();

            // 1. Update real-time Redis counters
            String metricsKey = "metrics:daily:" + sellerId;
            redisTemplate.opsForHash().increment(metricsKey, "orders", 1);
            redisTemplate.opsForHash().increment(
                metricsKey,
                "revenue",
                event.getOrderAmount().doubleValue()
            );

            // 2. Append to MongoDB daily snapshot (will be aggregated at EOD)
            DailyMetricSnapshot snapshot = new DailyMetricSnapshot();
            snapshot.setSellerId(sellerId);
            snapshot.setDate(today);
            snapshot.setOrderId(event.getOrderId());
            snapshot.setRevenue(event.getOrderAmount());
            snapshot.setCommission(event.getCommission());
            snapshot.setNetPayout(event.getOrderAmount().subtract(event.getCommission()));

            mongoTemplate.insert(snapshot, "seller_daily_metrics");

            log.debug("Updated metrics for seller {}", sellerId);

        } catch (Exception e) {
            log.error("Error updating seller analytics", e);
        }
    }

    // Runs daily at 11 PM to aggregate final metrics
    @Scheduled(cron = "0 23 * * *")
    @Transactional
    public void aggregateDailyMetrics() {
        log.info("Starting daily metrics aggregation");

        try {
            List<Long> sellers = sellerRepository.findAll()
                .stream()
                .map(Seller::getId)
                .collect(Collectors.toList());

            LocalDate today = LocalDate.now();

            for (Long sellerId : sellers) {
                // 1. Sum up all raw metrics from MongoDB
                DailyAggregated aggregated = mongoTemplate.aggregate(
                    Aggregation.newAggregation(
                        Aggregation.match(Criteria.where("sellerId").is(sellerId)
                            .and("date").is(today)),
                        Aggregation.group().count().as("orderCount")
                            .sum("revenue").as("totalRevenue")
                            .sum("commission").as("totalCommission")
                    ),
                    "seller_daily_metrics",
                    DailyAggregated.class
                ).getUniqueMappedResult();

                if (aggregated != null) {
                    // 2. Fetch top products
                    List<TopProduct> topProducts = orderItemRepository
                        .findTopProductsBySeller(sellerId, today, 10);

                    // 3. Fetch unique customer count
                    Long uniqueCustomers = orderRepository
                        .countUniqueCustomersBySeller(sellerId, today);

                    // 4. Create final document
                    SellerDailySnapshot finalSnapshot = new SellerDailySnapshot();
                    finalSnapshot.setId(sellerId + "_" + today);
                    finalSnapshot.setSellerId(sellerId);
                    finalSnapshot.setDate(today);
                    finalSnapshot.setMetrics(Map.of(
                        "revenue", aggregated.getTotalRevenue(),
                        "commission", aggregated.getTotalCommission(),
                        "netPayout", aggregated.getTotalRevenue()
                            .subtract(aggregated.getTotalCommission()),
                        "orders", aggregated.getOrderCount(),
                        "customers", uniqueCustomers,
                        "topProducts", topProducts
                    ));

                    mongoTemplate.save(finalSnapshot, "seller_metrics_daily");
                }
            }

            log.info("Completed daily metrics aggregation");

        } catch (Exception e) {
            log.error("Error in daily metrics aggregation", e);
        }
    }

    public SellerAnalyticsDashboard getDashboard(Long sellerId, LocalDate startDate,
                                                   LocalDate endDate) {
        Long currentSellerId = TenantContextHolder.getSellerId();
        if (!sellerId.equals(currentSellerId)) {
            throw new UnauthorizedException("Cannot view other seller's data");
        }

        // Query pre-aggregated MongoDB collection
        List<SellerDailySnapshot> snapshots = mongoTemplate.find(
            Query.query(Criteria.where("sellerId").is(sellerId)
                .and("date").gte(startDate).lte(endDate)),
            SellerDailySnapshot.class
        );

        // Aggregate across date range
        return SellerAnalyticsDashboard.fromSnapshots(snapshots);
    }
}
```

---

## Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| **One seller's product query slow** | May timeout, not affect other sellers | Query timeouts, circuit breaker, separate connection pool |
| **Inventory reservation race** | Double-selling product | Pessimistic locking + PostgreSQL SERIAL; or optimistic with retry |
| **Sub-order creation fails mid-order** | Partial order, orphaned inventory | Transaction rollback; all sub-orders created atomically |
| **Payment fails but sub-orders created** | Order in inconsistent state | Saga pattern: wait for PaymentProcessed before confirming |
| **Payout calculation includes refunded order** | Seller overpaid | Ledger-based: ledger entries for refunds reduce payout |
| **Seller hits quota limits (100K products)** | Listing blocked | Rate limiting + quota enforcement in service layer |
| **MongoDB analytics behind** | Dashboard shows stale data | Redis for real-time; okay to be eventually consistent |

---

## Scaling Strategy

### Horizontal Scaling

1. **PostgreSQL sharding by seller_id mod N:** Distribute large tables across shards
2. **Read replicas:** Use for analytics, dashboard queries (separate from OLTP replicas)
3. **Kafka partitions:** One partition per seller or per shard
4. **Service instances:** Deploy N Order Service replicas behind load balancer

### Vertical Scaling

1. **Connection pooling:** HikariCP with max 200 connections per service instance
2. **Redis cluster:** Distribute inventory cache across nodes
3. **MongoDB sharding:** Shard by seller_id for analytics (100K sellers = manageable)

### Caching Strategy

- L1: Redis (inventory, session data)
- L2: In-memory cache (seller metadata, commission rates)
- L3: MongoDB (pre-aggregated analytics)

---

## Monitoring & Observability

### Key Metrics

```java
meterRegistry.timer("order.creation.time")
    .record(() -> createOrder(...));

meterRegistry.counter("orders.created.total",
    "seller_tier", sellerTier).increment();

meterRegistry.gauge("inventory.reservation.failures",
    failureCounter::get);
```

### Alerts

- Sub-order creation latency > 500ms
- Inventory reservation failures > 1%
- Payout scheduling delayed > 1 hour
- Seller dashboard queries > 2s

---

## Summary Cheat Sheet

| Component | Choice | Why |
|-----------|--------|-----|
| **Tenant isolation** | Row-Level Security (RLS) + TenantContext | Single DB, enforced isolation at query level |
| **Order splitting** | Master order → N sub-orders | Atomic creation, per-seller settlement |
| **Payment splitting** | Ledger-based with saga | Auditability, easy reconciliation |
| **Analytics** | Pre-aggregated MongoDB snapshots | Fast queries, denormalized reads |
| **Inventory** | Pessimistic lock + Redis cache | Prevents overselling, real-time visibility |
| **Payout scheduling** | Weekly batch job, ledger-based | Eventually consistent, handles refunds |
| **Scaling** | PostgreSQL sharding by seller_id | Linear scaling with seller count |

---

**Generated:** 2026-04-01 | **Tech Stack:** Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka
