# Day 25 - Full System Design: Video Streaming (Netflix/YouTube)

## Topic
**Design a video streaming platform** (Netflix/YouTube). Covers CDN strategy, adaptive bitrate streaming, video processing pipelines, and storage at exabyte scale.

---

## Why This Problem Matters
- Video is the highest-bandwidth use case on the internet. It tests CDN knowledge deeply.
- It introduces **adaptive bitrate streaming** (HLS/DASH) — a concept unique to video.
- It combines batch processing (video transcoding) with real-time delivery (CDN).

---

## RESHADED Framework

### R — Requirements (5 min)

**Functional:**
1. Users upload videos (YouTube) or Netflix uploads content (producer model).
2. Users stream videos on web, mobile, TV.
3. Adaptive quality: auto-adjust resolution based on bandwidth.
4. Search and browse videos.
5. Recommendations (stretch).
6. View count, likes, comments.

**Non-functional:**
1. Low startup time: < 2 seconds from click to playback.
2. Minimal buffering: < 1% of playback time.
3. Global delivery: users worldwide.
4. High availability: 99.99%.
5. 1B users, 500 hours of video uploaded per minute (YouTube-scale).

**Clarifying questions:**
- "User-generated (YouTube) or curated (Netflix)?" → Cover both; focus on delivery.
- "Live streaming or VOD?" → Focus on VOD (Video on Demand). Mention live as extension.
- "DRM?" → Mention it; not core to architecture.

### E — Estimation (3 min)

```
Users: 1B MAU, 500M DAU
Video uploads: 500 hours/minute = 30,000 hours/hour
Average video size (after compression): 100MB per 10-min video
Upload rate: 500 hours × 6 (10-min videos per hour) × 100MB = 300GB/minute

Storage:
  Raw uploads: 300GB/min × 1440 min/day = 432TB/day
  With transcoding (5 resolutions): ×5 = 2.16PB/day
  5 years: ~3.9EB (exabytes!)

Bandwidth (streaming):
  500M users × 10 min/day watch time × 5Mbps = 500M × 10×60 × 5/8 bytes
  = ~1.9TB/sec (egress) — massive!
  CDN handles 95%+ of this.
```

### S — Storage Schema (5 min)

```
S3 (object storage): video files
  - Original upload: s3://videos/original/{video_id}.mp4
  - Transcoded segments: s3://videos/{video_id}/360p/seg_001.ts, 720p/seg_001.ts, ...
  - Thumbnails: s3://thumbnails/{video_id}.jpg

PostgreSQL: metadata
  videos (id, title, description, uploader_id, duration, views, created_at)
  users (id, name, subscriptions, ...)

Cassandra: view history, comments (high write volume)
  views (user_id, video_id, timestamp, watch_duration)
  comments (video_id, comment_id, user_id, text, timestamp)

Elasticsearch: video search index
  index: videos (title, description, tags, transcript)

CDN: cached video segments at edge nodes worldwide
```

### H — High-Level Design (10 min)

```
=== UPLOAD FLOW ===

┌────────┐     ┌───────────┐     ┌──────────┐
│Uploader│────→│ Upload API│────→│   S3     │ (original)
└────────┘     └─────┬─────┘     └──────────┘
                     │
                     ▼
               ┌──────────┐
               │  Kafka   │ (video.uploaded event)
               └────┬─────┘
                    │
                    ▼
               ┌──────────────┐
               │ Transcoding   │ (encode to multiple resolutions)
               │ Workers       │ (split into HLS segments)
               └──────┬───────┘
                      │
                      ▼
               ┌──────────┐
               │   S3     │ (segmented video: 360p, 480p, 720p, 1080p)
               └──────────┘
                      │
                      ▼
               ┌──────────┐
               │   CDN    │ (pre-fetch popular videos to edge nodes)
               └──────────┘


=== STREAMING FLOW ===

┌────────┐     ┌───────────┐     ┌──────────┐
│ Viewer │────→│ Streaming │────→│  CDN     │ (cache hit: return segment)
│        │     │ API       │     │  Edge    │
└────────┘     └───────────┘     └────┬─────┘
                                     │ miss
                                     ▼
                                ┌──────────┐
                                │   S3     │ (origin: fetch segment)
                                └──────────┘
```

