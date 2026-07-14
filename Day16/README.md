# Day 16 - Distributed Locks, Transactions & Data Consistency

## Topic
**Distributed locks**, **distributed transactions** (2PC, Saga, Outbox pattern), and **data consistency patterns** across services. These solve "how do you coordinate state across multiple machines/services?"

---

## Why It Matters
- Microservices each have their own database → no ACID transactions across services.
- Distributed locks are essential for "only one instance does X" scenarios.
- Interviewers test: "How do you prevent double-charging if a retry happens?" → idempotency + distributed transactions.

---

## 1. Distributed Locks

**What it is:** A lock that works across multiple machines, ensuring only one process can access a resource at a time, even across a network.

### Use Cases
- Only one worker processes a job.
- Prevent double-updates to a shared resource.
- Leader election (lock = "I am the leader").
- Prevent concurrent updates to the same database row across services.

### Implementation with Redis (Redlock)

```go
// Simple Redis lock: SETNX with expiry
func acquireLock(key string, ttl time.Duration) bool {
    // SET key value NX EX ttl
    ok, _ := redis.SetNX(ctx, "lock:"+key, myNodeID, ttl).Result()
    return ok
}

func releaseLock(key string) {
    // Only release if I own the lock (check value == myNodeID)
    // Use a Lua script for atomic check-and-delete
    script := `
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
    `
    redis.Eval(ctx, script, []string{"lock:" + key}, myNodeID)
}
```

**Critical:** The release must check ownership (value == myNodeID). Otherwise:
```
1. Node A acquires lock (TTL 10s).
2. Node A is slow, takes 15s.
3. Lock expires at 10s → Node B acquires it.
4. Node A finishes, deletes the lock → DELETES NODE B'S LOCK!
5. Node C acquires → now B and C both think they have the lock. BAD.
```

