# Day 2 - Networking Fundamentals: DNS, TCP/IP, HTTP, TLS

## Topic
**Networking fundamentals** that every system design interview assumes you know: DNS, TCP/IP, HTTP/HTTPS, TLS, WebSockets, and how data flows across the internet.

---

## Why It Matters
- Every system design involves clients communicating with servers over a network.
- Interviewers expect you to explain how a request travels from browser to server.
- Understanding networking is the foundation for discussing load balancing, CDNs, API gateways, and real-time communication.

---

## The Request Journey (How a URL Becomes a Response)

```
User types: https://api.twitter.com/timeline

1. DNS Resolution
   api.twitter.com → 104.244.42.194 (IP address)
   
2. TCP Connection
   Client ↔ Server: 3-way handshake (SYN, SYN-ACK, ACK)
   
3. TLS Handshake (for HTTPS)
   Client ↔ Server: exchange certificates, agree on encryption keys
   
4. HTTP Request
   GET /timeline HTTP/1.1
   Host: api.twitter.com
   Authorization: Bearer <token>
   
5. Server Processing
   Load balancer → API server → database → response
   
6. HTTP Response
   HTTP/1.1 200 OK
   Content-Type: application/json
   {"tweets": [...]}
   
7. TCP Connection Close (or keep-alive)
```

---

## 1. DNS (Domain Name System)

**What it is:** A hierarchical, distributed system that translates human-readable domain names (`api.twitter.com`) to IP addresses (`104.244.42.194`).

### DNS Resolution Flow
```
Browser cache → OS cache → Resolver (ISP) cache →
Root nameserver → TLD nameserver (.com) → Authoritative nameserver (twitter.com)
```

### DNS in System Design
- **DNS-based load balancing:** Route users to the nearest data center (geo-DNS).
  - `api.twitter.com` → US users get a US IP, EU users get an EU IP.
- **TTL (Time to Live):** DNS records have a TTL. Changes take time to propagate.
  - Low TTL (60s): fast failover, more DNS queries.
  - High TTL (24h): fewer queries, slow failover.
- **Round-robin DNS:** One domain → multiple IPs. Simple but doesn't consider server health.

### When Interviewers Ask About DNS
- "How would you route users to the nearest region?" → Geo-DNS
- "How do you handle DNS failures?" → Multiple DNS providers, local caching
- "What's the DNS lookup cost?" → Typically 20-120ms (cached: <1ms)

---

## 2. TCP/IP Model

### The 4 Layers
```
┌─────────────────────────────────┐
│  Application Layer              │  HTTP, gRPC, DNS, SMTP
│  (data format, semantics)       │
├─────────────────────────────────┤
│  Transport Layer                │  TCP (reliable), UDP (fast)
│  (process-to-process)           │
├─────────────────────────────────┤
│  Internet Layer                 │  IP (routing)
│  (host-to-host)                 │
├─────────────────────────────────┤
│  Link Layer                     │  Ethernet, Wi-Fi
│  (physical transmission)        │
└─────────────────────────────────┘
```

### TCP vs UDP (MEMORIZE)

| Aspect | TCP | UDP |
|--------|-----|-----|
| Reliability | Reliable (guaranteed delivery) | Unreliable (best-effort) |
| Order | Ordered delivery | No ordering guarantee |
| Connection | Connection-oriented (3-way handshake) | Connectionless |
| Overhead | Higher (headers, ACKs) | Lower |
| Use cases | HTTP, gRPC, file transfer, DB connections | DNS, video streaming, VoIP, gaming |
| Flow control | Yes (sliding window) | No |
| Congestion control | Yes | No |

### When to Use Which
- **TCP:** Most API communication, database queries, file transfers — anything that needs reliability.
- **UDP:** Live video/audio (acceptable to lose a frame), DNS (single request/response, retransmit if needed), gaming (low latency matters more than reliability).

