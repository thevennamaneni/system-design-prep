# Day 20 - Security, Authentication & Authorization

## Topic
**Security fundamentals** — authentication (OAuth, JWT, sessions), authorization (RBAC, ABAC), API security (rate limiting, CORS, input validation), and encryption (at rest, in transit).

---

## Why It Matters
- Every system design includes an auth component. "How do users log in?" is always asked.
- Security mistakes are catastrophic (data breaches, unauthorized access).
- Senior engineers are expected to design secure systems, not just functional ones.

---

## 1. Authentication vs Authorization

| Term | Meaning | Example |
|------|---------|---------|
| **Authentication (AuthN)** | Who are you? | Login with username/password |
| **Authorization (AuthZ)** | What can you do? | Can user X delete post Y? |

---

## 2. Authentication Methods

### a) Session-Based Auth
```
1. User submits username + password.
2. Server verifies credentials → creates a session (stored in DB/Redis).
3. Server sends a session ID (cookie) to the client.
4. Client sends cookie with every request.
5. Server looks up session ID → identifies user.
```

```
Client ← Set-Cookie: session_id=abc123; HttpOnly; Secure; SameSite=Strict
Client → Cookie: session_id=abc123
Server → Redis: GET session:abc123 → { user_id: 42, role: "admin" }
```

**Pros:** Server can revoke sessions instantly (delete from Redis). Simple.
**Cons:** Requires server-side session storage. Doesn't scale naturally (needs shared session store like Redis).

### b) Token-Based Auth (JWT)
**JWT (JSON Web Token):** A self-contained token with user info encoded as JSON.

```
JWT Structure: header.payload.signature

header:    { "alg": "HS256", "typ": "JWT" }          → base64
payload:   { "user_id": 42, "role": "admin", "exp": 1642248000 } → base64
signature: HMAC-SHA256(header + "." + payload, secret_key)

Full JWT: eyJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjo0Mn0.abc123...
```

**How JWT works:**
```
1. User logs in → server generates JWT with secret key.
2. Client stores JWT (localStorage or cookie).
3. Client sends JWT in Authorization header: "Bearer eyJhbG..."
4. Server verifies signature → trusts the payload → identifies user.
5. No server-side session storage needed!
```

**Pros:**
- Stateless — no session storage on the server.
- Scales horizontally (any server can verify).
- Works across domains/services.

**Cons:**
- **Can't revoke easily** — once issued, valid until expiry. (Solution: short TTL + refresh token, or a blacklist.)
- **Token size** — larger than a session ID.
- **Security risk** if secret key leaks — attacker can forge tokens.

**JWT Best Practices:**
- Short-lived access tokens (15-30 minutes).
- Long-lived refresh tokens (7-30 days) to get new access tokens.
- Store in `HttpOnly` cookie (not localStorage — XSS vulnerable).
- Use strong signing key (RS256 for asymmetric, HS256 for symmetric).

### c) OAuth 2.0 (Third-Party Auth)
"Log in with Google/Facebook/GitHub" → OAuth 2.0.

```
1. User clicks "Login with Google."
2. Redirect to Google: "App X wants to access your profile."
3. User approves → Google redirects back with an authorization code.
4. App exchanges code for an access token (server-to-server).
5. App uses token to call Google's API → gets user info.
6. App creates/logs in the user.
```

**OAuth roles:**
| Role | Example |
|------|---------|
| Resource Owner | The user |
| Client | Your app (wants access) |
| Authorization Server | Google's auth server |
| Resource Server | Google's user info API |

### d) API Keys
For service-to-service or developer API access:
```
GET /api/v1/data
  X-API-Key: sk_test_1234567890
```
- Simple: generate a key, store in DB, validate on each request.
- **Cons:** no user context, no fine-grained permissions (unless you add scopes), can be leaked.
- Use for: server-to-server, developer APIs, rate limiting per key.

---

## 3. Authorization Patterns

### a) RBAC (Role-Based Access Control)
Users have **roles**; roles have **permissions**.

```
Roles: admin, editor, viewer

admin:  [create, read, update, delete]
editor: [create, read, update]
viewer: [read]

User Alice → role: admin → can do everything
User Bob   → role: viewer → can only read
```

**Pros:** Simple, widely understood.
**Cons:** Coarse-grained. If you need "Alice can edit posts but not delete," you need a new role.

### b) ABAC (Attribute-Based Access Control)
Permissions based on **attributes** (user, resource, environment):
```
Rule: "User can edit a post if:
  - user.role == 'editor' AND
  - post.author_id == user.id AND
  - current_time < post.created_at + 24h"
```

**Pros:** Fine-grained, flexible.
**Cons:** Complex to manage and debug.

### c) ACL (Access Control List)
Per-resource list of who can access:
```
Post 123:
  - Alice: [read, edit, delete]
  - Bob: [read]
  - public: [read]
```

