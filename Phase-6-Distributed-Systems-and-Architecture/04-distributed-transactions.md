# Distributed Transactions

**Phase 6 — Distributed Systems + Architecture | Topic 4**

---

## The Core Problem

In a monolith with one database, ACID transactions are straightforward:

```java
@Transactional
public void transferMoney(String fromId, String toId, BigDecimal amount) {
    Account from = accountRepo.findById(fromId);
    Account to = accountRepo.findById(toId);
    from.debit(amount);
    to.credit(amount);
    accountRepo.save(from);
    accountRepo.save(to);
    // Either both happen or neither — atomic
}
```

In microservices, this breaks completely:

```
Transfer $100 from Account A (Banking Service)
          to Account B (Payments Service)

Step 1: Banking Service:  debit $100 from Account A  ✅
Step 2: Payments Service: credit $100 to Account B   ❌ service crashes

Result: $100 debited, never credited → money lost

No single database → no ACID transaction spans both services
This is the fundamental distributed systems challenge
```

---

## Why This is Hard

```
The CAP theorem forces a choice:
  During network partition:
    CP: stop accepting writes until consistent → unavailable
    AP: accept writes, may be inconsistent → inconsistency

  "Consistent distributed transactions" requires coordination
  Coordination requires round trips
  Round trips cost latency
  Failures during coordination = inconsistency

The fundamental tension:
  Strong consistency + High availability = impossible during partitions
  All distributed transaction solutions make this tradeoff somewhere
```

---

## Solution 1: Two-Phase Commit (2PC)

A coordinator orchestrates all participants to either all-commit or all-rollback.

```
Phase 1 — Prepare:
  Coordinator → Banking Service:   "Can you debit $100?"
  Coordinator → Payments Service:  "Can you credit $100?"

  Banking Service:  acquires lock, writes to WAL → "Yes, prepared"
  Payments Service: acquires lock, writes to WAL → "Yes, prepared"

Phase 2 — Commit:
  All said yes → Coordinator: "Commit!"
  Banking Service:  commits debit, releases lock
  Payments Service: commits credit, releases lock

  Any said no → Coordinator: "Rollback!"
  All participants: undo their changes, release locks
```

### 2PC Failure Scenarios

```
Scenario 1: Participant fails during Phase 1
  Banking Service crashes before responding
  Coordinator waits → timeout → sends Rollback to all
  Safe: no participant committed anything

Scenario 2: Coordinator fails between Phase 1 and Phase 2
  Both participants said "Yes" and are LOCKED waiting
  Coordinator dead → no one sends Commit or Rollback
  Participants cannot proceed — BLOCKED INDEFINITELY

  This is called a "blocking protocol"
  In-doubt transactions: locks held, resources unavailable
  Manual intervention required to resolve

  This is why 2PC is rarely used in microservices

Scenario 3: Participant fails after Phase 2 starts
  Coordinator sends Commit
  Banking Service commits
  Payments Service crashes before receiving Commit

  Payments Service recovers → checks its WAL
  Sees it was "prepared" → asks coordinator for decision
  Coordinator says "committed" → Payments Service commits
  Recovery from WAL allows this to be resolved
```

### 2PC Implementation

```java
// XA transactions — Java's standard 2PC protocol
@Configuration
public class XAConfig {

    @Bean
    public AtomikosDataSourceBean bankingDataSource() {
        AtomikosDataSourceBean ds = new AtomikosDataSourceBean();
        ds.setXaDataSourceClassName("org.postgresql.xa.PGXADataSource");
        ds.setUniqueResourceName("banking-db");
        // ... config
        return ds;
    }

    @Bean
    public AtomikosDataSourceBean paymentsDataSource() {
        AtomikosDataSourceBean ds = new AtomikosDataSourceBean();
        ds.setXaDataSourceClassName("org.postgresql.xa.PGXADataSource");
        ds.setUniqueResourceName("payments-db");
        return ds;
    }

    @Bean
    public JtaTransactionManager transactionManager() {
        return new JtaTransactionManager();
    }
}

@Service
public class TransferService {

    @Transactional  // JTA manages 2PC across both DBs
    public void transfer(String fromId, String toId, BigDecimal amount) {
        bankingService.debit(fromId, amount);   // banking DB
        paymentsService.credit(toId, amount);   // payments DB
        // JTA coordinator handles 2PC protocol
    }
}
```

**When to use 2PC:**

