# Part 6, Module 3: Ollama & Local Development Serving

> The deliberately lighter-weight counterpart to Module 2: when
> production-grade continuous batching is the wrong tool — local
> development, quick experimentation, low-concurrency personal use —
> what the correctly-scoped alternative looks like, and how
> `llm-client-core`'s adapter pattern accommodates both without forcing
> a single one-size-fits-all serving choice.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. State precisely why Ollama and vLLM are not competing solutions to
   the same problem, but solutions scoped to different concurrency and
   operational-complexity regimes, using Module 2, Section 6.4's
   framework.
2. Explain Ollama's model-management and quantized-format defaults
   (GGUF) as a deliberate trade-off — ease of use and low-VRAM
   footprint over the raw throughput ceiling vLLM optimizes for.
3. Run a local model behind Ollama, integrate it as a second adapter
   in `llm-client-core`, and use it as the default local-development
   backend for every agent and RAG component built in Parts 3–5,
   without needing cloud GPU spend during iteration.
4. Design a clear decision rule — not a vague "it depends" — for when
   a project should move from Ollama-based local development to a real
   vLLM (or other production-grade) deployment.
5. Recognize and avoid the inverse mistake of Module 2: over-relying on
   Ollama for a workload that has actually outgrown it, silently
   degrading production latency/throughput while looking "fine" in
   local testing.

## 2. Prerequisites

- Part 6, Modules 1–2 — this module is meaningless without Module 2's
  contrast; you need to know what continuous batching and PagedAttention
  cost in complexity before appreciating why skipping them is sometimes
  correct.
- Part 0, Module 5 (Docker) — Ollama's local model management has a
  similar "pull an image, run it" mental model worth connecting
  explicitly.
- Part 3, Module 1 (`llm-client-core` adapters) — this module adds a
  second adapter to the same package from Module 2.

## 3. Estimated Study Time

6–8 hours across 1–2 sessions — deliberately the lightest module in
Part 6 so far, matching the lighter-weight tool it covers.

## 4. Difficulty

★★☆☆☆ (2/5) — Genuinely simpler than Module 2; the main intellectual
work is correctly reasoning about *when* this simplicity is the right
choice versus when it's a trap, not the mechanics of running Ollama
itself.

---

## 5. Why This Matters

It would be easy to read Module 2 and conclude that vLLM's
continuous-batching machinery is simply "the correct way to do
self-hosted serving" and that anything simpler is a lesser, temporary
stand-in. That conclusion is wrong, and this module exists specifically
to correct it: for the overwhelming majority of your day-to-day
development work across this entire handbook — testing a RAG pipeline
change (Part 4), iterating on an agent's action space (Part 5),
debugging a prompt (Part 3) — you have exactly one active request at a
time, running locally, and vLLM's entire value proposition (efficient
scheduling across many *concurrent* requests) simply does not apply.
Reaching for it anyway means paying real setup and operational
complexity for zero actual benefit, which is precisely the kind of
mismatched-tool-to-problem mistake that a good infrastructure engineer
is distinguished by *not* making.

This also matters financially and practically for your own learning
process specifically: every module in Parts 3–5 that calls an LLM can,
from this module forward, run against a local Ollama-served model
during development and testing — meaning you can iterate on
`agent-core`'s loop, `rag-engine`'s retrieval, or any eval bench without
burning hosted-API cost or needing a cloud GPU reservation for routine
iteration. That's a direct, concrete productivity and cost benefit to
your own ongoing bootcamp work, not just an abstract lesson.

---

## 6. Theory

### 6.1 Ollama and vLLM solve different problems, not the same problem twice

Restating Module 2, Section 6.4's framework directly: continuous
batching and PagedAttention exist to maximize throughput and memory
efficiency under **concurrent, production-scale traffic**. Ollama is
designed for the opposite regime: a single user (or a handful),
running models locally, prioritizing ease of setup, low resource
footprint, and quick iteration over raw serving throughput. Asking "is
Ollama as fast as vLLM" is close to a category error — under real
concurrent load, vLLM will outperform Ollama by design, because that's
specifically the problem vLLM's scheduler solves and Ollama's does not
attempt to. Under single-request local development load, the
difference in practice is often small or irrelevant, because there's no
concurrent traffic for continuous batching to help with in the first
place — Module 2, Section 6.4's "little to no benefit" case is exactly
Ollama's home turf.

