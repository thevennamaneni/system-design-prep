# Day 4 - Databases: SQL, NoSQL, ACID & BASE

## Topic
**Database types** — relational (SQL) vs non-relational (NoSQL), the **ACID** vs **BASE** consistency models, and when to use each. This is one of the most important decisions in any system design.

---

## Why It Matters
- Every system design interview requires a database choice. Choosing wrong = a system that can't scale.
- The SQL vs NoSQL decision is the #1 database question in interviews. You must justify your choice.
- Understanding ACID vs BASE is essential for discussing consistency trade-offs (Day 5).

---

## 1. Relational Databases (SQL)

**What it is:** Data stored in tables (rows + columns) with predefined schema and relationships. Uses SQL for queries.

**Examples:** PostgreSQL, MySQL, Oracle, SQL Server.

### Characteristics
- **Structured schema:** Tables with types, constraints, foreign keys.
- **ACID transactions:** Atomic, Consistent, Isolated, Durable.
- **Relational:** JOINs across tables, foreign key constraints.
- **Vertical scaling:** Primarily scales by upgrading hardware (sharding is possible but complex — Day 13).

### When to Use SQL
| Use Case | Why |
|----------|-----|
| User accounts, financial transactions | ACID guarantees, consistency |
| Complex queries (JOINs, aggregations) | SQL is designed for this |
| Data with clear relationships | Foreign keys enforce integrity |
| Small-to-medium scale (millions, not billions of rows) | Single instance handles this well |
| Transactional systems (e-commerce, banking) | ACID is non-negotiable |

### Example Schema (Twitter)
```sql
-- Users table
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Tweets table
CREATE TABLE tweets (
    id BIGSERIAL PRIMARY KEY,
    user_id BIGINT REFERENCES users(id),
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Follows table (many-to-many)
CREATE TABLE follows (
    follower_id BIGINT REFERENCES users(id),
    followee_id BIGINT REFERENCES users(id),
    created_at TIMESTAMP DEFAULT NOW(),
    PRIMARY KEY (follower_id, followee_id)
);
```

### SQL Scaling Challenges
- **Write scaling:** Single master handles all writes → write bottleneck.
- **Read scaling:** Add read replicas, but replication lag causes stale reads.
- **Storage scaling:** Single disk has limits → partitioning/sharding needed.
- **Schema changes:** `ALTER TABLE` on billion-row tables is expensive.

---

## 2. NoSQL Databases

**What it is:** Non-relational databases designed for specific use cases: high write throughput, flexible schema, horizontal scaling.

### Four Types of NoSQL

#### a) Key-Value Stores
**Examples:** Redis, DynamoDB, Memcached

```
Key: "user:123" → Value: {"name":"Alice", "age":30}
Key: "session:abc" → Value: "eyJhbGciOi..."
```

- **Fastest** NoSQL — O(1) lookups.
- **Use cases:** Caching, session storage, leaderboards, real-time data.
- **Limitations:** No complex queries, no JOINs, no range queries (mostly).

#### b) Document Stores
**Examples:** MongoDB, CouchDB, Firestore

```json
{
  "_id": "tweet_123",
  "user_id": "user_456",
  "content": "Hello world!",
  "likes": 42,
  "comments": [
    {"user": "Bob", "text": "Nice!"}
  ],
  "created_at": "2024-01-15T10:30:00Z"
}
```

- **Flexible schema:** Each document can have different fields.
- **Use cases:** Content management, catalogs, user profiles, CMS.
- **Limitations:** No ACID across documents (in some implementations), no JOINs.

#### c) Wide-Column Stores
**Examples:** Cassandra, HBase, DynamoDB (also key-value)

```
Row Key: "user:123"
  Column Family: "profile"
    name: "Alice"
    age: 30
  Column Family: "tweets"
    tweet_1: "Hello"
    tweet_2: "World"
```

- **Designed for massive write throughput** and horizontal scaling.
- **Use cases:** Time-series data, event logging, IoT data, timelines.
- **Query model:** Primary key lookups and range scans. No JOINs.
- **Cassandra example:** Twitter/Digg use Cassandra for timeline feeds.

#### d) Graph Databases
**Examples:** Neo4j, Amazon Neptune

```
(Alice) -[FOLLOWS]→ (Bob) -[FOLLOWS]→ (Charlie)
```

- **Designed for relationship-heavy data** (social networks, recommendation engines).
- **Use cases:** Social graphs, fraud detection, recommendation engines.
- **Limitations:** Not for general-purpose storage; complex to scale.

---

## 3. SQL vs NoSQL Decision Matrix

| Criterion | SQL | NoSQL |
|-----------|-----|-------|
| Data structure | Structured, tabular | Flexible, document/key-value |
| Schema | Fixed (migrations needed) | Flexible (add fields anytime) |
| Queries | Complex (JOINs, aggregations) | Simple (key-based lookups) |
| Scaling | Vertical (hard to shard) | Horizontal (built-in sharding) |
| Consistency | Strong (ACID) | Eventual (BASE) — usually |
| Transactions | Multi-row, multi-table | Limited (document-level usually) |
| Best for | Financial, transactional | High-volume, read/write-heavy |

### Interview Heuristic
```
Is the data relational (users ↔ orders ↔ items)?
  → SQL (PostgreSQL/MySQL)

Is the data a simple key-value lookup?
  → Key-Value (Redis/DynamoDB)

Is the data a flexible document (varying fields)?
  → Document (MongoDB)

Is it high-write, time-series, or timeline data?
  → Wide-Column (Cassandra)

Is it relationship-heavy (social graph)?
  → Graph (Neo4j) or SQL with adjacency lists
```