```
✅ Same organization controls all participants
✅ Low transaction volume (locking overhead acceptable)
✅ Participants are databases (XA support)
✅ Consistency is absolutely required
✅ Internal microservices within same data center

❌ Cross-organization transactions
❌ High throughput systems
❌ Long-running transactions (locks held too long)
❌ Participants without XA support
❌ Microservices over unreliable networks
```

---

## Solution 2: Saga Pattern

Break the distributed transaction into a sequence of local transactions. Each step publishes an event. If a step fails, compensating transactions undo previous steps.

```
Transfer $100 — Saga:

Step 1: Banking Service
  Local transaction: debit $100 from Account A
  Publish: MoneyDebited event

Step 2: Payments Service (reacts to MoneyDebited)
  Local transaction: credit $100 to Account B
  Publish: MoneyTransferred event

FAILURE SCENARIO:
Step 1: Banking Service
  Local transaction: debit $100 from Account A ✅
  Publish: MoneyDebited event

Step 2: Payments Service
  Credit fails (account doesn't exist) ❌
  Publish: CreditFailed event

COMPENSATION:
Step 1 Compensation: Banking Service (reacts to CreditFailed)
  Local transaction: REFUND $100 to Account A
  Publish: MoneyRefunded event

Saga guarantees: eventually consistent, not instantly consistent
Brief window where $100 is debited but not credited
System converges to correct state
```

### Saga Implementation — Choreography

Services react to events directly. No central coordinator.

```java
// Step 1: Banking Service publishes after debit
@Service
public class BankingService {

    @Transactional
    public void debit(String accountId, BigDecimal amount, String sagaId) {
        Account account = accountRepo.findById(accountId);
        account.debit(amount);
        accountRepo.save(account);

        // Publish event in same transaction (Outbox pattern)
        outboxRepo.save(new OutboxEvent("MoneyDebited", new MoneyDebitedEvent(
            sagaId, accountId, amount)));
    }
}

// Step 2: Payments Service reacts
@KafkaListener(topics = "banking-events", groupId = "payments-service")
public void onMoneyDebited(MoneyDebitedEvent event) {
    try {
        paymentsService.credit(event.getTargetAccountId(), event.getAmount());
        kafka.send("payment-events", new MoneyCreditedEvent(event.getSagaId()));
    } catch (Exception e) {
        kafka.send("payment-events", new CreditFailedEvent(event.getSagaId(),
            e.getMessage()));
    }
}

// Compensation: Banking Service reacts to failure
@KafkaListener(topics = "payment-events", groupId = "banking-service")
public void onCreditFailed(CreditFailedEvent event) {
    // Compensating transaction — refund the debit
    bankingService.refund(event.getSagaId()); // undo step 1
    kafka.send("banking-events", new MoneyRefundedEvent(event.getSagaId()));
}
```

**Choreography pros/cons:**

```
✅ Simple — services just react to events
✅ No central coordinator (no SPOF)
✅ Loose coupling between services

❌ Hard to track overall saga state
   "Is this saga complete? Which step are we on?"
❌ Difficult to debug complex sagas
❌ Risk of cyclic event chains
❌ Hard to add new steps without touching many services
```

### Saga Implementation — Orchestration

Central saga orchestrator tells each service what to do next.

```java
// Saga Orchestrator — manages the workflow
@Service
public class TransferSagaOrchestrator {

    private final SagaStateRepository sagaRepo;
    private final BankingServiceClient bankingClient;
    private final PaymentsServiceClient paymentsClient;

    public String startTransfer(TransferCommand command) {
        // Create saga with initial state
        SagaState saga = new SagaState(
            UUID.randomUUID().toString(),
            SagaStep.DEBIT_ACCOUNT,
            command
        );
        sagaRepo.save(saga);

        // Step 1: Tell Banking Service to debit
        bankingClient.debit(new DebitRequest(
            saga.getId(),
            command.getFromAccountId(),
            command.getAmount()
        ));

        return saga.getId();
    }

    // Handle response from Banking Service
    @KafkaListener(topics = "banking-events")
    public void onBankingEvent(BankingEvent event) {
        SagaState saga = sagaRepo.findBySagaId(event.getSagaId());

        switch (event.getType()) {
            case DEBIT_SUCCEEDED:
                saga.setStep(SagaStep.CREDIT_ACCOUNT);
                sagaRepo.save(saga);
                // Step 2: Tell Payments Service to credit
                paymentsClient.credit(new CreditRequest(
                    saga.getId(),
                    saga.getCommand().getToAccountId(),
                    saga.getCommand().getAmount()
                ));
                break;

            case DEBIT_FAILED:
                saga.setStep(SagaStep.FAILED);
                sagaRepo.save(saga);
                // No compensation needed (nothing happened yet)
                break;
        }
    }

    // Handle response from Payments Service
    @KafkaListener(topics = "payment-events")
    public void onPaymentEvent(PaymentEvent event) {
        SagaState saga = sagaRepo.findBySagaId(event.getSagaId());

        switch (event.getType()) {
            case CREDIT_SUCCEEDED:
                saga.setStep(SagaStep.COMPLETED);
                sagaRepo.save(saga);
                // Done!
                break;

            case CREDIT_FAILED:
                saga.setStep(SagaStep.COMPENSATING);
                sagaRepo.save(saga);
                // Compensation: tell Banking to refund
                bankingClient.refund(new RefundRequest(
                    saga.getId(),
                    saga.getCommand().getFromAccountId(),
                    saga.getCommand().getAmount()
                ));
                break;
        }
    }
}
```

