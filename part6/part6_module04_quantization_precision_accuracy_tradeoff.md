# Part 6, Module 4: Quantization — Precision, Accuracy, and the Trade-off Curve

> Full, rigorous treatment of the technique Module 3 previewed as
> "Ollama's default": precisely what precision reduction costs in
> accuracy, how to measure that cost rather than assume it's
> negligible, and how to choose a quantization level deliberately for a
> given deployment's quality bar.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain quantization mechanically — reducing the numeric precision
   used to store and/or compute a model's weights — and connect it
   directly back to Module 1's memory-bandwidth argument: fewer bytes
   per weight means less data moved per token, which is the actual
   mechanism behind the speedup.
2. Distinguish the major practical quantization approaches (post-
   training quantization vs. quantization-aware training; weight-only
   vs. weight-and-activation quantization) at a level sufficient to
   reason about their trade-offs, without needing to derive the
   numerical methods from scratch.
3. State precisely why quantization is not free — where accuracy loss
   comes from, why it's uneven across tasks (some capabilities degrade
   much faster than others at the same bit-width), and why "the
   benchmark average barely moved" can hide a real, task-specific
   regression.
4. Build a rigorous **quantization-accuracy eval**, reusing Part 2,
   Module 8's evaluation-framework and Part 3, Module 10's
   hallucination-rate methodology, rather than trusting a vendor's
   aggregate benchmark claim.
5. Choose a quantization level deliberately for a specific deployment,
   using `gpu-capacity-planner`'s VRAM savings estimate weighed against
   your own measured accuracy cost — a real trade-off decision, not a
   default setting.
6. Extend `gpu-capacity-planner` and the standing eval infrastructure
   with a `quantization-eval-bench` that makes this trade-off a
   re-runnable, evidence-based decision for any future model choice.

## 2. Prerequisites

- Part 6, Modules 1–3 — quantization is the direct payoff of Module 1's
  memory-bandwidth argument and Module 3's GGUF preview; this module
  assumes that foundation rather than re-deriving it.
- Part 2, Module 8 (Evaluating Models, `eval-framework`) — this
  module's rigor depends entirely on reusing that framework rather than
  inventing a new evaluation approach.
- Part 3, Module 10 (Hallucination Reduction) — the task-specific,
  category-based evaluation discipline from that module is the direct
  template for Section 6.3's "uneven degradation" argument.

## 3. Estimated Study Time

10–13 hours across 2–3 sessions.

## 4. Difficulty

★★★★☆ (4/5) — The mechanics of quantization are approachable; the
discipline of actually measuring task-specific accuracy cost rigorously,
instead of trusting an aggregate number, is where the real difficulty
(and the real value) is.

---

## 5. Why This Matters

Quantization is usually presented — in marketing material, in casual
engineering conversation, and often in tutorials — as a nearly-free
win: "4-bit quantization gives you a 4x memory reduction with minimal
quality loss." The second half of that sentence is doing enormous,
usually unexamined work. "Minimal quality loss" measured against what,
exactly? Averaged across which tasks? This module exists because the
single most common mistake engineers make with quantization is trusting
an aggregate benchmark number instead of measuring accuracy cost for
their own specific task and deployment — and aggregate numbers reliably
hide task-specific regressions, for reasons Section 6.3 makes precise.

This matters concretely for your own handbook's artifacts: if you
quantize the model behind `rag-engine`'s faithfulness-checking
reflection step (Part 5, Module 2) or `agent-core`'s planning calls
without measuring the specific accuracy cost on *those* specific
capabilities, you can silently degrade the very safety mechanisms
Part 5 spent six modules building — a reflection step that quietly gets
worse at self-critique, or a faithfulness scorer that quietly gets
worse at detecting unfaithfulness, is exactly the kind of regression an
aggregate "average benchmark score barely moved" claim would hide. This
module makes sure you never make that mistake by accident.

---

## 6. Theory

### 6.1 What quantization actually does, mechanically

