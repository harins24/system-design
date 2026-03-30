# Designing a Social Media Feed

**Phase 7 — Tradeoffs + Interview Problems | Topic 6 of 8**

---

## Overview

The social media feed is the canonical system design question for senior engineers. It tests fan-out at scale, caching strategy, the celebrity problem, and how to balance consistency with performance. Every large social platform (Twitter/X, Instagram, Facebook) has wrestled with exactly these problems.

---

## Phase 1: Requirements

### Functional Requirements

```
Core features:
  1. User can create a post (text, image, video)
  2. User can follow other users
  3. User sees a feed of posts from people they follow
  4. Feed sorted by: recency (for this walkthrough)

Questions to ask:
  "Is the feed ranked (algorithmic) or chronological?"
  "Can users like, comment, share? Do these appear in feed?"
  "Are there Stories, Reels, or just posts?"
  "Can users see posts from users they don't follow (explore/trending)?"
  "What media types? Text only or also images/video?"

For this walkthrough:
  Chronological feed (no ML ranking — simplifies design)
  Likes and comments: yes (but not surfaced in others' feeds)
  Posts only (no Stories)
  Images supported, video out of scope
  Feed is only followed users' posts
```

### Non-Functional Requirements

```
Scale:
  500M DAU
  Average user: follows 200 people, posts once per week

  Post writes:
    500M users / 7 days = 71M posts/day = 820 posts/sec

  Feed reads:
    Each user opens app 5x/day, loads 20 posts
    500M × 5 = 2.5B feed reads/day = 29,000 reads/sec

  Read:write ratio = 29,000 : 820 ≈ 35:1
  Read-heavy → optimize for read path

Latency:
  Feed load: <200ms P99
  Post creation: <500ms P99

Availability:
  99.99% for feed reads (users must be able to see content)
  Slightly lower for writes is acceptable

Consistency:
  User creates post → their followers see it within 30 seconds
  Eventual consistency acceptable
  User must see their own post immediately (read-your-writes)

Geography:
  Global: US, EU, Asia
```

---

## Phase 2: Capacity Estimation

```
Users and follows:
  500M users, average 200 follows each
  Total follow relationships: 500M × 200 = 100B follow pairs
  Storage: 100B × 16 bytes (followerId + followeeId) = 1.6 TB

Posts:
  820 posts/sec, average post size:
    Text: 500 bytes
    Image metadata: 200 bytes (actual image in object storage)
    Total: ~700 bytes per post record

  Storage: 820/sec × 700 bytes = 574 KB/sec
  1 year: ~18 TB of post metadata
  → Sharded PostgreSQL or DynamoDB

Images:
  50% of posts have images, average 2MB compressed
  820 × 0.5 × 2MB = 820 MB/sec of image uploads
  1 year: ~25 PB of images
  → S3 + CloudFront CDN (not served from app servers)

Feed cache:
  Active users (open app daily): 100M
  Each feed: 200 recent posts × 8 bytes (post ID) = 1.6 KB
  100M × 1.6 KB = 160 GB
  → Fits in Redis cluster (~10 nodes × 32GB)

Read QPS:
  29,000 feed reads/sec
  With Redis cache (95% hit rate): only 1,450/sec hit DB
  → Very manageable
```

---

## Phase 3: High-Level Design

### Key APIs

```
POST /api/posts
  Body: {text, imageUrl?, visibility: PUBLIC}
  Response: {postId, createdAt}

GET /api/feed?userId=X&cursor=Y&limit=20
  Response: {posts: [...], nextCursor: Z}
  cursor: opaque token encoding last seen timestamp + postId

POST /api/follow
  Body: {targetUserId}
  Response: {status: "FOLLOWING"}

GET /api/users/{userId}/posts?cursor=Y&limit=20
  Response: {posts: [...], nextCursor: Z}
```

### System Architecture

```
Write path (post creation):
[Client]
  → [API Gateway (auth, rate limit)]
  → [Post Service]
      → [Post DB (PostgreSQL, sharded by userId)]
      → [Object Storage (S3)] ← for image upload (presigned URL)
      → [Kafka: post-created]
      → [Fan-out Service] ← async, reads from Kafka
          → [Follow DB] "who follows this user?"
          → [Feed Cache (Redis)] ← update followers' feeds
          → [Feed DB] ← for cold/inactive users

Read path (load feed):
[Client]
  → [API Gateway]
  → [Feed Service]
      → [Feed Cache (Redis)] ← check user's precomputed feed
        (cache hit 95%): return feed
        (cache miss 5%):
            → [Feed DB] ← fetch stored feed or rebuild
                → [Post DB] ← fetch actual post details
            → [Feed Cache] ← populate cache
      → [Post Service] ← enrich post IDs with full post data
      → return enriched feed to client
```

