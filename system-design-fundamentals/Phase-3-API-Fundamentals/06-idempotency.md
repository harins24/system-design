# Idempotency

**Phase 3 — API Fundamentals | Topic 6 of 8**

---

## What is Idempotency?

An operation is idempotent if performing it multiple times produces the same result as performing it once.

```
Idempotent:
  Press elevator button 5 times → elevator still comes once
  Set alarm to 7am 3 times → alarm still rings once at 7am
  DELETE /users/123 three times → user deleted, same end state

Not idempotent:
  Click "Submit Order" 3 times → 3 orders created
  Send message 3 times → 3 duplicate messages
  Increment counter 3 times → counter increased by 3
```

In distributed systems, retries are inevitable. Networks fail, timeouts occur, services crash mid-request. If your operations aren't idempotent, retries cause disasters.

---

## Why Idempotency is Critical in Distributed Systems

```
Timeline without idempotency:
t=0ms:   Client sends POST /payments (charge $100)
t=100ms: Payment service processes payment → charges card ✅
t=150ms: Payment service tries to respond → network dies ❌
t=200ms: Client times out, sees no response
t=201ms: Client thinks "request failed" → retries
t=201ms: Client sends POST /payments again
t=300ms: Payment service processes AGAIN → charges card AGAIN ❌

Customer charged $200 instead of $100.
This is a real production disaster. Stripe sees this constantly.

Timeline with idempotency:
t=0ms:   Client sends POST /payments with Idempotency-Key: key_abc
t=100ms: Payment service processes payment → charges $100 ✅
t=150ms: Network dies, client doesn't get response ❌
t=201ms: Client retries POST /payments with SAME key: key_abc
t=300ms: Payment service checks: "Have I seen key_abc?"
         Yes → returns cached result, skips processing
         Customer charged $100 exactly once ✅
```

---

## HTTP Methods and Idempotency

| Method | Idempotent | Safe | Meaning |
|--------|-----------|------|---------|
| GET | ✅ | ✅ | Read only, no side effects |
| HEAD | ✅ | ✅ | Like GET, headers only |
| OPTIONS | ✅ | ✅ | Describes allowed methods |
| PUT | ✅ | ❌ | Replace entire resource |
| DELETE | ✅ | ❌ | Remove resource |
| POST | ❌ | ❌ | Create/action, side effects |
| PATCH | ❌* | ❌ | Partial update |

```
* PATCH can be idempotent if designed carefully
  PATCH /users/123 {"status": "active"} → idempotent
  PATCH /users/123 {"loginCount": "+1"} → not idempotent

Safe = no side effects (read-only)
Idempotent = same result if called multiple times

GET, PUT, DELETE are naturally idempotent:
GET /users/123 × 10 → returns same user each time ✅

PUT /users/123 {"name": "Hari"} × 10
→ user name is "Hari" after any number of calls ✅

DELETE /users/123 × 10
→ user deleted after first call
→ subsequent calls return 404 but state is same (deleted) ✅

POST is the dangerous one — needs explicit idempotency.
```

---

## Implementing Idempotency with Idempotency Keys

The universal pattern used by Stripe, Braintree, Adyen, and most payment systems:

```
Client generates a unique key per logical operation:
  UUID v4: "a8f3c2d1-4b5e-6f7a-8b9c-0d1e2f3a4b5c"

Client includes it in every request:
  POST /payments
  Idempotency-Key: a8f3c2d1-4b5e-6f7a-8b9c-0d1e2f3a4b5c
  {"amount": 100, "currency": "usd"}

Server stores result keyed by this value.
Any retry with same key → returns stored result.
```

### Server Implementation

```java
@RestController
public class PaymentController {

    @PostMapping("/payments")
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody PaymentRequest request) {

        // Step 1: Check if we've seen this key before
        Optional<IdempotencyRecord> existing =
            idempotencyStore.find(idempotencyKey);

        if (existing.isPresent()) {
            // Return cached response — don't process again
            return ResponseEntity
                .status(existing.get().getStatusCode())
                .body(existing.get().getResponse());
        }

        // Step 2: Lock this key (prevent concurrent duplicates)
        // Use Redis SETNX or DB unique constraint
        boolean locked = idempotencyStore.lock(idempotencyKey);
        if (!locked) {
            // Another request with same key is in-flight
            return ResponseEntity.status(409)
                .body(new ErrorResponse("Request in progress, retry shortly"));
        }

        try {
            // Step 3: Process the actual request
            PaymentResponse response = paymentService.charge(request);

            // Step 4: Store result with key
            idempotencyStore.save(IdempotencyRecord.builder()
                .key(idempotencyKey)
                .statusCode(200)
                .response(response)
                .expiresAt(Instant.now().plus(24, HOURS))
                .build());

            return ResponseEntity.ok(response);

        } catch (Exception e) {
            // Step 5: On failure, release lock (allow retry)
            // DON'T store failure — client should be able to retry
            idempotencyStore.releaseLock(idempotencyKey);
            throw e;
        }
    }
}
```

---

## Idempotency Storage — Redis vs Database

```
Redis (recommended for idempotency keys):
  SET idempotency:key_abc JSON_RESULT EX 86400
  → Automatic TTL (keys expire after 24 hours)
  → O(1) lookup
  → Low latency (~1ms)
  → Works across all service instances

  String key = "idempotency:" + idempotencyKey;
  String existing = redisTemplate.opsForValue().get(key);
  if (existing != null) return deserialize(existing);

  // Use SETNX for atomic lock
  Boolean isNew = redisTemplate.opsForValue()
      .setIfAbsent(key, "PROCESSING", Duration.ofHours(24));

Database (when audit trail needed):
  CREATE TABLE idempotency_keys (
      key          VARCHAR(255) PRIMARY KEY,
      request_hash VARCHAR(64),   -- hash of request body
      response     JSONB,
      status_code  INT,
      created_at   TIMESTAMP,
      expires_at   TIMESTAMP
  );

  Pros: Durable, auditable, complex queries
  Cons: Slower, DB load, need cleanup job for expired keys
```

