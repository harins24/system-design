# ACID Transactions

**Phase 4 — Database Fundamentals | Topic 1 of 9**

---

## What is a Transaction?

A transaction is a sequence of database operations treated as a single unit of work — either all operations succeed together, or none of them do.

```
Transfer $100 from Account A to Account B:

Step 1: Deduct $100 from Account A
Step 2: Add $100 to Account B

What if the system crashes between Step 1 and Step 2?
→ Account A lost $100
→ Account B never received it
→ $100 vanished from existence

Transactions prevent this.
Either both steps complete, or neither does.
Money is never lost mid-transfer.
```

ACID is the set of four properties that guarantee transactions behave correctly even in the face of crashes, errors, and concurrent access.

---

## A — Atomicity

**All operations in a transaction succeed, or none of them do. There is no partial completion.**

```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
COMMIT;

Scenario 1: Both succeed → COMMIT → changes permanent
Scenario 2: Second UPDATE fails → ROLLBACK → first UPDATE undone
Scenario 3: Server crashes mid-transaction → on restart,
            incomplete transaction rolled back automatically

The database is never left in a halfway state.
```

**How databases implement atomicity:**

Write-Ahead Log (WAL) — before modifying any data, the database writes what it intends to do to a log on disk. On crash recovery, it reads the log:

```
WAL entry:
[TXN_123][BEGIN]
[TXN_123][UPDATE accounts A balance: 500→400]
[TXN_123][UPDATE accounts B balance: 200→300]
[TXN_123][COMMIT]   ← if this line is here, replay
                      if missing, rollback
```

---

## C — Consistency

**A transaction brings the database from one valid state to another valid state. All defined rules, constraints, and invariants are preserved.**

```
Database rules defined:
  - account balance cannot be negative
  - user email must be unique
  - order total must equal sum of item prices
  - foreign key: order.userId must exist in users table

Consistency means:
  If these rules hold BEFORE the transaction,
  they must hold AFTER the transaction.
  A transaction that would violate rules is rejected.
```

```sql
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 1000 WHERE id = 'A';
  -- Account A only has $500, balance would become -$500
  -- Constraint: balance >= 0
  -- Database REJECTS this transaction
ROLLBACK; ← automatic
```

> ACID Consistency = data integrity rules are never violated
> CAP Consistency = all nodes see the same data at the same time
> (These are completely different concepts)

---

## I — Isolation

**Concurrent transactions execute as if they ran sequentially. One transaction's in-progress changes are not visible to other transactions until committed.**

### The Problems Isolation Prevents

**Dirty Read:**
```
Transaction A: UPDATE products SET price = 50 WHERE id = 1;
               (not committed yet)
Transaction B: SELECT price FROM products WHERE id = 1;
               → sees 50 (uncommitted!)
Transaction A: ROLLBACK; (price should still be 100)
Transaction B: made decision based on data that never existed
```

**Non-Repeatable Read:**
```
Transaction A: SELECT balance FROM accounts WHERE id = 1;
               → sees 500
Transaction B: UPDATE accounts SET balance = 300 WHERE id = 1;
               COMMIT;
Transaction A: SELECT balance FROM accounts WHERE id = 1;
               → sees 300 (different from first read!)
Transaction A: confused — same query, different result
```

**Phantom Read:**
```
Transaction A: SELECT COUNT(*) FROM orders WHERE status = 'PENDING';
               → sees 5
Transaction B: INSERT INTO orders (status) VALUES ('PENDING');
               COMMIT;
Transaction A: SELECT COUNT(*) FROM orders WHERE status = 'PENDING';
               → sees 6 (phantom row appeared!)
```

### Isolation Levels

| Level | Dirty Read | Non-Repeatable | Phantom |
|-------|-----------|----------------|---------|
| READ UNCOMMITTED | Possible | Possible | Possible |
| READ COMMITTED | Prevented | Possible | Possible |
| REPEATABLE READ | Prevented | Prevented | Possible |
| SERIALIZABLE | Prevented | Prevented | Prevented |

