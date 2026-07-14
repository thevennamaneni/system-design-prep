# Day 17 - Real-Time Systems: WebSockets, SSE & Push Notifications

## Topic
**Real-time communication** — WebSockets, Server-Sent Events (SSE), long polling, and push notifications. Design systems that deliver data instantly (chat, live updates, notifications).

---

## Why It Matters
- Chat apps (WhatsApp, Discord), live dashboards, and real-time notifications are common interview problems.
- "How do you push updates to millions of connected clients?" tests your understanding of connection management at scale.
- The choice between WebSocket, SSE, and long polling is a frequent interview question.

---

## 1. Real-Time Communication Options

### Comparison Table

| Feature | Long Polling | SSE | WebSocket |
|---------|-------------|-----|-----------|
| Direction | Client→Server (repeated) | Server→Client (one-way) | Bidirectional |
| Protocol | HTTP | HTTP | WS (upgraded from HTTP) |
| Connection | New each poll | Persistent (HTTP) | Persistent (TCP) |
| Reconnection | Manual | Automatic (browser) | Manual |
| Max connections/browser | ~6 (HTTP/1.1) | ~6 (HTTP/1.1) or unlimited (HTTP/2) | Unlimited |
| Best for | Fallback, legacy | Notifications, live feeds | Chat, gaming, collaboration |
| Complexity | Low | Low | Medium |

### When to Use Each

| Scenario | Best Choice |
|----------|-------------|
| Chat (bidirectional, frequent) | WebSocket |
| Live notification feed (server→client only) | SSE |
| Stock price ticker (server→client, stream) | SSE |
| Multiplayer game (bidirectional, low latency) | WebSocket |
| Collaborative editing (Google Docs) | WebSocket |
| Simple "new message" badge | SSE |
| Legacy browser support | Long polling |
| File upload progress | Long polling or SSE |

---

## 2. WebSocket Architecture at Scale

### The Challenge
A single server can handle ~65K WebSocket connections (TCP port limit). But what about 10 million connected users?

```
10M users → need ~200 servers (50K connections each)
  → How do you route messages to the right server?
  → How do you handle server crashes?
```

### WebSocket Cluster Architecture

```
                    ┌──────────────┐
                    │  Load Balancer │  (sticky sessions or WebSocket-aware)
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
      ┌─────────┐    ┌─────────┐    ┌─────────┐
      │WS Server1│   │WS Server2│   │WS Server3│  (each holds ~50K connections)
      │ 50K conns │   │ 50K conns │   │ 50K conns │
      └─────┬───┘    └─────┬───┘    └─────┬───┘
            │               │               │
            └───────────────┼───────────────┘
                            │
                    ┌───────┴───────┐
                    │  Redis Pub/Sub │  (message routing between servers)
                    └───────────────┘
```

### Message Routing Problem
```
User A is connected to Server 1.
User B is connected to Server 3.
User A sends a message to User B.

How does Server 1 deliver to User B on Server 3?
```

**Solution: Redis Pub/Sub (or Kafka)**
```
1. User A sends message to Server 1.
2. Server 1 publishes to Redis channel "user:B".
3. Server 3 (which has User B's connection) is subscribed to "user:B".
4. Server 3 receives the message from Redis.
5. Server 3 pushes it to User B's WebSocket connection.
```

### Connection Registry
Each server registers which users are connected to it:
```
Redis: user:connections
  "user:A" → "server-1"
  "user:B" → "server-3"
  "user:C" → "server-2"
```

When a message needs to reach User B, the server looks up which server holds the connection and routes accordingly.

### Sticky Sessions
The load balancer can use "sticky sessions" (IP hash) so the same user always connects to the same server. This simplifies connection management but makes failover harder (if the server dies, all connections must reconnect).

### Handling Server Crashes
```
Server 2 crashes → 50K users disconnect
  → Clients auto-reconnect (with backoff)
  → LB routes them to other servers
  → Connection registry updates
  → Messages resume

During the gap:
  → Messages published to Redis are buffered (if user is offline)
  → Or marked as "undelivered" and retried
```

---

## 3. Server-Sent Events (SSE) in Detail

### How SSE Works
```
Client: GET /events (Accept: text/event-stream)
  → Standard HTTP GET, but the server keeps the connection open

Server: (streams events)
  data: {"type": "like", "tweet_id": 123}\n\n
  data: {"type": "follow", "user_id": 456}\n\n
  data: {"type": "mention", "tweet_id": 789}\n\n
```

