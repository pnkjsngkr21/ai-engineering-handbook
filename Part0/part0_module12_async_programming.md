# PART 0 — Prerequisites
## Module 12: Async Programming (Python & JS)

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Explain Python's `asyncio` event loop model precisely, including how it
  differs from both Java's thread-based concurrency and JavaScript's event
  loop (Module 7).
- Write correct, efficient concurrent code for fan-out patterns (calling
  multiple LLM APIs or services in parallel) using `asyncio.gather`,
  directly mirroring the `CompletableFuture.allOf` patterns you've been
  interview-prepping.
- Diagnose and fix the most common async bugs: accidentally blocking the
  event loop, forgetting to await a coroutine, and unbounded concurrency
  overwhelming a downstream service.
- Apply backpressure/concurrency-limiting patterns (semaphores, bounded
  task groups) to avoid overwhelming rate-limited APIs — directly relevant
  to every LLM-provider integration you'll build.
- Compare Python's `asyncio` and JS's event loop side by side confidently,
  since you'll move between both constantly in this handbook.

### 2. Prerequisites
Modules 1, 7, 9 (Python, JS async, FastAPI). This module is the deep-dive
that Modules 7 and 9 previewed.

### 3. Estimated Study Time
10–12 hours over 5–6 days.

### 4. Difficulty
⭐⭐⭐☆☆ (Medium — the concepts are approachable, but correctness under
concurrency is genuinely tricky and worth real practice, not just reading.)

### 5. Why This Matters
AI systems are overwhelmingly **I/O-bound**: waiting on model API calls,
vector DB queries, and external services. Nearly every latency and cost win
in Parts 3–8 comes from correct concurrency (parallel fan-out instead of
sequential waiting) rather than algorithmic cleverness. This is also
directly interview-relevant given your existing fan-out/`CompletableFuture`
prep — this module gives you the Python-native version of exactly that
skill, plus the JS side for frontend work.

---

### 6. Theory

