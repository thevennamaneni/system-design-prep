# Day 9 - Message Queues & Event-Driven Architecture

## Topic
**Message queues** (Kafka, RabbitMQ, SQS) and **event-driven architecture**. Decouple services, handle asynchronous processing, and build resilient pipelines.

---

## Why It Matters
- Message queues solve the #1 problem in distributed systems: **coupling**. When Service A calls Service B synchronously, B's failure breaks A.
- Event-driven architecture is the backbone of modern systems (Uber, Netflix, LinkedIn all use Kafka heavily).
- Interviewers expect you to propose async processing for anything that doesn't need to be synchronous.

---

## 1. Why Message Queues?

### The Synchronous Problem
```
User signs up →
  1. Create user in DB (50ms)
  2. Send welcome email (2s) ← slow!
  3. Send SMS notification (1s) ← slow!
  4. Update analytics (500ms)
  5. Create default profile (100ms)

Total: ~4 seconds. User waits. If email service is down, signup fails.
```

### The Async Solution
```
User signs up →
  1. Create user in DB (50ms)
  2. Publish "user.created" event to queue (<5ms)
  → Return success to user (total: 55ms)

Background consumers:
  - Email service: consumes event → sends email
  - SMS service: consumes event → sends SMS
  - Analytics service: consumes event → updates metrics
  - Profile service: consumes event → creates profile

If email service is down → event stays in queue → retried later. Signup still succeeds.
```

### Benefits
1. **Decoupling:** Producers don't know about consumers. Add new consumers without touching producers.
2. **Resilience:** If a consumer is down, messages queue up. No data loss.
3. **Scalability:** Add more consumers to handle higher load.
4. **Async processing:** Users don't wait for slow operations.
5. **Load leveling:** Smooth out traffic spikes (queue absorbs the burst).

---

## 2. Message Queue Patterns

### a) Point-to-Point (Work Queue)
```
Producer → [Queue] → Consumer (one consumer processes each message)
```
- Each message is processed by **one** consumer.
- Use cases: task queues, job processing, email sending.
- Example: AWS SQS, RabbitMQ.

### b) Publish/Subscribe (Pub/Sub)
```
                    ┌→ Consumer A (email)
Producer → [Topic] ─┼→ Consumer B (analytics)
                    └→ Consumer C (SMS)
```
- Each message is delivered to **all** subscribers.
- Use cases: event notifications, fan-out.
- Example: Kafka, Google Pub/Sub, SNS.

### c) Competing Consumers
```
                    ┌→ Consumer 1
Producer → [Queue] ─┼→ Consumer 2  (competing for messages)
                    └→ Consumer 3
```
- Multiple instances of the same consumer; each message goes to one.
- Use cases: horizontal scaling of workers.
- Example: SQS with multiple polling consumers.

---

## 3. Kafka Deep Dive

Apache Kafka is the most frequently discussed message queue in system design interviews.

### Kafka Architecture
```
Producer                 Kafka Cluster                    Consumer
   │                     ┌──────────────┐                   │
   │── Produce ────────→ │  Broker 1     │ ←── Consume ────│
   │                     │  ┌─────────┐  │                   │
   │                     │  │ Topic A  │  │                   │
   │                     │  │ Partition│  │                   │
   │                     │  │  0: [msg]│  │                   │
   │                     │  │  1: [msg]│  │                   │
   │                     │  │  2: [msg]│  │                   │
   │                     │  └─────────┘  │                   │
   │                     │  Broker 2     │                   │
   │                     │  Broker 3     │                   │
   │                     └──────────────┘
```

### Key Concepts
| Concept | Meaning |
|---------|---------|
| **Topic** | A named stream of messages (e.g., "user_events") |
| **Partition** | A topic is split into partitions for parallelism. Each partition is an ordered log. |
| **Offset** | Position of a message within a partition. Consumers track their offset. |
| **Broker** | A Kafka server. Multiple brokers form a cluster. |
| **Consumer Group** | A group of consumers that share partitions (each partition → one consumer in the group). |
| **Replication** | Each partition has replicas across brokers for fault tolerance. |

