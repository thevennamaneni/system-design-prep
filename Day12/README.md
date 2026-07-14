# Day 12 - Microservices, Service Discovery & API Gateway

## Topic
**Microservices architecture**, **service discovery** (how services find each other), and **API gateway** (the single entry point for all clients). These define how modern distributed systems are structured.

---

## Why It Matters
- Every FAANG system is microservices-based. You must understand the architecture, trade-offs, and patterns.
- "Should this be a monolith or microservices?" is a common interview question.
- API gateway is a component you'll draw in almost every system design — know what it does.

---

## 1. Monolith vs Microservices

### Monolith
```
┌─────────────────────────┐
│      Monolith            │
│  ┌─────┐ ┌─────┐ ┌────┐ │
│  │Auth │ │Feed │ │Pay │ │  All code in one deployable unit
│  └─────┘ └─────┘ └────┘ │
│  ┌──────────────────┐   │
│  │   Shared DB       │   │
│  └──────────────────┘   │
└─────────────────────────┘
```

**Pros:**
- Simple to develop, test, deploy.
- No network overhead (function calls, not HTTP/gRPC).
- Easy to reason about (one codebase).
- ACID transactions across modules.

**Cons:**
- Scaling: must scale the entire app, even if only one module needs it.
- Deployment: change one module → redeploy everything.
- Technology lock-in: all modules use the same language/framework.
- As codebase grows: build times, merge conflicts, "bus factor."

### Microservices
```
┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
│ Auth │  │ Feed │  │ Pay  │  │ User │
│Service│  │Service│ │Service│  │Service│
└──┬───┘  └──┬───┘  └──┬───┘  └──┬───┘
   │         │         │         │
   ▼         ▼         ▼         ▼
┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐
│ DB  │   │ DB  │   │ DB  │   │ DB  │  Each has its own DB
└─────┘   └─────┘   └─────┘   └─────┘
```

**Pros:**
- Independent scaling: scale only the services that need it.
- Independent deployment: deploy each service separately.
- Technology diversity: Go for this service, Python for ML, etc.
- Fault isolation: one service crashing doesn't bring down others.
- Team ownership: each team owns its services.

**Cons:**
- **Network overhead:** service-to-service calls over HTTP/gRPC.
- **Distributed data:** no ACID across services (eventual consistency, sagas).
- **Operational complexity:** monitoring, tracing, deployment for N services.
- **Service discovery:** how do services find each other?
- **Latency:** a single user request may chain through 5+ services.

### When to Choose Microservices
- Large teams (multiple teams working in parallel).
- Different scaling needs (auth: low traffic; feed: high traffic).
- Different technology needs (Go for APIs, Python for ML).
- The organization has DevOps maturity (CI/CD, observability).

### When to Stay Monolith
- Small team (< 10 engineers).
- Early stage (product-market fit not yet found).
- Simple domain (not enough complexity to justify splitting).
- "Start monolith, extract microservices when needed" — a common approach.

### The Interview Answer
> "I'd start with a modular monolith for simplicity, and extract microservices when a specific module needs independent scaling or deployment. For this design, I'll show the microservices architecture since we're at scale."

---

## 2. Service Discovery

**The problem:** In a microservices world, services are dynamically created/destroyed. IP addresses change constantly. How does Service A find Service B?

### a) Client-Side Discovery
```
1. Service A queries a service registry: "Where is Service B?"
2. Registry returns: ["10.0.1.5:8080", "10.0.1.6:8080", "10.0.1.7:8080"]
3. Service A picks one (round-robin) and calls it directly
```

**Pros:** No intermediary (lower latency).
**Cons:** Client must implement load balancing logic.

### b) Server-Side Discovery
```
1. Service A calls a load balancer: "Give me Service B"
2. LB queries the registry, picks an instance, proxies the request
```

**Pros:** Client is simple (just call the LB).
**Cons:** Extra hop (through LB).

### Service Registry
A central registry where services register themselves and discover others.

```
Service B starts → registers: "I am service-b, at 10.0.1.5:8080"
Service B stops → deregisters
Service A needs B → queries registry → gets list of instances
```

**Tools:**
- **etcd:** Distributed key-value store (used by Kubernetes).
- **Consul:** HashiCorp's service discovery + health checking.
- **ZooKeeper:** Apache's coordination service (used by Kafka).
- **Kubernetes DNS:** K8s built-in service discovery (`service-b.namespace.svc.cluster.local`).

