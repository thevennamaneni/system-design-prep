# Day 27 - Full System Design: Rate Limiter & Notification System

## Topic
Two focused designs: a **Distributed Rate Limiter** (a common standalone component) and a **Notification System** (email, SMS, push). These are smaller in scope than the previous full systems but appear very frequently in interviews.

---

## Design 1: Distributed Rate Limiter

### Why This Matters
- Rate limiting is a feature in almost every system design ("how do you prevent abuse?").
- It's sometimes asked as a standalone design: "Design a rate limiter."
- It tests Redis, distributed systems, and algorithm knowledge.

### Requirements
**Functional:**
1. Limit requests per user/API key per time window.
2. Return 429 Too Many Requests when limit exceeded.
3. Return headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`.
4. Configurable limits per endpoint (e.g., `/api/v1/tweets` = 300/hour, `/api/v1/search` = 100/hour).

**Non-functional:**
1. < 1ms latency overhead per request.
2. Distributed (works across multiple API servers).
3. Eventually consistent (a few milliseconds of lag is OK).

### Architecture
```
Client → API Gateway → [Rate Limiter Middleware] → API Server
                          │
                          ▼
                      Redis (shared counter)
```

The rate limiter is a **middleware** that runs before the API handler. It checks Redis for the current count; if over limit, returns 429.

### Algorithms

#### a) Fixed Window Counter
```
Key: "rate:{user_id}:{endpoint}:{minute}"  (e.g., "rate:123:tweet:202401151030")
Count: INCR key → 1, 2, 3, ..., 301
If count > 300 → 429

Problem: burst at boundary (300 at 10:59 + 300 at 11:01 = 600 in 2 seconds).
```

#### b) Sliding Window (Sorted Set)
```
Key: "rate:{user_id}:{endpoint}"
Members: timestamps of each request (sorted set)

On each request:
1. ZREMRANGEBYSCORE key 0 (now - window)  // remove old entries
2. ZADD key now now  // add current request
3. ZCARD key  // count
4. If count > limit → 429

More accurate, but higher memory (store every request timestamp).
```

#### c) Token Bucket (Recommended)
```
Bucket per user+endpoint:
  - tokens: current count (starts at limit)
  - last_refill: timestamp

On each request:
1. Calculate elapsed time since last_refill.
2. Add tokens: tokens += elapsed * refill_rate (cap at limit).
3. If tokens >= 1: tokens -= 1, allow. Update last_refill.
4. If tokens < 1: 429.

Allows bursts (bucket can be full). Smooth average rate.
```

**Implementation in Redis (Lua script for atomicity):**
```lua
-- Token bucket in Redis (atomic)
local key = KEYS[1]
local capacity = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])  -- tokens per second
local now = tonumber(ARGV[3])
local tokens = tonumber(redis.call("GET", key .. ":tokens") or capacity)
local last_refill = tonumber(redis.call("GET", key .. ":last") or now)

-- Refill
local elapsed = now - last_refill
tokens = math.min(capacity, tokens + elapsed * refill_rate)

if tokens >= 1 then
    tokens = tokens - 1
    redis.call("SET", key .. ":tokens", tokens)
    redis.call("SET", key .. ":last", now)
    return 1  -- allowed
else
    return 0  -- denied
end
```

### Distributed Considerations
- All API servers share the same Redis → consistent rate limiting.
- Redis is fast (sub-ms) → minimal overhead.
- If Redis is down: fail-open (allow the request) or fail-closed (deny everything)? Usually fail-open for availability.
- Multi-region: use local Redis per region (slight overage is acceptable).

### Go Implementation
```go
func rateLimitMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        userID := getUserID(r)  // from JWT
        endpoint := r.URL.Path
        
        key := fmt.Sprintf("rate:%s:%s", userID, endpoint)
        allowed, remaining := checkRateLimit(key, 300, 5*time.Minute)
        
        w.Header().Set("X-RateLimit-Limit", "300")
        w.Header().Set("X-RateLimit-Remaining", strconv.Itoa(remaining))
        
        if !allowed {
            w.Header().Set("X-RateLimit-Reset", strconv.FormatInt(time.Now().Add(5*time.Minute).Unix(), 10))
            http.Error(w, `{"error":"rate_limit_exceeded"}`, 429)
            return
        }
        
        next.ServeHTTP(w, r)
    })
}
```

---

## Design 2: Notification System

### Why This Matters
- "Design a notification system" is a common standalone design question.
- It combines message queues, multiple delivery channels, templates, and user preferences.
- It's the backend for email, SMS, and push notifications across any product.

### Requirements
**Functional:**
1. Send notifications via email, SMS, and push.
2. Notifications triggered by events (order placed, payment failed, new message).
3. User preferences: opt-in/opt-out per channel, quiet hours.
4. Templates: reusable notification templates with variable substitution.
5. Deduplication: don't send the same notification twice.

**Non-functional:**
1. At-least-once delivery (may duplicate; consumers must be idempotent).
2. < 30 seconds latency for push.
3. < 5 minutes latency for email.
4. 10M notifications/day.

### Architecture
```
Event Sources                  Notification System
┌──────────┐                  ┌──────────────────────┐
│Order Svc │──┐              │  ┌──────────────┐    │
│Payment   │──┤              │  │ Notification │    │
│Chat Svc  │──┼──→ Kafka ──→│  │ Processor    │    │
│Auth Svc  │──┘              │  └──────┬───────┘    │
└──────────┘                  │         │             │
                              │    ┌────┴────┐       │
                              │    │ Check   │       │
                              │    │ Prefs   │       │
                              │    │ + Dedup │       │
                              │    └────┬────┘       │
                              │    ┌────┼────┐       │
                              │    ▼    ▼    ▼       │
                              │  ┌──┐┌──┐┌──┐       │
                              │  │Eml││SMS││PSH│     │
                              │  └─┬─┘└─┬─┘└─┬─┘     │
                              └────┼─────┼─────┼──────┘
                                   ▼     ▼     ▼
                              ┌──────┐┌──────┐┌──────┐
                              │SES/  ││Twilio││APNs/ │
                              │SendGrid││     ││FCM   │
                              └──────┘└──────┘└──────┘