**Orchestration pros/cons:**

```
✅ Clear visibility of saga state and progress
✅ Easy to debug ("saga abc123 failed at step 3")
✅ Easy to add/modify steps
✅ Natural retry and timeout handling
✅ Single place to understand business process

❌ Orchestrator can become a central dependency (manage HA carefully)
❌ Services know about orchestrator (some coupling)
❌ Orchestrator must be highly available
```

---

## Solution 3: TCC — Try-Confirm-Cancel

Three-phase protocol that reserves resources before committing:

```
Phase 1 — Try (reserve resources):
  Banking Service:    Reserve $100 from Account A (don't debit yet)
  Payments Service:   Reserve credit slot for Account B

  If any Try fails → Cancel all, release reservations

Phase 2 — Confirm (execute reservation):
  All Tries succeeded:
  Banking Service:    Actually debit $100
  Payments Service:   Actually credit $100

  OR

Phase 2 — Cancel (release reservation):
  Any Try failed:
  Banking Service:    Release $100 reservation
  Payments Service:   Release credit slot reservation

Compare to 2PC:
  2PC: one round trip (prepare + commit)
       blocking if coordinator fails
  TCC: two round trips (try + confirm)
       no blocking (participant can self-cancel after timeout)
```

```java
// TCC interfaces
public interface BankingTCCService {
    ReservationId tryDebit(String accountId, BigDecimal amount);
    void confirmDebit(ReservationId reservation);
    void cancelDebit(ReservationId reservation);
}

@Service
public class BankingTCCServiceImpl implements BankingTCCService {

    public ReservationId tryDebit(String accountId, BigDecimal amount) {
        Account account = accountRepo.findById(accountId);

        if (account.getAvailableBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException();
        }

        // Reserve (hold) funds without debiting
        Reservation reservation = account.holdFunds(amount);
        accountRepo.save(account);
        return reservation.getId();
    }

    public void confirmDebit(ReservationId reservationId) {
        // Convert hold to actual debit
        Reservation reservation = reservationRepo.findById(reservationId);
        Account account = accountRepo.findById(reservation.getAccountId());
        account.confirmDebit(reservation);
        accountRepo.save(account);
    }

    public void cancelDebit(ReservationId reservationId) {
        // Release hold — funds available again
        Reservation reservation = reservationRepo.findById(reservationId);
        Account account = accountRepo.findById(reservation.getAccountId());
        account.releaseHold(reservation);
        accountRepo.save(account);
    }
}
```

**TCC use cases:**

```
✅ Inventory reservation (hold stock before confirming order)
✅ Seat reservation (hold airline seat before payment)
✅ Financial transfers (reserve funds before committing)
✅ When resources need temporary reservation

❌ Complex business logic for try/confirm/cancel
❌ All participants must implement TCC interface
❌ More application code than Saga
```

---

## Solution 4: Eventual Consistency with Outbox

Accept that consistency is eventual. Guarantee it via reliable event delivery.

```
The Outbox Pattern guarantees events are published:

Order Service creates order AND publishes event atomically:
  BEGIN TRANSACTION;
    INSERT INTO orders VALUES (...);
    INSERT INTO outbox (event, status) VALUES ('OrderCreated', 'PENDING');
  COMMIT;

  Either both inserted or neither — ACID within one DB

Outbox Relay (separate process):
  Polls outbox table for PENDING events
  Publishes to Kafka
  Marks as PUBLISHED

  Even if Kafka is down at order creation time:
  → Event sits in outbox table
  → Relay retries until Kafka available
  → Event eventually published
  → Downstream services eventually consistent
```

