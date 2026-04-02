---
title: Store Locator and Pickup System (BOPIS)
layout: default
---

# Store Locator and Pickup System (BOPIS) — Deep Dive Design

## Scenario

Design a Buy Online Pick-up In Store (BOPIS) system where customers search for nearby stores with items in stock, select a store and pickup time, reserve items to prevent double-booking, receive notifications when orders are ready, and store associates can mark items as picked up. Support 2,000 stores, handle item damage or abandonment after 7 days, and coordinate between online and in-store inventory systems.

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
- Search stores near customer location (lat/lng) within radius
- Show real-time inventory at each store for searched product
- Customer selects store and pickup time window
- System reserves inventory at selected store (atomic DECR, prevent oversell)
- Reservation TTL: 7 days; after 7 days, auto-release and restock
- Notify customer when order is ready for pickup (push, SMS, email)
- Store associate app: scan item, mark as "picked up" (no longer in reservation)
- Handle item damage: associate can mark as damaged, inventory adjusted
- Handle customer no-show: 7-day automatic release with notification
- Support pickup time windows (8 AM-10 AM, 10 AM-12 PM, etc.)
- Coordinate with online inventory (prevent overselling across channels)
- Store staff can modify reservations (extend 7-day window, change items)

### Non-Functional Requirements
- **Throughput:** 10,000 pickup orders/day, 200-500 concurrent reservations
- **Read latency:** store search + inventory <300ms, reservation <500ms
- **Write latency:** reservation confirmation <2 seconds
- **Availability:** 99.9% SLA; pickup is critical path
- **Eventual consistency:** inventory eventual across channels
- **Durability:** reservations persisted immediately
- **Scalability:** support 2,000 stores with independent inventory

---

## 2. Capacity Estimation

```
Total Stores: 2,000
Pickup Orders Per Day: 10,000
Peak Orders Per Hour: 2,000 (during morning/evening)
Orders Per Second (peak): ~0.56 orders/sec

Concurrent Active Reservations: ~500-1,000
Reservation TTL: 7 days

Store Inventory Per Store: ~50,000 SKUs with varying quantities
Total Inventory Records: 2,000 stores × 50,000 SKUs = 100M records

PostgreSQL Size:
- Reservations: 10K/day × 7 days = 70K rows × 500 bytes = ~35 MB
- Stores: 2,000 records × 1 KB = 2 MB
- Pickup orders: 10K/day × 30 days = 300K rows × 2 KB = ~600 MB

Redis Size:
- Store inventory cache: 2,000 × 50K × 100 bytes = 10 GB
- Active reservations: 1,000 × 500 bytes = 500 KB
- Reservation locks: 1,000 × 100 bytes = 100 KB
- Store location index (geo): 2,000 × 200 bytes = 400 KB

Kafka Throughput:
- pickup.reservation_created: 0.56/sec
- pickup.ready_for_pickup: 0.56/sec (4-6 hours later)
- pickup.picked_up: 0.56/sec
- pickup.expired: 0.08/sec (every 7 days)
- inventory.updated: varies by store stocking schedule

API Calls:
- Search nearby stores: 50K/day (discovery)
- Check inventory: 50K/day
- Create reservation: 10K/day
- Mark pickup: 10K/day
```

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────┐
│         Customer Web/Mobile App                 │
└──────────────┬──────────────────────────────────┘
               │ REST APIs
        ┌──────┴──────────────────┐
        │                         │
┌───────▼──────────────┐  ┌──────▼──────────────┐
│ Store Locator API    │  │ Pickup             │
│ (search, inventory)  │  │ Reservation API    │
└──────────┬───────────┘  └────────┬────────────┘
           │                       │
           │ PostGIS               │ Distributed
           │ + Redis GEO           │ Lock + Redis
           │                       │ Atomic DECR
           └───────────┬───────────┘
                       │ Events
                ┌──────▼──────────────────┐
                │ Kafka Topics            │
                │ - pickup.reservation    │
                │ - pickup.ready          │
                │ - inventory.updated     │
                └──────┬───────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
┌───────▼────────┐ ┌──▼──────────┐ ┌──▼──────────────┐
│ Notification   │ │ Order       │ │ Reservation     │
│ Service        │ │ Fulfillment │ │ Expiry Job      │
│ (push/SMS)     │ │ Service     │ │ (daily scan)    │
└────────────────┘ └─────────────┘ └────────────────┘

                ┌──────────────────────────────────┐
                │ Store Associate App (Tablet/Web) │
                │ - scan barcode                   │
                │ - mark picked up                 │
                │ - real-time dashboard            │
                └──────────────────┬───────────────┘
                                   │ REST API
                            ┌──────▼──────────┐
                            │ Store Associate │
                            │ Controller      │
                            └────────────────┘

    ┌──────────────────────────────────────┐
    │ PostgreSQL                           │
    │ - stores (name, location, address)   │
    │ - reservations (UUID, store, SKU)    │
    │ - pickup_orders                      │
    │ - store_inventory_snapshot           │
    └──────────────────────────────────────┘

    ┌──────────────────────────────────────┐
    │ MongoDB                              │
    │ - reservation_events (audit trail)   │
    │ - pickup_order_events                │
    │ - store_associate_logs               │
    └──────────────────────────────────────┘

    ┌──────────────────────────────────────┐
    │ Redis                                │
    │ - store locations (GEOHASH)          │
    │ - inventory per store:SKU (HASH)     │
    │ - active reservations (ZSET by TTL)  │
    │ - reservation locks (STRING)         │
    │ - WebSocket connections (store app)  │
    └──────────────────────────────────────┘
