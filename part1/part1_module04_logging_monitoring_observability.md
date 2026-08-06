# PART 1 — Software Engineering
## Module 4: Logging, Monitoring & Observability

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Implement structured (JSON) logging with correlation IDs across an
  entire request/agent flow, so you can trace one user's request through
  every service and log line it touched.
- Explain the three pillars of observability (logs, metrics, traces)
  precisely, and know which question each one answers.
- Set up OpenTelemetry-based distributed tracing for a FastAPI service,
  including tracing a call out to an LLM provider as its own span.
- Design AI-specific observability: tracking token usage, cost per
  request, latency-to-first-token, and model/provider used — metrics that
  don't exist in typical backend observability but are essential for any
  AI system.
- Set up basic dashboards/alerting thinking (conceptually, tool-agnostic)
  for the metrics that actually matter in an AI application.

### 2. Prerequisites
Modules 1–3, and Part 0 Module 4 (networking/HTTP, for the request-tracing
mental model).

### 3. Estimated Study Time
10–12 hours over 5–6 days.

### 4. Difficulty
⭐⭐⭐☆☆ (Medium — logging you know well from backend work; distributed
tracing depth and AI-specific metrics are the newer territory.)

### 5. Why This Matters
AI systems fail in ways traditional backend systems don't: a "successful"
HTTP 200 response can still contain a hallucinated or low-quality answer,
latency varies wildly per-request (unlike typical CRUD calls), and cost
is a live, per-request concern (unlike most traditional backend calls).
Good observability is how you catch these problems in production instead
of via user complaints — and it's a skill directly relevant to your
platform-engineering goal, where observability of model-serving
infrastructure (Part 6) is a core responsibility.

---

### 6. Theory

**What is it? (the three pillars, precisely)**
- **Logs** — discrete, timestamped events ("what happened, in detail, at
  this specific moment"). Best for debugging a specific incident once
  you've narrowed down where to look.
- **Metrics** — aggregated numerical measurements over time (request
  count, p50/p95/p99 latency, error rate). Best for spotting trends and
  triggering alerts ("error rate just spiked") — cheap to store and query
  at scale, but lose per-request detail.
- **Traces** — the path of a single request as it flows through multiple
  services/functions, broken into timed **spans** (a "span" is one unit of
  work — e.g., "call the LLM provider," "query the vector DB," each with
  start/end time and metadata). Best for answering "why was *this specific*
  request slow, and where exactly did the time go?"

**Why do we need all three (not just logs, which you likely already
default to)?** Logs alone don't scale to answering "what's our overall
error rate this week" (you'd have to grep/aggregate manually — metrics
exist specifically to make this cheap and queryable). Metrics alone can't
tell you *why* a specific slow request was slow (traces exist specifically
for that). Each pillar answers a different class of question — reach for
the right one rather than defaulting to logs for everything.

