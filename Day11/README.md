# Day 11 - API Design: gRPC, GraphQL & Advanced Patterns

## Topic
**gRPC** (high-performance RPC), **GraphQL** (flexible queries), **idempotency**, **webhooks**, and **long polling/SSE/WebSockets** for real-time APIs. These complement REST for specific use cases.

---

## Why It Matters
- REST isn't always the right choice. Senior engineers know when to use gRPC (service-to-service) vs GraphQL (client-driven queries) vs REST (public APIs).
- Real-time API patterns (WebSockets, SSE, long polling) are essential for chat, notifications, and live updates.
- Interviewers test: "Why not REST for this?" — you need alternatives.

---

## 1. gRPC (Google RPC)

**What it is:** A high-performance RPC framework using **Protocol Buffers** (protobuf) for serialization and **HTTP/2** for transport.

### How gRPC Works
```
1. Define service in .proto file:
   service OrderService {
     rpc CreateOrder (CreateOrderRequest) returns (Order);
     rpc GetOrder (GetOrderRequest) returns (Order);
   }

2. Generate Go code from proto (protoc compiler)

3. Client calls generated function:
   order, err := client.CreateOrder(ctx, &CreateOrderRequest{...})

4. gRPC handles: serialization, HTTP/2, streaming, errors
```

### gRPC vs REST

| Feature | gRPC | REST |
|---------|------|------|
| Protocol | HTTP/2 | HTTP/1.1 (usually) |
| Serialization | Protobuf (binary, compact) | JSON (text, larger) |
| Performance | 5-10x faster | Baseline |
| Schema | Strong (proto file) | Weak (OpenAPI optional) |
| Streaming | Bidirectional streaming | No (request-response only) |
| Browser support | Requires gRPC-Web proxy | Native |
| Code generation | Built-in | Optional (OpenAPI tools) |
| Use case | Service-to-service | Public APIs, browser |

### When to Use gRPC
- **Service-to-service communication** in microservices (internal APIs).
- **High-throughput** data transfer (gRPC is 5-10x faster than REST/JSON).
- **Streaming** (bi-directional streaming for real-time data).
- **Strong contracts** (proto file enforces schema; changes are explicit).

### When NOT to Use gRPC
- **Browser-facing APIs** (requires gRPC-Web proxy; REST is simpler).
- **Public APIs** (REST is more universally understood and toolable).
- **Quick prototyping** (protobuf adds overhead).

### gRPC Streaming Modes
```
1. Unary:        Client → Request → Server → Response (like REST)
2. Server stream: Client → Request → Server → Stream of responses
3. Client stream: Client → Stream of requests → Server → Response
4. Bidirectional: Client ↔ Server (full-duplex stream)
```

---

## 2. GraphQL

**What it is:** A query language for APIs where the **client** specifies exactly what data it needs. Single endpoint returns precisely the requested fields.

### GraphQL vs REST

| Feature | REST | GraphQL |
|---------|------|---------|
| Endpoints | Multiple (`/users`, `/tweets`, `/comments`) | Single (`/graphql`) |
| Response shape | Server-defined | Client-defined |
| Over-fetching | Common (full user when you need name) | Eliminated (ask for name only) |
| Under-fetching | Common (need 3 requests for user + tweets + comments) | Eliminated (nested query in 1 request) |
| Versioning | URL versioning (`/v2/`) | Schema evolution (deprecate fields) |
| Caching | HTTP caching (CDN, browser) | Harder (single endpoint, POST) |
| Learning curve | Low | Medium |

### Example GraphQL Query
```graphql
# Client requests exactly what it needs:
query {
  user(id: "123") {
    name
    email
    tweets(limit: 5) {
      content
      created_at
      likes_count
    }
    followers {
      name
    }
  }
}

# Response (exactly the requested fields):
{
  "user": {
    "name": "Alice",
    "email": "alice@example.com",
    "tweets": [
      {"content": "Hello!", "created_at": "...", "likes_count": 42}
    ],
    "followers": [
      {"name": "Bob"},
      {"name": "Charlie"}
    ]
  }
}
```

