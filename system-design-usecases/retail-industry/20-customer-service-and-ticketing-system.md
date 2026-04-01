---
title: Q20 - Customer Service and Ticketing System
layout: default
---

# Q20 - Customer Service and Ticketing System — Deep Dive Design

> **Scenario**: Build a unified customer support platform. Customers create tickets via web form, email, or chat. Tickets are routed to appropriate teams (orders, returns, technical). Support agents respond with order context. Track SLAs (respond within 24h). Escalate unresolved tickets. Provide knowledge base for self-service. Analyze ticket trends. Support 100K tickets/month, 5K concurrent agents.
>
> **Tech Stack**: Java 17 + Spring Boot 3 + PostgreSQL + MongoDB + Redis + Kafka + Elasticsearch

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
- **Multi-Channel Intake**: Web form, email (IMAP/SMTP), chat (WebSocket), phone (webhook from call center)
- **Automated Routing**: Category-based routing (orders, returns, technical, billing) with load balancing
- **SLA Tracking**: Respond within 24 hours; escalate if breached
- **Knowledge Base**: Self-service articles; suggest when creating ticket
- **Unified Inbox**: Agents see all channels in single interface
- **Customer Context**: Fetch order history, previous tickets, account info on agent view
- **Escalation Workflow**: Unresolved after 14 days → escalate to senior agent
- **Analytics**: Ticket volume, resolution time, CSAT, team performance
- **Conversation History**: Full message history, file attachments

### Non-Functional Requirements
- **Availability**: 99.9% uptime
- **Latency**: Agent dashboard load < 2s, ticket creation < 500ms
- **Throughput**: 100K tickets/month (~3,300/day), 500 concurrent ticket creators
- **Data Consistency**: Strong for ticket state, eventual for analytics
- **Response Time**: Email response < 1s, chat response < 100ms
- **Scalability**: Support 1M+ tickets, 5K agents, 10K+ KB articles

### Constraints
- Email delivery not guaranteed (handle retries, DLQ)
- Chat messages transient (WebSocket), must persist to DB
- File uploads limited to 100MB per attachment
- SLA definitions per team (can vary)
- Customer privacy: don't expose agent names to customer

---

## 2. Capacity Estimation

| Metric | Value | Notes |
|--------|-------|-------|
| Monthly Tickets | 100,000 | 3,300/day average, 150/day/team |
| Daily Tickets (peak) | 5,000 | ~6 tickets/min at peak hours |
| Concurrent Agents | 5,000 | Across all shifts |
| Tickets per Agent/Day | 20 | Average handling capacity |
| Avg Resolution Time | 36 hours | 24h SLA + some escalations |
| Messages per Ticket | 3 | Customer + agent exchanges |
| Chat Sessions/Day | 2,000 | Real-time support |
| Email Replies/Day | 8,000 | Asynchronous replies |
| Knowledge Base Articles | 10,000 | ~1K per category |
| Daily Article Views | 50,000 | Self-service traffic |
| Avg Ticket Age | 2 days | Until resolved |

### Storage Estimates
- **PostgreSQL**: Tickets (~50GB/year), messages (~100GB/year), escalations (~5GB/year)
- **MongoDB**: Ticket audit trail (~50GB/year), full message history (~100GB/year)
- **Elasticsearch**: KB articles (~500MB), ticket full-text search (~10GB/year)
- **File Storage (S3)**: Attachments (~1TB/year), document uploads (~500GB/year)
- **Redis**: Agent sessions (~100MB), active tickets cache (~500MB)

### Network Bandwidth
- Email ingress: ~10 Mbps, IMAP polling 4x/day
- Chat: ~50 Mbps peak, WebSocket real-time
- Database: ~100K queries/s average, 500K/s peak
- Outbound (customer notifications): ~20 Mbps

---

## 3. High-Level Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                    Multi-Channel Intake                         │
├─────────────────┬─────────────────┬─────────────────┬──────────┤
│  Web Form       │  Email (IMAP)   │ Chat (WebSocket)│  Phone   │
│  Portal         │  Poller         │  Real-time      │  Webhook │
└────────┬────────┴────────┬────────┴────────┬────────┴────┬─────┘
         │                 │                 │             │
         └─────────────────┼─────────────────┼─────────────┘
                           │
              ┌────────────▼────────────┐
              │   Kafka Event Bus       │
              │  (Unified Intake)       │
              └────────────┬────────────┘
                           │
         ┌─────────────────┼─────────────────┐
         │                 │                 │
    ┌────▼───────┐  ┌─────▼───────┐  ┌────▼───────┐
    │ Ticket     │  │ Routing     │  │ SLA        │
    │ Service    │  │ Engine      │  │ Tracker    │
    └────┬───────┘  └─────┬───────┘  └────┬───────┘
         │                │               │
         └────────────────┼───────────────┘
                          │
         ┌────────────────▼────────────────┐
         │   Agent Dashboard Service       │
         │   (Context Aggregation)         │
         └────────────┬───────────────────┘
                      │
              ┌───────▼──────────────┐
              │  Response Service    │
              │  (Email/Chat Send)   │
              └───────┬──────────────┘
                      │
      ┌───────────────┼───────────────┐
      │               │               │
 ┌────▼───┐      ┌────▼────┐    ┌────▼────┐
 │SMTP    │      │ Kafka   │    │ WebSocket│
 │Service │      │ (async) │    │ Handler  │
 └────────┘      └────────┘    └──────────┘
      │               │              │
┌─────▼──────┐  ┌────▼──────┐  ┌────▼─────┐
│PostgreSQL  │  │ Elasticsearch│ │ MongoDB  │
│(Tickets)   │  │(KB + search)  │(History) │
└────────────┘  └───────────────┘ └──────────┘
```

---

## 4. Core Design Questions Answered

### Q1: How do you design the ticket routing and assignment logic?

**Answer**: Rules engine with category detection, load balancing, and priority-based queue.

**Design**:
1. **Category Detection**: ML model analyzes ticket subject/body → predict category (Orders, Returns, Technical, Billing)
2. **Routing Rules**:
   ```
   if category == "ORDERS" → route to orders-team
   if category == "RETURNS" → route to returns-team
   if vip_customer == true → add HIGH priority
   if backlog_for_team > 100 → escalate to overflow team
   ```
3. **Assignment Strategy**:
   - **Load Balancing**: Assign to agent with least open tickets
   - **Skill Matching**: Some agents specialize (e.g., VIP support, technical issues)
   - **Capacity Planning**: Don't assign if agent at max capacity (e.g., 20 concurrent tickets)

4. **Queue Management**: Redis sorted set for each team queue
   ```
   orders-team:queue = ZSET {
       ticket_1 (score=priority, created_at),
       ticket_2 (score=priority, created_at)
   }
   ```

5. **Reassignment**: If agent doesn't respond within 1h, reassign to next available agent

---

### Q2: How do you integrate multiple communication channels (email, chat, phone)?

**Answer**: Unified inbox model. Each channel publishes TicketCreatedEvent to Kafka; consumer normalizes and deduplicates.

**Design**:
```
Email (IMAP Poller)
  ↓
  Parse email → Extract subject, body, attachments
  Check for existing thread (subject matching)
  Convert to TicketEvent → Kafka "ticket-intake"

