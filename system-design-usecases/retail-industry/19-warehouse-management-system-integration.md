---
title: Q19 - Warehouse Management System Integration
layout: default
---

# Q19 - Warehouse Management System Integration — Deep Dive Design

> **Scenario**: Integrate with multiple external Warehouse Management Systems (WMS). When an order is placed, dispatch to appropriate WMS for fulfillment (picking, packing, shipping). Handle split shipments across multiple warehouses, track shipment status, sync inventory in real-time. Support 20 warehouses, 10+ WMS vendors (Manhattan, SAP EWM, custom legacy), 50K orders/day.
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
- **Order Dispatch**: Send orders to appropriate WMS within 5 minutes of confirmation
- **Multi-Warehouse Fulfillment**: Split order across multiple warehouses if single warehouse can't fulfill all items
- **Real-Time Inventory Sync**: Inventory updates from WMS every 5 minutes
- **Shipment Tracking**: Track shipment status (confirmed, picked, packed, shipped, delivered)
- **Backorder Handling**: Partial fulfillment, notify customer of expected delivery dates
- **WMS Webhook Integration**: Accept fulfillment updates from WMS (asynchronous push)
- **Legacy System Support**: Adapter pattern for 10+ WMS vendors with different APIs

### Non-Functional Requirements
- **Availability**: 99.95% (allow brief WMS downtime)
- **Latency**: Order dispatch < 2s, inventory sync < 30s propagation
- **Throughput**: 50K orders/day, inventory updates every 5 min from 20 warehouses
- **Order Consistency**: Strong consistency for order status, eventual consistency for inventory
- **Integration Reliability**: Exactly-once delivery of order dispatch events
- **Scalability**: Support up to 50 warehouses, 20 WMS vendors without code changes

### Constraints
- WMS APIs are synchronous (request-response), unreliable, rate-limited
- No direct database access to WMS systems (API-only)
- Warehouse assignment logic must be flexible (geolocation, inventory, capacity)
- Must handle WMS downtime gracefully (queue orders, replay when recovered)
- Shipment data is authoritative in WMS (read-only on e-commerce side)
- Regulatory compliance for tracking (audit trail required)

---

## 2. Capacity Estimation

| Metric | Value | Notes |
|--------|-------|-------|
| Daily Orders | 50,000 | Total across all channels |
| Orders/Min (peak) | 60 | 50K / 840 min business hours |
| Warehouses | 20 | Physical locations |
| Warehouse Capacity | 100K–500K SKUs | Per warehouse |
| Inventory Updates/Day | 288 | 20 warehouses × 14.4/day (every 5 min) |
| Shipment Status Updates/Day | 150,000 | Multiple updates per order |
| Backorder Rate | ~3% | Partial fulfillment |
| Split Shipment Rate | ~15% | Orders split across warehouses |
| WMS API Calls/Day | 75K | 50K orders + 25K inventory syncs |
| Average Order Items | 3 | Items per order |

### Storage Estimates
- **PostgreSQL**: Orders (~200GB/year), shipments (~150GB/year), backorders (~10GB/year)
- **MongoDB**: Shipment audit trail (~50GB/year), warehouse config (~100MB)
- **Redis**: Active orders cache (~500MB), inventory snapshot (~2GB), shipment tracking (~1GB)
- **Kafka**: Event backlog (~20GB retention)

### Network Bandwidth
- **Outbound to WMS**: ~10 Mbps average, 50 Mbps peak
- **Inbound from WMS**: ~20 Mbps average (shipment status updates)
- **Database**: ~50K req/s average, 200K req/s peak

---

## 3. High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Order Service (E-commerce)                      │
│                   Creates & Confirms Orders                         │
└────────────────────────┬────────────────────────────────────────────┘
                         │
                    Publishes: OrderConfirmedEvent
                         │
         ┌───────────────▼───────────────┐
         │       Fulfillment Service     │
         │   (Orchestration, Dispatch)   │
         └───────────────┬───────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    ┌────▼─────┐  ┌─────▼────┐  ┌──────▼──────┐
    │ Warehouse│  │ Inventory│  │   Shipment  │
    │  Adapter │  │  Service │  │  Tracker    │
    │ Layer    │  │  (Sync)  │  │  Service    │
    └────┬─────┘  └─────┬────┘  └──────┬──────┘
         │              │              │
         │          ┌───▼──────┐      │
         │          │Kafka Bus │      │
         │          │(Events)  │      │
         │          └─┬─────┬──┘      │
         │            │     │        │
    ┌────▼────┐   ┌───▼──┬─▼────┐    │
    │ WMS API │   │Redis │ PG   │    │
    │Clients  │   │Cache │OLTP  │    │
    │(M/SAP)  │   └──────┴──────┘    │
    └─────────┘         ▲            │
        ▲               │            │
        │          (Backfill)        │
        │               │            │
 ┌──────┴────────────────┼────────────┴──────┐
 │                       │                    │
 │  WMS 1 (Manhattan)    │   WMS 2 (SAP)    │  WMS N (Custom)
 │  Warehouse 1-5        │   Warehouse 6-15  │  Warehouse 16-20
 │  HTTP API             │   SOAP API        │  Kafka API
 └───────────────────────┴────────────────────┴────────────────┘
          ▲                        ▲                    ▲
          │        Webhook        │      Webhook      │
          │      Push Updates     │    Push Updates   │
          └─────────────────────────────────────────────┘
                FulfillmentWebhookController
```

---

## 4. Core Design Questions Answered

### Q1: How do you design the integration layer for multiple WMS systems?

**Answer**: Adapter pattern + Anti-corruption layer. Each WMS vendor has a concrete adapter implementing a unified interface.

**Design**:
```
WMSAdapter (interface)
  ├── ManhattanWMSAdapter (HTTP REST)
  ├── SAPEWMAdapter (SOAP)
  ├── CustomLegacyWMSAdapter (SFTP CSV)
  └── ShopifyAdapter (Shopify API)

Each adapter:
  - Translates e-commerce Order → Vendor-specific format
  - Translates Vendor shipment data → E-commerce ShipmentEvent
  - Handles vendor-specific auth, retries, rate limiting
  - Implements circuit breaker + timeout
```

**Anti-Corruption Layer**:
- Vendorspecific models (SAPWMSOrder, ManhattanWMSOrder) kept private to adapter
- Public interface uses only e-commerce models (Order, Shipment)
- Data transformation happens at boundary

**Flexibility**: Add new WMS without touching existing code; new adapter class + registration in factory.

---

### Q2: How do you handle asynchronous fulfillment updates?

**Answer**: Webhook + Kafka event-driven architecture. WMS pushes updates → webhook → Kafka → consumer updates DB.

**Design**:
```
WMS Shipment Status Changes (e.g., picked → shipped)
        │
        └─→ POST /webhooks/fulfillment (FulfillmentWebhookController)
                 │
                 └─→ Validate signature (HMAC)
                     Parse WMS-specific payload
                     Convert to FulfillmentUpdatedEvent
                     Publish to Kafka topic "fulfillment-updates"
                         │
                         └─→ FulfillmentConsumer subscribes
                             Updates shipment status in PostgreSQL
                             Publishes ShipmentStatusChangedEvent
                             Notifies customer (email, SMS)
