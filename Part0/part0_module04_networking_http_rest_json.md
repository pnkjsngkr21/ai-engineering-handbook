# PART 0 — Prerequisites
## Module 4: Networking, HTTP, REST & JSON

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Explain what actually happens, layer by layer, when your code calls an
  LLM provider's API — DNS, TCP handshake, TLS, HTTP request/response.
- Design and critique REST APIs with the same rigor you already apply in
  Spring Boot, but with an eye toward AI-specific concerns (streaming
  responses, long-running requests, idempotency for retries).
- Handle timeouts, retries, backoff, and connection pooling correctly —
  the single most common source of flaky AI-integration bugs.
- Work fluently with JSON, including the streaming/partial-JSON patterns
  used by LLM token streaming (SSE, chunked transfer encoding).
- Reason about where HTTP/2 and connection reuse matter for latency in
  high-throughput AI backends.

### 2. Prerequisites
Modules 1–3. You already know REST/HTTP from Spring Boot — this module
recalibrates that knowledge toward AI-integration specifics (streaming,
long tail latencies, provider rate limits) rather than re-teaching REST
basics from scratch.

### 3. Estimated Study Time
8–10 hours over 4–5 days.

### 4. Difficulty
⭐⭐☆☆☆ (Easy for the REST/HTTP fundamentals you know; Medium for the
streaming/SSE and retry-semantics parts, which are genuinely different from
typical CRUD API work.)

### 5. Why This Matters
Every single thing you build in Parts 3–8 is fundamentally "make an HTTP
call to a model provider or vector DB, handle the response (possibly
streamed), and don't fall over when it's slow or fails." Model provider APIs
have failure modes and latency characteristics (multi-second responses,
rate limits, partial failures mid-stream) that are meaningfully different
from typical internal microservice calls — this module is where you build
the specific instincts for that.

---

### 6. Theory

**What is it? (the stack, briefly, since you know this)**
```
DNS resolve → TCP handshake (SYN/SYN-ACK/ACK) → TLS handshake (cert
verification, key exchange) → HTTP request sent → HTTP response received →
(connection kept alive or closed)
```
You know this from Java/Spring — the AI-specific wrinkle is that **TLS
handshake cost and connection setup latency become proportionally more
important** because model API calls are so much slower (500ms–30s+) than
typical DB/cache calls (1–20ms) that engineers sometimes get sloppy about
connection reuse, not realizing it still matters at scale (10,000 concurrent
users all opening fresh connections is real overhead even if each individual
call is "slow anyway").

**Why do we need REST specifically (recap + AI angle)?**
REST's resource-oriented, stateless design is why model provider APIs
(Anthropic's `/v1/messages`, OpenAI's `/v1/chat/completions`) are easy to
integrate with any language/tool — no special protocol, just HTTP + JSON.
The interesting design decisions in these APIs (that you should study, not
just consume) are:
- **Idempotency keys** for safe retries of non-idempotent operations (a
  `POST` that might charge you or trigger a side effect if retried blindly).
- **Streaming via Server-Sent Events (SSE)** instead of a single JSON blob —
  because generation is incremental and users shouldn't wait for the full
  response before seeing anything.

**How does streaming actually work? (the part that's new even for you)**

A normal REST response: client sends request, server computes the full
response, sends it back as one chunk, connection (may) closes.

A streaming (SSE) response: the server keeps the HTTP connection open and
sends the response as a sequence of `data: {...}\n\n` events, one per token
(or small batch of tokens), as they're generated:

```
HTTP/1.1 200 OK
Content-Type: text/event-stream

data: {"type":"content_block_delta","delta":{"text":"Hello"}}

data: {"type":"content_block_delta","delta":{"text":" there"}}

data: {"type":"message_stop"}

```
Your client reads this incrementally and can render partial output
immediately — this is *why* ChatGPT/Claude interfaces show text appearing
token by token instead of all at once after a long pause. Understanding
this mechanically now means Part 3 (Streaming) is just "apply this pattern
to a specific SDK" rather than a new concept.

**Retries and backoff — where AI-integration bugs actually live:**
Model APIs rate-limit (`429`) and occasionally have transient failures
(`5xx`). Naive retry logic (`retry immediately, 3 times`) makes rate-limit
storms *worse*. Correct pattern: **exponential backoff with jitter**:
```
wait = min(max_wait, base * 2^attempt) + random_jitter
```
The jitter matters — without it, many clients that all got rate-limited at
the same moment retry in lockstep and hit the rate limit again
simultaneously (a classic distributed-systems "thundering herd," which you
already understand conceptually from backend work — this is the same
problem, new context).

**When should you build a synchronous vs. streaming endpoint?**
Streaming when the user is waiting live (chat UI). Synchronous/batched
when you're doing backend processing (e.g., bulk-summarizing 10,000
documents overnight) — streaming adds complexity with zero UX benefit
there.

---

### 7. Mental Models

**Model 1 — "Streaming is just a REST response that never says 'done'
until it actually says 'done.'"** Mechanically it's the same HTTP
connection, just kept open longer, with the body sent incrementally.

**Model 2 — "Retries without jitter turn one rate-limit event into a
recurring one."** This is the single most common cause of "our AI feature
works fine in testing but falls over under load" bugs.

**Model 3 — "Idempotency keys are optimistic locking for HTTP, not a
database concept."** You know optimistic locking from Spring/JPA — this is
the same idea applied to "don't double-charge/double-process if a client
retries a request whose response it never received."

---

### 8. Visual Explanation (described)

