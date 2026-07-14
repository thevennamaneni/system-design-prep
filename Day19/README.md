# Day 19 - Observability: Logging, Metrics & Distributed Tracing

## Topic
**Observability** — logging, metrics, and distributed tracing. How to monitor, debug, and alert on distributed systems. Essential for senior roles — "how do you know when something is wrong?"

---

## Why It Matters
- Senior engineers are expected to design for operability, not just functionality.
- Interviewers ask: "How would you monitor this system?" or "How do you debug a slow request?"
- Observability is the difference between "we have a system" and "we can operate a system."

---

## 1. The Three Pillars of Observability

```
┌────────────────────────────────────────────────────────────┐
│                    Observability                             │
├──────────────┬──────────────────┬───────────────────────────┤
│   Logging    │     Metrics       │   Distributed Tracing     │
│ (what happened│  (is it healthy?) │  (where is it slow?)      │
│  in detail)   │                   │                           │
├──────────────┼──────────────────┼───────────────────────────┤
│ ELK Stack     │ Prometheus        │ Jaeger                    │
│ Splunk        │ Grafana           │ Zipkin                    │
│ CloudWatch    │ Datadog           │ OpenTelemetry             │
│               │                   │                           │
│ "User 123     │ "CPU is at 80%"   │ "Request took 2s:         │
│  failed login"│ "QPS: 5000"       │  50ms auth → 1800ms DB    │
│               │ "p99 latency: 200ms│  → 50ms cache → done"    │
└──────────────┴──────────────────┴───────────────────────────┘
```

---

## 2. Logging

### What to Log
| Level | When to Use | Example |
|-------|-------------|---------|
| ERROR | Something failed; needs attention | "Payment failed: card declined" |
| WARN | Unexpected but non-fatal | "Retrying after timeout (attempt 2/3)" |
| INFO | Significant application events | "User 123 created order 456" |
| DEBUG | Detailed diagnostic info | "Cache miss for key user:123" |

### Structured Logging (ALWAYS)
Log in JSON (or key-value), not free text:

```json
// BAD (unstructured):
"2024-01-15 User 123 failed to login from IP 10.0.0.1"

// GOOD (structured):
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "WARN",
  "event": "login_failed",
  "user_id": 123,
  "ip": "10.0.0.1",
  "reason": "invalid_password",
  "attempt": 3,
  "request_id": "req_abc123"
}
```

**Why structured?** Searchable, filterable, machine-parseable. ELK/Datadog can index fields.

### Log Aggregation Architecture
```
App Server → Log Agent (Fluentd/Filebeat) → Log Store (Elasticsearch) → Dashboard (Kibana)
```

1. **App** writes structured logs to stdout/file.
2. **Agent** (Fluentd, Filebeat, Promtail) runs on each server, collects logs.
3. **Store** (Elasticsearch, Loki) indexes logs for fast search.
4. **Dashboard** (Kibana, Grafana) for querying and visualization.

### Log Sampling
At high traffic (100K req/sec), logging every request is too much. Sample:
- Log 1% of successful requests (INFO).
- Log 100% of errors (ERROR).
- Use a `request_id` to trace individual requests.

### Request ID Correlation
Every request gets a unique `request_id` (UUID). All logs for that request include it:
```
req_abc123 → [auth] user 123 authenticated
req_abc123 → [cache] miss for user:123
req_abc123 → [db] query took 50ms
req_abc123 → [response] 200 OK in 152ms
```
Search for `request_id=req_abc123` → see the full request journey in logs.

---

## 3. Metrics

### What Metrics to Track (USE Method)
For resources (servers, databases):
- **U**tilization: % of capacity used (CPU, memory, disk, network).
- **S**aturation: how much work is queued (queue length, request backlog).
- **E**rrors: error count/rate (5xx, exceptions, retries).

### What Metrics to Track (RED Method)
For services (APIs):
- **R**ate: requests per second (QPS).
- **E**rrors: error rate (5xx / total).
- **D**uration: latency (p50, p90, p99, p99.9).

### Latency Percentiles (CRITICAL)
```
p50 (median):   50ms  — half of requests are faster than this
p90:            150ms — 90% of requests are faster
p99:            500ms — 99% of requests are faster
p99.9:          2000ms — 99.9% of requests are faster

Average is misleading! A few 10-second outliers hide behind fast p50.
Use percentiles to understand tail latency.
```

### Key Metrics for System Design
| Metric | What It Tells You |
|--------|-------------------|
| QPS (queries per second) | Current load |
| Error rate (5xx/total) | System health |
| p99 latency | User experience (99% of users see this or better) |
| CPU/Memory utilization | Capacity planning |
| Queue depth | Backlog / saturation |
| Cache hit rate | Caching effectiveness |
| Database connections | Connection pool health |
| Disk I/O | Storage bottleneck |