```

**Idempotency**: Webhook can be retried; use idempotency key (WMS order ID + timestamp) with deduplication in Redis.

**Fallback**: If webhook fails, inventory sync consumer polls WMS API every 5 min as fallback.

---

### Q3: How do you coordinate split shipments to customer?

**Answer**: FulfillmentSplitter service determines best warehouse per item group; creates N shipment records linked to single order.

**Design**:
- **Bin Packing Algorithm**: Given order items + warehouse inventory, find optimal split
  - Goal: Minimize number of shipments (reduce shipping cost)
  - Constraint: Respect warehouse capacity, inventory availability
  - Use greedy algorithm or optimization library (OR-Tools)
- **Shipment Consolidation**: If 2 items ship from same warehouse, combine into 1 shipment
- **Customer Communication**: Display multiple tracking numbers + expected delivery dates per shipment
- **Data Model**:
  ```
  Order {id, customer_id, items[]}
    └─ Shipment {id, order_id, warehouse_id, items[], tracking_number, status, expected_delivery}
  ```

**Split Decision**:
```
Order has: 10x Widget A (warehouse 1 only), 5x Gadget B (warehouse 2 only)
Split into:
  - Shipment 1: Warehouse 1, 10x Widget A, tracking ABC123
  - Shipment 2: Warehouse 2, 5x Gadget B, tracking ABC124
Customer sees both tracking numbers on order page
```

---

### Q4: How do you keep inventory in sync between systems?

**Answer**: Dual-write pattern with eventual consistency. WMS is source of truth; e-commerce syncs every 5 minutes.

**Design**:
- **Inventory Sync Flow**:
  1. WMS publishes InventoryDeltaEvent every 5 min (API poll or webhook)
  2. InventorySyncConsumer subscribes to Kafka topic "inventory-updates"
  3. For each delta, apply to local cache (Redis sorted set by warehouse/SKU)
  4. Publish InventorySyncedEvent → analytics/reporting

- **Caching Strategy**:
  ```
  inventory:{warehouse_id}:{product_id} = STRING "{quantity}"
  inventory:{warehouse_id}:updated_at = STRING "{timestamp}"
  ```
  - TTL: 10 minutes (if not updated, mark as stale)
  - Read from cache for instant availability
  - Background sync to PostgreSQL (eventual consistency)

- **Conflict Resolution**: WMS version always wins (read-only on e-commerce side)

- **Monitoring**: Alert if warehouse inventory not updated for > 15 min (WMS down?)

---

### Q5: How do you handle WMS downtime?

**Answer**: Circuit breaker + queue-based fallback. If WMS down, queue orders in Kafka; replay when recovered.

**Design**:
- **Circuit Breaker** (Resilience4j):
  ```
  - CLOSED: Normal operation
  - OPEN: WMS fails > 50% calls → Reject new calls, queue to DLQ
  - HALF_OPEN: Probe with single test call; if OK → CLOSED
  - Timeout: 30s wait before retry
  ```

- **Failover Strategy**:
  1. Order dispatch fails → catch exception
  2. Log to `failed_order_dispatch` table
  3. Publish to Kafka DLQ topic "order-dispatch-dlq"
  4. DLQ consumer: retry every 5 min with exponential backoff
  5. Notify ops team if 100+ orders in DLQ

- **Recovery**:
  - Replay DLQ when WMS health OK
  - Process in FIFO order to maintain consistency
  - Alert customer on order page: "Fulfillment delayed, will process soon"

**Graceful Degradation**: Accept orders but don't dispatch; customer can't cancel. Once WMS recovers, auto-dispatch all queued orders.

---

### Q6: How do you track order fulfillment status across systems?

**Answer**: Event sourcing + state machine. Shipment record transitions through states; audit trail logged to MongoDB.

**Design**:
```
Shipment State Machine:
  PENDING → CONFIRMED → PICKED → PACKED → SHIPPED → IN_TRANSIT → DELIVERED

Events:
  - ShipmentDispatchedEvent (e-commerce → WMS)
  - ShipmentConfirmedEvent (WMS confirms order)
  - ShipmentPickedEvent (WMS items picked from bin)
  - ShipmentPackedEvent (WMS items packed)
  - ShipmentShippedEvent (WMS handed off to carrier)
  - ShipmentInTransitEvent (Carrier update)
  - ShipmentDeliveredEvent (Carrier delivery confirmation)
```

**Data Storage**:
- **PostgreSQL**: Current shipment state (for queries)
- **MongoDB**: Event audit log (immutable, all state transitions)
- **Redis**: Latest tracking update (for real-time dashboard)

**Customer Visibility**:
```
/orders/{id}/shipments
[
  {
    "shipment_id": "SHIP-001",
    "warehouse_id": 1,
    "status": "SHIPPED",
    "tracking_number": "ABC123",
    "carrier": "FedEx",
    "expected_delivery": "2024-04-05",
    "events": [
      {"status": "CONFIRMED", "timestamp": "2024-04-01T10:00Z"},
      {"status": "PICKED", "timestamp": "2024-04-01T14:00Z"},
      {"status": "SHIPPED", "timestamp": "2024-04-02T08:00Z"}
    ]
  }
]
```

---

## 5. Microservices Breakdown

| Service | Responsibility | Tech | Instances |
|---------|-----------------|------|-----------|
| **Fulfillment Service** | Order dispatch orchestration | Spring Boot + PostgreSQL | 5–10 |
| **WMS Adapter Layer** | Vendor-specific integrations | Spring Boot + Multiple APIs | 3–8 |
| **Inventory Sync Service** | Real-time inventory aggregation | Spring Boot + Kafka + Redis | 3–5 |
| **Shipment Tracker** | Shipment status tracking, customer notifications | Spring Boot + PostgreSQL + MongoDB | 5–10 |
| **Warehouse Service** | Warehouse config, capacity planning | Spring Boot + PostgreSQL | 2–3 |
| **Webhook Handler** | Async WMS updates ingestion | Spring Boot (lightweight) | 3–5 |

---

## 6. Database Design

### PostgreSQL Schema (OLTP)

```sql
-- Warehouses (config)
CREATE TABLE warehouses (
    id BIGSERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    location VARCHAR(255),
    wms_id VARCHAR(100) NOT NULL,
    wms_type VARCHAR(50), -- MANHATTAN, SAP_EWM, CUSTOM, etc.
    capacity INT,
    active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL,
    UNIQUE(wms_id),
    INDEX idx_wms_type (wms_type)
);

