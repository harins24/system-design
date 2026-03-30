# Webhooks

**Phase 3 — API Fundamentals | Topic 5 of 8**

---

## What is a Webhook?

A webhook is a mechanism where one system notifies another system by making an HTTP request when an event occurs — instead of the second system repeatedly asking "did anything happen?"

```
Polling (what you do without webhooks):
Your System: "Did payment succeed?" → Stripe: "No"
Your System: "Did payment succeed?" → Stripe: "No"
Your System: "Did payment succeed?" → Stripe: "No"
Your System: "Did payment succeed?" → Stripe: "Yes!" ← finally

Webhook (what Stripe does):
Payment succeeds → Stripe calls YOUR endpoint immediately
POST https://yoursite.com/webhooks/stripe
{"event": "payment.succeeded", "amount": 9999, "orderId": "456"}
→ You process instantly, no polling
```

Webhooks are sometimes called **reverse APIs** or **HTTP callbacks** — instead of you calling them, they call you.

---

## How Webhooks Work

```
Setup phase (one time):
1. You register your webhook URL with the provider
   "Send events to: https://api.yoursite.com/webhooks/stripe"
2. You specify which events you want
   "payment.succeeded, payment.failed, refund.created"

Runtime (every time event occurs):
1. Event happens in provider's system
   (customer pays → payment.succeeded)
2. Provider makes HTTP POST to your registered URL
3. Your endpoint receives the payload
4. You process it (update order status, send email, etc.)
5. You return 200 OK to acknowledge receipt
6. If you don't return 200, provider retries
```

### Webhook Payload Structure

```json
// Stripe sends this to your endpoint
POST https://api.yoursite.com/webhooks/stripe
Content-Type: application/json
Stripe-Signature: t=1708934400,v1=a3f8c2d1...  ← security signature

{
  "id": "evt_1234567890",
  "type": "payment_intent.succeeded",
  "created": 1708934400,
  "data": {
    "object": {
      "id": "pi_abc123",
      "amount": 9999,
      "currency": "usd",
      "metadata": {
        "orderId": "order_456",
        "userId": "user_789"
      }
    }
  }
}
```

### Implementing a Webhook Receiver — Spring Boot

```java
@RestController
@RequestMapping("/webhooks")
public class WebhookController {

    private final OrderService orderService;
    private final WebhookEventRepository eventRepository;

    @PostMapping("/stripe")
    public ResponseEntity<Void> handleStripeWebhook(
            @RequestBody String payload,
            @RequestHeader("Stripe-Signature") String signature) {

        // Step 1: Verify signature (CRITICAL - do this first)
        if (!stripeService.verifySignature(payload, signature)) {
            return ResponseEntity.status(403).build();
        }

        // Step 2: Parse event
        StripeEvent event = objectMapper.readValue(payload, StripeEvent.class);

        // Step 3: Idempotency check - have we processed this event?
        if (eventRepository.existsById(event.getId())) {
            return ResponseEntity.ok().build(); // Already processed, return 200
        }

        // Step 4: Acknowledge immediately (return 200 fast)
        // Don't do heavy processing here - push to queue
        kafkaTemplate.send("webhook-events", event.getId(), payload);

        // Step 5: Mark as received
        eventRepository.save(new WebhookEvent(event.getId(), "RECEIVED"));

        return ResponseEntity.ok().build();
    }
}

// Separate consumer processes events asynchronously
@KafkaListener(topics = "webhook-events")
public void processWebhookEvent(String payload) {
    StripeEvent event = objectMapper.readValue(payload, StripeEvent.class);

    switch (event.getType()) {
        case "payment_intent.succeeded":
            orderService.markAsPaid(event.getData().getOrderId());
            notificationService.sendConfirmation(event.getData().getUserId());
            break;
        case "payment_intent.failed":
            orderService.markAsFailed(event.getData().getOrderId());
            notificationService.sendFailureAlert(event.getData().getUserId());
            break;
    }

    eventRepository.updateStatus(event.getId(), "PROCESSED");
}
```

