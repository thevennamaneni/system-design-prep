# Day 5 - CAP Theorem, PACELC & Consistency Models

## Topic
**CAP theorem** (the most famous distributed systems concept), **PACELC** (its successor), and **consistency models** (strong, eventual, causal). These govern every database and distributed system decision.

---

## Why It Matters
- "CAP theorem" is the most frequently asked theoretical question in system design interviews.
- Every database choice involves a CAP trade-off. You must articulate: "This database is CP because..."
- Understanding consistency models is essential for discussing replication, caching, and distributed transactions.

---

## 1. CAP Theorem

**Definition:** In a distributed system, you can guarantee at most **two of three** properties:

| Property | Meaning |
|----------|---------|
| **Consistency** | Every read sees the latest write (all nodes agree) |
| **Availability** | Every request gets a response (not an error/timeout) |
| **Partition Tolerance** | System continues to operate despite network partitions (dropped/delayed messages between nodes) |

### The CAP Triangle
```
                    Consistency
                       /\
                      /  \
                     / CP \
                    /      \
                   /________\
                  |          |
                  |    CA    |
                  |          |
                  |__________|
                 /            \
                /              \
               /      AP        \
              /                  \
             /____________________\
        Availability          Partition Tolerance
```

### The Key Insight: P is Not Optional
In a distributed system, **network partitions WILL happen** (cables get cut, switches fail, packets drop). You cannot choose CA — you MUST handle partitions. So the real choice is:

- **CP (Consistency + Partition Tolerance):** When partitioned, refuse requests that can't be consistent. Return an error rather than stale data.
- **AP (Availability + Partition Tolerance):** When partitioned, keep serving requests with potentially stale data. Fix consistency later.

### CP Example: A Bank Transfer
```
Node A (New York) ←── network partition ──→ Node B (London)

User in NY tries to transfer money:
  - CP system: "I can't verify your balance because Node B is unreachable. Error."
  - AP system: "I'll process the transfer and sync later." (RISK: overdraft!)
```
Banks MUST be CP. Social media CAN be AP.

### Database CAP Classifications

