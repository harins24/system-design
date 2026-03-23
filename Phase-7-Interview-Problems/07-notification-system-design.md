# Designing a Notification System

**Phase 7 — Tradeoffs + Interview Problems | Topic 7 of 8**

---

## Overview

A notification system is a great interview question because it's deceptively simple on the surface but reveals deep knowledge of fan-out at scale, multi-channel delivery, reliability patterns, and user preference management. It also appears as a component in almost every other system design question.

---

## Phase 1: Requirements

### Functional Requirements

```
Core features:
  1. Send notifications to users via multiple channels:
     Push (iOS/Android), Email, SMS, In-app
  2. Respect user preferences (opt-out per channel per type)
  3. Notification types: transactional and marketing
  4. Guaranteed delivery (at-least-once)

Questions to ask:
  "What notification channels? Push, email, SMS, in-app, all?"
  "Who triggers notifications? Internal services or external?"
  "Do we need scheduling? (send at 9am user's local time)"
  "Do we need templating? (Hello {name}, your order {orderId}...)"
  "Rate limiting? (don't spam users)"
  "Analytics? (delivery rate, open rate, click-through)"

For this walkthrough:
  All four channels: push, email, SMS, in-app
  Internal services trigger (Order Service, Payment Service, etc.)
  Scheduling: yes (time-zone aware)
  Templating: yes (variable substitution)
  Rate limiting: yes (per user, per channel)
  Basic delivery analytics
```

### Non-Functional Requirements

```
Scale:
  500M users
  10M notifications/day across all channels
  (Push: 7M, Email: 2M, SMS: 500K, In-app: 500K)
  = ~116 notifications/sec average
  Peak: 10x = 1,160/sec (flash sale, breaking news)

Latency:
  Transactional (order confirmation, OTP): <5 seconds
  Marketing (promotions): minutes acceptable

Delivery guarantee:
  Transactional: at-least-once, must be delivered
  Marketing: best-effort, drop if bounced

Availability:
  99.9% for transactional notifications

Reliability:
  No duplicate notifications (user gets same notification twice = bad UX)
  Respect opt-outs immediately (max 15 minutes after user opts out)
```

---

## Phase 2: Capacity Estimation

```
Message volume:
  10M notifications/day ÷ 86,400 = 116/sec average
  Peak burst: 1,160/sec

Per channel:
  Push:   7M/day = 81/sec → via APNs (Apple) + FCM (Google)
  Email:  2M/day = 23/sec → via SendGrid/SES
  SMS:    500K/day = 6/sec → via Twilio (expensive, keep low)
  In-app: 500K/day = 6/sec → internal DB + websocket

Storage:
  Per notification record:
    userId (8B) + type (2B) + channel (1B) + status (1B) +
    templateId (4B) + createdAt (8B) + deliveredAt (8B) = ~32 bytes
    + payload (avg 200 bytes) = 232 bytes per notification

  10M/day × 232 bytes = 2.3 GB/day
  Retain 90 days = 207 GB total
  → Single PostgreSQL with partitioning, or Cassandra

User preferences:
  500M users × channels × notification types
  Assume 10 notification types × 4 channels = 40 preferences per user
  500M × 40 × 1 byte = 20 GB → fits in Redis or DB

Device tokens (push):
  500M users × avg 1.5 devices = 750M device tokens
  Each token: userId (8B) + token (100B) + platform (1B) = 109 bytes
  750M × 109 = 82 GB → DB with index on userId
```

---

## Phase 3: High-Level Design

### Key APIs

```
Internal API (used by other services):
  POST /notifications/send
  Body: {
    userId,
    type: "ORDER_SHIPPED",
    priority: "HIGH",
    data: {orderId, trackingUrl},
    channels?: ["push", "email"],  // optional override
    scheduledAt?: "2026-02-15T09:00:00Z"  // optional
  }

User preference API:
  GET  /users/{userId}/notification-preferences
  PUT  /users/{userId}/notification-preferences
  Body: {
    channels: {
      push:  {ORDER_SHIPPED: true, MARKETING: false},
      email: {ORDER_SHIPPED: true, MARKETING: true},
      sms:   {ORDER_SHIPPED: true, MARKETING: false}
    }
  }

Delivery status API:
  GET /notifications/{notificationId}/status
  Response: {channel, status, deliveredAt, clickedAt}
```