### SSE Advantages
- **Simple:** No protocol upgrade, just HTTP.
- **Auto-reconnect:** Browsers automatically reconnect on disconnect.
- **Caching-friendly:** Works with HTTP caching (unlike WebSockets).
- **Proxy-friendly:** Standard HTTP passes through all proxies.

### SSE Disadvantages
- **One-directional only** (server→client). Client can't send data over the same connection.
- **Connection limit:** HTTP/1.1 limits to 6 connections per domain (HTTP/2 removes this).
- **No binary:** Text-only (use base64 for binary, which is wasteful).

---

## 4. Push Notifications (Mobile)

For mobile devices (iOS/Android), keeping a WebSocket open drains battery. Use **push notifications** instead:

### Architecture
```
                    ┌──────────────┐
  Your Server ─────→│ APNs (Apple)  │─────→ iPhone
                    │ FCM (Google)  │─────→ Android
                    └──────────────┘
```

### How Push Works
1. App registers with APNs/FCM → gets a **device token**.
2. App sends device token to your server.
3. Your server stores: `user_id → device_token`.
4. When a notification is needed:
   - Server sends payload to APNs/FCM with the device token.
   - APNs/FCM delivers to the device (even if app is closed).
   - Device shows notification.

### Push vs WebSocket
| Aspect | WebSocket | Push Notification |
|--------|-----------|-------------------|
| App must be open? | Yes | No (delivered even when closed) |
| Battery | Drains (persistent connection) | Efficient (OS manages) |
| Latency | Very low (<50ms) | Higher (seconds to minutes) |
| Bidirectional? | Yes | No (server→device only) |
| Use case | Active chat, live data | "New message" alerts, updates |

**Typical pattern:** Use push notifications for "you have a new message," then open a WebSocket when the user opens the app for real-time delivery.

---

## 5. Presence System (Online/Offline Status)

```
User A wants to know if User B is online.

Implementation:
  1. User B connects via WebSocket → server marks B as "online" in Redis.
  2. User B disconnects → server marks B as "offline" (or uses a TTL).
  3. User A queries B's status → reads from Redis.

Redis:
  "presence:user:B" → "online" (with TTL of 60 seconds)
  
Heartbeat: User B sends a ping every 30 seconds → refreshes TTL.
If B crashes/disconnects → TTL expires → B marked offline automatically.
```

---

## 6. Real-Time System Design: WhatsApp/Chat

### High-Level Architecture
```
                    ┌──────────────┐
  Mobile/Web ──────→│  Load Balancer │ (WebSocket-aware, sticky)
                    └──────┬───────┘
           ┌───────────────┼───────────────┐
           ▼               ▼               ▼
      ┌─────────┐    ┌─────────┐    ┌─────────┐
      │WS Server1│   │WS Server2│   │WS Server3│  (connection managers)
      └─────┬───┘    └─────┬───┘    └─────┬───┘
            │               │               │
            └───────────────┼───────────────┘
                            │
              ┌─────────────┼─────────────┐
              │             │             │
        ┌─────┴─────┐ ┌────┴────┐ ┌─────┴─────┐
        │Redis PubSub│ │ Message │ │    DB     │
        │(routing)   │ │ Queue  │ │(persistence)│
        └───────────┘ └─────────┘ └───────────┘
```

### Message Flow
```
1. User A sends message to User B via WebSocket (to Server 1).
2. Server 1:
   a. Stores message in DB (Cassandra: high write throughput).
   b. Publishes to Redis Pub/Sub channel "user:B".
   c. Acknowledges to User A (message sent ✓).
3. Server 3 (holding User B's connection) receives from Redis.
4. Server 3 pushes to User B via WebSocket.
5. User B's client acknowledges receipt.
6. Server 3 updates DB: "message delivered ✓".
7. If User B is offline → send push notification via APNs/FCM.
```

### Read Receipts (Blue Ticks)
```
User B reads the message:
1. Client sends "message_read" event via WebSocket.
2. Server 3 updates DB: message.status = "read".
3. Server 3 publishes to Redis "user:A".
4. Server 1 pushes "message_read" receipt to User A.
```

### Scaling Considerations
- **Connection count:** 10M concurrent users → 200 servers (50K each).
- **Message storage:** Cassandra (write-optimized, time-series friendly).
- **Message ordering:** Within a chat, messages must be ordered. Use a per-chat sequence number (monotonic counter in Redis: `INCR chat:sequence:{chatID}`).
- **Group chats:** Fan-out to N members. Use a message queue for large groups.

