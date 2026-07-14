# Day 24 - Full System Design: Chat System (WhatsApp)

## Topic
**Design a chat application** (WhatsApp/Discord/Telegram). Covers real-time messaging, message delivery guarantees, presence, and group chats.

---

## Why This Problem Matters
- Chat is the #1 real-time system design question.
- It tests WebSocket management at scale, message ordering, delivery guarantees, and offline message handling.
- It combines real-time communication (Day 17) with message queues (Day 9) and distributed consistency (Day 16).

---

## RESHADED Framework

### R — Requirements (5 min)

**Functional:**
1. One-on-one chat (send/receive text messages).
2. Group chat (multiple participants).
3. Message delivery status: sent ✓, delivered ✓✓, read ✓✓ (blue ticks).
4. Online/offline presence.
5. Media sharing (images, videos) — stretch.
6. Message history (scroll back).

**Non-functional:**
1. Low latency: < 200ms for message delivery.
2. High availability: 99.99%.
3. Messages must not be lost (durability).
4. Messages must be ordered within a conversation.
5. 500M DAU, 10B messages/day.

**Clarifying questions:**
- "End-to-end encryption?" → Mention it as a consideration; not core to the design.
- "Max group size?" → Assume up to 200 members.
- "Message retention?" → 1 year.
- "Mobile + web?" → Both.

### E — Estimation (3 min)

```
DAU: 500M
Messages/day: 10B
Messages/sec (avg): 10B / 86400 ≈ 116,000/sec
Messages/sec (peak): ~350,000/sec

Storage:
  Message: 200 bytes (text + metadata)
  Daily: 10B × 200B = 2TB/day
  1 year: 730TB
  With media (20% of messages, 1MB avg): 10B × 20% × 1MB = 2PB/day → 730PB/year
  (Media dominates storage → use S3 with CDN)

Concurrent connections: 50M users online at any time
  → 50M WebSocket connections
  → 1,000 servers (50K connections each)
```

### S — Storage Schema (5 min)

```
Cassandra: messages (high write throughput, time-series)
  messages (chat_id, message_id, sender_id, content, timestamp, status)
  Partition key: chat_id (all messages in a chat on one partition)
  Clustering: timestamp DESC (sorted by time)

PostgreSQL: users, contacts
  users (id, name, phone, created_at)
  contacts (user_id, contact_id)

Redis: 
  presence:user:{id} → "online" (TTL 60s, refreshed by heartbeat)
  connections:user:{id} → "server-3" (which WS server holds the connection)
  sequence:chat:{id} → counter (message ordering per chat)

S3 + CDN: media (images, videos)
```

### H — High-Level Design (10 min)

```
┌─────────┐     ┌──────────┐
│ Mobile/  │────→│ Load Bal. │ (WebSocket-aware, sticky)
│ Web      │     └─────┬────┘
└─────────┘           │
           ┌──────────┼──────────┐
           ▼          ▼          ▼
      ┌─────────┐┌─────────┐┌─────────┐
      │WS Server││WS Server││WS Server│  (50K connections each)
      │  1      ││  2      ││  3      │
      └────┬────┘└────┬────┘└────┬────┘
           │          │          │
           └──────────┼──────────┘
                      │
           ┌──────────┼──────────┐
           ▼          ▼          ▼
      ┌────────┐ ┌────────┐ ┌────────┐
      │ Redis  │ │ Kafka  │ │Cassandra│
      │(routing│ │(message│ │(message │
      │presence│ │ queue) │ │ storage)│
      └────────┘ └────┬───┘ └────────┘
                      │
                      ▼
                 ┌──────────┐
                 │ Message  │ (processes messages, routes to recipients)
                 │ Service  │
                 └──────────┘
```

### A — API Design (5 min)

```
WebSocket events (client → server):
  { "type": "message", "chat_id": "c123", "content": "Hello!" }
  { "type": "read_receipt", "chat_id": "c123", "message_id": "m456" }
  { "type": "typing", "chat_id": "c123" }
  { "type": "heartbeat" }  (presence ping every 30s)

WebSocket events (server → client):
  { "type": "message", "chat_id": "c123", "message_id": "m789", "sender_id": "u1", "content": "Hi!" }
  { "type": "status", "message_id": "m789", "status": "delivered" }
  { "type": "presence", "user_id": "u2", "status": "online" }
  { "type": "typing", "chat_id": "c123", "user_id": "u2" }

REST API:
  GET /api/v1/chats          — list user's chats
  GET /api/v1/chats/{id}/messages?cursor=abc — message history (paginated)
  POST /api/v1/chats         — create new chat/group
  POST /api/v1/chats/{id}/members — add member to group
```

### D — Detailed Design (15 min)

#### Message Flow (The Core)

**User A sends a message to User B:**

```
1. User A's client sends via WebSocket: { type: "message", chat_id: "c123", content: "Hello" }
   → reaches WS Server 1.

2. WS Server 1:
   a. Get sequence number: INCR sequence:chat:c123 → 42
   b. Create message_id (Snowflake: timestamp + server_id + seq)
   c. Store message in Cassandra: (chat_id, message_id, sender_id=A, content, timestamp, status=sent)
   d. Publish to Kafka: topic "messages", key=chat_id
   e. Send acknowledgment to User A: { type: "ack", message_id: "m42", status: "sent" }

3. Message Service (Kafka consumer):
   a. Receives the message event.
   b. Looks up chat participants: chat c123 has [A, B].
   c. For each recipient (B):
      - Check presence: GET presence:user:B → "online"?
      - If online: find connection server: GET connections:user:B → "server-3"
      - Publish to Redis Pub/Sub: channel "server-3" → { type: "message", recipient: B, ... }
   d. If B is offline: send push notification via APNs/FCM.

4. WS Server 3 (holds B's connection):
   a. Receives from Redis Pub/Sub.
   b. Pushes to User B's WebSocket: { type: "message", ... }

5. User B's client receives:
   a. Sends read receipt: { type: "read_receipt", message_id: "m42" }

6. Read receipt flows back to User A (reverse of steps 3-4).
   → User A sees blue ticks ✓✓.
```