### When to Use GraphQL
- **Complex UIs** needing data from multiple resources (mobile apps, dashboards).
- **Eliminating over-fetching/under-fetching** (mobile bandwidth is precious).
- **Rapid frontend iteration** (frontend can change queries without backend changes).

### When NOT to Use GraphQL
- **Simple APIs** (REST is simpler).
- **Caching-heavy** scenarios (HTTP caching is harder with GraphQL's single POST endpoint).
- **File uploads** (GraphQL doesn't handle binary well; REST is better).

### GraphQL Architecture
```
Client → GraphQL Server (single endpoint)
              │
    ┌─────────┼─────────┐
    ▼         ▼         ▼
  User     Tweet    Comment
  Service  Service  Service  (resolvers fetch from each)
```

The GraphQL server has **resolvers** — functions that fetch data for each field. Resolvers can call REST APIs, databases, or other services.

---

## 3. API Communication Patterns

### a) Request-Response (REST/gRPC unary)
```
Client → Request → Server → Response
```
- Standard pattern. Client initiates, server responds.
- Use for: most CRUD operations.

### b) Webhooks (Server-to-Server Callback)
```
1. Client registers a webhook URL with the server:
   POST /webhooks { "url": "https://client.com/callback", "event": "order.created" }

2. When event occurs, server calls the client:
   POST https://client.com/callback
   { "event": "order.created", "data": {...} }
```
- Use for: payment processing (Stripe webhooks), CI/CD (GitHub webhooks), notifications.
- **Idempotency required:** webhooks may be delivered multiple times.

### c) Long Polling
```
1. Client sends request
2. Server holds the connection open (doesn't respond immediately)
3. When data is available, server responds
4. Client immediately sends another request (re-polls)
```
```
Client: GET /messages?wait=30  (wait up to 30 seconds)
  ... 25 seconds later ...
Server: 200 OK { "messages": [...] }
Client: GET /messages?wait=30  (immediately re-poll)
```
- Use for: simple real-time updates without WebSocket infrastructure.
- **Downside:** resource-intensive (each client holds a connection).

### d) Server-Sent Events (SSE)
```
Client: GET /events  (Accept: text/event-stream)
Server: (keeps connection open, pushes events)
  data: {"type": "like", "tweet_id": 123}
  data: {"type": "retweet", "tweet_id": 456}
```
- **One-directional** (server → client only).
- Uses standard HTTP (no protocol upgrade like WebSocket).
- Auto-reconnects on disconnect.
- Use for: live notifications, stock tickers, dashboard updates.
- **Better than long polling** for one-way push.

### e) WebSockets (covered Day 17)
- **Bidirectional** (full-duplex).
- Use for: chat, gaming, collaborative editing.

### Summary: Which Pattern?

| Need | Pattern |
|------|---------|
| CRUD operations | REST |
| High-perf service-to-service | gRPC |
| Flexible client queries | GraphQL |
| Server-to-server events | Webhooks |
| One-way push (notifications) | SSE |
| Bidirectional real-time (chat) | WebSocket |
| Simple polling (no infra) | Long polling |

---

## 4. Idempotency in Depth

### Idempotency Key Pattern
```
POST /payments
  Idempotency-Key: 7c8d2b3a-...
  Body: { "order_id": "order_123", "amount": 50.00 }

Server:
  1. Check: is key 7c8d2b3a already processed?
     → Yes: return cached result (don't charge again)
     → No: process payment, store result with key, return
```

### Implementation
```go
type IdempotencyStore struct {
    cache *redis.Client
}

func (s *IdempotencyStore) Execute(ctx context.Context, key string, fn func() (any, error)) (any, error) {
    // Try to acquire lock for this key
    locked, err := s.cache.SetNX(ctx, "idem:"+key, "processing", 5*time.Minute).Result()
    if !locked {
        // Key already exists — return cached result
        result, _ := s.cache.Get(ctx, "idem:"+key+":result").Result()
        return result, nil
    }
    
    // Execute the operation
    result, err := fn()
    if err != nil {
        s.cache.Del(ctx, "idem:"+key)  // release lock on failure
        return nil, err
    }
    
    // Cache the result
    s.cache.Set(ctx, "idem:"+key+":result", result, 24*time.Hour)
    return result, nil
}
```

---

## Go Connection

```go
// gRPC server in Go
import (
    "google.golang.org/grpc"
    pb "path/to/generated/proto"
)

type orderService struct {
    pb.UnimplementedOrderServiceServer
}

func (s *orderService) CreateOrder(ctx context.Context, req *pb.CreateOrderRequest) (*pb.Order, error) {
    order, err := service.Create(req)
    return &pb.Order{Id: order.ID, Status: order.Status}, err
}

lis, _ := net.Listen("tcp", ":50051")
grpcServer := grpc.NewServer()
pb.RegisterOrderServiceServer(grpcServer, &orderService{})
grpcServer.Serve(lis)

// gRPC client
conn, _ := grpc.Dial("localhost:50051", grpc.WithInsecure())
client := pb.NewOrderServiceClient(conn)
order, _ := client.CreateOrder(ctx, &pb.CreateOrderRequest{UserId: 123})

// SSE in Go
func sseHandler(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")
    
    flusher, _ := w.(http.Flusher)
    for {
        event := waitForEvent()
        fmt.Fprintf(w, "data: %s\n\n", event)
        flusher.Flush()  // send immediately
    }
}
```

**Go advantage:** Go has first-class gRPC support (protobuf generation, streaming). It's the most popular language for gRPC services.

---

## Exercise: Choose API Styles for Instagram

For each interaction, choose the best API style (REST, gRPC, GraphQL, WebSocket, SSE, Webhook) and justify:

1. Mobile app fetching a user's feed (needs different fields for different screens)
2. Internal service: photo processing service notifying the upload service
3. Live "user is typing..." indicator in direct messages
4. Pushing new like notifications to a user's browser
5. Image upload service communicating with the ML tagging service
6. Third-party API for developers (public API)
7. Live video stream comments during an Instagram Live

<details>
<summary>Reference Answer</summary>

| Interaction | API Style | Why |
|------------|-----------|-----|
| Mobile feed | **GraphQL** | Different screens need different fields; eliminates over-fetching on mobile bandwidth; nested data (user + posts + comments) in one query |
| Photo → Upload notify | **gRPC** | Service-to-service, high throughput, strong contract via protobuf |
| "User is typing" | **WebSocket** | Bidirectional, real-time, frequent small messages |
| Like notification → browser | **SSE** | One-way push (server→client), simple, auto-reconnect, no WebSocket infra needed |
| Image → ML tagging | **gRPC** | Internal service-to-service, high performance, streaming for batch processing |
| Third-party public API | **REST** | Universal, well-tooled, easy for external developers |
| Live video comments | **WebSocket** | Real-time, bidirectional, high frequency |

**Key insight:** One system uses 5+ API styles for different interactions. REST is the public face; gRPC is the internal backbone; WebSocket/SSE handle real-time.
</details>

---

## Common Mistakes
- Using REST for everything — some interactions need gRPC (performance) or GraphQL (flexibility).
- Using GraphQL for internal services — gRPC is better (stronger contracts, better performance).
- Forgetting idempotency keys on POST requests that create resources or process payments.
- Using long polling when SSE would be simpler (one-way push).
- Not understanding the difference between SSE (one-way) and WebSocket (two-way).

## Checklist Before Moving On
- [ ] Know when to use REST vs gRPC vs GraphQL
- [ ] Can explain gRPC's advantages (protobuf, HTTP/2, streaming)
- [ ] Understand GraphQL's strengths (no over-fetching, single endpoint)
- [ ] Know the 5 API communication patterns (request-response, webhook, long polling, SSE, WebSocket)
- [ ] Can implement idempotency keys for safe retries
- [ ] Solved the Instagram API style exercise