### Health Checking
The registry continuously checks service health:
```
Registry → GET /health → Service B instance 1: OK → keep in pool
Registry → GET /health → Service B instance 2: fail → remove from pool
```

---

## 3. API Gateway

**What it is:** A single entry point for all external clients. It routes requests to the appropriate microservice, and handles cross-cutting concerns.

```
         ┌──────────────┐
Clients  │  API Gateway  │  Microservices
  ──────→│              │──────→ Auth Service
         │  - Routing    │──────→ User Service
         │  - Auth       │──────→ Feed Service
         │  - Rate limit │──────→ Payment Service
         │  - Caching    │──────→ Notification Service
         │  - Logging    │
         │  - SSL term   │
         │  - Aggregation│
         └──────────────┘
```

### API Gateway Responsibilities

| Responsibility | Description |
|---------------|-------------|
| **Routing** | Route `/api/users/*` to User Service, `/api/feed/*` to Feed Service |
| **Authentication** | Verify JWT token once at gateway; pass user context to services |
| **Rate limiting** | Limit requests per user/IP to prevent abuse |
| **SSL termination** | Decrypt HTTPS; internal calls use plain HTTP |
| **Caching** | Cache responses for GET requests (reduce backend load) |
| **Logging/metrics** | Log all requests; collect latency metrics |
| **Request aggregation** | Combine responses from multiple services into one response |
| **Protocol translation** | External REST → internal gRPC |
| **CORS** | Handle cross-origin requests |
| **Versioning** | Route `/v1/*` and `/v2/*` to different service versions |

### Request Aggregation Example
```
Client requests: GET /api/dashboard
  → API Gateway calls:
    - User Service: GET /user/123 → user profile
    - Feed Service: GET /feed/123 → recent activity
    - Stats Service: GET /stats/123 → analytics
  → Gateway combines responses → returns to client
```
One client request → 3 backend calls → one response. Reduces client round-trips.

### API Gateway vs Load Balancer
| Aspect | API Gateway | Load Balancer |
|--------|------------|---------------|
| Layer | L7 (application) | L4 or L7 |
| Routing | By URL path, header | By IP/port (round-robin) |
| Features | Auth, rate limit, caching | Just traffic distribution |
| Scope | External-facing entry point | Internal traffic distribution |
| Example | Kong, AWS API Gateway, Nginx | AWS ALB, HAProxy |

### BFF (Backend for Frontend)
A variant of API Gateway: **one gateway per client type**.

```
Mobile App → Mobile BFF → (tailored API) → Microservices
Web App    → Web BFF    → (tailored API) → Microservices
```

**Why?** Mobile and web have different data needs. Mobile BFF returns less data (bandwidth); web BFF returns more (richer UI). Each BFF is optimized for its client.

---

## 4. Inter-Service Communication

### Synchronous (gRPC/REST)
```
Service A → HTTP/gRPC call → Service B → response
```
- **Pros:** Simple, immediate response.
- **Cons:** Tight coupling, cascading failures (if B is slow, A is slow).

### Asynchronous (Message Queue)
```
Service A → Publish event → Kafka → Service B consumes
```
- **Pros:** Decoupled, resilient (B can be down), load leveling.
- **Cons:** Eventual consistency, more complex.

### Service Mesh
A infrastructure layer for service-to-service communication:
```
Service A → [Sidecar Proxy] → [Sidecar Proxy] → Service B
```
- Handles: mTLS, retries, circuit breaking, observability, load balancing.
- Tools: **Istio**, **Linkerd** (both built on Envoy).
- Go connection: Istio and Linkerd are written in Go!

---

## 5. Circuit Breaker Pattern

Prevent cascading failures when a downstream service is failing:

```
States:
  CLOSED:     Requests flow normally. (monitor failures)
  OPEN:       Requests fail fast (don't call the failing service). 
              Wait for recovery period.
  HALF-OPEN:  Allow a test request. If it succeeds → CLOSED. If it fails → OPEN.
```

```
Normal:    Service A → Service B (OK)     [CLOSED]
Failure:   Service A → Service B (fail)   [failure count++]
Trip:      Service A → Service B (fail x5) [OPEN: stop calling B]
Recovery:  Wait 30s → Service A → Service B (test) [HALF-OPEN]
           If OK: [CLOSED]  If fail: [OPEN]
```

