# Part 6, Module 1: GPU Fundamentals & the Economics of Model Serving

> Part 6 begins here — Pankaj's platform/infrastructure-engineering
> track. This module builds the hardware and cost intuition every later
> Part 6 module (vLLM, quantization, Kubernetes-based scaling) depends
> on, and directly addresses Part 5, Module 6's first named limitation:
> no self-hosted, latency-optimized serving path for `agent-core`'s
> agents, especially the voice agent from Part 5, Module 5.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain why LLM inference is **memory-bandwidth-bound**, not
   compute-bound, for the token-generation phase specifically — and why
   this single fact determines almost every later design decision in
   Part 6 (batching, quantization, hardware choice).
2. Do the arithmetic that converts a model's parameter count and
   precision into a VRAM requirement, including the KV cache's growth
   with sequence length and batch size — the calculation that
   determines whether a model fits on a given GPU at all.
3. Distinguish the **prefill** phase (processing the prompt) from the
   **decode** phase (generating tokens one at a time) and explain why
   they have opposite hardware utilization profiles.
4. Build a first-principles cost comparison between hosted API calls
   and self-hosted serving, as a function of request volume — and state
   the volume threshold, not just a vague "it depends," at which
   self-hosting becomes economically rational for a given workload.
5. Map GPU vendor/type choices (consumer vs. datacenter, on-demand vs.
   spot/reserved cloud instances) onto the same infrastructure
   trade-off vocabulary you already have from Part 0, Module 14
   (cloud fundamentals) and Part 1's `ai-api-platform`.
6. Build `gpu-capacity-planner`, a tool that takes a model spec and
   traffic profile and outputs VRAM requirements, phase-specific
   utilization estimates, and a hosted-vs-self-hosted cost
   recommendation — extending `ai-api-platform` with real infrastructure
   planning capability.

## 2. Prerequisites

- Part 0, Module 14 (Cloud Fundamentals, AWS-first) — this module reuses
  that cost/capacity vocabulary directly, applied to GPU instances
  specifically.
- Part 1, Module 11 (`ai-api-platform`) — this module's Production
  Project extends it with a real capacity-planning capability.
- Part 2, Module 2 (Neural Networks from First Principles) — you need
  the basic notion of matrix multiplication as the core operation to
  follow the compute/memory-bandwidth argument in Section 6.
- Part 5, Module 6 — the named limitation this Part addresses.

## 3. Estimated Study Time

9–12 hours across 2 sessions.

## 4. Difficulty

★★★☆☆ (3/5) — No new ML theory; the difficulty is in genuinely internalizing
hardware-level arithmetic (memory bandwidth, VRAM budgets) that a
pure-software background doesn't usually build intuition for.

---

## 5. Why This Matters

Every AI engineering job posting mentions "inference optimization" and
"cost efficiency," and almost every candidate who hasn't done
infrastructure work treats this as a vague gesture toward "using a
smaller model" or "caching more." The engineers who actually get hired
into infrastructure-adjacent roles at AI labs can do the arithmetic:
given a model size and a target latency, they can say whether a
workload fits on a given GPU, whether it's compute-bound or
memory-bandwidth-bound, and at what request volume self-hosting starts
saving money over an API. That arithmetic is this module's entire
content, and it is exactly the kind of "read the actual numbers, don't
pattern-match to a vibe" skill your backend/ops background is well
suited to pick up quickly — it rhymes with JVM heap sizing and capacity
planning for a database cluster far more than it rhymes with anything
in a typical ML course.

This also matters for the specific thread running through Part 5:
Module 6 named "no self-hosted, latency-optimized serving" as an
explicit limitation, tied directly to the voice agent's tight latency
budget from Module 5. Everything from here through the rest of Part 6
is the answer to that limitation — but you cannot reason correctly
about vLLM's batching strategy (Module 2) or quantization's trade-offs
(Module 4) without first understanding *why* memory bandwidth, not raw
compute, is usually the bottleneck being optimized against. Skipping
this module and jumping straight to "just use vLLM" produces an
engineer who can follow a tutorial but can't explain, in an interview or
in a production incident, why a specific configuration choice helped.

---

## 6. Theory

### 6.1 Why inference is memory-bandwidth-bound, not compute-bound

