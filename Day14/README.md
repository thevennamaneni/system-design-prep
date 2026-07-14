# Day 14 - Review: Core Building Blocks & Pattern Recognition

## Topic
**Consolidation.** Review Week 2 concepts, practice pattern recognition ("when to use what"), and build your system design cheat sheet.

---

## Why This Day Matters
- Week 2 introduced the core building blocks (caching, MQs, APIs, microservices, sharding). These appear in EVERY system design.
- Without review, you'll confuse patterns under pressure.
- Pattern recognition — "this problem needs a message queue + cache + sharded DB" — is the skill that makes interviews feel easy.

---

## Part 1: Quick-Answer Drill

1. Name 4 cache patterns. Which is most common?
2. What's the thundering herd problem? How do you solve it?
3. Kafka: what's a partition? A consumer group? An offset?
4. At-least-once vs exactly-once delivery — which is more common? Why?
5. REST vs gRPC vs GraphQL — when do you use each?
6. What's an idempotency key? Why do POST requests need them?
7. Cursor pagination vs offset pagination — which is better for large datasets?
8. Monolith vs microservices — when would you choose monolith?
9. What does an API gateway do? Name 5 responsibilities.
10. What's a circuit breaker? Name its 3 states.
11. Consistent hashing vs `hash % N` — why is consistent hashing better?
12. What's the fan-out problem in Twitter's timeline? Push vs pull?
13. Master-slave replication: does it scale writes or reads?
14. What makes a good shard key? Give an example of a bad one.
15. What's the difference between SSE and WebSocket?

<details>
<summary>Quick Answers</summary>

1. Cache-aside (most common), write-through, write-back, write-around.
2. Cache stampede: many requests miss simultaneously, overwhelm DB. Fix: locking, early expiration, jittered TTL.
3. Partition: ordered log within a topic. Consumer group: set of consumers sharing partitions. Offset: position in partition.
4. At-least-once is more common. Exactly-once is hard; at-least-once + idempotent consumers is sufficient.
5. REST: public APIs. gRPC: internal service-to-service. GraphQL: flexible client queries.
6. Client-generated UUID sent with POST. Server deduplicates: if key seen, return previous result. Prevents duplicates on retry.
7. Cursor pagination — no offset scan, stable results, faster for large datasets.
8. Small team, early stage, simple domain, need ACID across modules.
9. Routing, auth, rate limiting, SSL termination, caching, logging, request aggregation, protocol translation.
10. Circuit breaker: prevents cascading failures. States: CLOSED (normal), OPEN (fail fast), HALF-OPEN (test).
11. Consistent hashing: adding a shard moves ~1/N of keys. `hash % N`: adding a shard can move 80%+ of keys.
12. Fan-out: when a user tweets, how to deliver to all followers' timelines. Push: write to all timelines. Pull: read-time merge. Hybrid: push for normal users, pull for celebrities.
13. Reads. Writes still go to one master.
14. Good: high cardinality, even distribution, query locality (e.g., `user_id`). Bad: auto-increment ID (hotspot), low cardinality (uneven).
15. SSE: one-way (server→client), HTTP-based. WebSocket: bidirectional, protocol upgrade.
</details>

---

## Part 2: Pattern Recognition Drill

For each scenario, name the building blocks/patterns you'd use:

1. "Reduce database load for a read-heavy API."
2. "Process a payment asynchronously — user shouldn't wait."
3. "1000 servers need to find Service X's current IP."
4. "Mobile app needs different data fields for different screens."
5. "A database has 5 billion rows and writes are too slow."
6. "If Service B is down, Service A should fail fast, not hang."
7. "Serve images globally with < 50ms latency."
8. "User posts a tweet — notify 10 million followers."
9. "API needs to handle 100x traffic spike without crashing."
10. "Two services need to update their data atomically."

<details>
<summary>Answers</summary>

1. **Cache (Redis, cache-aside)** — cache hot data, reduce DB reads.
2. **Message queue (Kafka/SQS)** — publish event, consumer processes payment async.
3. **Service discovery (etcd/Consul/K8s DNS)** — registry for service locations.
4. **GraphQL** — client requests exactly the fields it needs.
5. **Sharding** — split data across multiple DB instances.
6. **Circuit breaker** — trip after failures, return error immediately.
7. **CDN** — cache at edge nodes close to users.
8. **Fan-out on write (push to timeline caches)** + Kafka for decoupled delivery.
9. **Load balancer + auto-scaling + caching + rate limiting**.
10. **2-phase commit (2PC)** or **Saga pattern** (distributed transaction).
</details>