```

---

## 4. Core Design Questions Answered

### Q1: How do you implement location-based store search?

**Answer:** Use PostgreSQL with PostGIS extension for lat/lng spatial queries. Store store locations in `stores` table with `location` column of type `GEOGRAPHY(Point, 4326)` (WGS84). Query using `ST_Distance` and `ST_DWithin` functions to find stores within radius. Cache store location data in Redis using GEO commands (`GEO_ADD`) for faster searches (avoid DB round-trip). Return nearby stores sorted by distance. If PostGIS is overkill, use Redis GEO with acceptable accuracy loss. Typical query: "Find all stores within 5 km of customer location, sort by distance."

### Q2: How do you check real-time inventory at specific stores?

**Answer:** Maintain dual inventory: (1) **Redis hash per store** `store:{storeId}:inventory:{sku}` with quantity value, updated on each reservation/release. (2) **PostgreSQL table** `store_inventory_snapshot` with daily snapshots for durability/audit. When checking inventory, query Redis (fast, eventually consistent) and fall back to DB if cache miss. On every inventory-affecting operation (reservation, pickup, return), update both Redis and DB asynchronously via Kafka event. Inventory is soft real-time: Redis has latest, DB catches up within seconds.

### Q3: How do you reserve items at stores to prevent double-booking?

**Answer:** Use Redis atomic DECR with distributed lock: (1) Acquire lock `reservation:{storeId}:{sku}:lock` (10-second TTL). (2) Check inventory in Redis. (3) If sufficient, DECR quantity. (4) Write reservation record to PostgreSQL with status `RESERVED`. (5) Release lock. If operation fails at any step, release lock and return error (reservation not created). Use idempotency key `reservation_id` to prevent duplicate reservations on retry. PostgreSQL unique index on `(reservation_id)` ensures no duplicates. Kafka event `pickup.reservation_created` triggers order creation and notification.

### Q4: How do you handle reservation expiration?

**Answer:** Implement a **daily batch job** (Quartz) that scans `reservations` table for rows where `created_at + 7 days < now()` and `status = 'RESERVED'`. For each expired reservation: (1) Release quantity back to Redis `store:{storeId}:inventory:{sku}` INCR. (2) Update reservation status to `EXPIRED`. (3) Publish `pickup.reservation_expired` event. (4) Send customer notification ("Your reservation expired, item returned to inventory"). Also track expiration in Redis sorted set with score = `expiry_timestamp` for real-time visibility.

### Q5: How do you notify customers when order is ready?

**Answer:** Use Kafka topic `pickup.ready` consumed by `NotificationService`. When store associate scans item and marks "picked up," trigger `pickup.item_picked_up` event. Collect picked items; once all items in reservation are picked, publish `pickup.ready` event with `reservation_id`, `store_id`, `customer_id`. NotificationService consumes this and sends: (1) Push notification (Firebase Cloud Messaging). (2) SMS (Twilio). (3) Email. Support retry with exponential backoff if channels fail. Store notification preference in customer profile (which channels to use).

### Q6: How do you coordinate between online and in-store systems?

**Answer:** Implement shared inventory model: all channels (online, BOPIS, in-store) consume and publish to same Kafka topics. When item is reserved for BOPIS, publish `inventory.decremented` event (consumed by online system to reduce available stock). When item is released/returned, publish `inventory.incremented`. This ensures online inventory reflects BOPIS reservations in near real-time. Use eventual consistency: customers see inventory with slight lag (acceptable). If double-booking occurs (oversell), flag as critical incident, prioritize by channel (in-store gets priority), and notify customers. Implement daily reconciliation job to sync inventory across channels.

---

## 5. Microservices Breakdown

| Service | Responsibility | Tech | Scaling |
|---------|---|---|---|
| Store Locator Service | Find stores by location, return inventory | Spring Boot + PostGIS | Horizontal, read-heavy |
| Pickup Reservation Service | Create/manage reservations, atomic operations | Spring Boot + Redis Locks | Horizontal, distributed |
| Notification Service | Send push/SMS/email notifications | Spring Boot Kafka Consumer | Horizontal, scale 5-10 |
| Order Fulfillment Service | Create order, sync with warehouse | Spring Boot | Horizontal |
| Reservation Expiry Job | Batch scan expired reservations | Quartz Scheduler | Single, 1 node |
| Store Associate API | Mark pickup, report damage, real-time dashboard | Spring Boot WebSocket | Horizontal |
| Inventory Sync Service | Sync inventory across channels | Kafka Consumer | Single/pair, ordered |
| Audit Service | Log all reservation/pickup events | MongoDB sink | Horizontal |

---

## 6. Database Design

### PostgreSQL Schema

```sql
CREATE TABLE stores (
    id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    address VARCHAR(500) NOT NULL,
    city VARCHAR(100) NOT NULL,
    state VARCHAR(50) NOT NULL,
    zip_code VARCHAR(20) NOT NULL,
    phone VARCHAR(20),
    location GEOGRAPHY(Point, 4326) NOT NULL,  -- PostGIS point
    latitude DECIMAL(10, 8),
    longitude DECIMAL(11, 8),
    opening_time TIME,
    closing_time TIME,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_location USING GIST (location),
    INDEX idx_is_active (is_active)
);

CREATE TABLE reservations (
    id UUID PRIMARY KEY,
    reservation_id UUID NOT NULL UNIQUE,
    customer_id UUID NOT NULL,
    store_id UUID NOT NULL REFERENCES stores(id),
    order_id UUID,
    sku VARCHAR(50) NOT NULL,
    quantity INT NOT NULL,
    status VARCHAR(50) NOT NULL DEFAULT 'RESERVED'
        CHECK (status IN ('RESERVED', 'PICKED_UP', 'EXPIRED', 'CANCELLED', 'DAMAGED')),
    pickup_time_window VARCHAR(50),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP NOT NULL,
    picked_up_at TIMESTAMP,
    cancelled_at TIMESTAMP,
    cancellation_reason VARCHAR(255),

    UNIQUE (reservation_id),
    INDEX idx_customer_id (customer_id),
    INDEX idx_store_id (store_id),
    INDEX idx_status (status),
    INDEX idx_expires_at (expires_at),
    CONSTRAINT check_not_expired CHECK (created_at + INTERVAL '7 days' > NOW())
);

