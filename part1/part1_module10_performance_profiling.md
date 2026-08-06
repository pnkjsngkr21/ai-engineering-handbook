# PART 1 — Software Engineering
## Module 10: Performance & Profiling

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Profile Python code correctly using both statistical and deterministic
  profilers, and read a flame graph to find real bottlenecks instead of
  guessing.
- Explain precisely where performance actually matters in an AI
  application — usually not raw Python execution speed (Part 0 Module 1's
  lesson, revisited with real profiling evidence).
- Optimize the things that actually matter for AI-system latency: network
  round-trips, database query efficiency, connection reuse, and
  concurrency correctness — rather than micro-optimizing Python loops.
- Load-test an API to understand its actual capacity and failure
  characteristics under concurrent load.
- Make a deliberate build/optimize decision using data (a profile, a
  load-test result) rather than intuition, every time.

### 2. Prerequisites
Modules 1–9, Part 0 Modules 1, 4, 6, 12 (Python performance model,
networking, SQL indexing, async).

### 3. Estimated Study Time
8–10 hours over 4–5 days.

### 4. Difficulty
⭐⭐☆☆☆ (Easy-Medium — mostly applying tools correctly and building the
discipline of "measure before optimizing," which you likely already value
from backend work.)

### 5. Why This Matters
Performance work without measurement is guesswork, and guesswork in AI
systems is especially likely to be wrong — the bottleneck is almost always
network/model latency, not your Python code, and profiling is what proves
that instead of assuming it. This discipline is also directly interview-
relevant (performance/profiling questions are common in senior backend
interviews) and prevents wasted effort optimizing the wrong thing.

---

### 6. Theory

**What is it?**
Profiling measures where time (or memory) is actually spent in a running
program, so optimization effort goes where it actually pays off. Two main
approaches:
- **Deterministic profiling** (`cProfile`) — instruments every function
  call, giving exact call counts and cumulative time per function, but
  with real overhead that can distort timing for very fast functions.
- **Statistical/sampling profiling** (`py-spy`, `austin`) — periodically
  samples the call stack without instrumenting every call, much lower
  overhead, suitable for profiling production processes safely (`py-spy`
  can even attach to an already-running process without restarting it —
  genuinely useful for diagnosing a live production issue).

```bash
py-spy record -o profile.svg -- python myapp.py   # generates a flame graph
```

**Reading a flame graph:** each box is a function call; width represents
time spent (including time in functions it calls); stacking upward shows
the call hierarchy. **Wide boxes are where time is actually going** — look
for unexpectedly wide boxes, not necessarily the ones you assumed would be
slow.