---

## Phase 4: Deep Dives

### Deep Dive 1: Fan-Out Strategies

This is the central design challenge. When a user posts, how do followers' feeds get updated?

**Approach 1: Fan-Out on Write (Push Model)**
```
When user A posts:
  Fetch all followers of A (say 1,000 followers)
  For each follower: add post ID to their feed cache in Redis

  Redis:
    feed:{followerId} → sorted set of postIds, scored by timestamp

  Feed read:
    Get top 20 from sorted set → instant response

  ZADD feed:{followerId} timestamp postId  (for each follower)
  ZREMRANGEBYRANK feed:{followerId} 0 -501  (keep only top 500)
```

```java
@KafkaListener(topics = "post-created", groupId = "fan-out-service")
public void fanOut(PostCreatedEvent event) {
    String authorId = event.getAuthorId();
    String postId = event.getPostId();
    long timestamp = event.getTimestamp();

    // Get all followers (paginated for large follow counts)
    List<String> followers = followService.getFollowers(authorId);

    // Batch Redis writes for efficiency
    redis.executePipelined((RedisCallback<?>) connection -> {
        followers.forEach(followerId -> {
            String feedKey = "feed:" + followerId;
            connection.zAdd(feedKey.getBytes(), timestamp, postId.getBytes());
            // Trim feed to max 500 posts (memory control)
            connection.zRemRangeByRank(feedKey.getBytes(), 0, -501);
        });
        return null;
    });
}
```

```
✅ Feed reads are instant (O(1) sorted set lookup)
✅ Read path simple: just fetch sorted set
✅ Feed ready before user opens app

❌ Write amplification: 1 post × 1M followers = 1M Redis writes
❌ Celebrity problem: Kylie Jenner (400M followers) posts
   → 400M Redis writes → system overwhelmed
❌ Wasted work: inactive users get feed updates they never read
```

**Approach 2: Fan-Out on Read (Pull Model)**
```
When user A posts:
  Just write post to Post DB
  No fan-out

Feed read for user B:
  Fetch list of B's followees
  Query Post DB: "most recent posts from each followee"
  Merge and sort results
  Return top 20

  SELECT * FROM posts
  WHERE user_id IN (followee_1, followee_2, ..., followee_200)
  ORDER BY created_at DESC
  LIMIT 20
```

```
✅ No write amplification (write once, read on demand)
✅ Always fresh (no staleness)
✅ Celebrity posts are free to write

❌ Read is expensive: scatter-gather across 200 followees
❌ With 200 followees × DB latency = slow if not cached
❌ Hot users' post tables become hot spots under read load
❌ Not viable at 29,000 feed reads/sec without heavy caching
```

**Approach 3: Hybrid (Production Solution)**
```
The real solution: split users into two categories

Regular users (< 10,000 followers):
  Fan-out on write: precompute their followers' feeds
  Followers get instant feed reads from Redis cache

Celebrities (> 10,000 followers):
  No fan-out: their posts NOT pushed to follower caches

At feed read time:
  1. Fetch user's precomputed feed from Redis (regular followees)
  2. Fetch recent posts from each celebrity followee directly
  3. Merge and sort both sets
  4. Return top 20

Celebrity post fetch is fast:
  Each celebrity's recent posts cached separately:
  recent_posts:{celebrityId} → top 100 posts (sorted set)
  Reading 5 celebrities = 5 Redis lookups → still fast

  ZADD recent_posts:{authorId} timestamp postId
  (every post also added to author's own recent posts key)
```