### Redlock Algorithm (Multi-Node)
For higher reliability, acquire locks on multiple Redis nodes:
```
1. Acquire lock on N (e.g., 5) independent Redis nodes.
2. If acquired on majority (3+), the lock is held.
3. If not, release all and retry.
```
**Pros:** Survives single Redis node failure.
**Cons:** Controversial (Martin Kleppmann argues it's not safe under certain failure modes). In practice, single-Redis locks with TTL are sufficient for most use cases.

### Implementation with etcd/ZooKeeper
More correct than Redis (strong consistency via Raft):
```
1. Create an ephemeral sequential node: /locks/my-resource/node-0001
2. Check: am I the lowest-numbered node?
   → Yes: I have the lock.
   → No: watch the node immediately before me; when it disappears, I'm next.
```

### Lock Fairness
- **FIFO:** etcd sequential nodes ensure fairness (first to request = first to acquire).
- **Unfair:** Redis SETNX is non-deterministic (whoever's fastest wins).

### Lock Pitfalls
1. **No TTL:** If the lock holder crashes, the lock is held forever. Always set a TTL.
2. **TTL too short:** If work takes longer than TTL, another node acquires the lock → race condition.
3. **Not checking ownership on release:** Delete someone else's lock (shown above).
4. **Long-held locks:** If you need to hold a lock for minutes, use a **lock renewal** thread (extend TTL periodically).

---

## 2. Distributed Transactions

### The Problem
```
Service A (PostgreSQL): deduct $100 from Alice's account
Service B (PostgreSQL): add $100 to Bob's account

If A succeeds but B fails → Alice lost $100, Bob didn't receive it. MONEY LOST.
```

In a monolith with one DB, this is a simple `BEGIN TRANSACTION ... COMMIT`. In microservices with separate DBs, it's a distributed transaction.

### a) Two-Phase Commit (2PC)
```
Coordinator: "Everyone ready to commit?"
  → Service A: "Ready" (lock the row)
  → Service B: "Ready" (lock the row)

Coordinator: "Commit!"
  → Service A: commits
  → Service B: commits

If any service says "not ready": Coordinator says "Abort!" to all.
```

**Pros:** Strong consistency (ACID across services).
**Cons:**
- **Blocking:** If coordinator crashes, participants are stuck holding locks.
- **Slow:** Two round-trips + waiting for slowest participant.
- **Not partition-tolerant:** If a participant is unreachable, the transaction blocks.

**When to use:** Rarely in microservices. Use for same-datacenter, low-latency, strong-consistency needs (e.g., financial systems with tight coupling).

### b) Saga Pattern
Break the distributed transaction into a sequence of local transactions, each with a **compensating action** (undo).

```
Transfer $100 from Alice to Bob:

Step 1: Deduct $100 from Alice (Service A)
  → If success: proceed to Step 2
  → If fail: abort (nothing to undo)

Step 2: Add $100 to Bob (Service B)
  → If success: done!
  → If fail: COMPENSATE → undo Step 1 (add $100 back to Alice)
```

### Saga Types

**Choreography (Event-Driven):**
```
Service A deducts → publishes "MoneyDeducted" →
Service B receives → adds → publishes "MoneyAdded" → done
If B fails → publishes "AddFailed" →
Service A receives → compensates (refund)
```
No central coordinator; services react to events. Decentralized but harder to track.

**Orchestration (Central Coordinator):**
```
Orchestrator → "Deduct from A"
  ← A: "Success"
Orchestrator → "Add to B"
  ← B: "Failure"
Orchestrator → "Refund A" (compensate)
```
Central orchestrator manages the flow. Easier to track but adds a SPOF.

### Saga Trade-offs
| Aspect | 2PC | Saga |
|--------|-----|------|
| Consistency | Strong (ACID) | Eventual |
| Performance | Slow (blocking) | Fast (no locks) |
| Failure handling | Automatic rollback | Manual compensation logic |
| Complexity | Protocol-level | Application-level |
| Use case | Tight coupling, financial | Microservices, async |

### c) Outbox Pattern
Ensure a database write and an event publication happen atomically:

```
Problem:
  1. Write to DB (order created)
  2. Publish event to Kafka (order.created)
  If step 2 fails → DB has the order, but no event → downstream services never know.

Solution (Outbox):
  1. Write to DB: order + outbox event (in the SAME transaction)
  2. A separate process (CDC) reads the outbox table and publishes to Kafka
  3. If publish fails, retry (the event is safely in the DB)
```

```
┌──────────────────────────┐
│       Service A DB        │
│  ┌─────────┐ ┌─────────┐ │
│  │ orders  │ │ outbox  │ │  (same transaction)
│  └─────────┘ └────┬────┘ │
└───────────────────┼──────┘
                    │
            ┌───────┴───────┐
            │  CDC (Debezium)│  (reads outbox, publishes to Kafka)
            └───────┬───────┘
                    │
                    ▼
              ┌──────────┐
              │  Kafka   │
              └──────────┘
```

**Tools:** Debezium (CDC), Kafka Connect, or a simple background worker that polls the outbox table.

---

## 3. Idempotency in Distributed Systems

Every operation that can be retried MUST be idempotent.

### Idempotency Patterns

**a) Idempotency Key (for API requests):**
```
POST /payments
  Idempotency-Key: abc-123
  Body: { "amount": 100, "recipient": "Bob" }

Server: check if "abc-123" already processed.
  → Yes: return previous result (don't charge again)
  → No: process, store key + result, return
```

**b) Unique Constraint (for database writes):**
```sql
-- Prevent duplicate order creation
INSERT INTO orders (id, user_id, amount) VALUES (uuid(), 123, 100);
-- Use a unique constraint on (user_id, cart_id) to prevent duplicate orders
```

**c) Version/Optimistic Concurrency:**
```
UPDATE products SET stock = stock - 1, version = version + 1
WHERE id = 123 AND version = 5;
-- If 0 rows affected → someone else modified it first → retry
```

---

## 4. Eventual Consistency Patterns

### Read Repair
When a read detects stale data on a replica, the reader updates it:
```
Client reads from Replica B → sees old data
Client also reads from Replica A → sees new data
Client writes new data to Replica B (repair)
```

### Anti-Entropy (Background Sync)
A background process continuously compares replicas and syncs missing data:
```
Merkle tree hash comparison between nodes → sync differences
```
Used by: Cassandra (repair), DynamoDB (sync).

### Hinted Handoff
If a node is temporarily down, other nodes store "hints" for it:
```
Node C is down → Node A receives a write intended for C
Node A stores a "hint" (the write + "this is for C")
Node C recovers → Node A forwards the hint
```