Chat (WebSocket)
  ↓
  Real-time message received
  Create/append to ticket
  Publish ChatMessageEvent → Kafka "chat-messages"

Phone (Call Center Webhook)
  ↓
  Call center sends call_transcript JSON
  Convert to TicketEvent → Kafka "ticket-intake"

All events → Kafka "ticket-intake"
  ↓
TicketConsumer
  ↓
  Normalize to internal Ticket model
  Deduplicate (email reply to same thread → append, not new ticket)
  Save to PostgreSQL
  Publish TicketCreatedEvent → Kafka "ticket-events"
```

**Deduplication**:
- Email: Thread-ID or subject line matching
- Chat: Session ID
- Phone: Call ID + timestamp

**Channel ID**: Each ticket has `channel_id` (email_id, chat_session_id, phone_call_id) for tracking.

---

### Q3: How do you implement escalation workflows?

**Answer**: State machine + SLA timers. Unresolved tickets auto-escalate after thresholds.

**Design**:
```
Ticket State Machine:
  OPEN → ASSIGNED → IN_PROGRESS → WAITING_CUSTOMER → RESOLVED → CLOSED

Escalation Triggers:
  1. No response for 24h → escalate to next-level team
  2. No customer reply for 7 days → close with auto-message
  3. In progress for 14 days → escalate to supervisor
  4. 3+ escalations → auto-refund (disputes only)

Implementation:
  - SLA timer in Redis: ticket:{id}:sla_breached_at = timestamp
  - Background job every 15min: check Redis for breached tickets
  - On breach: change ticket status, assign to supervisor, publish event
```

**Example**:
```
Ticket created: 2024-04-01 10:00
SLA: 24h response required
SLA breach time: 2024-04-02 10:00

Background job checks at 10:15:
  if (now > sla_breach_time && ticket.status != CLOSED):
    ticket.status = ESCALATED
    ticket.assigned_to = supervisor
    publish TicketEscalatedEvent
    notify_customer("Your issue has been escalated")
```

---

### Q4: How do you track and enforce SLAs?

**Answer**: Redis timers + scheduled job. Track response time (first agent message), resolution time.

**Design**:
- **Response SLA**: First agent response within 24h
- **Resolution SLA**: Ticket closed within 72h
- **Tracking**:
  ```
  ticket:{id}:sla:response_deadline = timestamp (created_at + 24h)
  ticket:{id}:sla:resolution_deadline = timestamp (created_at + 72h)
  ticket:{id}:sla:first_response_at = timestamp (when agent first replies)
  ticket:{id}:sla:resolved_at = timestamp (when ticket marked resolved)
  ```

- **Monitoring**:
  ```
  @Scheduled(fixedRate = 900000) // Every 15 min
  public void checkSLABreaches() {
    Set<String> breachedKeys = redisTemplate.keys("ticket:*:sla:response_deadline");
    for (String key : breachedKeys) {
      Long deadline = Long.parseLong(redisTemplate.opsForValue().get(key));
      if (now > deadline && !ticket.hasAgentResponse()) {
        escalate(ticket);
      }
    }
  }
  ```

- **Reporting**:
  ```
  SLA metrics per team:
  - % tickets met response SLA
  - % tickets met resolution SLA
  - Avg response time
  - Avg resolution time
  ```

---

### Q5: How do you provide agents with customer context (order history, previous tickets)?

**Answer**: ContextAggregationService fetches data from multiple systems on agent dashboard load.

**Design**:
```
Agent opens ticket → Dashboard calls /api/tickets/{id}/context
  ↓
ContextAggregationService:
  1. Fetch customer profile (name, email, account status)
  2. Fetch order history (last 10 orders, statuses)
  3. Fetch previous tickets (last 5 resolved tickets)
  4. Fetch account balance, refunds, payment info
  5. Fetch product reviews (if product-related issue)
  6. Aggregate in Redis cache (5-min TTL)

Response:
  {
    "customer": {...},
    "orders": [{order_id, status, created_at, ...}],
    "previous_tickets": [{id, subject, resolved_at, ...}],
    "account": {balance, credit, ...},
    "related_products": [{id, name, rating}]
  }
```

**Performance**: Parallel fetches via CompletableFuture.allOf() → typical latency 500ms–1s.

**Caching**: Cache full context in Redis to avoid repeated DB hits for same customer within 5 min.

---

### Q6: How do you analyze ticket trends to identify product/service issues?

**Answer**: Stream analytics on ticket events. Aggregate by category, keyword extraction, anomaly detection.

**Design**:
```
TicketCreatedEvent → Kafka "ticket-events"
  ↓
TicketAnalyticsConsumer
  ↓
1. Extract keywords (NLP) from ticket subject + body
2. Count by category, product, issue type per day
3. Publish aggregation to MongoDB daily snapshot
4. Feed to ML model for trend detection

Kafka topic: "ticket-analytics"
Event example:
  {
    "event_id": "evt-123",
    "event_type": "TICKET_CREATED",
    "ticket_id": 999,
    "category": "RETURNS",
    "keywords": ["damaged", "packaging", "refund"],
    "product_id": 501,
    "customer_sentiment": "negative",
    "created_at": "2024-04-01T10:00:00Z"
  }

MongoDB collection: "ticket_daily_stats"
  {
    "date": 2024-04-01,
    "category": "RETURNS",
    "issue_keywords": {
      "damaged": 250,
      "packaging": 180,
      "refund": 220
    },
    "product_issues": {
      "501": {count: 25, reason: "DAMAGED"},
      "502": {count: 18, reason: "MISSING"}
    },
    "sentiment_distribution": {
      "positive": 5%,
      "neutral": 30%,
      "negative": 65%
    }
  }

Alerts:
  - If product_id has > 10 tickets in 1 day → flag for product team
  - If category has > 50% sentiment negative → escalate
  - If issue keyword appears > 5x spike vs baseline → investigate
```

---

## 5. Microservices Breakdown

| Service | Responsibility | Tech | Instances |
|---------|-----------------|------|-----------|
| **Ticket Service** | CRUD, state management | Spring Boot + PostgreSQL | 5–10 |
| **Routing Engine** | Category detection, assignment | Spring Boot + Redis | 3–5 |
| **SLA Tracker** | SLA monitoring, escalation | Spring Boot + Redis | 2–3 |
| **Channel Intake** | Email poller, chat handler, phone webhook | Spring Boot + Kafka | 4–6 |
| **Agent Dashboard** | Context aggregation, unified inbox | Spring Boot + PostgreSQL + Redis | 5–8 |
| **Response Service** | Email/chat sending, persistence | Spring Boot + Kafka + PostgreSQL | 3–5 |
| **Knowledge Base** | Article management, search | Spring Boot + Elasticsearch | 3–5 |
| **Analytics Service** | Trend analysis, reporting | Spring Boot + MongoDB + Kafka | 2–4 |

---

## 6. Database Design

### PostgreSQL Schema (OLTP)

```sql
-- Customers (reference)
CREATE TABLE customers (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    phone VARCHAR(20),
    name VARCHAR(255),
    account_status VARCHAR(50),
    created_at TIMESTAMP NOT NULL,
    INDEX idx_email (email)
);