### Partitioning
```
Topic: "tweets" (3 partitions)

Producer sends tweet by user_id:
  Partition = hash(user_id) % 3

Partition 0: [tweet from user 1] [tweet from user 4] [tweet from user 7]
Partition 1: [tweet from user 2] [tweet from user 5] [tweet from user 8]
Partition 2: [tweet from user 3] [tweet from user 6] [tweet from user 9]
```

**Why partition?**
- Parallelism: each partition is consumed by a different consumer.
- Ordering: messages within a partition are ordered. Across partitions, no ordering guarantee.
- Scalability: partitions can be on different brokers → horizontal scaling.

### Consumer Groups
```
Topic: "orders" (3 partitions)

Consumer Group "email-service":
  Consumer 1 ← Partition 0
  Consumer 2 ← Partition 1
  Consumer 3 ← Partition 2

Consumer Group "analytics-service":
  Consumer A ← Partition 0
  Consumer B ← Partition 1, 2
```
- Each consumer group independently reads the full topic.
- Within a group, each partition is consumed by exactly one consumer.

### Kafka Guarantees
- **Ordering:** Messages within a partition are ordered (FIFO).
- **At-least-once delivery:** Messages may be delivered more than once (consumer must be idempotent).
- **Durability:** Messages are persisted to disk and replicated. Configurable retention (e.g., 7 days).

---

## 4. Delivery Semantics

| Semantic | Guarantee | Risk |
|----------|-----------|------|
| **At-most-once** | Message delivered 0 or 1 times | May lose messages |
| **At-least-once** | Message delivered 1+ times | May duplicate (consumers must be idempotent) |
| **Exactly-once** | Message delivered exactly 1 time | Hard to achieve; Kafka supports it via transactions |

**In practice:** Most systems use at-least-once + idempotent consumers. If processing a message twice has no side effect (or is detected and deduplicated), at-least-once is sufficient.

### Idempotent Consumer Pattern
```go
func processOrder(msg Message) error {
    // Check if already processed (using message ID)
    if processed := cache.Get(msg.ID); processed {
        return nil  // already processed, skip
    }
    
    // Process the message
    err := handleOrder(msg)
    if err != nil { return err }
    
    // Mark as processed
    cache.Set(msg.ID, true, 24*time.Hour)
    return nil
}
```

---

## 5. Kafka vs RabbitMQ vs SQS

| Feature | Kafka | RabbitMQ | AWS SQS |
|---------|-------|----------|---------|
| Model | Log (persistent, replayable) | Queue (message removed on read) | Queue (message removed on read) |
| Ordering | Per-partition | Per-queue | FIFO queue (separate product) |
| Replay | Yes (consumers can re-read old messages) | No (message consumed = gone) | No |
| Throughput | Millions/sec | ~50K/sec | ~3K/sec (standard) |
| Retention | Configurable (days/forever) | Until consumed | Configurable (max 14 days) |
| Use case | Event streaming, analytics, logs | Task queues, RPC | Simple decoupling, AWS-native |

**Interview heuristic:**
- Need replay, high throughput, event streaming → **Kafka**
- Need simple task queue, complex routing → **RabbitMQ**
- Need simple decoupling on AWS → **SQS**

---

## 6. Event-Driven Architecture

### Pattern: Event Sourcing
Store all state changes as a sequence of events. Current state = replay of all events.

```
Events: [UserCreated] → [AddressChanged] → [EmailVerified] → [PlanUpgraded]
Current state: User {status: verified, plan: premium, address: "new addr"}
```

**Pros:** Full audit trail, time-travel debugging, easy to add new read models.
**Cons:** Complexity, event schema evolution, storage growth.

### Pattern: CQRS (Command Query Responsibility Segregation)
Separate write model (commands) from read model (queries).

```
Write side: API → Command Handler → Event Store → (publish events)
Read side: Event Consumer → Read Model (denormalized) → Query API
```

**Why?** Reads and writes have different needs:
- Writes need consistency, validation, ACID.
- Reads need speed, denormalization, multiple views.

**Example (Twitter):**
- Write: "post tweet" → append to event store → publish to Kafka.
- Read: consumer builds a pre-materialized timeline in Redis → users read from Redis.

