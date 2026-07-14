# Day 30 - Final Review & Interview Readiness

## Topic
**Consolidation day.** Review your weak list, finalize your cheat sheet, rehearse interview strategy, and build confidence. No new material — this is about readiness.

---

## Why This Day Matters
- 29 days of learning built breadth. Today builds **confidence and recall**.
- Your weak list (from Days 7, 14, 21, 28, 29) is your highest-ROI study target.
- A polished cheat sheet and rehearsed interview protocol reduce anxiety on the real day.

---

## Part 1: Re-Solve Your Weak List (2 hours)

Go through every topic you struggled with in mocks or reviews. Re-study and re-attempt.

### Common Weak Spots (check yours)

| Topic | Likely issue | Fix |
|-------|-------------|-----|
| Estimation | Getting lost in numbers | Practice 3 estimates (URL shortener, Twitter, chat) |
| Fan-out | Push vs pull confusion | Re-read Day 23, draw the hybrid model |
| Sharding | Consistent hashing explanation | Re-read Day 13, draw the ring |
| API design | Forgetting pagination/idempotency | Re-read Day 10, design 3 APIs |
| Trade-offs | Not articulating "because... trade-off is..." | Practice the trade-off drill from Day 21 |
| Real-time | WebSocket routing across servers | Re-read Day 17, draw the Redis Pub/Sub flow |
| CDC | Confusing with dual-writes | Re-read Day 18, draw the outbox pattern |
| Consensus | Raft election steps | Re-read Day 15, trace a leader election |
| Observability | RED vs USE confusion | Re-read Day 19, memorize the three pillars |
| Security | JWT vs sessions | Re-read Day 20, list pros/cons of each |

---

## Part 2: Finalize Your Cheat Sheet

Your `cheatsheet.md` should now contain all 30 days of knowledge. Here's the final structure:

