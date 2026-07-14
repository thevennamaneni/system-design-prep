# Day 15 - Distributed Consensus, Leader Election & Coordination

## Topic
**Distributed consensus** (Raft, Paxos), **leader election**, and **coordination services** (ZooKeeper, etcd). These are the algorithms that keep distributed systems consistent.

---

## Why It Matters
- Every distributed database (Cassandra, Kafka, etcd) uses consensus internally.
- Leader election is essential for primary-replica databases and distributed locks.
- Interviewers test: "How do you ensure only one node is the leader?" → consensus.

---

## 1. The Consensus Problem

**Definition:** Multiple nodes must **agree on a single value** (or a sequence of values) despite failures and network issues.

### Why It's Hard
- Nodes can crash and restart.
- Network can drop, delay, or reorder messages.
- No global clock — can't just use timestamps.

### The FLP impossibility result
In an **asynchronous** network (no timing guarantees), if even **one node can fail**, no deterministic consensus algorithm can guarantee both safety and liveness. In practice, we use randomized timeouts to work around this.

---

## 2. Raft (The Understandable Consensus Algorithm)

Raft is the most commonly discussed consensus algorithm in interviews (simpler than Paxos). It's used by **etcd** (which powers Kubernetes).

### Raft Key Concepts
| Concept | Meaning |
|---------|---------|
| **Term** | A logical time period with at most one leader |
| **Leader** | One node handles all writes; replicates to followers |
| **Follower** | Passive; responds to leader's requests |
| **Candidate** | A follower that's campaigning to be leader |

### Leader Election
```
1. All nodes start as followers.
2. If a follower doesn't hear from the leader within a random timeout (150-300ms)...
3. ...it becomes a candidate, increments the term, and requests votes.
4. It sends RequestVote RPCs to all nodes.
5. If it gets a majority (N/2 + 1), it becomes the leader.
6. The leader sends heartbeats (AppendEntries RPCs) to maintain authority.
```

```
Follower ──(timeout)──→ Candidate ──(majority votes)──→ Leader
    ↑                                                    │
    └──────────── heartbeat timeout (no heartbeat) ───────┘
```

### Log Replication
```
1. Client sends a write to the leader.
2. Leader appends the entry to its log (uncommitted).
3. Leader sends AppendEntries RPC to all followers.
4. Once a MAJORITY of followers acknowledge...
5. ...the leader commits the entry (marks it as durable).
6. Leader responds to the client: "success."
7. Leader notifies followers of the commit (next heartbeat).
```

### Why Majority?
- If 5 nodes, you need 3 acknowledgments.
- If a network partition splits 3|2, only the 3-node side can commit (majority).
- The 2-node side can't commit → stays consistent.
- This is the **quorum** concept from Day 5.

### Split Brain Scenario
```
5 nodes: A B C D E
Network partition: {A, B, C} | {D, E}

Side {A, B, C} (majority of 3):
  - A was leader, continues as leader.
  - Writes are committed (3/5 = majority).

Side {D, E} (minority of 2):
  - D doesn't get heartbeats → becomes candidate.
  - D can only get 2 votes (itself + E) → not a majority.
  - D cannot commit any writes.
  - D reverts to follower when partition heals.
```

---

## 3. Paxos (The Classic)

Paxos came before Raft and is more general but notoriously hard to understand. Key ideas:

- **Proposers** propose values.
- **Acceptors** vote to accept/reject.
- **Learners** learn the chosen value.
- Two phases: **Prepare** (promise not to accept older proposals) + **Accept** (propose the value).

**In interviews:** Mention Paxos exists and is the theoretical foundation, but focus on Raft (simpler, more practical). "Raft is designed for understandability; Paxos is the original but harder to implement."

---

## 4. Leader Election Patterns

### Bully Algorithm
The node with the **highest ID** becomes the leader:
```
1. Node X detects leader is down.
2. X sends election message to all higher-ID nodes.
3. If no higher node responds, X becomes leader.
4. If a higher node responds, it takes over the election.
```

### Ring Algorithm
Nodes form a logical ring; election message circulates:
```
1. Node X sends election message to its neighbor with its ID.
2. Each node adds its ID to the message and forwards.
3. When the message returns to X, the highest ID in the list is the leader.
```