| Database | CAP | Why |
|----------|-----|-----|
| PostgreSQL (single node) | CA | No partition tolerance (single node) |
| MongoDB | CP | Prioritizes consistency; primary down → read-only |
| Cassandra | AP | Prioritizes availability; accepts stale reads |
| DynamoDB | AP | Tunable consistency, defaults to eventual |
| Redis (single node) | CA | Single node; Redis Cluster is AP |
| HBase | CP | Strong consistency; if region server down, unavailable until recovered |
| Spanner | CP | Strong consistency across regions (Google's magic) |

> **Note:** CAP classifications are simplified. In reality, systems are tunable. But interviewers expect you to know these.

---

## 2. PACELC Theorem (CAP's Successor)

**CAP only considers the partitioned state.** PACELC also considers the **else** (no partition) state:

```
If Partition (P): choose between Availability (A) and Consistency (C)
Else (E): choose between Latency (L) and Consistency (C)

PACELC: PA/EL, PC/EC, PA/EC, PC/EL
```

### Examples
| System | Partitioned (PA/PC) | Normal (EL/EC) | Full |
|--------|---------------------|----------------|------|
| Cassandra | AP (available) | EL (low latency) | PA/EL |
| DynamoDB | AP | EL | PA/EL |
| MongoDB | CP | EC | PC/EC |
| Spanner | CP | EC | PC/EC |
| PNUTS (Yahoo) | PC | EL | PC/EL |

**Why PACELC matters:** Even without partitions, you trade latency for consistency. Strong consistency requires coordination (2-phase commit, consensus) which adds latency. Eventual consistency can be fast.

---

## 3. Consistency Models

### Strong Consistency
After a write completes, all subsequent reads see the new value — everywhere, immediately.

```
Write: x = 5 (to Node A)
Read: x (from Node B) → must return 5
```

**How it works:** Writes are replicated synchronously to all nodes. Reads consult all (or a quorum of) nodes.

**Cost:** Higher latency (must wait for replication/consensus).
**Use cases:** Banking, inventory, bookings — anywhere stale data causes harm.

### Eventual Consistency
After a write, reads may see stale data temporarily. Eventually (milliseconds to seconds), all nodes converge.

```
Write: x = 5 (to Node A)
Read: x (from Node B) → might return old value (3) for a few seconds
Eventually: Node B receives replication → x = 5
```

**How it works:** Writes are replicated asynchronously. Nodes accept reads without checking others.

**Cost:** Temporary inconsistency.
**Use cases:** Social media (like counts, timelines), DNS propagation, search indexes.

### Causal Consistency
Preserves causality: if operation A causally precedes B, then all nodes see A before B. But concurrent operations can be seen in any order.

```
Alice posts a comment → Bob replies to it (causal: Bob's reply depends on Alice's comment)
  → All nodes will see Alice's comment before Bob's reply.
  → But two unrelated posts can appear in different order on different nodes.
```

**Use cases:** Comment threads, collaborative editing.

### Read-Your-Writes Consistency
After a user writes, their subsequent reads always see that write — even if other nodes haven't received it yet.

```
Alice updates her profile picture → Alice refreshes → must see new picture
  (even if other users don't see it yet)
```

**How to achieve:**
- **Sticky sessions:** Route user to the same node that received their write.
- **Read-after-write from master:** If user wrote < X seconds ago, read from master (not replica).
- **Version tracking:** Client sends last-seen version; server ensures reads are at least that version.

### Monotonic Read Consistency
If a user reads a value, they won't later see an older value (no time travel).

```
Alice reads: post has 10 likes
Alice refreshes: must NOT see 8 likes (going backward)
```

**How to achieve:** Route user to the same replica (sticky) or track timestamps.

---

## 4. Quorum-Based Consistency

In a replicated system with N nodes, you can tune consistency:

```
W = number of nodes that must acknowledge a write
R = number of nodes that must respond to a read

If W + R > N: strong consistency (overlap guarantees latest read)
If W + R ≤ N: eventual consistency (possible stale reads)
```

### Example: 3 Replicas
```
N = 3, W = 2, R = 2 → strong consistency (2+2 > 3)
  Write: acknowledged by 2 nodes. Read: consulted 2 nodes. At least 1 has the latest.

N = 3, W = 1, R = 1 → eventual consistency (1+1 ≤ 3)
  Write: acknowledged by 1 node (fast). Read: consult 1 node (might be stale).

N = 3, W = 3, R = 1 → strong consistency, fast reads
  Write: all 3 nodes (slow writes). Read: any 1 node (fast reads, always latest).
```

**DynamoDB and Cassandra expose W and R as tunable parameters.**

---

## 5. Two-Phase Commit (2PC)

For distributed transactions across multiple databases:

```
Phase 1 (Prepare):
  Coordinator → "Are you ready to commit?" → All participants
  Participants → "Yes" or "No" → Coordinator

Phase 2 (Commit or Abort):
  If all "Yes": Coordinator → "Commit" → All participants
  If any "No": Coordinator → "Abort" → All participants
```

**Problems:**
- **Blocking:** If the coordinator dies, participants are stuck.
- **Slow:** Two round-trips + waiting for slowest participant.
- **Not partition-tolerant:** If a participant is unreachable, the transaction blocks.

**Use 2PC only when strong consistency across services is required.** Prefer eventual consistency + sagas for most cases.

---

## Go Connection

```go
// Go's context package is essential for consistency-related timeouts:
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

// Quorum read with timeout:
func quorumRead(ctx context.Context, nodes []Node, key string) (string, error) {
    type result struct { val string; err error }
    ch := make(chan result, len(nodes))
    for _, node := range nodes {
        go func(n Node) {
            val, err := n.Get(ctx, key)
            ch <- result{val, err}
        }(node)
    }
    // Wait for R responses...
}
```

---

## Exercise: CAP Decisions

For each scenario, choose CP or AP and justify:

1. A stock trading platform — users buy/sell stocks.
2. A social media "like" button.
3. A ride-sharing app showing nearby drivers.
4. A hotel booking system — reserve a room.
5. A DNS server.
6. A comment thread on Reddit.

<details>
<summary>Answers</summary>

1. **CP** — money can't be inconsistent. A double-spend bug is catastrophic.
2. **AP** — a slightly stale like count is fine. Availability matters more.
3. **AP** — a driver 5 seconds ago is still approximately nearby. Better to show something than nothing.
4. **CP** — can't double-book a room. Must be consistent.
5. **AP** — DNS is the canonical AP system (eventual consistency, high availability).
6. **AP** — comments can appear slightly out of order temporarily. Causal consistency is nice-to-have.
</details>

---

## Common Mistakes
- Saying "we'll use CAP" — CAP is a theorem about trade-offs, not a tool.
- Choosing CP for everything — kills availability; most systems can tolerate eventual consistency.
- Choosing AP for everything — dangerous for financial/inventory data.
- Not knowing the CAP classification of common databases.
- Forgetting that "P is not optional" — the real choice is CP vs AP.

## Checklist Before Moving On
- [ ] Can state CAP theorem precisely
- [ ] Know which databases are CP vs AP
- [ ] Understand PACELC and the latency-consistency trade-off
- [ ] Can explain strong vs eventual vs causal consistency
- [ ] Understand quorum-based consistency (W + R > N)
- [ ] Solved the CAP decisions exercise
