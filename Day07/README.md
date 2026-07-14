# Day 7 - Review & Capacity Estimation Practice

## Topic
**Consolidation day.** Review Week 1 concepts and practice **back-of-envelope estimation** — the skill of sizing systems with rough calculations. This is tested in nearly every SD interview.

---

## Why This Day Matters
- Capacity estimation is a **distinct interview skill** — it's not about exact numbers, it's about order-of-magnitude reasoning.
- Week 1 introduced foundational concepts; without review, they'll blur before Week 4's full designs.
- Interviewers use estimation to check if you understand scale. "How much storage for 5 years?" — you need to answer in 2 minutes.

---

## Part 1: Week 1 Review

### Quick-Answer Drill (answer in 10 seconds each)

1. What does CAP stand for? Which two can you guarantee?
2. Is Cassandra CP or AP? Is MongoDB CP or AP?
3. What's the latency of: L1 cache? Main memory? SSD read? Network RTT in datacenter? Disk seek?
4. Layer 4 vs Layer 7 load balancing — what's the difference?
5. When would you use Redis vs PostgreSQL vs S3?
6. What does a CDN do? What's a cache hit vs cache miss?
7. Vertical vs horizontal scaling — which is preferred and why?
8. What does ACID stand for? What does BASE stand for?
9. What's the difference between strong and eventual consistency?
10. What does the RESHADED framework stand for?

<details>
<summary>Quick Answers</summary>

1. Consistency, Availability, Partition Tolerance. Guarantee 2 of 3 (P is mandatory, so really CP or AP).
2. Cassandra: AP. MongoDB: CP.
3. L1: 0.5ns. Memory: 100ns. SSD: 150µs. Network RTT: 0.5ms. Disk: 20ms.
4. L4: TCP/UDP level (IP+port). L7: HTTP level (URL, headers, body).
5. Redis: caching/sessions. PostgreSQL: relational/transactional data. S3: large blobs (images/videos).
6. CDN caches content at edge nodes. Hit: content in cache. Miss: fetch from origin.
7. Horizontal — no limits, fault-tolerant, commodity hardware. Vertical hits hardware limits.
8. ACID: Atomicity, Consistency, Isolation, Durability. BASE: Basically Available, Soft State, Eventually Consistent.
9. Strong: all reads see latest write. Eventual: reads may be stale temporarily, converge over time.
10. Requirements, Estimation, Storage schema, High-level design, API design, Detailed design, Edge cases, Discussion.
</details>

---

## Part 2: Capacity Estimation

### The Estimation Framework (MEMORIZE)

```
1. Estimate the number of users
   → DAU (Daily Active Users), MAU (Monthly Active Users)

2. Estimate requests per second (QPS)
   → QPS = (DAU × actions_per_user_per_day) / 86400
   → Peak QPS = avg QPS × 2-3 (traffic isn't uniform)

3. Estimate storage
   → Per-day storage = (new items/day) × (size per item)
   → Per-year storage = per-day × 365
   → Add replication factor (×2 or ×3)

4. Estimate bandwidth
   → Upload bandwidth = (new items/sec) × (size per item)
   → Download bandwidth = upload × read:write ratio

5. Estimate memory (cache)
   → Cache what? (e.g., top 20% of content = 80% of traffic)
   → Cache size = (items to cache) × (size per item)
   → Daily cache capacity, plus overhead
```

### Key Numbers to Remember
```
Time:
  1 day = 86,400 seconds ≈ 100,000 (round up for estimates)
  1 month ≈ 2.5 million seconds
  1 year = 365 days

Sizes:
  1 KB = 10^3 bytes
  1 MB = 10^6 bytes
  1 GB = 10^9 bytes
  1 TB = 10^12 bytes
  1 PB = 10^15 bytes

Rule of thumb:
  1 million users ≈ 100K DAU (10% of registered)
  Read:write ratio: typically 10:1 to 100:1 for most apps
  Peak traffic = 2-3x average
```

---

## Worked Example: Twitter Estimation

**Assumptions:**
- 200M MAU, 50M DAU
- Each user tweets 5 times/day on average
- Each tweet: 200 bytes (text + metadata)
- Read:write ratio = 100:1 (timeline views >> tweet writes)
- 10% of tweets have images (500KB each)
- Retain data for 5 years

### Step 1: QPS
```
Tweets per day: 50M × 5 = 250M tweets/day
Write QPS (avg): 250M / 86400 ≈ 2,900 tweets/sec
Write QPS (peak): 2,900 × 3 ≈ 9,000 tweets/sec

Timeline views per day: 250M × 100 = 25B views/day
Read QPS (avg): 25B / 86400 ≈ 290,000 reads/sec
Read QPS (peak): 290,000 × 3 ≈ 870,000 reads/sec
```