---

## Security — The Most Critical Part

**Anyone can POST to your webhook endpoint.** Without verification, an attacker can forge fake events:

```
Attacker sends:
POST https://api.yoursite.com/webhooks/stripe
{"type": "payment_intent.succeeded", "orderId": "456"}

Without verification → you mark order as paid
Customer got product without paying → fraud
```

### HMAC Signature Verification

```
How it works:
1. Provider has a secret key (you set this during registration)
2. When sending webhook, provider computes:
   HMAC-SHA256(payload + timestamp, secret_key) = signature
3. Provider includes signature in request header
4. You recompute HMAC with YOUR copy of secret key
5. If signatures match → request is authentic
```

```java
public boolean verifyStripeSignature(String payload,
                                      String signatureHeader) {
    // Header format: "t=timestamp,v1=signature"
    String timestamp = extractTimestamp(signatureHeader);
    String receivedSig = extractSignature(signatureHeader);

    // Replay attack prevention: reject old webhooks
    long webhookTime = Long.parseLong(timestamp);
    if (System.currentTimeMillis()/1000 - webhookTime > 300) {
        return false; // Reject webhooks older than 5 minutes
    }

    // Recompute expected signature
    String signedPayload = timestamp + "." + payload;
    String expectedSig = hmacSha256(signedPayload, stripeWebhookSecret);

    // Constant-time comparison (prevents timing attacks)
    return MessageDigest.isEqual(
        expectedSig.getBytes(),
        receivedSig.getBytes()
    );
}
```

---

## Idempotency — Handle Duplicate Events

Webhook providers retry if they don't get a 200 quickly. Your endpoint will receive the same event multiple times.

```
Timeline:
t=0:    Stripe sends webhook → your server receives it
t=0:    Your server starts processing (slow DB write)
t=30s:  Stripe times out (didn't get 200 in 30s)
t=30s:  Stripe retries same event
t=35s:  Your server finishes processing first event → 200
t=35s:  Your server receives duplicate event → processes again?

Without idempotency:
→ Order marked as paid twice
→ Confirmation email sent twice
→ Inventory decremented twice

With idempotency:
→ Check event ID against processed events table
→ Already exists → return 200, skip processing
→ Exactly-once semantics
```

```sql
-- Events table for idempotency
CREATE TABLE webhook_events (
    event_id     VARCHAR(255) PRIMARY KEY,
    provider     VARCHAR(50),
    event_type   VARCHAR(100),
    payload      JSONB,
    status       VARCHAR(20),  -- RECEIVED, PROCESSING, PROCESSED, FAILED
    received_at  TIMESTAMP,
    processed_at TIMESTAMP
);

-- Before processing: INSERT with ON CONFLICT DO NOTHING
-- If insert succeeds → first time seeing this event → process it
-- If insert fails (conflict) → already seen → skip
INSERT INTO webhook_events (event_id, status, received_at)
VALUES ('evt_123', 'RECEIVED', NOW())
ON CONFLICT (event_id) DO NOTHING
RETURNING event_id;
-- If no row returned → duplicate → skip
```

---

## Respond Fast, Process Async

**Critical rule:** Return 200 within a few seconds or the provider will retry.

```
Bad pattern (synchronous):
Webhook arrives → process everything → send emails →
update DB → call 3 services → return 200
                  ↑
           takes 10 seconds
           provider times out at 5s → retries
           now you're processing it twice

Good pattern (async):
Webhook arrives → verify signature → push to Kafka → return 200
                                          ↓
                               Consumer processes asynchronously
                               No timeout risk
                               Retry logic handled by Kafka
```

---

## Retry Logic — What Providers Do

