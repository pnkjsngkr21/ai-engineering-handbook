# Part 3, Module 9: Latency & Cost Optimization — Streaming, Batching, Model Routing

> Module 8 handled the case where you don't need to call the model at
> all. This module handles the much more common case: you do need a
> fresh generation, so the question becomes how to make that call feel
> fast, cost as little as possible, and use no more model than the
> request actually warrants. Three distinct, complementary levers —
> streaming, batching, and routing — each targeting a different part of
> the cost/latency equation.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Distinguish latency from cost as separate optimization targets that
   sometimes trade against each other, and correctly identify which
   lever (streaming, batching, routing) primarily addresses which target.
2. Explain streaming mechanistically as a direct consequence of the
   autoregressive, token-by-token generation process (Module 1), and
   articulate precisely what streaming does and does not improve (it
   improves perceived/time-to-first-token latency; it does not reduce
   total token cost or total generation time).
3. Design a batching strategy for non-real-time workloads, correctly
   identifying which of your system's call types are actually eligible
   for batching versus which have a hard real-time requirement that
   forecloses it.
4. Design a model-routing policy that sends requests to different-
   capability (and different-cost) models based on estimated task
   difficulty, and build the classification mechanism that makes this
   routing decision reliably.
5. Reason correctly about the accuracy/cost/latency three-way tradeoff
   inherent in routing, and avoid the common failure of routing purely
   on cost without a rigorous accuracy safety net.