-- Tickets (main entity)
CREATE TABLE tickets (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL REFERENCES customers(id),
    subject VARCHAR(500) NOT NULL,
    description TEXT,
    category VARCHAR(50) NOT NULL, -- ORDERS, RETURNS, TECHNICAL, BILLING
    priority VARCHAR(50) DEFAULT 'NORMAL', -- LOW, NORMAL, HIGH, URGENT
    status VARCHAR(50) NOT NULL, -- OPEN, ASSIGNED, IN_PROGRESS, WAITING_CUSTOMER, RESOLVED, CLOSED
    assigned_to BIGINT, -- Agent ID (FK to users table, not shown)
    channel VARCHAR(50), -- WEB, EMAIL, CHAT, PHONE
    channel_id VARCHAR(255), -- Thread ID, email message ID, chat session ID
    order_id BIGINT, -- Related order (if any)
    product_id BIGINT, -- Related product (if any)
    sentiment VARCHAR(50), -- POSITIVE, NEUTRAL, NEGATIVE
    resolution_notes TEXT,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    resolved_at TIMESTAMP,
    closed_at TIMESTAMP,
    INDEX idx_customer_id (customer_id),
    INDEX idx_status (status),
    INDEX idx_category (category),
    INDEX idx_assigned_to (assigned_to),
    INDEX idx_created_at (created_at)
);

-- Ticket Messages (conversation thread)
CREATE TABLE ticket_messages (
    id BIGSERIAL PRIMARY KEY,
    ticket_id BIGINT NOT NULL REFERENCES tickets(id) ON DELETE CASCADE,
    sender_id BIGINT NOT NULL, -- Customer ID or Agent ID
    sender_type VARCHAR(50), -- CUSTOMER, AGENT
    message_body TEXT NOT NULL,
    attachment_url VARCHAR(500),
    attachment_size INT,
    created_at TIMESTAMP NOT NULL,
    INDEX idx_ticket_id (ticket_id),
    INDEX idx_sender_id (sender_id),
    INDEX idx_created_at (created_at)
);

-- Ticket Escalations (audit trail)
CREATE TABLE ticket_escalations (
    id BIGSERIAL PRIMARY KEY,
    ticket_id BIGINT NOT NULL REFERENCES tickets(id),
    escalated_from VARCHAR(50), -- Team or agent name
    escalated_to VARCHAR(50),
    reason VARCHAR(255),
    escalation_count INT,
    created_at TIMESTAMP NOT NULL,
    INDEX idx_ticket_id (ticket_id),
    INDEX idx_created_at (created_at)
);

-- Knowledge Base Articles
CREATE TABLE kb_articles (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(500) NOT NULL,
    content TEXT,
    category VARCHAR(100),
    tags VARCHAR(500), -- Comma-separated
    view_count INT DEFAULT 0,
    helpful_count INT DEFAULT 0,
    published BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP NOT NULL,
    updated_at TIMESTAMP NOT NULL,
    INDEX idx_category (category),
    INDEX idx_published (published)
);

-- Article Suggestions (when customer creates ticket)
CREATE TABLE article_suggestions (
    id BIGSERIAL PRIMARY KEY,
    ticket_id BIGINT NOT NULL,
    article_id BIGINT NOT NULL REFERENCES kb_articles(id),
    relevance_score DECIMAL(3, 2),
    clicked BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL,
    INDEX idx_ticket_id (ticket_id)
);

-- Ticket SLA Tracking
CREATE TABLE ticket_sla_tracking (
    id BIGSERIAL PRIMARY KEY,
    ticket_id BIGINT NOT NULL UNIQUE REFERENCES tickets(id),
    response_sla_deadline TIMESTAMP,
    resolution_sla_deadline TIMESTAMP,
    first_response_at TIMESTAMP,
    resolved_at TIMESTAMP,
    response_sla_breached BOOLEAN DEFAULT FALSE,
    resolution_sla_breached BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP NOT NULL,
    INDEX idx_ticket_id (ticket_id),
    INDEX idx_response_sla_deadline (response_sla_deadline)
);

-- Agent Assignment Metrics (for load balancing)
CREATE TABLE agent_workload (
    id BIGSERIAL PRIMARY KEY,
    agent_id BIGINT NOT NULL,
    open_tickets INT DEFAULT 0,
    assigned_tickets INT DEFAULT 0,
    avg_resolution_time INT, -- Minutes
    online BOOLEAN DEFAULT TRUE,
    updated_at TIMESTAMP NOT NULL,
    INDEX idx_agent_id (agent_id)
);

-- Chat Sessions (real-time)
CREATE TABLE chat_sessions (
    id BIGSERIAL PRIMARY KEY,
    customer_id BIGINT NOT NULL,
    agent_id BIGINT,
    ticket_id BIGINT REFERENCES tickets(id),
    session_status VARCHAR(50), -- ACTIVE, WAITING, CLOSED
    created_at TIMESTAMP NOT NULL,
    closed_at TIMESTAMP,
    INDEX idx_customer_id (customer_id),
    INDEX idx_agent_id (agent_id)
);

-- Email Config (for reply tracking)
CREATE TABLE email_threads (
    id BIGSERIAL PRIMARY KEY,
    ticket_id BIGINT NOT NULL REFERENCES tickets(id),
    message_id VARCHAR(255) UNIQUE, -- RFC 5322 Message-ID
    thread_id VARCHAR(255), -- Gmail-style thread ID
    reply_to_id VARCHAR(255), -- Message being replied to
    created_at TIMESTAMP NOT NULL,
    INDEX idx_ticket_id (ticket_id),
    INDEX idx_message_id (message_id)
);
```

### MongoDB Collections (Audit Trail & Analytics)

```json
// ticket_message_history (full archive, immutable)
{
    "_id": ObjectId(),
    "ticket_id": 999,
    "customer_id": 123,
    "message_id": "msg-456",
    "sender_type": "AGENT",
    "sender_id": 789,
    "sender_name": "Support Team",
    "message_body": "Thank you for contacting us...",
    "attachments": [
        {
            "filename": "invoice.pdf",
            "url": "s3://...",
            "size_bytes": 50000
        }
    ],
    "created_at": ISODate("2024-04-01T10:30:00Z"),
    "updated_at": ISODate("2024-04-01T10:30:00Z")
}