---

## Part 3: Architecture Component Drill

Name the component for each role:

| Role | Component |
|------|-----------|
| Distributes traffic across servers | ? |
| Decrypts HTTPS at the edge | ? |
| Caches database query results | ? |
| Decouples producers from consumers | ? |
| Routes `/api/users` to the user service | ? |
| Stores user session data | ? |
| Finds the IP of a service | ? |
| Serves images from a Tokyo edge node | ? |
| Prevents cascading failures | ? |
| Splits a table across 10 machines | ? |
| Generates sortable, unique IDs across shards | ? |
| Handles "user is typing" real-time | ? |

<details>
<summary>Answers</summary>

| Role | Component |
|------|-----------|
| Distributes traffic | Load balancer |
| Decrypts HTTPS | Load balancer (SSL termination) / API gateway |
| Caches query results | Redis (cache-aside) |
| Decouples producers/consumers | Message queue (Kafka/SQS) |
| Routes by URL path | API gateway |
| Session data | Redis |
| Finds service IP | Service registry (etcd/Consul) |
| Tokyo edge caching | CDN |
| Prevents cascading failures | Circuit breaker |
| Splits table across machines | Sharding |
| Unique sortable IDs | Snowflake ID |
| Real-time typing | WebSocket |
</details>

---

## Part 4: Update Your Cheat Sheet

Add to `cheatsheet.md`:

```
## Caching
- Patterns: cache-aside (common), write-through, write-back, write-around
- Eviction: LRU (common), LFU, TTL
- Problems: stampede (fix: lock+jitter), penetration (cache nulls), avalanche (jitter TTL)
- Redis: sessions, cache, leaderboards, rate limiting, geospatial, pub/sub

## Message Queues
- Kafka: topics → partitions → consumer groups; at-least-once + idempotent consumers
- RabbitMQ: task queues, complex routing
- SQS: simple AWS-native decoupling
- Patterns: point-to-point, pub/sub, competing consumers
- Use: async processing, decoupling, load leveling

## API Design
- REST: nouns, plurals, HTTP methods, cursor pagination, versioning (/v1/)
- gRPC: service-to-service, protobuf, HTTP/2, streaming
- GraphQL: flexible client queries, single endpoint, no over-fetching
- Idempotency key: UUID for safe POST retries
- Real-time: SSE (one-way push), WebSocket (bidirectional), long polling (fallback)

## Microservices
- When: large teams, different scaling needs, different tech stacks
- API gateway: routing, auth, rate limit, SSL, caching, aggregation
- Service discovery: registry (etcd/Consul/K8s DNS)
- Circuit breaker: CLOSED → OPEN → HALF-OPEN
- Service mesh: Istio/Linkerd (mTLS, retries, observability)
- Sync (gRPC) vs async (Kafka) inter-service communication

## Database Scaling
- Replication: master-slave (scales reads), master-master (scales writes, conflict risk)
- Sharding: split table across machines by shard key
  - Strategies: range, hash, consistent hashing (best), directory
  - Good key: high cardinality, even distribution, query locality
  - Bad key: auto-increment (hotspot), low cardinality (uneven)
- Consistent hashing: adding shards moves ~1/N keys (not 80%+)
- Fan-out: push (write-heavy), pull (read-heavy), hybrid (celebrity exception)
- Snowflake IDs: timestamp + shard ID + sequence → sortable, sharded
```

---

## Part 5: Mini Design Challenge (15 min)

**Prompt:** "Design a comment system for a blog platform."

Using RESHADED, sketch the design in 15 minutes:
1. **R:** What questions do you ask?
2. **E:** Estimate (100K posts/day, 10 comments/post avg)
3. **S:** Database choice and schema
4. **H:** High-level architecture (draw it)
5. **A:** API endpoints
6. **D:** How do you handle nested comments (replies to replies)?
7. **E:** What if the DB is slow?
8. **D:** Trade-offs

> Don't fully code — just sketch. This practices the "quick design" skill.

---

## Checklist Before Moving On
- [ ] Answered all 15 quick-answer questions
- [ ] Pattern recognition drill: 8/10 correct
- [ ] Component drill: 10/12 correct
- [ ] Updated cheat sheet with Week 2 building blocks
- [ ] Completed the mini design challenge
- [ ] Ready for Week 3 (advanced distributed systems)