### System Architecture

```
Trigger sources:
  [Order Service] ──┐
  [Payment Service] ─┤→ [Notification API Gateway]
  [Marketing Tool]  ─┘         ↓
                        [Notification Service]
                              ↓
                    [Preference Filter]    ← check user opt-ins
                              ↓
                    [Rate Limiter]         ← prevent spam
                              ↓
                    [Template Engine]      ← render message
                              ↓
                    [Kafka Topics]
                    ├── push-notifications
                    ├── email-notifications
                    ├── sms-notifications
                    └── inapp-notifications
                              ↓
          ┌───────────────────┼──────────────────┐
          ↓                   ↓                  ↓
  [Push Worker]       [Email Worker]      [SMS Worker]
  (APNs/FCM)          (SendGrid/SES)      (Twilio)
          ↓                   ↓                  ↓
  [Delivery DB]       [Delivery DB]       [Delivery DB]
          ↓
  [Analytics Service]
```

---

## Phase 4: Deep Dives

### Deep Dive 1: Channel Routing and Preference Management

```
When a notification arrives, three decisions:
  1. Which channels should this notification go to?
  2. Has the user opted out of any channels?
  3. Is the user eligible right now (rate limit, quiet hours)?

Preference model:
  notification_preferences:
    userId, notificationType, channel, enabled

  Examples:
    (user:123, ORDER_SHIPPED, push,  true)
    (user:123, ORDER_SHIPPED, email, true)
    (user:123, MARKETING,     push,  false)  ← opted out
    (user:123, MARKETING,     email, true)
    (user:123, MARKETING,     sms,   false)  ← opted out

Priority rules:
  Transactional notifications (OTP, password reset):
    Override opt-out → always send (legal requirement)
    Only SMS/email (never push for OTPs — security)

  Marketing notifications:
    Strictly respect opt-out
    Never send if opted out

  System alerts (account suspended):
    Override opt-out → always send

Quiet hours:
  User-configured: "Don't send push between 10pm-8am"
  Store: {userId, pushQuietStart: "22:00", pushQuietEnd: "08:00",
          timezone: "America/Chicago"}

  At send time:
    Convert scheduled time to user's timezone
    If in quiet hours: delay to quiet end time
    Exception: transactional (no quiet hours)
```

```java
@Service
public class NotificationRouter {

    public List<Channel> resolveChannels(
            String userId, String notificationType,
            List<Channel> requestedChannels) {

        NotificationCategory category = getCategory(notificationType);

        // Transactional always goes through
        if (category == TRANSACTIONAL) {
            return requestedChannels.isEmpty()
                ? defaultChannelsFor(notificationType)
                : requestedChannels;
        }

        // Load user preferences from cache
        Map<Channel, Boolean> prefs = preferenceCache.get(userId);

        // Filter by user preferences
        List<Channel> channels = (requestedChannels.isEmpty()
            ? defaultChannelsFor(notificationType)
            : requestedChannels)
            .stream()
            .filter(channel -> prefs.getOrDefault(
                preferenceKey(notificationType, channel), true))
            .filter(channel -> !inQuietHours(userId, channel))
            .collect(toList());

        return channels;
    }
}
```

---

### Deep Dive 2: Reliability — Guaranteed Delivery