### Metrics Architecture (Prometheus + Grafana)
```
App (exposes /metrics) ← scraped by Prometheus → stored in TSDB → Grafana dashboard
```

**Prometheus** is a pull-based metrics system:
- The app exposes a `/metrics` endpoint (text format).
- Prometheus scrapes it every 15 seconds.
- Prometheus stores metrics in a time-series database.
- **Grafana** queries Prometheus and renders dashboards.

**Metric types in Prometheus:**
```
Counter:    monotonically increasing (total requests, total errors)
Gauge:      goes up and down (queue depth, active connections)
Histogram:  distribution (request latency buckets)
Summary:    pre-computed percentiles (p50, p90, p99)
```

### Alerting
Alerts fire when metrics cross thresholds:
```
ALERT HighErrorRate
  IF error_rate > 5% FOR 5 minutes
  THEN page on-call engineer

ALERT HighLatency
  IF p99_latency > 500ms FOR 10 minutes
  THEN page on-call engineer

ALERT DiskFull
  IF disk_usage > 90%
  THEN page on-call engineer
```

**Alerting principles:**
- Alert on symptoms (high error rate, high latency), not causes (CPU is high — so what?).
- Every alert should be actionable. No "alert fatigue."
- Page for critical issues; email/Slack for non-critical.

---

## 4. Distributed Tracing

### The Problem
In microservices, a single user request may flow through 10 services:
```
API Gateway → Auth Service → User Service → Order Service → Payment Service → ...
```

If the request takes 3 seconds, WHICH service is slow? Logs alone can't tell you.

### Distributed Tracing Solution
A **trace** follows a single request across all services:

```
Trace ID: trace_abc123
├── API Gateway (5ms)
│   ├── Auth Service (20ms)
│   ├── User Service (50ms)
│   │   └── DB Query (45ms) ← slow!
│   ├── Order Service (2800ms) ← SLOWEST
│   │   ├── Kafka Publish (5ms)
│   │   ├── Payment Service (2790ms)
│   │   │   └── Payment Gateway API (2785ms) ← ROOT CAUSE
│   │   └── Inventory Service (5ms)
│   └── Response (2ms)
Total: 2882ms
```

### How Tracing Works
1. **Trace ID:** A unique ID for the entire request (generated at the entry point).
2. **Span:** A single operation within a trace (one service call). Has start time, duration, and parent.
3. **Span context:** Trace ID + Span ID passed via HTTP headers (or gRPC metadata) between services.

```
API Gateway generates: trace_id=abc123, span_id=001
  → passes to Auth Service via header: X-Trace-Id: abc123, X-Span-Id: 001
Auth Service creates span: trace_id=abc123, span_id=002, parent=001
  → passes to User Service: X-Trace-Id: abc123, X-Span-Id: 002
...
```

### OpenTelemetry (The Standard)
OpenTelemetry is the CNCF standard for traces, metrics, and logs:
- One SDK for all languages (Go, Java, Python, etc.).
- Exports to any backend (Jaeger, Zipkin, Datadog, Honeycomb).

```go
// OpenTelemetry in Go
import "go.opentelemetry.io/otel"

func GetUser(ctx context.Context, id int) (*User, error) {
    ctx, span := tracer.Start(ctx, "GetUser")
    defer span.End()
    
    span.SetAttributes(attribute.Int("user.id", id))
    
    user, err := db.QueryUser(ctx, id)
    if err != nil {
        span.RecordError(err)
        return nil, err
    }
    
    span.SetAttributes(attribute.String("user.name", user.Name))
    return user, nil
}
```

### Tracing Architecture
```
App (instrumented with OpenTelemetry)
  → OTLP exporter
  → Jaeger (trace storage)
  → Jaeger UI (visualize trace timeline)
```

---

## 5. Health Checks & Readiness Probes

### Liveness Probe
"Is the app running?"
```
GET /healthz → 200 OK (process is alive)
```
If liveness fails → Kubernetes restarts the container.

### Readiness Probe
"Is the app ready to serve traffic?"
```
GET /readyz → 200 OK (DB connected, cache warm, ready to serve)
           → 503 (still starting up, or dependency is down)
```
If readiness fails → Kubernetes removes the pod from the load balancer (no traffic).

### Deep Health Check
```go
func healthHandler(w http.ResponseWriter, r *http.Request) {
    checks := map[string]string{}
    healthy := true
    
    // Check DB
    if err := db.Ping(); err != nil {
        checks["database"] = "DOWN: " + err.Error()
        healthy = false
    } else {
        checks["database"] = "OK"
    }
    
    // Check Redis
    if _, err := redis.Ping(ctx).Result(); err != nil {
        checks["redis"] = "DOWN: " + err.Error()
        healthy = false
    } else {
        checks["redis"] = "OK"
    }
    
    if healthy {
        writeJSON(w, 200, checks)
    } else {
        writeJSON(w, 503, checks)
    }
}
```