---

## Request Fingerprinting — Extra Safety

What if the client sends the same idempotency key with different request bodies? (Bug or attack.)

```
Request 1: POST /payments  key=abc  {"amount": 100}  ← first time
Request 2: POST /payments  key=abc  {"amount": 999}  ← same key, different body!

Option 1: Accept and return cached result from Request 1
          (ignore body mismatch)

Option 2: Reject with 422 Unprocessable Entity
          "Idempotency key already used with different request body"
          This is what Stripe does — safest approach
```

```java
// Store hash of original request body alongside result
String requestHash = sha256(objectMapper.writeValueAsString(request));

IdempotencyRecord record = idempotencyStore.find(key);
if (record != null) {
    if (!record.getRequestHash().equals(requestHash)) {
        throw new IdempotencyConflictException(
            "Idempotency key reused with different request body");
    }
    return record.getResponse(); // Same request, return cached
}
```

---

## Idempotency in Kafka

Kafka has three delivery semantics:

### At-Most-Once
```
Producer sends → doesn't wait for ACK
Message may be lost if broker crashes
→ Never delivered twice, but may not be delivered at all
Use: Metrics, logs where occasional loss is acceptable
```

### At-Least-Once (default)
```
Producer waits for ACK
If no ACK (timeout/failure), producer retries
Message guaranteed to arrive but may arrive MULTIPLE TIMES
→ Consumer must handle duplicates idempotently

Your consumer must be idempotent:
@KafkaListener(topics = "payments")
public void processPayment(PaymentEvent event) {
    // Check if already processed
    if (processedEvents.contains(event.getId())) return;

    // Process
    paymentService.process(event);

    // Mark processed
    processedEvents.add(event.getId());
}
```

### Exactly-Once (idempotent producer + transactions)
```java
// Enable idempotent producer (Kafka handles dedup automatically)
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE);

// Kafka assigns producer ID + sequence number to each message
// Broker detects duplicates by sequence number → discards
// Producer can retry safely → exactly-once delivery to broker

// Transactions: read-process-write atomically
producer.initTransactions();
try {
    producer.beginTransaction();

    ConsumerRecords<String, Event> records = consumer.poll(Duration.ofMillis(100));

    for (ConsumerRecord<String, Event> record : records) {
        ProcessedEvent result = process(record.value());
        producer.send(new ProducerRecord<>("output-topic", result));
    }

    // Commit consumer offsets as part of transaction
    producer.sendOffsetsToTransaction(offsets, consumer.groupMetadata());

    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction(); // Rollback everything
}
```

---

## Idempotency Patterns Beyond APIs

### Database Upsert
```sql
-- Instead of INSERT (fails on duplicate) or
-- separate SELECT then INSERT (race condition):

INSERT INTO orders (id, user_id, amount, status)
VALUES ('order_123', 'user_456', 99.99, 'PENDING')
ON CONFLICT (id) DO UPDATE
SET status = EXCLUDED.status,
    updated_at = NOW();

-- Idempotent: run it 10 times → same result
```

### Natural Keys vs Generated Keys
```
Generated key (UUID) approach:
  Client generates UUID before sending request
  UUID becomes idempotency key
  Any retry uses same UUID
  Server deduplicates on UUID

Natural key approach:
  Use business logic to define uniqueness
  "One order per user per product per day"
  UNIQUE constraint on (user_id, product_id, DATE(created_at))
  Duplicate insert fails → client knows order already exists
```

### Conditional Updates (Optimistic Locking)
```java
// Only update if version matches — prevents double-processing
@Modifying
@Query("UPDATE Order o SET o.status = :status, o.version = o.version + 1 " +
       "WHERE o.id = :id AND o.version = :version")
int updateStatusIfVersionMatches(String id, String status, int version);

// If returns 0 rows → someone else already updated → skip
```

---

## The Idempotency Key Lifecycle

```
Key generated by client:  immediately before request
Key valid for:            24-48 hours (Stripe: 24h)
Key storage:              Redis with TTL or DB with expiry

Client rules:
→ Same key for retries of the SAME logical operation
→ New key for new logical operations
→ Store key until you receive successful response

Server rules:
→ Process once per key
→ Return same response for duplicate keys
→ Expire keys after TTL
→ Reject mismatched request bodies (same key, different body)
```

---

## Key Takeaways

```
Idempotency: Same operation N times = same as once
             Critical because retries are inevitable

HTTP methods:
  Naturally idempotent: GET, PUT, DELETE
  Not idempotent: POST, PATCH (by default)

Implementation:
  Idempotency-Key header (UUID per logical operation)
  Server stores key → result in Redis (TTL 24h)
  Duplicate request → return cached result, skip processing
  Different body, same key → reject 422

Kafka semantics:
  At-most-once:   fast, possible loss
  At-least-once:  safe, possible duplicates
  Exactly-once:   idempotent producer + transactions

Database patterns:
  UPSERT (ON CONFLICT DO UPDATE)
  Natural unique constraints
  Optimistic locking (version field)

Real usage:
  Stripe, Braintree: Idempotency-Key header for all POST
  Kafka: enable.idempotence=true on producer
  Your services: dedup table for critical operations

Rule: Any operation that changes state AND can be retried
      MUST be idempotent. No exceptions.
```
