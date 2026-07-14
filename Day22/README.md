# Day 22 - Full System Design: URL Shortener (Bitly)

## Topic
**Design a URL Shortener** (like Bitly or TinyURL). The most classic system design interview question — and a great first full design to practice.

---

## Why This Problem Matters
- It's the #1 most-asked system design question for screening rounds.
- It's simple enough to complete in 45 minutes but tests all core concepts: estimation, database choice, caching, scaling.
- It introduces the concept of **encoding** (generating short codes) — a recurring pattern.

---

## RESHADED Framework

### R — Requirements (5 min)

**Functional:**
1. Given a long URL, return a short URL (e.g., `bit.ly/abc123`).
2. Given a short URL, redirect to the original long URL.
3. Short URLs should be unique and not guessable.
4. Custom aliases (user chooses the short code, e.g., `bit.ly/my-promo`).
5. Analytics: click count, referral source (stretch goal).

**Non-functional:**
1. High availability (shortlinks must always redirect).
2. Low latency (< 50ms for redirect).
3. Short codes: 6-8 characters.
4. URLs don't expire (or optionally expire after X time).

**Clarifying questions to ask:**
- "How many URLs will be shortened per day?" → Assume 100M/month.
- "What's the read:write ratio?" → 100:1 (reads dominate).
- "Should URLs expire?" → No, permanent.
- "Do users need accounts?" → Optional for custom aliases.

### E — Estimation (3 min)

```
Writes: 100M URLs/month → ~40 URLs/sec (avg), ~120/sec (peak)
Reads: 100:1 ratio → 4,000 redirects/sec (avg), 12,000/sec (peak)

Storage per URL: ~500 bytes (short code + long URL + metadata)
Monthly storage: 100M × 500B = 50GB/month
5-year storage: 50GB × 60 = 3TB

Bandwidth:
  Write: 40/sec × 500B = 20KB/sec (negligible)
  Read: 4,000/sec × 500B = 2MB/sec
  Peak read: 12,000/sec × 500B = 6MB/sec
```

### S — Storage Schema (5 min)

**Database choice:** NoSQL (DynamoDB/Cassandra) — key-value access pattern, high read QPS, no JOINs needed.

**Schema:**
```
Table: url_mappings
  short_code (PK): "abc123"
  long_url: "https://www.example.com/very/long/url"
  user_id: 42 (optional, for custom aliases)
  created_at: 1642248000
  click_count: 0
```

**Also needed:**
- Redis (cache): cache `short_code → long_url` for hot URLs (reduces DB load).
- S3 (optional): store analytics events (click logs).

### H — High-Level Design (10 min)

```
┌─────────┐       ┌──────────┐       ┌───────────┐
│  Client  │──────→│ API Server│──────→│  ID Gen   │
│ (Browser)│       │(Go, horiz)│       │  Service  │
└─────────┘       └─────┬────┘       └───────────┘
      ↑                   │
      │                   ├──→ Redis (cache)
      │                   │
      │                   └──→ DynamoDB (persistent)
      │
      │   Redirect (302)
      │
┌─────┴───┐
│  Client  │  clicks bit.ly/abc123
└─────────┘
      │
      ▼
┌──────────┐       ┌───────┐       ┌───────────┐
│ Load Bal. │──────→│ API   │──────→│  Redis    │ (cache hit?)
└──────────┘       │ Server│       └─────┬─────┘
                   └───┬───┘             │
                       │          miss   │
                       ▼                 ▼
                  ┌──────────┐    ┌───────────┐
                  │ DynamoDB │←───│  (query)   │
                  └──────────┘    └───────────┘
                       │
                       ▼ (found)
                  Return 301/302 redirect
```

### A — API Design (5 min)

```
POST /api/v1/shorten
  Body: { "long_url": "https://example.com/...", "custom_alias": "my-promo" (optional) }
  Response: 201 Created
  { "short_url": "https://bit.ly/abc123", "long_url": "https://example.com/..." }

GET /{short_code}
  → 301 Moved Permanently (permanent redirect, browser caches)
  → Location: https://example.com/...
  OR
  → 302 Found (temporary redirect, allows analytics tracking)
  → Location: https://example.com/...

GET /api/v1/urls/{short_code}
  Response: 200 OK
  { "short_code": "abc123", "long_url": "...", "click_count": 1234, "created_at": "..." }

DELETE /api/v1/urls/{short_code}
  → 204 No Content
```

**301 vs 302 redirect:**
- **301 (permanent):** Browser caches the redirect → subsequent clicks don't hit our server. Reduces load but we can't track clicks.
- **302 (temporary):** Browser doesn't cache → every click hits our server → we can track clicks. Higher load.
- **Bitly uses 301** for non-analytics, **302** for analytics-enabled links.

