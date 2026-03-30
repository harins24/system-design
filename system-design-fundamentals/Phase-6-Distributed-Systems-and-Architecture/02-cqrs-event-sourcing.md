# CQRS and Event Sourcing

**Phase 6 — Distributed Systems + Architecture | Topic 2**

---

## The Problem These Patterns Solve

In microservices, each service owns its data. But two hard problems emerge:

```
Problem 1: Same data model for reads and writes is a bad fit

Write model needs:              Read model needs:
  Normalized (no duplication)     Denormalized (pre-joined)
  ACID transactions               Fast queries
  Validation and business rules   Flexible query shapes
  Optimized for correctness       Optimized for speed

Forcing one model to serve both → compromises on both

Problem 2: How do you reconstruct state after bugs?

Traditional: state is current DB value
  Bug corrupts 10,000 user balances
  What were the correct values?
  No history → can't recover

Events as source of truth:
  Have every transaction that ever happened
  Replay to reconstruct correct state
  Perfect audit trail
```

CQRS and Event Sourcing solve these problems separately but combine naturally.

---

## CQRS — Command Query Responsibility Segregation

Separate the model that changes state (Commands) from the model that reads state (Queries).

```
Traditional (one model):
  Service → [Single DB] → reads + writes through same model

CQRS (two models):
  Write path: Service → Command → [Write DB] → normalized, ACID
  Read path:  Service → Query  → [Read DB]  → denormalized, fast

  Write DB → events → Read DB (synced asynchronously)
```

### Commands vs Queries

```
Command: intent to change state
  CreateOrder
  UpdateProfile
  ProcessPayment
  CancelOrder

  Properties:
  → Has side effects (changes data)
  → Should return minimal data (id or void)
  → Can fail (validation, business rules)
  → Goes to write store

Query: request for data, no side effects
  GetOrder
  ListUserOrders
  SearchProducts
  GetUserProfile

  Properties:
  → No side effects (read-only)
  → Returns rich data
  → Always succeeds (or empty result)
  → Goes to read store
```

### CQRS Architecture

```
           [Client]
          /         \
   [Commands]      [Queries]
       ↓                ↓
[Command Handler]  [Query Handler]
       ↓                ↓
[Write Model]      [Read Model]
[PostgreSQL        [Elasticsearch /
 normalized]        Redis /
                    Read Replica /
                    Denormalized DB]
       ↓                ↑
  [Events] ────────────→ [Projections]
  (Kafka)               (event consumers
                         that build
                         read models)
```

### Write Side — Command Handling

```java
// Command — intent to change state
public record CreateOrderCommand(
    String userId,
    List<OrderItem> items,
    String shippingAddressId,
    String idempotencyKey
) {}

// Command Handler — validates + executes + publishes event
@Service
public class CreateOrderCommandHandler {

    private final OrderRepository orderRepo;         // write DB
    private final UserServiceClient userClient;      // validate user
    private final InventoryServiceClient inventory;  // validate stock
    private final KafkaTemplate<String, OrderEvent> kafka;

    @Transactional
    public String handle(CreateOrderCommand command) {

        // Validate
        User user = userClient.getUser(command.userId());
        command.items().forEach(item ->
            inventory.validateStock(item.productId(), item.quantity())
        );

        // Create domain object — write model (normalized)
        Order order = Order.create(command);
        orderRepo.save(order);

        // Publish event for read model sync
        kafka.send("order-events", order.getId(),
            new OrderCreatedEvent(order));

        return order.getId(); // return minimal data
    }
}

// Write model — normalized
@Entity
@Table(name = "orders")
public class Order {
    private String id;
    private String userId;       // just the ID, not the user object
    private OrderStatus status;
    private BigDecimal total;
    private Instant createdAt;

    @OneToMany(cascade = ALL)
    private List<OrderItem> items;
}
```

### Read Side — Query Handling + Projections