---

## Go Connection

```go
// Distributed lock with Redis (go-redis)
import "github.com/redis/go-redis/v9"

func AcquireLock(ctx context.Context, rdb *redis.Client, key, value string, ttl time.Duration) bool {
    ok, err := rdb.SetNX(ctx, key, value, ttl).Result()
    return err == nil && ok
}

// Release with ownership check (Lua script for atomicity)
var releaseScript = redis.NewScript(`
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
`)

func ReleaseLock(ctx context.Context, rdb *redis.Client, key, value string) error {
    _, err := releaseScript.Run(ctx, rdb, []string{key}, value).Result()
    return err
}

// Saga orchestration in Go
type SagaStep struct {
    Execute   func() error
    Compensate func() error
}

func RunSaga(steps []SagaStep) error {
    for i, step := range steps {
        if err := step.Execute(); err != nil {
            // Compensate all previous steps in reverse order
            for j := i - 1; j >= 0; j-- {
                steps[j].Compensate()
            }
            return err
        }
    }
    return nil
}

// Outbox pattern: write order + event in one transaction
func CreateOrderWithOutbox(tx *sql.Tx, order *Order) error {
    // Insert order
    _, err := tx.Exec("INSERT INTO orders (...) VALUES (...)", ...)
    if err != nil { return err }
    
    // Insert outbox event (same transaction)
    event := fmt.Sprintf(`{"type":"order.created","order_id":"%s"}`, order.ID)
    _, err = tx.Exec("INSERT INTO outbox (event_type, payload) VALUES (?, ?)", "order.created", event)
    return err
}
```

---

## Exercise: Design a Money Transfer System

**Requirements:**
1. Transfer money between accounts in different banks (different databases).
2. Must not lose money or double-charge.
3. Handle network failures and retries.
4. Handle the case where the transfer partially succeeds.

**Design:**
1. Would you use 2PC or Saga? Why?
2. Draw the flow (including compensation).
3. How do you handle idempotency?
4. How do you use the Outbox pattern?
5. What happens if the compensation itself fails?

<details>
<summary>Reference Answer</summary>

1. **Saga** — different banks = different organizations, can't do 2PC (too slow, too coupled). Saga with compensation is the right choice.

2. Flow:
```
Transfer $100 from Bank A (Alice) to Bank B (Bob):

Step 1: Bank A deducts $100 from Alice
  → Write to DB + outbox event "transfer.deducted" (same transaction)
  → Publish event to Kafka

Step 2: Bank B adds $100 to Bob
  → Consume "transfer.deducted" event
  → Add $100 to Bob's account
  → Write to DB + outbox event "transfer.completed" (same transaction)
  → Publish event

If Step 2 fails (e.g., Bob's account doesn't exist):
  → Publish "transfer.failed" event
  → Bank A consumes "transfer.failed"
  → COMPENSATE: add $100 back to Alice
  → Publish "transfer.compensated"
```

3. **Idempotency:** Each transfer has a unique `transfer_id`. Both banks check if they've already processed this ID before applying the change. Store in a `processed_transfers` table.

4. **Outbox:** Each bank writes the DB change + event to its outbox in the same transaction. A CDC process reads the outbox and publishes to Kafka. This ensures the event is never lost even if Kafka is down.

5. **Compensation failure:** If compensation fails (rare), the system must alert human operators. The transfer is stuck in a "needs manual intervention" state. Use a dead-letter queue for failed compensations and a monitoring alert.
</details>

---

## Common Mistakes
- Using 2PC for microservices — too slow, too coupled. Use Saga.
- Not setting TTL on distributed locks → lock held forever if process crashes.
- Releasing a lock without checking ownership → deleting someone else's lock.
- Not handling compensation failure — must have a fallback (alert, DLQ).
- Forgetting the Outbox pattern — events lost if DB write succeeds but publish fails.

## Checklist Before Moving On
- [ ] Can implement a Redis distributed lock with TTL and ownership check
- [ ] Understand 2PC vs Saga and when to use each
- [ ] Can design a Saga with compensation
- [ ] Know the Outbox pattern and why it's needed
- [ ] Understand idempotency patterns (key, unique constraint, optimistic concurrency)
- [ ] Solved the Money Transfer System exercise
