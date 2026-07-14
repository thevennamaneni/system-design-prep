# Day 3 - Scaling: Vertical vs Horizontal, Load Balancers & Proxies

## Topic
**Scaling strategies** (vertical/horizontal), **load balancers** (algorithms, health checks), and **reverse/forward proxies**. These are the first components you'll draw in any system design.

---

## Why It Matters
- Every system design interview asks: "How do you handle increased load?" — scaling is the answer.
- Load balancers are the entry point of every distributed system. You must know how they work.
- Understanding proxies (forward/reverse) is essential for discussing CDN, API gateways, and security.

---

## 1. Vertical vs Horizontal Scaling

### Vertical Scaling (Scale Up)
**Upgrade a single machine** — more CPU, RAM, disk.

```
Small Server (4 CPU, 16GB) → Bigger Server (64 CPU, 256GB)
```

**Pros:**
- Simple — no code changes needed.
- No network overhead (everything on one machine).
- Good for stateful systems (databases).

**Cons:**
- **Hard limit:** There's a maximum machine size (e.g., largest AWS instance: ~448 CPUs).
- **Single point of failure:** If the machine dies, the service dies.
- **Downtime:** Upgrading often requires a restart.
- **Cost:** High-end machines are disproportionately expensive.

### Horizontal Scaling (Scale Out)
**Add more machines** — distribute load across multiple servers.

```
1 Server (4 CPU) → 10 Servers (4 CPU each) → 100 Servers (4 CPU each)
```

**Pros:**
- **No theoretical limit:** Add as many as you need.
- **Fault tolerance:** If one dies, others handle the load.
- **No downtime:** Add/remove machines dynamically.
- **Cost-effective:** Commodity hardware is cheaper.

**Cons:**
- **Requires statelessness:** Servers can't hold session state (must use shared cache/DB).
- **Complexity:** Load balancing, distributed coordination, consistency challenges.
- **Network overhead:** Machines communicate over the network.

### The Golden Rule for Interviews
> "Design for horizontal scaling from the start."

Stateless API servers scale horizontally. Stateful components (databases, caches) require more care (replication, sharding — Days 13, 15).

```
                    ┌──────────┐
                    │  Users   │
                    └────┬─────┘
                         │
                  ┌──────┴──────┐
                  │  Load Balancer│
                  └──────┬──────┘
           ┌─────────┬───┴───┬─────────┐
           ▼         ▼       ▼         ▼
       ┌───────┐ ┌───────┐ ┌───────┐ ┌───────┐
       │Server1│ │Server2│ │Server3│ │Server4│  (stateless, identical)
       └───┬───┘ └───┬───┘ └───┬───┘ └───┬───┘
           └─────────┴───┬───┴─────────┘
                    ┌────┴────┐
                    │ Database │  (shared state)
                    └─────────┘
```

---

## 2. Load Balancers

**What it is:** A component that distributes incoming requests across multiple servers.

### Load Balancing Algorithms (MEMORIZE)

| Algorithm | How it works | Best for |
|-----------|-------------|----------|
| **Round Robin** | Requests go to servers in order (1→2→3→1→2→3) | Equal-capacity servers, simple |
| **Weighted Round Robin** | More requests to more powerful servers | Mixed-capacity servers |
| **Least Connections** | Send to server with fewest active connections | Long-lived connections (WebSocket, streaming) |
| **IP Hash** | Hash client IP → always same server | Session affinity (sticky sessions) |
| **Random** | Pick a random server | Simple, surprisingly effective |
| **Consistent Hashing** | Hash request key → same server (for cache) | Caching layers, sharded DBs (Day 13) |
| **Latency-based** | Send to server with lowest response time | Performance-sensitive apps |

### Health Checks
Load balancers continuously check server health:
```
LB → GET /health → Server 1: 200 OK ✓ (receive traffic)
LB → GET /health → Server 2: timeout  ✗ (stop sending traffic)
LB → GET /health → Server 3: 500      ✗ (stop sending traffic)
```

**Health check types:**
- **Active:** LB periodically sends a probe (e.g., `GET /health`).
- **Passive:** LB monitors real traffic — if a server returns 5xx, reduce its traffic.

### Layer 4 vs Layer 7 Load Balancing

| Aspect | Layer 4 (Transport) | Layer 7 (Application) |
|--------|---------------------|----------------------|
| Operates at | TCP/UDP level | HTTP level |
| Sees | IP, port, protocol | URL, headers, cookies, body |
| Speed | Faster (less inspection) | Slower (more inspection) |
| Features | Simple forwarding | Path-based routing, SSL termination, header manipulation |
| Use case | DB load balancing, internal services | HTTP API, web apps |
| Examples | AWS NLB, HAProxy (L4 mode) | AWS ALB, Nginx, HAProxy (L7 mode) |

**In interviews:** Most external-facing load balancers are Layer 7 (they route by URL path). Internal DB load balancers are Layer 4.