-- Orders (from order service, reference)
CREATE TABLE orders (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    status VARCHAR(50), -- CONFIRMED, DISPATCHED, FULFILLING, DELIVERED
    total_amount DECIMAL(12, 2),
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    INDEX idx_status (status),
    INDEX idx_customer_id (customer_id)
);

-- Order Items (what customer ordered)
CREATE TABLE order_items (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10, 2),
    INDEX idx_order_id (order_id)
);

-- Shipments (fulfillment units)
CREATE TABLE shipments (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    warehouse_id BIGINT NOT NULL REFERENCES warehouses(id),
    wms_order_id VARCHAR(100) NOT NULL, -- WMS reference
    status VARCHAR(50), -- PENDING, CONFIRMED, PICKED, PACKED, SHIPPED, DELIVERED
    tracking_number VARCHAR(100),
    carrier VARCHAR(50), -- FedEx, UPS, USPS, etc.
    expected_delivery_date DATE,
    shipped_at TIMESTAMP,
    delivered_at TIMESTAMP,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    UNIQUE(order_id, warehouse_id), -- One shipment per order per warehouse
    INDEX idx_order_id (order_id),
    INDEX idx_warehouse_id (warehouse_id),
    INDEX idx_tracking_number (tracking_number),
    INDEX idx_status (status)
);

-- Shipment Items (which items in this shipment)
CREATE TABLE shipment_items (
    id BIGSERIAL PRIMARY KEY,
    shipment_id BIGINT NOT NULL REFERENCES shipments(id),
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    INDEX idx_shipment_id (shipment_id)
);

-- Backorders (partial fulfillment)
CREATE TABLE backorders (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL REFERENCES orders(id),
    product_id BIGINT NOT NULL,
    requested_quantity INT,
    fulfilled_quantity INT DEFAULT 0,
    status VARCHAR(50), -- PENDING, PARTIAL, FULFILLED
    expected_availability_date DATE,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    INDEX idx_order_id (order_id),
    INDEX idx_status (status)
);

-- Failed Order Dispatch (for DLQ)
CREATE TABLE failed_order_dispatches (
    id BIGSERIAL PRIMARY KEY,
    order_id BIGINT NOT NULL,
    warehouse_id BIGINT NOT NULL,
    error_message TEXT,
    retry_count INT DEFAULT 0,
    last_retry_at TIMESTAMP,
    status VARCHAR(50), -- PENDING, RETRYING, FAILED
    created_at TIMESTAMP NOT NULL,
    INDEX idx_order_id (order_id),
    INDEX idx_status (status)
);

-- Fulfillment Audit Trail (immutable)
CREATE TABLE fulfillment_events (
    id BIGSERIAL PRIMARY KEY,
    shipment_id BIGINT NOT NULL REFERENCES shipments(id),
    event_type VARCHAR(100), -- DISPATCHED, CONFIRMED, PICKED, SHIPPED, etc.
    event_data JSONB,
    source VARCHAR(50), -- WMS, CARRIER, MANUAL
    created_at TIMESTAMP NOT NULL,
    INDEX idx_shipment_id (shipment_id),
    INDEX idx_event_type (event_type),
    INDEX idx_created_at (created_at)
);

-- Warehouse Inventory (current state, synced from WMS)
CREATE TABLE warehouse_inventory (
    id BIGSERIAL PRIMARY KEY,
    warehouse_id BIGINT NOT NULL REFERENCES warehouses(id),
    product_id BIGINT NOT NULL,
    quantity INT NOT NULL,
    reserved_quantity INT DEFAULT 0,
    available_quantity INT GENERATED ALWAYS AS (quantity - reserved_quantity),
    last_sync_at TIMESTAMP,
    updated_at TIMESTAMP NOT NULL,
    UNIQUE(warehouse_id, product_id),
    INDEX idx_warehouse_id (warehouse_id),
    INDEX idx_product_id (product_id)
);

-- Idempotency Keys (for webhook deduplication)
CREATE TABLE idempotency_keys (
    id BIGSERIAL PRIMARY KEY,
    idempotency_key VARCHAR(255) UNIQUE NOT NULL,
    shipment_id BIGINT,
    result JSONB,
    created_at TIMESTAMP NOT NULL,
    TTL TIMESTAMP, -- Auto-expire after 24 hours
    INDEX idx_idempotency_key (idempotency_key)
);
```

### MongoDB Collections (Audit Trail)

```json
// fulfillment_event_log
{
    "_id": ObjectId(),
    "shipment_id": 123,
    "order_id": 456,
    "warehouse_id": 1,
    "wms_order_id": "WMS-789",
    "event_type": "SHIPMENT_SHIPPED",
    "event_timestamp": ISODate("2024-04-02T08:00:00Z"),
    "event_source": "MANHATTAN_WMS",
    "payload": {
        "tracking_number": "FDX123456",
        "carrier": "FedEx",
        "expected_delivery": "2024-04-05"
    },
    "created_at": ISODate("2024-04-02T08:00:00Z")
}

// warehouse_sync_log
{
    "_id": ObjectId(),
    "warehouse_id": 1,
    "wms_type": "MANHATTAN",
    "sync_timestamp": ISODate("2024-04-02T10:00:00Z"),
    "inventory_changes": [
        {
            "product_id": 100,
            "old_quantity": 500,
            "new_quantity": 450,
            "delta": -50
        }
    ],
    "total_items_synced": 12500,
    "sync_duration_ms": 2345,
    "status": "SUCCESS"
}
```

---

## 7. Redis Data Structures

```redis
# Current Shipment Status (hot read for customer dashboard)
shipment:{shipment_id} = HASH {
    "order_id": "456",
    "status": "SHIPPED",
    "tracking_number": "FDX123456",
    "carrier": "FedEx",
    "expected_delivery": "2024-04-05",
    "last_update": "1712151600"
}
TTL: 86400 seconds (1 day, auto-refresh)

# Order to Shipments Mapping
order:{order_id}:shipments = SET {
    "shipment_123",
    "shipment_124",
    "shipment_125"
}
TTL: 604800 seconds (7 days)

# Latest Inventory Snapshot (per warehouse/product)
inventory:warehouse:{warehouse_id}:product:{product_id} = STRING "450"
inventory:warehouse:{warehouse_id}:updated_at = STRING "1712151600"
TTL: 600 seconds (5 minutes, refresh on sync)

# WMS Circuit Breaker State
wms:circuit-breaker:{wms_id} = HASH {
    "state": "CLOSED",
    "failure_count": "2",
    "last_failure_at": "1712151200"
}
TTL: 3600 seconds (expires state)