**Pros:** Per-resource granularity.
**Cons:** Doesn't scale (millions of resources = millions of ACLs).

---

## 4. Rate Limiting

Prevent abuse by limiting requests per user/IP/API key.

### Rate Limiting Algorithms

#### a) Token Bucket
```
Bucket holds N tokens. Each request consumes 1 token.
Tokens refill at rate R per second.

If bucket has tokens → allow.
If bucket empty → deny (429 Too Many Requests).
```
- Allows bursts (bucket can be full).
- Smooth average rate over time.

#### b) Sliding Window
```
Track request timestamps in a window.
If > N requests in the last W seconds → deny.
```
- More precise than token bucket.
- Higher memory (store timestamps).

#### c) Fixed Window
```
Count requests in fixed time windows (e.g., per minute).
If > N requests in current minute → deny.
```
- Simple.
- Problem: burst at window boundary (100 requests at 0:59 + 100 at 1:01 = 200 in 2 seconds).

### Where to Rate Limit
```
Client → [CDN rate limit] → [API Gateway rate limit] → [App-level rate limit]
```
- **CDN/Gateway:** per-IP, per-API-key. Coarse.
- **Application:** per-user, per-resource. Fine-grained.

### Rate Limit with Redis
```go
// Sliding window rate limiter in Redis (using sorted sets)
func isRateLimited(ctx context.Context, rdb *redis.Client, key string, limit int, window time.Duration) bool {
    now := time.Now().Unix()
    windowStart := now - int64(window.Seconds())
    
    pipe := rdb.TxPipeline()
    pipe.ZRemRangeByScore(ctx, key, "0", strconv.FormatInt(windowStart, 10))  // remove old
    pipe.ZAdd(ctx, key, redis.Z{Score: float64(now), Member: now})  // add current
    pipe.ZCard(ctx, key)  // count
    pipe.Expire(ctx, key, window)
    _, err := pipe.Exec(ctx)
    
    count, _ := rdb.ZCard(ctx, key).Result()
    return count > int64(limit)
}
```

---

## 5. Encryption

### In Transit (TLS)
```
Client ←── TLS encrypted ──→ Server
```
- Encrypts data between client and server.
- Already covered on Day 2 (TLS handshake).
- Use HTTPS everywhere. Terminate TLS at load balancer or API gateway.

### At Rest
```
Database disk → encrypted (even if stolen, data is unreadable)
```
- **Disk-level encryption:** AWS EBS encryption, LUKS. Transparent to the app.
- **Application-level encryption:** Encrypt specific fields (e.g., SSN, credit card) before storing.
  - Use AES-256 for field-level encryption.
  - Manage keys in a KMS (AWS KMS, HashiCorp Vault).

### Password Storage
**NEVER store passwords in plaintext.** Use **bcrypt** or **argon2**:
```
User registers: password = "hello123"
  → bcrypt hash: $2a$10$N9qo8uLOickgx2ZMRZoMye...
  → store the HASH, not the password

User logs in: password = "hello123"
  → bcrypt compare(input, stored_hash) → true/false
```

**Why bcrypt/argon2?** They're slow by design (salted + work factor) — brute-force attacks are impractical.

---

## 6. Common Security Best Practices

| Practice | Why |
|----------|-----|
| Use HTTPS everywhere | Prevents eavesdropping |
| `HttpOnly` cookies | Prevents XSS from stealing tokens |
| `SameSite=Strict` cookies | Prevents CSRF |
| Input validation/sanitization | Prevents SQL injection, XSS |
| Parameterized queries | Prevents SQL injection |
| CORS headers | Controls which origins can call your API |
| Rate limiting | Prevents brute-force, DDoS |
| Principle of least privilege | Users/services get minimum permissions |
| Secrets in env vars/KMS | Never hardcode secrets in code |
| Regular dependency updates | Patch known vulnerabilities (CVEs) |
| WAF (Web Application Firewall) | Blocks common attacks (SQLi, XSS) |

### SQL Injection Example (and Prevention)
```go
// BAD: string concatenation (SQL injection vulnerable)
query := "SELECT * FROM users WHERE name = '" + name + "'"
// If name = "'; DROP TABLE users; --" → disaster!

// GOOD: parameterized query
db.QueryRow("SELECT * FROM users WHERE name = $1", name)
// The driver escapes the parameter safely.
```

### CORS (Cross-Origin Resource Sharing)
```
Browser security: scripts on origin A can't call APIs on origin B by default.
CORS header: Access-Control-Allow-Origin: https://app.example.com
  → allows the browser to make the cross-origin request.
```

---

## Go Connection