A model's weights are normally stored and computed at a given numeric
precision — commonly 16-bit floating point. Quantization reduces this
precision — to 8-bit, 4-bit, or lower — representing each weight with
fewer bits. This directly reduces the number of bytes that must be
moved from VRAM to compute cores per weight, per token — precisely the
memory-bandwidth bottleneck Module 1, Section 6.1 identified as the
actual constraint on decode-phase latency. A model quantized from
16-bit to 4-bit moves a quarter of the bytes per weight, which — to the
extent decode really is memory-bandwidth-bound as Module 1 argued —
translates fairly directly into a real speedup, plus the VRAM savings
`gpu-capacity-planner` already accounts for in its weight-size term
(Module 1, Section 6.3).

The catch: fewer bits per weight means less numerical precision to
represent the model's learned parameters, and information is lost in
that reduction. The central question this module answers is not
whether information is lost (it definitively is) but *how much that
loss actually costs*, and — critically — *whether it costs the same
amount for every capability the model has*.

### 6.2 Post-training quantization vs. quantization-aware training, at the level you need

- **Post-training quantization (PTQ)**: take an already-fully-trained
  model and reduce its weights' precision afterward, typically using a
  small calibration dataset to choose quantization parameters that
  minimize the resulting error. This is the common, practical approach
  for most self-hosting scenarios (including what Ollama's GGUF
  ecosystem typically distributes) — cheap to apply (no retraining
  needed), with an accuracy cost that varies by bit-width and technique.
- **Quantization-aware training (QAT)**: incorporate the effects of
  quantization *during* training, so the model's weights are learned in
  a way that's more robust to the eventual precision reduction. This
  produces better accuracy at a given bit-width than PTQ, but requires
  access to the training pipeline and compute budget — usually out of
  reach unless you're doing your own fine-tuning (Part 2, Module 7) of
  a model you intend to quantize aggressively.

For your own work, PTQ applied to an already-released open-weight model
is the realistic default; QAT matters more if you're the one training
or heavily fine-tuning a model and know in advance it will be deployed
quantized.

### 6.3 Why accuracy loss is uneven, not uniform — and why the average hides it

This is the module's central, load-bearing claim, and the direct
analog of Part 3, Module 10's task-specific hallucination-category
discipline applied to quantization instead of generation quality:
**quantization does not degrade every capability by the same amount.**
Capabilities that depend on precise numerical reasoning, exact
multi-step logical chains, or fine-grained distinctions between
similar outputs tend to degrade faster at aggressive bit-widths than
capabilities like fluent text generation or broad topical
summarization, which tend to remain comparatively robust even at low
bit-widths. This is mechanically plausible, not just an empirical
curiosity: precise reasoning chains can be more sensitive to small,
compounding numerical errors introduced at each step than
tasks with more redundancy or more forgiving output spaces.

The direct, costly consequence: a standard aggregate benchmark (a broad
mixture of task types, averaged into one score) can show "barely any
degradation" at a given bit-width, while a *specific* capability your
system actually depends on — reflection's self-critique accuracy (Part
5, Module 2), a `FaithfulnessScorer`'s unfaithfulness-detection rate
(Part 4, Module 8), an agent's structured tool-call argument accuracy
(Part 3, Module 3) — degrades meaningfully at that same bit-width,
invisibly, because it's one of many tasks averaged into a score that
moved only slightly. **This is why you must evaluate quantization
against your own system's specific dependent capabilities, using your
own eval infrastructure, not a vendor's aggregate number** — exactly the
same discipline Part 3, Module 10 established for hallucination
categories, applied here to a hardware/precision decision instead of a
generation-technique decision.

### 6.4 Weight-only vs. weight-and-activation quantization

