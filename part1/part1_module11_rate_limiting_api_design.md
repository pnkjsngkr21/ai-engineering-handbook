# PART 1 — Software Engineering
## Module 11: Rate Limiting & API Design (Part 1 Capstone)

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Design rate limiting algorithms precisely (fixed window, sliding window,
  token bucket, leaky bucket) and choose deliberately among them based on
  traffic shape.
- Design a genuinely well-considered public REST API — resource modeling,
  versioning, pagination, error format consistency — to a standard you'd
  be comfortable shipping to external developers.
- Design API contracts specifically for AI endpoints: streaming vs.
  synchronous response modes, usage/cost metadata in responses, and
  clear error semantics for provider failures vs. your own service's
  failures.
- Synthesize every Part 1 module (architecture, patterns, DI, observability,
  caching, queues, auth, CI/CD, security, performance) into one cohesive,
  portfolio-defining project.

### 2. Prerequisites
All of Part 1 (Modules 1–10) and Part 0 in full — this module is a
deliberate synthesis point.

### 3. Estimated Study Time
12–16 hours over 6–8 days (larger than a typical module, reflecting its
role as Part 1's capstone).

### 4. Difficulty
⭐⭐⭐☆☆ (Medium — individual concepts are familiar; the synthesis into one
cohesive, well-designed API is where the real work and value are.)

### 5. Why This Matters
API design quality is one of the most visible signals of engineering
maturity — both to interviewers (system design questions often are, at
core, API design questions) and to freelance clients (Part 9) evaluating
whether to trust you with their product. Rate limiting specifically is a
direct cost-control mechanism for any AI product, tying back to Module 7's
cost-conscious auth design. This module's production project becomes your
single strongest Part 0/1 portfolio piece.

---

### 6. Theory

**Rate limiting algorithms, precisely:**

**Fixed window** (Part 0 Module 6's basic approach): count requests in
discrete time windows (e.g., per-minute buckets). Simple, but allows a
burst of 2x the limit right at a window boundary (max requests at the end
of window N, immediately followed by max requests at the start of window
N+1).

**Sliding window (log or counter-based):** tracks requests within a
continuously sliding time frame rather than discrete buckets, avoiding the
boundary-burst problem. More accurate, slightly more expensive to compute
(sliding window log stores timestamps; sliding window counter approximates
using weighted counts from the current and previous fixed windows).

**Token bucket:** a bucket holds tokens, refilled at a steady rate up to a
max capacity; each request consumes a token, requests are rejected (or
queued) when the bucket is empty. **Allows controlled bursts** (if the
bucket has accumulated tokens from a quiet period, a burst is allowed up to
the bucket's capacity) while still enforcing a long-term average rate —
this is often the best fit for API rate limiting where occasional bursts
are fine but sustained abuse isn't.

**Leaky bucket:** requests enter a queue (the "bucket") and are processed
("leak out") at a constant rate regardless of arrival burstiness — smooths
output rate perfectly, at the cost of added latency for bursty traffic
(requests wait in the queue rather than being processed immediately or
rejected).

**Choosing among them for AI-specific rate limiting:** token bucket is
usually the right default for LLM-backed API rate limiting — it allows a
user to make a quick burst of related requests (e.g., a multi-turn
conversation happening fast) while still capping sustained abuse/cost
over time, better matching real usage patterns than a rigid fixed window.

**API design — resource modeling and consistency (recap + precision):**
- **Nouns for resources, HTTP methods for actions**
  (`POST /conversations`, `GET /conversations/{id}`, not
  `/getConversation`).
- **Consistent pagination** — cursor-based pagination (an opaque
  `next_cursor` token) generally scales better than offset-based
  (`?offset=1000`) for large, frequently-changing datasets, since
  offset-based pagination can skip or duplicate items if data changes
  between page requests — worth knowing this trade-off precisely rather
  than defaulting to offset pagination out of habit.
- **Consistent error format** — a single, predictable error shape across
  every endpoint (`{"error": {"code": "...", "message": "...", "request_id":
  "..."}}`), including the correlation ID (Module 4) for support/debugging.
- **Versioning** — URL-based (`/v1/...`) is simplest and most visible;
  header-based versioning is more "correct" REST purism but less
  discoverable — URL versioning is the pragmatic default for most
  projects, including this handbook's.

**API design specifically for AI endpoints (the genuinely new content
this module adds):**
- **Explicit response mode** — support both synchronous (`POST
  /summarize`) and streaming (`POST /summarize?stream=true`, or a
  dedicated `/summarize/stream` endpoint using SSE, Part 0 Module 4)
  response modes where relevant, letting clients choose based on their own
  UX needs.
- **Usage/cost metadata in responses** — return token counts and
  cost alongside the actual result (`{"result": "...", "usage": {
  "input_tokens": 512, "output_tokens": 128, "cost_usd": 0.003}}`) —
  this is standard practice among real LLM provider APIs themselves
  (Anthropic/OpenAI both do this) and lets your own API's clients build
  their own cost tracking, exactly as you've been building yours (Module
  4) against the providers you depend on.
- **Distinguishing your service's errors from the underlying provider's
  errors** — if the LLM provider returns a `429` or `503`, decide
  deliberately whether to surface that distinctly to your own API's
  clients (e.g., a `502 Bad Gateway` indicating "our upstream dependency
  failed," distinct from your own service's `500` for a genuine bug in
  your code) — this distinction directly aids debugging for your API's
  own consumers, mirroring the clarity Module 4's observability work
  gives you internally.

---

### 7. Mental Models

**Model 1 — "Token bucket is usually the right default for LLM-API rate
limiting — it matches how users actually burst (a quick back-and-forth
conversation) while still capping sustained cost."**

**Model 2 — "Cursor-based pagination is the safer default for data that
changes — offset-based pagination silently corrupts results under
concurrent writes in a way that's easy to miss in testing but real in
production."**

**Model 3 — "Return usage/cost metadata in your own API responses,
exactly like the provider APIs you depend on do for you — it's the same
transparency principle, one layer up."**

**Model 4 — "Distinguish 'our upstream provider failed' from 'we have a
genuine bug' in your error responses — this single distinction saves your
own API's consumers (and your future self, debugging) significant
confusion."**

---

### 8. Visual Explanation (described)

**Diagram: "Token bucket rate limiting"**
```
Bucket capacity: 10 tokens, refill rate: 1 token/second

t=0:  [██████████] 10 tokens — burst of 10 requests allowed instantly
t=0:  [          ] 0 tokens after the burst
t=5:  [█████     ] 5 tokens refilled after 5 seconds — smaller burst allowed
t=10: [██████████] back to full capacity — sustained rate stays bounded
      to 1 request/second on average, but bursts are gracefully accommodated
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **Stripe's API design guide / blog posts on API design** — a
   widely-referenced, genuinely excellent real-world example of
   versioning, pagination, error format, and idempotency done well by a
   company operating at serious scale.
2. **Anthropic's and OpenAI's own API reference documentation** — study
   their response shapes (usage/cost metadata, error formats, streaming
   design) directly as concrete, current, real-world examples of exactly
   the AI-specific API design patterns this module covers.
3. **"API Design Patterns" by JJ Geewax (Manning)** — a thorough,
   practical reference covering pagination, versioning, and resource
   modeling in real depth.
4. **Cloudflare's blog — "How we built rate limiting"** (or similar
   engineering blog posts from companies operating rate limiters at
   scale) — for a grounded, real-world token-bucket implementation
   discussion.

**Official documentation:** your target model provider's (Anthropic/
OpenAI) API reference, studied specifically for response shape and error
design.

**GitHub repos:** any well-regarded API design style guide repo (e.g.,
Google's API design guide on GitHub, `googleapis/api-guide`) for a
comprehensive reference beyond what fits in this module.

---

### 17. Exercises

1. Implement token bucket rate limiting in Redis (using a Lua script for
   atomicity, or Redis's own rate-limiting patterns), and compare its
   behavior under bursty simulated traffic against your Part 0 Module 6
   fixed-window implementation.
2. Design (write out fully) a consistent error response format for an
   API, including how you'd distinguish upstream-provider failures from
   your own service's bugs, and retrofit it across `auth-gateway`'s
   existing endpoints.
3. Convert an offset-based paginated endpoint to cursor-based pagination,
   and write a test demonstrating the specific failure mode
   (skipped/duplicated items under concurrent writes) that cursor-based
   pagination avoids.
4. Design a response schema for a "generate text" endpoint that includes
   usage/cost metadata, matching the style of a real provider API you've
   studied.

### 18. Mini-Project
Retrofit `auth-gateway` with: token-bucket rate limiting (replacing the
simpler Part 0 Module 6 fixed-window limiter), a consistent error response
format across all endpoints (including correlation IDs), and cursor-based
pagination for any list endpoint.

### 19. Production Project — Part 1 Capstone
**Build:** `ai-api-platform` — the definitive synthesis of everything in
Part 0 and Part 1, structured as the single strongest portfolio piece so
far:
- Clean Architecture layering (Module 1) with Protocol-based abstractions
  at every genuine integration boundary
- Design patterns (Module 2) used deliberately, not gratuitously —
  Adapter for provider abstraction, Decorator for caching/logging/retry
  composition, Factory for provider selection
- A disciplined composition root (Module 3) using `lifespan` correctly
- Full observability (Module 4): structured logs with correlation IDs,
  OpenTelemetry tracing, AI-specific metrics (tokens, cost, latency)
- Production-grade caching (Module 5): cache-aside with stampede
  protection and versioned keys
- Background job processing (Module 6) for any long-running endpoint, with
  idempotent handlers and dead-lettering
- Full auth (Module 7): JWT for users, API keys for programmatic access,
  both with proper rate limiting
- Complete CI/CD (Module 8): tiered fast/slow pipelines, branch
  protection, Docker builds tagged by commit SHA
- A documented security review (Module 9) against the OWASP checklists,
  plus a prompt-injection risk-assessment section
- A performance report (Module 10) with real profiling and load-test
  evidence
- Token-bucket rate limiting, consistent error format with correlation
  IDs, cursor-based pagination, and AI-specific response metadata (this
  module's content)
- A comprehensive README/architecture document tying every decision back
  to the module that taught it, written as if this were a real design
  review document for a company — genuinely the best single artifact to
  link from your resume/portfolio at this stage of the handbook

This project has no real LLM integration yet (that's Part 3) — its
endpoints can use mocked/stub "generation" logic — but its **architecture,
operational maturity, and engineering discipline** should already be
indistinguishable from a production system, which is exactly the point:
by the time you add real AI functionality in Part 3, the hard software
engineering work will already be done and battle-tested.

---

### 20–21. Practice & Interview Questions

1. Compare fixed window, sliding window, token bucket, and leaky bucket
   rate limiting — which would you choose for an LLM-backed API and why?
2. Design a consistent error response format for a public API, including
   how you'd distinguish your own service's errors from an upstream
   provider's errors.
3. Explain why cursor-based pagination is generally preferred over
   offset-based pagination for large, frequently-changing datasets.
4. What metadata would you include in an AI-generation API's response
   beyond just the generated content, and why does each piece matter to
   the API's consumers?
5. Walk through, at a high level, how every module in Part 1 contributed
   to one cohesive production system — this is effectively "explain your
   own architecture," excellent interview practice for discussing a real
   project in depth.

---

### 22. Common Mistakes
- Using fixed-window rate limiting for bursty, conversational traffic
  patterns where token bucket would better match real usage.
- Inconsistent error formats across different endpoints, making client-
  side error handling fragile and support/debugging harder.
- Defaulting to offset-based pagination without considering its failure
  mode under concurrent writes.
- Omitting usage/cost metadata from AI-generation API responses, forcing
  clients to guess at cost rather than being told directly.
- Building `ai-api-platform` as disconnected modules rather than a
  genuinely cohesive, consistently-designed system — the value of this
  capstone is the *coherence*, not just the presence of each individual
  piece.

### 23. Debugging Exercise
Given a report that an API client is intermittently seeing duplicate items
across paginated requests during periods of high write activity, diagnose
that offset-based pagination is the cause (items shifting position between
page requests as new rows are inserted), and migrate the affected endpoint
to cursor-based pagination.

---

### 24. Checklist
- [ ] I can choose deliberately among rate-limiting algorithms based on
      traffic shape, not by default habit
- [ ] I've designed and implemented a consistent, correlation-ID-including
      error format across an entire API
- [ ] I understand cursor vs. offset pagination trade-offs precisely
- [ ] I include usage/cost metadata in AI-generation-style API responses
- [ ] `ai-api-platform` is complete, cohesive, and genuinely
      production-grade in architecture and operational maturity

### 25. Summary — Part 1 Complete
**Congratulations — Part 1 (Software Engineering) is complete.** You've
built `ai-api-platform`: a fully architected, observable, cached, secured,
tested, CI/CD-automated, well-designed API platform — with no real AI
functionality yet, by design. Every module in Part 1 contributed a
specific, now-battle-tested piece of production engineering discipline
that Part 2 onward will build real AI capability directly on top of,
without needing to revisit software-engineering fundamentals again.

### 26. Next Steps
**Part 2 — AI Foundations** begins next, teaching neural networks,
embeddings, transformers, attention, and the rest of the ML foundations
from **zero prior ML assumption**, as the handbook's teaching philosophy
requires — but now with a huge advantage: every AI system you build from
here forward has `ai-api-platform`'s production discipline as its
foundation, so Part 2-8 can focus entirely on the AI-specific concepts
themselves.

---