---

## Go Connection

```go
// Structured logging (using slog, Go 1.21+)
import "log/slog"

slog.Info("user created",
    "user_id", 123,
    "email", "alice@example.com",
    "request_id", requestID,
)

// Prometheus metrics
import "github.com/prometheus/client_golang/prometheus"

var (
    requestCount = prometheus.NewCounterVec(
        prometheus.CounterOpts{Name: "http_requests_total"},
        []string{"method", "path", "status"},
    )
    requestDuration = prometheus.NewHistogramVec(
        prometheus.HistogramOpts{Name: "http_request_duration_seconds"},
        []string{"method", "path"},
    )
)

// Middleware that tracks metrics
func metricsMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        start := time.Now()
        
        wrapped := &statusWriter{ResponseWriter: w, status: 200}
        next.ServeHTTP(wrapped, r)
        
        duration := time.Since(start).Seconds()
        requestCount.WithLabelValues(r.Method, r.URL.Path, strconv.Itoa(wrapped.status)).Inc()
        requestDuration.WithLabelValues(r.Method, r.URL.Path).Observe(duration)
    })
}

// Expose metrics endpoint
http.Handle("/metrics", promhttp.Handler())
```

**Go advantage:** Go has built-in `log/slog` (structured logging), excellent Prometheus client library, and first-class OpenTelemetry support. Prometheus itself is written in Go.

---

## Exercise: Design Observability for a Microservices System

**Requirements:**
1. 10 microservices, each with 20 instances (200 total instances).
2. Need to detect: high error rate, high latency, service outages.
3. Need to debug: "User X's request took 5 seconds — why?"
4. Need to alert: page on-call engineer if error rate > 1% for 5 minutes.

**Design:**
1. What logging setup? How do you correlate logs across services?
2. What metrics do you track? What thresholds for alerting?
3. How do you implement distributed tracing?
4. How do you handle health checks?

<details>
<summary>Reference Answer</summary>

1. **Logging:**
   - Each service emits structured JSON logs (using `log/slog`).
   - Fluentd agent on each instance collects logs → Elasticsearch.
   - Every request gets a `request_id` (UUID) at the API gateway, passed via header to all services.
   - All log entries include `request_id`, `trace_id`, `service_name`.
   - Kibana dashboard for searching logs by `request_id` or `trace_id`.

2. **Metrics (Prometheus + Grafana):**
   - Each service exposes `/metrics` (RED method):
     - `http_requests_total{service, method, path, status}`
     - `http_request_duration_seconds{service, method, path}` (histogram)
   - Plus infrastructure metrics: CPU, memory, disk, network (node_exporter).
   - Alerts:
     - Error rate > 1% for 5 min → page on-call.
     - p99 latency > 500ms for 10 min → page on-call.
     - Service down (no metrics for 2 min) → page on-call.
     - Disk > 90% → Slack notification.

3. **Distributed Tracing (OpenTelemetry + Jaeger):**
   - API Gateway generates a `trace_id` for each request.
   - Each service creates a span with OpenTelemetry SDK.
   - Trace context passed via HTTP headers (`traceparent`).
   - Spans exported to Jaeger → visualize the full request timeline.
   - To debug "User X's 5-second request": search Jaeger by `user_id` or `request_id` → see which span is slowest.

4. **Health Checks:**
   - `/healthz` (liveness): returns 200 if the process is alive.
   - `/readyz` (readiness): checks DB connection, Redis connection, downstream service availability.
   - Kubernetes probes: liveness every 10s, readiness every 5s.
   - If readiness fails → pod removed from LB → no traffic → auto-recovery or restart.
</details>

---

## Common Mistakes
- Logging unstructured text — not searchable.
- Not including request_id/trace_id in logs — can't correlate across services.
- Using average latency instead of percentiles — hides tail latency.
- Alerting on causes (CPU high) instead of symptoms (error rate high).
- Not having readiness vs liveness distinction (liveness checks dependencies → cascading restarts).

## Checklist Before Moving On
- [ ] Know the 3 pillars: logging, metrics, tracing
- [ ] Can design structured logging with request_id correlation
- [ ] Know RED method (Rate, Errors, Duration) and USE method (Utilization, Saturation, Errors)
- [ ] Understand latency percentiles (p50, p90, p99) and why average is misleading
- [ ] Can explain distributed tracing (trace_id, spans, parent-child)
- [ ] Know liveness vs readiness probes
- [ ] Solved the Observability design exercise
