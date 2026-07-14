# Day 28 - Mock Interview #1 (Full System Design)

## Topic
**Simulated system design interview.** Pick an unseen problem, use RESHADED, and practice the full 45-60 minute process. Today simulates the real interview.

---

## Why This Day Matters
- Days 22-27 taught you full designs step-by-step. Today you do it **without guidance**.
- System design interviews test process as much as knowledge — you need to rehearse thinking out loud.
- This mock reveals your weak spots (estimation? sharding? trade-offs?) before the real interview.

---

## Mock Interview Protocol

### Setup
- Pick **one problem** from the list below (that you haven't fully designed yet).
- Set a timer for **60 minutes**.
- Use a whiteboard, notes app, or text file for your design.
- **Narrate out loud** as if an interviewer is watching (record yourself if possible).
- No looking at notes or the previous days' solutions during the mock.

### The 60-Minute Structure (RESHADED)
| Time | Step | What to Do |
|------|------|-----------|
| 0-5 min | **R** — Requirements | Ask clarifying questions. Write down functional + non-functional requirements. Define scope. |
| 5-10 min | **E** — Estimation | Calculate QPS, storage, bandwidth, cache size. Show your work. |
| 10-15 min | **S** — Storage schema | Choose databases. Design tables/collections. Justify choices. |
| 15-25 min | **H** — High-level design | Draw the box-and-arrow diagram. Start simple, then add components. |
| 25-30 min | **A** — API design | Define endpoints. Request/response formats. Pagination. |
| 30-45 min | **D** — Detailed design | Deep dive into the hardest part. Sharding? Fan-out? Real-time? Discuss patterns. |
| 45-55 min | **E** — Edge cases | Failure handling. Scale. Bottlenecks. |
| 55-60 min | **D** — Discussion | Trade-offs. Alternatives. "What would you do differently with 10x budget?" |

### What to Narrate (even alone)
- "Let me clarify the requirements first. I'll assume..."
- "The key entities are..."
- "I'll use PostgreSQL for X because ACID is needed. For Y, I'll use Cassandra because of high write throughput."
- "The bottleneck here is... I'll solve it with..."
- "The trade-off of this approach is..."
- "If we need to scale 10x, I would..."

---

## Mock Problems (pick one you haven't designed)

### 1. Design Instagram (Photo Sharing)
**Focus:** Media storage (S3+CDN), feed generation (fan-out), social graph, image processing.
**Key challenges:** Feed generation (push vs pull), media storage at scale, CDN strategy.
**Similar to:** Twitter (Day 23) but media-heavy.

### 2. Design Google Docs (Collaborative Editing)
**Focus:** Real-time collaboration, conflict resolution (OT/CRDT), document storage.
**Key challenges:** Concurrent edits, operational transform, WebSocket management.
**New concepts:** Operational Transformation (OT) or CRDTs for conflict resolution.

### 3. Design Spotify (Music Streaming)
**Focus:** Media streaming, recommendations, playlist management.
**Key challenges:** CDN for audio, recommendation algorithm, offline playback.
**Similar to:** Netflix (Day 25) but audio (smaller files, different bitrate needs).

### 4. Design Airbnb (Property Booking)
**Focus:** Search (geospatial), availability calendar, booking flow, payments.
**Key challenges:** Date-range search, concurrent booking (two people booking same dates), pricing.
**New concepts:** Availability calendar data model, concurrent booking prevention.

### 5. Design Google Search (Web Search)
**Focus:** Crawling, indexing, ranking, query processing.
**Key challenges:** Web-scale crawling, inverted index, PageRank, query latency.
**New concepts:** Inverted index, crawler architecture, ranking signals.

### 6. Design a Distributed Cache (like Redis Cluster)
**Focus:** Consistent hashing, replication, failover, cache invalidation.
**Key challenges:** Sharding, rebalancing, consistency vs availability.
**New concepts:** Consistent hashing in practice, gossip protocol, partition healing.

---

## Self-Evaluation Rubric

Score each dimension (1-5):

| Dimension | Score | Notes |
|-----------|-------|-------|
| Requirements clarification | /5 | Did you ask good questions? Define scope? |
| Estimation | /5 | Were numbers reasonable? Showed your work? |
| Storage design | /5 | Right DB choices? Schema makes sense? |
| High-level architecture | /5 | Complete diagram? Correct data flow? |
| API design | /5 | RESTful? Good endpoints? Pagination? |
| Detailed design | /5 | Deep dive into the hard part? Discussed patterns? |
| Edge cases | /5 | Handled failures, scale, bottlenecks? |
| Trade-off analysis | /5 | Justified choices? "Because... trade-off is..." |
| Communication | /5 | Narrated throughout? Clear explanations? |
| Time management | /5 | Finished in 60 min? Didn't get stuck? |

**Total: ___ / 50.** Target: 35+ before the real interview.

---

## Post-Mock Reflection (write answers)

1. Did I spend enough time on requirements, or did I rush to draw?
2. Were my estimates reasonable, or did I get lost in numbers?
3. Did I choose the right databases? Could I justify each choice?
4. Did I identify the hardest part and deep-dive into it?
5. Did I discuss trade-offs for every major decision?
6. Could I explain this design to a non-engineer?
7. Where did I lose the most time? How to prevent that in the real interview?
8. What concept did I struggle with? → Add to weak list for Day 30.

---

## If You Got Stuck

This is normal — it's a mock. After 60 min:
1. Review what blocked you (estimation? sharding? a specific component?).
2. Re-read the relevant day's material.
3. Re-attempt the stuck part tomorrow before Day 29.
4. Add the blocking concept to your weak list.

---

## Bonus (if finished early)
Do a **second** mock with a different problem. The more reps, the calmer you'll be on the real day.

---

## Checklist Before Moving On
- [ ] Completed one full 60-min mock interview
- [ ] Filled out the self-evaluation rubric honestly
- [ ] Identified 2-3 weak areas to review
- [ ] Ready for the second mock (Day 29)
