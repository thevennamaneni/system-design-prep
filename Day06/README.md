# Day 6 - Storage Types & CDNs

## Topic
**Storage types** (block, object, file), **CDNs** (Content Delivery Networks), and how to choose storage for different data types (images, videos, logs, databases).

---

## Why It Matters
- Every system design stores data somewhere. Choosing the wrong storage type = poor performance or high cost.
- CDNs are the answer to "How do you serve content globally with low latency?" — asked in almost every media-heavy design.
- At senior level, you're expected to discuss storage trade-offs (cost, durability, latency).

---

## 1. Storage Types

### a) Block Storage
**What it is:** Raw blocks of storage presented to the OS as a disk. The OS formats it with a filesystem.

```
[Server] → [Block Storage Volume] → formatted as ext4/zfs → files
```

**Examples:** AWS EBS (Elastic Block Store), GCP Persistent Disk, physical SSD/HDD.

**Characteristics:**
- **Low latency:** Direct disk access, no network overhead (usually).
- **Block-level:** Read/write at the block level (512 bytes - 4KB).
- **Attached to one server:** Usually can't be shared (except with cluster filesystems).
- **Use cases:** Database storage (PostgreSQL, MySQL), OS boot drives, swap.

**When to use:** When you need a filesystem and low-latency disk I/O — primarily for databases.

### b) Object Storage
**What it is:** Data stored as "objects" (file + metadata + ID) in a flat namespace. Accessed via HTTP API.

```
PUT https://s3.amazonaws.com/my-bucket/photo.jpg → stores object
GET https://s3.amazonaws.com/my-bucket/photo.jpg → retrieves object
```

**Examples:** AWS S3, Google Cloud Storage, Azure Blob Storage, MinIO (self-hosted).

**Characteristics:**
- **HTTP API:** RESTful interface (PUT, GET, DELETE).
- **Flat namespace:** No real directories (keys simulate hierarchy).
- **Massive scale:** Practically unlimited storage.
- **High durability:** S3 offers 11 nines (99.999999999%).
- **Not for databases:** No random access, no modifications (must rewrite whole object).
- **Cheap:** Much cheaper than block storage per GB.

**When to use:** Images, videos, backups, logs, static assets, data archives.

### c) File Storage
**What it is:** A shared filesystem accessible by multiple servers via NFS/SMB protocols.

```
[Server1] ─┐
[Server2] ─┼→ [NFS/SMB Server] → /shared/data/
[Server3] ─┘
```

**Examples:** AWS EFS (Elastic File System), NFS, SMB/CIFS.

**Characteristics:**
- **Shared:** Multiple servers can read/write simultaneously.
- **File-level:** Operates on files and directories.
- **Higher latency** than block storage (network overhead).
- **Use cases:** Shared configs, CMS media uploads, home directories.

**When to use:** When multiple servers need access to the same files.

### Summary Table

| Type | Latency | Scale | Cost | Shared | Use Case |
|------|---------|-------|------|--------|----------|
| Block | Lowest | Limited (per volume) | High | No | Databases, OS disks |
| Object | Higher (HTTP) | Unlimited | Low | Yes (via API) | Media, backups, logs |
| File | Medium | Medium | Medium | Yes (NFS) | Shared files |

---

## 2. CDN (Content Delivery Network)

**What it is:** A geographically distributed network of servers that cache content close to users, reducing latency.

```
Without CDN:
  User in Tokyo → Origin Server in Virginia → 150ms latency

With CDN:
  User in Tokyo → Tokyo CDN Edge (cache hit) → 5ms latency
```

### How a CDN Works
```
1. User requests: https://cdn.example.com/image.jpg
2. CDN edge node (Tokyo) checks cache:
   - Cache HIT → return image immediately (fast)
   - Cache MISS → fetch from origin (Virginia), cache it, return
3. Next user in Tokyo requests same image → cache HIT (fast)
```

### CDN Architecture
```
                    ┌─────────────────┐
                    │  Origin Server   │
                    │  (Virginia, USA) │
                    └────────┬────────┘
                             │
              ┌──────────────┼──────────────┐
              │              │              │
         ┌────┴────┐   ┌────┴────┐   ┌────┴────┐
         │CDN Edge │   │CDN Edge │   │CDN Edge │
         │ Tokyo   │   │ London  │   │ Sydney  │
         └────┬────┘   └────┬────┘   └────┬────┘
              │              │              │
         Users in        Users in       Users in
         Asia            Europe         Oceania
```

### What to Cache on a CDN
| Content | Cache? | TTL |
|---------|--------|-----|
| Static images, CSS, JS | Yes | Long (days/months) |
| Videos (HLS/DASH segments) | Yes | Long |
| API responses (public data) | Maybe | Short (seconds/minutes) |
| User-specific data | No | — |
| Dynamic HTML | No (usually) | — |