### D — Detailed Design (15 min)

#### Short Code Generation (THE core problem)

**Option 1: Hashing + Encoding**
```
1. Take the long URL.
2. Hash it (MD5 or SHA-256).
3. Encode the hash in base62 (a-z, A-Z, 0-9 = 62 chars).
4. Take the first 6-7 characters.
```
```
MD5("https://example.com/very/long") = 5d41402abc4b2a76b9719d911017c592
Base62 encode first 8 bytes → "aB3x9K"
```

**Problem:** Hash collisions (two URLs might produce the same short code). Solution: check DB, if collision, append a random suffix and rehash.

**Option 2: Auto-Increment ID + Base62**
```
1. Insert the long URL into DB → get auto-increment ID (e.g., 123456789).
2. Convert ID to base62: 123456789 → "8M0kx"
3. Short URL: bit.ly/8M0kx
```

**Pros:** No collisions. Deterministic. Short codes are sequential (might be guessable — a security concern).
**Cons:** Single DB = bottleneck for ID generation. Need distributed ID generation.

**Option 3: Pre-Generated Keys (Key Generation Service)**
```
1. A separate service generates 6-character random strings in advance.
2. Stores them in a "available keys" table/queue.
3. When a URL is shortened, pop a key from the queue.
```

**Pros:** No collisions, no real-time generation, distributed (each server gets a batch of keys).
**Cons:** Extra service. Keys must be pre-generated.

**Recommended for interview:** Option 2 (auto-increment + base62) with **Snowflake-style distributed ID** for uniqueness across shards.

#### Base62 Encoding
```go
func Base62Encode(n int64) string {
    const chars = "0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
    if n == 0 { return "0" }
    result := ""
    for n > 0 {
        result = string(chars[n%62]) + result
        n /= 62
    }
    return result
}

// 6 characters of base62 = 62^6 = 56 billion combinations (enough for years)
```

#### Caching Strategy
```
GET /abc123:
1. Check Redis: GET url:abc123
   → HIT: return long URL (sub-ms)
   → MISS: query DynamoDB → store in Redis (TTL 1 hour) → return

Hot URLs (viral links): always in cache.
Cold URLs: cache miss → DB lookup (slower, but acceptable).
```

#### Sharding
If DynamoDB gets overwhelmed:
- Shard by `short_code` (hash-based).
- Each shard holds a subset of URLs.
- Consistent hashing for even distribution.

#### Analytics (Stretch)
```
On each redirect:
  → Publish event to Kafka: { short_code, user_ip, user_agent, timestamp, referrer }
  → Consumer aggregates → updates click_count in DB
  → Dashboard queries: "top URLs today", "clicks by country"
```

### E — Edge Cases (5 min)
- **Invalid URL:** validate the long URL before shortening (format check, DNS lookup optional).
- **Custom alias already taken:** return 409 Conflict.
- **Short code not found:** return 404.
- **Abuse:** rate-limit URL creation (prevent spam). Block known malicious URLs (use a URL safety API).
- **URL already shortened:** check if long URL already exists → return existing short code (dedup).

### D — Discussion (5 min)
- **Bottleneck:** Database reads. Solution: Redis cache (95%+ hit rate for popular URLs).
- **Availability:** Multi-region deployment (DNS geo-routing). DynamoDB is globally replicated.
- **Alternative:** If analytics isn't needed, use 301 redirects (browser caches → no server load).
- **Scaling writes:** If 100M/month grows to 1B, use Snowflake IDs + sharded DB.

---

## Exercise: Extend the URL Shortener

Add these features and discuss the design changes:

1. **Expiring URLs:** Short links that stop working after 7 days.
2. **Password-protected links:** User must enter a password before redirect.
3. **QR codes:** Generate a QR code for each short URL.
4. **A/B testing:** Redirect to different long URLs based on user segment.

> Think about: What changes to the schema? Caching? API? Architecture?

---

## Common Mistakes
- Not handling hash collisions in short code generation.
- Using 301 redirect when analytics is required (browser caches, no tracking).
- Forgetting to cache redirects (every request hits DB → slow + expensive).
- Making short codes guessable (sequential IDs without hashing).
- Not considering URL validation (malicious URLs, broken links).

## Checklist Before Moving On
- [ ] Can estimate storage/QPS for a URL shortener
- [ ] Know 3 approaches to short code generation (hash, auto-increment+base62, pre-generated keys)
- [ ] Understand 301 vs 302 redirect trade-off
- [ ] Can design caching for redirects (Redis cache-aside)
- [ ] Solved the URL Shortener using full RESHADED framework