```
Notification pipeline must survive failures at every step.

Sources of failure:
  Notification Service crashes after routing, before Kafka
  Kafka consumer (Push Worker) crashes mid-processing
  APNs/FCM API is down
  User's device is offline

Solution: every step writes state before advancing

Step 1: Persist notification before anything
  [Notification Service] → writes to NotificationDB with status=PENDING
  → then publishes to Kafka
  → if Kafka publish fails: status stays PENDING → retry job catches it

Step 2: Kafka as durable buffer
  Push Worker reads from Kafka (at-least-once)
  ACKs offset only AFTER successful delivery or permanent failure
  If worker crashes: Kafka replays from last committed offset

Step 3: Idempotent delivery
  Worker may process same notification twice (Kafka replay)
  Must not deliver twice to user

  Idempotency key: notificationId + channel
  Before sending to APNs: check idempotency store

  Redis:
    SET delivered:{notificationId}:push NX EX 86400
    NX: only set if not exists
    If returns nil: already sent → skip
    If returns OK: proceed with send

Step 4: Dead letter queue
  APNs returns permanent error (invalid token, app uninstalled)
  → Don't retry → move to DLQ → remove device token from DB

Step 5: Retry with backoff for transient failures
  APNs rate limited (429) or timeout
  → Retry with exponential backoff: 1s, 2s, 4s, 8s (max 4 retries)
  → After 4 retries: DLQ + alert
```

```java
@KafkaListener(topics = "push-notifications", groupId = "push-workers")
public void processPushNotification(
        PushNotificationTask task,
        Acknowledgment ack) {

    String idempotencyKey = "delivered:" + task.getNotificationId() + ":push";

    // Check idempotency
    Boolean firstTime = redis.opsForValue()
        .setIfAbsent(idempotencyKey, "1", Duration.ofDays(1));

    if (Boolean.FALSE.equals(firstTime)) {
        log.info("Duplicate notification skipped: {}", task.getNotificationId());
        ack.acknowledge();
        return;
    }

    try {
        DeliveryResult result = pushProvider.send(
            task.getDeviceToken(),
            task.getTitle(),
            task.getBody(),
            task.getData()
        );

        notificationRepo.updateStatus(task.getNotificationId(),
            result.isSuccess() ? DELIVERED : FAILED,
            result.getError());

        if (result.isInvalidToken()) {
            // Device uninstalled → remove token
            deviceTokenRepo.delete(task.getDeviceToken());
        }

        ack.acknowledge(); // commit offset

    } catch (RateLimitException e) {
        redis.delete(idempotencyKey); // allow retry
        throw e; // don't ack → Kafka replays with backoff
    }
}
```

---

### Deep Dive 3: Push Notification Architecture

```
Two push platforms:
  APNs (Apple Push Notification service): iOS devices
  FCM (Firebase Cloud Messaging): Android + web

Device token management:
  When user installs app → device registers → gets token
  Token sent to our backend → stored in device_tokens table

  device_tokens:
    userId, deviceToken, platform (ios/android),
    createdAt, lastActiveAt, isActive

  Tokens change: app reinstall, iOS background refresh
  → App sends new token on each launch
  → Upsert: UPDATE device_tokens SET token=new WHERE userId+platform

APNs integration:
  HTTP/2 connection to api.push.apple.com (persistent, reused)
  Authentication: JWT or certificate-based
  Request:
  {
    "aps": {
      "alert": {"title": "Order Shipped!", "body": "Your order #123 is on the way"},
      "badge": 3,
      "sound": "default"
    },
    "orderId": "123",         ← custom data
    "deepLink": "myapp://orders/123"
  }

  APNs response codes:
  200: success
  410: device token invalid (uninstalled) → remove token
  429: rate limited → backoff
  503: APNs overloaded → retry

High throughput connection pool:
  APNs uses HTTP/2 (multiplexed: many requests per connection)
  1 HTTP/2 connection handles ~1000 concurrent requests
  At 81 push/sec: 1 connection sufficient, pool of 5 for resilience

  At peak 810 push/sec: pool of 5 handles easily
```

---

### Deep Dive 4: Email Delivery

```
Email is complex:
  Deliverability: must not land in spam
  Bounces: invalid address → stop sending
  Unsubscribes: CAN-SPAM / GDPR compliance
  Templates: rich HTML rendering
  Attachments: invoices, tickets

Third-party providers (don't build this):
  SendGrid, AWS SES, Mailgun, Postmark
  They handle: spam reputation, bounces, unsubscribes, analytics

  We call their API:
  sendgrid.send({
    to: "user@email.com",
    from: "noreply@ourapp.com",
    templateId: "order-shipped-v2",
    dynamicTemplateData: {
      name: "Hari",
      orderId: "123",
      trackingUrl: "https://..."
    }
  })

Bounce handling:
  Hard bounce: invalid address (user doesn't exist)
    → Never send to this address again
    → Mark email as invalid in DB

  Soft bounce: mailbox full, temporary failure
    → Retry up to 3 times over 24 hours
    → After 3 failures: pause and alert

  SendGrid webhook → our webhook endpoint → update user DB

Email rate limiting (avoiding spam classification):
  Per domain: max 1M emails/day from single IP
  Warm up: gradually increase volume on new IPs
  SendGrid manages this automatically (managed IPs)

  Our rate limiting: max 5 marketing emails/user/month
  Hard limit: max 20 emails/user/day (all types)
```

