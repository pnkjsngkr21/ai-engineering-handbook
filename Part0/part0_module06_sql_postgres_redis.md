# PART 0 — Prerequisites
## Module 6: SQL, PostgreSQL & Redis

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Design schemas for AI-application-specific data (conversation history,
  tool-call logs, user/session state) with the same rigor you already apply
  to relational schemas.
- Explain exactly what an index does, when it helps, and when it silently
  doesn't (a gap many backend engineers have even after years of SQL use).
- Use Postgres for more than plain relational data: JSONB columns for
  semi-structured LLM outputs, and (conceptually, ahead of Part 4) pgvector
  for embeddings.
- Use Redis correctly for the specific things it's good at in AI systems —
  caching model responses, rate limiting, short-term conversation memory,
  distributed locks — and know when Postgres is actually the better choice.
- Reason about connection pooling, transactions, and consistency trade-offs
  under concurrent load, the exact skills tested in backend system-design
  interviews.

### 2. Prerequisites
Modules 1–5. You already know SQL and probably Postgres from backend work —
this module deepens indexing/query-planning intuition and adds the
AI-specific patterns (JSONB, pgvector, Redis-as-LLM-cache) you haven't
necessarily used yet.

### 3. Estimated Study Time
10–12 hours over 5–6 days.

### 4. Difficulty
⭐⭐⭐☆☆ (Medium — SQL syntax is easy for you; query planning and Redis data
modeling depth is where the real learning is.)

### 5. Why This Matters
This is squarely your strength area, and it's also *exactly* what shows up
in senior backend interviews (including the fan-out/aggregation patterns
you've been practicing) and in nearly every AI system you'll build: chat
history has to live somewhere, expensive LLM calls should be cached, rate
limits need fast counters, and — starting in Part 4 — vector search often
runs directly inside Postgres via pgvector rather than a separate database.
Getting this module genuinely solid pays off across the entire rest of the
handbook.

---

### 6. Theory

**What is it?**
PostgreSQL is a relational database with (crucially for AI work) strong
extension support — `pgvector` turns it into a capable vector database
without introducing a whole separate system. Redis is an in-memory data
structure store, used as a cache, message broker, and fast counter/lock
store — not a general-purpose database.

**Why do we need both (not just Postgres)?**
Different durability/latency trade-offs for different needs:
- Postgres: durable source of truth (users, conversations, documents),
  supports complex queries/joins/transactions.
- Redis: sub-millisecond reads for data that's okay to lose on restart (a
  cache) or that's inherently ephemeral (a rate-limit counter, a short TTL
  session).