**Structured logging — the concrete practice:**
```python
import structlog
logger = structlog.get_logger()

logger.info(
    "llm_call_completed",
    request_id=request_id,      # correlation ID, propagated through the whole request
    provider="anthropic",
    model="claude-sonnet-5",
    latency_ms=842,
    input_tokens=512,
    output_tokens=128,
    cost_usd=0.0034,
)
```
Structured (key-value/JSON) logs, unlike free-text `print`/`logging`
messages, are queryable and aggregable (you can ask "what's our average
`latency_ms` grouped by `provider`" directly against your log store) — this
is the practical difference between logs you can *search* and logs you
can actually *analyze*.

**The correlation ID pattern — tying it all together:**
Generate a unique `request_id` (or reuse a distributed tracing `trace_id`)
at the point a request enters your system, and propagate it through every
log line, every downstream service call (as a header), and every span —
this is what lets you later ask "show me every log line and span for this
one user's request" across your entire system, essential for debugging
multi-step agent flows (Part 5) where a single user action triggers many
internal calls.

**Distributed tracing with OpenTelemetry, conceptually:**
```
[ Incoming request, trace_id=abc123 ]
   |
   +-- span: "handle_summarize_request" (parent span)
         |
         +-- span: "check_cache" (50ms)
         +-- span: "call_llm_provider" (2100ms)  <- the actual bottleneck,
         |                                            visible immediately
         |                                            in a trace view
         +-- span: "write_to_db" (30ms)
```
OpenTelemetry is the vendor-neutral standard (works with Jaeger, Grafana
Tempo, Honeycomb, Datadog, etc.) for instrumenting spans like this — you
instrument once, and can send the data to whichever backend you (or a
client, for Part 9 freelancing work) prefers.

**AI-specific observability — the genuinely new material this module
adds:**
Track, for every LLM call, at minimum: **provider, model, input tokens,
output tokens, cost, latency, and latency-to-first-token** (for streaming
responses — the time until the *first* token arrives matters enormously
for perceived responsiveness, separately from total completion time).
Aggregate these into dashboards answering: "what's our daily LLM spend,"
"which model/provider combination has the best cost/latency trade-off for
this feature," "is latency-to-first-token degrading." None of this has a
direct analog in typical backend observability — it's a genuinely new
category of metric you need to build deliberately, since no off-the-shelf
APM tool tracks "tokens" or "cost per LLM call" out of the box without
custom instrumentation (which you'll build in `llm-client-core`'s
`EventBus` from Module 2 — this is exactly what that event bus is *for*).

**When should you alert vs. just log/dashboard?**
Alert on things requiring immediate human action (error rate exceeding a
threshold, cost spending unexpectedly spiking, latency p99 breaching an
SLA). Dashboard (without alerting) on things useful for trend awareness
but not urgent (daily token usage by feature, model usage distribution).
Over-alerting causes alert fatigue — a real, well-documented failure mode
where teams start ignoring alerts entirely because too many are noise —
be deliberate about the alert/dashboard split.

---

### 7. Mental Models

**Model 1 — "Logs, metrics, and traces answer different questions — pick
the right tool, don't force everything through logs."** "What exactly
happened at 3:42pm for this user" → logs. "Is our error rate trending up
this week" → metrics. "Why was *this* request slow" → traces.

**Model 2 — "A correlation ID is the thread that lets you follow one
request's entire story across every log line, span, and service it
touched."** Never skip propagating it, especially in multi-step agent
flows where one user action fans out into many internal calls.

**Model 3 — "Token/cost/latency-to-first-token are first-class AI metrics
that don't exist in generic APM tooling — you must instrument them
yourself, deliberately, from day one."** This is the single biggest gap
between "standard backend observability" and "AI-system observability,"
and it's exactly why `llm-client-core`'s Observer/EventBus (Module 2) was
built the way it was.

---

### 8. Visual Explanation (described)

**Diagram: "One request, three observability views"**
```
LOGS (detail, this moment):
  14:32:01 request_id=abc123 llm_call_started provider=anthropic
  14:32:03 request_id=abc123 llm_call_completed latency_ms=2100 cost_usd=0.003

METRICS (aggregated, over time):
  llm_call_latency_ms{provider="anthropic"} p50=1800 p95=3200 p99=5100
  llm_daily_cost_usd{feature="summarize"} = $42.17 (today, running total)

TRACES (this request's full path):
  handle_request [2400ms total]
    ├─ check_cache [50ms]
    ├─ call_llm_provider [2100ms]  <- bottleneck, immediately visible
    └─ write_to_db [30ms]
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **"Observability Engineering" by Charity Majors, Liz Fong-Jones, George
   Miranda (O'Reilly)** — the definitive modern text on the three pillars
   and why traditional monitoring falls short for complex/distributed
   systems; written by Honeycomb's founders, genuinely excellent.
2. **OpenTelemetry official docs — Python instrumentation guide**
   (opentelemetry.io) — the authoritative reference for instrumenting
   FastAPI with traces.
3. **`structlog` official documentation** — the structured logging library
   for Python you'll actually use.
4. **Google's SRE Book, Chapter 6 ("Monitoring Distributed Systems")**
   (free online, sre.google) — excellent, practical guidance on the
   alert/dashboard distinction and avoiding alert fatigue.

**Official documentation:** opentelemetry.io/docs, `structlog`'s docs.

**GitHub repos:** `open-telemetry/opentelemetry-python` examples
directory; any FastAPI + OpenTelemetry example repo for concrete wiring.

---

### 17. Exercises

1. Add `structlog`-based structured logging to `convo-api`, including a
   correlation ID (generated per request via FastAPI middleware) attached
   to every log line for that request.
2. Instrument `llm-client-core`'s Adapter calls with OpenTelemetry spans,
   and view the resulting trace in a local tool (e.g., Jaeger running via
   Docker Compose, reusing Module 5 skills).
3. Emit a custom metric (via your EventBus listener from Module 2) tracking
   cumulative token usage and cost per provider, and print a running
   summary.
4. Design (on paper) an alerting policy for an AI feature: which
   conditions genuinely warrant a page/alert vs. which belong only on a
   dashboard, and justify each choice.

### 18. Mini-Project
Add a request-ID-propagating middleware to `convo-api`, structured logging
throughout its use cases and adapters, and confirm (by grepping/filtering
your log output) that you can reconstruct one request's full story purely
from its correlation ID.

### 19. Production Project
**Build:** `observability-stack` — extend `local-ai-stack` (Part 0, Module
5) with a full local observability setup:
- OpenTelemetry instrumentation in `convo-api` and `llm-client-core`,
  exporting traces to a local Jaeger instance (added to your Compose stack)
- Structured logging throughout, with correlation IDs propagated via
  middleware and through every downstream call
- A custom metrics listener (via the `EventBus`) tracking, per request:
  provider, model, tokens in/out, cost, latency, and latency-to-first-token
  (simulate the last one with your existing streaming mock from Part 0)
- A simple dashboard (even a basic script printing a summary table, or a
  Grafana dashboard reading from Prometheus if you want to go further) for
  daily cost and latency trends
- A documented alerting policy (README section) distinguishing alert-
  worthy vs. dashboard-only conditions, with justification for each
- A written incident-response walkthrough: given a hypothetical "our AI
  feature's cost tripled overnight" scenario, document exactly which logs,
  metrics, and traces you'd check, in what order, and why

This is a genuinely differentiating portfolio piece — most junior/mid AI
engineers never build real observability; having this, with a documented
incident-response process, is exactly the kind of production maturity that
stands out in both interviews and freelance client conversations.

---

### 20–21. Practice & Interview Questions

1. Explain the three pillars of observability and give a concrete example
   of a question each one is uniquely suited to answer.
2. What's a correlation ID, and why does it become especially important in
   multi-step agent systems (Part 5 preview) versus simple single-call
   APIs?
3. What AI-specific metrics would you track that a generic APM tool
   wouldn't capture out of the box, and why do they matter?
4. Design an alerting policy for cost and latency on an LLM-powered
   feature — what thresholds would you set, and how would you avoid alert
   fatigue?
5. Walk through how you'd debug a specific slow request using traces
   versus how you'd debug a week-over-week latency trend using metrics —
   why are these genuinely different investigative processes?

---

### 22. Common Mistakes
- Relying only on unstructured, free-text logs, making aggregation and
  querying painful or impossible at scale.
- Not propagating a correlation ID, making it impossible to reconstruct a
  single request's story across services/logs/traces.
- Treating "200 OK" as sufficient success signal for an AI feature,
  ignoring output-quality signals entirely (tying back to Module 10's
  evaluation content — observability and evaluation are complementary, not
  substitutes for each other).
- Not tracking cost/token usage at all until a surprise bill arrives.
- Over-alerting on every possible metric, causing alert fatigue and
  eventual alert-ignoring.

### 23. Debugging Exercise
Given a "why did this specific user's request take 8 seconds" question
with only unstructured logs available, contrast how painful that
investigation is versus the same investigation with proper tracing —
implement the missing tracing and confirm the bottleneck (e.g., the LLM
call itself, or an unexpectedly slow cache lookup) becomes immediately
visible.

---

### 24. Checklist
- [ ] I can explain logs vs. metrics vs. traces precisely, with the
      distinct question each answers
- [ ] I propagate a correlation ID through every request in `convo-api`
- [ ] I've instrumented real OpenTelemetry traces and viewed them in
      Jaeger
- [ ] I track AI-specific metrics (tokens, cost, latency-to-first-token)
      via the EventBus
- [ ] I've written a documented incident-response walkthrough for a cost/
      latency scenario

### 25. Summary
Observability's three pillars each answer a different class of question,
and AI systems require an additional, genuinely new category of metrics
(tokens, cost, latency-to-first-token) that no generic APM tool tracks by
default — you must instrument these deliberately via the `EventBus`
pattern from Module 2. `observability-stack`'s combination of tracing,
structured logging, AI-specific metrics, and a documented
incident-response process is a real differentiator for both interviews
and freelance credibility.

### 26. Next Steps
Module 5: **Caching Strategies (Deep Dive)** — building on Part 0 Module
6's caching foundation with production-grade patterns (write-through,
cache stampede protection at scale, semantic caching preview) and tying
caching decisions directly to the cost/latency metrics you can now
actually measure.

---