6. Extend `llm-client-core` with working streaming support (already
   partially built in Module 1's adapters), a batching path for eligible
   call types, and a routing layer, all measured via `eval-framework` and
   `observability-stack`.

---

## 2. Prerequisites

- **Part 3, Module 1** — `AnthropicAdapter`/`OpenAIAdapter` already
  implement basic streaming; this module completes and formalizes it.
- **Part 3, Module 8** (Caching Strategies) — you need the cost-
  measurement discipline and per-call-type policy thinking from that
  module; routing and batching decisions are made per call type using
  the same discipline.
- **Part 2, Module 9** (Reasoning Models) — model routing decisions
  between standard and reasoning models directly reuse that module's
  cost/accuracy tradeoff framing.
- **Part 1, Module 6** (Background Workers & Queues) — `job-processor`
  is the direct foundation for this module's batching implementation.

---

## 3. Estimated Study Time

**7–9 hours** (2–3 hours theory/reading, 5–6 hours hands-on).

---

## 4. Difficulty

★★★★☆ (4/5)

Streaming and batching are mechanically straightforward extensions of
existing infrastructure. Routing is where the real difficulty is: it
requires building a reliable, evaluated difficulty-classification
mechanism, and getting the accuracy safety net wrong has real,
hard-to-detect production cost.

---

## 5. Why This Matters

At production scale, LLM inference cost and latency are frequently the
single largest driver of both user-facing responsiveness and unit
economics for an AI product. A system that calls the most capable
(and most expensive) model for every request regardless of actual
difficulty is systematically overpaying and, ironically, often *not*
faster, since larger models frequently have higher latency too. This is
precisely the kind of cost-consciousness that separates a working
prototype from a system that can actually run profitably at scale — the
exact distinction interviewers probe for when they ask "how would you
control costs on this AI feature" in a system-design round, and exactly
the operational maturity that platform-engineering teams at AI labs are
responsible for at an infrastructure level.

---

## 6. Theory

### 6.1 Latency vs. cost: distinct targets, sometimes in tension

Before reaching for any specific technique, separate what you're actually
optimizing for, because the right lever differs:

- **Latency** — specifically, *perceived* latency (time until the user
  sees something useful) is what streaming targets. **Total** latency
  (time until the full response is complete) is largely a function of
  output length and model size, and streaming does not reduce it — it
  only changes *when* the user starts seeing output.
- **Cost** — total tokens processed (input + output) across all calls,
  which is what caching (Module 8), batching, and routing all target,
  each via a different mechanism (avoid the call entirely, get a
  discount for deferred processing, or use a cheaper model).

**These can trade against each other**: routing to a smaller, faster,
cheaper model might increase both accuracy risk (Section 6.4) and, in
some cases, actually *not* reduce end-to-end latency if the smaller
model requires more retries due to lower reliability on a hard task.
Every optimization decision in this module needs to specify which
target it's actually improving and what, if anything, it's trading away
— never assume a change is a pure win on both axes without measuring
both.

### 6.2 Streaming: mechanism, not magic

Recall Module 1, Section 6.1: generation is inherently a token-by-token
process, and every token requires its own forward pass, appended to
context for the next token's prediction. **Streaming simply means your
application starts forwarding each generated token to the end user (or
to the next stage of your pipeline) as soon as it's produced by the
model, rather than waiting for the entire response to complete before
returning anything.** This is not a distinct generation mode the model
runs in — it's purely a transport-layer choice about when your
application code surfaces output it's already receiving incrementally
from the provider's API (typically via server-sent events or an
equivalent chunked-response mechanism, connecting directly back to Part
0, Module 4's SSE work in `streamcat`).

**What streaming actually improves**: time-to-first-token — the user
(or downstream system) sees the beginning of a response almost
immediately, which for long responses dramatically improves *perceived*
responsiveness even though the *total* time to finish generating is
unchanged. **What it does not improve**: total token cost (you're still
generating and paying for the same number of output tokens) and total
generation time (the model still has to produce every token in sequence
— streaming doesn't parallelize or speed up the underlying autoregressive
process). **A genuine complication it introduces**: partial output
handling — if you're using Module 2's structured-output/schema
validation on a streamed response, you cannot validate against the
schema until the full structured object has arrived, so streaming and
strict structured-output validation have a real interaction you must
design around explicitly (e.g., stream free-text portions live, but
buffer and validate structured portions before surfacing them).

### 6.3 Batching: trading latency for cost, deliberately

**Batch processing** (offered by both major providers at a meaningfully
reduced per-token cost compared to real-time calls) trades increased
latency (results delivered asynchronously, often within a defined
window rather than immediately) for reduced cost. This is a genuine,
substantial lever — but only for call types that don't have a hard
real-time requirement.

**Correctly identifying batching-eligible call types** is the actual
engineering judgment here, not the batching mechanism itself (which is
just an API call pattern). Ask, for each distinct call type in your
system: does a human or an automated process need this response
*immediately*, or can it tolerate being processed within a defined
window (minutes to hours, depending on the provider's batch SLA)? Bulk
classification/labeling tasks, offline evaluation-set generation
(exactly the kind of thing `eval-framework`, Part 2 Module 8, needs to
do regularly), and any nightly/scheduled processing are strong batching
candidates. A live conversational turn a user is actively waiting on is
not — this is precisely why `job-processor` (Part 1, Module 6) is the
right architectural home for batched LLM calls: it's fundamentally the
same asynchronous, queue-based execution pattern you already built for
non-real-time work, applied to a new kind of expensive, deferrable
operation.

### 6.4 Model routing: matching capability to actual task difficulty

The core idea: not every request needs your most capable (and most
expensive/slowest) model. A simple classification query, a short
factual lookup, or a well-structured extraction task may be handled
just as correctly by a smaller, cheaper, faster model as by your largest
option — while a genuinely complex multi-step reasoning task may
require the larger model to get a correct answer at all. **Routing** is
the mechanism that sends each request to the model tier actually
warranted by its difficulty, rather than defaulting every request to the
most capable (and expensive) tier out of an abundance of caution, or to
the cheapest tier out of cost-consciousness that ignores accuracy risk.

**Building the routing decision itself** requires a difficulty-
classification step — and this is a real engineering problem, not a
one-line heuristic. Options, roughly in increasing sophistication:

- **Rule-based routing** on request characteristics you can determine
  cheaply (query length, presence of certain keywords indicating a
  complex multi-step task, whether tool calling is involved) — cheap,
  fast, but coarse and brittle to novel request shapes, exactly the
  same tradeoff profile as Module 6's rule-based guardrails.
- **A small, fast classifier model** (potentially even a cheap LLM call
  itself, or a lightweight fine-tuned classifier) that estimates task
  difficulty before routing — more accurate than pure rules, at the
  cost of an additional (though typically much cheaper and faster than
  your main call) classification step.
- **Escalation-based routing**: start with the cheaper/faster model, and
  escalate to the more capable model only if the cheaper model's
  response fails a confidence/quality check (potentially reusing
  Module 6's guardrail/validation infrastructure) — this avoids
  needing an upfront difficulty classifier at all, at the cost of
  sometimes paying for two calls (the cheap attempt plus the escalated
  one) on genuinely hard requests.

### 6.5 The routing safety net: never trade away correctness silently

**The single most important discipline in this module**: routing
decisions must be validated against accuracy, not just cost savings.
It's easy to build a routing policy that looks like a clear win in
aggregate cost metrics while silently degrading answer quality on the
subset of requests that were mis-routed to an under-capable model — and
because this failure is distributed across many individual requests
rather than concentrated in an obvious spike, it's exactly the kind of
regression that's easy to miss without deliberate measurement (directly
echoing Module 5's pipeline-evaluation argument: aggregate metrics can
hide real, evidence-requiring failures).

**The required discipline**: use `eval-framework` to measure accuracy
*per routing tier*, not just in aggregate, on a golden set specifically
constructed to include both easy and hard tasks across your actual
request distribution. If your routing policy is escalation-based
(Section 6.4), measure and report the escalation rate itself as a
signal — a rising escalation rate over time may indicate either genuine
task-difficulty drift in your traffic or a routing/classification
regression, and either way it's an operational signal worth tracking,
not discarding. Never deploy a routing policy whose only reported metric
is aggregate cost savings.

---

## 7. Mental Models

1. **"Streaming changes when you see output, not how much it costs or
   how long generation actually takes."** It's a perceived-latency
   optimization on the transport layer, not a generation-speed or
   cost optimization.
2. **"Batching trades latency for cost, deliberately, and only for
   requests that can tolerate the wait."** The engineering judgment is
   in correctly identifying eligible call types, not in the batching
   mechanism itself.
3. **"Routing matches capability to difficulty — measured, not
   assumed."** A routing policy that only reports cost savings, without
   per-tier accuracy measurement, is exactly the kind of unmeasured
   optimization Module 5 and Module 6 already warned against elsewhere
   in the pipeline.
4. **"An unmeasured routing decision can look like a clear win while
   silently costing you correctness."** Aggregate cost savings and
   per-tier accuracy are both required metrics, never just the former.

---

## 8. Visual Explanation

**Diagram 1 — What streaming does and doesn't change**

```
WITHOUT STREAMING:
[.... entire generation happens ....][user sees full response at once]
 ▲                                    ▲
 request sent                        time = T (unchanged by streaming)

WITH STREAMING:
[tok1][tok2][tok3]...........................[tokN]
  ▲     ▲     ▲                                ▲
 user  user  user                          generation
 sees  sees  sees                          still finishes
 tok1  tok2  tok3   ... perceived as fast   at the same T
 almost                  from here on
 immediately

Total generation time (T) — UNCHANGED
Total token cost — UNCHANGED
Perceived responsiveness — DRAMATICALLY IMPROVED
```

**Diagram 2 — Batching eligibility by call type**

```
Call type                          Real-time need?     Batch-eligible?
──────────────────────────────────────────────────────────────────────
Live user conversation turn        YES                  NO
Nightly eval-set scoring           NO                   YES
Bulk document classification       NO                   YES
Interactive tool-calling loop      YES                  NO
Scheduled memory consolidation     NO                   YES
```

**Diagram 3 — Escalation-based routing flow**

```
request ──► [cheap/fast model attempt]
                     │
              [confidence/quality check]
                     │
        ┌────────────┴────────────┐
     PASSES                     FAILS
        │                          │
   return cheap-model         escalate to
   response (cost-              capable model
   optimal path)                     │
                              return capable-model
                              response (accuracy-
                              optimal path, higher
                              cost — paid only when
                              actually needed)

Required measurement: accuracy on EACH path separately,
                       plus overall escalation rate over time
```

---

## 9. Recommended Resources

1. **Anthropic and OpenAI — streaming API documentation**
   (docs.claude.com, platform.openai.com/docs) — the exact mechanics of
   server-sent events/chunked responses for each provider, since your
   adapter's streaming implementation must handle both correctly.
2. **Anthropic and OpenAI — batch API documentation** (docs.claude.com,
   platform.openai.com/docs) — read for the specific cost discount,
   turnaround-time SLA, and request-format requirements, since these are
   the concrete numbers your batching-eligibility decisions should be
   based on, not assumed defaults.
3. **Anthropic or OpenAI model comparison/pricing pages** — read
   directly for current per-model cost and capability tiers, since this
   is exactly the input your routing policy design needs, and it changes
   over time.
4. **Your own `job-processor` codebase (Part 1, Module 6)** — the
   direct architectural foundation for this module's batching
   implementation; review its idempotency and dead-lettering design
   before extending it to LLM batch calls.
5. **Any published engineering writeup on LLM request routing / model
   cascades** (search for recent posts from major AI infra teams or
   labs) — read for real-world routing architecture patterns to compare
   against Section 6.4's escalation-based design.

---

## 10. Exercises

1. **Measure streaming's actual effect.** For one representative call in
   your pipeline, measure time-to-first-token and total-completion-time
   both with and without streaming enabled. Confirm total-completion-time
   is unchanged (within noise) while time-to-first-token drops
   substantially with streaming.
2. **Audit your call types for batching eligibility.** List every
   distinct LLM call type in your `llm-client-core`-based system (live
   conversation turns, memory consolidation, eval-set scoring, etc.) and
   classify each as batch-eligible or not, justifying each classification
   against its actual real-time requirement.
3. **Build a rule-based router, measure it honestly.** Implement a simple
   rule-based router (e.g., route short, simple-looking queries to a
   cheaper model) across a golden set covering both easy and hard tasks.
   Measure aggregate cost savings *and* per-tier accuracy separately —
   confirm whether any accuracy regression occurred on the "easy"-routed
   tier for tasks that were actually harder than the rule assumed.
4. **Build and compare escalation-based routing.** Implement the
   escalation pattern (Section 6.4) for the same golden set. Compare its
   cost/accuracy tradeoff against the rule-based router from Exercise 3,
   and report the escalation rate.
5. **Deliberately induce and catch a routing regression.** Adjust your
   router's difficulty threshold to be more cost-aggressive. Confirm via
   per-tier accuracy measurement (not aggregate cost alone) whether this
   silently degraded correctness on any task category, and quantify it.

---

## 11. Mini-Project: `routing-bench`

A small standalone tool, built against `eval-framework`, that evaluates
a routing policy against a golden set spanning a real difficulty
distribution (easy, medium, hard tasks), reporting: aggregate cost
savings, per-tier accuracy, escalation rate (if applicable), and overall
end-to-end latency — the direct measurement discipline Section 6.5
insists on, built once and reusable for tuning any future routing policy
change.

---

## 12. Production Project: Streaming, Batching, and Routing in `llm-client-core`

### 12.1 What you're building

1. **Complete, robust streaming support**: finalize streaming across both
   `AnthropicAdapter` and `OpenAIAdapter` (building on Module 1's initial
   implementation), with explicit handling of the structured-output
   interaction (Section 6.2) — buffering and validating structured
   portions while streaming free-text portions live where applicable.

2. **A batching path using `job-processor`**: identify at least one real
   batch-eligible call type in your system (per Exercise 2's audit —
   likely `eval-framework`'s golden-set scoring or `LongTermMemoryStore`'s
   periodic consolidation, Module 4) and route it through the
   provider's batch API via `job-processor`'s existing queue/worker
   infrastructure, measuring the real cost reduction achieved.

3. **A routing layer** implementing at least the escalation-based
   pattern (Section 6.4) for one real call type in your pipeline where
   task difficulty genuinely varies, with a confidence/quality check
   gating escalation (potentially reusing Module 6's guardrail
   validation infrastructure for this check).

4. **`routing-bench` evaluation**: measure and report cost savings,
   per-tier accuracy, and escalation rate for the routing layer,
   integrated into pipeline CI (Module 5) so a future threshold change
   is caught if it silently degrades per-tier accuracy.

5. **Observability**: emit metrics via `observability-stack` (Part 1,
   Module 4) for time-to-first-token (streaming), batch job cost/
   turnaround (batching), and per-tier request volume/accuracy/
   escalation rate (routing) — three distinct metric families for three
   distinct levers, not conflated into one generic "optimization"
   dashboard.

### 12.2 What this sets up for later modules

- **Part 3's Hallucination Reduction module** may interact with routing
  decisions (a routed-to-cheaper-model response may need additional
  verification relative to a capable-model response) — this module's
  per-tier accuracy measurement discipline is the direct input.
- **Part 3's capstone** integrates all three levers (streaming, batching,
  routing) alongside caching (Module 8) as the complete cost/latency
  optimization layer of the finished production service.
- **Part 6 (AI Infrastructure)**, Pankaj's platform-engineering track,
  will revisit routing and batching at the infrastructure/serving layer
  (e.g., load balancing across self-hosted model deployments), building
  on the request-level routing judgment established here.

### 12.3 Definition of done

- Streaming works correctly across both adapters, with a verified
  correct interaction with structured-output validation.
- At least one real call type is routed through a working batch path via
  `job-processor`, with measured cost savings.
- The escalation-based routing layer works end-to-end for at least one
  real call type, with `routing-bench` reporting cost savings, per-tier
  accuracy, and escalation rate.
- Pipeline CI would catch a future routing-threshold change that
  degrades per-tier accuracy, not just aggregate cost.
- Distinct metric families for streaming, batching, and routing are all
  visible in `observability-stack`.

---

## 13. Practice & Interview Questions

1. Explain precisely what streaming does and does not improve, and why
   it doesn't reduce total generation cost or total generation time.
2. What real-time requirement determines whether a given LLM call type is
   a good candidate for batch processing? Give two examples from a
   typical production system, one eligible and one not.
3. Describe the escalation-based routing pattern and explain what it
   trades away relative to an upfront difficulty classifier.
4. Why is a routing policy that only reports aggregate cost savings
   insufficient evidence that the policy is a good idea?
5. Streaming and strict structured-output validation (Module 2) have a
   real interaction. Describe the conflict and a concrete way to design
   around it.
6. You're asked to reduce LLM inference cost by 40% for a production
   system. Walk through which of this module's (and Module 8's) levers
   you'd reach for first, and how you'd validate that the reduction
   didn't silently degrade quality.