### The Real-World Answer: Polyglot Persistence
Large systems use **multiple databases** for different needs:
```
Twitter:
  - PostgreSQL: users, follows (relational, ACID)
  - Cassandra: timelines (high write throughput)
  - Redis: cache (hot timelines, sessions)
  - S3: media (images, videos)
```

---

## 4. ACID vs BASE

### ACID (Relational Databases)
| Property | Meaning |
|----------|---------|
| **Atomicity** | All operations in a transaction succeed or none do (all-or-nothing) |
| **Consistency** | Data moves from one valid state to another (constraints enforced) |
| **Isolation** | Concurrent transactions don't interfere (they appear sequential) |
| **Durability** | Committed data survives crashes (written to disk) |

**Example (bank transfer):**
```
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- deduct from Alice
  UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- add to Bob
COMMIT;
```
If the system crashes after the first UPDATE, the transaction is rolled back — Alice doesn't lose money without Bob receiving it.

### BASE (NoSQL Databases)
| Property | Meaning |
|----------|---------|
| **Basically Available** | System remains available during failures (partial degradation OK) |
| **Soft State** | Data may be temporarily inconsistent (replication lag) |
| **Eventually Consistent** | Data becomes consistent over time (not immediately) |

**Example (social media like count):**
- User likes a post.
- The like is written to one node.
- Other nodes eventually receive the update (replication lag: milliseconds to seconds).
- During that window, different users may see different like counts — acceptable for social media.

### When ACID vs BASE Matters
| Requirement | Choose |
|------------|--------|
| Money, inventory, bookings | ACID (SQL) |
| Social feeds, analytics, search | BASE (NoSQL) |
| Mixed | Polyglot persistence |

---

## 5. Database Indexes

**What it is:** A data structure (usually B-tree) that speeds up queries on specific columns.

```sql
-- Without index: full table scan (O(n))
SELECT * FROM tweets WHERE user_id = 123;

-- With index: B-tree lookup (O(log n))
CREATE INDEX idx_tweets_user_id ON tweets(user_id);
```

### Types of Indexes
- **B-Tree:** Default. Good for equality and range queries.
- **Hash Index:** O(1) for exact match only. No range queries.
- **Composite Index:** Multiple columns. `(user_id, created_at)` speeds up "get user's tweets sorted by time."
- **Full-text Index:** For text search (e.g., PostgreSQL tsvector, Elasticsearch).

### Trade-offs
- **Faster reads** (the whole point).
- **Slower writes** (index must be updated on every insert/update/delete).
- **More storage** (indexes take space).
- **Rule of thumb:** Index columns used in WHERE, JOIN, ORDER BY clauses.

---

## Go Connection

```go
// Go's database/sql package works with any SQL database via drivers
import (
    "database/sql"
    _ "github.com/lib/pq"  // PostgreSQL driver
)

db, err := sql.Open("postgres", "host=localhost dbname=mydb")
db.QueryRow("SELECT username FROM users WHERE id = $1", userID)

// For Redis (key-value):
import "github.com/redis/go-redis/v9"
rdb := redis.NewClient(&redis.Options{Addr: "localhost:6379"})
rdb.Set(ctx, "user:123", "Alice", 0)

// For MongoDB (document):
import "go.mongodb.org/mongo-driver/mongo"
client, _ := mongo.Connect(ctx, options.Client().ApplyURI("mongodb://localhost:27017"))

// For Cassandra (wide-column):
import "github.com/gocql/gocql"
cluster := gocql.NewCluster("localhost:9042")
```

---

## Exercise: Choose Databases for a Ride-Sharing App (Uber)

For each data type, choose a database and justify:

1. User profiles (riders and drivers)
2. Real-time driver locations (GPS coordinates, updated every 5 seconds)
3. Ride history and receipts
4. Payment transactions
5. Driver-rider matching (find nearby drivers)
6. Push notifications / messages

<details>
<summary>Reference Answer</summary>

| Data Type | Database | Justification |
|-----------|----------|---------------|
| User profiles | PostgreSQL | Structured, relational (user → payment methods → rides), ACID |
| Real-time locations | Redis (GeoIndex) | Extremely high write rate (millions of updates/sec), TTL for expiration, geospatial queries |
| Ride history | PostgreSQL (recent) + Cassandra (archive) | Recent rides need ACID and JOINs; archived rides need high-volume storage |
| Payments | PostgreSQL | ACID is non-negotiable for money; transactional integrity |
| Driver matching | Redis (GeoIndex) or a geospatial DB | "Find drivers within 3km" = geospatial query, needs to be <100ms |
| Notifications | Kafka (event queue) + Redis (delivery state) | Async delivery, high throughput, decouple from ride logic |

**Key insight:** No single database handles all needs. Polyglot persistence is the answer.
</details>

---

## Common Mistakes
- Using NoSQL for financial data (no ACID = potential money loss).
- Using SQL for high-write, time-series data (write bottleneck).
- Not creating indexes on frequently queried columns — slow queries at scale.
- Creating too many indexes — slow writes.
- Forgetting that NoSQL doesn't mean "no schema" — it means "flexible schema."
- Saying "MongoDB" without knowing when it's better than PostgreSQL.

## Checklist Before Moving On
- [ ] Know the 4 types of NoSQL and when to use each
- [ ] Can explain ACID vs BASE and when each matters
- [ ] Know the SQL vs NoSQL decision criteria
- [ ] Understand indexes (B-tree, composite) and their trade-offs
- [ ] Solved the Uber database selection exercise