# Webhook Idempotency (dedup retries)
webhook:idempotency:{key} = STRING "{result_json}"
TTL: 86400 seconds (24 hours)

# Shipment in Transit (updates from carrier)
shipment:in-transit:{tracking_number} = HASH {
    "carrier": "FedEx",
    "last_location": "Memphis, TN",
    "last_update": "1712151900",
    "estimated_delivery": "2024-04-05"
}
TTL: 2592000 seconds (30 days)

# Backorder Alerts (push notifications)
backorder:alert:{product_id} = SORTED SET {
    order_id: score = created_at_timestamp
}
```

---

## 8. Kafka Event Flow

### Topics & Events

| Topic | Events | Partition Key | Retention |
|-------|--------|---------------|-----------|
| `order-confirmed` | OrderConfirmedEvent | order_id | 90 days |
| `fulfillment-dispatch` | OrderDispatchedEvent, OrderDispatchFailedEvent | order_id | 90 days |
| `fulfillment-updates` | ShipmentConfirmedEvent, ShipmentPickedEvent, ShipmentShippedEvent, ShipmentDeliveredEvent | shipment_id | 180 days |
| `inventory-sync` | InventoryDeltaEvent, InventorySyncStartedEvent | warehouse_id | 365 days |
| `inventory-updates` | InventoryUpdatedEvent | warehouse_id | 30 days |
| `shipment-status` | ShipmentStatusChangedEvent | order_id | 90 days |
| `backorder-alerts` | BackorderCreatedEvent, BackorderFulfilledEvent | product_id | 30 days |
| `fulfillment-dlq` | RetriableOrderDispatchEvent | order_id | 7 days |

### Event Examples

```json
// OrderConfirmedEvent
{
    "event_id": "evt-50000001",
    "event_type": "ORDER_CONFIRMED",
    "order_id": 789,
    "customer_id": 123,
    "items": [
        {"product_id": 1001, "quantity": 2},
        {"product_id": 1002, "quantity": 1}
    ],
    "total_amount": 150.00,
    "timestamp": "2024-04-01T10:30:00Z"
}

// ShipmentShippedEvent
{
    "event_id": "evt-50000002",
    "event_type": "SHIPMENT_SHIPPED",
    "shipment_id": 456,
    "order_id": 789,
    "warehouse_id": 2,
    "wms_order_id": "WMS-2024-001",
    "tracking_number": "FDX123456",
    "carrier": "FedEx",
    "expected_delivery": "2024-04-05",
    "timestamp": "2024-04-02T08:00:00Z"
}

// InventoryDeltaEvent
{
    "event_id": "evt-50000003",
    "event_type": "INVENTORY_DELTA",
    "warehouse_id": 2,
    "wms_type": "MANHATTAN",
    "deltas": [
        {"product_id": 1001, "old_qty": 500, "new_qty": 450, "reason": "PICKED"},
        {"product_id": 1002, "old_qty": 200, "new_qty": 150, "reason": "PICKED"}
    ],
    "sync_timestamp": "2024-04-02T10:00:00Z"
}
```

---

## 9. Implementation Code

### 9.1 FulfillmentService (Orchestration)

```java
@Service
@RequiredArgsConstructor
public class FulfillmentService {
    private final FulfillmentSplitter fulfillmentSplitter;
    private final WMSAdapterFactory wmsAdapterFactory;
    private final ShipmentRepository shipmentRepository;
    private final WarehouseRepository warehouseRepository;
    private final InventorySyncService inventorySyncService;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;
    private static final Logger logger = LoggerFactory.getLogger(FulfillmentService.class);

    @KafkaListener(topics = "order-confirmed", groupId = "fulfillment-service")
    public void handleOrderConfirmedEvent(ConsumerRecord<String, String> record) {
        try {
            String message = record.value();
            Map<String, Object> event = objectMapper.readValue(message, Map.class);

            Long orderId = ((Number) event.get("order_id")).longValue();
            List<Map<String, Object>> items = (List<Map<String, Object>>) event.get("items");

            logger.info("Processing order {} with {} items", orderId, items.size());

            // Dispatch order to WMS
            dispatchOrder(orderId, items);
        } catch (Exception e) {
            logger.error("Failed to process OrderConfirmedEvent", e);
            // Will be retried by Kafka consumer
        }
    }

    public void dispatchOrder(Long orderId, List<Map<String, Object>> items) {
        try {
            // Step 1: Convert items to internal format
            List<OrderItem> orderItems = items.stream()
                .map(item -> new OrderItem(
                    ((Number) item.get("product_id")).longValue(),
                    ((Number) item.get("quantity")).intValue()
                ))
                .collect(Collectors.toList());

            // Step 2: Check inventory & split across warehouses
            List<ShipmentPlan> shipmentPlans = fulfillmentSplitter.splitOrder(orderItems);

            if (shipmentPlans.isEmpty()) {
                logger.warn("No warehouse available for order {}", orderId);
                publishOrderDispatchFailedEvent(orderId, "No warehouse available");
                return;
            }

            // Step 3: Reserve inventory
            for (ShipmentPlan plan : shipmentPlans) {
                reserveInventory(plan.getWarehouseId(), plan.getItems());
            }

            // Step 4: Dispatch each shipment to WMS
            for (ShipmentPlan plan : shipmentPlans) {
                dispatchShipmentToWMS(orderId, plan);
            }

            logger.info("Order {} dispatched to {} warehouse(s)", orderId, shipmentPlans.size());
            publishOrderDispatchedEvent(orderId, shipmentPlans.size());

        } catch (Exception e) {
            logger.error("Dispatch failed for order {}", orderId, e);
            publishOrderDispatchFailedEvent(orderId, e.getMessage());

            // Queue to DLQ for retry
            queueFailedDispatch(orderId, e.getMessage());
        }
    }

    private void dispatchShipmentToWMS(Long orderId, ShipmentPlan plan) throws Exception {
        Warehouse warehouse = warehouseRepository.findById(plan.getWarehouseId())
            .orElseThrow(() -> new WarehouseNotFoundException(plan.getWarehouseId()));

        // Get appropriate adapter for this WMS
        WMSAdapter adapter = wmsAdapterFactory.getAdapter(warehouse.getWmsType());

        // Convert order to WMS-specific format
        WMSOrder wmsOrder = new WMSOrder();
        wmsOrder.setOrderId(orderId.toString());
        wmsOrder.setWarehouseId(warehouse.getWmsId());
        wmsOrder.setItems(plan.getItems().stream()
            .map(item -> new WMSOrderItem(item.getProductId(), item.getQuantity()))
            .collect(Collectors.toList()));

        // Dispatch via adapter (with circuit breaker)
        WMSDispatchResult result = adapter.dispatchOrder(wmsOrder);

        if (!result.isSuccess()) {
            throw new WMSDispatchException("WMS dispatch failed: " + result.getErrorMessage());
        }

        // Save shipment record
        Shipment shipment = new Shipment();
        shipment.setOrderId(orderId);
        shipment.setWarehouseId(plan.getWarehouseId());
        shipment.setWmsOrderId(result.getWmsOrderId());
        shipment.setStatus(ShipmentStatus.PENDING);
        shipment.setCreatedAt(LocalDateTime.now());

        Shipment savedShipment = shipmentRepository.save(shipment);

        // Cache shipment mapping
        String cacheKey = "shipment:" + savedShipment.getId();
        redisTemplate.opsForHash().putAll(cacheKey, new HashMap<String, Object>() {{
            put("order_id", orderId.toString());
            put("status", ShipmentStatus.PENDING.name());
            put("warehouse_id", plan.getWarehouseId().toString());
        }});
        redisTemplate.expire(cacheKey, Duration.ofHours(24));

        logger.info("Shipment {} created for order {} in warehouse {}",
            savedShipment.getId(), orderId, plan.getWarehouseId());
    }