### Step 2: Storage
```
Text tweets per day: 250M × 200B = 50GB/day
Image tweets per day: 250M × 10% × 500KB = 12.5TB/day
Total per day: ~12.5TB (dominated by images)
Total per year: 12.5TB × 365 ≈ 4.6PB/year
Total for 5 years: ~23PB
With replication (×3): ~69PB
```

### Step 3: Bandwidth
```
Upload (writes): 12.5TB / 86400 ≈ 145MB/sec (ingress)
Download (reads): 145MB × 100 (read:write) = 14.5GB/sec (egress)
```

### Step 4: Cache
```
Cache top 20% of tweets (80% of reads):
  Active tweets to cache: 250M × 20% = 50M tweets
  Cache size: 50M × 200B = 10GB (text only)
  With image references: 10GB + 50M × 10% × 500KB = 10GB + 2.5TB ≈ 2.5TB
```

### Step 5: Servers
```
Assume 1 server handles 5,000 QPS (read):
  Read servers needed: 870,000 / 5,000 ≈ 174 servers
  With overhead (×2 for failover): ~350 servers

Write servers (heavier, 1,000 QPS):
  Write servers: 9,000 / 1,000 = 9
  With overhead: ~20 servers
```

---

## Exercise: Estimate WhatsApp

**Make your own assumptions and estimate:**

1. 1B MAU, 30% are DAU
2. Each DAU sends 50 messages/day
3. Each message: 100 bytes (text) average
4. 20% of messages are media (avg 1MB each)
5. Messages retained for 1 year
6. Read:write = 1:1 (messaging is symmetric)
7. Replication factor: 3

Calculate:
- Write QPS (avg and peak)
- Read QPS (avg and peak)
- Storage per day, per year (with replication)
- Bandwidth (upload and download)
- Cache size (if caching 1 day of messages)

<details>
<summary>Reference Answer</summary>

```
DAU: 1B × 30% = 300M
Messages/day: 300M × 50 = 15B messages/day

Write QPS (avg): 15B / 86400 ≈ 174,000/sec
Write QPS (peak): 174,000 × 3 ≈ 522,000/sec
Read QPS = Write QPS (1:1 ratio) = 522,000/sec peak

Text messages/day: 15B × 80% × 100B = 120GB/day
Media messages/day: 15B × 20% × 1MB = 3,000TB/day = 3PB/day
Total/day: ~3PB (dominated by media)
Total/year: 3PB × 365 ≈ 1,095PB ≈ 1.1EB (exabyte!)
With replication (×3): ~3.3EB

Bandwidth:
  Upload: 3PB / 86400 ≈ 34.7GB/sec
  Download: same (1:1 ratio) = 34.7GB/sec

Cache (1 day of text messages):
  120GB (just text, not media)
```

**Key insight:** Media dominates storage and bandwidth. Text is tiny. This is why WhatsApp limits media size and auto-deletes old media.
</details>

---

## Part 3: Build Your Cheat Sheet

Start a `cheatsheet.md` in `system-design-prep/`. Add:

```
## RESHADED Framework
R - Requirements (functional + non-functional)
E - Estimation (QPS, storage, bandwidth)
S - Storage schema (DB choices, schema)
H - High-level design (box-and-arrow)
A - API design (endpoints, formats)
D - Detailed design (deep dive)
E - Edge cases & errors
D - Discussion (trade-offs)

## Latency Numbers
L1: 0.5ns | Memory: 100ns | SSD: 150µs | Network RTT: 0.5ms | Disk: 20ms

## Availability
99.9% = 8.76h downtime/year | 99.99% = 52.6 min | 99.999% = 5.26 min

## CAP Theorem
CP: consistency + partition tolerance (banks, MongoDB)
AP: availability + partition tolerance (social, Cassandra)
P is not optional — real choice is CP vs AP

## Database Choices
SQL (PostgreSQL): ACID, relational, transactions
Redis: cache, sessions, real-time
Cassandra: high-write, time-series, timelines
MongoDB: flexible schema, documents
S3: object storage, images, videos
Elasticsearch: full-text search

## Storage Tiers
Hot: Redis, SSD | Warm: DB on HDD | Cold: S3/Glacier

## Scaling
Vertical: bigger machine (limited, SPOF)
Horizontal: more machines (stateless, preferred)
LB algorithms: round-robin, least-connections, IP-hash, consistent-hashing
```

---

## Checklist Before Moving On
- [ ] Answered all 10 quick-answer questions
- [ ] Can estimate QPS, storage, bandwidth for any system
- [ ] Memorized key numbers (latency, availability, sizes)
- [ ] Completed the WhatsApp estimation exercise
- [ ] Started your system design cheat sheet
- [ ] Ready for Week 2 (caching, MQs, API design, microservices)
