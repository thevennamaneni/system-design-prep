# Day 29 - Mock Interview #2 (Different Domain) + Common Questions

## Topic
**Second mock interview** with a different problem domain, plus a review of **common system design interview questions** and how to approach them.

---

## Why This Day Matters
- Two mocks give you comparison data: are you improving?
- Different domains test different muscles (media vs real-time vs search vs booking).
- Familiarity with common questions reduces surprise on the real day.

---

## Part 1: Mock Interview #2 (60 minutes)

### Pick a problem from a DIFFERENT domain than Day 28.

If Day 28 was Instagram (social/media), pick from:
- **Booking system** (Airbnb, hotel, movie tickets)
- **Real-time system** (live scores, stock ticker, collaborative editing)
- **Search system** (Google search, product search, autocomplete)
- **Streaming system** (Netflix, Spotify, Twitch)
- **Infrastructure** (distributed cache, rate limiter, message queue)

### Follow the same RESHADED structure (60 min)
1. R — Requirements (5 min)
2. E — Estimation (5 min)
3. S — Storage schema (5 min)
4. H — High-level design (10 min)
5. A — API design (5 min)
6. D — Detailed design (15 min)
7. E — Edge cases (5 min)
8. D — Discussion (5 min)

### Self-Evaluation (same rubric as Day 28)
| Dimension | Score |
|-----------|-------|
| Requirements | /5 |
| Estimation | /5 |
| Storage | /5 |
| Architecture | /5 |
| API | /5 |
| Detailed design | /5 |
| Edge cases | /5 |
| Trade-offs | /5 |
| Communication | /5 |
| Time management | /5 |
| **Total** | **/50** |

**Compare to Day 28:** Did you improve? Where are you still weak?

---

## Part 2: Common System Design Questions (Quick-Fire)

These are the most frequently asked system design questions. For each, write down the **3 key design decisions** you'd make (2 min per question):

### 1. Design Twitter/X
**Key decisions:**
1. Fan-out: hybrid (push for normal, pull for celebrities)
2. Storage: Cassandra (tweets), Redis (timeline cache), PostgreSQL (users/follows)
3. Snowflake ID for tweet IDs (sortable + sharded)

### 2. Design a URL Shortener
**Key decisions:**
1. Short code: auto-increment ID + base62 encoding (or pre-generated keys)
2. Storage: DynamoDB/Cassandra (key-value, high read QPS)
3. Caching: Redis (cache short→long mapping, 95%+ hit rate)

### 3. Design Netflix/YouTube
**Key decisions:**
1. HLS: split video into segments, multiple resolutions, adaptive bitrate
2. CDN: segment-level caching at edge nodes (95%+ cache hit)
3. Transcoding: Kafka → FFmpeg workers → S3 → CDN

### 4. Design Uber
**Key decisions:**
1. Geospatial: Redis GEO for nearby driver search
2. Real-time: WebSocket + Redis Pub/Sub for location updates
3. Dispatch: send to N nearest drivers, first accept wins

### 5. Design WhatsApp
**Key decisions:**
1. WebSocket cluster: Redis Pub/Sub for cross-server routing
2. Message storage: Cassandra (high write, partitioned by chat_id)
3. Ordering: per-chat sequence number (Redis INCR)

### 6. Design Instagram
**Key decisions:**
1. Media: S3 + CDN (images/videos), PostgreSQL (metadata)
2. Feed: fan-out on write (push to followers' feeds in Redis)
3. Timeline: Redis list of post IDs, fetch details from DB

### 7. Design Google Docs
**Key decisions:**
1. Collaboration: Operational Transformation (OT) or CRDTs for conflict resolution
2. Real-time: WebSocket for live editing
3. Storage: versioned document state + operation log (event sourcing)

### 8. Design a Rate Limiter
**Key decisions:**
1. Algorithm: token bucket (allows bursts, smooth average)
2. Storage: Redis (atomic Lua script for check-and-increment)
3. Fail-open: if Redis is down, allow requests (availability over accuracy)

### 9. Design a Distributed Cache
**Key decisions:**
1. Sharding: consistent hashing (minimize redistribution on add/remove)
2. Replication: master-slave per shard for read scaling
3. Eviction: LRU (most common) or LFU

### 10. Design an Event Ticketing System (BookMyShow)
**Key decisions:**
1. Concurrency: seat locking (Redis SETNX with TTL) to prevent double-booking
2. Search: Elasticsearch (search by movie, theater, location)
3. Queue: Kafka for payment processing (async, decoupled)

### 11. Design Dropbox / Google Drive
**Key decisions:**
1. Storage: S3 for file blocks, PostgreSQL for metadata
2. Sync: file watching → diff → push changes via WebSocket/SSE
3. Chunking: split files into 4MB blocks (deduplication, efficient sync)

### 12. Design a Web Crawler
**Key decisions:**
1. URL frontier: priority queue (BFS with politeness + freshness)
2. Distributed: URL hash → assign to worker (sharded crawling)
3. Storage: crawled pages in S3, URL index in Cassandra/Bigtable

---

## Part 3: Common Follow-Up Questions

Interviewers often ask follow-up questions after your initial design. Prepare answers for:

### Scaling Questions
- "What if traffic increases 10x?" → Add more servers, shard DB, add cache, use CDN.
- "What if a data center goes down?" → Multi-region deployment, DNS failover, replicated data.
- "What's the bottleneck?" → Identify the slowest component and how to fix it.

### Consistency Questions
- "What if the cache and DB are out of sync?" → TTL + cache invalidation on write. Accept eventual consistency for non-critical data.
- "Can two users book the same seat?" → Distributed lock or DB unique constraint.
- "What if a message is delivered twice?" → Idempotent consumer (dedup by message ID).

### Failure Handling
- "What if Redis goes down?" → Fail-open (for rate limiting) or fail-closed (for locks). Have replicas.
- "What if Kafka is down?" → Messages buffered locally; retry on recovery. Or use SQS as fallback.
- "What if a downstream service is slow?" → Circuit breaker, timeout, fallback response.

### Deep Dives
- "How does consistent hashing work?" → Ring, virtual nodes, ~1/N redistribution.
- "How does TLS termination work?" → LB decrypts HTTPS, forwards HTTP to servers.
- "How does Kafka guarantee ordering?" → Within a partition (not across partitions).
- "How does HLS work?" → Segments + playlists; client fetches segments sequentially; adaptive bitrate.

---

## Part 4: Post-Mock Reflection

1. Which domain was harder? Why?
2. Did you improve from Day 28's mock?
3. Can you now structure a design in 60 minutes without getting stuck?
4. What's your single biggest weakness? → Focus Day 30 on fixing it.
5. Are you comfortable narrating throughout? If not, practice more.

---

## Checklist Before Moving On
- [ ] Completed second full 60-min mock interview
- [ ] Filled out the self-evaluation rubric
- [ ] Reviewed all 12 common system design questions
- [ ] Can answer common follow-up questions (scaling, consistency, failure)
- [ ] Compared performance to Day 28 mock — identified improvement
- [ ] Ready for final review (Day 30)