    private void reserveInventory(Long warehouseId, List<OrderItem> items) {
        for (OrderItem item : items) {
            String inventoryKey = "inventory:warehouse:" + warehouseId + ":product:" + item.getProductId();
            Long currentQty = Long.parseLong(redisTemplate.opsForValue().get(inventoryKey));
            if (currentQty < item.getQuantity()) {
                throw new InsufficientInventoryException(
                    "Warehouse " + warehouseId + " has only " + currentQty + " of product " + item.getProductId());
            }
            // Decrement (simplified; actual implementation needs better locking)
            redisTemplate.opsForValue().decrement(inventoryKey, item.getQuantity());
        }
    }

    private void publishOrderDispatchedEvent(Long orderId, int shipmentCount) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "ORDER_DISPATCHED");
            event.put("order_id", orderId);
            event.put("shipment_count", shipmentCount);
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("fulfillment-dispatch", orderId.toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish OrderDispatchedEvent", e);
        }
    }

    private void publishOrderDispatchFailedEvent(Long orderId, String reason) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "ORDER_DISPATCH_FAILED");
            event.put("order_id", orderId);
            event.put("reason", reason);
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("fulfillment-dispatch", orderId.toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish OrderDispatchFailedEvent", e);
        }
    }

    private void queueFailedDispatch(Long orderId, String errorMessage) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "RETRIABLE_ORDER_DISPATCH");
            event.put("order_id", orderId);
            event.put("error", errorMessage);
            event.put("retry_count", 0);
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("fulfillment-dlq", orderId.toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to queue failed dispatch", e);
        }
    }
}

@Data
class OrderItem {
    private Long productId;
    private Integer quantity;

    public OrderItem(Long productId, Integer quantity) {
        this.productId = productId;
        this.quantity = quantity;
    }
}

@Data
class ShipmentPlan {
    private Long warehouseId;
    private List<OrderItem> items;
}
```

### 9.2 WMSAdapter Pattern

```java
// Interface (unified across all WMS vendors)
public interface WMSAdapter {
    String getName();
    WMSDispatchResult dispatchOrder(WMSOrder order) throws WMSException;
    InventorySnapshot getInventory() throws WMSException;
}

// Factory
@Component
@RequiredArgsConstructor
public class WMSAdapterFactory {
    private final ManhattanWMSAdapter manhattanAdapter;
    private final SAPEWMAdapter sapAdapter;
    private final CustomLegacyAdapter customAdapter;

    public WMSAdapter getAdapter(WMSType wmsType) {
        switch (wmsType) {
            case MANHATTAN:
                return manhattanAdapter;
            case SAP_EWM:
                return sapAdapter;
            case CUSTOM_LEGACY:
                return customAdapter;
            default:
                throw new UnsupportedWMSException("Unknown WMS type: " + wmsType);
        }
    }
}

// Concrete Implementation: Manhattan WMS (HTTP REST)
@Component
@RequiredArgsConstructor
public class ManhattanWMSAdapter implements WMSAdapter {
    private final RestTemplate restTemplate;
    private final ManhattanProperties properties;
    private final CircuitBreaker circuitBreaker;
    private static final Logger logger = LoggerFactory.getLogger(ManhattanWMSAdapter.class);

    @Override
    public String getName() {
        return "MANHATTAN";
    }

    @Override
    public WMSDispatchResult dispatchOrder(WMSOrder order) throws WMSException {
        // Convert to Manhattan-specific format
        ManhattanOrderRequest request = convertOrder(order);

        try {
            // Call with circuit breaker
            WMSDispatchResult result = circuitBreaker.executeSupplier(() -> {
                ResponseEntity<ManhattanOrderResponse> response = restTemplate.postForEntity(
                    properties.getBaseUrl() + "/orders",
                    request,
                    ManhattanOrderResponse.class,
                    new HttpHeaders() {{
                        setBearerAuth(properties.getApiKey());
                    }}
                );

                if (response.getStatusCode() != HttpStatus.CREATED) {
                    throw new WMSException("Manhattan API returned " + response.getStatusCode());
                }

                ManhattanOrderResponse body = response.getBody();
                return new WMSDispatchResult(true, body.getOrderId(), null);
            });

            logger.info("Order {} dispatched to Manhattan successfully", order.getOrderId());
            return result;

        } catch (Exception e) {
            logger.error("Failed to dispatch order {} to Manhattan", order.getOrderId(), e);
            return new WMSDispatchResult(false, null, e.getMessage());
        }
    }

    @Override
    public InventorySnapshot getInventory() throws WMSException {
        try {
            ResponseEntity<ManhattanInventoryResponse> response = restTemplate.getForEntity(
                properties.getBaseUrl() + "/inventory",
                ManhattanInventoryResponse.class,
                new HttpHeaders() {{
                    setBearerAuth(properties.getApiKey());
                }}
            );

            ManhattanInventoryResponse body = response.getBody();
            return convertInventory(body);

        } catch (Exception e) {
            logger.error("Failed to fetch inventory from Manhattan", e);
            throw new WMSException("Inventory fetch failed: " + e.getMessage());
        }
    }

    private ManhattanOrderRequest convertOrder(WMSOrder order) {
        // Anti-corruption layer: e-commerce model → Manhattan model
        ManhattanOrderRequest request = new ManhattanOrderRequest();
        request.setOrderId(order.getOrderId());
        request.setWarehouse(order.getWarehouseId());
        request.setLines(order.getItems().stream()
            .map(item -> new ManhattanOrderRequest.Line(item.getProductId(), item.getQuantity()))
            .collect(Collectors.toList()));
        return request;
    }