```java
@Service
public class FeedService {

    private static final int CELEBRITY_THRESHOLD = 10_000;

    public Feed getFeed(String userId, String cursor, int limit) {

        // 1. Get user's precomputed feed (regular followees)
        List<String> precomputedPostIds = redis.opsForZSet()
            .reverseRangeByScore("feed:" + userId,
                Double.NEGATIVE_INFINITY, Double.POSITIVE_INFINITY,
                0, 100);

        // 2. Get celebrity followees
        List<String> celebrities = followService
            .getCelebrityFollowees(userId, CELEBRITY_THRESHOLD);

        // 3. Fetch recent posts for each celebrity
        List<String> celebrityPostIds = new ArrayList<>();
        if (!celebrities.isEmpty()) {
            redis.executePipelined((RedisCallback<?>) connection -> {
                celebrities.forEach(celebId ->
                    connection.zRevRange(
                        ("recent_posts:" + celebId).getBytes(), 0, 19));
                return null;
            }).forEach(result ->
                celebrityPostIds.addAll((List<String>) result));
        }

        // 4. Merge, sort, deduplicate, paginate
        List<String> allPostIds = Stream.concat(
            precomputedPostIds.stream(),
            celebrityPostIds.stream())
            .distinct()
            .sorted(this::byTimestampDesc)
            .limit(limit)
            .collect(toList());

        // 5. Fetch full post details (batch)
        List<Post> posts = postService.getPosts(allPostIds);

        return new Feed(posts, generateNextCursor(posts));
    }
}
```

---

### Deep Dive 2: Feed Storage in Redis

```
Data structure: sorted set per user
  Key:   feed:{userId}
  Score: post creation timestamp (Unix ms)
  Member: postId (UUID, 36 bytes)

Operations:
  Fan-out:  ZADD feed:{userId} {timestamp} {postId}   O(log N)
  Read:     ZREVRANGEBYSCORE feed:{userId} +inf -inf LIMIT 0 20
  Cleanup:  ZREMRANGEBYRANK feed:{userId} 0 -501     (keep 500)

Memory per user:
  500 posts × (36 bytes postId + 8 bytes score) = ~22KB per user
  Active users needing feed: 100M
  100M × 22KB = 2.2TB
  → Too large for single Redis, need cluster

  Solution: only cache active users
    Feed TTL: 24 hours
    Inactive user's feed expires → rebuilt on next login
    30-day inactive users: ~70% of users
    Only cache 30M active users: 30M × 22KB = 660GB → manageable
```

**Feed Pagination with Cursor:**
```
Problem: offset-based pagination breaks on new posts
  Page 1: LIMIT 20 OFFSET 0  → posts 1-20
  New post arrives
  Page 2: LIMIT 20 OFFSET 20 → includes post 20 again (shifted)

Cursor-based pagination:
  Cursor encodes: last seen timestamp + last seen postId

  Page 1: ZREVRANGEBYSCORE feed:{userId} +inf -inf LIMIT 0 20
          Returns: post A (ts=1000), post B (ts=999), ..., post T (ts=981)
          Next cursor: encode(ts=981, postId=T)

  Page 2: ZREVRANGEBYSCORE feed:{userId} 981 -inf LIMIT 0 21
          (fetch 21, skip first if postId=T)
          Returns: post T+1 (ts=980), ...

  New posts during pagination don't affect cursor position

// Cursor encoding
String encodeCursor(long timestamp, String postId) {
    return Base64.encode(timestamp + ":" + postId);
}

Pair<Long, String> decodeCursor(String cursor) {
    String decoded = new String(Base64.decode(cursor));
    String[] parts = decoded.split(":");
    return Pair.of(Long.parseLong(parts[0]), parts[1]);
}
```

---

### Deep Dive 3: Read-Your-Writes

```
Problem:
  User posts photo → writes to Post DB
  User immediately refreshes feed
  Feed Service reads from Redis cache
  New post not yet fan-outed to their own cache
  User doesn't see their own post!

Solutions:

Option 1: Write to own feed synchronously
  Post Service: after writing to DB,
  synchronously add to author's own feed cache
  (not async fan-out — immediate)

  ✅ Author always sees own post
  ❌ Adds Redis write to critical post creation path

Option 2: Read-your-writes via author's own post list
  Feed Service: before returning feed, check if user has
  recent posts not yet in feed cache

  // At feed read time
  long latestFeedTimestamp = getLatestTimestampInFeed(userId);
  List<Post> ownRecentPosts = postService.getUserPostsSince(
      userId, latestFeedTimestamp);
  // Prepend own recent posts to feed

  ✅ Guaranteed to see own posts
  ❌ Extra DB lookup for recent own posts

Option 3: Short delay + client optimistic update
  Client shows post immediately after creation (optimistic)
  Server fan-out happens async
  Within 30s, post appears in cache for everyone

  Most social apps do this (post appears instantly on your screen,
  shows in others' feeds within seconds)
```

---

### Deep Dive 4: Post Database Design

