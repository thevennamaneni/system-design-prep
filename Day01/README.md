# Day 1 - What is System Design? The Interview Process & Framework

## Topic
**System Design (SD) overview**, the interview format, and the **RESHADED framework** you'll use to structure every answer. Today sets the stage for the next 29 days.

---

## Why It Matters
- System design interviews are unstructured — there's no single "correct" answer. Having a **repeatable framework** prevents you from freezing or rambling.
- At 9 YOE, this is the **most important interview round** for senior roles. It determines your level (Senior vs Staff vs Principal).
- Most candidates fail not because they don't know the concepts, but because they **can't structure their thoughts** under time pressure.

---

## What Makes SD Different from LLD?

| Aspect | LLD (Code-level) | System Design (Architecture-level) |
|--------|------------------|------------------------------------|
| Scope | Single application/service | Entire distributed system |
| Output | Code (structs, interfaces, patterns) | Architecture diagram (boxes & arrows) |
| Concerns | SOLID, patterns, encapsulation | Scalability, availability, latency, cost |
| Scale | One process | Millions of users, thousands of servers |
| Example | Design a Parking Lot class hierarchy | Design Twitter (the whole platform) |

---

## The Interview Format (45-60 minutes)

```
┌─────────────────────────────────────────────────────────────┐
│  0-5 min   │ Requirements clarification                     │
│  5-10 min  │ Back-of-envelope estimation                   │
│  10-20 min │ High-level design (draw boxes & arrows)        │
│  20-35 min │ Detailed design (deep dive into components)    │
│  35-45 min │ Scalability, bottlenecks, trade-offs           │
│  45-50 min │ Discussion / Q&A                               │
└─────────────────────────────────────────────────────────────┘
```

### What the Interviewer Evaluates
1. **Problem clarification:** Do you ask the right questions?
2. **Architecture:** Can you design a working system?
3. **Scalability:** Can it handle 10x, 100x growth?
4. **Trade-off analysis:** Do you justify your choices?
5. **Communication:** Can you explain complex ideas clearly?
6. **Depth:** Do you understand the components deeply?
7. **Practicality:** Is your design realistic, not academic?

---

## The RESHADED Framework (MEMORIZE)

### R — Requirements (5 min)
**Never start designing without this.** Ask about:

**Functional requirements** (what the system does):
- "What features are needed?" (e.g., for Twitter: post tweet, follow, timeline)
- "Who are the users?" (consumers, admins, API clients)

**Non-functional requirements** (how the system behaves):
- **Scale:** How many users? QPS (queries per second)?
- **Latency:** Real-time? Near-real-time? Batch?
- **Availability:** 99.9%? 99.99%?
- **Consistency:** Strong? Eventual?
- **Durability:** Can we lose data? Never?

**Scope:** What's in/out?
- "Should I handle authentication?" (often: no, assume it exists)
- "Mobile only? Web? API?"

> **Pro tip:** Write the requirements down on the whiteboard. Interviewers want to see this.

### E — Estimation (3 min)
Back-of-envelope calculations to size the system:

```
Users: 100M MAU (monthly active users)
Daily active: 30M
Tweets per user/day: 5
Total tweets/day: 150M
Tweets/sec (avg): 150M / 86400 ≈ 1,700
Peak (3x avg): ~5,000 tweets/sec

Storage per tweet: ~200 bytes (text + metadata)
Daily storage: 150M × 200B = 30GB
Yearly storage: 30GB × 365 ≈ 11TB

Read:write ratio = 100:1 (timeline reads >> tweet writes)
Read QPS: 500,000/sec
```

> Numbers don't need to be exact. They show you think about scale.

### S — Storage Schema (5 min)
Choose databases and design schemas:

- **Relational (PostgreSQL/MySQL):** users, tweets, follows — structured data with relationships.
- **NoSQL (Cassandra/DynamoDB):** timelines, feeds — high write throughput, wide rows.
- **Object storage (S3):** images, videos — large blobs.
- **Cache (Redis):** hot timelines, session data.

Design the key tables/collections. For relational:
```sql
users (id, username, email, created_at)
tweets (id, user_id, content, created_at)
follows (follower_id, followee_id, created_at)
```

### H — High-Level Design (10 min)
Draw the box-and-arrow diagram. Start simple:

```
Client (Web/Mobile)
       │
       ▼
  Load Balancer
       │
       ▼
  API Servers (stateless, horizontally scalable)
       │
  ┌────┼────┐
  ▼    ▼    ▼
 DB   Cache  Queue
```

Get a working design first, then add complexity.