Higher isolation = fewer anomalies = lower concurrency = lower performance.

```
READ UNCOMMITTED:
  Can read uncommitted changes. Fastest, most dangerous. Rarely used.

READ COMMITTED (PostgreSQL default):
  Only reads committed data. Prevents dirty reads.

REPEATABLE READ (MySQL InnoDB default):
  Within a transaction, repeated reads return same data.
  Snapshot taken at start of transaction.

SERIALIZABLE:
  Strongest isolation. Transactions run as if fully sequential.
  Prevents all anomalies. Highest overhead.
  Use for: financial calculations, inventory management.
```

### MVCC — How Modern Databases Implement Isolation

Multi-Version Concurrency Control — instead of locking rows, databases keep multiple versions:

```
Row: account_id=1, balance=500, txn_id=100

Transaction 101 starts:
  Reads account_id=1 → sees version from txn_id=100 (balance=500)

Transaction 102 updates:
  balance = 300, writes new version with txn_id=102

Transaction 101 reads again:
  Still sees txn_id=100 version (balance=500) ← snapshot isolation
  Transaction 102's write doesn't affect Transaction 101

No read locks needed → reads never block writes
No write locks on reads → writes never block reads
→ Higher concurrency than traditional locking
```

PostgreSQL, MySQL InnoDB, Oracle all use MVCC.

---

## D — Durability

**Once a transaction is committed, it remains committed even if the system crashes immediately after.**

```
t=0ms: Transaction commits (COMMIT returns success)
t=1ms: Server power cut, complete memory loss

On restart:
  Database reads WAL
  Finds committed transaction
  Re-applies changes to data files
  Data is fully restored

The commit guarantee is permanent.
"Committed" means "on disk forever" not "in memory for now."
```

**How durability is implemented:**
```
Synchronous disk write:
  COMMIT command blocks until WAL flushed to disk
  Only then returns success to application
  Cost: disk I/O on every commit (slow but safe)

fsync():
  OS-level call ensuring data physically written to disk
  Not just OS buffer — physically on the storage medium
  PostgreSQL calls fsync() on WAL writes

Battery-backed write cache:
  Enterprise storage has RAM cache + battery
  Writes acknowledged when in battery-backed cache
  Even on power loss, cache survives long enough to flush
```

**Durability vs Performance tradeoff:**
```
synchronous_commit = on  (PostgreSQL default)
  → Every commit waits for disk flush
  → Safe, but ~5ms per transaction

synchronous_commit = off
  → Commits return before disk flush
  → Risk: up to ~200ms of transactions lost on crash
  → But 5-10x higher throughput
  → Acceptable for: logs, metrics, non-critical data
  → Never for: financial transactions, user data
```

---

## ACID in Distributed Systems — The Hard Part

ACID is straightforward in a single database. It gets extremely hard across multiple databases or microservices.

```
Order Service (PostgreSQL):
  INSERT INTO orders ...

Payment Service (separate PostgreSQL):
  INSERT INTO payments ...

Inventory Service (separate PostgreSQL):
  UPDATE stock SET quantity = quantity - 1 ...

How do you make these three operations atomic?
If payment succeeds but inventory update fails:
  → Order created ✅
  → Payment charged ✅
  → Stock not decremented ❌
  → Oversold inventory
```

### Two-Phase Commit (2PC)

```
Phase 1 — Prepare:
  Coordinator: "Can you commit?"
  Order DB:    "Yes, prepared" (locks resources)
  Payment DB:  "Yes, prepared" (locks resources)
  Inventory DB:"Yes, prepared" (locks resources)

Phase 2 — Commit:
  Coordinator: "All prepared, now commit"
  All DBs:     Commit and release locks

Problem:
  Coordinator crashes between Phase 1 and 2
  → All DBs locked forever waiting for Phase 2
  → "In-doubt transaction" — system stuck

2PC is rarely used in microservices because of this blocking problem
```

