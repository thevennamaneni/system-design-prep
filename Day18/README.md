# Day 18 - Data Pipelines, Batch Processing & CDC

## Topic
**Data pipelines** (ETL/ELT), **batch processing** (MapReduce, Spark), **Change Data Capture** (CDC), and **stream processing** (Kafka Streams, Flink). How data moves from operational databases to analytics warehouses.

---

## Why It Matters
- Every large system needs analytics. "How do you power a dashboard showing daily active users?" → data pipeline.
- CDC is the modern way to sync databases to search indexes, caches, and warehouses without dual-writes.
- Interviewers test: "How do you keep Elasticsearch in sync with PostgreSQL?" → CDC, not application-level dual-writes.

---

## 1. The Data Pipeline Problem

```
Operational Systems (OLTP):         Analytics (OLAP):
  PostgreSQL (users)                  ┌──────────────┐
  MongoDB (orders)         ────────→ │ Data Warehouse │ → BI dashboards
  Cassandra (events)                 │ (Snowflake)    │   ML training
  Redis (sessions)                   └──────────────┘   Reports
```

You need to move data from operational databases to an analytics warehouse without impacting production performance.

---

## 2. ETL vs ELT

### ETL (Extract, Transform, Load)
```
Source DB → Extract → Transform (clean, join, aggregate) → Load → Warehouse
```
- Transform happens in a processing layer (Spark, custom code).
- Warehouse receives clean, ready-to-query data.

### ELT (Extract, Load, Transform)
```
Source DB → Extract → Load (raw) → Warehouse → Transform (in warehouse)
```
- Raw data is loaded into the warehouse first.
- Transformation happens inside the warehouse (SQL, dbt).
- Modern approach (Snowflake, BigQuery have cheap compute).

**Trend:** ELT is more popular now (warehouses are powerful enough to transform data).

---

## 3. Batch Processing

### MapReduce (The Classic)
```
Map:    Process each record independently (parallel)
Shuffle: Group by key
Reduce: Aggregate per group

Example: Count words in 1TB of text
  Map:    each doc → (word, 1) pairs
  Shuffle: group by word
  Reduce: sum counts per word → (word, total_count)
```

### Apache Spark (Modern Batch Processing)
- In-memory processing (100x faster than disk-based MapReduce).
- Supports SQL, streaming, ML, graph processing.
- RDDs (Resilient Distributed Datasets) — fault-tolerant data partitions.

### When to Use Batch Processing
- Large datasets (TBs/PBs) that don't need real-time results.
- Periodic reports (daily/weekly analytics).
- ML model training.
- Data warehouse loading.

### Lambda Architecture (Batch + Speed)
```
                    ┌──────────┐
        ┌─────────→│ Batch Layer│ → batch views (comprehensive, slow)
        │           └──────────┘
Data ───┤                        ┌──────────┐
        │           ┌──────────┐→│ Merge    │ → query results
        └─────────→│ Speed Layer│ → real-time views (approximate, fast)
                    └──────────┘
```
- **Batch layer:** processes ALL data periodically (hours/days). Accurate.
- **Speed layer:** processes recent data in real-time. Approximate.
- **Serving layer:** merges batch + speed views for queries.

### Kappa Architecture (Stream-Only)
```
Data → Kafka (all events) → Stream Processing → Views
```
- Everything is a stream. No batch layer.
- Re-process by replaying Kafka (it retains messages).
- Simpler than Lambda (one processing engine, not two).

---

## 4. Change Data Capture (CDC)

**What it is:** Automatically capture database changes (INSERT, UPDATE, DELETE) and propagate them to downstream systems — without application-level dual-writes.

### The Dual-Write Problem
```
Application writes to:
  1. PostgreSQL (source of truth)
  2. Elasticsearch (search index)

Problem: If step 2 fails, Elasticsearch is out of sync with PostgreSQL.
  → Search results are stale/missing.
  → No automatic recovery.
```