---

## Go Connection

```go
// Kafka producer with confluent-kafka-go
import "github.com/confluentinc/confluent-kafka-go/kafka"

producer, _ := kafka.NewProducer(&kafka.ConfigMap{"bootstrap.servers": "localhost:9092"})
producer.Produce(&kafka.Message{
    TopicPartition: kafka.TopicPartition{Topic: &topic, Partition: kafka.PartitionAny},
    Value:          []byte("event data"),
}, nil)

// Kafka consumer
consumer, _ := kafka.NewConsumer(&kafka.ConfigMap{
    "bootstrap.servers": "localhost:9092",
    "group.id":          "email-service",
})
consumer.SubscribeTopics([]string{"user_events"}, nil)
for {
    msg, err := consumer.ReadMessage(100 * time.Millisecond)
    if err == nil {
        processEvent(msg)
        consumer.CommitMessage(msg)  // acknowledge
    }
}

// Go channels = in-process message queue!
// For single-process async processing:
ch := make(chan Event, 1000)  // buffered channel = queue
go func() {
    for event := range ch {
        process(event)
    }
}()
ch <- event  // non-blocking (if buffered)
```

**Go advantage:** Go's channels are a built-in message queue for in-process async. For cross-service, use Kafka/SQS. The mental model transfers directly.

---

## Exercise: Design an Order Processing System

**Requirements:**
1. User places an order (items, payment, shipping address).
2. Payment must be processed (external payment gateway).
3. Inventory must be updated (deduct stock).
4. Confirmation email sent.
5. Shipping label generated.
6. Analytics updated.

**Design:**
1. Which steps are synchronous vs asynchronous?
2. What events/messages would you publish?
3. How do you handle payment failures?
4. How do you ensure exactly-once processing for payments?
5. Draw the architecture.

<details>
<summary>Reference Answer</summary>

**Synchronous (user waits):**
- Create order in DB (pending status)
- Return order ID to user

**Asynchronous (background):**
- Publish `order.created` event to Kafka
- Consumers:
  - Payment service: processes payment → publishes `payment.succeeded` or `payment.failed`
  - On `payment.succeeded`:
    - Inventory service: deducts stock → publishes `inventory.updated`
    - Email service: sends confirmation → publishes `email.sent`
    - Shipping service: generates label → publishes `shipping.created`
    - Analytics service: updates metrics
  - On `payment.failed`:
    - Email service: sends failure notification
    - Order service: marks order as cancelled

**Payment failures:** The order remains "pending" until payment succeeds or fails. If payment fails, the order is cancelled, and inventory is NOT deducted. A timeout mechanism (e.g., 15 minutes) cancels orders that are stuck in pending.

**Exactly-once for payments:** Use idempotency keys. The payment service checks if this order ID has already been processed before calling the payment gateway. Store the idempotency key in the DB.

```
Architecture:

User → API → DB (create order, pending)
          → Kafka: order.created
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
       Payment    Inventory    Email
       Service    Service      Service
       (consumer)  (consumer)  (consumer)
            │
            ▼
      Kafka: payment.succeeded
                        │
            ┌───────────┼───────────┐
            ▼           ▼           ▼
       Inventory    Email       Shipping
       (deduct)     (confirm)   (label)
```
</details>

---

## Common Mistakes
- Using a message queue for everything — some operations MUST be synchronous (payment authorization).
- Not handling duplicate messages — consumers must be idempotent.
- Not discussing delivery semantics — know at-least-once vs exactly-once.
- Overcomplicating: if a Go channel suffices (in-process), don't introduce Kafka.
- Not mentioning dead-letter queues (DLQ) for messages that can't be processed.

## Checklist Before Moving On
- [ ] Know when to use synchronous vs asynchronous processing
- [ ] Can explain Kafka topics, partitions, consumer groups, offsets
- [ ] Understand at-least-once delivery and idempotent consumers
- [ ] Know the difference between Kafka, RabbitMQ, and SQS
- [ ] Understand event-driven architecture and CQRS basics
- [ ] Solved the Order Processing System exercise
