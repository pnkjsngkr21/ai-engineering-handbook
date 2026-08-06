# PART 0 — Prerequisites
## Module 13: System Design Fundamentals

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Walk through a structured system design interview framework confidently
  — requirements clarification, capacity estimation, high-level design,
  deep dives, trade-offs — directly applicable to your DoorDash-style
  interviews.
- Explain and correctly apply core patterns: load balancing, caching
  layers, message queues, database replication/sharding, and the CAP
  theorem trade-offs behind them.
- Design fan-out/aggregation architectures (the exact pattern you've been
  practicing) with proper failure-handling: timeouts, circuit breakers,
  graceful degradation.
- Recognize which of these patterns show up, with a twist, in AI system
  architecture (Parts 3–8) — e.g., caching becomes "semantic caching,"
  queues become "async agent task orchestration."
- Estimate scale (back-of-envelope capacity planning) for both traditional
  and AI-specific workloads (tokens/sec, embedding throughput).

### 2. Prerequisites
Modules 1–12. This module is a synthesis point — it assumes comfort with
async/concurrency (Module 12), caching (Module 6), and networking (Module
4), pulling them into a coherent system-design vocabulary.

### 3. Estimated Study Time
14–18 hours over 7–9 days. This is a dense, high-leverage module — treat
it as a genuine investment given your interview timeline, not a quick
skim.

### 4. Difficulty
⭐⭐⭐☆☆ (Medium for you specifically — you already have real production
instincts from backend work; the value here is formalizing vocabulary and
a repeatable interview framework, not learning new engineering concepts
from scratch.)

### 5. Why This Matters
This module is the most directly interview-relevant material in Part 0,
given your active DoorDash-style prep. It's also the connective tissue for
the rest of this handbook: every AI system you design from Part 3 onward
(RAG pipelines, agent orchestration, model-serving infrastructure) is a
system design problem wearing an AI costume — the same patterns
(caching, queues, load balancing, graceful degradation) reappear with
AI-specific variations throughout.

---

### 6. Theory

**What is it?**
System design is the practice of decomposing a problem into components
(services, data stores, queues) and reasoning about their interactions
under real-world constraints: scale, latency, consistency, and failure.
There's no single "correct" design — only better and worse trade-offs for
a given set of requirements, which is exactly why interviews probe your
reasoning, not just your final diagram.

**A repeatable interview framework (use this structure every time):**
1. **Clarify requirements** — functional (what must it do) and
   non-functional (scale, latency targets, consistency needs). *Always* ask
   about scale explicitly (users, requests/sec, data volume) before
   designing — designs for 100 users and 100 million users look
   completely different.
2. **Capacity estimation** — rough back-of-envelope math: requests/sec,
   storage growth/day, bandwidth. Purpose isn't precision — it's revealing
   which components will actually be bottlenecks.
3. **High-level design** — draw the major components and how data flows
   between them, before drilling into any one piece.
4. **Deep dive** — pick the 1-2 hardest/most interesting parts (usually
   what the interviewer probes) and go deep: schema design, specific
   algorithm, specific failure mode.
5. **Trade-offs and failure modes** — explicitly discuss what breaks under
   load, under partial failure, and what you'd monitor.

**Core patterns, precisely (you know most of these — this is calibration,
not first-exposure):**

**Load balancing:** distributing requests across multiple service
instances. Algorithms: round-robin (simple, ignores load), least-
connections (better under uneven request costs), consistent hashing
(critical when you need request affinity, e.g., routing to the instance
holding relevant cached state).

**Caching layers:** you know this deeply from Module 6. The system-design
addition here: **cache invalidation strategies** (TTL-based, write-through,
write-behind, cache-aside) and **where** to cache (CDN edge, application-
layer Redis, database query cache) — each layer trades freshness for
latency differently.