### CDC Solution
```
PostgreSQL WAL (Write-Ahead Log) → CDC Tool (Debezium) → Kafka → Consumers
  → Elasticsearch (search index auto-updated)
  → Redis cache (auto-invalidated)
  → Data warehouse (auto-loaded)
```

### How CDC Works
1. The database writes all changes to a **transaction log** (WAL in PostgreSQL, binlog in MySQL).
2. A CDC tool (Debezium) tails the log in real-time.
3. Each change is published as an event to Kafka.
4. Consumers (Elasticsearch indexer, cache invalidator, warehouse loader) consume events and update their systems.

### CDC Benefits
- **No dual-writes:** Application writes only to the DB. CDC handles propagation.
- **Atomicity:** The change is in the log only after the transaction commits. No partial updates.
- **Ordering:** Events are in commit order. Consumers see changes in the correct sequence.
- **Recovery:** If a consumer is down, it catches up by replaying Kafka from its last offset.

### CDC Tools
| Tool | Description |
|------|-------------|
| **Debezium** | Open-source, Kafka Connect-based. Supports PostgreSQL, MySQL, MongoDB, etc. |
| **AWS DMS** | AWS Database Migration Service (also does CDC). |
| **Maxwell** | MySQL binlog reader → JSON → Kafka. |
| **CloudCannon** | Managed CDC. |

### CDC in System Design Interviews
When asked "How do you keep Elasticsearch in sync with your database?" → CDC is the answer:
```
PostgreSQL → Debezium → Kafka → Elasticsearch Connector → Elasticsearch
```
Not: "In my application code, after writing to PostgreSQL, I also write to Elasticsearch." (Dual-write = fragile.)

---

## 5. Stream Processing

### What It Is
Processing data in real-time as it arrives (unlike batch processing which processes stored data).

### Use Cases
- Real-time fraud detection (analyze transactions as they happen).
- Live dashboards (active users, revenue today).
- Real-time alerts (temperature exceeded threshold).
- Anomaly detection (unusual login patterns).

### Kafka Streams / Apache Flink
```
Input Stream → Filter → Map → Window → Aggregate → Output Stream
```

**Windowing:** Group events into time windows for aggregation:
```
Tumbling window: [0-5s] [5-10s] [10-15s]  (non-overlapping, fixed)
Sliding window:  [0-5s] [2-7s] [4-9s]     (overlapping)
Session window:  grouped by activity gaps (user session)
```

### Example: Real-Time Word Count
```
Input: stream of tweets
  → tokenize → (word, 1) → window(5 sec) → count per word → output

Every 5 seconds: output the count of each word seen in that window.
```

### Stream Processing vs Batch Processing

| Aspect | Batch | Stream |
|--------|-------|--------|
| Latency | Minutes to hours | Milliseconds to seconds |
| Data | Stored (all at once) | Flowing (continuous) |
| Complexity | Simpler | More complex (ordering, late data, windows) |
| Use case | Reports, training | Real-time alerts, dashboards |
| Tools | Spark, MapReduce | Flink, Kafka Streams |

---

## 6. Data Warehouse vs Data Lake

### Data Warehouse
- **Structured data** (tables, schemas).
- **Optimized for analytics** (columnar storage, OLAP queries).
- **Examples:** Snowflake, BigQuery, Redshift, Hive.
- **Use case:** BI dashboards, SQL-based analytics.

### Data Lake
- **Raw, unstructured/semi-structured data** (JSON, Parquet, images, logs).
- **Schema-on-read** (define schema when querying, not when storing).
- **Examples:** S3 + Athena, Azure Data Lake, Hadoop HDFS.
- **Use case:** ML training, data science, exploratory analysis.

### Modern: Lakehouse
- Combines data lake (cheap storage, all formats) + warehouse (querying, ACID).
- **Examples:** Databricks Delta Lake, Apache Iceberg, Apache Hudi.
- Best of both worlds.

---

## Go Connection