// ticket_daily_stats (aggregated for analytics)
{
    "_id": ObjectId(),
    "date": ISODate("2024-04-01"),
    "stats": {
        "total_tickets": 3300,
        "by_category": {
            "ORDERS": 1200,
            "RETURNS": 800,
            "TECHNICAL": 800,
            "BILLING": 500
        },
        "by_status": {
            "OPEN": 150,
            "RESOLVED": 3000,
            "CLOSED": 150
        },
        "avg_response_time_minutes": 45,
        "avg_resolution_time_hours": 24,
        "sla_breaches": 85,
        "csat_score": 4.2
    },
    "sentiment_analysis": {
        "positive": 300,
        "neutral": 1500,
        "negative": 1500
    },
    "top_issues": [
        {
            "keyword": "damaged",
            "count": 250,
            "category": "RETURNS"
        }
    ]
}

// ticket_escalation_log
{
    "_id": ObjectId(),
    "ticket_id": 999,
    "escalation_chain": [
        {
            "timestamp": ISODate("2024-04-01T10:00:00Z"),
            "from": "orders-team",
            "to": "returns-team",
            "reason": "MISCLASSIFIED"
        },
        {
            "timestamp": ISODate("2024-04-02T10:00:00Z"),
            "from": "returns-team",
            "to": "supervisor",
            "reason": "SLA_BREACH"
        }
    ]
}
```

---

## 7. Redis Data Structures

```redis
# Active Ticket Cache
ticket:{ticket_id} = HASH {
    "customer_id": "123",
    "status": "IN_PROGRESS",
    "assigned_to": "agent_789",
    "category": "ORDERS",
    "created_at": "1712099999",
    "priority": "HIGH"
}
TTL: 86400 seconds (1 day)

# Queue for each team (sorted by priority + created_at)
team:orders:queue = ZSET {
    ticket_1 (score = priority_weight + timestamp),
    ticket_2,
    ticket_3
}
TTL: 7200 seconds (2 hours, refresh on activity)

# Agent Workload (load balancing)
agent:workload:{agent_id} = HASH {
    "open_tickets": "8",
    "assigned_tickets": "12",
    "avg_resolution_minutes": "240",
    "online": "true"
}
TTL: 3600 seconds (1 hour, refreshed frequently)

# SLA Breaches (for alerts)
sla:breached_tickets = ZSET {
    ticket_id: score = breach_timestamp
}
TTL: 604800 seconds (7 days)

# Email Thread Mapping (dedup)
email:thread:{subject_hash} = STRING "{ticket_id}"
TTL: 2592000 seconds (30 days)

# Chat Session (real-time)
chat:session:{session_id} = HASH {
    "customer_id": "123",
    "agent_id": "789",
    "ticket_id": "999",
    "status": "ACTIVE",
    "created_at": "1712099999"
}
TTL: 86400 seconds (expires if inactive)

# Customer Context Cache
customer:context:{customer_id} = HASH {
    "name": "John Doe",
    "email": "john@example.com",
    "vip": "true",
    "total_orders": "25",
    "last_order_at": "1712099999"
}
TTL: 300 seconds (5 minutes, refresh on load)

# KB Search Results (cache)
kb:search:{query_hash} = STRING "{result_json}"
TTL: 3600 seconds (1 hour)

# SLA Deadlines
ticket:{ticket_id}:sla:response_deadline = STRING "1712186399"
ticket:{ticket_id}:sla:resolution_deadline = STRING "1712359199"
TTL: 345600 seconds (4 days, auto-expire)
```

---

## 8. Kafka Event Flow

### Topics & Events

| Topic | Events | Partition Key | Retention |
|-------|--------|---------------|-----------|
| `ticket-intake` | TicketCreatedEvent (from all channels) | customer_id | 90 days |
| `ticket-events` | TicketAssignedEvent, TicketStatusChangedEvent, TicketEscalatedEvent | ticket_id | 365 days |
| `email-messages` | EmailReceivedEvent, EmailSentEvent | thread_id | 365 days |
| `chat-messages` | ChatMessageEvent, ChatSessionClosedEvent | session_id | 90 days |
| `ticket-analytics` | TicketCreatedEvent, TicketResolvedEvent | category | 365 days |
| `sla-events` | SLABreachEvent, SLAMetEvent | ticket_id | 90 days |
| `response-notifications` | CustomerNotificationEvent | customer_id | 30 days |

### Event Examples

```json
// TicketCreatedEvent
{
    "event_id": "evt-200000",
    "event_type": "TICKET_CREATED",
    "ticket_id": 999,
    "customer_id": 123,
    "channel": "WEB",
    "subject": "Order not received",
    "category": "ORDERS",
    "priority": "NORMAL",
    "created_at": "2024-04-01T10:00:00Z"
}

// TicketAssignedEvent
{
    "event_id": "evt-200001",
    "event_type": "TICKET_ASSIGNED",
    "ticket_id": 999,
    "agent_id": 789,
    "team": "orders-team",
    "assigned_at": "2024-04-01T10:05:00Z"
}

// TicketEscalatedEvent
{
    "event_id": "evt-200002",
    "event_type": "TICKET_ESCALATED",
    "ticket_id": 999,
    "escalation_reason": "SLA_BREACH",
    "escalated_to": "supervisor",
    "escalated_at": "2024-04-02T10:00:00Z"
}

// ChatMessageEvent
{
    "event_id": "evt-200003",
    "event_type": "CHAT_MESSAGE",
    "session_id": "session-xyz",
    "ticket_id": 999,
    "sender_type": "AGENT",
    "message": "Thank you for waiting...",
    "created_at": "2024-04-01T10:15:00Z"
}
```

---

## 9. Implementation Code

### 9.1 TicketService (Core)

```java
@Service
@RequiredArgsConstructor
public class TicketService {
    private final TicketRepository ticketRepository;
    private final TicketMessageRepository messageRepository;
    private final TicketSLARepository slaRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final TicketRouter routingEngine;
    private final RedisTemplate<String, Object> redisTemplate;
    private final ObjectMapper objectMapper;
    private static final Logger logger = LoggerFactory.getLogger(TicketService.class);

    public TicketResponse createTicket(TicketCreationRequest request) {
        // Step 1: Create ticket record
        Ticket ticket = new Ticket();
        ticket.setCustomerId(request.getCustomerId());
        ticket.setSubject(request.getSubject());
        ticket.setDescription(request.getDescription());
        ticket.setChannel(request.getChannel());
        ticket.setChannelId(request.getChannelId());
        ticket.setStatus(TicketStatus.OPEN);
        ticket.setPriority(TicketPriority.NORMAL);
        ticket.setCreatedAt(LocalDateTime.now());

        // Step 2: Detect category (simple keyword matching; in production use ML)
        TicketCategory detectedCategory = detectCategory(request.getSubject(), request.getDescription());
        ticket.setCategory(detectedCategory);

        // Step 3: Detect sentiment (ML model)
        TicketSentiment sentiment = detectSentiment(request.getDescription());
        ticket.setSentiment(sentiment);

        Ticket savedTicket = ticketRepository.save(ticket);

        // Step 4: Create SLA tracking record
        TicketSLA sla = new TicketSLA();
        sla.setTicketId(savedTicket.getId());
        sla.setResponseSLADeadline(LocalDateTime.now().plusHours(24));
        sla.setResolutionSLADeadline(LocalDateTime.now().plusHours(72));
        slaRepository.save(sla);

        // Step 5: Cache ticket
        cacheTicket(savedTicket);

        // Step 6: Publish event
        publishTicketCreatedEvent(savedTicket);

        // Step 7: Route ticket
        routingEngine.routeTicket(savedTicket);

        logger.info("Ticket {} created by customer {}", savedTicket.getId(), request.getCustomerId());

        return new TicketResponse(savedTicket.getId(), TicketStatus.OPEN.name(), detectedCategory.name());
    }