---

### Deep Dive 5: Scheduling and Timezone Handling

```
Marketing notifications:
  "Send this flash sale notification at 9am in each user's timezone"

  Not: send to everyone at 9am UTC
  Not: store 500M scheduled jobs

Solution: time-zone bucketed delivery

1. Segment users by UTC offset:
   UTC-8 (Pacific): 100M users → send at 09:00 = 17:00 UTC
   UTC-5 (Eastern): 80M users  → send at 09:00 = 14:00 UTC
   UTC+0 (London):  40M users  → send at 09:00 = 09:00 UTC
   UTC+5:30 (India): 60M users → send at 09:00 = 03:30 UTC
   ...24 buckets

2. Scheduler creates batch jobs per timezone bucket
   "At 14:00 UTC: fan out ORDER_PROMO to 80M Eastern users"

3. Batch job reads user IDs in that timezone
   Queues notifications to Kafka in batches of 1000

Rate limiting the batch:
  80M users / 60 minutes = 1.3M/min = 22K/sec
  Peak burst at exactly 9am
  → Use leaky bucket: even 22K/sec → fine for email workers
  → SMS would be too expensive at this scale: use sampling or skip
```

```java
@Scheduled(cron = "0 * * * * *")  // every minute
public void processScheduledNotifications() {
    Instant now = Instant.now();

    // Find notifications scheduled for this minute
    List<ScheduledNotification> due = scheduledRepo.findDue(
        now, now.plus(Duration.ofMinutes(1)));

    due.parallelStream().forEach(scheduled -> {
        try {
            // Expand to individual users if batch campaign
            if (scheduled.isCampaign()) {
                campaignExpander.expand(scheduled);  // async, uses Kafka
            } else {
                notificationService.send(scheduled.getUserId(),
                    scheduled.getType(), scheduled.getData());
            }
            scheduledRepo.markProcessed(scheduled.getId());
        } catch (Exception e) {
            scheduledRepo.markFailed(scheduled.getId(), e.getMessage());
        }
    });
}
```

---

### Deep Dive 6: In-App Notifications

```
In-app: notifications visible inside the app
  "You have 3 new notifications"
  Notification center / bell icon

Two delivery mechanisms:

Polling (simple):
  Client polls GET /notifications?since=lastSeen every 30s
  Easy to implement, works everywhere
  Cost: 500M × 2 polls/min = 17M req/min = 280K req/sec
  → Too expensive for 30s polling on 500M users
  → Use longer interval (5 min) or only on app foreground

WebSocket (real-time):
  Persistent connection per user
  Server pushes notification immediately

  Challenge: 500M concurrent WebSocket connections
  → Only maintain for active users (app in foreground)
  → At any moment: 5% of DAU active = 25M connections

  25M WebSocket connections:
  Each connection: ~50KB RAM + 1 file descriptor
  25M × 50KB = 1.25 TB RAM → need cluster of WebSocket servers

  WebSocket servers: stateful → sticky routing (same user → same server)
  Server crashes → client reconnects → re-establishes connection
```

```java
@Component
public class NotificationWebSocketHandler
    extends TextWebSocketHandler {

    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String userId = extractUserId(session);
        sessionRegistry.register(userId, session);
    }

    public void sendToUser(String userId, Notification notification) {
        WebSocketSession session = sessionRegistry.get(userId);
        if (session != null && session.isOpen()) {
            session.sendMessage(new TextMessage(
                objectMapper.writeValueAsString(notification)));
        }
        // Always persist to DB regardless of WebSocket delivery
        notificationRepo.save(notification);
    }
}
```