**Message queues:** decouple producers from consumers, absorb bursty load,
and enable async processing. Key trade-off: **at-least-once vs. exactly-
once delivery** — most real queues (SQS, Kafka) give at-least-once by
default, meaning your consumers must be **idempotent** (directly connecting
back to Module 4's idempotency-key discussion).

**Database replication/sharding:** replication (read replicas) scales
reads and adds durability; sharding (horizontal partitioning) scales writes
and total data volume, at the cost of cross-shard query complexity and
harder transactions.

**CAP theorem — precisely, not just the slogan:** during a network
partition, a distributed system must choose between **Consistency**
(every read sees the latest write) and **Availability** (every request
gets a response, possibly stale). This is a **partition-time** trade-off
specifically — outside of a partition, you can often have both. Most
practical systems pick a point on the **consistency spectrum**
(strong/linearizable → eventual) per use case rather than treating CP/AP
as one global system-wide choice.

**Fan-out with graceful degradation (your existing practice area,
formalized):**
```
Request comes in -> fan out to Service A, B, C in parallel (CompletableFuture.allOf)
    - A: critical, must succeed, or fail the whole request
    - B: important but optional; on failure/timeout, use a cached/default value
    - C: nice-to-have; on failure/timeout, simply omit that part of the response
Circuit breaker: if a downstream service fails repeatedly, stop calling it
    for a cooldown period, failing fast instead of waiting on doomed calls
```
This exact pattern reappears in Part 5 (multi-agent systems: some
sub-agent calls are critical, others are enrichment that can degrade
gracefully) and Part 6 (model-serving with fallback models).

**How AI systems reuse these same patterns (a preview, connecting Part 0
to the rest of the handbook):**

| Traditional pattern           | AI-system variant (previewed, built for real in Parts 3-6)          |
|--------------------------------|------------------------------------------------------------------------|
| Cache-aside (Redis)             | **Semantic caching** — cache by embedding similarity, not exact key match, since near-duplicate prompts should hit cache too |
| Message queue for async jobs   | Agent task orchestration / job queues for long-running agent workflows  |
| Load balancing across instances | Load balancing across multiple model replicas or providers (fallback between providers on failure) |
| Circuit breaker on a flaky DB   | Circuit breaker / fallback model when a primary LLM provider is degraded or rate-limiting hard |
| Database sharding               | Vector index sharding/partitioning at large scale (Part 4)             |

---

### 7. Mental Models

**Model 1 — "There is no perfect design, only trade-offs matched to
requirements you clarified up front."** The interview signal isn't "did
you draw the right boxes" — it's "did you reason correctly about
trade-offs given explicit constraints."

**Model 2 — "CAP is about behavior *during a partition*, not a permanent
label on a whole system."** Different operations within the same system
can make different CAP trade-offs (e.g., strongly consistent writes to a
user's own data, eventually consistent for a public feed).

**Model 3 — "Graceful degradation means deciding, per dependency, whether
its failure is fatal, degradable, or omittable — and building that
decision into the code, not hoping it doesn't happen."** This maps
directly onto your fan-out REST endpoint practice work.

---

### 8. Visual Explanation (described)

**Diagram: "Fan-out with graceful degradation"**
```
                    [ Incoming Request ]
                             |
          -------------------------------------------
          |                 |                        |
   [Service A: CRITICAL] [Service B: IMPORTANT] [Service C: OPTIONAL]
     must succeed          fallback to cache      fallback: omit
          |                 |                        |
          -------------------------------------------
                             |
                    [ Aggregate response ]
                    (A's result required;
                     B's cache-or-fresh;
                     C's data or nothing)
```

**Diagram: "Circuit breaker states"**
```
CLOSED (normal) --[failure threshold exceeded]--> OPEN (fail fast, no calls)
   ^                                                        |
   |                                          [cooldown timer expires]
   |                                                        v
   ------------- [test request succeeds] ---------- HALF-OPEN (try one request)
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **"Designing Data-Intensive Applications" by Martin Kleppmann** —
   Chapters 1, 5, and 9 specifically (reliability/scalability, replication,
   consistency/consensus) — this book is *the* canonical reference and
   you'll return to it in Part 6 as well; don't try to read it cover to
   cover yet, target these chapters now.
2. **"System Design Interview" by Alex Xu (Vol. 1)** — practical,
   interview-framework-oriented, good complement to Kleppmann's deeper
   theory.
3. **AWS Architecture Blog — "Exponential Backoff and Jitter"** (revisit
   from Module 4, now in the fuller system-design context) and
   "Circuit Breaker pattern" (Microsoft's Azure Architecture Center has an
   excellent write-up of this pattern specifically).
4. **Martin Fowler's "CircuitBreaker" article** (martinfowler.com) — the
   canonical explanation of the pattern.

**Official documentation/deep resources:** Azure Architecture Center's
"Cloud Design Patterns" catalog (comprehensive, pattern-by-pattern,
genuinely excellent even outside an Azure context).

**GitHub repos:** look at `resilience4j` (Java) even though you're working
mostly in Python now — its documentation of circuit breaker, retry, and
bulkhead patterns is exceptionally clear and the concepts transfer directly
to any language.

---

### 17. Exercises

1. Do a full back-of-envelope capacity estimate for a hypothetical
   "AI customer support chat" system: assume 1M daily active users, 5
   messages/user/day average — estimate requests/sec (accounting for peak
   vs. average), storage growth/day for conversation history, and
   bandwidth for typical message sizes.
2. Design (diagram + written trade-off discussion) a fan-out endpoint that
   aggregates a "product detail page" from 3 services (pricing, inventory,
   reviews) — decide per-service whether it's critical/important/optional
   and justify each choice.
3. Explain, in writing, a real (or hypothetical) scenario where you'd
   choose eventual consistency over strong consistency, and one where
   you'd insist on strong consistency — justify both.
4. Implement a simple circuit breaker (state machine: closed → open →
   half-open) in Python, wrapping a mock flaky function, and demonstrate
   it transitioning through all three states under simulated failures and
   recovery.

### 18. Mini-Project
**Build:** A written system design document (the kind you'd produce in a
real interview, or as an RFC at a company) for "Design a URL shortener" —
a classic, well-scoped practice problem — following the 5-step framework
above in full: requirements, capacity estimate, high-level design (with a
described diagram), a deep dive (e.g., ID generation strategy, choosing
between counter-based vs. hash-based short codes), and trade-offs/failure
modes.

### 19. Production Project
**Build:** `resilient-gateway` — extend `convo-api` (Module 9) with real
resilience patterns, directly rehearsing your fan-out interview practice:
- An endpoint that fans out to 3 mock downstream services (simulate with
  configurable random latency/failure rates) using `asyncio.gather`
  (Module 12)
- Per-service classification (critical/important/optional) with
  appropriate fallback behavior for non-critical failures
- A real circuit breaker implementation (not a library — build it
  yourself once, to understand it fully) wrapping each downstream call,
  with configurable failure threshold and cooldown
- Per-call timeouts (Module 4) so one slow service can't stall the whole
  request
- Structured logging showing circuit breaker state transitions and
  fallback decisions
- Tests simulating sustained failures (confirming the breaker opens),
  cooldown expiry (confirming it moves to half-open), and recovery
  (confirming it closes again)
- README explaining the design, explicitly framed as interview-practice
  material you can walk through out loud

This project is essentially your system-design interview prep made
tangible and testable — a strong artifact to reference directly in
interviews ("I built and tested this exact pattern").

---

### 20–21. Practice & Interview Questions

1. Walk through your system design interview framework end to end for a
   prompt like "design a rate limiter" or "design a notification system."
2. Explain the CAP theorem precisely — what does it actually constrain,
   and what's a common misunderstanding of it?
3. Design a fan-out endpoint aggregating data from 3 downstream services
   with different criticality levels — explain your timeout, fallback, and
   circuit-breaker choices for each.
4. What's the difference between at-least-once and exactly-once message
   delivery, and why does at-least-once delivery require idempotent
   consumers?
5. When would you choose database sharding over read replicas, and what
   new problems does sharding introduce (cross-shard queries/transactions,
   rebalancing)?
6. Explain the three states of a circuit breaker and why the half-open
   state exists (to avoid either permanently giving up or immediately
   overwhelming a recovering service).

---

### 22. Common Mistakes
- Diving into a detailed design before clarifying requirements and scale —
  the single most common system design interview mistake.
- Treating CAP theorem as "pick CP or AP for your whole system forever"
  instead of a per-operation, partition-time trade-off.
- Building fan-out logic where one optional dependency's failure takes
  down the entire response, instead of failing gracefully.
- Retrying against a downstream service that's clearly down repeatedly
  instead of using a circuit breaker to fail fast.
- Skipping capacity estimation entirely, missing which component will
  actually be the bottleneck at the stated scale.

### 23. Debugging Exercise
Given a production incident narrative where one slow, degraded downstream
service caused a cascading failure across an entire fan-out API (thread/
connection pool exhaustion from many requests all waiting on the slow
service), diagnose the missing pieces (no timeout, no circuit breaker, no
bulkhead isolating the failing dependency's resource pool from the rest)
and redesign the system to prevent recurrence.

---

### 24. Checklist
- [ ] I can run the 5-step system design framework fluently, unprompted,
      on a new prompt
- [ ] I can explain CAP theorem correctly, including the common
      misconception
- [ ] I've built and tested a real circuit breaker myself, not just read
      about the pattern
- [ ] I can design and justify per-dependency fallback strategies in a
      fan-out architecture
- [ ] I've completed the URL shortener design doc and `resilient-gateway`

### 25. Summary
This module formalized system design vocabulary and a repeatable interview
framework around patterns you already have real instincts for from backend
work — fan-out, caching, graceful degradation. The circuit breaker you
built by hand in `resilient-gateway` and the classification of downstream
dependencies as critical/important/optional are exactly the concepts your
DoorDash-style interviews will probe, and exactly the patterns that
reappear, with an AI twist, throughout Parts 3–8.

### 26. Next Steps
Module 14: **Cloud Fundamentals (AWS-first)** — the final module of Part
0, covering the cloud primitives (compute, storage, networking, IAM) that
every deployment from Part 8 onward assumes, and that directly support your
longer-term platform-engineering goal.

---

**Reply "continue" for Module 14 (the final Part 0 module), or flag
anything to go deeper on.**
