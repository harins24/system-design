# How to Approach System Design Interviews

**Phase 7 — Tradeoffs + Interview Problems | Topic 1**

---

## What Interviewers Are Actually Evaluating

Before the framework, understand what's being measured. Interviewers are not looking for the "right answer." They're evaluating:

```
1. Structured thinking under ambiguity
   Can you take a vague problem and systematically clarify it?
   Do you work through complexity in an organized way?

2. Engineering judgment
   Do you know WHEN to use each tool?
   Do you justify choices with tradeoffs, not just buzzwords?

3. Communication
   Can you explain complex ideas clearly?
   Do you drive the conversation or wait for hints?

4. Experience signals
   Do your examples feel real or textbook?
   Can you speak to failure modes from experience?

5. Depth vs breadth balance
   Do you know when to go deep vs stay high-level?
   Can you answer follow-up probes without collapsing?

What kills most candidates:
  → Jumping to solutions before understanding the problem
  → Silence (not thinking out loud)
  → "It depends" without saying what it depends on
  → Listing technologies without justifying choices
  → Getting deep into one area, ignoring the rest
  → Never acknowledging tradeoffs (no solution is perfect)
```

---

## The Framework — Five Phases

```
Phase 1: Requirements (5-7 minutes)
Phase 2: Capacity Estimation (3-5 minutes)
Phase 3: High-Level Design (10-15 minutes)
Phase 4: Deep Dives (15-20 minutes)
Phase 5: Tradeoffs and Wrap-Up (3-5 minutes)

Total: ~45 minutes

Time management is critical:
  Most candidates spend 30 minutes on Phase 1-2 (too long)
  or rush Phase 4 (not enough depth)
  Practice each phase independently
```

---

## Phase 1: Requirements Clarification

Never start designing before you understand the problem. This is where most candidates fail.

### Functional Requirements — What does the system do?

```
Ask yourself: what are the core user-facing features?

Questions to ask:
  "Who are the users? Consumers, businesses, internal teams?"
  "What are the core operations? Read, write, search?"
  "What does a user journey look like end to end?"
  "Are there any features I should explicitly exclude for scope?"

Example — "Design Instagram":
  You: "Should I focus on photo upload and feed, or also
        Stories, Reels, DMs?"
  Interviewer: "Focus on upload and feed"
  You: "Can users follow other users? Is the feed chronological
        or ranked?"
  Interviewer: "Yes to follow. Ranked by recency."

  Now you know: upload, follow, ranked feed. Scope is clear.
```

### Non-Functional Requirements — How well must it work?

```
These drive architecture more than functional requirements.
Always ask ALL of these:

Scale:
  "How many users? DAU (daily active users)?"
  "How many reads vs writes per second?"
  "Data volume — how much data are we storing?"

Latency:
  "What's acceptable latency for reads? Writes?"
  "Are there operations that must be <100ms?"

Availability:
  "What's the availability target? 99.9%? 99.99%?"
  "Is this acceptable to have planned maintenance windows?"

Consistency:
  "Is strong consistency required or is eventual consistency OK?"
  "Example: if I post a photo, must I see it immediately?"

Durability:
  "Is data loss ever acceptable?"
  "What's the RPO (recovery point objective)?"

Geographic:
  "Is this global or single region?"
  "Do users in different regions need low latency?"

Common answers and their implications:
  100M DAU → 1,000-2,000 requests/sec sustained
  1B DAU  → 10,000-20,000 requests/sec sustained
  "99.99% availability" → 52 minutes downtime/year max
  "Eventual consistency OK" → read replicas, cache, CDN viable
  "Strong consistency required" → single leader writes, careful design
```

### Setting Scope Explicitly

```
At the end of Phase 1, state your scope clearly:
  "OK so to summarize what we're building:

   Core features:
   - Users can upload photos
   - Users can follow other users
   - Users see a ranked feed of photos from followed users

   Scale:
   - 500M DAU, 100M photos uploaded per day
   - Read heavy: 10:1 read to write ratio
   - Feed must load in <200ms P99

   Out of scope today:
   - Stories, Reels, DMs
   - Notifications
   - Content moderation

   Does that match your expectations?"

Interviewer confirms → now you can design confidently
Interviewer corrects → you caught a misunderstanding early
```