    public void addMessage(Long ticketId, MessageCreationRequest request) {
        Ticket ticket = ticketRepository.findById(ticketId)
            .orElseThrow(() -> new TicketNotFoundException(ticketId));

        // Create message record
        TicketMessage message = new TicketMessage();
        message.setTicketId(ticketId);
        message.setSenderId(request.getSenderId());
        message.setSenderType(request.getSenderType());
        message.setMessageBody(request.getMessageBody());
        message.setCreatedAt(LocalDateTime.now());

        if (request.getAttachmentUrl() != null) {
            message.setAttachmentUrl(request.getAttachmentUrl());
            message.setAttachmentSize(request.getAttachmentSize());
        }

        TicketMessage savedMessage = messageRepository.save(message);

        // Update ticket's updated_at
        ticket.setUpdatedAt(LocalDateTime.now());

        // If agent replied, update SLA first_response_at
        if (request.getSenderType() == SenderType.AGENT && ticket.getStatus() == TicketStatus.OPEN) {
            ticket.setStatus(TicketStatus.ASSIGNED);
            TicketSLA sla = slaRepository.findByTicketId(ticketId).orElse(null);
            if (sla != null && sla.getFirstResponseAt() == null) {
                sla.setFirstResponseAt(LocalDateTime.now());
                slaRepository.save(sla);
            }
        }

        ticketRepository.save(ticket);

        // Cache message
        String cacheKey = "ticket:" + ticketId + ":messages:" + savedMessage.getId();
        redisTemplate.opsForValue().set(cacheKey, savedMessage, Duration.ofHours(24));

        // Publish event
        publishMessageAddedEvent(ticket, savedMessage);

        logger.info("Message {} added to ticket {}", savedMessage.getId(), ticketId);
    }

    public void assignTicket(Long ticketId, Long agentId) {
        Ticket ticket = ticketRepository.findById(ticketId)
            .orElseThrow(() -> new TicketNotFoundException(ticketId));

        ticket.setAssignedTo(agentId);
        ticket.setStatus(TicketStatus.ASSIGNED);
        ticket.setUpdatedAt(LocalDateTime.now());

        Ticket savedTicket = ticketRepository.save(ticket);

        // Update agent workload
        updateAgentWorkload(agentId, 1); // Increment open tickets

        // Cache update
        cacheTicket(savedTicket);

        // Publish event
        publishTicketAssignedEvent(savedTicket, agentId);

        logger.info("Ticket {} assigned to agent {}", ticketId, agentId);
    }

    public void resolveTicket(Long ticketId, String resolutionNotes) {
        Ticket ticket = ticketRepository.findById(ticketId)
            .orElseThrow(() -> new TicketNotFoundException(ticketId));

        ticket.setStatus(TicketStatus.RESOLVED);
        ticket.setResolutionNotes(resolutionNotes);
        ticket.setResolvedAt(LocalDateTime.now());
        ticket.setUpdatedAt(LocalDateTime.now());

        Ticket savedTicket = ticketRepository.save(ticket);

        // Update SLA
        TicketSLA sla = slaRepository.findByTicketId(ticketId).orElse(null);
        if (sla != null) {
            sla.setResolvedAt(LocalDateTime.now());
            sla.setResolutionSLABreached(LocalDateTime.now().isAfter(sla.getResolutionSLADeadline()));
            slaRepository.save(sla);
        }

        // Update agent workload
        if (ticket.getAssignedTo() != null) {
            updateAgentWorkload(ticket.getAssignedTo(), -1); // Decrement open tickets
        }

        // Remove from queue
        removeFromQueue(ticket.getCategory(), ticketId);

        // Publish event
        publishTicketResolvedEvent(savedTicket);

        logger.info("Ticket {} resolved", ticketId);
    }

    public void escalateTicket(Long ticketId, String escalationReason) {
        Ticket ticket = ticketRepository.findById(ticketId)
            .orElseThrow(() -> new TicketNotFoundException(ticketId));

        ticket.setStatus(TicketStatus.ESCALATED);
        ticket.setUpdatedAt(LocalDateTime.now());
        Ticket savedTicket = ticketRepository.save(ticket);

        // Publish escalation event (routing engine will handle re-assignment)
        publishTicketEscalatedEvent(savedTicket, escalationReason);

        logger.info("Ticket {} escalated: {}", ticketId, escalationReason);
    }

    private TicketCategory detectCategory(String subject, String description) {
        String combined = (subject + " " + description).toLowerCase();

        if (combined.contains("order") || combined.contains("track")) {
            return TicketCategory.ORDERS;
        } else if (combined.contains("return") || combined.contains("refund")) {
            return TicketCategory.RETURNS;
        } else if (combined.contains("error") || combined.contains("bug") || combined.contains("crash")) {
            return TicketCategory.TECHNICAL;
        } else if (combined.contains("payment") || combined.contains("invoice") || combined.contains("billing")) {
            return TicketCategory.BILLING;
        }
        return TicketCategory.GENERAL;
    }

    private TicketSentiment detectSentiment(String text) {
        // Stub: in production, call ML sentiment analysis API
        if (text.contains("angry") || text.contains("frustrated")) {
            return TicketSentiment.NEGATIVE;
        }
        return TicketSentiment.NEUTRAL;
    }

    private void cacheTicket(Ticket ticket) {
        String cacheKey = "ticket:" + ticket.getId();
        Map<String, Object> ticketData = new HashMap<>();
        ticketData.put("customer_id", ticket.getCustomerId().toString());
        ticketData.put("status", ticket.getStatus().name());
        ticketData.put("assigned_to", ticket.getAssignedTo() != null ? ticket.getAssignedTo().toString() : "");
        ticketData.put("category", ticket.getCategory().name());

        redisTemplate.opsForHash().putAll(cacheKey, ticketData);
        redisTemplate.expire(cacheKey, Duration.ofHours(24));
    }

    private void updateAgentWorkload(Long agentId, int delta) {
        String key = "agent:workload:" + agentId;
        redisTemplate.opsForHash().increment(key, "open_tickets", delta);
    }

    private void removeFromQueue(TicketCategory category, Long ticketId) {
        String queueKey = "team:" + category.name().toLowerCase() + ":queue";
        redisTemplate.opsForZSet().remove(queueKey, ticketId.toString());
    }