    private InventorySnapshot convertInventory(ManhattanInventoryResponse response) {
        // Anti-corruption layer: Manhattan response → internal model
        InventorySnapshot snapshot = new InventorySnapshot();
        snapshot.setWarehouseId(response.getWarehouseId());
        snapshot.setItems(response.getItems().stream()
            .collect(Collectors.toMap(
                item -> Long.parseLong(item.getSku()),
                item -> item.getQuantity()
            )));
        return snapshot;
    }
}

// Concrete Implementation: SAP EWM (SOAP)
@Component
@RequiredArgsConstructor
public class SAPEWMAdapter implements WMSAdapter {
    private final SAPEWMWebServiceClient soapClient;
    private final CircuitBreaker circuitBreaker;
    private static final Logger logger = LoggerFactory.getLogger(SAPEWMAdapter.class);

    @Override
    public String getName() {
        return "SAP_EWM";
    }

    @Override
    public WMSDispatchResult dispatchOrder(WMSOrder order) throws WMSException {
        SAPOrder sapOrder = convertToSAPOrder(order);

        try {
            WMSDispatchResult result = circuitBreaker.executeSupplier(() -> {
                SAPOrderResponse response = soapClient.createOrder(sapOrder);
                if (!response.isSuccess()) {
                    throw new WMSException("SAP returned error: " + response.getErrorMessage());
                }
                return new WMSDispatchResult(true, response.getOrderId(), null);
            });

            logger.info("Order {} dispatched to SAP successfully", order.getOrderId());
            return result;

        } catch (Exception e) {
            logger.error("Failed to dispatch order {} to SAP", order.getOrderId(), e);
            return new WMSDispatchResult(false, null, e.getMessage());
        }
    }

    @Override
    public InventorySnapshot getInventory() throws WMSException {
        try {
            SAPInventoryResponse response = soapClient.getInventory();
            return convertFromSAPInventory(response);
        } catch (Exception e) {
            logger.error("Failed to fetch inventory from SAP", e);
            throw new WMSException("Inventory fetch failed: " + e.getMessage());
        }
    }

    private SAPOrder convertToSAPOrder(WMSOrder order) {
        SAPOrder sapOrder = new SAPOrder();
        sapOrder.setOrderNumber(order.getOrderId());
        sapOrder.setWarehouseCode(order.getWarehouseId());
        // Additional SAP-specific fields...
        return sapOrder;
    }

    private InventorySnapshot convertFromSAPInventory(SAPInventoryResponse response) {
        // Conversion logic
        return new InventorySnapshot();
    }
}

@Data
class WMSDispatchResult {
    private boolean success;
    private String wmsOrderId;
    private String errorMessage;

    public WMSDispatchResult(boolean success, String wmsOrderId, String errorMessage) {
        this.success = success;
        this.wmsOrderId = wmsOrderId;
        this.errorMessage = errorMessage;
    }
}

class WMSException extends Exception {
    public WMSException(String message) {
        super(message);
    }
}

class WMSDispatchException extends WMSException {
    public WMSDispatchException(String message) {
        super(message);
    }
}
```

### 9.3 FulfillmentSplitter (Bin Packing)

```java
@Service
@RequiredArgsConstructor
public class FulfillmentSplitter {
    private final WarehouseInventoryRepository inventoryRepository;
    private final WarehouseRepository warehouseRepository;
    private static final Logger logger = LoggerFactory.getLogger(FulfillmentSplitter.class);

    public List<ShipmentPlan> splitOrder(List<OrderItem> items) {
        List<ShipmentPlan> plans = new ArrayList<>();

        // Step 1: Fetch all warehouses with their current inventory
        List<Warehouse> activeWarehouses = warehouseRepository.findByActiveTrue();
        Map<Long, Map<Long, Integer>> warehouseInventory = loadInventory(activeWarehouses);

        // Step 2: Greedy bin packing algorithm
        // Try to fit as many items as possible in one warehouse
        Set<Long> remainingItems = new HashSet<>();
        for (OrderItem item : items) {
            remainingItems.add(item.getProductId());
        }

        for (Warehouse warehouse : activeWarehouses) {
            if (remainingItems.isEmpty()) {
                break;
            }

            ShipmentPlan plan = new ShipmentPlan();
            plan.setWarehouseId(warehouse.getId());
            plan.setItems(new ArrayList<>());

            Map<Long, Integer> warehouseQty = warehouseInventory.get(warehouse.getId());
            if (warehouseQty == null) {
                continue;
            }

            // Allocate items to this warehouse
            Iterator<OrderItem> iterator = items.iterator();
            while (iterator.hasNext()) {
                OrderItem item = iterator.next();
                Integer available = warehouseQty.getOrDefault(item.getProductId(), 0);

                if (available >= item.getQuantity()) {
                    plan.getItems().add(item);
                    warehouseQty.put(item.getProductId(), available - item.getQuantity());
                    remainingItems.remove(item.getProductId());
                }
            }

            if (!plan.getItems().isEmpty()) {
                plans.add(plan);
            }
        }

        // Step 3: Check if all items allocated
        if (!remainingItems.isEmpty()) {
            logger.warn("Could not allocate {} items across warehouses", remainingItems.size());
            // Could trigger backorder logic here
        }

        logger.info("Order split into {} shipments", plans.size());
        return plans;
    }

    private Map<Long, Map<Long, Integer>> loadInventory(List<Warehouse> warehouses) {
        Map<Long, Map<Long, Integer>> result = new HashMap<>();

        for (Warehouse warehouse : warehouses) {
            List<WarehouseInventory> inventory = inventoryRepository.findByWarehouseId(warehouse.getId());
            Map<Long, Integer> warehouseQty = new HashMap<>();

            for (WarehouseInventory item : inventory) {
                warehouseQty.put(item.getProductId(), item.getAvailableQuantity());
            }

            result.put(warehouse.getId(), warehouseQty);
        }

        return result;
    }
}
```

### 9.4 FulfillmentWebhookController (Async Updates)

```java
@RestController
@RequestMapping("/webhooks/fulfillment")
@RequiredArgsConstructor
public class FulfillmentWebhookController {
    private final ShipmentRepository shipmentRepository;
    private final FulfillmentEventRepository eventRepository;
    private final IdempotencyKeyRepository idempotencyKeyRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;
    private final WMSHMACValidator hmacValidator;
    private static final Logger logger = LoggerFactory.getLogger(FulfillmentWebhookController.class);