---

### Deep Dive 7: Notification Rate Limiting

```
Prevent notification fatigue (user gets 50 notifications/day = uninstall)

Per-user rate limits:
  Push: max 10/day total, max 1/hour per type
  Email: max 5/day total, max 2/month marketing
  SMS:   max 3/day total (expensive + intrusive)

Implementation:
  Redis counter with TTL

  Key: ratelimit:{userId}:push:day  → max 10, TTL 24hr
  Key: ratelimit:{userId}:email:month → max 5, TTL 30 days

  On each notification attempt:
    count = INCR ratelimit:{userId}:push:day
    if count == 1: EXPIRE key 86400  ← set TTL on first increment
    if count > 10: drop notification (or delay to tomorrow)

  Priority override:
    Transactional notifications skip rate limit
    OTP, password reset, account security → always send
```

```java
@Service
public class NotificationRateLimiter {

    public boolean isAllowed(String userId, Channel channel,
                              NotificationCategory category) {
        if (category == TRANSACTIONAL) return true; // never rate limit

        String key = String.format("ratelimit:%s:%s:day", userId, channel);
        Long count = redis.opsForValue().increment(key);
        if (count == 1) redis.expire(key, Duration.ofDays(1));

        int limit = getLimitFor(channel);
        return count <= limit;
    }
}
```

---

## Phase 5: Tradeoffs

```
1. Kafka as notification buffer:
   Trade-off: latency (message sits in Kafka before processing)
   At 116/sec: Kafka lag is milliseconds, not seconds
   Benefit: decoupled workers, retry on failure, replay
   Accepted for all but most latency-critical transactional

2. Third-party email/SMS vs build own:
   Trade-off: vendor dependency vs massive engineering cost
   Building deliverability reputation, bounce handling,
   carrier relationships takes years and hundreds of engineers
   Always use third-party for email/SMS
   Push: can use APNs/FCM directly (simpler protocols)

3. At-least-once delivery + idempotency:
   Trade-off: complexity of idempotency checks
   Alternative: at-most-once (drop duplicates, simpler)
   For OTP/transactional: must guarantee delivery → at-least-once
   For marketing: at-most-once acceptable

4. WebSocket vs polling for in-app:
   Trade-off: WebSocket = real-time but stateful (harder to scale)
              Polling = simple but latency and DB load
   Hybrid: WebSocket for foreground users, polling on resume
   Accepted: right tool per use case

5. Opt-out propagation delay:
   User opts out → preference updated in DB
   Redis cache TTL: 15 minutes
   During TTL: user may still get notifications
   Trade-off: cache freshness vs performance
   For legal compliance (email unsubscribe): invalidate cache immediately
```

---

## Key Takeaways

```
Notification system: multi-channel, reliable, preference-aware

Channels:
  Push:   APNs (iOS) + FCM (Android) — free, immediate
  Email:  SendGrid/SES — deliverability is the hard problem
  SMS:    Twilio — expensive, use sparingly
  In-app: WebSocket (real-time) + DB (persistent)

Reliability:
  Persist before Kafka (no lost notifications)
  Kafka as durable buffer (replay on failure)
  Idempotency: SET NX in Redis prevents duplicates
  DLQ: permanent failures don't block the queue

Preference management:
  Stored in DB, cached in Redis (15-min TTL)
  Transactional: override opt-out (legal)
  Marketing: strictly respect opt-out
  Quiet hours: per-user timezone-aware

Rate limiting:
  Per-user per-channel daily/monthly limits
  Prevents notification fatigue
  Transactional exempted

Scheduling:
  Timezone-bucketed delivery
  24 UTC-offset buckets
  Batch job per bucket at correct local time

Fan-out at scale:
  10M notifications/day = 116/sec average
  Marketing campaigns: burst to 1000s/sec
  Kafka workers per channel absorb burst

Push token management:
  Token changes on reinstall
  Upsert on app launch
  Remove on APNs 410 (invalid token)

Delivery tracking:
  Every notification: status (PENDING/DELIVERED/FAILED)
  Delivery analytics: per channel, per type, per time
```