---

## 14. Common Mistakes

- **Treating streaming as a cost or total-latency optimization.** It
  only improves perceived, time-to-first-token latency — conflating this
  with actual cost or completion-time savings leads to wrong
  expectations and wrong prioritization.
- **Batching real-time-required call types** because the cost savings
  looked attractive, without correctly auditing the actual latency
  tolerance of that call type first.
- **Routing purely on cost, with no per-tier accuracy measurement.**
  This is the single most consequential mistake this module warns
  against — an aggregate cost win can hide a real, distributed accuracy
  loss.
- **Ignoring the structured-output/streaming interaction** and either
  disabling streaming entirely for anything structured, or naively
  streaming partial, unvalidated structured output to a downstream
  consumer that expects a fully validated object.
- **Treating routing thresholds as "set once" rather than an ongoing,
  monitored policy** — traffic difficulty distributions shift over time,
  and an escalation rate or per-tier accuracy that looked fine at launch
  can drift.
- **Building three separate ad hoc optimization efforts** (for
  streaming, batching, routing) instead of a unified measurement
  discipline (`routing-bench`, per-metric-family observability) that
  makes all three auditable the same way.

---

## 15. Debugging Exercise

Your escalation-based router has been running in production for a month.
Aggregate cost is down as expected, and the escalation rate has looked
stable in dashboards. A recent support escalation reveals a cluster of
subtly wrong answers on a specific, moderately complex query category
that the router has been sending to the cheap-tier model without
escalating — the confidence/quality check that should trigger escalation
isn't firing for this category, even though the cheap model's answers
are, in fact, wrong.