**Where AI-application bottlenecks actually live (revisiting Part 0
Module 1's claim, now with a profiling-backed methodology):**
Profile a typical AI-application request, and you'll almost always find:
1. **Network I/O waiting on the LLM provider** — often 80-95%+ of total
   request time, completely outside your code's control except through
   caching (Module 5) and choosing appropriate models/providers.
2. **Database queries** — if unindexed (Part 0 Module 6) or N+1 query
   patterns, this is the next most common real bottleneck.
3. **Your actual Python code's CPU time** — usually a rounding error by
   comparison, *unless* you're doing something CPU-bound in pure Python
   (unvectorized numeric loops, Part 0 Module 1's specific warning) inside
   the request path.

**The practical implication:** for most AI applications, "optimize the
Python code" is rarely the right instinct. **Optimize the network calls
(caching, batching, concurrency — Modules 5/12) and the database queries
(indexing — Part 0 Module 6) first**, and only reach for Python-level
micro-optimization once profiling evidence specifically points there.

**N+1 query problem — the classic real-world database bottleneck
(you likely know this from JPA/Hibernate, worth precise restating in
SQLAlchemy terms):**
```python
# N+1: one query for conversations, then ONE ADDITIONAL query per
# conversation to fetch its messages (N extra queries for N conversations)
conversations = await session.execute(select(Conversation))
for conv in conversations:
    messages = await session.execute(  # <- this runs N times!
        select(Message).where(Message.conversation_id == conv.id)
    )

# Fixed: eager-load in one query using a JOIN
conversations = await session.execute(
    select(Conversation).options(selectinload(Conversation.messages))
)
```
This is directly analogous to the classic Hibernate N+1 problem you likely
already know how to spot and fix — same root cause, same fix strategy
(eager loading), different ORM syntax.

**Load testing — understanding capacity, not just single-request
latency:**
```bash
# Using a tool like locust or k6
locust -f loadtest.py --host=http://localhost:8000
```
A single request's latency tells you little about how the system behaves
under **concurrent** load — connection pool exhaustion, database
contention, and (Part 0 Module 12's lesson) accidentally blocking the
event loop all only reveal themselves under genuine concurrent load
testing. Always load-test before claiming a system "can handle X
requests/sec."

**Memory profiling, briefly:**
```bash
py-spy dump --pid <pid>          # snapshot of what a running process is doing
memray run myapp.py               # detailed memory allocation profiling
```
Relevant for diagnosing memory leaks (Part 0 Module 3's debugging runbook,
now with a Python-specific tool) or unexpectedly high memory usage in a
long-running worker process (Module 6's background workers, particularly
relevant since a slow memory leak in a long-lived worker process is a
classic, hard-to-spot production issue).

---

### 7. Mental Models

**Model 1 — "Profile first, optimize second, always — intuition about
where time is going is wrong more often than engineers expect, including
experienced ones."**

**Model 2 — "In an AI application, the bottleneck is almost always network
waiting on the model provider, then database queries, then — rarely —
your actual Python code."** This ordering should shape where you spend
optimization effort by default, before you've even profiled, as a starting
hypothesis to then confirm or refute with real data.

**Model 3 — "Load testing reveals concurrency-specific problems (event
loop blocking, connection pool exhaustion) that single-request testing
can never reveal, no matter how carefully you measure one request."**

---

### 8. Visual Explanation (described)

**Diagram: "A typical AI-request flame graph, described"**
```
[=========================== handle_request (2400ms total) ============]
[=] parse_request (5ms)
    [======================= call_llm_provider (2100ms) ================]
        (this wide box is almost entirely NETWORK WAIT, not CPU —
         profiling makes this immediately, visually obvious)
    [==] db_write (30ms)
    [=] serialize_response (3ms)
```
The visual width immediately draws your eye to where optimization effort
would (and wouldn't) pay off — in this example, clearly the LLM call
itself (via caching/batching), not the surrounding Python glue code.

---

### 9–16. Recommended Resources

**Reading order:**
1. **`py-spy` official documentation/README (GitHub: benfred/py-spy)** —
   concise, practical, and the tool you'll use most for real profiling.
2. **Python official docs — `cProfile` and `profile`** — for the
   deterministic-profiling complement to `py-spy`'s sampling approach.
3. **SQLAlchemy docs — "Relationship Loading Techniques"** — the
   authoritative reference on eager-loading strategies (`selectinload`,
   `joinedload`) to fix N+1 query patterns.
4. **`locust` official documentation** — for load-testing setup and
   interpreting results (percentile latencies under load, not just
   averages).

**Official documentation:** docs.python.org for `cProfile`, `locust.io`
docs, SQLAlchemy's relationship-loading docs.

**GitHub repos:** `benfred/py-spy`, `bloomberg/memray` (for memory
profiling) — both have excellent example usage in their READMEs.

---

### 17. Exercises

1. Profile a request to `convo-api` (or any prior project) using `py-spy`,
   generate a flame graph, and confirm empirically where time is actually
   spent — compare against your intuition beforehand.
2. Deliberately introduce an N+1 query pattern into a SQLAlchemy-based
   endpoint, observe it via query-count logging, fix it with
   `selectinload`, and measure the before/after difference under load.
3. Load-test an endpoint with `locust`, ramping concurrent users, and
   identify at what concurrency level latency starts degrading — correlate
   this with a specific resource limit (connection pool size, event loop
   blocking) if you can find one.
4. Deliberately write a CPU-bound, unvectorized Python loop (Part 0 Module
   1's warning), profile it, confirm it's genuinely the bottleneck in that
   specific case (unlike the typical AI-request case), and fix it with a
   vectorized approach.

### 18. Mini-Project
Profile `job-processor`'s (Module 6) document-processing task end to end,
identify the actual bottleneck (likely I/O, per the theory above), and
write up a short before/after report with flame graph evidence supporting
any change you made (or evidence that no Python-level change was
warranted, which is itself a valid, documented finding).

### 19. Production Project
**Build:** `perf-report` — a genuinely evidence-based performance audit of
your most complete project so far (`auth-gateway` or `job-processor`):
- A `py-spy` flame graph of a representative request/task, with a written
  interpretation of where time is actually going
- An N+1 query audit of your SQLAlchemy usage, with any found instances
  fixed and measured before/after
- A `locust` load test report showing latency percentiles (p50/p95/p99)
  at increasing concurrency levels, with the point where degradation
  begins clearly identified and explained
- A documented decision log: for each potential optimization you
  considered, state whether you implemented it or explicitly decided
  against it, and why — evidence-based reasoning, not "optimized
  everything I could think of"
- README summarizing findings in a way that would make sense to a
  non-profiling-expert reviewer (a genuinely useful skill — communicating
  performance findings clearly, not just producing raw profiler output)

This kind of evidence-based performance report is exactly the artifact
that distinguishes "I care about performance" from "I can actually
demonstrate and measure performance work" — strong for both interviews and
freelance client trust.

---

### 20–21. Practice & Interview Questions

1. Explain the difference between deterministic and statistical/sampling
   profiling, and when you'd choose each.
2. Where do performance bottlenecks typically live in an AI application,
   and why is "optimize the Python code" usually not the right first
   instinct?
3. Explain the N+1 query problem and how eager loading fixes it, using a
   concrete SQLAlchemy example.
4. Why does load testing reveal problems that single-request latency
   testing can't, and what specific AI-application concurrency bug (Part 0
   Module 9's event-loop-blocking lesson) would only show up under genuine
   concurrent load?
5. Walk through your systematic process for investigating a "this endpoint
   is slow" report, from initial hypothesis to profiling evidence to a
   specific, justified fix.

---

### 22. Common Mistakes
- Optimizing Python code based on intuition without profiling first, often
  optimizing something that wasn't actually the bottleneck.
- Missing N+1 query patterns because they're invisible in single-record
  testing and only show up as query count multiplies with data volume.
- Testing only single-request latency and drawing capacity conclusions
  that don't hold under real concurrent load.
- Treating memory profiling as unnecessary until a long-running process
  has already caused a production incident from a slow leak.
- Presenting "I optimized it" without measurable before/after evidence.

### 23. Debugging Exercise
Given a reported "this endpoint got much slower after we added a feature"
regression, apply the profile-first discipline to distinguish between
several plausible causes (a newly introduced N+1 query, a newly blocking
synchronous call inside an async endpoint per Part 0 Module 9, or
genuinely increased LLM provider latency outside your control) using
flame graphs and query-count logging rather than guessing.

---

### 24. Checklist
- [ ] I profile before optimizing, every time, without exception
- [ ] I can read a flame graph and correctly identify where time is
      actually spent
- [ ] I can spot and fix N+1 query patterns in SQLAlchemy
- [ ] I've load-tested an API and can interpret p50/p95/p99 latency under
      increasing concurrency
- [ ] I've completed `perf-report` with real profiling and load-test
      evidence

### 25. Summary
Profiling replaces intuition with evidence — and in AI applications, the
evidence almost always points to network/model latency and database
query efficiency as the real bottlenecks, not raw Python execution speed.
N+1 queries and event-loop-blocking bugs are the two most common concrete,
fixable culprits once you actually look. `perf-report`'s evidence-based
before/after methodology is a genuinely differentiating habit to carry
into every future project.

### 26. Next Steps
Module 11 (Part 1's final module): **Rate Limiting & API Design** — a
synthesizing capstone module for Part 1, pulling together everything from
Clean Architecture through performance into a cohesive, well-designed
public API, before Part 2 begins teaching AI/ML from first principles.

---