```java
// Read model — denormalized, optimized for display
// Stored in Elasticsearch or dedicated read DB
public class OrderReadModel {
    private String orderId;
    private String userId;
    private String userName;        // denormalized from User
    private String userEmail;       // denormalized from User
    private List<OrderItemView> items;  // includes product name, image
    private String statusLabel;     // "In Progress" not "IN_PROGRESS"
    private String formattedTotal;  // "$99.99"
    private String createdAt;       // "Feb 14, 2026"
}

// Projection — listens to events, builds read model
@Component
public class OrderReadModelProjection {

    private final OrderReadModelRepository readRepo;
    private final UserServiceClient userClient;
    private final ProductServiceClient productClient;

    @KafkaListener(topics = "order-events",
                   groupId = "order-read-model-builder")
    public void on(OrderCreatedEvent event) {
        // Enrich with data from other services
        User user = userClient.getUser(event.getUserId());

        List<OrderItemView> items = event.getItems().stream()
            .map(item -> {
                Product product = productClient.getProduct(item.getProductId());
                return new OrderItemView(
                    item.getProductId(),
                    product.getName(),     // denormalized
                    product.getImageUrl(), // denormalized
                    item.getQuantity(),
                    item.getPrice()
                );
            }).collect(toList());

        OrderReadModel readModel = OrderReadModel.builder()
            .orderId(event.getOrderId())
            .userId(event.getUserId())
            .userName(user.getFullName())   // denormalized
            .userEmail(user.getEmail())     // denormalized
            .items(items)
            .statusLabel("Pending")
            .formattedTotal("$" + event.getTotal())
            .createdAt(format(event.getCreatedAt()))
            .build();

        readRepo.save(readModel);
    }

    @KafkaListener(topics = "order-events",
                   groupId = "order-read-model-builder")
    public void on(OrderShippedEvent event) {
        OrderReadModel model = readRepo.findById(event.getOrderId());
        model.setStatusLabel("Shipped");
        model.setTrackingNumber(event.getTrackingNumber());
        readRepo.save(model);
    }
}

// Query Handler — reads from optimized read model
@Service
public class OrderQueryHandler {

    private final OrderReadModelRepository readRepo;

    public OrderReadModel getOrder(String orderId) {
        return readRepo.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
    }

    public Page<OrderReadModel> getUserOrders(
            String userId, Pageable pageable) {
        return readRepo.findByUserIdOrderByCreatedAtDesc(userId, pageable);
    }

    public List<OrderReadModel> searchOrders(OrderSearchCriteria criteria) {
        // Full Elasticsearch query — full text, filters, aggregations
        return elasticsearchRepo.search(criteria);
    }
}
```

---

## Event Sourcing

**Instead of storing current state, store the sequence of events that led to it. Current state is derived by replaying events.**

```
Traditional (state storage):
  Order table:
  id: 123, status: SHIPPED, total: 99.99, updatedAt: ...

  Question: "What was the status 3 days ago?"
  Answer: Gone — only current state stored

Event Sourcing (event storage):
  Events for order 123:
  [OrderCreated,    t=0,  {items, total}]
  [PaymentCharged,  t=1,  {amount: 99.99}]
  [OrderConfirmed,  t=2,  {}]
  [ShipmentCreated, t=3,  {trackingId}]
  [OrderShipped,    t=4,  {carrier: UPS}]

  Current state = replay all events
  State at any time = replay events up to that timestamp
  Complete audit trail always available
```

### Event Store