Write down at least two concrete hypotheses for why a confidence/quality
check could systematically fail to catch wrong answers in this specific
category (consider: is the quality check itself a guardrail-style
model-based check, per Module 6, and could it share the same
`judge-bias-lab`-style blind spot — e.g., confidently wrong answers that
are also fluently and confidently phrased, fooling the check the same
way it could fool a human skimmer? does your `routing-bench` golden set
actually have coverage for this specific query category, or is this
exactly the kind of coverage gap Module 5's debugging exercise already
illustrated?), and describe how you'd extend `routing-bench`'s golden
set and the escalation check itself to close this gap.

---

## 16. Checklist

- [ ] I can explain precisely what streaming improves and what it does
      not, and why.
- [ ] I can correctly classify a given LLM call type as batch-eligible or
      not, based on its real-time requirement.
- [ ] I can describe the escalation-based routing pattern and its
      tradeoff against upfront difficulty classification.
- [ ] I understand why aggregate cost savings alone is insufficient
      evidence for a routing policy's soundness, and what additional
      metric is required.
- [ ] I can describe the structured-output/streaming interaction and a
      concrete design to handle it.
- [ ] `routing-bench` is built and reports cost savings, per-tier
      accuracy, and escalation rate for a real routing policy.
- [ ] Streaming works correctly across both provider adapters with
      verified structured-output interaction handling.