### Lease-Based Election
Leader holds a "lease" (time-limited):
```
1. Leader gets a lease for T seconds.
2. Before T expires, leader renews the lease.
3. If leader crashes, lease expires → new election.
```
Used by: GFS (Google File System), Chubby (Google's lock service).

---

## 5. Coordination Services

### ZooKeeper
A hierarchical key-value store for coordination:
```
/app
  /leader        → "node-3"  (who is the leader)
  /members
    /node-1      → "10.0.1.1:8080"
    /node-2      → "10.0.1.2:8080"
    /node-3      → "10.0.1.3:8080"
  /config
    /db_url      → "postgres://..."
```

**Features:**
- **Ephemeral nodes:** A node that disappears when the creating session ends (detects crashes).
- **Watchers:** Clients can "watch" a znode for changes (get notified).
- **Sequential nodes:** Auto-incremented names (useful for queues and locks).

**Used by:** Kafka (metadata), HBase (master election), Solr (leader election).

### etcd
A key-value store using Raft (powers Kubernetes):
```
/registry/services/default/my-app  → service definition
/registry/pods/default/pod-123     → pod info
```

**Features:**
- Strong consistency (Raft).
- TTL on keys.
- Watch API (react to changes).
- **Written in Go!**

**Used by:** Kubernetes (all cluster state), CoreDNS, Rook.

### Use Cases for Coordination Services
| Use Case | How |
|----------|-----|
| Leader election | All nodes create an ephemeral node; the one that succeeds is the leader |
| Service discovery | Services register their addresses; clients watch for changes |
| Distributed locks | Create a znode; if it exists, wait (see Day 16) |
| Configuration management | Store config; services watch for updates |
| Group membership | Ephemeral nodes track live members; disappear on crash |

---

## 6. Split-Brain Prevention

**Split-brain:** Two nodes both think they're the leader.

### Prevention: Quorum
```
3 nodes: need 2 to agree (majority)
5 nodes: need 3 to agree
7 nodes: need 4 to agree

Always use an ODD number of nodes.
With 4 nodes, a 2|2 split has no majority → both sides block.
With 5 nodes, a 3|2 split: the 3-node side wins (majority).
```

### Fencing Tokens
When a leader changes, the new leader gets a higher "fencing token":
```
Leader 1 (token=1) → sends write with token=1
Leader 1 crashes → Leader 2 elected (token=2)
Leader 1 recovers, sends write with token=1
Storage rejects: "token 1 < current token 2" → write blocked
```
Prevents a "zombie" leader from corrupting data after it's been deposed.

---

## Go Connection

```go
// etcd client in Go
import "go.etcd.io/etcd/client/v3"

cli, _ := clientv3.New(clientv3.Config{
    Endpoints: []string{"localhost:2379"},
})

// Leader election using etcd
import "go.etcd.io/etcd/client/v3/concurrency"

session, _ := concurrency.NewSession(cli)
election := concurrency.NewElection(session, "/leader-election/")

// Campaign to become leader
err := election.Campaign(context.Background(), "my-node-id")
if err == nil {
    // I am the leader!
    defer election.Resign(context.Background())  // give up leadership
    // ... do leader work ...
}

// Watch for leadership changes
ctx, cancel := context.WithCancel(context.Background())
defer cancel()
ch := election.Observe(ctx)
for resp := range ch {
    fmt.Println("New leader:", string(resp.Kvs[0].Value))
}
```

**Go advantage:** etcd is written in Go. Kubernetes is written in Go. You can speak to the coordination layer natively.

---

## Exercise: Design a Distributed Job Scheduler

**Requirements:**
1. Multiple scheduler nodes (for HA).
2. Only one scheduler should dispatch each job (no duplicates).
3. If the active scheduler crashes, another takes over.
4. Workers pull jobs from a shared queue.

**Design:**
1. How do you elect a leader among schedulers?
2. How do you detect a scheduler crash?
3. How do you prevent split-brain?
4. What coordination service would you use?

<details>
<summary>Reference Answer</summary>

1. **Leader election via etcd:** All schedulers attempt to create an ephemeral node `/job-scheduler/leader`. The first one succeeds → leader. Others watch the node; if it disappears (leader crashed), they try again.

2. **Crash detection:** Ephemeral nodes disappear when the etcd session (heartbeat) expires. If the leader crashes, its ephemeral node vanishes within a few seconds. Watchers are notified → new election.

3. **Split-brain prevention:** Use etcd's Raft consensus (requires majority). With 3 etcd nodes, a network partition can't produce two leaders (minority side can't achieve quorum).

4. **Coordination service:** etcd (or ZooKeeper). Use ephemeral nodes for leader election, watches for failover, and a distributed lock for job assignment.

```
Architecture:
  Schedulers: [S1, S2, S3]
  etcd: [E1, E2, E3] (Raft cluster)
  Workers: [W1, W2, ..., W100]

  1. S1, S2, S3 all try to create /scheduler/leader (ephemeral)
  2. S1 succeeds → S1 is leader
  3. S2, S3 watch /scheduler/leader
  4. S1 dispatches jobs to Kafka topic "jobs"
  5. Workers consume from Kafka
  6. If S1 crashes → ephemeral node disappears → S2/S3 race → S2 becomes leader
```
</details>

---

## Common Mistakes
- Not understanding why majority/quorum prevents split-brain.
- Forgetting that coordination services (etcd, ZooKeeper) are themselves distributed and need their own consensus.
- Using a single etcd/ZooKeeper node → SPOF. Always run 3 or 5.
- Confusing leader election with distributed locks (related but different — Day 16).

## Checklist Before Moving On
- [ ] Can explain Raft leader election (terms, candidates, votes, majority)
- [ ] Understand log replication (leader appends → majority commits)
- [ ] Know how split-brain is prevented (quorum, odd nodes, fencing tokens)
- [ ] Know what etcd and ZooKeeper are used for
- [ ] Can design a leader election system using etcd
- [ ] Solved the Distributed Job Scheduler exercise