---

## Phase 2: Capacity Estimation

Back-of-envelope math to inform architectural decisions.

### Why It Matters

```
Estimation reveals:
  - Which components will be bottlenecks
  - Whether you need caching / sharding
  - Storage requirements
  - Bandwidth requirements

Without estimation:
  You might design for 1,000 users when you need 1,000,000
  You might shard a DB that doesn't need it
  You miss that images need 100x more storage than metadata
```

### The Math — Reference Numbers

```
Memorize these:
  1 million users:     ~12 requests/sec (if 1 req/user/day)
  100M DAU:            ~1,200 req/sec average (2,000 peak)
  1B DAU:              ~12,000 req/sec average (20,000 peak)

  1 KB  = 1,000 bytes
  1 MB  = 1,000,000 bytes
  1 GB  = 10^9 bytes
  1 TB  = 10^12 bytes
  1 PB  = 10^15 bytes

  Character (UTF-8):   1-4 bytes
  Integer:             4 bytes
  Long:                8 bytes
  UUID:                36 bytes
  Average URL:         100 bytes
  Tweet:               280 bytes
  Small JSON object:   ~500 bytes
  Small image (thumb): ~50KB
  Profile photo:       ~200KB
  Instagram photo:     ~3MB
  1 minute HD video:   ~100MB

  Day   = 86,400 seconds ≈ 100,000 seconds (useful approximation)
  Month = 30 days = 2.5M seconds
  Year  = 3.15 × 10^7 seconds
```

### Worked Example — Instagram

```
Given: 500M DAU, 100M photo uploads/day

Write QPS:
  100M uploads/day ÷ 86,400 sec/day = ~1,160 uploads/sec
  Peak (2x average): ~2,400 uploads/sec

Read QPS:
  Each user views ~30 photos/day (browsing feed)
  500M × 30 = 15B photo views/day
  15B ÷ 86,400 = ~174,000 photo reads/sec
  Peak: ~350,000 reads/sec
  Read:write ratio = 150:1 → heavily read-optimized

Storage (photos):
  100M uploads/day × 3MB = 300TB/day raw
  After compression + thumbnail generation: ~150TB/day
  1 year: 150TB × 365 = ~55 PB
  → Need distributed object storage (S3, GCS)
  → CDN essential (can't serve 350K reads/sec from origin)

Metadata storage:
  Each photo: {photoId, userId, caption, timestamp, location, tags}
  ~500 bytes per photo
  100M photos/day × 500 bytes = 50GB/day metadata
  1 year: ~18TB metadata → fits in distributed DB with sharding

Bandwidth:
  Uploads: 2,400 × 3MB = 7.2 GB/s inbound
  Downloads: 174,000 × 200KB (thumbnail) = 34.8 GB/s outbound
  → CDN handles most outbound, origin sees fraction

Conclusions from estimation:
  ✅ Need CDN (350K reads/sec, 35GB/s outbound)
  ✅ Need object storage (55PB in a year)
  ✅ Need DB sharding for metadata (18TB/year)
  ✅ Read-heavy (150:1) → aggressive caching viable
  ✅ Upload path must handle 7.2 GB/s → async upload via S3 presigned URLs
```

---

## Phase 3: High-Level Design

Draw the system end-to-end before diving into any component.

### Start With the API

```
Define key APIs first:
  POST /photos
    Request: {caption, location, tags} + multipart image
    Response: {photoId, status: "PROCESSING"}

  GET /feed?userId=X&cursor=Y&limit=20
    Response: {photos: [...], nextCursor: Z}

  POST /follow
    Request: {targetUserId}
    Response: {status: "OK"}

Why start with API:
  Forces you to define input/output clearly
  Reveals implicit requirements ("cursor-based pagination needed")
  Interviewer can correct misunderstandings early
  API shapes data models and service boundaries
```

### Draw End-to-End Flow

