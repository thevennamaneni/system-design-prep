# Day 10 - API Design: REST, Resource Modeling & Versioning

## Topic
**REST API design** — resource modeling, URL conventions, HTTP methods, status codes, pagination, versioning, and error handling. This is a core interview skill — every system design includes an API design step.

---

## Why It Matters
- The "A" in RESHADED is API design. You'll spend 5-10 minutes defining endpoints in every interview.
- Poor API design signals lack of experience; clean, RESTful APIs signal seniority.
- API design directly affects frontend experience, caching, and scalability.

---

## 1. REST Principles

**REST** (Representational State Transfer) is an architectural style for APIs based on resources.

### Six REST Constraints
| Constraint | Meaning |
|-----------|---------|
| **Client-Server** | UI and data concerns are separated |
| **Stateless** | Each request contains all info needed; server stores no session state |
| **Cacheable** | Responses must declare if they're cacheable |
| **Uniform Interface** | Consistent resource identification and manipulation |
| **Layered System** | Client can't tell if it's talking to the server or an intermediary |
| **Code on Demand** (optional) | Server can send executable code (rarely used) |

### The Key Principle: Resources
Everything is a **resource** identified by a URL. Operations are HTTP methods.

```
Resource: /users/123
  GET    /users/123        → retrieve user 123
  POST   /users            → create a new user
  PUT    /users/123        → update user 123 (full replace)
  PATCH  /users/123        → partial update user 123
  DELETE /users/123        → delete user 123
  GET    /users            → list all users (with pagination)
```

---

## 2. Resource Modeling

### Naming Conventions
- Use **nouns**, not verbs: `/users` (not `/getUsers`)
- Use **plurals**: `/users` (not `/user`)
- Use **kebab-case** for multi-word: `/order-items` (not `/orderItems` or `/order_items`)
- Nest for relationships: `/users/123/orders` (orders belonging to user 123)

### URL Structure
```
Collection:     /api/v1/users
Single item:    /api/v1/users/{id}
Sub-collection: /api/v1/users/{id}/orders
Sub-item:       /api/v1/users/{id}/orders/{orderId}
Action (rare):  /api/v1/users/{id}/activate  (use POST, for non-CRUD actions)
```

### Example: Twitter API
```
POST   /api/v1/tweets                    — create tweet
GET    /api/v1/tweets/{id}               — get tweet
DELETE /api/v1/tweets/{id}               — delete tweet
GET    /api/v1/users/{id}/timeline       — get user's timeline
POST   /api/v1/users/{id}/follows        — follow a user
DELETE /api/v1/users/{id}/follows/{fid}  — unfollow
POST   /api/v1/tweets/{id}/likes         — like a tweet
DELETE /api/v1/tweets/{id}/likes/{uid}   — unlike
GET    /api/v1/tweets/{id}/comments      — get comments
POST   /api/v1/tweets/{id}/comments      — add comment
```

---

## 3. HTTP Methods & Idempotency

| Method | Safe? | Idempotent? | Purpose |
|--------|-------|-------------|---------|
| GET | Yes | Yes | Read data (no side effects) |
| POST | No | No | Create a new resource |
| PUT | No | Yes | Replace a resource (full update) |
| PATCH | No | Maybe | Partial update |
| DELETE | No | Yes | Delete a resource |

### Why Idempotency Matters
If a network timeout occurs, the client retries. Idempotent methods can be retried safely:

```
DELETE /users/123
  → Network timeout
  → Client retries: DELETE /users/123
  → Result: same (user is deleted, second call is a no-op)
```

POST is NOT idempotent:
```
POST /orders (create order)
  → Network timeout (but server actually created the order)
  → Client retries: POST /orders
  → Result: TWO orders created! ← BAD
```

**Solution for POST:** Use an **idempotency key** (client-generated UUID):
```
POST /orders
  Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
  Body: {"item": "book", "quantity": 1}

Server checks: have I seen this key?
  → If yes: return the previous result (don't create a duplicate)
  → If no: create order, store the key, return result
```