**Transactional Outbox with Debezium (CDC):**

```
Debezium streams PostgreSQL WAL → Kafka automatically

No polling needed:
  Row inserted into outbox → WAL entry created
  Debezium reads WAL → publishes to Kafka

More efficient than polling
Sub-second latency from write to event publication
No additional load on DB (reads WAL, not table)
```

### Idempotency — Critical for All Solutions

All distributed transaction solutions involve retries. Retries mean operations may execute multiple times. Every participant MUST be idempotent.

```java
@Service
public class IdempotentPaymentService {

    @Transactional
    public PaymentResult processPayment(PaymentCommand command) {
        String idempotencyKey = command.getSagaId() + "-payment";

        // Check if already processed
        Optional<PaymentResult> existing =
            paymentResultRepo.findByIdempotencyKey(idempotencyKey);

        if (existing.isPresent()) {
            log.info("Duplicate payment request, returning cached result: {}",
                idempotencyKey);
            return existing.get(); // Return same result
        }

        // Process payment
        PaymentResult result = chargeCard(command.getAmount(),
                                          command.getCardToken());
        result.setIdempotencyKey(idempotencyKey);
        paymentResultRepo.save(result);

        return result;
    }
}

// Database constraint as safety net
CREATE UNIQUE INDEX idx_payment_idempotency
ON payment_results(idempotency_key);
// Concurrent duplicates → one succeeds, one gets constraint violation
```

---

## Choosing the Right Solution

```
Choose 2PC when:
  → Both services are databases with XA support
  → Same infrastructure/organization controls all participants
  → Transaction volume is low
  → Strict ACID consistency required
  → Short transaction duration (no long locks)

Choose Saga (Choreography) when:
  → Long-running business processes
  → Loose coupling between services preferred
  → Simple linear workflow
  → Teams prefer event-driven independence

Choose Saga (Orchestration) when:
  → Complex multi-step workflow
  → Need clear visibility into transaction state
  → Easy debugging and monitoring required
  → Workflow may change frequently
  → Temporal.io or Axon Framework available

Choose TCC when:
  → Resource reservation required
  → Time-bounded holds (seats, inventory, funds)
  → Can implement try/confirm/cancel in all participants

Choose Eventual Consistency + Outbox when:
  → Strict consistency not required
  → Simpler implementation preferred
  → Events are the natural integration mechanism
  → Acceptable to have brief inconsistency window
```

---

## Real World: How Payments Work

```
Stripe's approach (simplified):

1. Create PaymentIntent (reservations)
   → Authorizes card (TCC try phase)
   → Returns client_secret

2. Confirm PaymentIntent (commit)
   → Charges card (TCC confirm phase)
   → Updates all downstream systems via events

3. If failure at step 2:
   → Cancel authorization (TCC cancel phase)
   → Refund if already captured
   → All done via idempotent, retryable operations

Key insight: no single atomic transaction
            carefully ordered operations with compensation
            idempotency keys prevent double charges
            webhooks for eventual consistency with merchants
```

---

## Key Takeaways

```
Distributed transactions: maintaining consistency
across multiple services/databases

No single database → no ACID across services
Must choose: strong consistency (slower) vs availability (faster)

Four approaches:

2PC (Two-Phase Commit):
  Coordinator prepares all → commit or rollback
  Blocking: coordinator failure → participants stuck
  Use: low volume, same-org, XA-capable DBs
  Avoid: high traffic, cross-org, unreliable networks

Saga:
  Local transactions + compensating transactions
  Choreography: event-driven, decentralized
  Orchestration: central coordinator, clear state
  Use: most distributed transaction needs in microservices
  Accept: eventual consistency, brief inconsistency window

TCC (Try-Confirm-Cancel):
  Reserve resources before committing
  Non-blocking: participants self-cancel on timeout
  Use: inventory/seat/fund reservations

Eventual Consistency + Outbox:
  Accept inconsistency, guarantee it converges
  Outbox pattern: atomic DB write + reliable event publish
  Use: when eventual consistency is acceptable

Critical requirement for all: IDEMPOTENCY
  Retries are inevitable
  Operations must be safe to execute multiple times
  Idempotency key + unique constraint = safe retries

Interview answer:
  "Microservices can't have ACID transactions across services.
   I'd use the Saga pattern — each service executes a local
   transaction and publishes events. If a step fails, compensating
   transactions roll back previous steps. Each step is idempotent
   to handle retries safely. For resource reservation specifically,
   I'd consider TCC."
```
