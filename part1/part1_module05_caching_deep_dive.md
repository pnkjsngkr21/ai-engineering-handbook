# PART 1 — Software Engineering
## Module 5: Caching Strategies (Deep Dive)

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Choose deliberately among cache-aside, write-through, and write-behind
  strategies based on read/write patterns and consistency requirements.
- Solve the cache stampede problem properly at production scale (not just
  the basic lock-based fix from Part 0).
- Understand and design **semantic caching** — caching LLM responses by
  meaning/similarity rather than exact key match — a genuinely AI-specific
  technique previewed here, built for real in Part 4 once you have
  embeddings.
- Reason about cache invalidation correctly, including the famous "there
  are only two hard things in computer science" caveat, with concrete
  strategies for each invalidation scenario.
- Use your Module 4 observability work to make caching decisions
  data-driven (measured hit rates, cost savings) rather than guesswork.

### 2. Prerequisites
Part 0 Module 6 (SQL/Redis basics), Module 4 (observability, for measuring
cache effectiveness).

### 3. Estimated Study Time
8–10 hours over 4–5 days.

### 4. Difficulty
⭐⭐⭐☆☆ (Medium — cache-aside you know; stampede protection at scale and
semantic caching are the genuinely new depth here.)

### 5. Why This Matters
Caching is one of the highest-leverage cost/latency levers in any AI
system — a well-designed cache can cut LLM spend dramatically for
workloads with repeated or similar queries. This module gives you the
production-grade depth (beyond Part 0's basic Redis cache) that
distinguishes "I added a cache" from "I designed a caching strategy with
measured impact."

---

### 6. Theory

**The three core strategies, precisely:**

**Cache-aside (lazy loading)** — the pattern from Part 0 Module 6: check
cache first, on miss read from source and populate cache. Simple, only
caches what's actually requested, but leaves a window where cache and
source can diverge if the source changes without cache invalidation.

**Write-through** — every write goes to the cache *and* the source
synchronously, keeping them always consistent, at the cost of added write
latency (every write now waits on two systems). Good fit when reads must
never see stale data and write volume is manageable.

**Write-behind (write-back)** — writes go to the cache immediately and are
asynchronously flushed to the source later, minimizing write latency at
the cost of a durability window (data in cache-only state could be lost on
a crash before flush) and eventual, not immediate, consistency with the
source. Rarely worth the complexity/risk for most AI-application use
cases in this handbook — mentioned for completeness and so you recognize
it when you meet it in a system design interview.

**For nearly all AI-caching scenarios in this handbook (caching LLM
responses, caching embeddings), cache-aside is the right default** — you're
caching the *results of expensive computations*, not maintaining a
consistent view of frequently-written data, so write-through/write-behind's
consistency concerns mostly don't apply.

**Cache stampede protection, properly (beyond Part 0's basic lock):**

The problem, precisely: many concurrent requests for the same
(uncached, or just-expired) key all miss simultaneously and all hammer the
expensive backend (an LLM call) at once — Part 0 introduced a short-lived
lock as a basic fix. At production scale, refine this further:

```python
async def get_or_compute(key: str, compute_fn, ttl: int):
    if cached := await redis.get(key):
        return cached
    # Try to acquire a short-lived lock; only ONE concurrent request
    # actually computes, others wait briefly then re-check cache
    lock_acquired = await redis.set(f"lock:{key}", "1", nx=True, ex=10)
    if lock_acquired:
        try:
            result = await compute_fn()
            await redis.set(key, result, ex=ttl)
            return result
        finally:
            await redis.delete(f"lock:{key}")
    else:
        # Someone else is computing it — wait briefly and re-check,
        # rather than also calling the expensive backend
        await asyncio.sleep(0.1)
        return await get_or_compute(key, compute_fn, ttl)  # retry, likely a cache hit now
```
**A further refinement — probabilistic early expiration ("XFetch"):**
instead of waiting for a hard TTL expiry (where *many* requests can all
miss at the exact same moment right at expiry), have a small, randomized
percentage of requests **before** expiry proactively recompute and refresh
the cache slightly early — this smooths out the "everyone misses at
exactly the same second" thundering-herd pattern entirely, at the cost of
some extra (deliberately small, probabilistically rare) recomputation.

**Semantic caching — the genuinely AI-specific technique (previewed here,
built for real in Part 4):**
Exact-key caching (Part 0's approach) only helps for *identical* prompts.
But "summarize this article for me" and "can you summarize this article?"
are different cache keys under exact matching, despite being
near-identical in meaning. **Semantic caching** embeds the incoming
prompt (Part 2/4 will teach you what an embedding actually is), searches a
vector index of previously-cached prompt embeddings for a close-enough
match (above some similarity threshold), and returns the cached response
if a sufficiently similar prior prompt exists — dramatically increasing
cache hit rates for natural-language input where users phrase similar
requests differently. This requires embeddings and vector search (Part
4), so this module only previews the concept precisely enough that Part 4
feels like "applying a technique I already understand the shape of,"
rather than new territory.

**Cache invalidation, properly (beyond "just set a TTL"):**

| Invalidation strategy         | When to use                                            |
|---------------------------------|-----------------------------------------------------------|
| TTL-only                        | Data that's fine being briefly stale (most LLM response caching) |
| Explicit invalidation on write  | Data with a clear "this specific thing changed" event (a user updates their profile → invalidate that user's cached data) |
| Versioned cache keys            | Include a version/hash of upstream config (e.g., prompt template version) in the cache key itself, so a prompt-template change automatically "invalidates" old entries by simply no longer matching any key |

The versioned-key technique is particularly relevant for LLM caching: if
you change your prompt template, you want old cached responses (generated
under the old template) to stop being served — include a hash of the
prompt template (or its version number) as part of your cache key, so a
template change naturally produces new keys instead of serving stale
responses under a new template silently.

---

### 7. Mental Models

**Model 1 — "Cache-aside is the default for caching expensive computation
results (like LLM calls); write-through/write-behind solve a different
problem (keeping frequently-written data consistent) that mostly doesn't
apply here."**

**Model 2 — "Stampede protection isn't just a lock — probabilistic early
refresh smooths out the thundering herd that even a lock-based fix still
concentrates right at the exact expiry moment."**

**Model 3 — "Semantic caching trades exact-match precision for much
higher hit rates on natural-language input, at the cost of needing an
embedding model and a similarity threshold judgment call."** This is a
direct preview of Part 4's core RAG techniques, applied to caching instead
of retrieval.

**Model 4 — "Version your cache keys around anything that changes the
meaning of a cached response (prompt templates, model versions) instead of
relying on manual invalidation you might forget."**

---

### 8. Visual Explanation (described)

**Diagram: "Probabilistic early expiration vs. hard TTL stampede"**
```
Hard TTL (all requests miss at the same instant):
  ...cache hit, hit, hit... [TTL expires at t=60s] ...MISS MISS MISS MISS
  (many simultaneous misses, only lock protects the backend, but many
   requests still wait on the lock)

Probabilistic early refresh (staggered):
  ...hit, hit, [small random chance: proactively refresh early], hit,
  hit, [another small chance: refresh], hit...
  (cache is refreshed gradually before hard expiry, no single moment of
   mass simultaneous misses)
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **"Scaling Memcache at Facebook" (research paper, USENIX NSDI 2013)** —
   the canonical real-world treatment of cache stampede/thundering herd
   problems at genuine scale, still highly relevant.
2. **Redis official docs — "Client-side caching" and "Redis as a cache"**
   pages — for production Redis-specific caching guidance.
3. **"Optimal Probabilistic Cache Stampede Prevention" (the XFetch paper,
   Vattani, Chierichetti, Lowenstein — Google research)** — the source of
   the probabilistic early-refresh technique above.
4. Preview only for now: any introductory blog post on "semantic caching
   for LLMs" (search current results, since this is a fast-evolving,
   recent technique) — don't go deep yet, Part 4 builds the full
   embedding/vector-search foundation first.

**Official documentation:** redis.io/docs (caching-specific sections).

---

### 17. Exercises

1. Implement the basic lock-based stampede protection (from the theory
   section) and demonstrate, under simulated concurrent load, that only
   one request actually computes the expensive value.
2. Implement probabilistic early expiration and, via your Module 4
   observability instrumentation, measure and compare the distribution of
   "cache miss" events over time with vs. without it, under simulated
   concurrent load approaching a shared TTL boundary.
3. Design a versioned cache key scheme for an LLM-caching layer that
   includes the prompt template version and model name — write out the
   key-construction logic and explain what happens (correctly) when you
   change the prompt template.
4. Write a short design doc (on paper) sketching how semantic caching
   would work end to end, including where you'd set the similarity
   threshold and what you'd do about false-positive matches (returning a
   cached answer for a prompt that was actually meaningfully different).

### 18. Mini-Project
Upgrade the caching layer inside `llm-client-core`'s `CachingDecorator`
(Module 2) to include: probabilistic early expiration, versioned cache
keys (incorporating a prompt-template-version parameter), and
Module-4-style metrics tracking hit rate, miss rate, and stampede-lock
contention rate.

### 19. Production Project
**Build:** `smart-cache` — a standalone, well-tested caching library
(structured as its own package, per Module 1/11 conventions) implementing:
- Cache-aside with configurable TTL
- Lock-based stampede protection with a bounded wait/retry
- Probabilistic early expiration (XFetch-style), with a configurable
  refresh probability curve
- Versioned key support (a clean API for including arbitrary
  version/config identifiers in the cache key)
- Metrics emission (hit rate, miss rate, stampede events, early-refresh
  events) via the same `EventBus` pattern from Module 2, so it composes
  cleanly with your existing observability stack
- A load-test script demonstrating stampede protection under simulated
  concurrent traffic, with before/after metrics comparing naive TTL-only
  caching vs. `smart-cache`'s protections
- README documenting the design, explicitly citing the "Scaling Memcache
  at Facebook" and XFetch papers as the basis for the design decisions —
  a nice touch that signals genuine research grounding, not just following
  a tutorial

This becomes the caching library `llm-client-core` depends on going
forward, and directly sets up Part 4's semantic caching work, which
extends this exact library with a similarity-based lookup instead of
exact-key lookup.

---

### 20–21. Practice & Interview Questions

1. Explain cache-aside, write-through, and write-behind, and which is the
   right default for caching expensive LLM call results specifically, and
   why.
2. Describe the cache stampede problem and at least two complementary
   mitigation strategies (lock-based short-circuiting, probabilistic early
   refresh).
3. What is semantic caching, and what trade-off does it make compared to
   exact-key caching? What's the risk of setting the similarity threshold
   too low?
4. Design a cache-key versioning scheme for a system where the underlying
   prompt template changes periodically — how do you ensure old cached
   responses stop being served without ever running an explicit "flush the
   whole cache" operation?
5. How would you measure whether a caching layer is actually paying for
   itself in cost/latency savings, using the observability tooling from
   Module 4?

---

### 22. Common Mistakes
- Using write-through/write-behind for caching computation results (LLM
  responses) where cache-aside is the actually appropriate pattern.
- Only implementing a basic lock for stampede protection and not
  considering that hard TTL expiry still concentrates misses at one
  moment even with a lock in place.
- Manually invalidating caches on every config/prompt change instead of
  using versioned keys, and inevitably forgetting an invalidation at some
  point.
- Setting a semantic cache's similarity threshold without any measurement
  or evaluation, risking returning wrong cached answers for meaningfully
  different prompts.
- Not measuring cache hit rate/cost savings at all, caching "because it
  seems like a good idea" without evidence it's actually helping.

### 23. Debugging Exercise
Given a production incident where cost spiked sharply every hour on the
hour (a classic hard-TTL stampede symptom — many cache entries with the
same round-number TTL all expiring simultaneously), diagnose the pattern
from your Module 4 metrics/traces, and fix it with staggered TTLs (adding
random jitter to TTL values themselves, in addition to or instead of
probabilistic early refresh) so expirations don't cluster.

---

### 24. Checklist
- [ ] I can choose deliberately among cache-aside/write-through/write-
      behind based on the actual access pattern
- [ ] I've implemented and load-tested proper stampede protection
      (lock-based and probabilistic early refresh)
- [ ] I understand semantic caching's trade-offs well enough to design it,
      ready to implement for real in Part 4
- [ ] I use versioned cache keys instead of manual invalidation where
      applicable
- [ ] I've built `smart-cache` with measured, documented stampede
      protection

### 25. Summary
This module went from Part 0's basic cache-aside pattern to
production-grade caching: proper stampede protection (lock-based plus
probabilistic early refresh, grounded in real industry research),
versioned cache keys to solve invalidation cleanly, and a precise preview
of semantic caching — the genuinely AI-specific technique that Part 4 will
build for real once you have embeddings. `smart-cache` becomes a
foundational library the rest of the handbook builds on.

### 26. Next Steps
Module 6: **Background Workers & Queues** — handling long-running AI
tasks (batch embedding generation, multi-step agent workflows, large
document processing) asynchronously, outside the request/response cycle,
using proper job queues.

---

**Reply "continue" for Module 6, or flag anything to go deeper on.**