---

## 4. Request & Response Design

### Request Format (JSON)
```json
POST /api/v1/tweets
Authorization: Bearer eyJhbG...
Content-Type: application/json

{
  "content": "Hello, world!",
  "media_ids": ["img_123", "img_456"],
  "reply_to": null
}
```

### Response Format
```json
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/v1/tweets/987654

{
  "id": "987654",
  "content": "Hello, world!",
  "user_id": "123",
  "created_at": "2024-01-15T10:30:00Z",
  "likes_count": 0,
  "retweets_count": 0
}
```

### Error Format (Consistent!)
```json
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": {
    "code": "INVALID_INPUT",
    "message": "Content exceeds 280 characters",
    "details": {
      "max_length": 280,
      "actual_length": 295
    },
    "request_id": "req_abc123"
  }
}
```

---

## 5. Pagination

Never return an unbounded list. Always paginate.

### a) Offset/Limit Pagination
```
GET /api/v1/tweets?offset=20&limit=10

Response:
{
  "data": [...10 tweets...],
  "pagination": {
    "offset": 20,
    "limit": 10,
    "total": 1500
  }
}
```

**Pros:** Simple, allows jumping to a specific page.
**Cons:** Slow for large offsets (`OFFSET 1000000` scans 1M rows). Items may shift if new data arrives.

### b) Cursor-Based Pagination (Preferred)
```
GET /api/v1/tweets?cursor=abc123&limit=10

Response:
{
  "data": [...10 tweets...],
  "pagination": {
    "next_cursor": "def456",
    "has_more": true
  }
}
```