### 6.2 GGUF and quantization as a deliberate default, not a limitation

Ollama defaults to serving models in the GGUF format, typically
quantized (a preview here; full treatment in Module 4). This is a
deliberate design choice matched to its target use case: a quantized
model has a much smaller VRAM/RAM footprint, making it possible to run
meaningfully-sized models on a laptop or a modest single GPU that
couldn't hold the full-precision weights (Module 1's VRAM arithmetic
applies directly — quantization is one of the direct techniques Section
6.1 of Module 1 predicted would help the bandwidth-bound bottleneck,
now showing up as a practical default rather than an optional
optimization). The trade-off (some accuracy loss, covered rigorously in
Module 4) is usually an entirely acceptable price for local
development and testing, where you're validating logic and behavior,
not benchmarking production-grade output quality — a RAG pipeline's
retrieval logic or an agent's loop-termination guards can be fully
tested against a quantized local model; only your final production
quality bar needs the full-precision (or carefully-chosen quantization
level) model from Module 2's deployment.

### 6.3 The decision rule: when to graduate from Ollama to production serving

State this as an explicit rule, not a feeling:

- **Stay on Ollama** while: you are developing or testing logic (not
  benchmarking final production quality or throughput), concurrent
  request volume is at most a handful, and latency/throughput at scale
  is not yet the question being asked.