#### Message Ordering
```
Each chat has a monotonically increasing sequence number (Redis INCR).
message_id = Snowflake(timestamp + server_id + sequence)
→ Messages within a chat are ordered by sequence number.
→ No race conditions (INCR is atomic).
```

#### Delivery Guarantees
- **At-least-once:** Messages are stored in Cassandra before delivery. If delivery fails, the Message Service retries.
- **Idempotent delivery:** Client tracks `message_id`s it has seen; deduplicates.
- **Offline messages:** If B is offline, the message is in Cassandra. When B reconnects, WS Server fetches undelivered messages from Cassandra and pushes them.

#### Presence System
```
User B connects via WebSocket:
  → SET presence:user:B "online" EX 60  (TTL 60 seconds)

User B sends heartbeat every 30 seconds:
  → EXPIRE presence:user:B 60  (refresh TTL)

User B disconnects (or crashes):
  → TTL expires after 60 seconds → presence:user:B deleted → "offline"

User A checks B's presence:
  → GET presence:user:B → "online" or nil (offline)
```

#### Group Chat
```
Chat c123 has 200 members.
User A sends a message.

Message Service:
1. Look up all 200 members (from PostgreSQL: SELECT member_id FROM chat_members WHERE chat_id = 'c123').
2. For each online member: route via Redis Pub/Sub to their WS server.
3. For offline members: store for later delivery + push notification.

Optimization: batch Redis Pub/Sub publishes (group by WS server).
  → 200 members across 50 WS servers → 50 publishes (not 200).
```

#### Media Sharing
```
1. Client uploads image to S3 via presigned URL (direct upload, bypasses WS).
2. Client sends message: { type: "message", content: "", media_url: "s3://..." }.
3. Recipient's client fetches image from S3 (via CDN for speed).
```

### E — Edge Cases (5 min)
- **User in multiple sessions (phone + web):** All sessions receive the message (broadcast to all connections of the user).
- **Message sent while recipient is offline:** Store in Cassandra; deliver on reconnect. Also send push notification.
- **Network flickering:** Client retries with the same message (idempotency via client-generated message ID). Server deduplicates.
- **Group chat with 200 members:** Fan-out is 200x. Use batched Redis publishes. For very large groups (broadcast channels), use Kafka consumer-per-member or a pull model.
- **Message deletion:** Store a tombstone (message marked as deleted). Clients remove it from the UI.

### D — Discussion (5 min)
- **Bottleneck:** 50M concurrent WebSocket connections → 1,000 servers. Redis Pub/Sub bandwidth.
- **Alternative to Redis Pub/Sub:** Kafka per WS server (overkill for this use case). Redis is simpler and fast enough.
- **End-to-end encryption:** Messages encrypted on client before sending. Server stores ciphertext. Only recipients can decrypt. (Mention this as a design consideration; WhatsApp does this.)
- **Scaling:** Shard Cassandra by `chat_id`. Add more WS servers as connections grow. Redis cluster for presence/routing.

---

## Exercise: Design Message Search

Add the ability to search within chat history:
1. User searches for "project deadline" across all their chats.
2. Return matching messages with context.
3. Latency < 500ms.

**Think about:**
- Which data store for search? (Elasticsearch)
- How to index messages? (CDC from Cassandra → ES)
- Privacy (user can only search their own messages).

<details>
<summary>Approach</summary>

1. **Elasticsearch** for full-text search.
2. **CDC:** Debezium tails Cassandra's commit log → publishes to Kafka → Elasticsearch connector indexes.
3. **Index:** `messages` index with fields: `message_id`, `chat_id`, `sender_id`, `content`, `timestamp`. Only index messages where the user is a participant.
4. **Query:** `GET /api/v1/search?q=project+deadline` → Elasticsearch query (content matches "project deadline", filtered by chats the user is a member of) → return matching messages with chat context.
5. **Privacy:** Elasticsearch query includes a filter: `chat_id IN (user's chat list)`. This ensures users can only search their own conversations.
6. **Latency:** Elasticsearch is sub-100ms for full-text queries. Plus network overhead → < 500ms easily.
</details>

---

## Common Mistakes
- Not handling offline users (messages lost if recipient is offline).
- Not guaranteeing message ordering (use sequence numbers per chat).
- Trying to send media over WebSocket (use S3 + presigned URLs).
- Not using Redis Pub/Sub for cross-server routing (each server can't know about all connections).
- Forgetting delivery/read receipts (users expect to see ✓✓).

## Checklist Before Moving On
- [ ] Can design a WebSocket cluster with Redis Pub/Sub routing
- [ ] Understand the full message flow (send → store → route → deliver → receipt)
- [ ] Know how to handle offline users (store + push notification)
- [ ] Can design presence (Redis with TTL + heartbeat)
- [ ] Understand message ordering (per-chat sequence numbers)
- [ ] Solved the Chat System design using RESHADED