**What is it? (Python's asyncio, precisely)**
`asyncio` is Python's single-threaded, event-loop-based concurrency model
— structurally identical in spirit to JavaScript's event loop (Module 7):
one thread, cooperative multitasking, coroutines that yield control at
`await` points instead of blocking.

```python
async def fetch_one(client, url):
    response = await client.get(url)   # yields control here while waiting
    return response.json()

async def fetch_all(urls):
    async with httpx.AsyncClient() as client:
        tasks = [fetch_one(client, url) for url in urls]
        return await asyncio.gather(*tasks)   # runs all concurrently
```

**Direct comparison to your `CompletableFuture` fan-out pattern:**

| Java (`CompletableFuture`)                          | Python (`asyncio`)                        |
|-------------------------------------------------------|----------------------------------------------|
| `CompletableFuture.supplyAsync(() -> call())`         | `async def call(): ...` (a coroutine)         |
| Runs on a **real thread** from a thread pool           | Runs on the **same single thread**, cooperatively yielding |
| `CompletableFuture.allOf(f1, f2, f3).join()`          | `await asyncio.gather(t1, t2, t3)`            |
| Genuine parallelism (multi-core)                       | Concurrency, NOT parallelism (single thread) — but for I/O-bound work, this doesn't matter, since the "waiting" happens off-thread anyway (in the OS/network layer) |
| Backpressure via bounded thread pool / `Semaphore`     | Backpressure via `asyncio.Semaphore`          |

**The most important distinction to internalize:** Java's
`CompletableFuture` gives you **real parallelism** — multiple threads
genuinely executing simultaneously on different cores. Python's `asyncio`
gives you **concurrency without parallelism** — one thread, switching
between tasks at `await` points. For **I/O-bound** work (waiting on network
calls, which is nearly everything in AI systems), this distinction barely
matters in practice — the "waiting" happens in the OS/network stack, not
consuming CPU either way, so single-threaded cooperative concurrency
achieves effectively the same wall-clock speedup for I/O-bound fan-out. For
**CPU-bound** work (e.g., local embedding computation, image processing),
asyncio gives you **zero speedup** — you need `multiprocessing` (real OS
processes, real parallelism) instead, since asyncio's single thread can't
parallelize CPU work at all.

**Backpressure — why unbounded fan-out is dangerous:**
```python
# DANGEROUS: launches ALL requests at once, can overwhelm the provider,
# hit rate limits instantly, or exhaust local resources (file descriptors,
# memory) if the list is large
await asyncio.gather(*[call_llm(p) for p in thousand_prompts])

# CORRECT: bounded concurrency via a semaphore
sem = asyncio.Semaphore(10)   # at most 10 concurrent calls
async def bounded_call(prompt):
    async with sem:
        return await call_llm(prompt)
await asyncio.gather(*[bounded_call(p) for p in thousand_prompts])
```
This is the single most common real-world async bug in AI integrations:
naive `gather` over a large input list, immediately triggering `429` rate
limit storms (tying directly back to Module 4's backoff/jitter content —
bounded concurrency and backoff-with-jitter are complementary, not
alternatives; you generally want both).

**Common async bugs, precisely:**
- **Forgetting to `await` a coroutine** — in Python, calling an `async def`
  function without `await` doesn't run it; it just creates a coroutine
  object that silently does nothing (Python usually warns about this, but
  it's easy to miss) — very different from Java, where forgetting to
  handle a `CompletableFuture` at least still runs the async work, just
  possibly unobserved.
- **Blocking the event loop** — exactly the FastAPI trap from Module 9,
  generalized: any blocking call inside an async function stalls every
  other concurrent coroutine on that same event loop.
- **Mixing sync and async incorrectly** — calling `asyncio.run()` from
  inside code that's already running inside an event loop raises an error;
  understand the "one event loop per thread, entered once" model.

---

### 7. Mental Models

**Model 1 — "Asyncio concurrency ≈ Java's non-blocking I/O (like Netty/
WebFlux), not Java's thread-pool-based `CompletableFuture` parallelism."**
For I/O-bound fan-out, both approaches achieve similar wall-clock
improvements — they're solving the same problem with different
mechanisms (many OS threads waiting vs. one thread cooperatively
multiplexing waits).

**Model 2 — "A semaphore in asyncio is a concurrency *limiter*, not a
lock."** It's the direct Python-async equivalent of bounding a thread
pool's size in Java — controlling *how many* things run concurrently, not
mutual exclusion of a single resource.

**Model 3 — "CPU-bound work needs multiprocessing; I/O-bound work needs
asyncio; never confuse which one your bottleneck actually is."** Profile
first (Module 1's lesson) before choosing a concurrency strategy.

---

### 8. Visual Explanation (described)

**Diagram: "Unbounded vs. bounded fan-out"**
```
Unbounded (1000 prompts, gather all at once):
[========================================] 1000 concurrent requests fired
   -> provider immediately returns 429s for most of them

Bounded (semaphore = 10):
[==========] 10 concurrent -> as each finishes, next one starts
   -> stays within rate limits, steady throughput, no wasted retries
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **Python official docs — `asyncio` "Coroutines and Tasks"** page —
   precise, authoritative, and shorter than most tutorials attempt to make
   it seem.
2. **"Async IO in Python: A Complete Walkthrough" — Real Python** — a
   thorough, well-regarded practical guide beyond the bare docs.
3. **`httpx` docs — "Async Support"** — since this is the async HTTP
   client you've already been using since Module 4.
4. **MDN/JS event loop material from Module 7** — revisit now, side by
   side with `asyncio`, to solidify the comparison.

**Official documentation:** docs.python.org/3/library/asyncio.html.

**GitHub repos:** look at how `anthropic-sdk-python` and `openai-python`
implement their async client variants — both offer sync and async client
classes side by side, a good real-world example of supporting both
paradigms cleanly.

---

### 17. Exercises

1. Write an `asyncio.gather`-based fan-out over 5 mock "API calls"
   (`asyncio.sleep` standing in for network latency), and time it against
   a sequential-`await`-in-a-loop version — confirm the wall-clock
   difference matches your mental model.
2. Add an `asyncio.Semaphore(2)` to the above and observe (via timestamped
   logging) that only 2 run concurrently at any moment, even though 5 were
   scheduled.
3. Deliberately forget an `await` in front of an async function call,
   observe Python's warning (`RuntimeWarning: coroutine was never
   awaited`), and explain why this bug is easy to introduce accidentally.
4. Combine bounded concurrency (semaphore) with exponential backoff +
   jitter (Module 4) in one function that fans out over many mock "LLM
   calls," some of which simulate `429` responses — confirm the combined
   strategy handles both concerns correctly without them interfering with
   each other.

### 18. Mini-Project
**Build:** `fanout-bench` — a small script that fans out over N mock async
API calls (configurable N and per-call latency), with configurable
concurrency limits, and reports total wall-clock time for different
concurrency settings (1, 5, 10, unbounded) — a hands-on demonstration of
exactly how concurrency limits trade off throughput vs. burst load on a
downstream dependency.

### 19. Production Project
**Build:** `batch-caller` — a production-quality async batch-processing
utility, designed to be genuinely reusable starting in Part 3 for calling
LLM APIs over large input sets:
- Takes a list of inputs and an async function to apply to each
- Bounded concurrency via semaphore (configurable)
- Exponential backoff with jitter on transient failures (reusing Module 4)
- Per-item error isolation — one item's permanent failure doesn't crash
  the whole batch; failures are collected and reported alongside successes
- Progress reporting (e.g., a simple counter of completed/failed/remaining)
- Full test suite, including tests that verify concurrency is actually
  bounded (timing-based assertions) and that backoff/jitter behaves
  correctly on simulated transient failures
- README explaining the design and explicitly connecting it back to the
  `CompletableFuture` fan-out pattern you already know from Java

This is the exact utility you'll use in Part 3 (LLM Engineering) and Part 4
(RAG, e.g., batch-embedding many documents) — building it properly now
saves significant time later.

---

### 20–21. Practice & Interview Questions

1. Explain the difference between concurrency and parallelism, and where
   `asyncio` sits on that distinction versus `CompletableFuture`/threads.
2. Why does unbounded `asyncio.gather` over a large list of API calls risk
   overwhelming a rate-limited provider, and how does a semaphore fix
   this?
3. When would `asyncio` give you zero speedup, and what would you use
   instead?
4. What happens if you call an `async def` function without `await`, and
   why is this an easy mistake to make?
5. Design a function that batch-processes 10,000 items against a
   rate-limited external API, respecting both a concurrency limit and
   exponential backoff on transient errors — walk through your design out
   loud.

---

### 22. Common Mistakes
- Unbounded fan-out over large input lists, immediately triggering rate
  limits.
- Forgetting `await`, silently not running async work.
- Blocking the event loop with a synchronous call inside async code
  (recurring theme from Module 9 — worth over-learning).
- Using `asyncio` for CPU-bound work expecting a speedup that structurally
  cannot happen on a single thread.
- Not isolating per-item failures in batch processing, letting one bad
  input crash an entire large batch job.

### 23. Debugging Exercise
Given a batch job that intermittently hangs forever on large inputs,
diagnose that unbounded concurrency exhausted a connection pool or hit a
provider-side connection limit with no timeout set, causing some tasks to
wait indefinitely — fix with a semaphore-bounded concurrency limit plus
explicit per-call timeouts (Module 4).

---

### 24. Checklist
- [ ] I can explain concurrency vs. parallelism precisely, with the
      Java/Python comparison
- [ ] I default to bounded concurrency (semaphore) for any fan-out over an
      unbounded/large input list
- [ ] I've combined bounded concurrency with backoff+jitter successfully
- [ ] I've built `batch-caller` with proper tests, including timing-based
      concurrency assertions
- [ ] I know precisely when asyncio helps (I/O-bound) and when it doesn't
      (CPU-bound)

### 25. Summary
Python's `asyncio` gives you the same fan-out speedups you already know
from `CompletableFuture`, via a different mechanism (single-threaded
cooperative concurrency instead of a real thread pool) — a distinction that
matters for CPU-bound work but rarely for the I/O-bound work that dominates
AI systems. Bounded concurrency (semaphores) combined with backoff+jitter
(Module 4) is the production-grade pattern for any fan-out against
rate-limited external APIs — exactly what `batch-caller` implements, ready
to reuse starting in Part 3.

### 26. Next Steps
Module 13: **System Design Fundamentals** — connecting everything from
Part 0 so far into the vocabulary and patterns (load balancing, caching
layers, message queues, CAP theorem trade-offs) you'll need for both your
DoorDash-style interviews and for designing the AI system architectures in
Parts 3–8.

---

**Reply "continue" for Module 13, or flag anything to go deeper on.**