---

## Go Connection

```go
// WebSocket server in Go (using nhooyr/websocket or gorilla/websocket)
import "nhooyr.io/websocket"

func wsHandler(w http.ResponseWriter, r *http.Request) {
    conn, err := websocket.Accept(w, r, nil)
    if err != nil { return }
    defer conn.Close(websocket.StatusNormalClosure, "")
    
    userID := authenticate(r)  // get user from JWT
    
    // Register connection
    connectionManager.Register(userID, conn)
    defer connectionManager.Unregister(userID)
    
    // Subscribe to user's Redis channel for incoming messages
    pubsub := redis.Subscribe(ctx, "user:"+userID)
    defer pubsub.Close()
    
    // Goroutine: forward Redis messages to WebSocket
    go func() {
        for msg := range pubsub.Channel() {
            conn.Write(ctx, websocket.MessageText, []byte(msg.Payload))
        }
    }()
    
    // Main loop: read from WebSocket
    for {
        _, data, err := conn.Read(ctx)
        if err != nil { break }  // disconnected
        
        var msg Message
        json.Unmarshal(data, &msg)
        
        // Store in DB
        db.SaveMessage(&msg)
        
        // Route to recipient via Redis
        redis.Publish(ctx, "user:"+msg.RecipientID, data)
    }
}

// Connection manager (maps user → WebSocket connection)
type ConnectionManager struct {
    connections sync.Map  // map[string]*websocket.Conn
}

func (cm *ConnectionManager) Register(userID string, conn *websocket.Conn) {
    cm.connections.Store(userID, conn)
}
func (cm *ConnectionManager) Unregister(userID string) {
    cm.connections.Delete(userID)
}
```

**Go advantage:** Go's goroutine-per-connection model is perfect for WebSockets. Each connection gets a goroutine (2KB initial stack). A single Go server handles 100K+ WebSocket connections efficiently.

---

## Exercise: Design a Live Score Update System (ESPN/Cricbuzz)

**Requirements:**
1. Millions of users watching a live cricket/football match.
2. Score updates every few seconds.
3. Users can follow specific matches.
4. Latency < 2 seconds from event to user.

**Design:**
1. WebSocket or SSE? Why?
2. How do you route updates to users following Match X?
3. How do you handle 10M concurrent connections?
4. What if a score update service crashes?

<details>
<summary>Reference Answer</summary>

1. **SSE** — score updates are one-directional (server→client). No need for bidirectional WebSocket. SSE is simpler, auto-reconnects, and works through proxies.

2. **Channel per match:** Redis Pub/Sub with a channel per match (`match:INDvsAUS`). Users subscribe to the channel for matches they follow. Score update service publishes to the match's channel. All SSE servers subscribed to that channel receive and push to their connected clients.

```
Score Update Service → Redis Pub/Sub: "match:INDvsAUS" 
  → SSE Server 1 (has 1000 users watching) → push to all 1000
  → SSE Server 2 (has 2000 users watching) → push to all 2000
```

3. **10M connections:**
   - 200 SSE servers (50K connections each).
   - Load balancer with sticky sessions (user stays on same server).
   - Each server subscribes to Redis channels for active matches.
   - Use HTTP/2 to avoid the 6-connection-per-domain limit.

4. **Score update service crash:**
   - Score data is stored in a database (source of truth).
   - On restart, the service reads the last score and resumes publishing.
   - Clients auto-reconnect (SSE behavior) and catch up.
   - Multiple score update instances (active-passive) for HA.
</details>

---

## Common Mistakes
- Using WebSockets for one-way push (notifications) — SSE is simpler.
- Not handling reconnection — clients WILL disconnect; the system must recover.
- Not using Redis Pub/Sub for cross-server message routing.
- Trying to hold 10M connections on one server — scale horizontally.
- Forgetting presence (online/offline) for chat systems.

## Checklist Before Moving On
- [ ] Know when to use WebSocket vs SSE vs long polling
- [ ] Can design a WebSocket cluster with Redis Pub/Sub routing
- [ ] Understand connection registry and sticky sessions
- [ ] Know how push notifications work (APNs/FCM)
- [ ] Can design a presence system (online/offline with TTL)
- [ ] Solved the Live Score Update exercise