**In Go:** Use `github.com/afex/hystrix-go` or `github.com/sony/gobreaker`.

```go
import "github.com/sony/gobreaker"

cb := gobreaker.NewCircuitBreaker(gobreaker.Settings{
    Name:        "UserService",
    MaxRequests: 5,               // half-open: allow 5 test requests
    Interval:    60 * time.Second,
    Timeout:     30 * time.Second, // open: wait 30s before half-open
    ReadyToTrip: func(counts gobreaker.Counts) bool {
        return counts.ConsecutiveFailures > 5
    },
})

result, err := cb.Execute(func() (any, error) {
    return userService.GetUser(ctx, id)
})
```

---

## Exercise: Design Microservices for an E-Commerce Platform (Amazon-lite)

**Requirements:**
1. User browses products, adds to cart, checks out.
2. Services: User, Product, Cart, Order, Payment, Inventory, Notification.
3. Each service has its own database.
4. External clients access via an API gateway.
5. Services communicate via gRPC (sync) and Kafka (async).

**Design:**
1. Draw the architecture (API gateway, services, databases, message queue).
2. Which calls are synchronous vs asynchronous?
3. What does the API gateway handle?
4. How do services discover each other?
5. What happens if the Payment service is down?

<details>
<summary>Reference Answer</summary>

```
                    ┌──────────────┐
    Clients ───────→│  API Gateway  │
                    │  (Kong/Nginx) │
                    └──────┬───────┘
            ┌─────────┬────┴────┬──────────┐
            ▼         ▼         ▼          ▼
       ┌───────┐ ┌───────┐ ┌───────┐  ┌───────┐
       │ User  │ │Product│ │ Cart  │  │ Order │
       │Service│ │Service│ │Service│  │Service│
       └───┬───┘ └───┬───┘ └───┬───┘  └───┬───┘
           │         │         │          │
       ┌───┴───┐ ┌───┴───┐ ┌───┴───┐  ┌───┴───┐
       │  DB   │ │  DB   │ │ Redis │  │  DB   │
       └───────┘ └───────┘ └───────┘  └───────┘
                                          │
                              ┌───────────┼───────────┐
                              ▼           ▼           ▼
                         ┌───────┐  ┌────────┐  ┌──────────┐
                         │Payment│  │Inventory│  │Notification│
                         │Service│  │Service  │  │Service    │
                         └───┬───┘  └───┬────┘  └─────┬────┘
                             │          │             │
                         ┌───┴───┐  ┌───┴───┐         │
                         │  DB   │  │  DB   │         │
                         └───────┘  └───────┘         │
                                                       │
                    Kafka: order.created ──────────────┘
                    (Payment, Inventory, Notification are consumers)
```

**Sync calls (gRPC):**
- Gateway → User Service (auth)
- Gateway → Product Service (browse)
- Gateway → Cart Service (add/remove items)
- Order Service → Payment Service (process payment)
- Order Service → Inventory Service (check/deduct stock)

**Async calls (Kafka):**
- Order Service publishes `order.created` → consumed by:
  - Notification Service (send confirmation email)
  - Analytics Service (update metrics)
  - Inventory Service (deduct stock — async if not critical path)

**API Gateway handles:** auth, routing, rate limiting, SSL termination, response caching.

**Service discovery:** Kubernetes DNS or Consul. Services register on startup; gateway queries registry.

**Payment service down:**
- Circuit breaker in Order Service trips after 5 failures → returns "payment temporarily unavailable" to user.
- Order remains in "pending" state; user can retry payment later.
- Alternatively, use async: publish `order.created` → Payment Service processes when it recovers.
</details>

---

## Common Mistakes
- Starting with microservices for a small system — "start monolith, extract when needed."
- Not handling cascading failures — one slow service shouldn't bring down everything (circuit breakers!).
- Forgetting the API gateway — drawing services without an entry point.
- Synchronous everything — some calls should be async (notifications, analytics).
- Shared database between microservices — defeats the purpose (tight coupling).

## Checklist Before Moving On
- [ ] Can explain monolith vs microservices trade-offs
- [ ] Understand service discovery (client-side vs server-side, registry)
- [ ] Know API gateway responsibilities (routing, auth, rate limit, caching, aggregation)
- [ ] Understand the circuit breaker pattern (CLOSED, OPEN, HALF-OPEN)
- [ ] Know when to use sync vs async inter-service communication
- [ ] Solved the E-Commerce microservices exercise