```
[Client] → [CDN] → [API Gateway] → [Load Balancer]
                                        ↓
                              [Upload Service]
                                        ↓
                              [Object Storage (S3)]
                                        ↓
                              [Message Queue (Kafka)]
                                        ↓
                 ┌─────────────────────┴───────────────────┐
                 ↓                                         ↓
         [Photo Processor]                        [Feed Update Service]
    (resize, thumbnail,                         (update feed for followers)
     metadata extract)                                     ↓
                 ↓                                  [Feed Cache (Redis)]
         [Photo DB]
     (metadata storage)

Read path:
[Client] → [CDN] → (cached: return)
               → (miss) → [Feed Service]
                              → [Feed Cache (Redis)]
                              → (miss) → [Feed DB]
```

### Iterating the Design

```
Walk through the request flow out loud:
  "A user uploads a photo.
   Request hits the API Gateway which validates the JWT.
   Upload service generates a pre-signed S3 URL.
   Client uploads directly to S3 — this offloads transfer
   from our servers.
   S3 triggers an event → Kafka.
   Photo Processor consumes the event: creates thumbnails,
   extracts metadata, writes to Photo DB.
   Feed Update Service consumes the event: fetches the user's
   followers, adds the photo to each follower's feed cache.

   On read, a user requests their feed.
   Feed Service checks Redis — 95% hit rate for active users.
   Cache miss: fetch from Feed DB → populate cache → return."

Narrating the flow:
  Proves you understand data movement
  Lets interviewer follow along and correct mistakes
  Shows you're thinking about each component's role
```

---

## Phase 4: Deep Dives

The interviewer will guide you to components they want to explore. Know the common deep dive areas.

### Common Deep Dive Areas

```
Database:
  "How do you handle fan-out for celebrity users with 10M followers?"
  "How do you shard this database?"
  "What happens when a shard goes down?"

Caching:
  "How do you handle cache invalidation?"
  "What's your eviction policy?"
  "Cache stampede on popular content?"

Scale bottlenecks:
  "Your system has 150M DAU now, how does it scale to 1B?"
  "What's the single bottleneck in your design?"

Reliability:
  "What happens if the feed service goes down?"
  "How do you handle network partition between US and EU?"

Consistency:
  "User posts photo — when does their follower see it?"
  "Is eventual consistency acceptable for feeds?"
```

### How to Handle Deep Dives

```
Structure each deep dive:
  1. Acknowledge the problem clearly
     "Good question. The fan-out problem is real for celebrity accounts."

  2. Explain the challenge
     "If a celebrity has 100M followers, a single post would require
     100M writes to the feed cache — this is called 'fan-out on write'
     and at that scale it's unacceptable."

  3. Present options with tradeoffs
     "We have two approaches:
     Fan-out on write: write to all follower caches on post.
       Pro: reads are instant
       Con: writes are expensive for celebrities

     Fan-out on read: read and merge at query time.
       Pro: writes are cheap
       Con: reads are expensive (fetch posts from all followees)"

  4. State your recommendation
     "I'd use a hybrid approach: fan-out on write for regular users
     (<10K followers), fan-out on read for celebrities.
     At read time, merge the pre-built feed with a live fetch of
     celebrity posts. Celebrities are rare — maybe 0.01% of users.
     Most users benefit from instant reads, celebrity edge case handled."

  5. Acknowledge remaining issues
     "This adds complexity: need to classify users as celebrities,
     need efficient merge at read time. But it's the right tradeoff."
```

---

## Phase 5: Tradeoffs and Wrap-Up

```
Near the end, address what you didn't solve:
  "I want to flag a few tradeoffs in this design:

  1. Feed eventual consistency:
     A user posts and their followers may not see it for
     30 seconds if they're not in the active set.
     For a social feed, I believe this is acceptable.
     For financial data, it would not be.

  2. Celebrity fan-out:
     Our hybrid approach adds code complexity.
     The definition of 'celebrity' (>10K followers) is arbitrary
     and would need tuning in production.

  3. Storage cost:
     We're storing 55PB/year of photos.
     Cost optimization would involve tiered storage:
     hot (S3 Standard) → warm (S3 IA) → cold (S3 Glacier)
     based on access patterns.

  Is there any area you'd like me to go deeper on?"
```

---

## Anti-Patterns That Kill Candidates