### SSL/TLS Termination at the LB
```
Client ──(HTTPS/TLS)──→ Load Balancer ──(HTTP, unencrypted)──→ Servers
```
The LB decrypts TLS, freeing servers from the CPU cost. Servers only see plaintext HTTP (within the internal network, which is trusted).

**Alternative: SSL passthrough** — LB doesn't decrypt, forwards TLS to servers. More secure (end-to-end encryption), but servers handle the CPU cost.

### Sticky Sessions (Session Affinity)
Some apps need a user to always hit the same server (e.g., if session is stored locally):
- **IP hash:** Same client IP → same server.
- **Cookie-based:** LB sets a cookie; subsequent requests include it.

**Problem:** If the server dies, the session is lost. Better to store sessions externally (Redis) and make all servers stateless.

---

## 3. Proxies

### Forward Proxy (Client-Side Proxy)
```
Client → Forward Proxy → Internet → Server
```
- Sits between client and internet.
- **Hides the client** from the server.
- Use cases: corporate firewalls, censorship circumvention (VPN), caching.

### Reverse Proxy (Server-Side Proxy)
```
Client → Internet → Reverse Proxy → Server(s)
```
- Sits between internet and servers.
- **Hides the servers** from the client.
- Use cases: load balancing, SSL termination, caching, security, rate limiting.

**Load balancers ARE reverse proxies** (they're a specific type).

### Common Reverse Proxy Software
- **Nginx:** Most popular. HTTP/HTTPS, load balancing, caching.
- **HAProxy:** High-performance TCP/HTTP load balancer.
- **Envoy:** Modern, built for microservices (used by Istio service mesh).
- **Traefik:** Container-native, auto-discovers services (K8s/Docker).

### CDN (Content Delivery Network) — Preview (Full coverage Day 6)
A CDN is a **distributed network of reverse proxy servers** that cache content close to users:
```
User in Tokyo → Tokyo CDN Edge (cached image) → no need to hit US origin
```

---

## 4. Auto-Scaling

**What it is:** Automatically add/remove servers based on load.

```
CPU > 70% for 5 min → Add 2 servers
CPU < 30% for 10 min → Remove 2 servers
```

### Auto-Scaling in System Design
- **Scale-up trigger:** CPU, memory, request queue length, custom metrics.
- **Cooldown period:** Don't thrash (add/remove too quickly).
- **Predictive scaling:** Pre-scale before expected traffic spikes (Super Bowl, Black Friday).

**In Go:** Use Kubernetes Horizontal Pod Autoscaler (HPA) or cloud auto-scaling groups.

---

## Go Connection

```go
// Go HTTP servers are inherently good at horizontal scaling:
// Each request runs in its own goroutine — no thread-per-request overhead.

// Health check endpoint:
http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
})

// Graceful shutdown (critical for load balancer health checks):
srv := &http.Server{Addr: ":8080"}
go srv.ListenAndServe()

// Wait for shutdown signal
<-ctx.Done()
srv.Shutdown(context.Background())  // finish in-flight requests, then stop
```

**Go advantage:** Go's HTTP server handles each connection in a goroutine (2KB initial stack). A single Go server can handle 100K+ concurrent connections. This makes horizontal scaling very efficient — each server handles more load.

---

## Exercise: Design a Scalable API Server

**Scenario:** You have a Go API server handling 1,000 requests/second. Traffic is expected to grow to 10,000 req/sec in 6 months.

1. What scaling strategy would you use? Why?
2. Draw the architecture with load balancing.
3. How would you handle health checks?
4. What happens if a server crashes?
5. How would you handle session state (users need to stay logged in)?

<details>
<summary>Reference Answer</summary>

1. **Horizontal scaling** — add more servers. Vertical scaling would hit limits and create a SPOF.
2. Architecture:
```
Users → DNS (round-robin or geo-DNS) → Load Balancer (L7, Nginx/ALB)
    → [Server1, Server2, ..., Server10] (stateless Go API, identical)
    → Shared Redis (session/cache) + PostgreSQL (data)
```
3. Health checks: each server exposes `GET /health` returning 200 if healthy (DB connected, Redis connected). LB checks every 5-10 seconds; removes unhealthy servers.
4. Server crash: LB health check detects failure within seconds, stops sending traffic. Auto-scaler replaces the crashed server. No user impact (other servers handle load).
5. Sessions: store in Redis (not on the server). Any server can read the session by token. This makes all servers interchangeable — true horizontal scaling.
</details>

---

## Common Mistakes
- Designing stateful API servers — they can't scale horizontally. Move state to Redis/DB.
- Forgetting health checks — dead servers stay in the pool, causing 5xx errors.
- Not understanding Layer 4 vs Layer 7 LB — interviewers ask this directly.
- Over-engineering early — start with 2-3 servers + one LB, then discuss auto-scaling as a follow-up.

## Checklist Before Moving On
- [ ] Know vertical vs horizontal scaling pros/cons
- [ ] Can explain 5+ load balancing algorithms
- [ ] Understand Layer 4 vs Layer 7 load balancing
- [ ] Know SSL termination at the LB
- [ ] Understand forward vs reverse proxy
- [ ] Solved the Scalable API Server exercise