- [ ] At least one real call type is successfully routed through a batch
      path via `job-processor`, with measured savings.
- [ ] The escalation-based routing layer is implemented and evaluated for
      at least one real call type, with pipeline CI protection against
      future accuracy regressions.
- [ ] Distinct metric families for streaming, batching, and routing are
      visible in `observability-stack`.

---

## 17. Summary

Latency and cost are distinct optimization targets requiring different
levers: streaming improves perceived, time-to-first-token latency by
surfacing tokens as they're generated (a transport-layer choice, not a
change to the underlying autoregressive generation process, cost, or
total completion time); batching trades increased latency for reduced
per-token cost, and is only appropriate for call types without a hard
real-time requirement — correctly auditing which call types qualify is
the real engineering judgment, not the batching mechanism itself; and
model routing matches request difficulty to model capability, saving
cost by not defaulting every request to the most capable tier, but only
soundly if paired with rigorous per-tier accuracy measurement rather than
aggregate cost savings alone — an unmeasured routing policy can look like
a clear win while silently degrading correctness on a distributed subset
of requests. `llm-client-core` now has complete streaming support with
correct structured-output interaction handling, a working batch path via
`job-processor` for eligible call types, and an evaluated,
escalation-based routing layer with `routing-bench` protecting against
silent accuracy regressions — completing, alongside Module 8's caching
work, the full cost/latency optimization toolkit for the pipeline.

---

## 18. Next Steps

**Next: Part 3, Module 10 — Hallucination Reduction Techniques.** Having
just built routing (which introduces a real accuracy-risk dimension when
using cheaper models) and completed the full cost/latency toolkit, this
module addresses the correctness problem underlying much of Part 3's
evaluation and guardrail work directly: why models produce confident,
plausible-sounding but false statements, and the concrete, evidence-based
techniques (grounding, citation requirements, uncertainty expression,
verification passes) available to reduce it — directly building on
Module 5's evaluation infrastructure and Module 6's guardrail
architecture.

Reply "continue" for Module 10, or flag anything to go deeper on.