```
Access patterns:
  Write: INSERT new post by userId
  Read:  SELECT posts WHERE userId = X ORDER BY createdAt DESC
  Read:  SELECT post by postId (for enrichment)

No cross-user joins in hot path → good sharding candidate

Shard by userId:
  All posts for a user → same shard
  Query "posts by userId" → single shard (no scatter-gather)
  Cross-shard queries: rare (admin, analytics → use separate pipeline)

Schema:
  CREATE TABLE posts (
      post_id       UUID        PRIMARY KEY,
      user_id       UUID        NOT NULL,
      text          TEXT,
      image_url     TEXT,
      created_at    TIMESTAMP   DEFAULT NOW(),
      like_count    INT         DEFAULT 0,
      comment_count INT         DEFAULT 0
  );

  CREATE INDEX idx_user_created ON posts(user_id, created_at DESC);
  -- Covers query: WHERE user_id = X ORDER BY created_at DESC

Follow relationships:
  followers table:
    follower_id UUID
    followee_id UUID
    created_at  TIMESTAMP
    PRIMARY KEY (follower_id, followee_id)

  Indexes:
    (follower_id): "who does user X follow?" → for feed building
    (followee_id): "who follows user X?" → for fan-out

  At 100B follow pairs: 1.6 TB
  Shard by followee_id for fan-out path
  Separate read replica for follower_id queries
```

---

### Deep Dive 5: Global Architecture

```
Multi-region deployment:

US Region:
  [CDN PoP] → [US Load Balancer] → [US API Cluster]
  [US Post DB (primary)]
  [US Feed Cache (Redis)]
  [US Follow DB]

EU Region:
  [CDN PoP] → [EU Load Balancer] → [EU API Cluster]
  [EU Post DB (replica of US primary)]
  [EU Feed Cache (Redis)]
  [EU Follow DB (replica)]

Cross-region event replication:
  US Post DB → Kafka → EU Post DB replica
  Lag: ~50-100ms (acceptable for social feed)
  EU users see US users' posts within 1-2 seconds

User routing:
  Route 53 latency-based → nearest region
  US users → US cluster (<20ms)
  EU users → EU cluster (<20ms)

  GDPR: EU user data stays in EU
        User marked EU → writes go to EU primary
```

---

## Phase 5: Tradeoffs

```
Fan-out approach:
  Hybrid: regular users = fan-out on write, celebrities = on read
  Trade-off: complexity (need to classify users, merge at read time)
  Accepted: necessary to avoid both write amplification and slow reads

Feed staleness:
  Followers see new posts within 30 seconds (fan-out is async)
  Trade-off: eventual consistency, not immediate
  Accepted: standard expectation for social feeds

Redis as feed store:
  Trade-off: additional complexity and cost vs pure DB approach
  Accepted: 29K reads/sec cannot be served from DB alone

Memory optimization:
  Only cache active users (last 30 days)
  Trade-off: cold start for returning users (rebuild feed from DB)
  Accepted: ~500ms one-time cost for cold users vs perpetual memory waste

Celebrity threshold:
  10K followers is arbitrary — needs tuning
  Trade-off: too low = too many celebrities, slow reads
             too high = write amplification for large accounts
  In production: monitor and adjust based on performance data
```

---

## Key Takeaways

```
Social feed is the canonical fan-out problem.

Core tension:
  Write optimization (fan-out on read): expensive reads
  Read optimization (fan-out on write): expensive writes, celebrity problem
  Solution: hybrid based on follower count

Fan-out on write (regular users):
  Post created → push to all follower feed caches
  Feed read: instant Redis sorted set lookup
  Problem: 1M followers × 1 write = write storm

Fan-out on read (celebrities):
  Post stored once, read from author's recent posts cache
  Merged at read time with precomputed feed
  Handles unlimited followers with no write amplification

Feed cache (Redis sorted set):
  Score = timestamp, Member = postId
  ZADD, ZREVRANGEBYSCORE
  TTL-based eviction for inactive users
  Only store postIds (not full posts) to minimize memory

Pagination:
  Cursor-based (timestamp + postId)
  Handles new posts during pagination without duplication

Read-your-writes:
  Author sees own post immediately
  Implementation: write to own feed cache synchronously during post creation

Database:
  Posts sharded by userId (single-shard query for user's posts)
  Follows sharded by followee_id for fan-out
  Index on (userId, createdAt DESC) for efficient timeline query

Scale numbers:
  500M DAU, 820 posts/sec, 29K feed reads/sec
  95% cache hit rate → DB sees ~1,450 reads/sec
  Fan-out workers: ~820 posts/sec × 1K followers = 820K Redis ops/sec
  Redis cluster easily handles 1M+ ops/sec
```
