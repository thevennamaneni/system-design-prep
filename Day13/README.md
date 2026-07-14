# Day 13 - Database Scaling: Sharding, Replication & Partitioning

## Topic
**Database scaling** — replication (read replicas), **sharding** (horizontal partitioning), consistent hashing, and the challenges of distributed data. This is how you scale a database beyond a single machine.

---

## Why It Matters
- "Your database is the bottleneck" is the most common scaling problem. Knowing how to shard is essential.
- Sharding is asked in nearly every senior-level system design interview.
- Consistent hashing is a fundamental algorithm used in sharding, load balancing, and caching.

---

## 1. The Database Scaling Problem

```
Single database:
  - Writes: 5,000/sec (max on one machine)
  - Reads: 50,000/sec (with caching)
  - Storage: 10TB (disk limit)

When you exceed these → you must scale the database.
```

### Scaling Approaches (in order of complexity)
1. **Vertical scaling:** Bigger machine (limited, expensive).
2. **Read replicas:** Copy data to read-only replicas (helps reads, not writes).
3. **Caching:** Redis in front of DB (helps reads, not writes).
4. **Sharding:** Split data across multiple machines (scales both reads AND writes).
5. **Federation/Functional partitioning:** Split by function (users DB, orders DB, etc.).

---

## 2. Replication

**What it is:** Copying data from a primary (master) database to one or more replicas (slaves).

### Master-Slave (Primary-Replica) Replication
```
                    ┌──────────┐
Writes ────────────→│  Master   │
                    │  (RW)     │
                    └────┬─────┘
                         │ (async replication)
              ┌──────────┼──────────┐
              ▼          ▼          ▼
         ┌────────┐ ┌────────┐ ┌────────┐
         │Replica1│ │Replica2│ │Replica3│
         │  (RO)  │ │  (RO)  │ │  (RO)  │
         └────────┘ └────────┘ └────────┘
              ▲          ▲          ▲
              │          │          │
Reads ───────┘──────────┘──────────┘
```

- **Master:** Handles all writes (INSERT, UPDATE, DELETE). Single writer.
- **Replicas:** Handle reads (SELECT). Multiple readers.
- **Replication lag:** Replicas are updated asynchronously (ms to seconds behind master).

**Pros:**
- Scales reads (add more replicas).
- Read replicas can be in different regions (lower latency for global users).
- Failover: if master dies, promote a replica.

**Cons:**
- Doesn't scale writes (still one master).
- Replication lag → stale reads.
- Master is a single point of failure (until failover completes).

### Master-Master (Multi-Master) Replication
```
┌──────────┐               ┌──────────┐
│ Master 1  │ ←── synch ──→│ Master 2  │
│  (RW)     │               │  (RW)     │
└──────────┘               └──────────┘
     ▲                           ▲
Writes                        Writes
```

- Both masters accept writes.
- Writes are replicated bidirectionally.

**Pros:**
- Scales writes (two masters).
- Both can serve reads.
- Region-local writes (lower latency).

