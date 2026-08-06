# Part 2, Module 9: Reasoning Models

---

## 1. Learning Objectives

By the end of this module, you will be able to:

- Explain chain-of-thought (CoT) reasoning precisely: why generating intermediate steps improves accuracy on multi-step problems, grounded in the autoregressive mechanics from Module 6.
- Distinguish *prompted* CoT (a prompting technique on any model) from *trained* reasoning behavior (a model specifically optimized, via RL, to produce long internal reasoning traces) — and know which current models do which.
- Explain test-time compute scaling as a genuinely new axis of capability improvement, distinct from and complementary to the pretraining-scale axis from Module 7.
- Explain, at a mechanically honest level, how RL on verifiable rewards trains a model to reason, and why "verifiable" is the operative word.
- Reason correctly about the cost/latency/accuracy trade-off of reasoning models, and know how to decide, empirically rather than by assumption, whether reasoning mode is worth it for a given task.
- Build an evaluation suite (extending `eval-framework` from Module 8) that rigorously compares a reasoning model against a standard model on a task, including cost and latency, not just accuracy.

---

## 2. Prerequisites

- Part 2, Module 6 (Attention & the Transformer Architecture) — you need the autoregressive generation mechanism solid to understand why generating more tokens before an answer can causally improve that answer.
- Part 2, Module 7 (Training vs. Inference, and Fine-tuning) — you need the pretraining/SFT/RLHF pipeline and the concept of RL-based optimization against a reward signal.
- Part 2, Module 8 (Evaluating Models) — this module's central practical exercise is an evaluation task, and assumes `eval-framework` is built and working.

---

## 3. Estimated Study Time

**5–7 hours** (2 hours theory, 2–3 hours implementation/evaluation, 1–2 hours exercises).

---

## 4. Difficulty

★★★☆☆ (3/5) — Fewer new primitive mechanisms than Modules 6 or 7 (this module leans heavily on machinery you already have); the difficulty is in correctly reasoning about *when* the added cost of reasoning is actually worth it, resisting the intuitive but frequently wrong assumption that "more visible thinking = better answer, always."

---

## 5. Why This Matters

If you've used Claude, ChatGPT, or similar assistants recently, you've likely noticed a visible "thinking" or "reasoning" phase before certain answers — long, sometimes meandering internal deliberation, especially on math, coding, and multi-step logic problems, followed by a final answer. This is not a UI gimmick layered on top of an ordinary model; it reflects a genuinely different family of models (OpenAI's o-series, Claude's extended thinking mode, DeepSeek-R1, and others), trained with a specifically different objective, that represent one of the most significant capability jumps in the field in the last two years.