**Diagram: "Synchronous vs. streaming response timeline"**
```
Synchronous:
client --request--> [ server thinks for 8 seconds ] --full response--> client
                     (user sees nothing for 8s, then everything at once)

Streaming (SSE):
client --request--> server
        <--token 1-- (200ms)
        <--token 2-- (250ms)
        <--token 3-- (200ms)
        ... (user sees text appearing continuously)
        <--[done]--
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **MDN's HTTP guide** (developer.mozilla.org/en-US/docs/Web/HTTP) —
   specifically the sections on caching headers, status codes, and
   Server-Sent Events — the best free, precise reference.
2. **Anthropic's own API docs on streaming** (docs.claude.com) — read this
   as a case study of a well-designed streaming API, since you'll be
   building against it directly in Part 3.
3. **Google's "API Design Guide"** — for REST resource modeling conventions
   at production scale.
4. **"Exponential Backoff And Jitter"** — AWS Architecture Blog post — the
   canonical explanation of why naive retries fail and jitter fixes it.

**Official documentation:** MDN (best general HTTP reference), your target
model provider's API reference (Anthropic/OpenAI docs) as the concrete
example you'll actually be integrating against soon.

**GitHub repos:** `anthropics/anthropic-sdk-python` (again) — specifically
look at its retry/backoff and streaming implementation code, now that you
understand the concepts behind it.

---

### 17. Exercises

1. Use `curl -v` against any public HTTPS API and identify, in the verbose
   output, where the TCP handshake, TLS handshake, and HTTP request/response
   each happen.
2. Write a small Python script using `httpx` or `requests` that calls a
   public API and implements exponential backoff with jitter manually (no
   library helper) for `429`/`5xx` responses.
3. Write a script that consumes an SSE stream (you can fake one with a
   simple local server, or use a public SSE demo endpoint) and prints each
   event as it arrives, not after the connection closes.
4. Design (on paper/markdown, not code) a REST endpoint for "submit a
   document for AI summarization" that may take 30+ seconds — compare a
   polling design (`POST` returns a job ID, `GET /jobs/{id}` for status) vs.
   a streaming design, and explain when you'd pick each.

### 18. Mini-Project
**Build:** `apiclient` — a small Python HTTP client wrapper (using `httpx`)
with: connection pooling/reuse, configurable timeouts, exponential backoff
with jitter on `429`/`5xx`, and structured logging of each request/response
(status, latency, retry count). This exact wrapper pattern is what you'll
extend in Part 3 into your first real LLM API client.

### 19. Production Project
**Build:** `streamcat` — a CLI tool that connects to any SSE endpoint (you
can point it at a public SSE demo, or a small FastAPI SSE server you write
yourself) and displays events live in the terminal, with:
- Proper handling of connection drops (auto-reconnect with backoff)
- A `--replay` flag that logs the full stream to a file and can replay it
  later without hitting the network (useful for testing/demos)
- Tests for the parsing logic (SSE frame parsing is fiddly — worth testing
  properly)
- README explaining the SSE protocol for someone who's never seen it

This project directly rehearses the mechanics you'll need when you build a
real streaming chat UI in Part 3/7.

---

### 20–21. Practice & Interview Questions

1. Walk through, layer by layer, what happens between calling
   `requests.get(url)` and receiving a response.
2. Why is naive immediate-retry logic dangerous under rate limiting, and
   what does jitter fix?
3. Explain Server-Sent Events vs. WebSockets — when would you choose one
   over the other for streaming LLM output specifically? (Expected: SSE is
   simpler, HTTP-native, unidirectional — perfect for token streaming;
   WebSockets are bidirectional and heavier, useful if the client also needs
   to send data mid-stream, e.g., interrupting generation.)
4. What's an idempotency key, and why would a payment API or a "trigger an
   expensive AI job" API want one?
5. Given a service making 3 outbound calls to different providers in
   parallel, how would you set per-call timeouts so one slow provider
   doesn't block the whole request? (Ties directly to your existing
   `CompletableFuture` fan-out experience.)

---

### 22. Common Mistakes
- No timeout set at all on HTTP clients (a hung provider takes down your
  whole request pipeline).
- Retrying without backoff/jitter, worsening rate-limit storms.
- Not reusing HTTP connections (creating a new client/connection per
  request), adding needless TLS handshake overhead at scale.
- Treating SSE streams as "just read until the socket closes" without
  handling partial/malformed frames or dropped connections.
- Building a synchronous endpoint for a task that takes 30+ seconds instead
  of a job-queue or streaming design (leads to gateway timeouts).

### 23. Debugging Exercise
You're given a script that occasionally raises a raw connection-reset
exception under load, but works fine one request at a time. Diagnose: no
connection pooling/reuse configured combined with no timeout, so under
concurrent load the client exhausts ephemeral ports / hits provider-side
limits. Fix by properly configuring a connection pool and explicit timeouts.

---

### 24. Checklist
- [ ] I can explain the full request lifecycle (DNS→TCP→TLS→HTTP) unprompted
- [ ] I've implemented backoff-with-jitter myself, without a library helper,
      at least once
- [ ] I understand SSE mechanically, not just "it streams somehow"
- [ ] I've built and tested `apiclient` and `streamcat`
- [ ] I can design idempotent APIs for retry-safe operations

### 25. Summary
This module recalibrated your existing REST/HTTP knowledge toward the
specific failure modes and patterns of AI-provider integration: streaming
responses via SSE, rate-limit-aware retries with jitter, and connection
reuse discipline. These aren't new concepts so much as new emphasis — the
same backend rigor you already apply, pointed at a slower, more
failure-prone class of external dependency.

### 26. Next Steps
Module 5: **Docker & Docker Compose** — packaging everything you build from
here forward so it runs identically on your machine, in CI, and in
production, and the direct on-ramp to Part 6's Kubernetes content.

---