```
## RESHADED Framework
R - Requirements (functional + non-functional, scope)
E - Estimation (QPS, storage, bandwidth, cache)
S - Storage schema (DB choices, schema)
H - High-level design (box-and-arrow)
A - API design (endpoints, pagination, idempotency)
D - Detailed design (deep dive, patterns)
E - Edge cases (failures, scale, bottlenecks)
D - Discussion (trade-offs, alternatives)

## Key Numbers
- Latency: L1 0.5ns, Memory 100ns, SSD 150µs, Network RTT 0.5ms, Disk 20ms
- Availability: 99.9% = 8.76h/yr, 99.99% = 52.6min/yr, 99.999% = 5.26min/yr
- 1 day = 86400 sec, 1 KB = 10^3, 1 MB = 10^6, 1 GB = 10^9, 1 TB = 10^12, 1 PB = 10^15

## Networking
- TCP: reliable, ordered, connection-oriented. UDP: fast, unreliable.
- HTTP/2: multiplexing, header compression. HTTP/3 (QUIC): 0-RTT.
- TLS: encrypts in transit. Terminate at LB or server.
- WebSocket: bidirectional. SSE: one-way push. Long polling: fallback.

## Scaling
- Vertical: bigger machine (limited, SPOF). Horizontal: more machines (preferred).
- LB algorithms: round-robin, least-connections, IP-hash, consistent hashing.
- Layer 4 (TCP) vs Layer 7 (HTTP) load balancing.

## Databases
- SQL (PostgreSQL): ACID, relational, transactions. Use for: users, payments, orders.
- NoSQL: 
  - Redis: cache, sessions, leaderboards, rate limiting, geospatial
  - Cassandra: high-write, time-series, timelines (partition by key, cluster by time)
  - MongoDB: flexible schema, documents
  - DynamoDB: managed key-value, auto-scaling
- Indexes: B-tree (equality + range), composite (multi-column). Trade-off: faster reads, slower writes.

## CAP Theorem
- C: consistency, A: availability, P: partition tolerance. Guarantee 2 of 3 (P mandatory).
- CP: MongoDB, HBase (consistency over availability). AP: Cassandra, DynamoDB (availability over consistency).
- PACELC: even without partitions, trade Latency vs Consistency.

## Storage
- Block: low latency, single-server (databases). EBS.
- Object: HTTP API, unlimited, cheap. S3. (images, videos, backups)
- File: shared, NFS. EFS.
- CDN: cache at edge nodes, reduce latency from 150ms to <20ms.

## Caching
- Patterns: cache-aside (common), write-through, write-back, write-around.
- Eviction: LRU (common), LFU, TTL.
- Problems: stampede (lock+jitter), penetration (cache nulls), avalanche (jitter TTL).
- Redis: in-memory, single-threaded, persistence (RDB/AOF).

## Message Queues
- Kafka: topics → partitions → consumer groups. At-least-once + idempotent consumers.
- RabbitMQ: task queues, complex routing. SQS: simple AWS-native.
- Patterns: point-to-point, pub/sub, competing consumers.
- Use: async processing, decoupling, load leveling.

## API Design
- REST: nouns, plurals, HTTP methods, cursor pagination, versioning (/v1/).
- gRPC: service-to-service, protobuf, HTTP/2, streaming.
- GraphQL: flexible client queries, single endpoint, no over-fetching.
- Idempotency key: UUID for safe POST retries.
- Real-time: SSE (one-way), WebSocket (bidirectional), long polling (fallback).

## Microservices
- When: large teams, different scaling needs, different tech.
- API gateway: routing, auth, rate limit, SSL, caching, aggregation.
- Service discovery: registry (etcd/Consul/K8s DNS).
- Circuit breaker: CLOSED → OPEN → HALF-OPEN.
- Sync (gRPC) vs async (Kafka) communication.

## Database Scaling
- Replication: master-slave (scales reads), master-master (scales writes, conflict risk).
- Sharding: split by shard key (high cardinality, even distribution, query locality).
- Consistent hashing: adding shards moves ~1/N keys.
- Snowflake ID: timestamp + shard + sequence → sortable, sharded.

## Distributed Systems
- Raft: leader election (majority), log replication (majority commit).
- Split-brain: quorum (odd nodes), fencing tokens.
- Distributed lock: Redis SETNX + TTL + ownership check (Lua script).
- 2PC vs Saga: Saga for microservices (compensating actions).
- Outbox pattern: DB write + event in same tx → CDC publishes.

## Real-Time Systems
- WebSocket cluster: Redis Pub/Sub for cross-server routing.
- Presence: Redis with TTL + heartbeat.
- Push notifications: APNs (iOS), FCM (Android).

## Data Pipelines
- CDC (Debezium): tail DB log → Kafka → sync to ES/cache/warehouse.
- Batch (Spark) vs Stream (Flink/Kafka Streams).
- Lambda (batch+speed) vs Kappa (stream-only).
- Warehouse (structured) vs Lake (raw) vs Lakehouse.

## Observability
- Logging: structured JSON, request_id correlation.
- Metrics: RED (Rate, Errors, Duration), USE (Utilization, Saturation, Errors).
- Percentiles: p50, p90, p99 (not average!).
- Tracing: trace_id + spans, OpenTelemetry, Jaeger.
- Health: liveness (alive?) vs readiness (ready to serve?).

## Security
- AuthN: sessions (server-side) vs JWT (stateless) vs OAuth (third-party).
- AuthZ: RBAC (roles), ABAC (attributes), ACL (per-resource).
- Rate limiting: token bucket, sliding window.
- Encryption: TLS (transit), disk/app (rest), bcrypt (passwords).
- Best practices: HttpOnly cookies, parameterized queries, CORS, least privilege.
```

---

## Part 3: Interview Meta-Strategy (rehearse this)

### Before the Interview
- Re-read your cheat sheet (no new learning).
- Do 1 warmup design (a problem you know cold) to build momentum.
- Prepare your environment: whiteboard/notes app, water, quiet space.
- Sleep 8 hours.

### During the Interview
1. **Start with requirements.** Ask 3-5 clarifying questions. Write them down.
2. **Estimate.** Show your work. Round numbers are fine.
3. **Draw before code.** Sketch the high-level design. Get buy-in.
4. **Name your components.** "This is the API Gateway, this is Redis for caching..."
5. **Justify choices.** "I chose Cassandra because of high write throughput. The trade-off is no JOINs."
6. **Deep dive.** Pick the hardest part and explain it in detail.
7. **Handle edge cases.** "What if the DB is down?" "What if traffic spikes 10x?"
8. **Discuss trade-offs.** "The alternative is X, but I chose Y because Z."
9. **Drive.** Don't wait for the interviewer. Propose, justify, move forward.