**Cons:**
- **Conflict resolution:** What if both masters update the same row simultaneously?
- Complex to set up and maintain.
- Risk of split-brain (network partition → both think they're the only master).

**Use cases:** Multi-region deployments (write in US and EU simultaneously). Usually avoided unless necessary.

### Synchronous vs Asynchronous Replication

| Type | How it works | Latency | Data loss risk |
|------|-------------|---------|----------------|
| **Synchronous** | Master waits for replica to confirm before acknowledging write | High (waits for replica) | None |
| **Asynchronous** | Master acknowledges write immediately; replica catches up later | Low | Possible (if master crashes before replication) |

**PostgreSQL:** Supports both. Synchronous = `synchronous_commit = on`. Asynchronous = `synchronous_commit = off`.
**MySQL:** Async by default. Group replication for sync.
**Cassandra:** Tunable consistency (choose per-query).

---

## 3. Sharding (Horizontal Partitioning)

**What it is:** Splitting a large table across multiple database instances. Each instance holds a subset of the data (a "shard").

```
Before sharding (one DB):
  users table: 1 billion rows (too big for one machine)

After sharding (4 shards):
  Shard 0: users with id 0-250M
  Shard 1: users with id 250M-500M
  Shard 2: users with id 500M-750M
  Shard 3: users with id 750M-1B
```

### Sharding Strategies

#### a) Range-Based Sharding
Shard by a range of the key:
```
Shard 0: users with id 1 - 1,000,000
Shard 1: users with id 1,000,001 - 2,000,000
Shard 2: users with id 2,000,001 - 3,000,000
```

**Pros:** Easy to implement. Range queries are efficient (all in one shard).
**Cons:** Hotspots (latest IDs go to the last shard → uneven load).

#### b) Hash-Based Sharding
Shard by hash of the key:
```
shard = hash(user_id) % N
```

**Pros:** Even distribution (no hotspots).
**Cons:** Range queries must hit all shards. Adding shards requires rehashing (expensive).

#### c) Consistent Hashing (THE interview favorite)
Solves the "rehashing problem" of hash-based sharding.

**The Problem with `hash % N`:**
```
4 shards: hash(id) % 4 → shard 0, 1, 2, 3
Add a 5th shard: hash(id) % 5 → many keys move to different shards!
  → 80% of data must be redistributed. Terrible.
```

**Consistent Hashing Solution:**
```
1. Place shards on a "ring" (hash space 0 to 2^32)
2. Key maps to a position on the ring (hash(key))
3. Key belongs to the first shard clockwise from its position

When adding a shard:
  → Only keys between the new shard and its predecessor move
  → ~1/N of keys move (not 80%+)
```

```
        0
      /   \
   Shard A  Shard D
    |        |
   Shard B  Shard C
      \   /
       2^32

Key hashes to position between Shard A and Shard B → goes to Shard B
Add Shard E between A and B → only keys between A and E move to E
```

**Used by:** Cassandra, DynamoDB, Redis Cluster, Memcached, Akamai CDN.

#### d) Directory-Based Sharding
A lookup table maps keys to shards:
```
Lookup: "user_123" → Shard 2
Lookup: "user_456" → Shard 0
```

**Pros:** Flexible (any mapping). Easy to move data (update lookup).
**Cons:** Lookup service is a SPOF (must be highly available).

### Sharding Challenges

#### Cross-Shard Queries
```
"Get all users who signed up in the last 24 hours"
  → Must query ALL shards → merge results → slow
```
**Solution:** Maintain a separate "signup_date" index in Elasticsearch or a summary table.

#### Distributed Transactions
```
"Transfer money from user on Shard 1 to user on Shard 2"
  → Requires a 2-phase commit (2PC) across shards → slow, complex
```
**Solution:** Avoid cross-shard transactions. Design shard key so related data is on the same shard.

#### Joining Across Shards
```
"Get user's tweets and their follower count"
  → Users on one shard, tweets on another → can't JOIN
```
**Solution:** Denormalize (store follower count in the tweet) or use a separate analytics DB.

#### Resharding (Adding Shards)
Adding a shard requires redistributing data. Consistent hashing minimizes this, but it's still complex.

---

## 4. Choosing a Shard Key (CRITICAL)

The shard key determines where each record lives. A bad shard key creates hotspots.

### Good Shard Key Properties
1. **High cardinality:** Many distinct values (user_id is good; gender is bad).
2. **Even distribution:** Data spreads evenly across shards.
3. **Monotonic writes avoided:** Don't use auto-incrementing IDs (all writes go to the latest shard).
4. **Query locality:** Frequently queried together data is on the same shard.

### Examples
| Data | Good Shard Key | Bad Shard Key |
|------|---------------|---------------|
| Users | `user_id` (hash) | `created_at` (time-based: latest shard is hot) |
| Tweets | `user_id` (user's tweets together) | `tweet_id` (range queries hit all shards) |
| Orders | `customer_id` | `order_id` (can't get all orders for a customer from one shard) |

**Key insight:** Shard by the entity that "owns" the data. For tweets, shard by `user_id` — all of a user's tweets are on the same shard, making "get user's tweets" a single-shard query.

---

## 5. Federation (Functional Partitioning)

Split the database by function, not by data range:
```
User DB (PostgreSQL): users, auth, profiles
Order DB (PostgreSQL): orders, payments
Analytics DB (Columnar DB): events, metrics
Media DB (S3): images, videos
```

**Pros:** Each database is smaller, simpler, independently scalable.
**Cons:** Cross-database queries require application-level joins.

---

## Go Connection

```go
// Sharding in Go: a simple sharded DB client
type ShardedDB struct {
    shards []*sql.DB
    numShards int
}

func (s *ShardedDB) GetShard(userID int) *sql.DB {
    shard := userID % s.numShards  // simple hash
    return s.shards[shard]
}

func (s *ShardedDB) GetUser(userID int) (*User, error) {
    shard := s.GetShard(userID)
    return shard.QueryRow("SELECT * FROM users WHERE id = $1", userID)
}

// Consistent hashing (using a library like github.com/stathat/consistent)
import "github.com/stathat/consistent"

c := consistent.New()
c.Add("shard1")
c.Add("shard2")
c.Add("shard3")

server, _ := c.Get(fmt.Sprintf("user:%d", userID))
// → "shard2"
```

---

## Exercise: Shard a Twitter-like System

**Requirements:**
- 500M users, 10B tweets, 500M follows
- Write QPS: 10,000 tweets/sec
- Read QPS: 500,000 timeline reads/sec
- Need to query: "get user's tweets" and "get user's timeline" efficiently

**Design:**
1. What shard key for the `tweets` table? Why?
2. What shard key for the `users` table?
3. How do you generate tweet IDs that work with sharding?
4. How do you handle "get timeline" (which requires tweets from many users)?
5. How many shards do you need?

<details>
<summary>Reference Answer</summary>

1. **Tweets shard key:** `user_id` (hash-based). All of a user's tweets are on the same shard → "get user's tweets" is a single-shard query. This is the most common read pattern.

2. **Users shard key:** `user_id` (hash-based). Each user's data is on one shard.

3. **Tweet ID generation:** Use **Snowflake** (Twitter's algorithm): combine timestamp + shard ID + sequence number into a 64-bit ID. This gives sortable IDs that encode which shard they came from.

4. **Timeline generation:** This is the hard part. A user's timeline = tweets from all their followees, which are on different shards.
   
   Two approaches:
   
   **a) Pull (Fan-out on Read):**
   - Get user's followee list.
   - Query each followee's shard for recent tweets.
   - Merge and sort by time.
   - Problem: if following 10,000 people, that's 10,000 shard queries. Slow.
   
   **b) Push (Fan-out on Write):**
   - When a user tweets, push the tweet to all followers' timeline caches (in Redis).
   - Timeline read = single Redis GET (pre-built).
   - Problem: celebrities with 50M followers → 50M writes per tweet. "Celebrity problem."
   
   **c) Hybrid (the real answer):**
   - For normal users: push to followers' timelines (write-heavy, but manageable).
   - For celebrities (> 100K followers): don't push. Followers pull the celebrity's tweets at read time and merge.
   
5. **Shard count:** Assume each shard handles 5,000 writes/sec. 10,000 writes/sec → 2 shards minimum. With replication and overhead: 10-20 shards for tweets. Plus read replicas for the 500K reads/sec.

**Key insight:** The timeline problem (fan-out) is the core architectural challenge of Twitter. The push/pull/hybrid decision is what interviewers want to hear you discuss.
</details>

---

## Common Mistakes
- Sharding by auto-increment ID → all writes go to the latest shard (hotspot).
- Sharding by a low-cardinality key (e.g., country) → uneven distribution.
- Not thinking about cross-shard queries — they're expensive.
- Forgetting that sharding doesn't solve write scaling if the shard key creates hotspots.
- Not knowing consistent hashing — it's the answer to "how do you add shards without rehashing everything?"

## Checklist Before Moving On
- [ ] Understand replication (master-slave, master-master, sync vs async)
- [ ] Know 4 sharding strategies (range, hash, consistent, directory)
- [ ] Can explain consistent hashing and why it's better than `hash % N`
- [ ] Know how to choose a shard key (cardinality, distribution, query locality)
- [ ] Understand the fan-out problem (push vs pull vs hybrid) for timelines
- [ ] Solved the Twitter sharding exercise