CREATE TABLE pickup_orders (
    id UUID PRIMARY KEY,
    order_id UUID NOT NULL UNIQUE,
    customer_id UUID NOT NULL,
    store_id UUID NOT NULL REFERENCES stores(id),
    total_amount DECIMAL(10, 2),
    status VARCHAR(50) NOT NULL DEFAULT 'PENDING'
        CHECK (status IN ('PENDING', 'READY', 'PICKED_UP', 'CANCELLED')),
    ready_at TIMESTAMP,
    picked_up_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_customer_id (customer_id),
    INDEX idx_store_id (store_id),
    INDEX idx_status (status),
    INDEX idx_ready_at (ready_at)
);

CREATE TABLE store_inventory_snapshot (
    id UUID PRIMARY KEY,
    store_id UUID NOT NULL REFERENCES stores(id),
    sku VARCHAR(50) NOT NULL,
    quantity INT NOT NULL,
    reserved_count INT DEFAULT 0,
    available_count INT GENERATED ALWAYS AS (quantity - reserved_count) STORED,
    last_synced_at TIMESTAMP,
    snapshot_date DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    UNIQUE (store_id, sku, snapshot_date),
    INDEX idx_store_id (store_id),
    INDEX idx_sku (sku)
);

CREATE TABLE store_associate_pickups (
    id UUID PRIMARY KEY,
    associate_id UUID NOT NULL,
    reservation_id UUID NOT NULL REFERENCES reservations(id),
    store_id UUID NOT NULL,
    sku VARCHAR(50) NOT NULL,
    quantity_picked INT NOT NULL,
    is_damaged BOOLEAN DEFAULT FALSE,
    damage_reason VARCHAR(255),
    picked_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_associate_id (associate_id),
    INDEX idx_store_id (store_id),
    INDEX idx_picked_at (picked_at)
);

CREATE TABLE reservation_audit_log (
    id UUID PRIMARY KEY,
    reservation_id UUID NOT NULL,
    event_type VARCHAR(50),
    old_status VARCHAR(50),
    new_status VARCHAR(50),
    actor_id UUID,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,

    INDEX idx_reservation_id (reservation_id),
    INDEX idx_timestamp (timestamp)
);
```

### MongoDB Collections

```json
// reservation_events (event sourcing audit trail)
{
  "_id": ObjectId,
  "reservation_id": UUID,
  "event_type": "CREATED|READY|PICKED_UP|EXPIRED|CANCELLED|DAMAGED",
  "timestamp": ISODate,
  "customer_id": UUID,
  "store_id": UUID,
  "sku": "PROD-12345",
  "quantity": 2,
  "event_data": {
    "status_before": "RESERVED",
    "status_after": "PICKED_UP",
    "actor_id": UUID,
    "actor_type": "CUSTOMER|STORE_ASSOCIATE",
    "notes": "Item damaged during storage"
  },
  "version": 1,
  "created_at": ISODate
}

// pickup_order_events (order lifecycle)
{
  "_id": ObjectId,
  "order_id": UUID,
  "event_type": "CREATED|READY_FOR_PICKUP|PICKED_UP|EXPIRED",
  "timestamp": ISODate,
  "customer_id": UUID,
  "store_id": UUID,
  "items": [
    {
      "sku": "PROD-12345",
      "quantity": 2,
      "reservation_id": UUID
    }
  ],
  "notification_sent": {
    "push": true,
    "sms": true,
    "email": false,
    "timestamps": [ISODate, ISODate]
  },
  "created_at": ISODate
}

// store_associate_logs (real-time associate actions)
{
  "_id": ObjectId,
  "associate_id": UUID,
  "store_id": UUID,
  "action": "SCAN|MARK_PICKED_UP|REPORT_DAMAGE|UPDATE_ORDER_STATUS",
  "reservation_id": UUID,
  "sku": "PROD-12345",
  "timestamp": ISODate,
  "device_info": {
    "tablet_id": "TABLET-001",
    "location": { "lat": 40.7128, "lng": -74.0060 }
  },
  "created_at": ISODate
}
```

---

## 7. Redis Data Structures

```
# Store location geospatial index (TTL: none, updates on store master changes)
store_locations → GEOHASH SET {
    member = store_id,
    longitude = -74.0060,
    latitude = 40.7128
}

# Store inventory per SKU (TTL: 1 hour or on update)
store:{storeId}:inventory:{sku} → Integer (quantity available)

# Store inventory full snapshot (TTL: 30 minutes)
store:{storeId}:inventory → Hash {
    "{sku}": quantity,
    "{sku}": quantity,
    ...
}

# Active reservations sorted by expiry (TTL: 7 days + cleanup)
active_reservations → Sorted Set {
    score = expires_at_timestamp,
    member = reservation_id
}

# Reservation details (TTL: 7 days)
reservation:{reservationId} → Hash {
    "customer_id": UUID,
    "store_id": UUID,
    "sku": "PROD-12345",
    "quantity": 2,
    "status": "RESERVED",
    "created_at": "2026-04-01T10:00:00Z",
    "expires_at": "2026-04-08T10:00:00Z"
}

# Distributed lock for reservation (TTL: 10 seconds)
reservation:lock:{storeId}:{sku} → "locked"

# Store associate WebSocket connections (TTL: session duration)
store_associate:{storeId}:connected → Set [associate_id1, associate_id2, ...]

# Real-time pickup queue per store (TTL: 24 hours)
store:{storeId}:pending_pickups → Sorted Set {
    score = created_at_timestamp,
    member = reservation_id
}

# Notification status tracking (TTL: 7 days)
notification:{reservationId} → Hash {
    "push": "2026-04-01T10:05:00Z",
    "sms": "2026-04-01T10:06:00Z",
    "email": null
}
```

---

## 8. Kafka Event Flow

### Topics & Events

```
Topic: pickup.reservations (Retention: 30 days)
├─ pickup.reservation_created
│  └─ {reservation_id, customer_id, store_id, sku, quantity, expires_at}
├─ pickup.reservation_ready
│  └─ {reservation_id, store_id, ready_at}
├─ pickup.reservation_expired
│  └─ {reservation_id, store_id, expired_at}
└─ pickup.reservation_cancelled
   └─ {reservation_id, cancellation_reason}

Topic: pickup.fulfillment (Retention: 30 days)
├─ pickup.picked_up
│  └─ {reservation_id, associate_id, picked_up_at}
├─ pickup.damaged
│  └─ {reservation_id, damage_reason, quantity_damaged}
└─ [consumed by Notification Service, Order service]