```java
// Every change is an immutable event appended to a log
@Entity
@Table(name = "order_events")
public class StoredEvent {
    @Id
    private String eventId;
    private String aggregateId;    // order ID
    private String aggregateType;  // "Order"
    private String eventType;      // "OrderCreated"
    private int sequenceNumber;    // ordering within aggregate
    private String payload;        // JSON event data
    private Instant occurredAt;
}

// Appending events (never UPDATE or DELETE)
public class OrderEventStore {

    public void append(String orderId, List<DomainEvent> events) {
        int nextSeq = eventRepo.getMaxSequence(orderId) + 1;

        for (DomainEvent event : events) {
            StoredEvent stored = StoredEvent.builder()
                .eventId(UUID.randomUUID().toString())
                .aggregateId(orderId)
                .aggregateType("Order")
                .eventType(event.getClass().getSimpleName())
                .sequenceNumber(nextSeq++)
                .payload(serialize(event))
                .occurredAt(Instant.now())
                .build();

            eventRepo.save(stored);
        }
    }

    // Reconstruct current state by replaying events
    public Order load(String orderId) {
        List<StoredEvent> events = eventRepo
            .findByAggregateIdOrderBySequenceNumber(orderId);

        Order order = new Order(); // empty
        events.forEach(e -> order.apply(deserialize(e)));
        return order;
    }

    // Time travel — state at specific point in time
    public Order loadAt(String orderId, Instant pointInTime) {
        List<StoredEvent> events = eventRepo
            .findByAggregateIdAndOccurredAtBefore(orderId, pointInTime);

        Order order = new Order();
        events.forEach(e -> order.apply(deserialize(e)));
        return order;
    }
}
```

### The Aggregate

Domain object that handles commands and emits events:

```java
public class Order {

    // Current state (rebuilt from events)
    private String id;
    private String userId;
    private OrderStatus status;
    private List<OrderItem> items;
    private BigDecimal total;

    // Pending events (not yet persisted)
    private List<DomainEvent> uncommittedEvents = new ArrayList<>();

    // Command handler — validates business rules, emits events
    public void create(CreateOrderCommand command) {
        if (this.status != null) {
            throw new IllegalStateException("Order already exists");
        }

        // Don't change state here — emit event
        apply(new OrderCreatedEvent(
            command.getOrderId(),
            command.getUserId(),
            command.getItems(),
            calculateTotal(command.getItems())
        ));
    }

    public void ship(String trackingId) {
        if (this.status != OrderStatus.CONFIRMED) {
            throw new IllegalStateException("Can only ship confirmed orders");
        }
        apply(new OrderShippedEvent(this.id, trackingId));
    }

    // Event handler — changes state (no business logic here)
    public void apply(OrderCreatedEvent event) {
        this.id = event.getOrderId();
        this.userId = event.getUserId();
        this.items = event.getItems();
        this.total = event.getTotal();
        this.status = OrderStatus.PENDING;

        uncommittedEvents.add(event); // track for persistence
    }

    public void apply(OrderShippedEvent event) {
        this.status = OrderStatus.SHIPPED;
        uncommittedEvents.add(event);
    }

    // Replay method (loading from store)
    public void replayEvent(DomainEvent event) {
        apply(event); // apply but don't add to uncommitted
        // We're replaying existing events, not creating new ones
    }
}
```

---

## CQRS + Event Sourcing Together

Natural combination:

```
Write path:
  1. Load aggregate by replaying events from event store
  2. Execute command → aggregate validates + emits new events
  3. Append new events to event store (append-only)
  4. Publish events to Kafka

Read path:
  1. Kafka consumers (projections) receive events
  2. Build/update denormalized read models
  3. Queries hit read models (Elasticsearch, Redis, PostgreSQL)

  Read models are disposable:
  Event store is source of truth
  Delete and rebuild any read model by replaying all events
  Build new read model for new query requirements
  → Replay events → new projection built
```

---

## Snapshots — Optimization for Event Sourcing

