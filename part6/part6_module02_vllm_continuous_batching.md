# Part 6, Module 2: vLLM & Continuous Batching

> Takes Module 1's memory-bandwidth argument and shows exactly how a
> modern serving framework exploits it: batching multiple requests'
> decode steps together to amortize the cost of reading weights from
> VRAM, and managing the KV cache dynamically (PagedAttention) to avoid
> the capacity waste naive implementations hit under real concurrent
> traffic.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain why naive, static batching wastes GPU memory-bandwidth
   advantage for real (variable-length) traffic, and how **continuous
   batching** fixes this at the scheduler level.
2. Explain **PagedAttention** as a direct application of a concept from
   your own OS/backend background (virtual memory paging) to the KV
   cache problem from Module 1, Section 6.3.
3. Deploy a real, self-hosted model behind vLLM, and measure — not
   assume — the throughput and latency improvement continuous batching
   provides over a naive single-request-at-a-time baseline.
4. Add a self-hosted serving adapter to `llm-client-core` (Part 3,
   Module 1), so `agent-core`'s agents (Part 5) can route to a
   self-hosted vLLM endpoint exactly as they route to a hosted API,
   with no change to any calling code.
5. Correctly identify which of Part 5's workloads (per Module 1's
   prefill/decode framing) benefit most from continuous batching, and
   which don't benefit much, so you don't over-apply this module's
   technique where it isn't the bottleneck.

## 2. Prerequisites

- Part 6, Module 1 — this module is a direct, concrete answer to the
  memory-bandwidth argument built there; without it, vLLM's design
  choices will look like arbitrary engineering rather than a specific
  answer to a specific bottleneck.
- Part 0, Module 12 (Async Programming) — continuous batching is, at
  its core, a scheduling problem, and the concurrency vocabulary from
  that module transfers directly.