### TCP Connection Establishment (3-Way Handshake)
```
Client                    Server
  │                          │
  │ ── SYN (seq=x) ────────→ │
  │                          │
  │ ←── SYN-ACK (seq=y, ──── │
  │       ack=x+1)           │
  │                          │
  │ ── ACK (ack=y+1) ──────→ │
  │                          │
  │ ═══ Connection ═════════ │
  │     established           │
```

This adds latency (1 RTT) before the first byte of data. For HTTPS, there's an additional TLS handshake (1-2 more RTTs). HTTP/2 and HTTP/3 (QUIC over UDP) reduce this.

---

## 3. HTTP (HyperText Transfer Protocol)

### HTTP/1.1 vs HTTP/2 vs HTTP/3

| Feature | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---------|----------|--------|--------|
| Transport | TCP | TCP | UDP (QUIC) |
| Multiplexing | No (one request per connection) | Yes (multiple requests per connection) | Yes |
| Header compression | No | Yes (HPACK) | Yes (QPACK) |
| Server push | No | Yes | Yes |
| Connection setup | 1-RTT TCP + 1-RTT TLS | Same | 0-RTT (QUIC) |

### HTTP Methods (for API design)
```
GET    — read resource (idempotent, safe)
POST   — create resource (not idempotent)
PUT    — update/replace resource (idempotent)
DELETE — delete resource (idempotent)
PATCH  — partial update (not necessarily idempotent)
```

**Idempotency** is critical in distributed systems: if a request times out and is retried, the result should be the same. This is why `PUT` and `DELETE` should be idempotent.

### HTTP Status Codes (know these ranges)
```
2xx Success:  200 OK, 201 Created, 204 No Content
3xx Redirect: 301 Moved Permanently, 302 Found, 304 Not Modified
4xx Client error: 400 Bad Request, 401 Unauthorized, 403 Forbidden, 404 Not Found, 429 Too Many Requests
5xx Server error: 500 Internal Server Error, 502 Bad Gateway, 503 Service Unavailable, 504 Gateway Timeout
```

### Keep-Alive
In HTTP/1.1, connections can be reused (keep-alive) to avoid the TCP handshake overhead for subsequent requests. In HTTP/2, a single connection handles multiple parallel streams.

---

## 4. TLS (Transport Layer Security)

**What it is:** Encrypts data between client and server to prevent eavesdropping, tampering, and forgery. HTTPS = HTTP over TLS.

### TLS Handshake (Simplified)
```
Client                                    Server
  │                                          │
  │ ── ClientHello (TLS version, ──────────→ │
  │       cipher suites, random)             │
  │                                          │
  │ ←── ServerHello (chosen cipher, ──────── │
  │        random, certificate)              │
  │                                          │
  │ ── Key exchange (pre-master secret) ────→ │
  │                                          │
  │ ←── Finished (encrypted) ──────────────── │
  │                                          │
  │ ── Finished (encrypted) ────────────────→ │
  │                                          │
  │ ═══ Secure session established ═════════ │
```

### TLS in System Design
- **Termination:** TLS can be terminated at the load balancer (LB decrypts) or at the API server.
  - LB termination: less CPU on servers, simpler cert management. Common.
  - Server termination: end-to-end encryption. More secure, more CPU.
- **mTLS (mutual TLS):** Both client and server present certificates. Used for service-to-service communication in microservices (e.g., Istio service mesh).
- **Certificate management:** Use Let's Encrypt (free), ACM (AWS), or cert-manager (K8s).

---

## 5. WebSockets (Preview — Full Coverage Day 17)

**What it is:** A protocol that provides **full-duplex** communication over a single TCP connection. Unlike HTTP (request-response), WebSocket allows the server to push data to the client.

```
HTTP:     Client → Request → Server → Response → Client (one direction at a time)
WebSocket: Client ↔ Server (bidirectional, persistent connection)
```