### Communication Phrases
- "Let me clarify the requirements..."
- "I'll estimate the scale..."
- "The key entities are..."
- "I'll use [component] because [reason]. The trade-off is..."
- "The bottleneck is... I'll solve it with..."
- "If traffic increases 10x, I would..."
- "An alternative approach is... but I prefer... because..."
- "For this, I'd draw on the [pattern] we discussed..."

### If You Get Stuck
- Restate the problem and your current design.
- Try a simpler version first, then add complexity.
- Ask: "I'm considering two approaches: A and B. Any preference?"
- It's OK to say "I'm not sure about X, but I'd research Y" — honesty > BS.

---

## Part 4: The Senior Engineer Signal (9 YOE)

At your experience level, interviewers expect more than just boxes and arrows. Demonstrate:

1. **Operational awareness:** "I'd add health checks, monitoring, and alerting for this component."
2. **Cost consciousness:** "Using a CDN reduces egress costs significantly versus serving from S3 directly."
3. **Real-world experience:** "In my experience with Go services, I've found that goroutine-per-connection handles WebSocket scaling well."
4. **Leadership:** "I'd start with a simpler architecture and evolve it as we learn the traffic patterns."
5. **Trade-off fluency:** "The trade-off between strong and eventual consistency here is... I'd choose eventual because..."
6. **Failure mode thinking:** "If this service goes down, the impact would be... I'd mitigate with..."

---

## Part 5: Final Pattern-Frequency Table

| System Design Topic | Interview Frequency |
|---------------------|---------------------|
| Caching (Redis) | Very High |
| Load balancing | Very High |
| Database choice (SQL vs NoSQL) | Very High |
| API design (REST) | Very High |
| Sharding / partitioning | High |
| Message queues (Kafka) | High |
| Microservices + API gateway | High |
| Estimation (QPS, storage) | High |
| CDN | High |
| Fan-out (social media) | Medium-High |
| WebSockets / real-time | Medium-High |
| CAP theorem | Medium |
| Distributed locks | Medium |
| Observability | Medium |
| Security (JWT, OAuth) | Medium |
| CDC / data pipelines | Lower |
| Raft / consensus | Lower |
| gRPC / GraphQL | Lower |

---

## Part 6: Mindset

- You've learned 15+ building blocks, designed 6 full systems, and done 2 mock interviews.
- Your 9 years of Go experience means you understand production systems — that's your edge.
- System design interviews evaluate **how you think**, not just what you draw. A candidate who communicates well and designs 80% of a system beats one who silently draws 100%.
- If you blank: breathe, restate the problem, start with the simplest version. Complexity comes later.
- **Go is your advantage.** Mention Go's strengths (goroutines for concurrency, gRPC, context for cancellation, etcd for coordination) when relevant.

---

## Final Checklist (are you ready?)

- [ ] Memorized RESHADED framework
- [ ] Memorized key numbers (latency, availability, sizes)
- [ ] Can estimate QPS, storage, bandwidth for any system
- [ ] Know when to use SQL vs NoSQL vs Redis vs S3
- [ ] Can explain CAP theorem and CP vs AP databases
- [ ] Can design a caching layer (cache-aside, eviction, stampede prevention)
- [ ] Can design a WebSocket cluster with Redis Pub/Sub
- [ ] Can explain consistent hashing and sharding
- [ ] Can design REST APIs with pagination and idempotency
- [ ] Can explain the fan-out problem (push/pull/hybrid)
- [ ] Can discuss trade-offs for every major decision
- [ ] Can handle edge cases (failures, scale, bottlenecks)
- [ ] Completed 2 mock interviews (Day 28, 29)
- [ ] Cheat sheet is complete and reviewed

---

## You're Ready. Good Luck.

```
30 days. 15 building blocks. 6 full system designs. 2 mock interviews.
One senior Go engineer, now fluent in system design.
```

Open the real problem statement, breathe, and trust the RESHADED framework. You've got this.