```go
// Go can be a stream processor (though Flink/Spark are more common)
// Using Kafka consumer + Go channels:

func processStream(ctx context.Context, consumer *kafka.Consumer) {
    for {
        msg, err := consumer.ReadMessage(100 * time.Millisecond)
        if err != nil { continue }
        
        // Parse event
        var event Event
        json.Unmarshal(msg.Value, &event)
        
        // Process (filter, transform, aggregate)
        if shouldProcess(event) {
            transformed := transform(event)
            publishToOutputStream(transformed)
        }
    }
}

// CDC with Debezium: the Go app is a Kafka consumer
// Debezium produces change events like:
// {"before": null, "after": {"id":1,"name":"Alice"}, "op":"c"}  (create)
// {"before": {"id":1,"name":"Alice"}, "after": {"id":1,"name":"Bob"}, "op":"u"}  (update)
// {"before": {"id":1,"name":"Bob"}, "after": null, "op":"d"}  (delete)

func handleCDCEvent(event ChangeEvent) {
    switch event.Op {
    case "c", "u":  // create or update
        es.Index(event.After)  // update Elasticsearch
    case "d":  // delete
        es.Delete(event.Before.ID)  // remove from Elasticsearch
    }
}
```

---

## Exercise: Design an Analytics Dashboard for a Food Delivery App

**Requirements:**
1. Show real-time metrics: orders/min, revenue today, active delivery agents.
2. Show daily/weekly reports: top restaurants, average delivery time, user retention.
3. Data sources: Orders DB (PostgreSQL), Events (Kafka), Delivery tracking (Redis).
4. Dashboard updates every 5 seconds for real-time; daily for reports.

**Design:**
1. How do you power the real-time dashboard?
2. How do you power the daily reports?
3. How do you get data from operational DBs to the analytics warehouse?
4. Would you use Lambda or Kappa architecture?

<details>
<summary>Reference Answer</summary>

1. **Real-time dashboard (5-second updates):**
   - Stream processing: Kafka Streams or Flink consumes the events stream.
   - Aggregates: orders/min (tumbling window), revenue today (running total), active agents (count from Redis).
   - Results written to Redis (for sub-ms dashboard reads).
   - Dashboard frontend polls a `/metrics` API every 5 seconds → reads from Redis.

2. **Daily/weekly reports:**
   - Batch processing: Spark job runs nightly.
   - Reads from the data warehouse (Snowflake/BigQuery).
   - Computes: top restaurants (by order volume), avg delivery time, retention.
   - Results stored in the warehouse → dashboard queries directly.

3. **Data pipeline (operational → warehouse):**
   - **CDC (Debezium):** PostgreSQL → Debezium → Kafka → Warehouse connector.
   - **Direct streaming:** Kafka events → warehouse (already in Kafka).
   - **Redis:** periodic snapshot to warehouse (for delivery agent state).
   - ELT: raw data lands in warehouse, dbt transforms into report tables.

4. **Kappa architecture** (stream-only):
   - All data flows through Kafka (orders via CDC, events directly).
   - Stream processing handles both real-time and "batch" (replay Kafka for historical).
   - Simpler than Lambda (one processing engine).
   - Batch reports = scheduled query over the warehouse (which was populated by streams).
</details>

---

## Common Mistakes
- Dual-writing to DB + Elasticsearch from application code → use CDC.
- Using batch processing for real-time requirements → use stream processing.
- Not considering late/out-of-order events in stream processing.
- Confusing data warehouse (structured, queried) with data lake (raw, stored).
- Not knowing Debezium/CDC — it's the standard answer for DB-to-search sync.

## Checklist Before Moving On
- [ ] Understand ETL vs ELT
- [ ] Can explain CDC and the dual-write problem
- [ ] Know when to use batch vs stream processing
- [ ] Understand Lambda vs Kappa architecture
- [ ] Know the difference between data warehouse, data lake, and lakehouse
- [ ] Solved the Analytics Dashboard exercise