```
Problem: Order has 10,000 events
         Loading = replay 10,000 events = slow

Snapshot: periodically save current state
  Every 100 events → save snapshot of current state

  Loading:
  1. Load latest snapshot (state at event 10,000)
  2. Replay only events 10,001 onward
  3. Much faster

Snapshot storage:
  {aggregateId, sequenceNumber, state, createdAt}

Load with snapshot:
  public Order load(String orderId) {
      Snapshot snapshot = snapshotRepo.findLatest(orderId);

      Order order = snapshot != null
          ? deserialize(snapshot.getState())  // start from snapshot
          : new Order();                       // start from empty

      int fromSeq = snapshot != null ? snapshot.getSequenceNumber() + 1 : 0;

      List<StoredEvent> recentEvents = eventRepo
          .findFromSequence(orderId, fromSeq);

      recentEvents.forEach(e -> order.replayEvent(deserialize(e)));
      return order;
  }
```

---

## When to Use CQRS and Event Sourcing

```
Use CQRS when:
  ✅ Read and write workloads are very different
     (complex writes, flexible reads)
  ✅ Read model needs to be rebuilt as product evolves
  ✅ Different scaling requirements for reads vs writes
  ✅ Multiple read representations needed
     (API response, search index, reporting)
  ✅ Team size justifies complexity

Don't use CQRS when:
  ❌ Simple CRUD application (overkill)
  ❌ Reads and writes have similar patterns
  ❌ Small team / early stage
  ❌ Complexity not justified by scale

Use Event Sourcing when:
  ✅ Audit trail is a hard requirement (finance, compliance)
  ✅ Time travel / historical state needed
  ✅ Debugging via event replay is valuable
  ✅ Undo/redo functionality needed
  ✅ Event-driven integration with many services
  ✅ Domain is complex with rich state transitions

Don't use Event Sourcing when:
  ❌ Simple entities that don't change much
  ❌ Events don't have business meaning (just CRUD)
  ❌ Team unfamiliar with pattern (learning curve is real)
  ❌ Reporting requires current state primarily

Real usage:
  Banking:      Event Sourcing (every transaction = event)
  E-commerce:   CQRS for orders (complex write + flexible reads)
  Social media: CQRS for posts (timeline = denormalized read model)
  Healthcare:   Event Sourcing (audit trail required)
```

---

## Practical Trade-offs

```
CQRS complexity costs:
  Two models to maintain
  Eventual consistency between write and read models
  "Why doesn't my order show up immediately after creating?"
  → Read model may lag by milliseconds to seconds
  → Usually acceptable, but must communicate to users

Event Sourcing complexity costs:
  Event schema evolution is hard
     (how to handle events from 3 years ago with old schema?)
  Querying event store is not natural SQL
  Aggregate loading cost without snapshots

Event schema evolution strategies:
  Upcasting: when loading old events, transform to current schema
  Versioned events: OrderCreatedV1, OrderCreatedV2
  Schema registry: Avro + Confluent Schema Registry

The golden rule:
  CQRS and Event Sourcing solve real problems at scale
  They are NOT default patterns for all services
  Apply where the domain complexity and scale justify them
```

---

## Key Takeaways

```
CQRS: Separate write model from read model
  Commands → write side (normalized, ACID, validation)
  Queries  → read side (denormalized, fast, flexible)
  Sync via: events (Kafka) → projections → read models

  Benefits:
  Each side optimized independently
  Read models are disposable (rebuild anytime)
  Different DBs per side (PostgreSQL write, ES read)

Event Sourcing: events are source of truth, not current state
  Every change = immutable event appended to log
  Current state = replay all events
  State at time T = replay events up to T

  Benefits:
  Complete audit trail
  Time travel / debugging
  Multiple read models from same events
  Event-driven integration naturally

  Optimization: snapshots every N events
                Load snapshot + replay delta only

Combined:
  Write: Command → Aggregate → Events → Event Store → Kafka
  Read:  Kafka → Projection → Read Model → Query Handler

Use CQRS when: reads and writes have very different needs
Use Event Sourcing when: audit trail, time travel, undo needed
Don't use for: simple CRUD, small teams, early-stage products

Eventual consistency trade-off:
  Read model may lag milliseconds after write
  Accept and design UI accordingly (optimistic updates)
```