Topic: notifications (Retention: 7 days)
├─ notification.ready_for_pickup
│  └─ {reservation_id, customer_id, store_id, pickup_window}
├─ notification.sent
│  └─ {reservation_id, channel: PUSH|SMS|EMAIL, sent_at}
└─ notification.failed
   └─ {reservation_id, channel, error_code}

Topic: inventory (Retention: 90 days, compacted by store:sku)
├─ inventory.decremented
│  └─ {store_id, sku, quantity, reservation_id, timestamp}
├─ inventory.incremented
│  └─ {store_id, sku, quantity, reason: RETURN|EXPIRE, timestamp}
└─ inventory.synced
   └─ {store_id, sku, new_quantity, sync_source: SYSTEM|MANUAL}
```

---

## 9. Implementation Code

### 9.1 Store & Reservation Entities

```java
package com.retail.pickup.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;
import java.math.BigDecimal;
import java.time.LocalDateTime;
import java.time.LocalTime;
import java.util.UUID;

@Entity
@Table(name = "stores")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Store {

    @Id
    @Column(columnDefinition = "UUID")
    private UUID id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private String address;

    @Column(nullable = false)
    private String city;

    @Column(nullable = false)
    private String state;

    @Column(nullable = false)
    private String zipCode;

    private String phone;

    @Column(nullable = false)
    private BigDecimal latitude;

    @Column(nullable = false)
    private BigDecimal longitude;

    private LocalTime openingTime;

    private LocalTime closingTime;

    @Column(nullable = false)
    private Boolean isActive;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(nullable = false)
    private LocalDateTime updatedAt;

    @PrePersist
    protected void onCreate() {
        this.id = UUID.randomUUID();
        this.createdAt = LocalDateTime.now();
        this.updatedAt = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
}

@Entity
@Table(name = "reservations")
@Data
@NoArgsConstructor
@AllArgsConstructor
@Builder
public class Reservation {

    @Id
    @Column(columnDefinition = "UUID")
    private UUID id;

    @Column(nullable = false, unique = true)
    private UUID reservationId;

    @Column(nullable = false)
    private UUID customerId;

    @Column(nullable = false)
    private UUID storeId;

    private UUID orderId;

    @Column(nullable = false)
    private String sku;

    @Column(nullable = false)
    private Integer quantity;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private ReservationStatus status;

    private String pickupTimeWindow;

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(nullable = false)
    private LocalDateTime expiresAt;

    private LocalDateTime pickedUpAt;

    private LocalDateTime cancelledAt;

    private String cancellationReason;

    @PrePersist
    protected void onCreate() {
        this.id = UUID.randomUUID();
        this.createdAt = LocalDateTime.now();
        if (this.expiresAt == null) {
            this.expiresAt = this.createdAt.plusDays(7);
        }
    }
}

public enum ReservationStatus {
    RESERVED, PICKED_UP, EXPIRED, CANCELLED, DAMAGED
}
```

### 9.2 Store Locator Service with PostGIS

```java
package com.retail.pickup.service;

import com.retail.pickup.entity.Store;
import com.retail.pickup.repository.StoreRepository;
import com.retail.pickup.dto.StoreWithInventoryDto;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;

import java.math.BigDecimal;
import java.util.*;

@Service
@Slf4j
public class StoreLocatorService {

    private final StoreRepository storeRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final JdbcTemplate jdbcTemplate;
    private final MeterRegistry meterRegistry;

    public StoreLocatorService(
        StoreRepository storeRepository,
        RedisTemplate<String, String> redisTemplate,
        JdbcTemplate jdbcTemplate,
        MeterRegistry meterRegistry
    ) {
        this.storeRepository = storeRepository;
        this.redisTemplate = redisTemplate;
        this.jdbcTemplate = jdbcTemplate;
        this.meterRegistry = meterRegistry;
    }

    public List<StoreWithInventoryDto> findNearbyStores(
        BigDecimal customerLat,
        BigDecimal customerLng,
        String sku,
        int radiusKm
    ) {
        long startTime = System.currentTimeMillis();

        // Use PostGIS to find stores within radius
        String sql = """
            SELECT s.id, s.name, s.address, s.city, s.state, s.zip_code,
                   s.latitude, s.longitude, s.phone,
                   ST_Distance(
                       ST_GeogFromText('SRID=4326;POINT(' || ? || ' ' || ? || ')'),
                       s.location
                   ) / 1000.0 AS distance_km
            FROM stores s
            WHERE s.is_active = true
            AND ST_DWithin(
                s.location,
                ST_GeogFromText('SRID=4326;POINT(' || ? || ' ' || ? || ')'),
                ? * 1000
            )
            ORDER BY distance_km ASC
            LIMIT 20
            """;

        List<StoreWithInventoryDto> storesWithInventory = jdbcTemplate.query(
            sql,
            new Object[]{customerLng, customerLat, customerLng, customerLat, radiusKm},
            (rs, rowNum) -> {
                UUID storeId = UUID.fromString(rs.getString("id"));

                // Check inventory in Redis first
                Integer availableQty = getAvailableInventory(storeId, sku);

                return StoreWithInventoryDto.builder()
                    .storeId(storeId)
                    .name(rs.getString("name"))
                    .address(rs.getString("address"))
                    .city(rs.getString("city"))
                    .state(rs.getString("state"))
                    .zipCode(rs.getString("zip_code"))
                    .latitude(rs.getBigDecimal("latitude"))
                    .longitude(rs.getBigDecimal("longitude"))
                    .phone(rs.getString("phone"))
                    .distanceKm(rs.getDouble("distance_km"))
                    .availableQuantity(availableQty)
                    .inStock(availableQty > 0)
                    .build();
            }
        );

        long duration = System.currentTimeMillis() - startTime;
        meterRegistry.timer("store.search.latency").record(duration, java.util.concurrent.TimeUnit.MILLISECONDS);

        log.info("Found {} stores within {} km for SKU {}, latency: {} ms",
            storesWithInventory.size(), radiusKm, sku, duration);

        return storesWithInventory;
    }

    private Integer getAvailableInventory(UUID storeId, String sku) {
        // Check Redis first
        String redisKey = "store:" + storeId + ":inventory:" + sku;
        String cachedQty = redisTemplate.opsForValue().get(redisKey);

        if (cachedQty != null) {
            return Integer.parseInt(cachedQty);
        }

        // Fall back to PostgreSQL
        String sql = """
            SELECT available_count FROM store_inventory_snapshot
            WHERE store_id = ? AND sku = ? AND snapshot_date = CURRENT_DATE
            ORDER BY created_at DESC LIMIT 1
            """;

        Integer dbQty = jdbcTemplate.queryForObject(
            sql,
            new Object[]{storeId, sku},
            Integer.class
        );

        // Cache in Redis for 30 minutes
        if (dbQty != null) {
            redisTemplate.opsForValue().set(redisKey, dbQty.toString(), 30, java.util.concurrent.TimeUnit.MINUTES);
        }

        return dbQty != null ? dbQty : 0;
    }

    public Store getStore(UUID storeId) {
        return storeRepository.findById(storeId)
            .orElseThrow(() -> new StoreNotFoundException(storeId));
    }
}

public interface StoreRepository extends JpaRepository<Store, UUID> {
    List<Store> findByIsActiveTrue();
}
```

### 9.3 Pickup Reservation Service with Atomic Operations

```java
package com.retail.pickup.service;

import com.retail.pickup.entity.Reservation;
import com.retail.pickup.entity.ReservationStatus;
import com.retail.pickup.repository.ReservationRepository;
import com.retail.pickup.event.ReservationEvent;
import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.UUID;
import java.util.concurrent.TimeUnit;

@Service
@Slf4j
public class PickupReservationService {

    private final ReservationRepository reservationRepository;
    private final StoreLocatorService storeLocatorService;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, ReservationEvent> kafkaTemplate;
    private final MeterRegistry meterRegistry;

    private static final long LOCK_TIMEOUT_SECONDS = 10;
    private static final long RESERVATION_TTL_DAYS = 7;

    public PickupReservationService(
        ReservationRepository reservationRepository,
        StoreLocatorService storeLocatorService,
        RedisTemplate<String, String> redisTemplate,
        KafkaTemplate<String, ReservationEvent> kafkaTemplate,
        MeterRegistry meterRegistry
    ) {
        this.reservationRepository = reservationRepository;
        this.storeLocatorService = storeLocatorService;
        this.redisTemplate = redisTemplate;
        this.kafkaTemplate = kafkaTemplate;
        this.meterRegistry = meterRegistry;
    }

    @Transactional
    public Reservation createReservation(
        UUID customerId,
        UUID storeId,
        String sku,
        int quantity,
        String pickupTimeWindow
    ) throws Exception {
        String lockKey = "reservation:lock:" + storeId + ":" + sku;

        // Acquire distributed lock
        Boolean lockAcquired = redisTemplate.opsForValue()
            .setIfAbsent(lockKey, "locked", LOCK_TIMEOUT_SECONDS, TimeUnit.SECONDS);

        if (!Boolean.TRUE.equals(lockAcquired)) {
            throw new ReservationLockException("Could not acquire lock for store:" + storeId);
        }

        try {
            // Check current inventory in Redis
            String inventoryKey = "store:" + storeId + ":inventory:" + sku;
            String currentQtyStr = redisTemplate.opsForValue().get(inventoryKey);
            Integer currentQty = currentQtyStr != null ? Integer.parseInt(currentQtyStr) : 0;

            if (currentQty < quantity) {
                throw new InsufficientInventoryException(
                    "Only " + currentQty + " available, requested " + quantity
                );
            }

            // Atomic DECR
            Long newQty = redisTemplate.opsForValue().decrement(inventoryKey, quantity);
            log.info("Inventory decremented: store={}, sku={}, newQty={}", storeId, sku, newQty);

            // Create reservation in PostgreSQL
            Reservation reservation = Reservation.builder()
                .reservationId(UUID.randomUUID())
                .customerId(customerId)
                .storeId(storeId)
                .sku(sku)
                .quantity(quantity)
                .status(ReservationStatus.RESERVED)
                .pickupTimeWindow(pickupTimeWindow)
                .expiresAt(LocalDateTime.now().plusDays(RESERVATION_TTL_DAYS))
                .build();

            reservation = reservationRepository.save(reservation);

            // Cache in Redis
            cacheReservation(reservation);

            // Publish event
            ReservationEvent event = ReservationEvent.created(
                reservation.getReservationId(),
                customerId,
                storeId,
                sku,
                quantity,
                reservation.getExpiresAt()
            );
            kafkaTemplate.send("pickup.reservations", reservation.getReservationId().toString(), event);

            // Add to active reservations sorted set
            redisTemplate.opsForZSet().add(
                "active_reservations",
                reservation.getReservationId().toString(),
                reservation.getExpiresAt().atZone(java.time.ZoneId.systemDefault()).toInstant().toEpochMilli() / 1000.0
            );

            // Metrics
            Counter.builder("reservation.created")
                .tag("store_id", storeId.toString())
                .register(meterRegistry)
                .increment();

            log.info("Reservation created: id={}, customer={}, store={}, sku={}, qty={}",
                reservation.getReservationId(), customerId, storeId, sku, quantity);

            return reservation;

        } catch (Exception e) {
            // On error, restore inventory (INCR back)
            log.error("Reservation creation failed, restoring inventory", e);
            redisTemplate.opsForValue().increment(inventoryKey, quantity);
            throw e;

        } finally {
            // Always release lock
            redisTemplate.delete(lockKey);
        }
    }

    @Transactional
    public void markPickedUp(UUID reservationId, UUID associateId) {
        Reservation reservation = reservationRepository.findByReservationId(reservationId)
            .orElseThrow(() -> new ReservationNotFoundException(reservationId));

        if (reservation.getStatus() != ReservationStatus.RESERVED) {
            throw new InvalidReservationStateException("Can only pick up RESERVED reservations");
        }

        reservation.setStatus(ReservationStatus.PICKED_UP);
        reservation.setPickedUpAt(LocalDateTime.now());
        reservationRepository.save(reservation);

        // Update cache
        cacheReservation(reservation);

        // Publish event
        ReservationEvent event = ReservationEvent.pickedUp(reservationId, associateId);
        kafkaTemplate.send("pickup.fulfillment", reservationId.toString(), event);

        Counter.builder("reservation.picked_up").register(meterRegistry).increment();
        log.info("Reservation marked picked up: id={}, associate={}", reservationId, associateId);
    }

    @Transactional
    public void reportDamage(UUID reservationId, String damageReason, int quantityDamaged) {
        Reservation reservation = reservationRepository.findByReservationId(reservationId)
            .orElseThrow();

        reservation.setStatus(ReservationStatus.DAMAGED);
        reservationRepository.save(reservation);

        // Restore damaged inventory
        String inventoryKey = "store:" + reservation.getStoreId() + ":inventory:" + reservation.getSku();
        redisTemplate.opsForValue().increment(inventoryKey, quantityDamaged);

        // Publish event
        ReservationEvent event = ReservationEvent.damaged(reservationId, damageReason, quantityDamaged);
        kafkaTemplate.send("pickup.fulfillment", reservationId.toString(), event);

        // Notify customer
        kafkaTemplate.send("notifications",
            reservationId.toString(),
            new DamageNotificationEvent(reservation.getCustomerId(), damageReason));

        Counter.builder("reservation.damaged").register(meterRegistry).increment();
        log.error("Item damaged: reservation={}, reason={}, qty={}", reservationId, damageReason, quantityDamaged);
    }

    public void cancelReservation(UUID reservationId, String reason) {
        Reservation reservation = reservationRepository.findByReservationId(reservationId)
            .orElseThrow();

        if (reservation.getStatus() == ReservationStatus.CANCELLED) {
            throw new InvalidReservationStateException("Already cancelled");
        }

        // Restore inventory
        String inventoryKey = "store:" + reservation.getStoreId() + ":inventory:" + reservation.getSku();
        redisTemplate.opsForValue().increment(inventoryKey, reservation.getQuantity());

        reservation.setStatus(ReservationStatus.CANCELLED);
        reservation.setCancelledAt(LocalDateTime.now());
        reservation.setCancellationReason(reason);
        reservationRepository.save(reservation);

        ReservationEvent event = ReservationEvent.cancelled(reservationId, reason);
        kafkaTemplate.send("pickup.reservations", reservationId.toString(), event);

        Counter.builder("reservation.cancelled").register(meterRegistry).increment();
        log.info("Reservation cancelled: id={}, reason={}", reservationId, reason);
    }

    private void cacheReservation(Reservation reservation) {
        String key = "reservation:" + reservation.getReservationId();
        // Serialize and cache with 7-day TTL
        redisTemplate.opsForValue().set(
            key,
            serializeReservation(reservation),
            RESERVATION_TTL_DAYS,
            TimeUnit.DAYS
        );
    }

    private String serializeReservation(Reservation r) {
        // Use ObjectMapper in real code
        return r.getReservationId().toString();
    }
}
```

### 9.4 Reservation Expiry Job (Quartz)

```java
package com.retail.pickup.scheduler;

import com.retail.pickup.entity.Reservation;
import com.retail.pickup.entity.ReservationStatus;
import com.retail.pickup.repository.ReservationRepository;
import com.retail.pickup.event.ReservationEvent;
import lombok.extern.slf4j.Slf4j;
import org.quartz.Job;
import org.quartz.JobExecutionContext;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.kafka.core.KafkaTemplate;
import org.springframework.stereotype.Component;

import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;

@Component
@Slf4j
public class ReservationExpiryJob implements Job {

    private final ReservationRepository reservationRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, ReservationEvent> kafkaTemplate;

    public ReservationExpiryJob(
        ReservationRepository reservationRepository,
        RedisTemplate<String, String> redisTemplate,
        KafkaTemplate<String, ReservationEvent> kafkaTemplate
    ) {
        this.reservationRepository = reservationRepository;
        this.redisTemplate = redisTemplate;
        this.kafkaTemplate = kafkaTemplate;
    }

    @Override
    public void execute(JobExecutionContext context) {
        log.info("Starting reservation expiry job at {}", LocalDateTime.now());

        LocalDateTime now = LocalDateTime.now();

        // Find all RESERVED reservations that have expired
        List<Reservation> expiredReservations = reservationRepository
            .findByStatusAndExpiresAtBefore(
                ReservationStatus.RESERVED,
                now
            );

        log.info("Found {} expired reservations", expiredReservations.size());

        for (Reservation reservation : expiredReservations) {
            try {
                // Restore inventory to Redis
                String inventoryKey = "store:" + reservation.getStoreId()
                    + ":inventory:" + reservation.getSku();
                redisTemplate.opsForValue().increment(inventoryKey, reservation.getQuantity());

                // Update status
                reservation.setStatus(ReservationStatus.EXPIRED);
                reservationRepository.save(reservation);

                // Publish expiration event
                ReservationEvent event = ReservationEvent.expired(reservation.getReservationId());
                kafkaTemplate.send(
                    "pickup.reservations",
                    reservation.getReservationId().toString(),
                    event
                );

                log.info("Reservation expired and released: id={}, inventory restored",
                    reservation.getReservationId());

            } catch (Exception e) {
                log.error("Error processing expired reservation: {}",
                    reservation.getReservationId(), e);
            }
        }

        log.info("Reservation expiry job completed at {}", LocalDateTime.now());
    }
}
```

### 9.5 Store Associate Controller (WebSocket)

```java
package com.retail.pickup.controller;

import com.retail.pickup.service.PickupReservationService;
import com.retail.pickup.service.StoreLocatorService;
import com.retail.pickup.dto.PickupDashboardDto;
import io.micrometer.core.instrument.MeterRegistry;
import lombok.extern.slf4j.Slf4j;
import org.springframework.messaging.handler.annotation.MessageMapping;
import org.springframework.messaging.handler.annotation.SendTo;
import org.springframework.messaging.simp.SimpMessagingTemplate;
import org.springframework.web.bind.annotation.*;

import java.util.UUID;

@RestController
@RequestMapping("/api/store-associate")
@Slf4j
public class StoreAssociateController {

    private final PickupReservationService pickupReservationService;
    private final SimpMessagingTemplate messagingTemplate;
    private final MeterRegistry meterRegistry;

    public StoreAssociateController(
        PickupReservationService pickupReservationService,
        SimpMessagingTemplate messagingTemplate,
        MeterRegistry meterRegistry
    ) {
        this.pickupReservationService = pickupReservationService;
        this.messagingTemplate = messagingTemplate;
        this.meterRegistry = meterRegistry;
    }

    @PostMapping("/reservations/{reservationId}/mark-picked-up")
    public void markPickedUp(
        @PathVariable UUID reservationId,
        @RequestParam UUID storeId,
        @RequestHeader("X-Associate-Id") UUID associateId
    ) {
        long startTime = System.currentTimeMillis();

        try {
            pickupReservationService.markPickedUp(reservationId, associateId);

            // Notify store dashboard in real-time
            messagingTemplate.convertAndSend(
                "/topic/store/" + storeId + "/pickups",
                new PickupStatusUpdate(reservationId, "PICKED_UP")
            );

            meterRegistry.timer("associate.mark_pickup_latency")
                .record(System.currentTimeMillis() - startTime, java.util.concurrent.TimeUnit.MILLISECONDS);

            log.info("Item marked picked up: reservation={}, associate={}", reservationId, associateId);

        } catch (Exception e) {
            log.error("Error marking pickup", e);
            throw new RuntimeException(e);
        }
    }

    @PostMapping("/reservations/{reservationId}/report-damage")
    public void reportDamage(
        @PathVariable UUID reservationId,
        @RequestParam String reason,
        @RequestParam int quantityDamaged,
        @RequestParam UUID storeId,
        @RequestHeader("X-Associate-Id") UUID associateId
    ) {
        try {
            pickupReservationService.reportDamage(reservationId, reason, quantityDamaged);

            // Notify store management
            messagingTemplate.convertAndSend(
                "/topic/store/" + storeId + "/alerts",
                new DamageAlert(reservationId, reason, quantityDamaged)
            );

            log.warn("Damage reported: reservation={}, reason={}, qty={}, associate={}",
                reservationId, reason, quantityDamaged, associateId);

        } catch (Exception e) {
            log.error("Error reporting damage", e);
            throw new RuntimeException(e);
        }
    }

    @GetMapping("/stores/{storeId}/dashboard")
    public PickupDashboardDto getStoreDashboard(@PathVariable UUID storeId) {
        // Real-time pickup queue, pending items, throughput metrics
        return new PickupDashboardDto(); // Implementation
    }

    @MessageMapping("/associate/scan")
    @SendTo("/topic/store/{storeId}/scan-results")
    public ScanResultDto scanBarcode(@RequestBody ScanRequest request) {
        // Barcode scanner integration for pick confirmation
        log.info("Barcode scanned: sku={}, store={}", request.sku(), request.storeId());
        return new ScanResultDto();
    }
}

record PickupStatusUpdate(UUID reservationId, String status) {}
record DamageAlert(UUID reservationId, String reason, int quantity) {}
record ScanRequest(String barcode, String sku, UUID storeId, UUID associateId) {}
record ScanResultDto(String sku, int quantity, boolean found) {}
```

### 9.6 Notification Service (Kafka Consumer)

```java
package com.retail.pickup.notification;

import com.retail.pickup.event.ReservationEvent;
import com.retail.pickup.service.CustomerService;
import com.retail.pickup.client.NotificationClient;
import lombok.extern.slf4j.Slf4j;
import org.springframework.kafka.annotation.KafkaListener;
import org.springframework.stereotype.Service;

import java.util.UUID;

@Service
@Slf4j
public class PickupNotificationService {

    private final CustomerService customerService;
    private final NotificationClient notificationClient;

    public PickupNotificationService(
        CustomerService customerService,
        NotificationClient notificationClient
    ) {
        this.customerService = customerService;
        this.notificationClient = notificationClient;
    }

    @KafkaListener(topics = "pickup.fulfillment", groupId = "pickup-notification-group")
    public void handlePickupEvent(ReservationEvent event) {
        if (event.getEventType().equals("PICKED_UP")) {
            sendReadyNotification(event.getReservationId(), event.getCustomerId());
        } else if (event.getEventType().equals("DAMAGED")) {
            sendDamageNotification(event.getReservationId(), event.getCustomerId());
        }
    }

    private void sendReadyNotification(UUID reservationId, UUID customerId) {
        try {
            CustomerProfile customer = customerService.getCustomer(customerId);

            String message = "Your order " + reservationId + " is ready for pickup!";

            // Send via all channels based on customer preference
            if (customer.getPushNotificationEnabled()) {
                notificationClient.sendPush(
                    customer.getFirebaseToken(),
                    "Order Ready",
                    message
                );
            }

            if (customer.getSmsNotificationEnabled()) {
                notificationClient.sendSms(
                    customer.getPhoneNumber(),
                    message
                );
            }

            if (customer.getEmailNotificationEnabled()) {
                notificationClient.sendEmail(
                    customer.getEmail(),
                    "Your Order is Ready for Pickup",
                    buildReadyEmailBody(reservationId, customer)
                );
            }

            log.info("Ready notification sent: reservation={}, customer={}", reservationId, customerId);

        } catch (Exception e) {
            log.error("Error sending ready notification", e);
            // Retry via DLQ
        }
    }

    private void sendDamageNotification(UUID reservationId, UUID customerId) {
        try {
            CustomerProfile customer = customerService.getCustomer(customerId);

            String message = "Unfortunately, your reserved item was damaged. " +
                "Please contact the store for alternatives.";

            notificationClient.sendEmail(
                customer.getEmail(),
                "Item Damaged - Reservation Updated",
                buildDamageEmailBody(reservationId, customer)
            );

            log.info("Damage notification sent: reservation={}, customer={}", reservationId, customerId);

        } catch (Exception e) {
            log.error("Error sending damage notification", e);
        }
    }

    private String buildReadyEmailBody(UUID reservationId, CustomerProfile customer) {
        return """
            Hi %s,

            Great news! Your order is ready for pickup!

            Order ID: %s

            Please pick it up at your selected store within 24 hours.
            Items not picked up within 7 days will be returned to inventory.

            Thank you!
            """.formatted(customer.getFirstName(), reservationId);
    }

    private String buildDamageEmailBody(UUID reservationId, CustomerProfile customer) {
        return """
            Hi %s,

            We regret to inform you that your reserved item was damaged
            during storage at our facility.

            Reservation ID: %s

            We sincerely apologize for the inconvenience.
            Please reply to this email or contact our customer service team
            to discuss alternatives.

            Best regards,
            Customer Service Team
            """.formatted(customer.getFirstName(), reservationId);
    }
}
```

---

## 10. Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| Inventory goes negative | Double-booking | Distributed lock + atomic DECR, unique constraint |
| Reservation locked 10+ sec | Customer timeout | Lock TTL 10s, exponential backoff retry |
| Kafka message loss | Missed notifications | Kafka retention 30 days, replay on gaps |
| Store inventory cache miss | Fallback to old data | Query PostgreSQL, refresh cache |
| Reservation expiry job crashes | Orphaned reservations | Daily manual reconciliation, alerts on job failure |
| Customer no-show | Lost inventory | 7-day auto-release, send reminder before expiry |
| Associate never marks pickup | Stuck reservation | Flag unprocessed after 24h, manager escalation |
| Network partition | Can't acquire lock | Fail closed (reject reservation) vs fail open (allow & reconcile) |
| Store closes early | Can't find items | Pre-closing notification to customers, auto-release option |

---

## 11. Scaling Strategy

### Horizontal Scaling
- **Store Locator API:** Stateless; scale based on search volume (read-heavy)
- **Pickup Reservation API:** Stateless with distributed locks; scale for concurrent reservations
- **Notification Service:** Independent Kafka consumer; scale 5-10 instances
- **Store Associate App:** WebSocket connections; use sticky sessions or Redis session store

### Database Optimization
- Partition reservations by `store_id` (2,000 partitions) for parallel access
- Index on `(status, expires_at)` for expiry job
- Denormalize store details in reservation record for faster queries
- Archive old reservations (>30 days) to separate table

### Redis Caching
- Store locations: cached at startup, updated on schedule
- Inventory per store: TTL 30 minutes or on update
- Active reservations: TTL 7 days + cleanup
- Locks: TTL 10 seconds (short, fail-fast)

### Kafka Partitioning
- `pickup.reservations`: partition by `store_id` (ensure store-level ordering)
- `pickup.fulfillment`: partition by `store_id` (real-time pickup tracking)
- `notifications`: partition by `customer_id` (batch notifications per customer)

---

## 12. Monitoring & Observability

### Key Metrics
```
store.search.latency (timer, tag: radius_km)
inventory.check.cache_hit_rate (gauge)
reservation.created (counter, tag: store_id)
reservation.picked_up (counter)
reservation.expired (counter)
reservation.damaged (counter)
reservation.lock_acquisition_time (timer)
notification.sent (counter, tag: channel: PUSH|SMS|EMAIL)
store_associate.items_processed (counter, tag: store_id)
pickup_queue.depth (gauge, tag: store_id)
```

### Alerting
- If reservation latency p99 > 2s: investigate lock contention
- If inventory cache hit rate < 90%: increase TTL or cache size
- If notification.failed rate > 2%: page oncall
- If expiry job misses: investigate Quartz scheduler health
- If reservation lock acquisition fails >1% of requests: scale REDIS or reduce lock time

---

## 13. Summary Cheat Sheet

### Core Technology Choices
- **Location Search:** PostGIS + Spring Data JPA (ST_DWithin queries)
- **Inventory Management:** Redis atomic DECR + PostgreSQL snapshots
- **Distributed Lock:** Redis with 10-second TTL + exponential backoff
- **Event Streaming:** Kafka for all async operations
- **Real-time Dashboard:** WebSocket (Spring WebSocket + SockJS)

### Key Operations
| Operation | Technology | Latency | Consistency |
|-----------|-----------|---------|-------------|
| Search stores | PostGIS query | ~300ms | Strong |
| Check inventory | Redis GET | ~10ms | Eventual |
| Create reservation | Distributed lock + DECR | ~500ms | Strong |
| Mark pickup | PostgreSQL update | ~200ms | Strong |
| Send notification | Kafka → async | ~5sec | Eventual |
| Expire reservation | Batch job (daily) | N/A | Strong |

### Data Model Summary
```
PostgreSQL:
- stores (2,000 records)
- reservations (70K active)
- pickup_orders (10K/day)
- store_inventory_snapshot (daily)

MongoDB:
- reservation_events (100M documents/year)
- pickup_order_events
- store_associate_logs

Redis:
- Inventory per store:SKU
- Active reservations
- Distributed locks
- GEO index for stores
```

### API Endpoints
```
GET /api/stores/search?lat=40.7128&lng=-74.0060&radius=5&sku=PROD-123
GET /api/stores/{storeId}/inventory/{sku}
POST /api/reservations (store_id, sku, quantity, pickup_window)
POST /api/reservations/{reservationId}/mark-picked-up
POST /api/reservations/{reservationId}/report-damage
DELETE /api/reservations/{reservationId} (cancel)
GET /api/store-associate/stores/{storeId}/dashboard
WS /ws/store/{storeId}/pickups (WebSocket)
```

### Best Practices
1. **Lock briefly:** 10-second TTL, fail-fast on contention
2. **Cache everything:** Store locations, inventory snapshots
3. **Notify aggressively:** Push, SMS, email on state changes
4. **Audit thoroughly:** All actions logged to MongoDB
5. **Handle no-show:** Auto-release after 7 days + reminder emails
6. **Monitor queue depth:** Alert if pickup backlog grows
7. **Reconcile daily:** Sync inventory across all channels

