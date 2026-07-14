# Day 21 - Review: Advanced Topics & Trade-Off Analysis

## Topic
**Consolidation.** Review Week 3, practice trade-off analysis (the #1 senior skill), and rehearse the full RESHADED framework on a mini design.

---

## Why This Day Matters
- Week 3 covered the hardest material (consensus, distributed transactions, real-time, observability, security).
- Trade-off analysis is what separates senior from mid-level candidates. Every design decision must have a "because" and a "trade-off is..."
- You're 70% through the plan. This review cements the foundation for full system designs.

---

## Part 1: Quick-Answer Drill

1. Raft: how does leader election work? What's a "term"?
2. Why do we need an ODD number of nodes in a consensus cluster?
3. What's a fencing token and why is it needed?
4. Redis distributed lock: what 3 things must you get right?
5. 2PC vs Saga: which is better for microservices? Why?
6. What's the Outbox pattern? What problem does it solve?
7. WebSocket vs SSE: when would you choose each?
8. How do you route a WebSocket message from User A (on Server 1) to User B (on Server 3)?
9. What's CDC? Name a tool.
10. What's the difference between data warehouse and data lake?
11. Three pillars of observability?
12. What's the RED method? What's the USE method?
13. Why use percentiles (p99) instead of average latency?
14. JWT vs session-based auth: pros/cons?
15. What's the token bucket rate limiting algorithm?

<details>
<summary>Quick Answers</summary>

1. Follower times out → becomes candidate → requests votes → majority wins → sends heartbeats. A term is a logical period with at most one leader.
2. Odd numbers guarantee a majority in any partition. With 4 nodes, a 2|2 split has no majority → both sides block. With 5, a 3|2 split always has a majority.
3. A monotonically increasing number assigned to each leader. Prevents a "zombie" leader from corrupting data after it's been deposed (storage rejects old tokens).
4. (1) Set TTL (prevent permanent lock if holder crashes). (2) Use unique value (check ownership before release). (3) Use Lua script for atomic release.
5. Saga — 2PC is blocking, slow, and not partition-tolerant. Saga uses compensation, is async, and fits microservices.
6. Write DB change + event in the same transaction. A CDC process reads the outbox and publishes events. Solves the dual-write problem (DB write succeeds but event publish fails).
7. WebSocket: bidirectional (chat, gaming). SSE: one-way push (notifications, live feeds).
8. Redis Pub/Sub. Server 1 publishes to "user:B" channel. Server 3 (subscribed to "user:B") receives and pushes to User B's WebSocket.
9. Change Data Capture: capture DB changes from the transaction log. Tool: Debezium.
10. Warehouse: structured, queried (SQL). Lake: raw, unstructured, schema-on-read.
11. Logging, metrics, distributed tracing.
12. RED: Rate, Errors, Duration (for services). USE: Utilization, Saturation, Errors (for resources).
13. Average hides tail latency. p99 shows the experience of 99% of users — outliers are visible.
14. JWT: stateless, scales, but can't revoke easily. Session: revocable, but needs server-side storage.
15. Bucket holds N tokens, refills at R/sec. Each request takes 1 token. Empty bucket = deny. Allows bursts.
</details>

---

## Part 2: Trade-Off Analysis Practice

For each decision, state the trade-off in the format: "I chose X because Y. The trade-off is Z."

1. **SQL vs NoSQL for a social media feed.**
2. **Push vs Pull for timeline generation.**
3. **Strong vs eventual consistency for a shopping cart.**
4. **gRPC vs REST for internal service communication.**
5. **Redis lock vs etcd lock for distributed locking.**
6. **Monolith vs microservices for a startup.**
7. **WebSocket vs long polling for a chat app.**
8. **Synchronous vs async payment processing.**
9. **Cache-aside vs write-through for a product catalog.**
10. **Kafka vs SQS for an event-driven system.**

<details>
<summary>Sample Trade-Off Answers</summary>

1. "I chose NoSQL (Cassandra) for the feed because of high write throughput and horizontal scaling. The trade-off is no ACID transactions and no JOINs — I'd use PostgreSQL for user data that needs consistency."

2. "I chose push (fan-out on write) for timeline generation because it makes reads O(1) (pre-built in Redis). The trade-off is high write amplification — a celebrity's tweet must be pushed to millions of timelines. I'd use a hybrid: push for normal users, pull for celebrities."

3. "I chose eventual consistency for the cart because it's not money — a slightly stale cart is acceptable. The trade-off is potential race conditions if two devices edit simultaneously. I'd use strong consistency at checkout (the actual order creation)."

4. "I chose gRPC for internal communication because of performance (protobuf, HTTP/2) and strong contracts (proto file). The trade-off is it's not browser-friendly and adds protobuf compilation overhead."

5. "I chose Redis for the lock because it's faster and we already use Redis. The trade-off is Redis isn't as strongly consistent as etcd (Raft). For critical locks (money), I'd use etcd."

6. "I chose monolith for the startup because the team is small and the domain is simple. The trade-off is future scaling challenges — we'll need to extract services as we grow."

7. "I chose WebSocket for the chat app because it's bidirectional — users both send and receive messages. The trade-off is more complex infrastructure (connection management, reconnection) compared to long polling."

8. "I chose async payment processing because the user shouldn't wait for the payment gateway (which can take seconds). The trade-off is the user doesn't know immediately if payment succeeded — I'd show 'processing' and notify via webhook/push."

9. "I chose cache-aside for the catalog because reads are frequent but writes are rare (products don't change often). The trade-off is stale data on cache miss + DB update — I'd use a short TTL (5 min) or cache invalidation on product update."

10. "I chose Kafka because I need event replay, high throughput, and multiple consumers. The trade-off is operational complexity (managing Kafka clusters) vs SQS's simplicity. For simple decoupling, SQS would be better."
</details>

---

## Part 3: Mini System Design (20 min)

**Prompt:** "Design a Notification System" (email, SMS, push).

Using RESHADED, sketch the design in 20 minutes:

1. **R:** Requirements — what types of notifications? Who triggers them? Real-time or batched?
2. **E:** Estimate — 10M notifications/day. QPS? Storage?
3. **S:** Storage — what databases? What schema?
4. **H:** High-level design — draw the architecture.
5. **A:** API design — what endpoints?
6. **D:** Detailed design — how do you handle multiple channels (email/SMS/push)? How do you handle failures?
7. **E:** Edge cases — what if the email service is down? How to prevent spam?
8. **D:** Discussion — trade-offs, alternatives.

> This is practice for Week 4's full designs. Don't code — sketch and discuss.

<details>
<summary>Reference Design Sketch</summary>

**R:** 
- Functional: send email, SMS, push notifications. Triggered by events (order placed, payment failed, new message). User preferences (opt-in/opt-out per channel).
- Non-functional: at-least-once delivery, < 30 sec latency for push, < 5 min for email, 99.9% availability.

**E:**
- 10M notifications/day → ~115 QPS avg, ~350 peak.
- Each notification: ~500 bytes → 5GB/day, ~2TB/year.

**S:**
- PostgreSQL: notification metadata (user_id, type, status, content, timestamps).
- Redis: user preferences (opt-in), dedup (prevent same notification twice).
- S3: email templates (HTML).

**H:**
```
Event Source → Kafka → Notification Service → Channel Adapter → Email/SMS/Push Provider
                                    ↓
                              PostgreSQL (store)
                              Redis (prefs, dedup)
```

**A:**
```
POST /notifications  — send notification (internal API)
GET /notifications?user_id=X — get user's notifications
PUT /preferences/{user_id} — update notification preferences
```

**D:**
- **Strategy pattern:** `NotificationChannel` interface with `EmailAdapter`, `SMSAdapter`, `PushAdapter`.
- **Queue per channel:** separate Kafka topics for email, SMS, push (different rate limits, providers).
- **Retry:** exponential backoff (1s, 2s, 4s, 8s... max 5 retries). After max → dead-letter queue.
- **User preferences:** check Redis before sending (opted out? quiet hours?).
- **Dedup:** hash(user_id + type + content) → check Redis (prevent duplicate notifications).

**E:**
- Email service down → retry (exponential backoff). After max retries → DLQ → alert ops.
- Prevent spam → rate limit per user (max 10 notifications/hour). 
- Quiet hours → check user's timezone + preferences before sending push at 3 AM.

**D:**
- Trade-off: at-least-once delivery means possible duplicates. Idempotent consumers (dedup hash).
- Alternative: direct call to providers (no queue) → simpler but less resilient. Queue is better for decoupling.
</details>

---

## Part 4: Update Cheat Sheet

Add Week 3 topics to `cheatsheet.md`:

```
## Distributed Consensus
- Raft: leader election (terms, majority), log replication (majority commit)
- Split-brain prevention: quorum (odd nodes), fencing tokens
- etcd/ZooKeeper: coordination, leader election, service discovery, config

## Distributed Transactions
- 2PC: strong consistency, blocking, slow → avoid in microservices
- Saga: sequence of local transactions + compensations
- Outbox pattern: DB write + event in same tx → CDC publishes event
- Idempotency: idempotency key (POST), unique constraint, optimistic concurrency

## Real-Time Systems
- WebSocket: bidirectional, persistent (chat, gaming)
- SSE: one-way push (notifications, live feeds), auto-reconnect
- Long polling: fallback, legacy
- WebSocket cluster: Redis Pub/Sub for cross-server routing
- Push notifications: APNs (iOS), FCM (Android) for mobile
- Presence: Redis with TTL + heartbeat

## Data Pipelines
- ETL vs ELT
- CDC (Debezium): tail DB transaction log → Kafka → sync to ES/cache/warehouse
- Batch (Spark) vs Stream (Flink/Kafka Streams)
- Lambda (batch+speed) vs Kappa (stream-only)
- Warehouse (structured) vs Lake (raw) vs Lakehouse (both)

## Observability
- Logging: structured JSON, request_id correlation, ELK/Datadog
- Metrics: RED (Rate, Errors, Duration), USE (Utilization, Saturation, Errors)
- Percentiles: p50, p90, p99 (not average!)
- Tracing: trace_id + spans, OpenTelemetry, Jaeger
- Health: liveness (alive?) vs readiness (ready to serve?)

## Security
- AuthN: sessions (server-side) vs JWT (stateless) vs OAuth (third-party)
- AuthZ: RBAC (roles), ABAC (attributes), ACL (per-resource)
- Rate limiting: token bucket, sliding window, fixed window
- Encryption: TLS (transit), disk/app-level (rest), bcrypt (passwords)
- Best practices: HttpOnly cookies, parameterized queries, CORS, least privilege
```

---

## Checklist Before Moving On
- [ ] Answered all 15 quick-answer questions
- [ ] Trade-off analysis: can articulate "I chose X because Y, trade-off is Z" for any decision
- [ ] Completed the Notification System mini design (20 min)
- [ ] Updated cheat sheet with Week 3 topics
- [ ] Ready for Week 4 (full system designs + mock interviews)
