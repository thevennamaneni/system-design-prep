# Day 8 - Caching: Strategies, Eviction & Cache Patterns

## Topic
**Caching** — the most powerful performance optimization in distributed systems. Cover cache patterns (cache-aside, write-through, write-back), eviction policies, and distributed caching with Redis.

---

## Why It Matters
- Caching is the answer to 80% of performance problems. Interviewers expect you to propose it instinctively.
- "How do you reduce database load?" → caching. "How do you achieve <10ms latency?" → caching.
- Cache invalidation is one of the "two hard problems in computer science." Understanding trade-offs shows seniority.

---

## 1. Why Cache?

```
Without cache: DB read = 10-50ms → 10,000 QPS → DB overwhelmed
With cache:    Cache hit = 0.1ms → 95% hit rate → DB load = 500 QPS → DB happy
```

### The Locality Principle
- **Temporal locality:** Data accessed recently will be accessed again soon (user's timeline).
- **Spatial locality:** Data near recently accessed data will be accessed (user's followers' tweets).

---

## 2. Cache Patterns (MEMORIZE)

### a) Cache-Aside (Lazy Loading)
```
1. App checks cache for data
2. Cache HIT → return data
3. Cache MISS → read from DB → write to cache → return data
```

```
┌──────┐   GET key    ┌───────┐
│ App  │ ────────────→│ Cache │
│      │ ←── MISS ────│       │
│      │              └───────┘
│      │   SELECT ...  ┌───────┐
│      │ ────────────→│  DB   │
│      │ ←── data ────│       │
│      │              └───────┘
│      │   SET key     ┌───────┐
│      │ ────────────→│ Cache │
└──────┘              └───────┘
```

**Pros:**
- Simple, widely used.
- Cache only contains requested data (no wasted space).
- If cache crashes, system still works (just slower).

**Cons:**
- Cache miss = 3 round trips (cache check, DB read, cache write).
- Stale data: if DB is updated externally, cache has old value until TTL expires.
- Thundering herd on cache miss (many requests hit DB simultaneously).

### b) Write-Through
```
1. App writes to cache
2. Cache synchronously writes to DB
3. Return success
```

```
┌──────┐   WRITE      ┌───────┐
│ App  │ ────────────→│ Cache │
│      │ ←── OK ──────│       │
│      │              │   │   │
│      │              │   ▼   │ (sync write)
│      │              │ ┌─────┐
│      │              │ │ DB  │
│      │              │ └─────┘
└──────┘              └───────┘
```

**Pros:**
- Cache is always consistent with DB — no stale data.
- Read is always a cache hit (data is already cached).

**Cons:**
- Higher write latency (write to cache + DB synchronously).
- Cache stores everything (even rarely-read data) — wasted memory.

### c) Write-Back (Write-Behind)
```
1. App writes to cache
2. Cache returns success immediately
3. Cache asynchronously writes to DB (after a delay or periodically)
```

**Pros:**
- Very fast writes (only one operation — cache write).
- Write batching: multiple writes to same key → one DB write.

**Cons:**
- **Data loss risk:** If cache crashes before writing to DB, data is lost.
- Complex to implement.
- Use cases: write-heavy, loss-tolerant data (analytics, counters).

### d) Write-Around
```
1. App writes directly to DB (bypass cache)
2. Cache is populated only on read (cache-aside on miss)
```

**Pros:** Cache isn't polluted with write-once, never-read data.
**Cons:** First read is always a miss.
**Use cases:** Logging, write-heavy data that's rarely read back immediately.

### Summary Table

| Pattern | Write Latency | Read Latency | Consistency | Data Loss Risk |
|---------|--------------|-------------|-------------|----------------|
| Cache-Aside | N/A (read pattern) | Miss: slow, Hit: fast | Eventual (stale) | None |
| Write-Through | High (cache + DB) | Always fast | Strong | None |
| Write-Back | Very fast (cache only) | Always fast | Eventual | Possible |
| Write-Around | Low (DB only) | First read slow | Eventual | None |

---

## 3. Cache Eviction Policies

When the cache is full, what do we remove?

| Policy | What it removes | Best for |
|--------|----------------|----------|
| **LRU** (Least Recently Used) | Item not accessed for longest time | General purpose, temporal locality |
| **LFU** (Least Frequently Used) | Item with fewest accesses | Stable popularity (popular items stay) |
| **FIFO** (First In First Out) | Oldest item | Simple, rarely used |
| **TTL** (Time To Live) | Item past expiration | Time-sensitive data (sessions, news) |
| **Random** | Random item | Simple, surprisingly effective |

**LRU is the most common in interviews.** Redis uses an approximation of LRU.

---

## 4. Distributed Caching with Redis

### What is Redis?
- In-memory key-value store.
- Single-threaded (mostly) — no lock contention.
- Supports data structures: strings, lists, sets, sorted sets, hashes, streams.
- Persistence options: RDB (snapshots), AOF (append-only log), both.

### Redis in System Design
```
┌──────┐     ┌──────────┐     ┌───────┐
│ App  │ ──→ │  Redis   │ ←── │ App   │
│Server│     │ Cluster  │     │Server │
└──────┘     └────┬─────┘     └───────┘
                  │
          ┌───────┼───────┐
          ▼       ▼       ▼
       Node1   Node2   Node3  (sharded by consistent hashing)
```

### Redis Use Cases
| Use Case | Redis Structure |
|----------|----------------|
| Session store | `SET session:abc "data" EX 3600` |
| Cache | `GET user:123` → JSON |
| Leaderboard | Sorted Set (`ZADD`, `ZREVRANGE`) |
| Real-time analytics | Counters (`INCR`) |
| Rate limiting | Sliding window with sorted sets |
| Pub/Sub | `PUBLISH`/`SUBSCRIBE` |
| Geospatial | `GEOADD`, `GEORADIUS` (find nearby drivers) |
| Message queue | Lists (`LPUSH`/`RPOP`) or Streams |

### Redis Persistence Trade-offs
| Mode | Speed | Durability |
|------|-------|-----------|
| No persistence (cache only) | Fastest | Data lost on restart |
| RDB (snapshots) | Fast | Lose data since last snapshot |
| AOF (append log) | Slower | Lose at most 1 second |
| RDB + AOF | Slower | Best durability |

---

## 5. Cache Challenges

### Cache Stampede (Thundering Herd)
**Problem:** A popular cache key expires. 1000 requests simultaneously miss, hit the DB, and rebuild the cache — DB is overwhelmed.

**Solutions:**
1. **Locking:** First request acquires a lock, fetches from DB, populates cache. Others wait.
2. **Early expiration:** Refresh cache before TTL expires (refresh-ahead).
3. **Jittered TTL:** Add random jitter to TTL so keys don't expire simultaneously.

```go
// Locking solution in Go
func getCached(key string) (string, error) {
    val, err := redis.Get(key)
    if err == nil { return val, nil }  // cache hit

    // Cache miss — acquire lock
    locked := redis.SetNX("lock:"+key, "1", 10*time.Second)
    if !locked {
        time.Sleep(50 * time.Millisecond)  // wait for other request to populate
        return redis.Get(key)  // retry
    }
    defer redis.Del("lock:" + key)

    val = db.Query(key)  // fetch from DB
    redis.Set(key, val, 5*time.Minute)  // populate cache
    return val, nil
}
```

### Cache Penetration
**Problem:** Requests for non-existent keys bypass cache every time → DB hammered by useless queries.

**Solution:** Cache null results with short TTL:
```go
redis.Set("user:nonexistent", "NULL", 60*time.Second)
```

### Cache Avalanche
**Problem:** Many cache keys expire simultaneously → flood of DB queries.

**Solution:** Add random jitter to TTLs:
```go
ttl := 300 + rand.Intn(60)  // 300-360 seconds
redis.Set(key, val, time.Duration(ttl)*time.Second)
```

### Hot Keys
**Problem:** One key is accessed so frequently that a single Redis node is overwhelmed.

**Solution:** Replicate the hot key to multiple nodes; route reads randomly.

---

## Go Connection

```go
import "github.com/redis/go-redis/v9"

// Redis client
rdb := redis.NewClient(&redis.Options{
    Addr:         "localhost:6379",
    PoolSize:     100,
    MinIdleConns: 10,
})

// Cache-aside pattern
func GetUser(ctx context.Context, id int) (*User, error) {
    key := fmt.Sprintf("user:%d", id)
    
    // Check cache
    if data, err := rdb.Get(ctx, key).Bytes(); err == nil {
        var user User
        json.Unmarshal(data, &user)
        return &user, nil  // cache hit
    }
    
    // Cache miss — read from DB
    user, err := db.GetUser(id)
    if err != nil { return nil, err }
    
    // Populate cache (with jittered TTL)
    data, _ := json.Marshal(user)
    ttl := 300 + rand.Intn(60)
    rdb.Set(ctx, key, data, time.Duration(ttl)*time.Second)
    
    return user, nil
}
```

---

## Exercise: Design a Cache Layer for a News Feed

**Requirements:**
1. 10M users, each viewing their feed 10x/day.
2. Feed generation from DB takes 500ms; cache hit takes 5ms.
3. Target: 95% cache hit rate, <50ms p99 latency.
4. Feeds update when followees post new content.

**Design:**
1. Which cache pattern? (cache-aside, write-through, write-back?)
2. What's the cache key? What's the TTL?
3. How do you handle cache invalidation when a new tweet is posted?
4. How do you prevent cache stampede?
5. Estimate cache size.

<details>
<summary>Reference Answer</summary>

1. **Cache-aside:** Feeds are read-heavy, and we don't want to cache every user's feed (write-through would populate cache for inactive users). Generate on first miss, cache with TTL.

2. **Key:** `feed:user:{userID}`, **TTL:** 5 minutes (feeds are somewhat real-time; stale for a few minutes is acceptable).

3. **Invalidation:** When a user posts, **push** the tweet to all followers' cached feeds (if cached):
   - For each follower: `LPUSH feed:user:{followerID} {tweetID}` (prepend to list).
   - Alternatively, **invalidate** followers' feeds: `DEL feed:user:{followerID}` — they'll regenerate on next read.
   - Trade-off: Push = faster reads, more write work. Pull (invalidate) = slower reads, simpler writes.

4. **Stampede prevention:** Use locking (SETNX) — first miss acquires a lock, fetches from DB, populates cache. Others wait 50ms and retry.

5. **Cache size:** 10M users × average feed size (100 tweets × 200 bytes = 20KB) = 200GB. Only cache active users (30% = 3M) → 60GB. With Redis, this is feasible across a small cluster.
</details>

---

## Common Mistakes
- Caching everything — wastes memory and creates invalidation headaches. Cache only hot data.
- No TTL — stale data forever. Always set a TTL.
- Not handling cache failures — if Redis is down, the system should degrade to DB reads, not crash.
- Forgetting cache stampede protection on hot keys.
- Not discussing read-your-writes consistency (user posts, then doesn't see their own post because cache is stale).

## Checklist Before Moving On
- [ ] Know the 4 cache patterns and their trade-offs
- [ ] Can explain LRU vs LFU vs TTL eviction
- [ ] Understand cache stampede, penetration, and avalanche + solutions
- [ ] Know Redis use cases beyond simple caching (leaderboards, rate limiting, geospatial)
- [ ] Solved the News Feed caching exercise