### Saga Pattern (Preferred for Microservices)

```
Instead of atomic transaction, use compensating transactions:

Step 1: Create order         → success
Step 2: Charge payment       → success
Step 3: Deduct inventory     → FAILS

Compensating actions (run in reverse):
Step 2 undo: Refund payment  ← compensation
Step 1 undo: Cancel order    ← compensation

No distributed lock needed
Each service maintains its own ACID transactions locally
Compensating transactions handle failures
```

Two saga implementations:
```
Choreography: Services emit events, others react
  Order created → Payment service listens → charges card
  Payment charged → Inventory service listens → deducts stock
  Inventory failed → emits event → Payment service refunds

Orchestration: Central orchestrator directs each step
  Saga Orchestrator: "Payment service, charge card"
  Saga Orchestrator: "Inventory service, deduct stock"
  On failure: "Payment service, refund charge"
```

---

## ACID in Practice — Spring Boot

```java
@Service
@Transactional  // Applies to all public methods by default
public class OrderService {

    @Transactional(isolation = Isolation.REPEATABLE_READ)
    public Order createOrder(CreateOrderRequest request) {
        // All DB operations in this method are in one transaction

        // Check inventory (read)
        Product product = productRepo.findById(request.getProductId())
            .orElseThrow(() -> new ProductNotFoundException());

        if (product.getStock() < request.getQuantity()) {
            throw new InsufficientStockException();
        }

        // Create order (write)
        Order order = orderRepo.save(new Order(request));

        // Deduct inventory (write)
        product.setStock(product.getStock() - request.getQuantity());
        productRepo.save(product);

        // All or nothing — if productRepo.save fails,
        // orderRepo.save is also rolled back
        return order;
    }

    @Transactional(
        propagation = Propagation.REQUIRES_NEW,  // new transaction
        isolation = Isolation.SERIALIZABLE,       // strongest isolation
        timeout = 5,                              // rollback after 5s
        rollbackFor = Exception.class             // rollback on any exception
    )
    public Payment processPayment(PaymentRequest request) {
        // Runs in a NEW transaction, independent of caller's
        ...
    }

    @Transactional(readOnly = true)  // optimization hint
    public List<Order> getOrders(String userId) {
        // readOnly: skip dirty check, use read replica, faster
        return orderRepo.findByUserId(userId);
    }
}
```

**Transaction propagation levels:**
```
REQUIRED (default): Join existing or create new
REQUIRES_NEW:       Always create new, suspend existing
MANDATORY:          Must join existing, else throw
SUPPORTS:           Join if exists, else no transaction
NOT_SUPPORTED:      Always run without transaction
NEVER:              Must run without, else throw
NESTED:             Savepoint within existing transaction
```

---

## Key Takeaways

```
ACID: Guarantees that make database transactions reliable

Atomicity:   All or nothing — no partial transactions
             WAL (Write-Ahead Log) enables rollback on crash

Consistency: Rules never violated — constraints enforced
             DB enforces schema rules, app enforces business rules
             ACID-C ≠ CAP-C (completely different concepts)

Isolation:   Concurrent transactions don't interfere
             Levels: Read Uncommitted < Read Committed <
                     Repeatable Read < Serializable
             MVCC: multiple versions → reads don't block writes

Durability:  Committed data survives crashes
             WAL + fsync() + disk flush = permanent writes
             synchronous_commit=off trades safety for performance

Distributed ACID:
  2PC:   atomic but blocking, coordinator SPOF
  Saga:  eventual consistency, compensating transactions
  Preferred: Saga pattern for microservices

Spring Boot:
  @Transactional: manages transaction lifecycle
  isolation:      controls isolation level
  propagation:    controls transaction joining behavior
  readOnly:       optimization for read-only operations
```