For AI systems specifically: **caching LLM responses** is one of the
highest-leverage cost/latency optimizations available (identical or
near-identical prompts shouldn't re-trigger a paid, slow model call), and
Redis is the natural place to do this — you'll build this for real in Part 3.

**How does indexing actually work (the depth most engineers skip)?**

A B-tree index (Postgres's default) lets the query planner find rows
matching a condition in `O(log n)` instead of scanning every row (`O(n)`).
But an index only helps if the query planner *chooses* to use it, which
depends on:
- **Selectivity** — if a `WHERE` clause matches most of the table (e.g.,
  `status != 'deleted'` on a table where nearly everything is
  non-deleted), a sequential scan can be *faster* than an index scan, and
  Postgres's planner will correctly choose the seq scan. This surprises
  people who assume "index = always faster."
- **Column order in composite indexes** — an index on `(user_id,
  created_at)` helps queries filtering by `user_id` alone or by both
  columns, but does **not** help a query filtering by `created_at` alone
  (leftmost-prefix rule — directly analogous to how you'd design a compound
  key in any ordered structure).
- **Function/expression wrapping** — `WHERE LOWER(email) = 'x'` cannot use
  a plain index on `email`; you need an expression index on `LOWER(email)`,
  or normalize case on write instead.

**Always confirm with `EXPLAIN ANALYZE`, never assume.** This one habit —
actually reading query plans instead of guessing — is the single biggest
lever for backend interview performance on DB questions, and for real
production query debugging.

**JSONB — the AI-relevant Postgres feature:**
LLM outputs (tool calls, structured extraction results) are often
semi-structured JSON that doesn't map cleanly to rigid columns. Postgres's
`JSONB` type stores this efficiently and **can be indexed** (GIN indexes),
letting you query inside JSON blobs without abandoning relational
guarantees for the rest of your schema. This is the pattern you'll use
constantly in Part 3/4/5 for storing tool-call logs, agent traces, and RAG
metadata.

**When should Redis be used vs. avoided?**
Use for: caching, rate limiting (`INCR` + `EXPIRE` is the classic sliding-
window-ish counter pattern), distributed locks, short-lived session/queue
data, pub/sub for lightweight real-time signals.
Avoid for: anything that must survive a restart with strong durability
guarantees by default (Redis *can* persist to disk, but it's not the
primary use case and adds operational complexity) — that data belongs in
Postgres.

---

### 7. Mental Models

**Model 1 — "An index is a sorted lookup structure, and the planner is a
cost-based optimizer, not a rule-based one."** It picks the seq scan or
index scan based on estimated cost, using table statistics — this is why
`ANALYZE` (updating statistics) matters after big data changes.

**Model 2 — "Redis is a cache, not a database, even though it can persist
data."** If losing the data on a restart would be a real problem, it
belongs in Postgres (possibly with Redis in front of it as a cache).

**Model 3 — "JSONB in Postgres is 'schema-on-read' living inside a
'schema-on-write' system."** You get relational guarantees for your core
schema and flexible, indexable semi-structured storage for the genuinely
variable parts (LLM outputs) — best of both, used deliberately, not as an
excuse to avoid schema design entirely.

---

### 8. Visual Explanation (described)

**Diagram: "Where Postgres and Redis sit in an AI request's path"**
```
User message
   |
   v
[ App server ] --check cache--> [ Redis: has this exact prompt been
   |                               answered recently? ]
   | (cache miss)
   v
[ LLM Provider API ] --response--> [ Redis: cache response, TTL'd ]
   |
   v
[ Postgres: durably store conversation turn, user_id, tool calls (JSONB) ]
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **"Use the Index, Luke!" (use-the-index-luke.com)** — the single best
   free resource on how indexes and query planners actually work,
   database-agnostic principles with SQL examples.
2. **PostgreSQL official docs — "Indexes" and "JSON Types" chapters** —
   read these directly rather than a third-party summary; Postgres's own
   docs are unusually clear.
3. **Redis official docs — "Data types" and "Redis as a cache" pages** —
   understand each data structure (string, hash, sorted set, list) as a
   distinct tool, not interchangeable.
4. **`pgvector` GitHub README** — skim now (full depth comes in Part 4) so
   you recognize the extension when you meet it again.

**Official documentation:** postgresql.org/docs (excellent, authoritative),
redis.io/docs.

**GitHub repos:** `pgvector/pgvector` (README + examples), any well-known
FastAPI + SQLAlchemy + Alembic starter template for schema/migration
conventions you'll reuse in Part 1.

---

### 17. Exercises

1. Create a table with 100k+ synthetic rows, run a query with and without
   an index on the filtered column, and compare `EXPLAIN ANALYZE` output —
   confirm you can read "Seq Scan" vs. "Index Scan" and the estimated vs.
   actual row counts.
2. Design a composite index for a `messages` table queried both by
   `(conversation_id, created_at)` and by `conversation_id` alone; explain
   why a single composite index (in the right column order) serves both
   query shapes.
3. Implement a Redis-based rate limiter using `INCR` + `EXPIRE` (fixed
   window) and explain its main weakness (bursty traffic at window
   boundaries) compared to a sliding-window approach.
4. Store a sample LLM tool-call response as JSONB in Postgres, then write a
   query that filters on a nested field using a GIN index.

### 18. Mini-Project
**Build:** A `conversations` schema (Postgres) with `users`, `conversations`,
`messages` (with a `JSONB` column for tool-call metadata) tables, proper
foreign keys, and at least one composite index justified by a real query
pattern you write and `EXPLAIN ANALYZE` yourself.

### 19. Production Project
**Build:** `llm-cache-layer` — a small service (FastAPI, building slightly
ahead — a bare-bones app is fine here) that:
- Wraps a (mocked, for now) "expensive LLM call" function
- Checks Redis first using a cache key derived from a hash of the
  normalized prompt + model + parameters
- On a miss, calls the (mocked) function, stores the result in Redis with a
  sensible TTL, and logs cache hit/miss rates
- Also implements a Redis-based rate limiter (per-user) protecting the
  endpoint
- Includes tests for both the caching logic and the rate limiter,
  including edge cases (TTL expiry, concurrent requests for the same
  uncached prompt — the "cache stampede" problem, worth researching and
  explicitly handling, e.g., via a short-lived lock)

This becomes the real caching layer you'll plug an actual LLM call into in
Part 3 — cost/latency optimization is one of the highest-value skills in
this whole handbook, and this project is where you build it properly.

---

### 20–21. Practice & Interview Questions

1. Explain, using `EXPLAIN ANALYZE` output as your evidence, why an index
   isn't always used even when one exists on the filtered column.
2. What's the leftmost-prefix rule for composite indexes, and how does it
   change how you order columns when creating one?
3. Design a caching strategy for an LLM-powered feature that reduces both
   cost and latency — what do you cache, what's your cache key, what
   invalidates it, and how do you avoid a cache stampede?
4. Why would you choose Redis over Postgres for rate limiting, and what
   would go wrong if you tried to do high-frequency rate-limit counters
   directly against Postgres instead?
5. Explain the difference between a fixed-window and sliding-window rate
   limiter, and a scenario where the difference actually matters (bursty
   clients right at window boundaries).

---

### 22. Common Mistakes
- Assuming any index automatically speeds up any query touching that
  column, without checking selectivity or `EXPLAIN ANALYZE`.
- Wrapping indexed columns in functions in `WHERE` clauses (`LOWER(col)`)
  without an expression index, silently disabling index usage.
- Using Redis as a system of record for data that actually needs durability
  guarantees.
- Building a cache without solving the "cache stampede" problem — many
  concurrent requests for the same uncached key all missing simultaneously
  and hammering the expensive backend at once.
- Forgetting TTLs on cache entries, leading to unbounded memory growth or
  serving stale data indefinitely.

### 23. Debugging Exercise
Given a query that's slow in production but fast on a small local dataset,
practice diagnosing via `EXPLAIN ANALYZE` on a realistically-sized dataset
(not the tiny local one) — the classic trap of "works fine in dev" because
dev data is too small for the query planner's choices to matter yet.

---

### 24. Checklist
- [ ] I can read `EXPLAIN ANALYZE` output confidently, not just run it
- [ ] I understand the leftmost-prefix rule for composite indexes
- [ ] I've used JSONB + GIN indexing for semi-structured data
- [ ] I've built a working Redis cache layer with stampede protection
- [ ] I can articulate when data belongs in Postgres vs. Redis, with reasons

### 25. Summary
This module deepened existing SQL/Postgres strength (query planning,
indexing internals, JSONB for AI's semi-structured outputs) and added the
specific Redis patterns — caching, rate limiting, stampede protection —
that matter most in AI systems, where the "expensive backend call" you're
protecting against is a slow, costly LLM API rather than a typical DB
query. The caching-layer project here is a direct rehearsal for real cost
optimization work in Part 3.

### 26. Next Steps
Module 7: **JavaScript, TypeScript & the Modern Frontend Toolchain** —
building the frontend fluency needed for Part 7 (chat UIs, streaming
rendering) and the freelancing/SaaS-building track in Parts 9/11, taught
efficiently for someone whose primary strength is backend.

---

**Reply "continue" for Module 7, or flag anything to go deeper on.**