    @PostMapping
    public ResponseEntity<?> handleFulfillmentUpdate(
            @RequestBody String payload,
            @RequestHeader("X-Webhook-Signature") String signature,
            @RequestHeader("X-Idempotency-Key") String idempotencyKey) {

        try {
            // Step 1: Validate webhook signature (prevent spoofing)
            if (!hmacValidator.validate(payload, signature)) {
                logger.warn("Invalid webhook signature");
                return ResponseEntity.status(HttpStatus.UNAUTHORIZED).build();
            }

            // Step 2: Check idempotency (prevent duplicate processing)
            IdempotencyKey existing = idempotencyKeyRepository.findByKey(idempotencyKey);
            if (existing != null && existing.getResult() != null) {
                logger.info("Webhook already processed: {}", idempotencyKey);
                return ResponseEntity.ok(existing.getResult());
            }

            // Step 3: Parse webhook payload
            Map<String, Object> data = objectMapper.readValue(payload, Map.class);
            String eventType = (String) data.get("event_type");
            String wmsOrderId = (String) data.get("order_id");

            logger.info("Received {} event for WMS order {}", eventType, wmsOrderId);

            // Step 4: Update shipment status
            Shipment shipment = shipmentRepository.findByWmsOrderId(wmsOrderId)
                .orElseThrow(() -> new ShipmentNotFoundException(wmsOrderId));

            updateShipmentStatus(shipment, eventType, data);

            // Step 5: Publish event to Kafka
            publishFulfillmentUpdateEvent(shipment, eventType, data);

            // Step 6: Log event in MongoDB (audit trail)
            logFulfillmentEvent(shipment, eventType, data);

            // Step 7: Save idempotency result
            Map<String, Object> response = new HashMap<>();
            response.put("shipment_id", shipment.getId());
            response.put("status", shipment.getStatus());

            IdempotencyKey idempotencyRecord = new IdempotencyKey();
            idempotencyRecord.setKey(idempotencyKey);
            idempotencyRecord.setShipmentId(shipment.getId());
            idempotencyRecord.setResult(objectMapper.writeValueAsString(response));
            idempotencyKeyRepository.save(idempotencyRecord);

            logger.info("Successfully processed webhook for shipment {}", shipment.getId());
            return ResponseEntity.ok(response);

        } catch (JsonProcessingException e) {
            logger.error("Failed to parse webhook payload", e);
            return ResponseEntity.status(HttpStatus.BAD_REQUEST).body("Invalid payload");
        } catch (Exception e) {
            logger.error("Failed to process fulfillment webhook", e);
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Processing failed");
        }
    }

    private void updateShipmentStatus(Shipment shipment, String eventType, Map<String, Object> data) {
        switch (eventType) {
            case "ORDER_CONFIRMED":
                shipment.setStatus(ShipmentStatus.CONFIRMED);
                break;
            case "ITEMS_PICKED":
                shipment.setStatus(ShipmentStatus.PICKED);
                break;
            case "ITEMS_PACKED":
                shipment.setStatus(ShipmentStatus.PACKED);
                break;
            case "SHIPMENT_SHIPPED":
                shipment.setStatus(ShipmentStatus.SHIPPED);
                shipment.setTrackingNumber((String) data.get("tracking_number"));
                shipment.setCarrier((String) data.get("carrier"));
                shipment.setShippedAt(LocalDateTime.now());
                break;
            case "SHIPMENT_DELIVERED":
                shipment.setStatus(ShipmentStatus.DELIVERED);
                shipment.setDeliveredAt(LocalDateTime.now());
                break;
        }
        shipment.setUpdatedAt(LocalDateTime.now());
        shipmentRepository.save(shipment);
    }

    private void publishFulfillmentUpdateEvent(Shipment shipment, String eventType, Map<String, Object> data) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", eventType);
            event.put("shipment_id", shipment.getId());
            event.put("order_id", shipment.getOrderId());
            event.put("status", shipment.getStatus().name());
            event.put("tracking_number", shipment.getTrackingNumber());
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("fulfillment-updates", shipment.getOrderId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish fulfillment event", e);
        }
    }

    private void logFulfillmentEvent(Shipment shipment, String eventType, Map<String, Object> data) {
        FulfillmentEvent event = new FulfillmentEvent();
        event.setShipmentId(shipment.getId());
        event.setOrderId(shipment.getOrderId());
        event.setEventType(eventType);
        event.setEventData(data);
        event.setSource("WMS");
        event.setCreatedAt(LocalDateTime.now());
        eventRepository.save(event);
    }
}

@Data
class IdempotencyKey {
    @Id
    private String key;
    private Long shipmentId;
    private String result;
    private LocalDateTime createdAt;

    @PrePersist
    void prePersist() {
        createdAt = LocalDateTime.now();
    }
}
```

### 9.5 InventorySyncService

```java
@Service
@RequiredArgsConstructor
public class InventorySyncService {
    private final WMSAdapterFactory wmsAdapterFactory;
    private final WarehouseRepository warehouseRepository;
    private final WarehouseInventoryRepository inventoryRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;
    private static final Logger logger = LoggerFactory.getLogger(InventorySyncService.class);

    @Scheduled(fixedRate = 300000) // Every 5 minutes
    public void syncAllWarehouses() {
        logger.info("Starting warehouse inventory sync");

        List<Warehouse> warehouses = warehouseRepository.findByActiveTrue();
        for (Warehouse warehouse : warehouses) {
            syncWarehouse(warehouse);
        }

        logger.info("Completed warehouse inventory sync");
    }

    private void syncWarehouse(Warehouse warehouse) {
        try {
            WMSAdapter adapter = wmsAdapterFactory.getAdapter(warehouse.getWmsType());
            InventorySnapshot snapshot = adapter.getInventory();

            // Compare with current inventory and publish deltas
            List<InventoryDelta> deltas = computeDeltas(warehouse, snapshot);

            if (deltas.isEmpty()) {
                logger.debug("No inventory changes for warehouse {}", warehouse.getId());
                return;
            }

            // Update PostgreSQL
            updateInventory(warehouse.getId(), snapshot);

            // Update Redis cache
            updateRedisCache(warehouse.getId(), snapshot);

            // Publish delta events to Kafka
            publishInventoryDeltas(warehouse, deltas);

            logger.info("Synced {} items for warehouse {}", deltas.size(), warehouse.getId());

        } catch (Exception e) {
            logger.error("Failed to sync warehouse {}", warehouse.getId(), e);
            // Trigger alert if consistent failures
        }
    }

    private List<InventoryDelta> computeDeltas(Warehouse warehouse, InventorySnapshot snapshot) {
        List<InventoryDelta> deltas = new ArrayList<>();

        List<WarehouseInventory> current = inventoryRepository.findByWarehouseId(warehouse.getId());
        Map<Long, Integer> currentMap = current.stream()
            .collect(Collectors.toMap(WarehouseInventory::getProductId, WarehouseInventory::getQuantity));

        for (Map.Entry<Long, Integer> entry : snapshot.getItems().entrySet()) {
            Long productId = entry.getKey();
            Integer newQty = entry.getValue();
            Integer oldQty = currentMap.getOrDefault(productId, 0);

            if (!oldQty.equals(newQty)) {
                InventoryDelta delta = new InventoryDelta();
                delta.setProductId(productId);
                delta.setOldQuantity(oldQty);
                delta.setNewQuantity(newQty);
                delta.setDelta(newQty - oldQty);
                deltas.add(delta);
            }
        }

        return deltas;
    }