### A — API Design (5 min)

```
=== Upload ===
POST /api/v1/videos/init-upload
  Body: { "title": "My Video", "description": "..." }
  Response: { "video_id": "v123", "upload_url": "https://s3.presigned.url/..." }

PUT {upload_url}  (direct upload to S3 via presigned URL)
  Body: <video binary data>
  Response: 200 OK

POST /api/v1/videos/{video_id}/complete-upload
  → Triggers transcoding pipeline

=== Streaming (HLS) ===
GET /api/v1/videos/{video_id}/master.m3u8
  → Returns master playlist (lists available resolutions)
  Response:
    #EXTM3U
    #EXT-X-STREAM-INF:BANDWIDTH=800000,RESOLUTION=640x360
    360p/playlist.m3u8
    #EXT-X-STREAM-INF:BANDWIDTH=2800000,RESOLUTION=1280x720
    720p/playlist.m3u8

GET /api/v1/videos/{video_id}/720p/playlist.m3u8
  → Returns segment list for 720p

GET /api/v1/videos/{video_id}/720p/seg_001.ts
  → Returns video segment (served from CDN)

=== Metadata ===
GET /api/v1/videos/{video_id}  → video details
GET /api/v1/videos/{video_id}/comments?cursor=abc  → comments
POST /api/v1/videos/{video_id}/views  → record view
```

### D — Detailed Design (15 min)

#### Adaptive Bitrate Streaming (HLS)

**HLS (HTTP Live Streaming):** Video is split into small segments (2-10 seconds), each available at multiple bitrates.

```
Video: "My Video" (10 minutes)

Transcoded into:
  360p  (800 Kbps)  → seg_001.ts, seg_002.ts, ..., seg_060.ts  (6-sec segments)
  480p  (1400 Kbps) → seg_001.ts, seg_002.ts, ..., seg_060.ts
  720p  (2800 Kbps) → seg_001.ts, seg_002.ts, ..., seg_060.ts
  1080p (5000 Kbps) → seg_001.ts, seg_002.ts, ..., seg_060.ts

Master playlist (master.m3u8):
  Lists all available resolutions + bandwidth.

Client (video player):
  1. Fetches master.m3u8.
  2. Chooses initial resolution based on detected bandwidth.
  3. Fetches the resolution's playlist.m3u8.
  4. Fetches segments sequentially.
  5. Monitors bandwidth: if high → switch to higher resolution.
                   if low → switch to lower resolution.
  6. Segments are independent → resolution can change between segments.
```

**Why HLS?**
- Segments are cached on CDN (small, cacheable files).
- Adaptive: quality changes based on user's bandwidth (no buffering).
- Standard HTTP (no special server needed — just a CDN).
- Client-side switching (no server round-trip to change quality).

#### Transcoding Pipeline
```
1. Video uploaded to S3 (original).
2. Kafka event: "video.uploaded" published.
3. Transcoding workers (consumers):
   a. Download original from S3.
   b. Use FFmpeg to transcode into 4 resolutions.
   c. Split each resolution into 6-second HLS segments.
   d. Upload segments to S3.
   e. Generate master.m3u8 and per-resolution playlists.
   f. Publish "video.ready" event.
4. Video becomes available for streaming.
```

**Scaling transcoding:**
- CPU-intensive → use spot instances / auto-scaling.
- Each video = one Kafka message → one worker handles it.
- Large videos: split into chunks, transcode in parallel, merge.

#### CDN Strategy (CRITICAL)
```
Without CDN:
  User in Tokyo → S3 in Virginia → 150ms latency per segment → buffering.

With CDN:
  User in Tokyo → Tokyo CDN edge (cache hit) → 5ms → smooth playback.
```

**CDN caching:**
- Popular videos: pre-cached at all edge nodes (proactive push).
- Long-tail videos: cached on first request (lazy load).
- Segment-level caching: each segment is independently cacheable.
- Cache TTL: long (videos are immutable; new versions = new video_id).