A further distinction worth knowing at a practical level: some
quantization schemes reduce only the stored weights' precision while
keeping activations (intermediate computed values during the forward
pass) at higher precision; others quantize both. Weight-only schemes
are generally simpler and see wider practical adoption for self-hosted
LLM serving specifically because weights dominate the VRAM/bandwidth
cost (Module 1's arithmetic) far more than activations do for typical
batch sizes — meaning weight-only quantization captures most of the
memory-bandwidth benefit with a comparatively smaller accuracy cost
than also quantizing activations. This is not a claim to take on faith
for your own deployment — it's exactly the kind of distinction your own
`quantization-eval-bench` should let you test empirically rather than
assume from general knowledge.

### 6.5 Choosing a quantization level: a real trade-off, made explicit

The correct decision process, stated as a rule rather than a vibe:

1. Use `gpu-capacity-planner` (Module 1) to compute the VRAM/cost
   savings at each candidate bit-width for your target deployment.
2. Run `quantization-eval-bench` (this module's Production Project) at
   each candidate bit-width, specifically against the capabilities
   *your own system* depends on (retrieval-quality proxies from Part 4,
   reflection-agreement rates from Part 5, tool-call schema accuracy
   from Part 3) — not a generic public benchmark.
3. Choose the most aggressive (cheapest, fastest) bit-width whose
   measured accuracy cost, on your specific dependent capabilities, is
   within a bar you set deliberately and explicitly before running the
   eval — not adjusted after seeing results to justify whatever number
   came out.

This mirrors Part 4, Module 4's re-ranking cost/benefit discipline
(measure at the values that actually matter to your system, not
generically) and Part 6, Module 2's benchmarking discipline
(`batching-benchmark`, measured against your own traffic) — this
module's contribution is applying the same "measure your own system's
dependency, don't trust an aggregate," now to a precision/accuracy
trade-off specifically.

---

## 7. Mental Models

1. **"Fewer bits per weight means fewer bytes moved per token — that's
   the entire mechanism behind quantization's speedup."**
2. **"Accuracy loss from quantization isn't uniform — precise reasoning
   degrades faster than fluent generation, and the average score hides
   exactly that gap."**
3. **"Evaluate quantization against the specific capabilities your own
   system depends on — not a vendor's aggregate benchmark."**
4. **"Set your accuracy bar before you see the eval results, or you're
   not making a decision, you're rationalizing one."**

---

## 8. Visual Explanation

```
   PRECISION REDUCTION -> MEMORY-BANDWIDTH SPEEDUP (Module 1 payoff):

   16-bit weight: [================] 2 bytes/weight
    8-bit weight: [========]         1 byte/weight  -- half the bytes
    4-bit weight: [====]             0.5 bytes/weight -- quarter the bytes
                        fewer bytes moved per token during decode
                        = the actual mechanism, per Module 1 Section 6.1

   ──────────────────────────────────────────────────

   WHY THE AGGREGATE AVERAGE HIDES THE REAL COST:

   aggregate benchmark score (many task types averaged):
   16-bit: ██████████████████████████████████████  92%
    4-bit: █████████████████████████████████████   90%   <- "barely moved"

   but broken out by capability:
                                16-bit      4-bit
   fluent summarization:        94%         93%     <- robust
   broad Q&A:                   90%         88%     <- mild
   precise multi-step reasoning: 88%         71%     <- SEVERE, hidden
   structured tool-call schema:  95%         80%     <- SEVERE, hidden
                                             by the averaged 90%

   -> quantization-eval-bench must test the SPECIFIC capabilities
      your system depends on, not just the aggregate
```

---

## 9. Recommended Resources

1. **Dettmers et al., "LLM.int8()" and related quantization papers** —
   primary technical sources for how modern PTQ methods actually work;
   read for the mechanics behind Section 6.2's PTQ description.
2. **Hugging Face — "Quantization" documentation** — practical, current
   reference for applying different quantization schemes to real
   open-weight models.
3. **Your own Part 2, Module 8 (`eval-framework`) and Part 3, Module 10
   (`hallucination-eval-suite`) code** — the evaluation infrastructure
   this module reuses directly; re-read before building
   `quantization-eval-bench` rather than writing new eval scaffolding
   from scratch.
4. **GGUF/llama.cpp quantization scheme documentation** (revisit from
   Module 3) — the practical bit-width options (e.g., various named
   quantization levels) you'll actually be choosing between for local
   and self-hosted deployments.

---

## 10. Exercises

1. Quantize the same open-weight model at two or three different
   bit-widths (e.g., 16-bit baseline, 8-bit, 4-bit). Run a broad,
   generic benchmark and confirm the aggregate score moves only
   modestly, as Section 6.3 predicts.
2. Using the same quantized models, run a task specifically resembling
   one of your system's dependent capabilities (e.g., structured
   tool-call argument generation, from Part 3, Module 3's schema) and
   measure accuracy separately. Confirm whether the gap is larger than
   the aggregate benchmark suggested.
3. Using `gpu-capacity-planner`, compute the VRAM and cost savings at
   each bit-width you tested, and place them side by side with
   Exercise 2's accuracy numbers to make the trade-off visible in one
   table.
4. State, in writing, your accuracy bar for one specific capability
   (e.g., "reflection agreement rate must stay within 5 percentage
   points of the 16-bit baseline") *before* running the eval for a new
   bit-width, then run it and report whether it passed — resist the
   temptation to adjust the bar afterward.
5. Test Section 6.4's claim (weight-only quantization costs less
   accuracy than weight-and-activation quantization for a similar
   memory benefit) empirically against your own model and task, rather
   than accepting it as given.

---

## 11. Mini-Project

**`quantization-tradeoff-table`**: a single table, generated from real
runs (not estimated), with rows for each candidate bit-width and
columns for VRAM/cost savings (from `gpu-capacity-planner`) and
accuracy on at least three of your system's specific dependent
capabilities (from Exercise 2's methodology). This table is the
concrete artifact that turns "quantization involves a trade-off" from
an abstract statement into an evidence-based decision you can actually
defend.

---

## 12. Production Project: `quantization-eval-bench`

### Scope

Build `quantization-eval-bench`, extending Part 2, Module 8's
`eval-framework` and Part 3, Module 10's category-based evaluation
methodology, that:

- Runs a configurable model at multiple quantization levels against a
  golden dataset assembled specifically from your own system's
  dependent capabilities — reflection agreement (Part 5, Module 2),
  faithfulness detection (Part 4, Module 8), tool-call schema accuracy
  (Part 3, Module 3), and RAG retrieval-quality proxies (Part 4,
  Module 1) — not a generic public benchmark suite.
- Reports results broken out per capability, never collapsed into a
  single aggregate score, specifically to prevent Section 6.3's
  hiding-the-regression failure mode from recurring in your own
  reporting.
- Integrates with `gpu-capacity-planner` (Module 1) to present the
  VRAM/cost savings alongside the accuracy cost for each candidate
  bit-width, producing `quantization-tradeoff-table` (Mini-Project) as
  a standing, re-runnable report rather than a one-time analysis.
- Ships with an explicit, pre-committed accuracy bar per capability
  (Section 6.5's discipline), checked into the repository before any
  new model or bit-width is evaluated, so future evaluations can't be
  quietly adjusted after the fact to justify a convenient result.
- Is wired as a required step in `agent-ci-gate` (Part 5, Module 6)
  and `rag-ci-gate` (Part 4, Module 9) whenever the underlying
  self-hosted model or its quantization level changes — a quantization
  change is now a change that must pass the same regression bar as any
  other code change to these systems.

### Explicit extension point

**Part 6's remaining modules on GPU load balancing and Kubernetes-based
scaling** will use `quantization-eval-bench`'s chosen bit-width as a
fixed input to capacity planning — the VRAM savings from this module's
decision directly determine how many concurrent requests a given GPU
fleet can serve, feeding forward into those modules' scaling
calculations.

---

## 13. Practice & Interview Questions

1. Explain, mechanically, why reducing a model's weight precision
   produces a real inference speedup, connecting your answer to Module
   1's memory-bandwidth argument.
2. Why can an aggregate benchmark show minimal quantization-related
   degradation while a specific capability your system depends on
   degrades severely?
3. What's the practical difference between post-training quantization
   and quantization-aware training, and when would you actually need
   the latter?
4. Design an evaluation plan for choosing a quantization level for a
   deployment that specifically depends on structured tool-call
   accuracy. What would you measure, and what would you refuse to rely
   on?
5. Why should your accuracy bar for a quantization decision be set
   before running the evaluation, not after?
6. Why might weight-only quantization be preferred over
   weight-and-activation quantization for typical LLM serving, in terms
   of Module 1's memory-bandwidth argument specifically?

---

## 14. Common Mistakes

- **Trusting an aggregate benchmark's "minimal degradation" claim** —
  the single most consequential mistake in this module; averages
  systematically hide capability-specific regressions.
- **Evaluating quantization against a generic public benchmark instead
  of your own system's dependent capabilities** — producing a number
  that's technically true but irrelevant to whether your actual
  deployment will regress.
- **Setting the accuracy bar after seeing the results** — turns a
  decision into a post-hoc rationalization; commit to the bar first.
- **Assuming weight-only and weight-and-activation quantization have
  similar cost/benefit without testing it** — a claim worth verifying
  empirically for your own model and task, not accepting from general
  knowledge.
- **Treating a quantization-level change as a config tweak, not a
  regression-worthy change** — skipping `quantization-eval-bench` in
  CI when the underlying model or bit-width changes, silently letting a
  degraded reflection or faithfulness capability into production.

---

## 15. Debugging Exercise

Several weeks after quantizing your self-hosted model from 8-bit to
4-bit to save on GPU cost, users start reporting that your agent
(Part 5) occasionally takes actions that don't match what it seemed to
plan — a subtle but real degradation. Your aggregate quality benchmark,
run before the change, showed only a 1-point drop.

Using Section 6.3 and this module's `quantization-eval-bench`, walk
through: (a) which specific capability is most likely responsible (hint:
structured tool-call/action-schema generation is exactly the kind of
precise, low-redundancy task Section 6.3 identifies as disproportionately
sensitive to aggressive quantization), (b) why the aggregate benchmark's
1-point drop was consistent with a much larger capability-specific
regression, and (c) the specific evaluation that should have been run,
and the specific bar that should have blocked this deployment, before
it reached production.

---

## 16. Checklist

- [ ] I can explain quantization's speedup mechanism directly in terms
      of Module 1's memory-bandwidth argument.
- [ ] I understand why quantization's accuracy cost is uneven across
      capabilities, and can name the mechanical reason precise/low-
      redundancy tasks degrade faster.
- [ ] I've measured — not assumed — a real gap between an aggregate
      benchmark score and a capability-specific score for at least one
      quantized model.
- [ ] `quantization-tradeoff-table` exists, built from real runs, for
      at least two candidate bit-widths.
- [ ] I set an explicit accuracy bar before running an evaluation, not
      after, at least once, and can show the result either passed or
      failed against that pre-committed bar.
- [ ] `quantization-eval-bench` evaluates against my own system's
      dependent capabilities (reflection, faithfulness, tool-call
      accuracy), not a generic public benchmark.
- [ ] `quantization-eval-bench` is wired into `agent-ci-gate` and
      `rag-ci-gate` as a required check whenever the underlying model
      or quantization level changes.

---

## 17. Summary

Quantization's speedup is a direct, mechanical consequence of Module
1's memory-bandwidth argument — fewer bytes per weight means fewer
bytes moved per token. Its accuracy cost is real, and critically,
uneven: precise, low-redundancy capabilities (structured reasoning,
exact tool-call schemas) degrade faster than fluent, high-redundancy
ones (summarization, casual generation), and an aggregate benchmark
score reliably hides this gap. The discipline this module adds to Part
6 is refusing to trust an aggregate number for a decision that affects
your own system's specific dependent capabilities — measuring against
your own eval infrastructure, with an accuracy bar set before you see
the results, exactly as rigorous as Part 2 and Part 3 already taught
you to be about model evaluation generally, now applied to a
hardware/precision trade-off.

---

## 18. Next Steps

Next: **Part 6, Module 5 — Load Balancing & Horizontal Scaling for
Model Serving**, which takes a single quantized, vLLM-served model
instance (Modules 1–4) and scales it across multiple GPU instances to
handle real production traffic volume — reusing Part 1, Module 13's
fan-out/circuit-breaker patterns (`resilient-gateway`) at the model-
serving layer instead of the backend-service layer.

Reply "continue" for Module 5, or flag anything to go deeper on.