```go
// JWT authentication in Go
import "github.com/golang-jwt/jwt/v5"

func GenerateToken(userID int, role string) (string, error) {
    claims := jwt.MapClaims{
        "user_id": userID,
        "role":    role,
        "exp":     time.Now().Add(30 * time.Minute).Unix(),
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(secretKey))
}

func VerifyToken(tokenString string) (int, string, error) {
    token, err := jwt.Parse(tokenString, func(t *jwt.Token) (any, error) {
        return []byte(secretKey), nil
    })
    if err != nil || !token.Valid {
        return 0, "", err
    }
    claims := token.Claims.(jwt.MapClaims)
    return int(claims["user_id"].(float64)), claims["role"].(string), nil
}

// Auth middleware
func authMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        token := r.Header.Get("Authorization")  // "Bearer eyJ..."
        token = strings.TrimPrefix(token, "Bearer ")
        
        userID, role, err := VerifyToken(token)
        if err != nil {
            http.Error(w, "Unauthorized", 401)
            return
        }
        
        // Add user info to context
        ctx := context.WithValue(r.Context(), "userID", userID)
        ctx = context.WithValue(ctx, "role", role)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// RBAC authorization
func requireRole(role string, next http.HandlerFunc) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        userRole := r.Context().Value("role").(string)
        if userRole != role {
            http.Error(w, "Forbidden", 403)
            return
        }
        next(w, r)
    }
}

// Password hashing with bcrypt
import "golang.org/x/crypto/bcrypt"

func HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), 10)  // cost = 10
    return string(bytes), err
}

func CheckPassword(password, hash string) bool {
    return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}
```

**Go advantage:** Go's standard library and ecosystem (`golang-jwt`, `bcrypt`, `crypto/tls`) make security implementation clean and idiomatic.

---

## Exercise: Design Auth for a Multi-Tenant SaaS Application

**Requirements:**
1. Users belong to organizations (tenants).
2. Users have roles within their org (admin, member, viewer).
3. Users can log in with email/password OR Google OAuth.
4. API keys for programmatic access (per org).
5. Admins can invite new users to their org.
6. Must prevent users from accessing other orgs' data.

**Design:**
1. Session-based or JWT? Why?
2. How do you handle multi-tenancy in authorization?
3. How do you implement OAuth (Google login)?
4. How do you secure API keys?
5. How do you prevent cross-tenant data access?

<details>
<summary>Reference Answer</summary>

1. **JWT with refresh tokens:**
   - Access token (30 min TTL): contains `user_id`, `org_id`, `role`.
   - Refresh token (7 day TTL): stored in DB, used to get new access tokens.
   - Stateless → scales horizontally across API servers.

2. **Multi-tenancy in AuthZ:**
   - JWT includes `org_id` → every API request is scoped to that org.
   - Every DB query includes `WHERE org_id = ?` → tenant isolation at the data layer.
   - RBAC per org: `admin`, `member`, `viewer` scoped to the org.

3. **OAuth (Google login):**
   - Register app with Google → get `client_id` + `client_secret`.
   - User clicks "Login with Google" → redirect to Google.
   - Google redirects back with authorization code.
   - Server exchanges code for Google access token → fetches user email.
   - If email matches an existing user → generate JWT.
   - If not → "no account found, ask your admin to invite you."

4. **API keys:**
   - Generate random 32-byte key → hash with bcrypt → store hash in DB.
   - On each API request: `X-API-Key: sk_live_...` → look up by hash → get `org_id` + `scopes`.
   - API keys have scopes (e.g., `read:orders`, `write:orders`).
   - Admins can revoke keys (delete from DB).

5. **Cross-tenant prevention:**
   - JWT contains `org_id`. Every query is scoped: `SELECT * FROM orders WHERE org_id = ? AND id = ?`.
   - Never trust a `org_id` from the request body — always from the JWT.
   - Row-level security in PostgreSQL: `SET app.current_org = org_id` + RLS policies.
   - API key validation always includes org scoping.
</details>

---

## Common Mistakes
- Storing JWTs in localStorage (XSS vulnerable) → use HttpOnly cookies.
- Not expiring JWTs (long-lived = high risk if stolen) → short TTL + refresh tokens.
- Storing passwords in plaintext or MD5/SHA (fast hashes) → use bcrypt/argon2.
- Not using parameterized queries → SQL injection.
- Rate limiting only at the app level (not CDN/gateway) → DDoS reaches your servers.

## Checklist Before Moving On
- [ ] Understand session-based vs JWT auth (pros/cons)
- [ ] Can explain OAuth 2.0 flow (authorization code grant)
- [ ] Know RBAC vs ABAC vs ACL
- [ ] Can implement rate limiting (token bucket, sliding window)
- [ ] Know encryption at rest vs in transit
- [ ] Know password hashing (bcrypt/argon2)
- [ ] Understand CORS, HttpOnly cookies, SameSite
- [ ] Solved the Multi-Tenant SaaS auth exercise