    private void updateInventory(Long warehouseId, InventorySnapshot snapshot) {
        for (Map.Entry<Long, Integer> entry : snapshot.getItems().entrySet()) {
            WarehouseInventory inventory = inventoryRepository
                .findByWarehouseIdAndProductId(warehouseId, entry.getKey())
                .orElse(new WarehouseInventory());

            inventory.setWarehouseId(warehouseId);
            inventory.setProductId(entry.getKey());
            inventory.setQuantity(entry.getValue());
            inventory.setUpdatedAt(LocalDateTime.now());

            inventoryRepository.save(inventory);
        }
    }

    private void updateRedisCache(Long warehouseId, InventorySnapshot snapshot) {
        for (Map.Entry<Long, Integer> entry : snapshot.getItems().entrySet()) {
            String key = "inventory:warehouse:" + warehouseId + ":product:" + entry.getKey();
            redisTemplate.opsForValue().set(key, entry.getValue().toString(), Duration.ofMinutes(10));
        }

        String lastSyncKey = "inventory:warehouse:" + warehouseId + ":updated_at";
        redisTemplate.opsForValue().set(lastSyncKey, String.valueOf(System.currentTimeMillis()), Duration.ofHours(1));
    }

    private void publishInventoryDeltas(Warehouse warehouse, List<InventoryDelta> deltas) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "INVENTORY_DELTA");
            event.put("warehouse_id", warehouse.getId());
            event.put("wms_type", warehouse.getWmsType().name());
            event.put("deltas", deltas);
            event.put("sync_timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("inventory-sync", warehouse.getId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish inventory delta event", e);
        }
    }
}

@Data
class InventorySnapshot {
    private Long warehouseId;
    private Map<Long, Integer> items; // product_id -> quantity
}

@Data
class InventoryDelta {
    private Long productId;
    private Integer oldQuantity;
    private Integer newQuantity;
    private Integer delta;
}
```

---

## 10. Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| **WMS API Timeout** | Order dispatch hangs | Timeout: 10s, circuit breaker after 50% failures, queue to DLQ |
| **WMS Down (maintenance)** | Orders can't dispatch | Queue to Kafka topic, alert ops, replay when WMS recovers |
| **Inventory Sync Lag** | Stale inventory shown to users | Display "updated 5 min ago" timestamp, fallback to Redis cache |
| **Split Shipment Fails** | Partial fulfillment lost | Retry with different warehouse assignment, enable backorder |
| **Webhook Duplicate** | Order status updated twice | Idempotency key (WMS order ID + timestamp) with Redis dedup |
| **Shipment Never Confirmed** | Order stuck in PENDING | SLA timer: alert after 1 hour, escalate to warehouse |
| **Inventory Reserved but Never Used** | False scarcity | TTL on reservations (24h); reconcile daily with WMS |
| **Network Partition | Orders dispatch to multiple warehouses | Idempotent dispatch: check if order already in WMS before re-dispatching |

---

## 11. Scaling Strategy

### Horizontal Scaling
- **Fulfillment Service**: 10 instances (stateless, load-balanced)
- **Inventory Sync Service**: 3 instances (Kafka consumer group auto-scales)
- **Webhook Handler**: 5 instances (lightweight, async)
- **WMS Adapter**: 8 instances (encapsulate external calls)

### Database Scaling
- **PostgreSQL**: Read replicas for inventory queries, connection pooling (30 connections/instance)
- **Partitioning**: Shipments table partitioned by warehouse_id
- **Indexing**: idx_order_id, idx_status, idx_tracking_number

### Kafka Scaling
- **Partitions**: `order-confirmed`: 10 partitions (one per warehouse cluster)
- **Consumer Groups**: `fulfillment-service` scales to 10 instances

### Circuit Breaker Tuning
- **Failure threshold**: 50% (5 consecutive failures triggers OPEN)
- **Half-open delay**: 30s before retry
- **Success count**: 3 successful calls to close circuit

---

## 12. Monitoring & Observability

### Key Metrics
- **Order Dispatch**: Dispatch rate, dispatch latency (P50, P99), failure rate per WMS
- **Shipment Status**: Time to pickup, time to ship, time to deliver per warehouse
- **Inventory**: Sync lag (time between WMS update and Redis update), accuracy (discrepancy %)
- **Webhook**: Webhook latency, retry rate, idempotency key cache hit rate
- **Circuit Breaker**: State transitions, failure rate, recovery time

### Sample Alert Rules
```yaml
alerts:
  - name: WmsDispatchLatencyHigh
    condition: histogram_quantile(0.99, fulfillment_dispatch_latency_seconds) > 5
    severity: warning

  - name: WmsCircuitBreakerOpen
    condition: wms_circuit_breaker_state{state="OPEN"} > 0
    severity: critical

  - name: InventorySyncLag
    condition: time() - inventory_last_sync_at_seconds > 600
    severity: warning

  - name: BacklogOrders
    condition: fulfillment_dlq_queue_size > 1000
    severity: critical
```

---

## 13. Summary Cheat Sheet

```
ORDER DISPATCH FLOW
1. Order confirmed → Kafka event "order-confirmed"
2. FulfillmentService consumes event
3. FulfillmentSplitter determines best warehouse per item group
4. WMSAdapter calls vendor-specific API (with circuit breaker)
5. Shipment record created in PostgreSQL
6. Event published to Kafka "fulfillment-dispatch"

SPLIT SHIPMENTS
• Greedy bin packing: maximize items per warehouse
• Multiple shipments linked to single order
• Each shipment has own tracking number
• Customer sees all tracking numbers on order page

INVENTORY SYNC
• WMS is source of truth (read-only on e-commerce)
• Sync every 5 minutes via adapter.getInventory()
• Deltas published to Kafka "inventory-sync"
• Redis cache updated immediately (TTL: 10 min)
• PostgreSQL updated asynchronously

WEBHOOK HANDLING
• WMS pushes shipment status updates
• Validate HMAC signature (prevent spoofing)
• Idempotency key dedup (prevent duplicate processing)
• Update shipment status in PostgreSQL
• Publish event to Kafka "fulfillment-updates"

FAILURE HANDLING
• WMS down → Queue to fulfillment-dlq
• Retry exponential backoff: 1m, 5m, 15m, 1h, 4h
• Circuit breaker prevents cascading failures
• Manual recovery: replay DLQ when WMS recovers

MULTI-WMS SUPPORT
• WMSAdapter interface (Manhattan, SAP, Custom, etc.)
• Factory pattern: wmsAdapterFactory.getAdapter(wmsType)
• Anti-corruption layer: vendor models stay private
• Add new WMS without modifying existing code
```