    private void publishTicketCreatedEvent(Ticket ticket) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "TICKET_CREATED");
            event.put("ticket_id", ticket.getId());
            event.put("customer_id", ticket.getCustomerId());
            event.put("category", ticket.getCategory().name());
            event.put("priority", ticket.getPriority().name());
            event.put("created_at", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("ticket-events", ticket.getId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish TicketCreatedEvent", e);
        }
    }

    private void publishMessageAddedEvent(Ticket ticket, TicketMessage message) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "TICKET_MESSAGE_ADDED");
            event.put("ticket_id", ticket.getId());
            event.put("message_id", message.getId());
            event.put("sender_type", message.getSenderType().name());
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("ticket-events", ticket.getId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish MessageAddedEvent", e);
        }
    }

    private void publishTicketAssignedEvent(Ticket ticket, Long agentId) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "TICKET_ASSIGNED");
            event.put("ticket_id", ticket.getId());
            event.put("agent_id", agentId);
            event.put("assigned_at", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("ticket-events", ticket.getId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish TicketAssignedEvent", e);
        }
    }

    private void publishTicketResolvedEvent(Ticket ticket) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "TICKET_RESOLVED");
            event.put("ticket_id", ticket.getId());
            event.put("resolved_at", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("ticket-events", ticket.getId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish TicketResolvedEvent", e);
        }
    }

    private void publishTicketEscalatedEvent(Ticket ticket, String reason) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "TICKET_ESCALATED");
            event.put("ticket_id", ticket.getId());
            event.put("escalation_reason", reason);
            event.put("escalated_at", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("ticket-events", ticket.getId().toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish TicketEscalatedEvent", e);
        }
    }
}

@Data
class TicketCreationRequest {
    private Long customerId;
    private String subject;
    private String description;
    private String channel; // WEB, EMAIL, CHAT, PHONE
    private String channelId;
}

@Data
class TicketResponse {
    private Long ticketId;
    private String status;
    private String category;

    public TicketResponse(Long ticketId, String status, String category) {
        this.ticketId = ticketId;
        this.status = status;
        this.category = category;
    }
}
```

### 9.2 TicketRouter (Routing & Assignment)

```java
@Service
@RequiredArgsConstructor
public class TicketRouter {
    private final TicketRepository ticketRepository;
    private final AgentWorkloadRepository agentWorkloadRepository;
    private final RedisTemplate<String, Object> redisTemplate;
    private final TicketService ticketService;
    private static final Logger logger = LoggerFactory.getLogger(TicketRouter.class);

    public void routeTicket(Ticket ticket) {
        try {
            // Step 1: Determine team queue based on category
            String queueKey = "team:" + ticket.getCategory().name().toLowerCase() + ":queue";

            // Step 2: Find best available agent
            Long agentId = findAvailableAgent(ticket.getCategory());

            if (agentId == null) {
                logger.warn("No available agent for category {}", ticket.getCategory());
                // Add to queue, will be assigned when agent becomes available
                addToQueue(queueKey, ticket.getId(), ticket.getPriority());
                return;
            }

            // Step 3: Assign ticket to agent
            ticketService.assignTicket(ticket.getId(), agentId);

        } catch (Exception e) {
            logger.error("Failed to route ticket {}", ticket.getId(), e);
        }
    }

    private Long findAvailableAgent(TicketCategory category) {
        // Query agents for this category with lowest workload
        List<AgentWorkload> agents = agentWorkloadRepository.findByTeamAndOnlineTrueOrderByOpenTickets(
            category.name().toLowerCase());

        if (agents.isEmpty()) {
            return null;
        }

        // Find agent with capacity
        for (AgentWorkload agent : agents) {
            if (agent.getOpenTickets() < 20) { // Max 20 open tickets per agent
                return agent.getAgentId();
            }
        }

        return null; // All agents at capacity
    }

    private void addToQueue(String queueKey, Long ticketId, TicketPriority priority) {
        // Add to Redis sorted set (lower score = higher priority, newer = higher priority)
        double score = calculateQueueScore(priority);
        redisTemplate.opsForZSet().add(queueKey, ticketId.toString(), score);

        logger.info("Ticket {} added to queue {}", ticketId, queueKey);
    }

    private double calculateQueueScore(TicketPriority priority) {
        // Score = priority weight + timestamp (lower score = higher priority)
        int priorityWeight = switch (priority) {
            case URGENT -> 0;
            case HIGH -> 1000;
            case NORMAL -> 5000;
            case LOW -> 10000;
        };
        return priorityWeight + System.currentTimeMillis() / 1000;
    }

    @Scheduled(fixedRate = 60000) // Every minute
    public void assignQueuedTickets() {
        // Periodically try to assign queued tickets to available agents
        List<TicketCategory> categories = Arrays.asList(TicketCategory.values());

        for (TicketCategory category : categories) {
            String queueKey = "team:" + category.name().toLowerCase() + ":queue";
            Set<Object> queuedTickets = redisTemplate.opsForZSet().range(queueKey, 0, -1);

            for (Object ticketIdObj : queuedTickets) {
                Long ticketId = Long.parseLong(ticketIdObj.toString());
                Long agentId = findAvailableAgent(category);

                if (agentId != null) {
                    try {
                        ticketService.assignTicket(ticketId, agentId);
                        redisTemplate.opsForZSet().remove(queueKey, ticketId.toString());
                        logger.info("Assigned queued ticket {} to agent {}", ticketId, agentId);
                    } catch (Exception e) {
                        logger.error("Failed to assign queued ticket {}", ticketId, e);
                    }
                }
            }
        }
    }
}
```

### 9.3 SLATrackerService

```java
@Service
@RequiredArgsConstructor
public class SLATrackerService {
    private final TicketSLARepository slaRepository;
    private final TicketService ticketService;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;
    private static final Logger logger = LoggerFactory.getLogger(SLATrackerService.class);

    @Scheduled(fixedRate = 900000) // Every 15 minutes
    public void checkSLABreaches() {
        logger.info("Checking for SLA breaches");

        LocalDateTime now = LocalDateTime.now();
        List<TicketSLA> slas = slaRepository.findAll(); // In production, query only active SLAs

        for (TicketSLA sla : slas) {
            // Check response SLA
            if (!sla.isResponseSLABreached() && now.isAfter(sla.getResponseSLADeadline())) {
                sla.setResponseSLABreached(true);
                slaRepository.save(sla);

                // Escalate ticket
                ticketService.escalateTicket(sla.getTicketId(), "SLA_RESPONSE_BREACH");
                publishSLABreachEvent(sla.getTicketId(), "RESPONSE_SLA");

                logger.warn("Response SLA breached for ticket {}", sla.getTicketId());
            }

            // Check resolution SLA
            if (!sla.isResolutionSLABreached() && now.isAfter(sla.getResolutionSLADeadline())) {
                sla.setResolutionSLABreached(true);
                slaRepository.save(sla);

                publishSLABreachEvent(sla.getTicketId(), "RESOLUTION_SLA");

                logger.warn("Resolution SLA breached for ticket {}", sla.getTicketId());
            }
        }
    }