### A — API Design (5 min)
Define the endpoints:

```
POST /api/v1/tweets          — create tweet
GET  /api/v1/tweets/{id}     — get tweet
GET  /api/v1/timeline        — get home timeline
POST /api/v1/follows         — follow a user
DELETE /api/v1/follows/{id}  — unfollow
```

Specify request/response formats, pagination, authentication.

### D — Detailed Design (15 min)
Deep dive into the most interesting/complex parts:
- How does the timeline generation work? (Push vs Pull model)
- How do we handle the fan-out problem?
- How do we shard the database?
- What happens when a cache miss occurs?

This is where you demonstrate **depth** — the key differentiator for senior roles.

### E — Edge Cases & Errors (5 min)
- What if the database goes down? (Replication, failover)
- What if a cache is full? (Eviction policy)
- What if a message is lost? (At-least-once delivery)
- What about thundering herd? (Cache stampede prevention)
- Hot keys? (Consistent hashing, replicated reads)

### D — Discussion (5 min)
- What are the bottlenecks?
- What would you do differently with 10x budget?
- Alternative approaches and why you didn't choose them.
- Monitoring and alerting.

---

## Key Metrics to Know (MEMORIZE)

### Latency Numbers Every Engineer Should Know
```
L1 cache reference ................. 0.5 ns
Branch mispredict .................. 5 ns
L2 cache reference ................. 7 ns
Mutex lock/unlock ................ 25 ns
Main memory reference ............. 100 ns
Compress 1KB with Snappy ........ 3,000 ns (3 µs)
Send 1KB over 1Gbps network ...... 10,000 ns (10 µs)
SSD random read ................ 150,000 ns (150 µs)
Read 1MB from SSD ............. 1,000,000 ns (1 ms)
Round trip in datacenter ........ 500,000 ns (0.5 ms)
Read 1MB from disk .......... 20,000,000 ns (20 ms)
Send packet CA→Netherlands→CA . 150,000,000 ns (150 ms)
```

### Availability Numbers
```
99%       = 3.65 days downtime/year
99.9%     = 8.76 hours downtime/year    (3 nines)
99.99%    = 52.6 minutes downtime/year   (4 nines)
99.999%   = 5.26 minutes downtime/year   (5 nines)
99.9999%  = 31.5 seconds downtime/year   (6 nines)
```

### Useful Numbers
- 1 day = 86,400 seconds
- 1 KB = 10^3 bytes, 1 MB = 10^6, 1 GB = 10^9, 1 TB = 10^12, 1 PB = 10^15
- 1 KB (binary) = 1024 bytes (KiB), but use 10^3 for estimation
- Web page load budget: ~1-3 seconds
- API response budget: <200ms for interactive, <50ms for sub-resource

---

## Go Connection
As a Go developer, you already understand:
- **Concurrency:** goroutines for handling thousands of concurrent requests.
- **Networking:** `net/http`, gRPC, context for timeouts/cancellation.
- **Services:** you've built microservices in Go.
- **Tooling:** Docker, Kubernetes, Prometheus — all Go-based.

Throughout this plan, we'll connect SD concepts to Go implementations you already know.

---

## Exercise: Practice the Framework

Take this prompt: **"Design a URL Shortener (like Bitly)."**

Spend **30 minutes** going through RESHADED:
1. **R:** What questions would you ask? (What's the expected QPS? Custom URLs? Analytics? Expiry?)
2. **E:** Estimate: if 100M URLs shortened/month, what's the storage? QPS?
3. **S:** What database? Schema?
4. **H:** Draw the high-level architecture.
5. **A:** What API endpoints?
6. **D:** How would you generate short codes? How handle collisions?
7. **E:** What if the DB is down? Cache misses?
8. **D:** What are the trade-offs of your short-code generation approach?

Don't code — just think and sketch. We'll do the full solution on Day 22.

---

## Common Mistakes
- **Jumping to the solution** without clarifying requirements. This is the #1 failure mode.
- **No estimation** — interviewers want to see you think about scale.
- **Drawing a diagram with 50 boxes** in the first 5 minutes — start simple.
- **Not discussing trade-offs** — every choice should have a "because" and a "trade-off is..."
- **Silence** — the interviewer can't evaluate your thinking if you don't speak.

## Checklist Before Moving On
- [ ] Memorized the RESHADED framework
- [ ] Memorized key latency numbers (at least: memory, SSD, network RTT, disk)
- [ ] Memorized availability numbers (3 nines through 5 nines)
- [ ] Practiced the URL shortener exercise using the framework
- [ ] Understand the difference between LLD and System Design