A transformer's forward pass is, mechanically, a sequence of matrix
multiplications (Part 2, Module 2/6). A GPU's raw compute throughput
(FLOPs) is enormous relative to how fast it can move data between its
memory (VRAM) and its compute cores. For every matrix multiplication in
a decode step, the GPU must first load the relevant weights from VRAM
into its compute units — and for LLM inference at batch size 1 (a
single user's request, one token at a time), the amount of *compute*
per token is small relative to the amount of *data* (the model's
weights, which don't change between tokens but still have to be read)
that must move across the memory bus to do it.

This is the single most load-bearing fact in this module, so state it
precisely: **decoding one token requires reading essentially the
entire model's weights from VRAM, but performs comparatively little
computation with them.** The bottleneck is therefore the GPU's memory
bandwidth (how many bytes per second it can move from VRAM to its
compute cores), not its FLOP throughput. This is the mechanical reason
behind almost every optimization technique you'll meet in this Part:

- **Batching** (Module 2) helps because if you're going to pay the cost
  of reading the weights from VRAM anyway, you might as well use those
  same weights, once loaded, to compute multiple requests' next tokens
  simultaneously — amortizing the memory-bandwidth cost across more
  useful compute.
- **Quantization** (Module 4) helps because it directly shrinks the
  number of bytes that must be read from VRAM per token — a model in
  8-bit precision moves half the bytes per weight that a 16-bit model
  does, directly speeding up the bandwidth-bound bottleneck (with an
  accuracy trade-off, covered when you get there).
- **The KV cache** (Section 6.3) exists specifically to avoid
  re-computing (and re-reading weights for) information you already
  computed for earlier tokens in the same sequence.

Contrast this with **training** (Part 2, Module 3), which processes
large batches of many sequences at once and is far more compute-bound —
this is precisely why training and inference hardware/optimization
strategies genuinely differ, and why "just use the same setup you
trained on" is a common, costly mistake for a team's first production
serving deployment.

### 6.2 Prefill vs. decode: opposite utilization profiles

A single inference request has two distinct phases:

- **Prefill**: processing the entire input prompt at once, computing
  attention over all prompt tokens in parallel. This phase *is*
  reasonably compute-bound — there's enough parallel work (many tokens
  processed simultaneously) to keep the GPU's compute cores busy
  relative to memory movement.
- **Decode**: generating output tokens one at a time, each one
  depending on all previous tokens (via the KV cache). This phase is
  the memory-bandwidth-bound one from Section 6.1, and it dominates
  total latency for any reasonably long generation.

This distinction matters practically because a serving system that's
tuned only against prefill-heavy benchmarks (e.g., short-generation
question answering) can look excellent and then perform poorly on
decode-heavy workloads (e.g., long-form generation, or an agent's
multi-step reasoning trace from Part 5, which is exactly the kind of
workload this Part exists to optimize). When you evaluate a serving
setup (Module 2 onward), always ask which phase your actual workload
spends more time in, and benchmark against that phase specifically —
this is the same discipline as Part 3, Module 9's insistence that
latency/cost tradeoffs be measured per-tier rather than only in
aggregate, applied to inference phases instead of request tiers.

### 6.3 VRAM arithmetic: fitting a model, and the KV cache's growth

The core capacity-planning calculation, stated precisely enough to
actually compute:

**Model weights VRAM** ≈ `parameter_count × bytes_per_parameter`. A 7B-parameter
model at 16-bit precision (2 bytes/parameter) needs roughly 14 GB just
to hold the weights — before any room for activations or the KV cache.
This is the first, non-negotiable check: does the model even fit on the
GPU you're considering, at the precision you intend to serve it at?

**KV cache VRAM** grows with `batch_size × sequence_length × model_dimension`-
related terms, and — this is the part that surprises engineers coming
from a stateless-request mental model — it grows *linearly with the
number of concurrent requests* and *linearly with how long each
conversation/context gets*. This is why a chat or agent workload (Part
5, with long scratch-pads per Part 3, Module 7's context-engineering
discipline) can exhaust available VRAM well before the model weights
themselves would suggest a problem — the KV cache, not the weights, is
often the actual capacity ceiling in a real serving deployment with
concurrent long-context requests. This is the mechanical reason Part 3,
Module 7's "context window is a finite contested resource" framing
applies doubly hard once you're self-hosting: it's contested against
your own GPU's VRAM budget, not just against the model's advertised
context length.

**Rule of thumb, worth internalizing and then testing with real
numbers in the exercises:** total VRAM needed ≈ weights + KV cache
(scales with concurrent load and context length) + a working-memory
overhead for activations. Any capacity plan that only accounts for
weights and ignores KV cache growth under real concurrent traffic will
be wrong at production scale, sometimes badly.

### 6.4 The hosted-vs-self-hosted cost crossover

An API call has a per-token cost with zero fixed cost and zero
operational burden. Self-hosting has a large fixed cost (GPU
instance-hours, whether idle or busy) and a much lower marginal
per-token cost, plus real operational burden (this Part's remaining
modules: serving software, scaling, monitoring, on-call). The crossover
point — the request volume above which self-hosting is cheaper — is a
straightforward break-even calculation: `fixed_hourly_GPU_cost / (hosted_API_cost_per_token − self_hosted_marginal_cost_per_token)`
gives you the token volume per hour at which the two options cost the
same. Below that volume, hosted APIs win on pure cost (and they also
win on operational simplicity at any volume, which is a real cost even
if not a monetary one). Above it, self-hosting wins on cost, and the
decision becomes whether the operational investment is worth it given
your team's size and the workload's latency/data-residency
requirements — which is often the real deciding factor, not raw
per-token economics, especially for a workload like Part 5's voice
agent where the latency requirement, not the cost, is the primary
driver toward self-hosting.

**Do this calculation for your own workload before choosing a hardware
strategy** — this is precisely the "measure, don't assume" discipline
from Part 3, Module 9, applied to a much larger, much less reversible
decision (a GPU reservation commitment, unlike an API call, isn't
something you can casually undo).

### 6.5 GPU choice: consumer vs. datacenter, on-demand vs. reserved/spot

Mapping onto Part 0, Module 14's cloud vocabulary directly:

- **Consumer GPUs** (e.g., gaming-class cards): cheap per unit,
  sometimes with real VRAM limitations and without datacenter-class
  reliability/multi-GPU interconnect features. Reasonable for local
  development and small-scale self-hosting; usually the wrong choice
  for production serving at any real scale or reliability requirement.
- **Datacenter GPUs**: designed for exactly the sustained,
  high-utilization, multi-tenant workloads a production serving system
  is. Higher cost per unit, but the correct default for anything beyond
  development.
- **On-demand vs. reserved vs. spot cloud GPU instances**: the same
  trade-off as Part 0, Module 14's general cloud compute discussion —
  on-demand for unpredictable or bursty load, reserved for a
  known-steady baseline (directly relevant once you have real
  production traffic data from Module 2 onward), spot for
  interruption-tolerant, non-latency-critical batch workloads (e.g.,
  the ingestion-time embedding computation from Part 4, Module 1 — not
  live inference serving a voice agent).

---

## 7. Mental Models

1. **"Decoding a token means reading the whole model from memory to do
   comparatively little math with it — bandwidth, not compute, is what
   you're fighting."**
2. **"Prefill is compute-bound, decode is memory-bound — benchmark
   against whichever one your actual workload spends more time in."**
3. **"The KV cache, not the model weights, is usually what actually
   runs out of VRAM first under real concurrent traffic."**
4. **"Do the break-even arithmetic before committing to a GPU — a
   reservation isn't an API call you can casually undo."**

---

## 8. Visual Explanation

```
   ONE decode step, batch size 1:

   VRAM (weights, ~14GB for a 7B model @ 16-bit)
        │
        │  read ~ALL weights to compute
        │  ONE next token
        ▼
   compute cores ──────► 1 token out
        │
        │  (small amount of actual math
        │   relative to bytes moved --
        │   THIS is why it's bandwidth-
        │   bound, not compute-bound)

   ────────────────────────────────────────────

   PREFILL vs DECODE, same request:

   prefill: [tok1][tok2][tok3]...[tokN]  <- all processed
            ══════════════════════════      in parallel,
            compute-bound, GPU cores busy    compute-heavy

   decode:  [tok N+1] -> [tok N+2] -> [tok N+3] -> ...
            each step reads full weights again,
            memory-bandwidth-bound, dominates
            total latency for long generations

   ────────────────────────────────────────────

   VRAM BUDGET, growing with real traffic:

   ┌─────────────────────────────────────┐
   │  model weights (fixed, one-time)       │
   ├─────────────────────────────────────┤
   │  KV cache (grows with:                 │
   │    batch_size × concurrent requests    │
   │    × context length per request)       │  <- usually the
   ├─────────────────────────────────────┤     REAL ceiling
   │  activation working memory (smaller)   │     at scale
   └─────────────────────────────────────┘
```

---

## 9. Recommended Resources

1. **NVIDIA — "Inference Technical Overview" / GPU architecture
   whitepapers** — the official source for memory-bandwidth vs. compute
   throughput numbers for specific GPU models; use real published specs
   for the exercises rather than approximations.
2. **Kipply (Kipply's blog) — "Transformer Inference Arithmetic"** — a
   widely-cited, precise, first-principles derivation of the exact VRAM
   and FLOP arithmetic this module introduces at an intuitive level;
   read this for the full derivation.
3. **Hugging Face — "Optimizing LLMs for Speed and Memory"** — practical,
   official documentation connecting the theory here to real
   configuration choices you'll make starting in Module 2.
4. **Your own Part 0, Module 14 material** — re-read the cloud
   cost/capacity section before Section 6.5, since this module is a
   direct, GPU-specific application of that vocabulary rather than a
   fresh topic.

---

## 10. Exercises

1. Using a real, published GPU spec (VRAM size, memory bandwidth in
   GB/s), calculate whether a 7B-parameter model at 16-bit precision
   fits, and estimate a rough per-token decode latency bound from the
   bandwidth figure alone (bytes that must move ÷ bandwidth).
2. Calculate KV cache growth for a fixed model as concurrent request
   count goes from 1 to 50, holding context length constant, and
   separately as context length goes from 500 to 8,000 tokens, holding
   concurrency constant. Which grows the VRAM requirement faster in
   your numbers?
3. Do the hosted-vs-self-hosted break-even calculation (Section 6.4)
   for a real, published hosted-API price per token and a real,
   published cloud GPU hourly price. State the token-per-hour volume at
   which self-hosting wins.
4. For three workloads — a chatbot answering short questions, a
   document-summarization batch job, and Part 5's voice agent — decide
   whether each is prefill-heavy or decode-heavy, and justify each.
5. Using Part 0, Module 14's cloud vocabulary, decide which GPU
   provisioning strategy (on-demand/reserved/spot) fits each of the
   three workloads from Exercise 4, and justify each choice.

---

## 11. Mini-Project

**`vram-calculator`**: a small script or notebook that takes a model's
parameter count, precision, target concurrency, and target context
length as input, and outputs an estimated total VRAM requirement
(weights + KV cache + overhead), with the KV cache term implemented
using the actual formula from Kipply's derivation (Resource 2), not a
rough approximation. Validate it against at least one real, published
deployment's documented VRAM requirements.

---

## 12. Production Project: `gpu-capacity-planner`

### Scope

Extend `ai-api-platform` (Part 1, Module 11) with a real
capacity-planning capability:

- `gpu-capacity-planner`, built on top of the Mini-Project's
  `vram-calculator`, that takes a model spec and a traffic profile
  (requests/second, average context length, average generation length)
  and outputs: (a) minimum VRAM required, with weights and KV cache
  broken out separately; (b) whether the workload is prefill-heavy or
  decode-heavy, using the profile's average context vs. generation
  length ratio; (c) a hosted-vs-self-hosted cost recommendation using
  Section 6.4's break-even calculation, parameterized by real,
  configurable API and GPU pricing so it stays current as prices
  change; (d) a GPU provisioning-strategy recommendation
  (on-demand/reserved/spot) based on the traffic profile's variability.
- Wired into `ai-api-platform`'s existing observability (Part 1, Module
  4) so that *real* traffic data — not just hypothetical profiles — can
  feed the planner directly, turning it from a one-off calculator into
  a tool that can be re-run as traffic patterns actually evolve.
- Documented with the specific numbers used (GPU bandwidth figures,
  pricing) cited to their sources, so the tool's recommendations are
  auditable, not black-box.

### Explicit extension point

**Part 6, Module 2 (vLLM & Continuous Batching)** will take
`gpu-capacity-planner`'s output as its starting input — the batching
strategy you choose there is a direct function of the
prefill/decode-heaviness and concurrency numbers this module's tool
produces. **Part 6's later Kubernetes/scaling modules** will use the
provisioning-strategy recommendation as the seed for actual autoscaling
policy design.

---

## 13. Practice & Interview Questions

1. Why is LLM token generation (decode) considered memory-bandwidth-
   bound rather than compute-bound? Walk through the mechanism.
2. What's the practical difference between the prefill and decode
   phases of inference, and why does it matter which one dominates a
   given workload's latency?
3. Given a model's parameter count and precision, walk through the VRAM
   calculation for the weights alone, then explain what else has to be
   added and why that additional term can dominate at scale.
4. Explain, with real (or realistic) numbers, how you'd calculate the
   request-volume crossover point at which self-hosting becomes cheaper
   than a hosted API.
5. Why might a long-context agent workload (like one from Part 5) run
   out of GPU memory well before the model's own weight size would
   suggest a problem?
6. When would spot instances be the wrong choice for a GPU workload,
   even though they're the cheapest option per hour?

---

## 14. Common Mistakes

- **Assuming inference is compute-bound, like training** — leads to
  optimizing the wrong thing (more FLOPs) instead of the actual
  bottleneck (memory bandwidth) for decode-heavy workloads.
- **Capacity-planning only for model weights** — ignoring KV cache
  growth under real concurrent, long-context traffic, and then being
  surprised by out-of-memory errors at production scale that a correct
  calculation would have predicted.
- **Benchmarking a serving setup only against prefill-heavy workloads**
  — producing numbers that don't predict real performance for
  decode-heavy production traffic like an agent's multi-step reasoning.
- **Choosing self-hosting based on "it feels cheaper" instead of the
  break-even calculation** — a GPU reservation is a much less reversible
  commitment than an API call, and deserves the same measured rigor
  Part 3, Module 9 insisted on for cost/latency tradeoffs generally.
- **Defaulting to spot instances for cost savings on a latency-critical
  workload** — a spot instance's interruption risk is incompatible with
  a live-serving voice agent's latency guarantees, even though it's the
  cheapest option on paper.

---

## 15. Debugging Exercise

Your self-hosted serving deployment handles single-user testing fine,
but under realistic concurrent production traffic, it starts throwing
out-of-memory errors — even though the model's weights alone
comfortably fit in the GPU's VRAM with room to spare.

Using Section 6.3, walk through: (a) the most likely cause (hint: check
what happens to VRAM usage as concurrent requests and average context
length both increase under real traffic, versus the single-request test
that passed), (b) the specific calculation from `vram-calculator` that
would have predicted this failure before it happened in production, and
(c) two structural mitigations (one about capacity, one about request
admission) you could apply, distinguishing which one is a capacity fix
and which is a traffic-shaping fix.

---

## 16. Checklist

- [ ] I can explain, precisely, why decode is memory-bandwidth-bound
      and prefill is comparatively compute-bound.
- [ ] I can calculate a model's weight VRAM requirement from parameter
      count and precision, from memory, without looking it up.
- [ ] I can explain why KV cache growth, not model weight size, is
      often the real production capacity ceiling.
- [ ] I've done the hosted-vs-self-hosted break-even calculation with
      real, current pricing numbers, not hypothetical ones.
- [ ] I can match a workload's prefill/decode heaviness to an
      appropriate benchmarking and optimization strategy.
- [ ] I can match a traffic profile's variability to an appropriate
      GPU provisioning strategy (on-demand/reserved/spot).
- [ ] `vram-calculator` is validated against at least one real
      published deployment's documented requirements.
- [ ] `gpu-capacity-planner` is wired into `ai-api-platform`'s real
      observability data, not just hypothetical inputs.

---

## 17. Summary

LLM inference's core hardware fact — decode is memory-bandwidth-bound,
not compute-bound, because generating one token means reading nearly
the entire model's weights from VRAM to do comparatively little
computation with them — is the single idea that explains why batching,
quantization, and KV-cache management (this Part's remaining modules)
all matter, and why they matter in the specific ways they do. VRAM
capacity planning must account for the KV cache's growth under real
concurrent, long-context traffic, not just the model's static weight
size, or production deployments fail in ways a correct calculation
would have predicted. And the decision to self-host at all should rest
on an explicit break-even calculation against your actual traffic
volume, not a vague sense that self-hosting is "more efficient" —
`gpu-capacity-planner` makes that calculation a re-runnable tool rather
than a one-time guess.

---

## 18. Next Steps

Next: **Part 6, Module 2 — vLLM & Continuous Batching**, which takes
this module's memory-bandwidth argument and shows exactly how modern
serving frameworks exploit it — batching multiple requests' decode
steps together to amortize the cost of reading weights from VRAM, and
managing the KV cache dynamically (PagedAttention) to avoid the
capacity waste this module's arithmetic predicts naive implementations
will hit. `gpu-capacity-planner`'s output becomes that module's starting
input.

Reply "continue" for Module 2, or flag anything to go deeper on.