    private void publishSLABreachEvent(Long ticketId, String slaBreach) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "SLA_BREACHED");
            event.put("ticket_id", ticketId);
            event.put("sla_type", slaBreach);
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("sla-events", ticketId.toString(), eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish SLABreachEvent", e);
        }
    }

    public SLAMetrics getSLAMetrics(String teamName, LocalDate startDate, LocalDate endDate) {
        LocalDateTime startDateTime = startDate.atStartOfDay();
        LocalDateTime endDateTime = endDate.atTime(23, 59, 59);

        List<TicketSLA> slas = slaRepository.findBetweenDates(startDateTime, endDateTime);

        long totalTickets = slas.size();
        long metResponseSLA = slas.stream()
            .filter(s -> !s.isResponseSLABreached())
            .count();
        long metResolutionSLA = slas.stream()
            .filter(s -> !s.isResolutionSLABreached())
            .count();

        SLAMetrics metrics = new SLAMetrics();
        metrics.setTeamName(teamName);
        metrics.setStartDate(startDate);
        metrics.setEndDate(endDate);
        metrics.setTotalTickets(totalTickets);
        metrics.setResponseSLAMet(totalTickets > 0 ? (metResponseSLA * 100) / totalTickets : 0);
        metrics.setResolutionSLAMet(totalTickets > 0 ? (metResolutionSLA * 100) / totalTickets : 0);

        return metrics;
    }
}

@Data
class SLAMetrics {
    private String teamName;
    private LocalDate startDate;
    private LocalDate endDate;
    private long totalTickets;
    private long responseSLAMet; // %
    private long resolutionSLAMet; // %
}
```

### 9.4 ContextAggregationService (Agent Dashboard)

```java
@Service
@RequiredArgsConstructor
public class ContextAggregationService {
    private final TicketRepository ticketRepository;
    private final CustomerServiceClient customerService;
    private final OrderServiceClient orderService;
    private final KnowledgeBaseRepository kbRepository;
    private final RedisTemplate<String, String> redisTemplate;
    private final ObjectMapper objectMapper;
    private static final Logger logger = LoggerFactory.getLogger(ContextAggregationService.class);

    public CustomerContextResponse getCustomerContext(Long ticketId) {
        Ticket ticket = ticketRepository.findById(ticketId)
            .orElseThrow(() -> new TicketNotFoundException(ticketId));

        Long customerId = ticket.getCustomerId();

        // Try cache first
        String cacheKey = "customer:context:" + customerId;
        String cached = redisTemplate.opsForValue().get(cacheKey);
        if (cached != null) {
            try {
                return objectMapper.readValue(cached, CustomerContextResponse.class);
            } catch (JsonProcessingException e) {
                logger.debug("Cache parse error, fetching fresh", e);
            }
        }

        // Parallel fetch from multiple services
        CustomerContextResponse context = new CustomerContextResponse();
        context.setCustomerId(customerId);

        try {
            CompletableFuture<CustomerProfile> customerFuture = CompletableFuture.supplyAsync(
                () -> customerService.getCustomerProfile(customerId));

            CompletableFuture<List<Order>> ordersFuture = CompletableFuture.supplyAsync(
                () -> orderService.getCustomerOrders(customerId, 10));

            CompletableFuture<List<Ticket>> previousTicketsFuture = CompletableFuture.supplyAsync(
                () -> ticketRepository.findByCustomerIdOrderByCreatedAtDesc(customerId, 5));

            // Wait for all
            CompletableFuture.allOf(customerFuture, ordersFuture, previousTicketsFuture).join();

            context.setCustomerProfile(customerFuture.join());
            context.setOrders(ordersFuture.join());
            context.setPreviousTickets(previousTicketsFuture.join());

            // Cache result
            redisTemplate.opsForValue().set(cacheKey,
                objectMapper.writeValueAsString(context),
                Duration.ofMinutes(5));

            logger.info("Aggregated context for customer {}", customerId);
            return context;

        } catch (Exception e) {
            logger.error("Failed to aggregate context for customer {}", customerId, e);
            throw new RuntimeException("Context aggregation failed: " + e.getMessage());
        }
    }

    public List<KBArticle> getSuggestedArticles(Long ticketId, int limit) {
        Ticket ticket = ticketRepository.findById(ticketId)
            .orElseThrow(() -> new TicketNotFoundException(ticketId));

        // Extract keywords from ticket
        Set<String> keywords = extractKeywords(ticket.getSubject(), ticket.getDescription());

        // Search KB for matching articles
        List<KBArticle> articles = kbRepository.findByKeywords(keywords, limit);

        logger.info("Suggested {} articles for ticket {}", articles.size(), ticketId);
        return articles;
    }

    private Set<String> extractKeywords(String subject, String description) {
        String combined = (subject + " " + description).toLowerCase();
        // Simple tokenization (in production, use NLP library)
        return Arrays.stream(combined.split("\\s+"))
            .filter(word -> word.length() > 3) // Filter short words
            .limit(10)
            .collect(Collectors.toSet());
    }
}

@Data
class CustomerContextResponse {
    private Long customerId;
    private CustomerProfile customerProfile;
    private List<Order> orders;
    private List<Ticket> previousTickets;
}

@Data
class CustomerProfile {
    private Long id;
    private String email;
    private String name;
    private String phone;
    private boolean vip;
    private int totalOrders;
}

@Data
class Order {
    private Long id;
    private String status;
    private BigDecimal total;
    private LocalDateTime createdAt;
}
```

### 9.5 EmailChannelConsumer

```java
@Component
@RequiredArgsConstructor
public class EmailChannelConsumer {
    private final TicketService ticketService;
    private final TicketRepository ticketRepository;
    private final EmailThreadRepository emailThreadRepository;
    private final JavaMailSender mailSender;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;
    private static final Logger logger = LoggerFactory.getLogger(EmailChannelConsumer.class);

    @Scheduled(fixedRate = 900000) // Every 15 minutes
    public void pollEmails() {
        logger.info("Polling email inbox");

        try {
            // Connect to IMAP
            Properties props = new Properties();
            props.put("mail.imap.host", "imap.gmail.com");
            props.put("mail.imap.port", "993");
            props.put("mail.imap.starttls.enable", "true");
            props.put("mail.imap.starttls.required", "true");

            Session session = Session.getInstance(props);
            Store store = session.getStore("imaps");
            store.connect("support@company.com", "password");

            Folder inbox = store.getFolder("INBOX");
            inbox.open(Folder.READ_WRITE);

            Message[] messages = inbox.search(new FlagTerm(new Flags(Flags.Flag.SEEN), false));

            for (Message message : messages) {
                processEmailMessage(message);
                message.setFlag(Flags.Flag.SEEN, true);
            }

            inbox.close();
            store.close();

        } catch (Exception e) {
            logger.error("Failed to poll emails", e);
        }
    }