### When to Use WebSockets
- Chat applications (WhatsApp, Discord)
- Real-time notifications
- Live dashboards (stock prices, sports scores)
- Collaborative editing (Google Docs)
- Multiplayer games

### WebSocket Connection Upgrade
```
Client sends HTTP request with Upgrade header:
  GET /ws HTTP/1.1
  Upgrade: websocket
  Connection: Upgrade

Server responds:
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
```

After upgrade, the connection is a WebSocket — full-duplex, persistent.

---

## Go Connection

Go's networking is world-class:
```go
// HTTP server
http.HandleFunc("/api", handler)
http.ListenAndServe(":8080", nil)

// HTTP client with timeout
client := &http.Client{Timeout: 10 * time.Second}
resp, err := client.Get("https://api.example.com/data")

// WebSocket (using gorilla/websocket or nhooyr/websocket)
conn, err := websocket.Dial(ctx, "wss://api.example.com/ws", nil)

// gRPC (HTTP/2 based)
lis, _ := net.Listen("tcp", ":50051")
grpcServer := grpc.NewServer()
pb.RegisterMyServiceServer(grpcServer, &myService{})
grpcServer.Serve(lis)

// Context for timeouts/cancellation
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
```

**Go advantage:** `net/http` uses goroutines per connection — a single Go server can handle millions of concurrent connections efficiently. This is why Go powers many of the world's largest services.

---

## Exercise: Trace a Request

**Scenario:** A user opens `https://api.uber.com/trips` on their phone.

Trace the complete journey:
1. What happens at the DNS level?
2. What TCP connections are established?
3. What TLS handshakes occur?
4. What HTTP request is sent?
5. What components process the request on the server side?
6. What response comes back?

Draw the full flow as an ASCII diagram.

<details>
<summary>Reference Answer</summary>

```
Phone                          DNS Resolver              Uber Servers
  │                                 │                        │
  │── DNS lookup: api.uber.com ───→│                        │
  │←── 104.x.x.x (IP) ─────────────│                        │
  │                                 │                        │
  │── TCP SYN ──────────────────────────────────────────────→│ (Load Balancer)
  │←── TCP SYN-ACK ──────────────────────────────────────────│
  │── TCP ACK ──────────────────────────────────────────────→│
  │   (TCP connection established)                            │
  │                                                           │
  │── TLS ClientHello ──────────────────────────────────────→│
  │←── TLS ServerHello + Certificate ────────────────────────│
  │── TLS Key Exchange ─────────────────────────────────────→│
  │←── TLS Finished ─────────────────────────────────────────│
  │── TLS Finished ─────────────────────────────────────────→│
  │   (TLS session established)                               │
  │                                                           │
  │── GET /trips HTTP/1.1 ──────────────────────────────────→│ LB → API Server
  │   Authorization: Bearer <JWT>                             │ → Auth check
  │   Host: api.uber.com                                      │ → DB query
  │                                                           │ → Redis cache
  │←── HTTP 200 OK ─────────────────────────────────────────│
  │   Content-Type: application/json                          │
  │   {"trips": [...]}                                        │
  │                                                           │
  │── TCP FIN ──────────────────────────────────────────────→│
  │←── TCP FIN-ACK ──────────────────────────────────────────│
```
</details>

---

## Common Mistakes
- Not knowing the difference between TCP and UDP — interviewers test this directly.
- Forgetting that HTTPS adds TLS handshake latency (2 RTTs in HTTP/1.1).
- Confusing HTTP methods' idempotency — critical for retry-safe API design.
- Not knowing when to use WebSockets vs HTTP polling — Day 17 will cover this deeply.

## Checklist Before Moving On
- [ ] Can explain the full request journey (DNS → TCP → TLS → HTTP → Server → Response)
- [ ] Know TCP vs UDP differences and when to use each
- [ ] Understand HTTP methods and idempotency
- [ ] Know HTTP status code ranges
- [ ] Understand TLS termination at the load balancer
- [ ] Traced the Uber request exercise