```
1. The Silent Designer
   Problem: designs for 5 minutes without speaking
   Fix: narrate every thought, even uncertainty
   "I'm thinking about whether to use SQL or NoSQL here.
    The access pattern is key-value lookups by userId,
    which suggests NoSQL, but we also need..."

2. The Technology Enthusiast
   Problem: "I'd use Kafka, Flink, Elasticsearch, Redis,
             DynamoDB, GraphQL, and Kubernetes"
   Fix: justify every choice with the specific problem it solves
   "I'd use Kafka here specifically because we need fan-out
    to multiple consumers and durable message replay
    if the feed service falls behind."

3. The Premature Optimizer
   Problem: spends 20 minutes designing a sharding strategy
            for a system that could fit on one server
   Fix: estimate first, then decide if sharding is needed
   "With 10M DAU and a 10:1 read ratio, we're looking at
    1,000 reads/sec. A single PostgreSQL with read replicas
    handles this fine. I'd only shard at 10x growth."

4. The Perfectionist
   Problem: tries to solve every problem before moving forward
            never completes the design
   Fix: flag known issues, move forward
   "I'll note this cache invalidation race condition as
    a known issue and come back to it — let me first
    complete the overall design."

5. The Agreement Machine
   Problem: agrees with all interviewer suggestions
            even when they're wrong
   Fix: defend good decisions with data
   "I hear your suggestion to use SQL here, but given
    the 500M rows expected, the lack of relational
    queries, and the key-value access pattern,
    I'd still prefer DynamoDB. Here's why..."
```

---

## The Meta-Framework: Driving the Conversation

```
Every few minutes, orient the conversation:
  "I've finished the upload path. Before I go deeper on the
   feed generation, is there anything you want me to
   clarify about what I've covered so far?"

  "I see two areas to go deep: feed caching strategy and
   the fan-out problem. Which would you prefer to explore first?"

  "I'm about to make a key decision about the database.
   I'm choosing between PostgreSQL sharded by userId
   and DynamoDB. Let me explain both options..."

This shows:
  You're aware of time
  You're collaborative (not just monologuing)
  You know what the key decisions are
  You're giving the interviewer control
```

---

## Quick Reference — The 5-Phase Checklist

```
Phase 1: Requirements (5-7 min)
  □ Clarify users and use cases
  □ Ask about scale (DAU, read/write ratio)
  □ Ask about latency requirements
  □ Ask about consistency requirements
  □ Ask about availability target
  □ State scope explicitly, get confirmation

Phase 2: Estimation (3-5 min)
  □ Calculate write QPS
  □ Calculate read QPS (note read:write ratio)
  □ Calculate storage (data + media separately)
  □ Calculate bandwidth
  □ Identify what the bottleneck will be

Phase 3: High-Level Design (10-15 min)
  □ Define key APIs first
  □ Draw end-to-end system
  □ Walk through write path
  □ Walk through read path
  □ Label every component (no mystery boxes)

Phase 4: Deep Dives (15-20 min)
  □ Ask which area to explore
  □ Acknowledge the problem
  □ Present 2-3 options
  □ State recommendation with tradeoffs
  □ Acknowledge remaining issues

Phase 5: Tradeoffs (3-5 min)
  □ Flag top 2-3 known weaknesses
  □ Explain why you accepted each tradeoff
  □ Ask if interviewer wants to explore any area further
```

---

## Key Takeaways

```
Interviews evaluate: structured thinking, judgment, communication

Five phases:
  1. Requirements: clarify before designing
  2. Estimation: math reveals bottlenecks
  3. High-level: end-to-end before depth
  4. Deep dives: options → tradeoffs → recommendation
  5. Wrap-up: acknowledge weaknesses proactively

Non-functional requirements drive architecture:
  Scale → sharding, caching, CDN decisions
  Latency → async vs sync, caching aggressiveness
  Consistency → single leader vs eventual
  Availability → replication, failover design

Key habits:
  Think out loud — always
  Justify every technology choice
  Estimate before architecting
  State tradeoffs proactively
  Drive the conversation

Anti-patterns to avoid:
  Silence during design
  Technology laundry list without justification
  Premature optimization
  Never completing the design
  Caving on good decisions under pressure
```