- **Move to vLLM (or an equivalent production framework)** when any of
  the following becomes true: you need to serve genuinely concurrent
  production traffic (Module 2's whole value proposition activates),
  you need to benchmark real production-grade throughput/cost numbers
  (using `gpu-capacity-planner` and `batching-benchmark` from Modules 1–2
  to make that decision with real numbers, not a guess), or your
  quality bar requires a precision/quantization level beyond what your
  local Ollama setup is running.

The mistake this rule exists to prevent is symmetrical to Module 2's
common mistake: just as it's wrong to over-apply vLLM's machinery where
it isn't needed, it's equally wrong to keep a production or
near-production workload running on a local-development-scoped tool
past the point where its concurrency or quality assumptions have been
outgrown, because the failure there is silent — everything looks fine
in low-traffic testing, and the gap only appears under real load,
often first noticed by users rather than by your own monitoring.

### 6.4 What stays constant regardless of which tool you use

Because both Module 2's `VLLMAdapter` and this module's `OllamaAdapter`
implement the same `llm-client-core` interface (Part 3, Module 1's
Adapter pattern), everything built on top of `llm-client-core` in Parts
3–5 — every prompt template, every tool-calling loop, every agent
action, every RAG retrieval call — is completely unaffected by which
backend is actually serving requests. This is the concrete payoff of
having insisted on that pattern from the very first module that used an
LLM: the choice between local-development and production serving
becomes a configuration decision, not a code change, and switching
between them to test the "graduate to production" decision from Section
6.3 costs you nothing beyond changing which adapter is instantiated.

---

## 7. Mental Models

1. **"Asking whether Ollama is 'as fast as' vLLM under concurrent load is
   close to a category error — they're scoped to different traffic
   regimes."**
2. **"Quantized-by-default is the right trade-off for validating logic,
   not for benchmarking production quality — know which one you're
   doing."**
3. **"Graduate to production serving when concurrency or quality
   requirements demand it — not when the tool starts to feel
   'unserious.'"**
4. **"If switching your backend costs more than a config change, your
   adapter pattern wasn't built correctly — this module is the
   proof."**

---

## 8. Visual Explanation

```
        LOCAL DEVELOPMENT                 PRODUCTION SERVING
        (this module)                     (Module 2)

   ┌─────────────────────┐         ┌─────────────────────┐
   │   Ollama                │         │   vLLM                  │
   │   - single/few users     │         │   - continuous batching  │
   │   - GGUF, quantized       │         │   - PagedAttention       │
   │     by default            │         │   - full precision       │
   │   - easy setup, low        │        │     (or tuned quant)     │
   │     resource footprint     │        │   - tuned for concurrent │
   │   - optimized for ITERATION│        │     production traffic   │
   └──────────┬────────────┘         └──────────┬────────────┘
              │                                    │
              │      both implement the SAME        │
              │      llm-client-core interface       │
              └───────────────────┬───────────────┘
                                  ▼
                    every Part 3-5 caller (agent-core,
                    rag-engine, prompt templates, tool
                    calling) is UNCHANGED -- swapping
                    backends is a config change, per
                    the Adapter pattern (Part 1, M2;
                    Part 3, M1)

              graduate this direction when:
              - concurrency demands Module 2's scheduler
              - quality bar needs full precision
              - you're benchmarking real production numbers
                 (using gpu-capacity-planner, batching-benchmark)
```

---

## 9. Recommended Resources

1. **Ollama official documentation** — the practical, current reference
   for local model management and serving configuration.
2. **GGUF format documentation (llama.cpp project)** — the technical
   detail behind the quantized local-serving format Ollama defaults to;
   read this now as a preview before Module 4's full quantization
   treatment.
3. **Your own Part 0, Module 5 (Docker)** — the "pull an image, run a
   container" mental model maps closely onto "pull a model, run it
   locally" — worth explicitly re-noting the parallel rather than
   treating Ollama's model management as an unfamiliar new concept.

---

## 10. Exercises

1. Pull and run a small open-weight model locally via Ollama. Confirm
   it responds correctly to a basic prompt before wiring it into
   anything else.
2. Implement `OllamaAdapter` in `llm-client-core`, matching the same
   interface as `VLLMAdapter` and the hosted-API adapters. Confirm you
   can switch your `agent-core` test suite's backend from a hosted API
   to Ollama purely via configuration, with no code changes anywhere in
   Part 5's code.
3. Re-run one of Part 5's regression drills (e.g., `loop-stress-test`)
   against the Ollama-backed local model instead of a hosted API.
   Confirm it passes, and note any behavioral differences you observe
   (e.g., due to the local model being smaller or more quantized than
   your usual production model).
4. Using Section 6.3's decision rule, write a one-paragraph
   justification for whether your own personal development setup for
   this handbook's projects should be Ollama-backed, vLLM-backed, or a
   hosted API, given your actual current traffic (yourself, iterating,
   low concurrency).
5. Deliberately run a concurrency test (5–10 simultaneous requests)
   against your local Ollama setup and observe what happens to latency
   per request, to build direct, personal intuition for Module 2,
   Section 6.4's claim about Ollama's scope limits — don't just accept
   it as stated.

---

## 11. Mini-Project

**`local-dev-profile`**: a configuration profile (not new code) for
`ai-api-platform` and `agent-core` that defaults to the Ollama-backed
adapter for local development and testing, with a single, clearly
documented switch to move to the hosted-API or vLLM adapter for
anything resembling production or benchmarking use — making the
Section 6.3 decision an explicit, visible configuration choice for
anyone (including future-you) picking up this codebase, rather than an
implicit assumption buried in code.

---

## 12. Production Project: `OllamaAdapter` for `llm-client-core`

### Scope

Extend `llm-client-core` (already carrying `VLLMAdapter` from Module 2)
with:

- `OllamaAdapter`, implementing the identical interface, so
  `llm-client-core` now cleanly supports three backend categories:
  hosted API, production self-hosted (vLLM), and local-development
  self-hosted (Ollama) — one interface, three interchangeable
  implementations, exactly as the Adapter pattern promises.
- `local-dev-profile` (Mini-Project) wired in as the default
  configuration for this handbook's own development environment going
  forward — every subsequent module's exercises can be run against a
  local model at zero marginal API cost, unless a specific exercise
  calls for production-quality output.
- A short, explicit written statement of Section 6.3's decision rule,
  attached to the adapter's documentation, so the choice between
  backends is never left as tribal knowledge.

### Explicit extension point

This module deliberately does not introduce a new artifact of its own
beyond the adapter and dev-profile — its contribution is completing the
Part 6 backend picture that Modules 1–2 started, and it will be used
constantly, implicitly, as the default backend for every remaining
exercise and mini-project in this handbook from here forward unless
explicitly noted otherwise.

---

## 13. Practice & Interview Questions

1. Why is "is Ollama slower than vLLM" often the wrong question to ask,
   and what's the right question instead?
2. What's the deliberate trade-off behind Ollama's default use of
   quantized GGUF models, and when is that trade-off acceptable versus
   unacceptable?
3. State a concrete decision rule for when a project should move from
   local Ollama-based development to a production vLLM deployment.
4. What does it prove about your `llm-client-core` architecture if you
   can switch between Ollama and vLLM with only a configuration change?
5. What's the risk of leaving a workload on Ollama past the point where
   its concurrency or quality needs have genuinely outgrown it?

---

## 14. Common Mistakes

- **Treating Ollama as "vLLM but worse"** — a category error; they're
  scoped to different traffic regimes, and Ollama isn't trying to win
  the throughput benchmark Module 2 cares about.
- **Benchmarking production quality claims against a quantized local
  model** — conflating "logic works" (fine to validate locally) with
  "output quality meets the production bar" (needs the actual
  production-precision deployment).
- **Leaving a genuinely production, concurrent workload on Ollama** —
  the silent-failure risk symmetrical to Module 2's over-engineering
  mistake; the gap only shows up under real load.
- **Hardcoding a specific backend into calling code** — defeating the
  entire point of `llm-client-core`'s adapter pattern and making the
  Section 6.3 graduation decision a code migration instead of a config
  change.

---

## 15. Debugging Exercise

A teammate reports that a RAG pipeline change "tested fine locally" but
produces noticeably worse answer quality once deployed to production.
Local testing was done against an Ollama-served, heavily quantized
model; production runs the full-precision model via vLLM.

Using Section 6.2 and 6.3, walk through: (a) why "the logic passed
local testing" and "the output quality meets the production bar" are
different claims that a quantized local model can only ever answer the
first of, (b) what specifically should have been tested against the
actual production-precision deployment before shipping (hint: this
connects forward to Module 4's quantization-accuracy evaluation, not
just a logic/functional test), and (c) how `local-dev-profile`'s
explicit documented distinction should have flagged this gap before
it reached production.

---

## 16. Checklist

- [ ] I can state, precisely, why Ollama and vLLM solve different
      problems rather than one being a "worse" version of the other.
- [ ] I understand Ollama's quantized-by-default design as a deliberate
      trade-off for ease of use and low footprint, not a limitation to
      work around.
- [ ] I have a working local Ollama deployment wired into
      `llm-client-core` as `OllamaAdapter`.
- [ ] I can switch my development environment between Ollama, vLLM, and
      a hosted API purely via configuration, with zero code changes in
      any calling code.
- [ ] I have an explicit, stated decision rule (Section 6.3) for when to
      graduate from local Ollama development to production serving.
- [ ] `local-dev-profile` is in place and documented as this handbook's
      default going-forward development configuration.

---

## 17. Summary

Ollama is not a lesser vLLM — it's the correctly-scoped tool for a
different problem: low-concurrency local development and iteration,
where continuous batching's entire value proposition doesn't apply
because there's no concurrent load to schedule across. Its
quantized-by-default GGUF format is a deliberate trade-off, appropriate
for validating logic but not for benchmarking final production quality.
The explicit decision rule for graduating from Ollama to production
serving — driven by real concurrency, real quality requirements, or
real benchmarking needs, not by a vague sense of "seriousness" — is
this module's actual content, made possible entirely because Module
2's Adapter-pattern discipline means the choice is a configuration
switch, not an architecture change.

---

## 18. Next Steps

Next: **Part 6, Module 4 — Quantization: Precision, Accuracy, and the
Trade-off Curve**, giving full, rigorous treatment to the technique
this module previewed as "Ollama's default" — precisely what precision
reduction costs in accuracy, how to measure that cost rather than
assume it's negligible, and how to choose a quantization level
deliberately for a given deployment's quality bar, extending both
`gpu-capacity-planner` (Module 1) and the eval infrastructure from Part
2, Module 8 and Part 3, Module 10.

Reply "continue" for Module 4, or flag anything to go deeper on.