### Cache Invalidation
How to update content before TTL expires:
- **Cache busting:** Append version hash to URL (`app.js?v=a1b2c3`). New version = new URL = new cache entry.
- **Purge API:** CDN provides an API to invalidate specific URLs.
- **Surrogate keys:** Tag related objects; purge by key.

### CDN Providers
- **Cloudflare:** Most popular, generous free tier.
- **AWS CloudFront:** Integrates with S3 seamlessly.
- **Akamai:** Enterprise, massive global footprint.
- **Fastly:** Real-time purging, popular for streaming.

### CDN in System Design Interviews
When designing media-heavy systems (Netflix, YouTube, Instagram), always propose a CDN:
```
Users → CDN (cache videos/images) → Origin (if cache miss) → S3 (storage)
```

**Key talking points:**
- "I'd use a CDN to cache static assets close to users, reducing latency from 150ms to <20ms."
- "For cache invalidation, I'd use content-versioned URLs (cache busting) — new versions get new URLs."
- "For video streaming, I'd use HLS with segments cached at CDN edges."

---

## 3. Data Lifecycle: Hot, Warm, Cold

Not all data is accessed equally. Tier your storage:

```
Hot Data (frequently accessed):
  → Redis cache, SSD block storage
  → Examples: recent tweets, active sessions, live dashboards

Warm Data (occasionally accessed):
  → PostgreSQL, MongoDB on standard storage
  → Examples: 30-day-old posts, user profiles

Cold Data (rarely accessed):
  → S3 (object storage), Glacier
  → Examples: 1-year-old logs, archived tweets, old backups
```

### Storage Cost vs Performance
```
Redis (cache)     ████████████████████ $$$$$  (fastest, most expensive)
SSD Block Storage ████████████████    $$$$   
Standard HDD      ████████████        $$$
S3 (object)       ████████            $$
Glacier (archive) ██                  $      (cheapest, slowest retrieval)
```

---

## Go Connection

```go
// S3 upload from Go (using AWS SDK)
import "github.com/aws/aws-sdk-go-v2/service/s3"

uploader := s3manager.NewUploader(sess)
result, err := uploader.Upload(&s3manager.UploadInput{
    Bucket: aws.String("my-bucket"),
    Key:    aws.String("images/" + filename),
    Body:   file,
})

// Serving through CDN:
// URL: https://cdn.example.com/images/photo.jpg
// → CloudFront fetches from S3 if not cached

// File storage (NFS) in Go:
// Just use standard os/file operations — the mount is transparent
data, err := os.ReadFile("/shared/config/app.yaml")
```

---

## Exercise: Design Storage for Instagram

For each data type, choose the storage type and explain:

1. User profile photos (uploaded once, viewed millions of times)
2. User feed (real-time, personalized)
3. Posts (photo + caption + metadata)
4. Stories (expire after 24 hours)
5. Direct messages
6. Search index (find posts by hashtag)
7. Old posts (> 1 year, rarely viewed)

<details>
<summary>Reference Answer</summary>

| Data | Storage | Why |
|------|---------|-----|
| Profile photos | S3 + CDN | Object storage for images, CDN for global delivery. Immutable — cache forever with versioned URLs. |
| User feed | Redis (cache) | Personalized, frequently updated, low latency needed. Rebuild from DB on cache miss. |
| Posts metadata | PostgreSQL | Structured: user_id, caption, timestamps, relationships to comments/likes. |
| Post images/videos | S3 + CDN | Large blobs, object storage + CDN for delivery. |
| Stories | Redis (with TTL) | Expire after 24h — Redis TTL is perfect. High write rate. |
| Direct messages | Cassandra or PostgreSQL | If high volume: Cassandra (write-optimized). If ACID needed: PostgreSQL. |
| Search index | Elasticsearch | Full-text search, hashtag queries, ranking. |
| Old posts | S3 (Glacier for images) + Cassandra (metadata archive) | Cold storage — rarely accessed, optimize for cost. |

**Key insight:** One system, 5+ storage technologies. This is normal at scale.
</details>

---

## Common Mistakes
- Storing images in a database (BLOB) — use object storage + CDN.
- Using block storage for shared files — use file storage or object storage.
- Not using a CDN for global content — 150ms latency vs 20ms is huge for UX.
- Forgetting to discuss data lifecycle (hot/warm/cold) — shows senior thinking.
- Not knowing S3 durability (11 nines) vs availability (different metric!).

## Checklist Before Moving On
- [ ] Know the 3 storage types and when to use each
- [ ] Can explain how a CDN works (edge nodes, cache hit/miss, origin)
- [ ] Know cache invalidation strategies (cache busting, purge API)
- [ ] Understand hot/warm/cold data tiering
- [ ] Solved the Instagram storage exercise