**CDN origin shield:**
```
CDN Edge (Tokyo) → Origin Shield (regional) → S3 (origin)
  → Origin shield reduces S3 load (multiple edges share one shield).
```

#### Recommendations (Stretch)
```
Collaborative filtering: "Users who watched X also watched Y."
Content-based: "Videos with similar tags/descriptions."
Implementation: pre-computed recommendations in Redis (updated nightly via batch job).
```

### E — Edge Cases (5 min)
- **Video still transcoding:** Return "processing" status; notify when ready.
- **Corrupt upload:** Validate video format after upload; reject if invalid.
- **Copyright strike:** Remove video from CDN cache (purge) + S3.
- **Low bandwidth:** HLS automatically drops to lower resolution.
- **Mobile data vs Wi-Fi:** Default to lower resolution on mobile; let user override.
- **Live streaming:** Different protocol (RTMP ingest → HLS segments in real-time). Mention as extension.

### D — Discussion (5 min)
- **Bottleneck:** Bandwidth (egress). CDN handles 95%+ → S3 only serves cache misses.
- **Cost:** Egress bandwidth is the biggest cost. CDN + multi-region S3.
- **Alternative to HLS:** DASH (similar, different standard). MPEG-DASH is more flexible; HLS is more widely supported (Apple).
- **DRM:** Encrypt segments; decrypt on client with a license key. Widevine (Google), FairPlay (Apple).
- **Monitoring:** Buffering rate (% of playback time spent buffering), startup time, CDN hit rate, transcoding queue depth.

---

## Exercise: Design Live Streaming (Twitch)

Modify the design for **live streaming** (not VOD):
1. Streamer broadcasts live; viewers watch with < 5 second delay.
2. No pre-transcoding — must transcode in real-time.
3. 100K concurrent viewers for a popular stream.

**Think about:**
- Ingest protocol (RTMP).
- Real-time transcoding (vs batch).
- CDN for live segments (very short TTL).
- Chat alongside video (WebSocket).

<details>
<summary>Approach</summary>

1. **Ingest:** Streamer uses OBS/XSplit to send RTMP stream to an ingest server.
   `rtmp://ingest.twitch.tv/app/{stream_key}`

2. **Real-time transcoding:**
   - Ingest server receives RTMP → transcodes to multiple resolutions in real-time (FFmpeg).
   - Splits into HLS segments (2-second segments for low latency).
   - Uploads segments to S3/CDN as they're generated.

3. **CDN for live:**
   - Very short segment TTL (2 seconds).
   - CDN edges pull segments as they're created.
   - Viewers' players poll for new segments every 2 seconds.

4. **Low latency:**
   - Use short segments (1-2 seconds, not 6-10 like VOD).
   - Use LL-HLS (Low Latency HLS) or WebRTC for < 2 second latency.
   - Trade-off: shorter segments = more overhead (more HTTP requests).

5. **Chat:**
   - WebSocket connection for each viewer.
   - Messages routed via Redis Pub/Sub (Day 24 design).
   - Chat is independent of video (different infrastructure).

6. **Scaling:**
   - Popular stream (100K viewers): CDN absorbs 99% of delivery.
   - Transcoding: one instance per stream (real-time, can't parallelize easily).
   - Chat: 100K WebSocket connections → 2-3 WS servers.
</details>

---

## Common Mistakes
- Not splitting video into segments (trying to stream one big file → no adaptive bitrate, no CDN caching).
- Forgetting transcoding pipeline (uploading raw video → too large for most bandwidths).
- Not using a CDN (S3 directly → high latency, high egress cost).
- Not explaining adaptive bitrate (the key innovation of HLS).
- Mixing up HLS (Apple) and RTMP (ingest protocol, not delivery).

## Checklist Before Moving On
- [ ] Understand HLS: segments, playlists, adaptive bitrate
- [ ] Can design the transcoding pipeline (upload → Kafka → workers → S3)
- [ ] Know CDN strategy for video (segment-level caching, origin shield)
- [ ] Understand the upload and streaming flows
- [ ] Can explain why video is split into segments
- [ ] Solved the Video Streaming design using RESHADED