    private void processEmailMessage(Message message) throws Exception {
        String messageId = message.getHeader("Message-ID")[0];
        String subject = message.getSubject();
        String from = ((InternetAddress) message.getFrom()[0]).getAddress();
        String body = getTextFromMessage(message);

        logger.info("Processing email from {} with subject {}", from, subject);

        // Check if this is a reply to existing ticket
        EmailThread existingThread = emailThreadRepository.findByMessageId(messageId);
        if (existingThread != null) {
            // Append to existing ticket
            Ticket ticket = ticketRepository.findById(existingThread.getTicketId())
                .orElseThrow();

            TicketService.MessageCreationRequest request = new TicketService.MessageCreationRequest();
            request.setSenderId(extractCustomerIdFromEmail(from));
            request.setSenderType(SenderType.CUSTOMER);
            request.setMessageBody(body);

            ticketService.addMessage(ticket.getId(), request);

            logger.info("Appended email to existing ticket {}", ticket.getId());
        } else {
            // Create new ticket
            TicketService.TicketCreationRequest request = new TicketService.TicketCreationRequest();
            request.setCustomerId(extractCustomerIdFromEmail(from));
            request.setSubject(subject);
            request.setDescription(body);
            request.setChannel("EMAIL");
            request.setChannelId(messageId);

            TicketResponse response = ticketService.createTicket(request);

            // Log email thread mapping
            EmailThread thread = new EmailThread();
            thread.setTicketId(response.getTicketId());
            thread.setMessageId(messageId);
            emailThreadRepository.save(thread);

            logger.info("Created new ticket {} from email", response.getTicketId());
        }

        // Publish event
        publishEmailReceivedEvent(messageId, subject, from);
    }

    private String getTextFromMessage(Message message) throws Exception {
        if (message.isMimeType("text/plain")) {
            return (String) message.getContent();
        } else if (message.isMimeType("text/html")) {
            // Strip HTML tags (simplified)
            String html = (String) message.getContent();
            return html.replaceAll("<[^>]*>", "");
        }
        return "";
    }

    private Long extractCustomerIdFromEmail(String email) {
        // Query customer by email; in production, may need to handle unregistered emails
        return 123L; // Stub
    }

    private void publishEmailReceivedEvent(String messageId, String subject, String from) {
        try {
            Map<String, Object> event = new HashMap<>();
            event.put("event_id", UUID.randomUUID().toString());
            event.put("event_type", "EMAIL_RECEIVED");
            event.put("message_id", messageId);
            event.put("subject", subject);
            event.put("from", from);
            event.put("timestamp", LocalDateTime.now().toString());

            String eventJson = objectMapper.writeValueAsString(event);
            kafkaTemplate.send("email-messages", messageId, eventJson);
        } catch (JsonProcessingException e) {
            logger.error("Failed to publish EmailReceivedEvent", e);
        }
    }
}
```

---

## 10. Failure Scenarios & Mitigations

| Scenario | Impact | Mitigation |
|----------|--------|-----------|
| **Email Polling Fails** | Customer emails not received | Queue to DLQ, retry next cycle (every 15 min); alert ops |
| **IMAP Server Down** | Email intake blocked | Graceful degrade, accept form submissions, backlog to Kafka |
| **Agent Dashboard Hangs** | Agents can't see customer context | Cache context in Redis, fallback to basic ticket view |
| **SLA Check Miss** | Breached SLAs not escalated | Double-check on agent view, manual escalation queue |
| **Kafka Consumer Lag** | Delayed ticket routing | Monitor lag, autoscale consumers, alert if lag > 5K |
| **File Upload Large** | Storage explosion | Limit uploads to 100MB, scan for malware, compress PDFs |
| **Email Reply Rate Limit** | Customer not notified | Queue email in DLQ, retry with exponential backoff |
| **Chat Session Timeout** | Real-time messages lost | Persist all chat messages to MongoDB before WebSocket send |

---

## 11. Scaling Strategy

### Horizontal Scaling
- **Ticket Service**: 10 instances (stateless)
- **Routing Engine**: 5 instances (load balancing)
- **SLA Tracker**: 2 instances (single responsible for scheduling)
- **Email Poller**: 3 instances (distributed IMAP folders)
- **Agent Dashboard**: 8 instances (stateless)

### Database Scaling
- **PostgreSQL**: Read replicas for agent dashboard queries, connection pooling (50 connections/instance)
- **Partitioning**: Tickets partitioned by date (monthly), messages by ticket_id
- **Indexing**: idx_status, idx_category, idx_assigned_to, idx_created_at

### Cache Strategy
- **Redis**: Ticket cache TTL 24h, customer context TTL 5 min
- **Cluster mode**: 6+ nodes, auto-failover

### Elasticsearch Scaling
- **KB articles**: 3-node cluster, daily snapshots, alias rollover

---

## 12. Monitoring & Observability

### Key Metrics
- **Ticket Volume**: Created/day, resolved/day, closed/day
- **Response Time**: Time to first response (SLA), P50/P99
- **Resolution Time**: Time to resolution, by category and agent
- **Queue Health**: Queue depth, age of oldest ticket
- **Agent Performance**: Tickets/agent/day, resolution rate, CSAT
- **Channel Distribution**: % tickets by channel (web/email/chat/phone)
- **SLA Compliance**: % met response SLA, % met resolution SLA

### Sample Alert Rules
```yaml
alerts:
  - name: HighTicketBacklog
    condition: ticket_queue_size > 500
    severity: warning

  - name: SLABreach
    condition: sla_breached_tickets_total > 100
    severity: critical

  - name: EmailPollingFailed
    condition: email_polling_errors_total > 10
    severity: warning

  - name: AgentResponseLatency
    condition: histogram_quantile(0.99, agent_response_latency) > 60000
    severity: warning
```

---

## 13. Summary Cheat Sheet

```
TICKET ROUTING
1. Category detected (ORDERS, RETURNS, TECHNICAL, BILLING)
2. Team queue determined by category
3. Available agent found by workload (load balancing)
4. Ticket assigned to agent, removed from queue
5. If no agent available, ticket queued in Redis sorted set

MULTI-CHANNEL INTAKE
• Email: IMAP poller every 15 min, dedup by thread ID
• Chat: WebSocket real-time, persisted to PostgreSQL
• Web: Form submission, Kafka event
• Phone: Webhook from call center
All → Kafka "ticket-intake" → TicketConsumer → unified ticket

SLA TRACKING
• Response SLA: 24 hours (first agent message)
• Resolution SLA: 72 hours (ticket closed)
• Background job checks every 15 min
• On breach: escalate ticket, notify customer, publish event

AGENT DASHBOARD
• Fetch customer profile, order history, previous tickets in parallel
• Context aggregation latency: 500ms–1s (CompletableFuture.allOf)
• Cache context 5 min in Redis
• Suggested KB articles: search by keywords from ticket text

ESCALATION
• Unresolved > 14 days → escalate to supervisor
• Breached SLA → auto-escalate
• Multiple escalations → escalation chain (audit trail)

ANALYTICS
• Ticket daily stats: by category, sentiment, resolution time
• Top issues: keyword extraction, product correlation
• SLA metrics: % met by team, avg response time
• Agent performance: tickets/day, resolution rate, CSAT
```

