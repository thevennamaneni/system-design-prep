# 30-Day System Design Interview Prep Plan

> **For:** Senior Go Developer (9 yrs exp), zero system design experience
> **Goal:** Pass a FAANG-level System Design interview

---

## What Is a System Design Interview?

Unlike LeetCode (algorithmic coding) or LLD (code-level design), a **System Design (SD) interview** asks you to design a **large-scale distributed system** at the architecture level. You'll be asked questions like:

- "Design Twitter"
- "Design a URL shortener"
- "Design Netflix"
- "Design Uber"

You're expected to discuss **servers, databases, caches, message queues, load balancers, APIs, and how they all interact at scale** — handling millions of users, terabytes of data, and high availability requirements.

A typical SD round is **45-60 minutes**: 5 min requirements, 10 min high-level design, 25 min detailed design, 10 min scalability/deep-dive, 5 min discussion.

---

## The Senior Engineer Advantage (READ THIS)

With 9 years of Go experience, you have a **massive advantage** in system design interviews:

1. **You understand production systems.** You've dealt with real services, databases, deployments. That intuition is invaluable.
2. **Go is built for distributed systems.** Goroutines, channels, context, gRPC — Go powers Kubernetes, Docker, Terraform, etcd. You can speak to real implementation details.
3. **Senior interviews expect depth.** FAANG expects you to discuss trade-offs, operational concerns, and edge cases — not just draw boxes. Your experience qualifies you.
4. **You can anchor abstract concepts.** When we discuss "eventual consistency," you can relate it to real bugs you've seen. That's gold in an interview.

---

## How To Use This Plan

Each day has its own folder with a `README.md` containing:
- **Topic** — the concept(s) covered
- **Why It Matters** — relevance to SD interviews
- **Detailed Explanation** — theory, architecture diagrams (ASCII), trade-offs
- **Go Connection** — how this relates to your Go experience
- **Exercise / Example** — a problem to think through
- **Common Mistakes** — what trips up candidates
- **Checklist** — self-verification

## Daily Routine (2-3 hours/day)

1. **Read** the day's README (30-45 min) — understand the concept deeply.
2. **Study** a real-world example (30 min) — read engineering blogs (Uber, Netflix, Meta).
3. **Draw** the architecture (30 min) — practice whiteboarding. ASCII diagrams are fine.
4. **Discuss trade-offs** out loud (15 min) — rehearse interview communication.
5. **Notes** — add to your personal cheat sheet.

## Weekly Schedule

| Week | Theme | Days |
|------|-------|------|
| **Week 1** | Foundations: Networking, Scaling & Databases | 1-7 |
| **Week 2** | Core Building Blocks: Caching, MQs, API Design & Microservices | 8-14 |
| **Week 3** | Advanced: Consensus, Real-time, Streaming & Observability | 15-21 |
| **Week 4** | Full System Designs & Mock Interviews | 22-30 |

## The 15 Building Blocks of System Design

Every system you design will use some combination of these:

| # | Building Block | Day |
|---|---------------|-----|
| 1 | DNS, TCP/IP, HTTP | 2 |
| 2 | Load Balancers | 3 |
| 3 | Horizontal & Vertical Scaling | 3 |
| 4 | SQL Databases | 4 |
| 5 | NoSQL Databases | 4 |
| 6 | CAP Theorem & Consistency | 5 |
| 7 | Object/Block/File Storage & CDNs | 6 |
| 8 | Caching (Redis, Memcached) | 8 |
| 9 | Message Queues (Kafka, RabbitMQ) | 9 |
| 10 | API Design (REST, gRPC, GraphQL) | 10-11 |
| 11 | Microservices & API Gateway | 12 |
| 12 | Database Sharding & Replication | 13 |
| 13 | Distributed Consensus & Coordination | 15-16 |
| 14 | Real-time Communication | 17 |
| 15 | Observability (Metrics, Logs, Traces) | 19 |

## The System Design Interview Framework (RESHADED)

Use this acronym to structure any system design answer:

| Letter | Step | Time |
|--------|------|------|
| **R** | **Requirements** — functional & non-functional | 5 min |
| **E** | **Estimation** — capacity, QPS, storage, bandwidth | 3 min |
| **S** | **Storage schema** — database choices, schema design | 5 min |
| **H** | **High-level design** — draw the box-and-arrow diagram | 10 min |
| **A** | **API design** — endpoints, request/response formats | 5 min |
| **D** | **Detailed design** — deep dive into components | 15 min |
| **E** | **Edge cases & Errors** — failure handling, scaling | 5 min |
| **D** | **Discussion** — trade-offs, alternatives, bottlenecks | 5 min |

> We'll practice this framework repeatedly in Week 4.

## Golden Rules

1. **Clarify before designing.** Never start drawing without understanding requirements.
2. **Think out loud.** The interviewer evaluates your thought process, not just the final diagram.
3. **Start simple, then scale.** Design for 100 users first, then scale to millions.
4. **Always discuss trade-offs.** Every choice has pros/cons. "I chose Redis for caching because... the trade-off is..."
5. **Use numbers.** Back-of-envelope estimation shows senior-level thinking.
6. **Don't name-drop without understanding.** Saying "Kafka" without knowing partitioning is worse than saying "message queue" and explaining the concept.
7. **Drive the interview.** Don't wait for the interviewer to steer — propose, justify, move forward.
8. **Relate to Go.** When relevant, mention Go's strengths (goroutines for concurrency, gRPC, context for cancellation).

---

**Let's begin. Open `Day01/README.md`.**