**Pros:** Fast (uses indexed cursor, no offset scan). Stable (new items don't shift results).
**Cons:** Can't jump to a specific page. Cursor is opaque.

**How it works:** The cursor encodes the last item's sort key (e.g., `created_at` or ID). Next query: `WHERE created_at < cursor_value ORDER BY created_at DESC LIMIT 10`.

### c) Keyset Pagination
```
GET /api/v1/tweets?after_id=98765&limit=10

→ SELECT * FROM tweets WHERE id > 98765 ORDER BY id LIMIT 10
```
Simple and fast. Use for ID-ordered data.

---

## 6. API Versioning

When you change the API (breaking changes), version it to avoid breaking existing clients.

### Versioning Strategies

| Strategy | URL Example | Pros | Cons |
|----------|-------------|------|------|
| **URI versioning** | `/api/v1/users` | Simple, visible | Ugly URLs |
| **Header versioning** | `Accept: application/vnd.api+json; version=1` | Clean URLs | Less discoverable |
| **Query parameter** | `/api/users?version=1` | Simple | Easy to forget |

**Recommendation:** URI versioning (`/api/v1/`) — it's the most common and visible.

### When to Version
- **Breaking change** (rename field, change type, remove endpoint) → new version.
- **Non-breaking change** (add optional field, add endpoint) → no new version needed.

---

## 7. Rate Limiting in APIs

Protect your API from abuse:
```
429 Too Many Requests
Headers:
  X-RateLimit-Limit: 100
  X-RateLimit-Remaining: 42
  X-RateLimit-Reset: 1642248000
```

**Rate limiting algorithms** (covered Day 27):
- Token bucket
- Sliding window
- Fixed window

---

## Go Connection

```go
// RESTful API in Go (using standard library or chi/gin)
// Using net/http with gorilla/mux or chi:

r := chi.NewRouter()
r.Route("/api/v1", func(r chi.Router) {
    r.Use(middleware.Logger)
    r.Use(middleware.RequestID)
    r.Use(authMiddleware)
    
    r.Get("/tweets/{id}", getTweet)
    r.Post("/tweets", createTweet)
    r.Delete("/tweets/{id}", deleteTweet)
    r.Get("/users/{id}/timeline", getTimeline)
    r.Post("/users/{id}/follows", followUser)
})

// Handler with pagination:
func getTimeline(w http.ResponseWriter, r *http.Request) {
    userID := chi.URLParam(r, "id")
    cursor := r.URL.Query().Get("cursor")
    limit := parseLimit(r.URL.Query().Get("limit"), 20)  // default 20, max 100
    
    tweets, nextCursor, err := service.GetTimeline(userID, cursor, limit)
    if err != nil {
        writeError(w, 500, "INTERNAL_ERROR", err.Error())
        return
    }
    
    writeJSON(w, 200, map[string]any{
        "data": tweets,
        "pagination": map[string]any{
            "next_cursor": nextCursor,
            "has_more":    nextCursor != "",
        },
    })
}
```

---

## Exercise: Design the API for a Ride-Sharing App (Uber)

Define REST endpoints for:
1. Rider requests a ride
2. Driver accepts a ride
3. Track ride status
4. Rider cancels a ride
5. Complete a ride and process payment
6. Get ride history
7. Rate the driver/rider

For each: specify method, URL, request body, response, and status codes.

<details>
<summary>Reference Answer</summary>

```
1. Request a ride:
   POST /api/v1/rides
   Body: { "pickup": {"lat": 37.7, "lng": -122.4}, "dropoff": {"lat": 37.8, "lng": -122.5}, "ride_type": "uber_x" }
   Response: 201 Created
   { "ride_id": "ride_123", "status": "searching", "estimated_fare": 15.50, "estimated_eta": 300 }

2. Driver accepts a ride:
   POST /api/v1/rides/{ride_id}/accept
   Body: { "driver_id": "driver_456" }
   Response: 200 OK
   { "ride_id": "ride_123", "status": "driver_assigned", "driver": {...}, "eta": 180 }

3. Track ride status:
   GET /api/v1/rides/{ride_id}
   Response: 200 OK
   { "ride_id": "ride_123", "status": "ongoing", "driver": {...}, "current_location": {...} }

4. Cancel a ride:
   POST /api/v1/rides/{ride_id}/cancel
   Body: { "reason": "changed_mind" }
   Response: 200 OK
   { "ride_id": "ride_123", "status": "cancelled", "cancellation_fee": 0 }

5. Complete a ride:
   POST /api/v1/rides/{ride_id}/complete
   Body: { "actual_fare": 16.20, "distance_km": 5.2, "duration_sec": 900 }
   Response: 200 OK
   { "ride_id": "ride_123", "status": "completed", "fare": 16.20, "payment_status": "charged" }

6. Get ride history:
   GET /api/v1/users/{user_id}/rides?cursor=abc&limit=20
   Response: 200 OK
   { "data": [...], "pagination": {"next_cursor": "def", "has_more": true} }

7. Rate:
   POST /api/v1/rides/{ride_id}/ratings
   Body: { "rating": 5, "comment": "Great ride!", "rated_by": "rider_789", "rated_role": "driver" }
   Response: 201 Created
   { "rating_id": "rating_999", "ride_id": "ride_123", "rating": 5 }
```

**Design decisions:**
- Rides are resources (`/rides/{id}`). Actions (accept, cancel, complete) are sub-resources.
- POST for actions (not PUT/PATCH) because they're not idempotent (accepting twice = error).
- Cursor-based pagination for ride history (could be thousands of rides).
- Status codes: 201 for creation, 200 for success, 4xx for client errors, 5xx for server errors.
</details>

---

## Common Mistakes
- Using verbs in URLs (`/getUser` instead of `GET /users/{id}`).
- Not paginating list endpoints — returning 10,000 results in one response.
- Using offset pagination for large datasets — slow and unstable.
- Not using idempotency keys for POST requests that create resources.
- Inconsistent error formats — every error should follow the same structure.
- Not versioning the API — breaking changes break all clients.

## Checklist Before Moving On
- [ ] Can model REST resources from requirements
- [ ] Know HTTP method semantics and idempotency
- [ ] Can design cursor-based pagination
- [ ] Understand API versioning strategies
- [ ] Can design a consistent error response format
- [ ] Solved the Uber API design exercise