```
Most webhook providers retry with exponential backoff:

Attempt 1: immediately
Attempt 2: 5 minutes later
Attempt 3: 30 minutes later
Attempt 4: 2 hours later
Attempt 5: 5 hours later
Attempt 6: 10 hours later
...
Attempt N: give up after 72 hours / 16 attempts

Your responsibilities:
→ Return 200 quickly (within 5-30 seconds depending on provider)
→ Handle duplicates idempotently
→ Monitor failed webhooks in provider dashboard
→ Provide a "replay" mechanism for manually retrying failed events
```

---

## Webhooks vs Polling vs WebSockets vs Kafka

```
Polling:
  You ask provider repeatedly
  Simple to implement, wasteful
  Use: when webhooks not available, infrequent events

Webhook:
  Provider notifies you via HTTP push
  Efficient, real-time, event-driven
  Use: cross-system notifications, third-party integrations
  Examples: Stripe, GitHub, Twilio, Shopify, Slack

WebSocket:
  Persistent bidirectional connection
  Real-time, low latency
  Use: user-facing real-time features (chat, live updates)

Kafka:
  Internal event streaming between your own services
  High throughput, persistent, replayable
  Use: internal microservice communication

Rule of thumb:
  Kafka for internal service-to-service events
  Webhooks for cross-company/cross-system events
  WebSockets for server-to-browser real-time
```

---

## Building Your Own Webhook System

If you're the provider (sending webhooks to customers), not just the receiver:

```
Architecture:
                                    Customer A endpoint
Event occurs → Webhook Service  ──► Customer B endpoint
               (your system)    ──► Customer C endpoint

Components needed:
1. Event capture
   Service publishes events to Kafka topic

2. Webhook dispatcher
   Consumes from Kafka
   Looks up registered endpoints for this event type
   Makes HTTP POST to each endpoint

3. Retry queue
   If POST fails, push to retry queue with delay
   Exponential backoff: 1m, 5m, 30m, 2h, 5h...

4. Dead letter queue
   After N failures → DLQ → alert customer

5. Webhook management API
   POST /webhooks          → register endpoint + events
   GET  /webhooks          → list registrations
   DELETE /webhooks/{id}   → unregister
   GET  /webhooks/events   → delivery history, retry

6. Signature generation
   Generate secret per customer
   HMAC-SHA256 every payload before sending
   Include in header
```

```java
@Service
public class WebhookDispatcher {

    @KafkaListener(topics = "domain-events")
    public void dispatchWebhooks(DomainEvent event) {
        List<WebhookRegistration> registrations =
            webhookRepository.findByEventType(event.getType());

        for (WebhookRegistration reg : registrations) {
            try {
                String payload = objectMapper.writeValueAsString(event);
                String signature = hmac.sign(payload, reg.getSecret());

                restTemplate.exchange(
                    RequestEntity.post(reg.getUrl())
                        .header("X-Signature", signature)
                        .header("X-Event-Type", event.getType())
                        .body(payload),
                    Void.class
                );

                deliveryLog.record(reg.getId(), event.getId(), "SUCCESS");

            } catch (Exception e) {
                retryQueue.schedule(reg, event, nextRetryDelay(attempt));
                deliveryLog.record(reg.getId(), event.getId(), "FAILED");
            }
        }
    }
}
```

---

## Key Takeaways

```
Webhook: HTTP callback — provider calls you when event occurs
         Reverse API / event-driven HTTP

vs Polling:   Webhooks push instantly, polling wastes requests
vs Kafka:     Webhooks = cross-system; Kafka = internal services
vs WebSocket: Webhooks = server-to-server; WS = server-to-browser

Critical implementation rules:
  Verify HMAC signature before doing anything
  Return 200 immediately (within 5 seconds)
  Process asynchronously (Kafka, queue)
  Handle duplicates idempotently (event ID dedup table)
  Prevent replay attacks (timestamp check ±5 min)

Building your own webhook system needs:
  Event capture → dispatcher → retry queue → DLQ
  Signature per customer → delivery logs → replay API

Real examples:
  Stripe:  payment.succeeded, payment.failed
  GitHub:  push, pull_request, issue
  Twilio:  message delivered, call completed
  Shopify: order.created, inventory.updated
```