```

### Components

**1. Event Ingestion:**
```
Services publish events to Kafka:
  topic: "notifications"
  event: { "user_id": 123, "type": "order_placed", "data": {...}, "channels": ["email", "push"] }
```

**2. Notification Processor (Kafka consumer):**
```
For each event:
1. Look up user preferences: "Does user 123 want 'order_placed' via email? Via push?"
   → Redis: prefs:{user_id} → { "order_placed": {"email": true, "push": true, "sms": false} }

2. Check dedup: hash(user_id + type + content)
   → Redis: dedup:{hash} → if exists, skip (already sent)

3. Check quiet hours: is it 2 AM in the user's timezone?
   → If yes and not urgent → schedule for later (delayed queue).

4. For each allowed channel:
   → Render template (substitute variables).
   → Publish to channel-specific Kafka topic: "notifications.email", "notifications.sms", "notifications.push"
```

**3. Channel Adapters (Strategy Pattern):**
```
EmailAdapter: consumes "notifications.email" → calls SendGrid/SES API
SMSAdapter: consumes "notifications.sms" → calls Twilio API
PushAdapter: consumes "notifications.push" → calls APNs/FCM API

Each adapter:
  - Retries on failure (exponential backoff: 1s, 2s, 4s, 8s... max 5 retries)
  - Dead-letter queue after max retries → alert ops
  - Rate-limited per provider (don't exceed SendGrid's 10 emails/sec)
```

**4. Template Engine:**
```
Template stored in DB or S3:
  Subject: "Your order #{order_id} has been placed!"
  Body: "Hi {user_name}, your order for {item_name} totaling ${amount} has been confirmed."

Processor substitutes variables from the event data.
```

### API
```
POST /api/v1/notifications (internal API, called by services)
  Body: { "user_id": 123, "type": "order_placed", "channels": ["email", "push"], "data": {...} }
  Response: 202 Accepted { "notification_id": "n789" }

GET /api/v1/notifications?user_id=123&cursor=abc  (notification history)

PUT /api/v1/preferences/{user_id}
  Body: { "order_placed": {"email": true, "push": false, "sms": false} }
```

### Edge Cases
- **Provider down:** Retry (exponential backoff). After max → DLQ. Don't lose the notification.
- **User unsubscribed:** Check preferences before sending. Include unsubscribe link in emails.
- **Rate limit per provider:** SendGrid allows 10/sec. Batch emails if needed.
- **Duplicate events:** Dedup hash (user_id + type + content) in Redis with 24h TTL.
- **Internationalization:** Template per locale (`template_en.html`, `template_es.html`).
- **Quiet hours:** Check user's timezone. Delay non-urgent notifications.

---

## Exercise: Design a Webhook Delivery System

**Requirements:**
1. Deliver webhooks to registered URLs (like Stripe/GitHub webhooks).
2. At-least-once delivery.
3. Retry on failure (exponential backoff).
4. Users can view delivery history and manually retry.

**Think about:**
- How to store pending deliveries.
- Retry strategy and backoff.
- How to handle slow/unresponsive endpoints.
- How to prevent overwhelming a recipient.

<details>
<summary>Approach</summary>

1. **Storage:** PostgreSQL table `webhook_deliveries` (id, url, payload, status, attempt_count, next_retry_at).
2. **Queue:** Kafka or a background worker polls for pending deliveries.
3. **Retry:** Exponential backoff: 1min, 5min, 30min, 2hr, 12hr. After 5 attempts → mark as "failed."
4. **Timeout:** HTTP request with 10-second timeout. If no response → mark as failed → retry later.
5. **Rate limiting per URL:** Max 100 deliveries/min per URL. Use Redis to track.
6. **Manual retry:** API endpoint `POST /webhooks/{id}/retry` → resets status to "pending."
7. **Signature:** Sign each payload with HMAC so the recipient can verify authenticity.
8. **Idempotency:** Each delivery has a unique ID. Recipient should deduplicate.
</details>

---

## Common Mistakes
- Rate limiter: not using an atomic operation (race condition between read and increment).
- Rate limiter: not handling Redis failure (fail-open vs fail-closed).
- Notifications: not checking user preferences before sending (spam!).
- Notifications: not handling retries (provider down → notification lost).
- Webhooks: not signing payloads (security risk — spoofing).

## Checklist Before Moving On
- [ ] Can implement a rate limiter (token bucket or sliding window)
- [ ] Know 3 rate limiting algorithms and their trade-offs
- [ ] Can design a notification system with multiple channels
- [ ] Understand the Strategy pattern for channel adapters
- [ ] Can handle retries, deduplication, and quiet hours
- [ ] Solved the webhook delivery exercise