- Part 3, Module 1 (`llm-client-core`'s adapter pattern) — this
  module's Production Project adds a new adapter to the same package,
  reusing its Factory/Adapter design rather than building parallel
  infrastructure.

## 3. Estimated Study Time

10–13 hours across 2–3 sessions.

## 4. Difficulty

★★★★☆ (4/5) — The concepts are approachable given Module 1's
foundation, but getting a real vLLM deployment running, measuring real
throughput numbers, and correctly wiring it into `llm-client-core`
without breaking any existing Part 3/5 behavior is genuine, hands-on
infrastructure work.

---

## 5. Why This Matters

Module 1 established that decode is memory-bandwidth-bound: generating
one token means reading nearly the entire model's weights, for
comparatively little computation. The direct, valuable consequence: if
you're already paying the cost of reading those weights from VRAM, you
should use that one costly read to compute the next token for as many
concurrent requests as possible, not just one. This is not a minor
optimization — it is the difference between a self-hosted deployment
that's competitive with hosted APIs on throughput and cost, and one
that isn't, and it is precisely why production-grade serving frameworks
like vLLM exist rather than everyone writing their own inference loop
around a model's raw forward pass.

This matters for your specific track for a second reason: this is the
module where "AI infrastructure engineer" stops being an abstract job
title and starts being concrete, hands-on systems work — configuring a
real scheduler, measuring real throughput under real concurrent load,
debugging real memory fragmentation. This is squarely the kind of work
your ops/infrastructure background gives you a real head start on
relative to a typical ML-first candidate, and it's exactly the kind of
concrete, measured (not tutorial-copied) experience that differentiates
a candidate in an infrastructure-adjacent interview at an AI lab.

---

## 6. Theory

### 6.1 Why naive batching fails for real traffic

The simplest possible improvement on Module 1's single-request
bottleneck: batch multiple requests together, so one weight-read serves
several requests' next-token computation at once. The naive
implementation — collect a fixed batch of N requests, run them through
decode together, wait for all N to finish before starting the next
batch — has a specific, serious flaw for real traffic: **requests don't
all finish generating at the same time.** A batch of 8 requests where
one needs 500 tokens and the rest need 20 forces the GPU to keep
processing the batch (wasting compute on requests that already reached
their stopping point, or leaving batch slots idle) until the longest
request finishes — the batch's total latency is bounded by its slowest
member, and GPU utilization for the shorter requests' idle slots is
wasted the entire time.

This is a direct structural analog to a problem you already know from
backend work: static thread-pool batching with heterogeneous job
durations has exactly this head-of-line-blocking-adjacent problem — a
batch of database queries where one is slow holds up the whole batch's
resource release, even though the other queries in the batch finished
long ago. The fix in both domains is the same idea: don't treat the
batch as a fixed, synchronized unit — make scheduling **continuous** and
per-request.

### 6.2 Continuous batching: scheduling at the token level, not the request level

**Continuous batching** (the technique vLLM is built around) changes the
unit of scheduling from "a batch of requests, synchronized" to "a single
decode step, across whatever requests currently need one." At every
decode step, the scheduler looks at all currently-active requests,
computes the next token for each of them together (a real batched
matrix operation, still amortizing the weight-read cost from Module
1), and as soon as any individual request finishes (hits a stop token
or its max length), it's immediately removed from the batch and a new
waiting request is immediately admitted to take its place — all without
waiting for the rest of the batch to finish. This directly fixes
Section 6.1's problem: no request's completion is held hostage by
another request's length, and no GPU capacity sits idle waiting for a
slow request while a new request is queued and ready.

The throughput gain this produces is not incremental — for realistic,
variable-length production traffic, continuous batching is frequently
cited as producing multi-x throughput improvements over naive static
batching, precisely because it eliminates both the head-of-line-
blocking waste and the idle-slot waste in one mechanism. **Measure this
yourself in the Production Project rather than taking the multiplier on
faith** — the exact number depends heavily on your specific traffic's
length distribution, which is exactly Module 1's "prefill/decode
heaviness" question resurfacing in a new form.

### 6.3 PagedAttention: your OS knowledge, applied to the KV cache

Module 1, Section 6.3 established that the KV cache grows with
concurrent requests and context length, and is often the true VRAM
ceiling. A naive KV cache implementation allocates one large,
contiguous block of VRAM per request, sized for that request's *maximum
possible* length — which wastes VRAM badly (most requests don't use
their maximum length) and causes fragmentation (VRAM freed by a
finished request may not be contiguous enough to serve a new request's
allocation, even if the total free VRAM would technically suffice).

**PagedAttention** is vLLM's answer, and it is, precisely, the same
idea as OS-level virtual memory paging (Part 0, Module 3's Linux
fundamentals touches the OS side of this; if you've ever reasoned about
page tables, this will feel immediately familiar): instead of one
contiguous allocation per request, the KV cache is divided into
fixed-size **blocks**, allocated to requests on demand as they actually
generate tokens (not pre-reserved for a hypothetical maximum length),
with a block table mapping each request's logical sequence position to
physical blocks that need not be contiguous in VRAM. This eliminates
both failure modes of the naive approach at once: no VRAM is wasted
reserving for a maximum length that mostly won't be used, and
fragmentation stops being a problem because blocks don't need to be
contiguous — exactly the reason paged virtual memory solved the
equivalent fragmentation problem for process memory decades ago. If you
can already explain why paged virtual memory beats a naive contiguous
allocation model for a general-purpose OS, you already understand *why*
PagedAttention works — the KV cache application is new, the underlying
argument is not.

### 6.4 What continuous batching does and doesn't help

Tying back to Module 1's prefill/decode distinction directly: continuous
batching's throughput gain comes specifically from better utilizing GPU
capacity during the memory-bandwidth-bound decode phase across many
concurrent requests. It does relatively little for:

- **Single-user, low-concurrency workloads** — if there's rarely more
  than one active request, there's nothing to batch, and the
  scheduling machinery is pure overhead with no benefit. This is
  directly relevant to whether a small self-hosted deployment for
  personal or low-traffic use (versus a production multi-tenant
  service) actually needs vLLM's full machinery, or whether a simpler
  serving setup (Ollama, covered in Module 3) is the correctly-scoped
  choice — over-engineering here is a real cost, not a virtue.
- **Latency for a single request in isolation** — continuous batching
  improves aggregate *throughput* under concurrent load; it does not
  make any single request's decode phase inherently faster in
  isolation. For Part 5's voice agent, where a single conversation's
  round-trip latency is the metric that matters, continuous batching
  helps only insofar as the deployment is actually serving many
  concurrent voice sessions — the benefit is a fleet-level throughput
  story, not a single-conversation latency story, and conflating the
  two is a common mistake when justifying this technique to a
  non-infrastructure stakeholder.

---

## 7. Mental Models

1. **"If you're paying to read the weights anyway, use that read for as
   many requests as currently need it — that's the whole idea of
   continuous batching."**
2. **"A batch's latency is bounded by its slowest member — schedule per
   token, not per fixed batch, to stop the slow one from holding up the
   rest."**
3. **"PagedAttention is virtual memory paging, applied to the KV
   cache — same fragmentation problem, same block-based fix."**
4. **"Continuous batching is a throughput story under real concurrency,
   not a single-request latency story — measure it against the traffic
   pattern it actually helps."**

---

## 8. Visual Explanation

```
   NAIVE STATIC BATCHING (batch of 4, heterogeneous lengths):

   req A: ██████████████████████████████ (30 tokens)
   req B: ████████ (8 tokens)                [idle after tok 8]
   req C: ████ (4 tokens)                    [idle after tok 4]
   req D: ██████████████ (14 tokens)         [idle after tok 14]
          └────────────── batch doesn't finish/release until req A
                          finishes -- B, C, D's slots sit idle,
                          wasting GPU capacity the whole time

   CONTINUOUS BATCHING (per-token scheduling):

   req A: ██████████████████████████████
   req B: ████████ [DONE, slot freed] → req E admitted ████████████
   req C: ████ [DONE, slot freed] → req F admitted ██████████████
   req D: ██████████████ [DONE, slot freed] → req G admitted ██████
          └── no request waits on another; no GPU slot sits idle
              waiting for the batch's slowest member

   ──────────────────────────────────────────────────

   PAGED ATTENTION (KV cache as pages, not one contiguous block):

   naive: [--- reserved for req A's MAX possible length ---]
                (mostly wasted if A finishes early)

   paged: req A's logical sequence → [block 3][block 7][block 1]
          req B's logical sequence → [block 2][block 5]
          (physical blocks allocated on demand, not required to
           be contiguous -- exactly like OS virtual memory pages)
```

---

## 9. Recommended Resources

1. **Kwon et al., "Efficient Memory Management for Large Language Model
   Serving with PagedAttention" (2023, the original vLLM paper)** — the
   primary source; read this now that Section 6.3's OS-paging analogy
   has given you the conceptual entry point, and connect its formal
   description back to what you already know about virtual memory.
2. **vLLM official documentation** — the practical, current reference
   for deployment configuration; use this (not a third-party tutorial)
   for the Production Project's actual setup steps, since serving-
   framework APIs change quickly.
3. **Your own Part 0, Module 3 (Linux, Bash & the Terminal)** — reread
   whatever you covered there on virtual memory/paging before Section
   6.3, if it's not already fresh; this module leans on that
   intuition directly.
4. **Your own Part 0, Module 12 (Async Programming)** — the scheduling
   vocabulary (queues, admission, preemption) transfers directly to
   understanding continuous batching's scheduler.

---

## 10. Exercises

1. Deploy a small open-weight model behind vLLM locally (or on a rented
   GPU instance). Confirm it serves requests successfully before
   measuring anything.
2. Design a synthetic traffic generator that sends concurrent requests
   with a realistic, heterogeneous length distribution (some short,
   some long) — not uniform-length requests, which would hide Section
   6.1's problem entirely.
3. Measure throughput (tokens/second, aggregate) for your vLLM
   deployment under Exercise 2's traffic, and compare against a naive
   single-request-at-a-time baseline you implement yourself (even a
   simple sequential loop calling the model directly). Report the real
   multiplier you observe, not a cited figure.
4. Using `gpu-capacity-planner` (Part 6, Module 1), estimate your KV
   cache's VRAM budget for your test deployment's concurrency level,
   and confirm (via vLLM's own memory reporting) that it roughly
   matches what PagedAttention is actually allocating.
5. Identify one workload from Part 5 (pick a specific specialized
   agent from Module 5) and argue, using Section 6.4, whether it would
   meaningfully benefit from continuous batching in your own
   deployment scenario, or whether a simpler serving setup would be
   correctly scoped instead.

---

## 11. Mini-Project

**`batching-benchmark`**: a small, repeatable benchmark harness that
runs your Exercise 2/3 comparison (naive sequential vs. vLLM continuous
batching) against a fixed, documented traffic profile, and reports
throughput, p50/p95 latency, and GPU memory utilization for both. This
becomes your first real, personally-measured evidence for exactly the
kind of claim ("continuous batching gave us a Nx throughput
improvement under our traffic profile") that's valuable in an interview
precisely because you measured it yourself rather than citing a
benchmark from someone else's paper.

---

## 12. Production Project: `self-hosted-adapter` for `llm-client-core`

### Scope

Extend `llm-client-core` (Part 3, Module 1/12) with:

- A new `VLLMAdapter`, implementing the same interface as the existing
  `AnthropicAdapter`/`OpenAIAdapter`, so that any code calling
  `llm-client-core` — including every `agent-core` agent from Part 5 —
  can route to a self-hosted vLLM endpoint by configuration, with zero
  changes to calling code. This is the Factory/Adapter pattern from
  Part 1, Module 2 paying off exactly as designed: a new provider is a
  new adapter, not a new integration surface.
- Model-routing logic (reusing Part 3, Module 9's routing discipline)
  that can send some requests to the self-hosted endpoint and others to
  a hosted API — useful for gradual migration, cost-sensitive routing,
  or fallback if the self-hosted deployment is unavailable.
- `batching-benchmark` (Mini-Project) wired in as a repeatable
  regression check: if a future vLLM version upgrade or configuration
  change regresses throughput, this benchmark should catch it before
  it reaches production traffic.
- Documentation of the exact deployment configuration used (model,
  quantization setting if any — full treatment in Module 4 — batch size
  limits, GPU type), so the setup is reproducible, not tribal knowledge.

### Explicit extension point

**Part 6, Module 3 (Ollama & Local Development)** will add a second,
lighter-weight adapter for local/development use, alongside
`VLLMAdapter`, giving `llm-client-core` a clear production-vs-development
self-hosted story. **Part 5's voice agent** (Module 5) becomes the
first real consumer of `VLLMAdapter` routed for low latency, directly
closing the limitation named in Part 5, Module 6.

---

## 13. Practice & Interview Questions

1. Why does naive, fixed-batch-size batching underperform for
   variable-length production traffic, specifically in terms of GPU
   utilization?
2. Explain continuous batching as a scheduling-level fix, and name the
   backend/systems concept it most closely resembles.
3. Explain PagedAttention using the virtual-memory-paging analogy,
   without using the word "attention" until you've fully explained the
   memory-management idea.
4. When would continuous batching provide little to no benefit, and why
   would adopting vLLM's full machinery be over-engineering for that
   case?
5. How would you measure, rather than assume, the actual throughput
   improvement continuous batching provides for your specific traffic
   pattern?
6. How does adding a self-hosted serving adapter to `llm-client-core`
   demonstrate that the Adapter pattern from Part 1 was designed
   correctly?

---

## 14. Common Mistakes

- **Assuming a cited throughput multiplier applies to your own
  traffic** — the real gain depends heavily on your traffic's
  length-distribution heterogeneity; measure it with
  `batching-benchmark` rather than quoting a paper's number.
- **Deploying vLLM's full continuous-batching machinery for a
  low-concurrency personal or dev use case** — real overhead for no
  real benefit when there's rarely more than one active request; a
  simpler setup (Module 3) is correctly scoped there.
- **Conflating throughput gains with single-request latency gains** —
  continuous batching is a fleet-level throughput story; a single
  isolated request's decode latency isn't inherently faster because of
  it.
- **Sizing a naive KV cache allocation for a request's theoretical
  maximum length** — wastes VRAM and reintroduces the fragmentation
  problem PagedAttention exists specifically to solve.
- **Building a new, parallel integration path for self-hosted serving
  instead of a new adapter** — ignoring Part 3, Module 1's existing
  Adapter/Factory pattern and duplicating integration logic that should
  be a drop-in extension.

---

## 15. Debugging Exercise

Your vLLM deployment's aggregate throughput numbers look excellent in
your benchmark, but a specific class of production requests — Part 5's
voice agent's live conversations — reports worse perceived latency after
migrating to this self-hosted deployment than the previous hosted-API
setup had.

Using Section 6.4, walk through: (a) why an excellent aggregate
throughput benchmark doesn't guarantee good single-conversation
latency, especially under lower real concurrency than your benchmark
assumed, (b) the specific measurement (from `batching-benchmark`,
re-run under the voice agent's actual concurrency level, not your
original benchmark's traffic profile) that would reveal the gap, and
(c) whether the fix is a batching-configuration change, a routing
change (send voice traffic to a differently-tuned deployment), or a
signal that this specific workload was never well-suited to
continuous batching's throughput-under-concurrency value proposition in
the first place.

---

## 16. Checklist

- [ ] I can explain why naive static batching wastes GPU capacity for
      variable-length, real production traffic.
- [ ] I can explain continuous batching as per-token, not per-batch,
      scheduling, and why that fixes the head-of-line problem.
- [ ] I can explain PagedAttention using the virtual-memory-paging
      analogy, and connect it back to Module 1's KV cache VRAM
      argument.
- [ ] I have a working vLLM deployment serving a real model.
- [ ] `batching-benchmark` reports a real, personally-measured
      throughput comparison against a naive baseline, not a cited
      figure.
- [ ] I can identify at least one workload where continuous batching's
      benefit is small, and explain why, using Section 6.4.
- [ ] `VLLMAdapter` is wired into `llm-client-core` using the existing
      Adapter/Factory pattern, with zero changes required in any
      calling code from Part 3 or Part 5.

---

## 17. Summary

Continuous batching is the direct, concrete answer to Module 1's
memory-bandwidth argument: since generating a token means paying to
read the model's weights regardless of how many requests need that
next token, schedule at the per-token level so no request's completion
is held hostage by another's length, and no GPU capacity sits idle
waiting for a batch's slowest member. PagedAttention solves the KV
cache's corresponding capacity-management problem the same way OS
virtual memory solves general memory fragmentation — block-based
allocation on demand, not one contiguous reservation sized for a
theoretical maximum. Both gains are real and measurable, but they are
specifically throughput-under-concurrency gains, not universal wins —
correctly scoping when this machinery is worth its complexity is as
much a part of this module's skill as knowing how to deploy it.

---

## 18. Next Steps

Next: **Part 6, Module 3 — Ollama & Local Development Serving**, a
deliberately lighter-weight counterpart to this module: when the
production-grade continuous-batching machinery is the wrong tool
(low-concurrency local development, quick experimentation, Part 0's
original dev environment), what the correctly-scoped alternative looks
like, and how `llm-client-core`'s adapter pattern accommodates both
without forcing a single one-size-fits-all serving choice.

Reply "continue" for Module 3, or flag anything to go deeper on.