As an AI engineer, this matters for a very concrete reason: **reasoning models are dramatically more expensive and slower per query** (they generate far more tokens before answering, and every one of those tokens costs money and time, per Module 5's token-economics discussion), and they are **not uniformly better** — on straightforward tasks, a standard model is often just as accurate, much cheaper, and much faster, while a reasoning model can occasionally even perform *worse* by overthinking a simple problem. The decision "should this feature call a reasoning model or a standard model" is a real, recurring, financially consequential engineering decision you will make repeatedly — and it should be made with the rigorous, evidence-based evaluation discipline from Module 8, not by a vague intuition that "more thinking is always better." This module gives you both the mechanistic understanding to reason about *why* these models work, and the concrete evaluation methodology to decide, for your specific task, whether they're worth using.

This is also a satisfying module in the sense that it ties together nearly everything in Part 2 so far: autoregressive generation (Module 6), the training pipeline and RL-based optimization (Module 7), and rigorous comparative evaluation (Module 8) — reasoning models are best understood as a specific, deliberate combination of ideas you already have, not a wholly new mechanism to learn from scratch.

---

## 6. Theory

### 6.1 Chain-of-Thought: Why Generating Intermediate Steps Actually Helps

Recall from Module 6, section 6.7, and Module 7, section 6.5: a decoder-only transformer generates one token at a time, autoregressively, and — critically — **each new token is generated conditioned on every token generated so far, including tokens the model itself just generated.** This single fact is the entire mechanical reason chain-of-thought reasoning works.

Consider a multi-step arithmetic word problem. If you force the model to jump directly from the question to a final numeric answer in one token (or a few tokens), it has to perform the *entire* multi-step calculation "in its head," within a single forward pass, with no ability to externalize or "write down" any intermediate result. This is a genuinely harder computational demand than it sounds — it's directly analogous to asking a person to solve a multi-step long-division problem entirely mentally, with no scratch paper, versus letting them write out each intermediate step.

**Chain-of-thought prompting** ("let's think step by step," or providing a few worked examples showing step-by-step reasoning before the final answer) exploits the autoregressive mechanism directly: by generating intermediate reasoning tokens *before* the final answer, the model effectively gets to use its own previous outputs as external "scratch space" that future token generations can condition on. Each individual reasoning step is a comparatively easy next-token-prediction task (much like each step of the long-division example), and the final answer, conditioned on a correct chain of intermediate steps, becomes a much easier prediction than jumping directly to it. This isn't a trick that fools the model into "seeming" smarter — it's a genuine, mechanistically real expansion of the effective computation available for a given problem, purely by using more forward-pass steps (more generated tokens) as intermediate scratch space.

**This was first observed as a pure prompting technique** (Wei et al., 2022, "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models") — you could get a significant accuracy boost on math and reasoning benchmarks from any sufficiently capable model, purely by asking it to show its work, with zero retraining. This is directly actionable for you *today*, with any model, and is worth internalizing as a default prompting habit for non-trivial reasoning tasks (a direct preview of Part 3's prompt engineering content).

### 6.2 From Prompted CoT to Trained Reasoning: The Key Distinction

Here's the distinction that defines this entire module and that's genuinely easy to blur if you're not precise about it: **prompted chain-of-thought is a technique you apply to any existing model via your prompt.** **Trained reasoning models (OpenAI's o-series, Claude's extended thinking mode, DeepSeek-R1) are a specific model *trained*, via additional RL-based post-training beyond standard RLHF (Module 7, section 6.3), to autonomously and extensively produce long internal reasoning traces before answering — without needing to be prompted to do so, and often generating far longer, more thorough, and more self-correcting reasoning chains than a standard model would produce even with an explicit "think step by step" prompt.**

The practical, observable differences:
- A standard model given a CoT prompt will produce a reasonably short, roughly linear chain of steps.
- A trained reasoning model, given the *same* problem with no special prompting at all, will often generate a substantially longer, messier-looking internal reasoning trace — including exploring an approach, noticing a mistake, backtracking, trying a different approach, and only then converging on a final answer. This self-correction behavior — genuinely reconsidering and revising an approach mid-reasoning — is a hallmark specifically of trained reasoning models and is not reliably elicited by prompting alone on a standard model.
- Reasoning models are often deployed with the lengthy internal reasoning trace either hidden from the end user by default (surfaced only as a summarized "thinking" indicator) or fully shown, depending on the product — but in either case, that reasoning is genuinely generated (and billed as tokens, per Module 5) even when not directly displayed.

### 6.3 Test-Time Compute Scaling: A New, Complementary Axis

Module 7 focused entirely on **training-time compute scaling** — bigger models, more pretraining data, more RLHF, all invested *once*, upfront, by the model provider, before the model is ever deployed. Reasoning models introduce and popularize a genuinely distinct, complementary axis: **test-time (inference-time) compute scaling** — spending *more compute per individual query*, at the moment you actually use the model, by letting it generate more reasoning tokens before answering.

This is a conceptually important shift worth sitting with: for years, the dominant lever for improving model capability was "train a bigger/better model, once." Reasoning models demonstrate a second, independent lever: **for a fixed, already-trained model, you can trade latency and inference cost for accuracy, on a per-query basis, by allocating more "thinking" budget.** This is why reasoning models typically expose a tunable "reasoning effort" or "thinking budget" parameter (directly analogous to a resource limit you'd configure in a backend system — more allocated compute generally yields better results up to a point, with diminishing returns, exactly the shape of a resource-vs-quality curve you already have strong intuition for from performance engineering, `perf-report`, Part 1 Module 10).

**The precise, honest claim, without overstating it:** more test-time reasoning compute reliably helps on tasks with genuine multi-step structure and a checkable, well-defined correct answer (competition math, coding problems with test cases, logic puzzles) — and reliably helps *much less*, or not at all, on tasks that are fundamentally about knowledge recall, subjective judgment, or simple pattern-matching, where there's no deep multi-step computation for the extra reasoning tokens to usefully perform. Treating "reasoning mode" as a universal quality upgrade, rather than a targeted tool for genuinely reasoning-heavy tasks, is the single most common practical misconception this module aims to correct.

### 6.4 How Reasoning Models Are Actually Trained: RL on Verifiable Rewards

Recall Module 7, section 6.3's RLHF pipeline: a reward *model*, itself trained from human preference comparisons, provides the training signal, because most everyday chat tasks have no simple, automatic "correct answer" to check against. Reasoning-model training uses a related but importantly different signal: **RL on verifiable rewards** — for tasks where correctness can be checked *automatically and objectively* (does the final numeric answer match the known correct answer? does the generated code pass the provided unit tests? does the logical proof actually hold?), the reward signal comes directly from this automatic check, with **no separate learned reward model, and no human preference labeling required for this part of training.**

This distinction matters mechanically: a learned reward model (Module 7) is itself an imperfect, approximate proxy for human preference, vulnerable to reward hacking (Module 7, section 6.3) precisely because it's a *model's guess* at what humans would prefer. A verifiable reward (did the code pass the tests? is the math answer correct?) is a **ground-truth, unhackable-in-principle signal** for the specific narrow class of problems where such automatic checking is possible — which is exactly the class of problems (math, code, formal logic) where reasoning models show their most dramatic, well-documented improvements. This also explains a real, honest limitation: reasoning models' advantages are most pronounced and best-evidenced precisely on these verifiable-reward domains, and comparatively far less pronounced (and considerably harder to even measure rigorously, per Module 8's entire discussion of graded/subjective evaluation) on open-ended, subjective tasks where no such automatic verification exists.

**The training loop, at a mechanically honest level (without needing to derive the specific RL algorithm):** the model generates a full reasoning trace and final answer for a given problem with a known, checkable correct answer; the verifiable reward signal (correct/incorrect, or a graded score like "tests passed") is computed automatically; an RL algorithm updates the model to make reasoning traces that led to correct/high-reward answers more likely in the future. Run over a large number of verifiable problems (competition math datasets, coding-problem-with-tests datasets, and similar), this process is what shapes a base or SFT model into a genuinely different reasoning-trained model, distinct from ordinary RLHF's chat-preference optimization.

### 6.5 The Cost/Latency/Accuracy Trade-off, Made Concrete

This is the section that turns theory into an actual engineering decision. Every reasoning-model query costs more and takes longer than an equivalent standard-model query, for two compounding reasons: (1) the reasoning trace itself consists of many additional generated tokens, each billed and each taking real generation time (recall the strictly sequential, autoregressive nature of generation from Module 7, section 6.5 — there is no way to "skip ahead" and generate only the final answer's tokens faster), and (2) reasoning-model API pricing frequently bills reasoning tokens at the same (or occasionally a different, but still non-trivial) rate as output tokens, even when those tokens are hidden from the end user by default.

**The correct engineering process, precisely, is Module 8's evaluation discipline applied to this specific question:**
1. Build (or reuse) a golden dataset genuinely representative of your actual production task.
2. Run it through both a standard model and a reasoning model, recording accuracy (via a task-appropriate metric, per Module 8, section 6.4), total latency, and total token cost for each.
3. Compute confidence intervals (per Module 8, section 6.4) on any observed accuracy difference — resist the urge to conclude "reasoning mode is better" from a small, uncontrolled sample.
4. Make the decision based on your specific task's actual measured accuracy delta, weighed explicitly against the measured cost/latency delta — a straightforward, standard engineering cost-benefit trade-off, no different in kind from deciding whether a more expensive, more accurate but slower search algorithm is worth its cost for a specific product feature.

**A genuinely common, real-world finding worth setting as your prior (to be updated by your own measurements, not taken on faith): for tasks that are primarily knowledge retrieval, straightforward classification, or simple formatting/extraction, a standard model very often matches a reasoning model's accuracy at a fraction of the cost and latency** — the extra reasoning budget has little genuine multi-step computation to usefully spend itself on. **For tasks with real multi-step logical or mathematical structure and a checkable correct answer, reasoning models frequently show a measurable, sometimes dramatic, accuracy improvement that can justify the added cost** — but "frequently" and "can" are doing real work in that sentence, and the only way to know for *your* specific task is to actually run the comparison.

---

## 7. Mental Models

1. **"Generated tokens are scratch space, and scratch space is genuinely useful computation, not theater."** Chain-of-thought works because the autoregressive mechanism lets earlier generated tokens causally influence later ones — it's real, not a magic trick that merely looks more thoughtful.
2. **"Training-time compute and test-time compute are two separate, complementary levers, not the same lever twice."** Module 7 was entirely about the first; this module introduces the second.
3. **"Verifiable rewards are a ground-truth signal; learned reward models are an educated guess."** This is precisely why reasoning models' gains are most dramatic and best-evidenced on math/code/logic — the exact domains where "correct" can be checked automatically.
4. **"More thinking helps when there's real multi-step work to do, and doesn't when there isn't."** Reasoning mode is a targeted tool for a specific class of problem, not a universal quality dial.
5. **"Never assume the trade-off — measure it."** The entire practical payoff of this module is applying Module 8's evaluation discipline to a real cost/latency/accuracy decision, on your own task, with your own numbers.

---

## 8. Visual Explanation

**Diagram 1 — Direct Answer vs. Chain-of-Thought, Autoregressive View**

```
DIRECT (no intermediate steps):
  [Problem tokens] → [Answer token(s)]
  Model must perform the entire multi-step computation
  "in one shot," within a single forward pass per output token,
  with no externalized intermediate state.

CHAIN-OF-THOUGHT:
  [Problem tokens] → [step 1 tokens] → [step 2 tokens] →
  [step 3 tokens] → ... → [Answer token(s)]

  Each arrow: the NEXT tokens are generated conditioned on ALL
  previous tokens, including the model's own just-generated
  reasoning steps — genuine externalized scratch space, not
  just a more verbose-looking answer.
```

**Diagram 2 — Prompted CoT vs. Trained Reasoning Model**

```
STANDARD MODEL + CoT PROMPT ("think step by step"):
  Problem → [short, roughly linear reasoning] → Answer
  (technique applied via prompting; works on any capable model;
   no special training required)

TRAINED REASONING MODEL (o-series, Claude extended thinking,
DeepSeek-R1-style), no special prompting needed:
  Problem → [try approach A] → [notice a flaw] → [backtrack] →
  [try approach B] → [verify against problem constraints] →
  [converge] → Answer
  (long, often messy, SELF-CORRECTING trace — a trained
   behavior, not elicited by prompting alone on a standard model)
```

**Diagram 3 — Two Independent Scaling Axes**

```
                    Training-time compute scaling (Module 7)
                    (bigger model, more pretraining data,
                     more RLHF — invested ONCE, upfront,
                     by the provider)
                              │
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                       │
        ▼                     ▼                       ▼
   Small model           Medium model            Large model
   (any of these can ALSO be given more or less test-time
    reasoning budget, independently, PER QUERY, at inference time)
        │                     │                       │
        ▼                     ▼                       ▼
   Test-time compute scaling (this module) — same trained
   model, more or fewer reasoning tokens spent per query,
   configured at call time, trading latency/cost for accuracy
```

**Diagram 4 — The Decision Process, as an Evaluation Pipeline**

```
   Golden dataset             Standard model          Reasoning model
   (your real task,     ──>   run + score        ──>  run + score
    per Module 8)              (accuracy, cost,        (accuracy, cost,
                                latency)                latency)
                                    │                        │
                                    └───────────┬────────────┘
                                                ▼
                                  Compare with confidence intervals
                                  (Module 8, section 6.4) — is the
                                  accuracy delta real, and does it
                                  justify the measured cost/latency
                                  delta for THIS specific task?
                                                │
                                                ▼
                                  A genuine, evidence-based
                                  engineering decision — not
                                  an assumption
```

---

## 9. Recommended Resources

1. **[Wei et al. — "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models" (2022)](https://arxiv.org/abs/2201.11903)** — The original CoT prompting paper; short, directly readable, and the primary source for section 6.1.
2. **[OpenAI — "Learning to Reason with LLMs" (o1 announcement/technical overview)](https://openai.com/index/learning-to-reason-with-llms/)** — OpenAI's own account of test-time compute scaling and RL-trained reasoning; read this as the primary source for sections 6.2–6.4.
3. **[DeepSeek-AI — "DeepSeek-R1: Incentivizing Reasoning Capability in LLMs via Reinforcement Learning" (2025)](https://arxiv.org/abs/2501.12948)** — A rare, unusually transparent, fully open technical paper describing RL-on-verifiable-rewards training for a reasoning model in genuine mechanistic detail; an excellent, concrete complement to OpenAI's higher-level account.
4. **[Anthropic — "Claude's extended thinking" documentation](https://docs.claude.com)** — Search Anthropic's official docs for the current extended-thinking/reasoning-mode documentation, since this is a fast-moving product surface; read this for the concrete, practical API-level knobs (thinking budget, etc.) you'll actually configure.
5. **[Snell et al. — "Scaling LLM Test-Time Compute Optimally Can Be More Effective than Scaling Model Parameters" (2024)](https://arxiv.org/abs/2408.03314)** — A rigorous empirical study directly comparing the two scaling axes from section 6.3; read this once the practical material feels solid, to deepen the quantitative grounding.
6. **[Hugging Face — "Open-R1" project blog/repo](https://huggingface.co/blog/open-r1)** — A well-documented open effort to reproduce reasoning-model training; useful for seeing the RL-on-verifiable-rewards pipeline described in genuinely reproducible, engineering-level detail rather than only at a research-paper level of abstraction.

---

## 10. Exercises

1. **CoT prompting, hands-on.** Take a multi-step math word problem and run it through any available standard (non-reasoning) model twice: once asking directly for just the final answer, once with an explicit "think step by step" instruction. Compare correctness across both, ideally across 5-10 different problems, and report your observed accuracy difference.
2. **Reasoning trace inspection.** Using a model with an available "show thinking" or extended-thinking mode, run a genuinely difficult multi-step logic or math problem and read through the full reasoning trace. Identify at least one point where the model appears to backtrack or reconsider an earlier step — describe what you observed.
3. **Verifiable vs. non-verifiable, sorted.** For each of the following tasks, classify whether it has a genuinely automatically-verifiable correct answer (usable for RL-on-verifiable-rewards training) or not, and justify briefly: (a) "is this Sudoku solution valid," (b) "is this product review positive or negative in tone," (c) "does this code pass these unit tests," (d) "is this poem beautiful," (e) "is this geometric proof logically valid."
4. **Cost estimation.** Using published pricing for a reasoning model and a comparable standard model from the same provider, estimate the cost difference for running 1,000 queries of a specific task through each, assuming a reasonable estimate of reasoning-token overhead (you can look this up or estimate conservatively) — connect this back to Module 5's token-cost discussion.
5. **Predict, then test, where reasoning mode helps.** Before running any code, write down a prediction: for which of these three task types would you expect a reasoning model to show the largest accuracy improvement over a standard model — (a) simple factual lookup ("what's the capital of Peru"), (b) a multi-step logic puzzle, (c) sentiment classification of short reviews? Justify your prediction using section 6.3's theory, then, if you have access to both model types, test at least one of these predictions empirically.

---

## 11. Mini-Project: `reasoning-vs-standard-lab`

A small, focused, hands-on experiment designed to make the cost/accuracy trade-off from section 6.5 concrete and personally measured, rather than assumed.

**Requirements:**
- Choose two distinct task types: one with genuine multi-step structure and a checkable answer (e.g., multi-step arithmetic word problems, or a small set of logic puzzles), and one that's primarily knowledge recall or simple classification (e.g., factual Q&A, or sentiment classification).
- For each task type, assemble a small golden dataset (10-20 examples is enough for this mini-project; note explicitly, per Module 8, that this is too small for strong statistical confidence and treat conclusions accordingly).
- Run both task types through both a standard model and a reasoning model (or a reasoning model with reasoning explicitly enabled vs. disabled/minimized, if that's what's available to you), recording accuracy, approximate token cost, and latency for each combination.
- Produce a simple 2x2 summary table (task type × model type, with accuracy/cost/latency in each cell) and write a short conclusion: for which task type did reasoning mode show a meaningful accuracy improvement, and was it worth the measured cost/latency overhead?

---

## 12. Production Project: `reasoning-eval-suite` (extends `eval-framework`)

This directly extends `eval-framework` from Part 2, Module 8 — you are not building new evaluation infrastructure from scratch, you're adding a genuinely new, reusable capability to it: **cost/latency-aware comparative evaluation**, not just accuracy comparison.

**What to build:**

1. **A `ModelComparisonRun` abstraction** in `eval-framework` that, given a golden dataset and two or more model/configuration variants (e.g., "standard model," "reasoning model, low effort," "reasoning model, high effort"), runs all of them against the same golden dataset and records, per example, per variant: correctness score (via the appropriate scorer from Module 8), token count (input + output/reasoning tokens separately), estimated cost, and wall-clock latency.
2. **A comparative report generator** producing a clear summary: accuracy (with confidence interval, per Module 8) per variant, median and p95 latency per variant, total estimated cost per variant for a representative query volume (e.g., "cost per 1,000 queries at your production traffic estimate"), and a plain-language recommendation given a configurable priority (e.g., "optimize for accuracy regardless of cost" vs. "optimize for cost, accept a small accuracy trade-off if the confidence interval allows it").
3. **A "task-type detector" heuristic (documented as a heuristic, not a guarantee)**: using your `reasoning-vs-standard-lab` findings and section 6.3's theory as a starting point, write a short, explicit checklist (in the README, not hidden in code) for engineers on your (hypothetical) team to use when deciding whether a *new* task is a good candidate for reasoning mode at all, before even running a full comparison — e.g., "does this task have genuine multi-step logical/mathematical structure with a checkable answer? If no, default to standard model and only test reasoning mode if standard-model accuracy is measurably insufficient."
4. **Regression protection**: wire `reasoning-eval-suite`'s cost/latency tracking into the same CI-gating pattern from `eval-framework` (Module 8) so that a future change (e.g., a provider pricing change, or a prompt change that inadvertently causes a reasoning model to "think" much longer than before) that causes a statistically credible cost or latency regression is caught automatically, not discovered via a surprise bill.

**Explicitly designed for extension:** `reasoning-eval-suite`'s cost/latency-aware comparison pattern becomes directly reusable in Part 3 (comparing prompting strategies' cost-effectiveness), Part 5 (deciding whether a given agent step genuinely needs a reasoning model call or can use a cheaper standard call), and Part 6 (informing model-serving and routing decisions at the infrastructure level — e.g., a smart router that sends only genuinely reasoning-heavy sub-tasks to an expensive reasoning model, and everything else to a cheaper standard model).

---

## 13. Practice & Interview Questions

1. Explain, mechanistically, why generating intermediate reasoning steps before a final answer can genuinely improve accuracy, grounding your answer in the autoregressive generation mechanism from Module 6.
2. What is the precise difference between prompted chain-of-thought and a trained reasoning model? Can you elicit trained-reasoning-model-like behavior from a standard model purely through prompting?
3. What is "test-time compute scaling," and how is it a genuinely different lever from the training-time compute scaling covered in Module 7?
4. Explain "RL on verifiable rewards" and why it's meaningfully different from the RLHF reward-model approach covered in Module 7. Why does this distinction explain why reasoning models show their most dramatic gains specifically on math/code/logic tasks?
5. A product manager asks you to "just turn on reasoning mode for everything, it can only make answers better." How would you respond, using this module's concepts?
6. Design a rigorous (not vibes-based) process for deciding whether a specific production feature should use a reasoning model or a standard model. What would you measure, and what would count as sufficient evidence to justify the reasoning model's added cost?
7. Why might a reasoning model occasionally perform *worse* than a standard model on a very simple task? (Hint: think about what "overthinking" might mean in terms of the reasoning trace and self-correction behavior from section 6.2.)
8. What specific property must a task have for it to be a good candidate for RL-on-verifiable-rewards training, and why does that same property make it a good candidate for benefiting from test-time reasoning compute?

---

## 14. Common Mistakes

- **Assuming "reasoning mode" is a universal quality upgrade** and enabling it by default for all tasks, rather than treating it as a targeted tool for genuinely multi-step, verifiable-answer tasks.
- **Confusing prompted CoT with trained reasoning capability** — assuming that because "think step by step" helps a standard model, a trained reasoning model is doing fundamentally the same, just-a-longer-prompt thing (it isn't; the training process, per section 6.4, is genuinely different).
- **Evaluating a reasoning model vs. standard model trade-off on too small a sample** and drawing a confident conclusion anyway — this is precisely Module 8's small-sample-noise lesson, directly recurring in this module's central practical decision.
- **Ignoring cost and latency entirely and optimizing for accuracy alone**, when in most real production contexts the actual engineering decision is a genuine three-way trade-off, not a pure accuracy-maximization problem.
- **Assuming all "reasoning model" products work identically under the hood** — implementation details (whether the reasoning trace is shown, how "reasoning effort" is configured and billed) vary meaningfully by provider, and you should verify current documentation (per this module's resources) rather than assuming based on how one specific provider's product works.
- **Forgetting that reasoning tokens are still real, billed tokens** (per Module 5's token-cost discussion), even when hidden from the end user by default — a genuine, sometimes underestimated line item in a cost model.

---

## 15. Debugging Exercise

Your team has enabled reasoning mode by default for a customer-support-ticket classification feature (classify each incoming ticket into one of 8 predefined categories), reasoning that "it can only help accuracy." Three weeks later, a cost review flags that this feature's LLM API spend is now roughly 6x higher than projected, while a quick accuracy spot-check shows classification accuracy is statistically indistinguishable from what a much cheaper standard model achieved in an earlier internal test.

**Your task:** Using this module's concepts, explain precisely why this specific task type (ticket classification into predefined categories) was a poor candidate for reasoning mode in the first place — reference section 6.3's theory about which task properties predict genuine benefit from test-time reasoning compute. Then, propose a concrete remediation plan using `reasoning-eval-suite` (this module's production project): what would you measure, what golden dataset would you build or reuse, and what specific evidence would justify keeping reasoning mode enabled for this feature versus reverting to a standard model, given that the informal spot-check already suggests no meaningful accuracy benefit? Be specific about what statistically rigorous evidence (not just a "quick spot-check") this decision actually requires before being finalized either way.

---

## 16. Checklist

- [ ] I can explain, mechanistically, why chain-of-thought reasoning improves accuracy, grounded in the autoregressive generation mechanism.
- [ ] I can clearly distinguish prompted CoT from trained reasoning-model behavior, and I know current examples of each category of model.
- [ ] I can explain test-time compute scaling as a distinct, complementary axis to the training-time scaling from Module 7.
- [ ] I can explain RL on verifiable rewards and why it produces the most dramatic, best-evidenced gains specifically on math/code/logic tasks.
- [ ] I've run `reasoning-vs-standard-lab` and have my own concrete, measured evidence about when reasoning mode helps and when it doesn't, rather than relying on assumption.
- [ ] I've built `reasoning-eval-suite`, extending `eval-framework` with cost/latency-aware comparative evaluation and CI-gated regression protection.
- [ ] I can answer all 8 interview questions without notes.

---

## 17. Summary

Reasoning models are best understood not as a mysterious new architecture, but as a deliberate combination of ideas you already have: the autoregressive generation mechanism (Module 6) makes chain-of-thought genuinely useful, not just cosmetically longer, because generated reasoning tokens function as real, causally-influential scratch space for subsequent tokens; RL on verifiable rewards (a variant of the RL-based post-training from Module 7, using automatic correctness checks instead of a learned reward model) trains a model to produce long, self-correcting reasoning traces autonomously, without needing to be prompted; and this unlocks test-time compute scaling — a genuinely new, complementary lever to training-time scaling, letting you trade latency and cost for accuracy on a per-query basis. This benefit is real and often substantial on tasks with genuine multi-step, verifiable structure (math, code, logic), and comparatively small or absent on tasks that are primarily knowledge recall or simple classification — meaning "should this feature use reasoning mode" is a genuine, evidence-based engineering decision, not a default you should flip on and forget. You put this into practice with `reasoning-vs-standard-lab`'s hands-on comparison, and by extending `eval-framework` into `reasoning-eval-suite`, giving yourself a reusable, cost/latency-aware evaluation tool for this exact decision, applicable to every future task you'll encounter across the rest of this handbook.

---

## 18. Next Steps

**Next module: Part 2, Module 10 — "Vision, Speech & Multimodal Models" (Part 2 capstone).** You've now covered the complete text-based pipeline in depth: tokenization, embeddings, attention/transformers, training/fine-tuning, evaluation, and reasoning. Module 10 closes out Part 2 by extending everything you've learned to non-text modalities — how images get tokenized into a transformer-compatible sequence (vision transformers, patch embeddings), how speech is processed (audio tokenization, Whisper-style architectures), and how multimodal models combine these with the text pipeline you already understand deeply into a single, unified architecture. As the Part 2 capstone, this module will also include a cumulative review connecting every mechanism from Modules 4 through 10 into one coherent mental model, before Part 3 moves from "how models work" into "how to build production systems with them."

---

Reply "continue" for Module N, or flag anything to go deeper on.
