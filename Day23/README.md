# Day 23 - Full System Design: Twitter / X (News Feed & Timeline)

## Topic
**Design Twitter/X** — the canonical "hard" system design problem. Covers timeline generation (fan-out), the celebrity problem, tweet storage, and search.

---

## Why This Problem Matters
- It's the most frequently asked senior-level system design question.
- It tests the **fan-out problem** (push vs pull vs hybrid) — a concept unique to social media at scale.
- It combines every building block: sharding, caching, message queues, CDNs, real-time delivery.

---

## RESHADED Framework

### R — Requirements (5 min)

**Functional:**
1. Post a tweet (text, optionally images/video).
2. Follow/unfollow users.
3. Home timeline: see tweets from people you follow (most recent first).
4. User timeline: see a specific user's tweets.
5. Search tweets (by text, hashtag).
6. Retweet, like, reply (stretch).

**Non-functional:**
1. Timeline latency: < 200ms (p99).
2. High availability: 99.99%.
3. Eventually consistent (a tweet may take seconds to appear in all followers' timelines — acceptable).
4. 300M MAU, 500M tweets/day.

**Clarifying questions:**
- "Real-time timeline or can it be slightly delayed?" → Near real-time (seconds).
- "Media support?" → Yes, but focus on text; media is an extension.
- "Search?" → Yes, but as a separate subsystem.

### E — Estimation (3 min)

```
MAU: 300M
DAU: ~100M
Tweets/day: 500M
Tweets/sec (avg): 500M / 86400 ≈ 5,800/sec
Tweets/sec (peak): ~17,000/sec

Read:write ratio: 100:1 (timeline reads >> tweet writes)
Timeline reads/sec: 580,000/sec (avg), 1.7M/sec (peak)

Storage:
  Tweet: 300 bytes (text + metadata)
  Daily: 500M × 300B = 150GB/day
  5 years: 273TB (text only)
  With media: ×10 = ~2.7PB

Cache (Redis):
  Active timelines to cache: 100M (DAU)
  Timeline size: 100 tweets × 300B = 30KB
  Total cache: 100M × 30KB = 3TB
```

### S — Storage Schema (5 min)

**Polyglot persistence:**
```
PostgreSQL: users, follows (relational, ACID needed)
  users (id, username, email, created_at)
  follows (follower_id, followee_id, created_at)

Cassandra: tweets (high write throughput, time-series)
  tweets (user_id, tweet_id, content, created_at)
  Partition key: user_id (all tweets by a user on one partition)
  Clustering: created_at DESC (sorted by time)

Redis: timelines (pre-built, cached)
  timeline:user:{id} → list of tweet_ids (sorted by time)

S3 + CDN: media (images, videos)

Elasticsearch: tweet search index
  index: tweets (tweet_id, content, hashtags, user_id, created_at)
```

### H — High-Level Design (10 min)

```
┌─────────┐     ┌──────────┐
│  Client  │────→│ API Gateway│
└─────────┘     └─────┬────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ Tweet    │ │ Timeline │ │ Follow   │
   │ Service  │ │ Service  │ │ Service  │
   └────┬─────┘ └────┬─────┘ └────┬─────┘
        │            │            │
        ▼            ▼            ▼
   ┌──────────┐ ┌──────────┐ ┌──────────┐
   │Cassandra │ │  Redis   │ │PostgreSQL│
   │ (tweets) │ │(timelines│ │(users,   │
   │          │ │  cache)  │ │ follows) │
   └──────────┘ └──────────┘ └──────────┘
        │
        ▼
   ┌──────────┐
   │  Kafka   │ (tweet.published event)
   └────┬─────┘
        │
        ▼
   ┌──────────┐
   │ Fan-out   │ (push tweet to followers' timelines)
   │ Worker    │
   └──────────┘
```

### A — API Design (5 min)

```
POST /api/v1/tweets
  Body: { "content": "Hello world!", "media_ids": ["img_123"] }
  Response: 201 { "id": "twe_789", "content": "...", "created_at": "..." }

GET /api/v1/timeline/home
  → Returns home timeline (tweets from followed users)
  Query: ?cursor=abc&limit=20
  Response: { "data": [tweets], "pagination": {"next_cursor": "def"} }

GET /api/v1/timeline/user/{user_id}
  → Returns a specific user's tweets

POST /api/v1/follows
  Body: { "user_id": 123 }
  Response: 201 { "following": true }

DELETE /api/v1/follows/{user_id}
  Response: 204

GET /api/v1/search?q=hashtag&sort=recent
  Response: { "data": [tweets] }
```

### D — Detailed Design (15 min)

#### The Fan-Out Problem (CORE of the design)

When Alice tweets, how do her followers see it in their timeline?

**Option 1: Pull (Fan-out on Read)**
```
User opens timeline:
1. Get user's followee list (from PostgreSQL).
2. Query Cassandra for each followee's recent tweets.
3. Merge, sort by created_at, return top 20.

Problem: Following 1,000 people → 1,000 Cassandra queries. Slow.
```

**Option 2: Push (Fan-out on Write)**
```
Alice tweets:
1. Store tweet in Cassandra.
2. Get Alice's follower list.
3. Push tweet_id to each follower's timeline (Redis list: LPUSH timeline:user:{follower} {tweet_id}).
4. Trim timeline to latest 1,000 tweets (LTRIM).

User opens timeline:
1. GET timeline:user:{id} from Redis → list of tweet_ids.
2. Fetch tweet details from Cassandra.
3. Return.

Problem: If Alice has 50M followers → 50M Redis writes per tweet.
"Celebrity problem" — one tweet causes massive write amplification.
```

**Option 3: Hybrid (THE answer)**
```
For normal users (< 100K followers):
  → PUSH to followers' timelines (write-heavy, but manageable).

For celebrities (> 100K followers):
  → DON'T push. At read time, PULL celebrity tweets and merge with the pushed timeline.

User opens timeline:
1. Get pushed timeline from Redis (tweets from normal followees).
2. For each celebrity the user follows:
   → Fetch recent tweets directly from Cassandra.
3. Merge pushed + celebrity tweets, sort by time, return top 20.

This keeps write amplification manageable while maintaining fast reads.
```

#### Tweet ID Generation (Snowflake)
```
64-bit ID: [timestamp (41 bits)] [user_id/shard (10 bits)] [sequence (13 bits)]
  → Globally unique, sortable by time, encodes shard info.
  → No central ID generator needed.
```

#### Timeline Caching
```
Redis structure: timeline:user:{id} = [tweet_id, tweet_id, ...] (sorted list)

TTL: 24 hours (timelines older than 24h are rebuilt on demand)
Cache miss: rebuild from Cassandra (fan-out on read for the miss)
Cache hit: O(1) read from Redis → fetch tweet details → return
```

#### Search Subsystem
```
Tweet published → Kafka event → Elasticsearch indexer → index tweet
Search query → Elasticsearch → return matching tweet_ids → fetch details
```

### E — Edge Cases (5 min)
- **Celebrity tweets:** handled by hybrid fan-out.
- **Tweet deletion:** publish `tweet.deleted` event → remove from all cached timelines.
- **Stale timeline (replication lag):** acceptable — near real-time is fine.
- **Hot users (Justin Bieber follows you):** their timeline read is expensive — cache aggressively.
- **Tweet exceeds 280 chars:** validate and reject.

### D — Discussion (5 min)
- **Bottleneck:** Fan-out writes for popular users. Hybrid model solves this.
- **Alternative:** All-pull (no push) = simpler but slower reads. Acceptable if timeline latency tolerance is higher.
- **Scaling:** Shard Cassandra by `user_id`. Shard Redis by `user_id` (consistent hashing).
- **CDN:** Media (images, videos) served via CDN. Tweet text doesn't need CDN.
- **Monitoring:** Track timeline generation latency (p99), fan-out write latency, cache hit rate.

---

## Exercise: Add Trends/Topics to Twitter

Design the "Trending Topics" feature:
1. Show top 10 trending hashtags for a user's region.
2. Updated every 5 minutes.
3. Must handle sudden spikes (e.g., breaking news).

**Think about:**
- How to count hashtag occurrences in real-time (stream processing).
- How to rank trends (velocity, not just volume).
- How to regionalize trends.
- How to cache the top 10 per region.

<details>
<summary>Approach</summary>

1. **Stream processing:** Kafka consumer processes tweet events → extracts hashtags → counts per hashtag in tumbling 5-min windows (Flink or Kafka Streams).

2. **Ranking:** Not just raw count — use velocity (rate of increase). A hashtag that went from 100 to 10,000 mentions is "trending" more than one with a steady 50,000.

3. **Regionalization:** Tag each tweet with the user's region (from profile or IP). Count per region.

4. **Caching:** Compute top 10 per region every 5 minutes → store in Redis (`trends:region:US` → [list of hashtags]). API reads from Redis (O(1)).

5. **Spikes:** The 5-minute window naturally captures spikes. If a breaking event happens, the next window update reflects it within 5 minutes.
</details>

---

## Common Mistakes
- Choosing all-push without addressing the celebrity problem.
- Choosing all-pull without addressing the latency problem (1000+ queries per timeline view).
- Not separating tweet storage (Cassandra) from timeline cache (Redis).
- Forgetting that timelines are personalized (can't cache one timeline for all users).
- Not handling tweet deletion from cached timelines.

## Checklist Before Moving On
- [ ] Can explain the fan-out problem (push vs pull vs hybrid)
- [ ] Understand the celebrity problem and the hybrid solution
- [ ] Know the data model: Cassandra for tweets, Redis for timelines, PostgreSQL for users/follows
- [ ] Can design the API for posting tweets and reading timelines
- [ ] Understand Snowflake ID generation
- [ ] Solved the Twitter design using RESHADED
