# Part 2, Module 7: Training vs. Inference, and Fine-tuning

---

## 1. Learning Objectives

By the end of this module, you will be able to:

- Precisely distinguish pretraining, fine-tuning, and inference as three distinct phases with different objectives, different data, and different compute profiles.
- Explain what "pretraining" actually optimizes for at internet scale, and why next-token prediction alone produces a model with broad world knowledge but poor instruction-following behavior.
- Explain Supervised Fine-Tuning (SFT) and why it's needed to turn a base (pretrained) model into an instruction-following assistant.
- Explain, at a mechanically honest level, RLHF and preference-based alignment (reward modeling + policy optimization) — enough to reason about it correctly, without needing to derive PPO from scratch.
- Explain parameter-efficient fine-tuning (LoRA) and why it's the practical, affordable option for engineers who will never pretrain a model from scratch.
- Reason clearly about the inference-time distinction between a base model and a chat/instruct model, and why this matters for prompt design in Part 3.
- Fine-tune a small open model on a custom dataset using LoRA, producing a genuinely useful, evaluable artifact.

---

## 2. Prerequisites

- Part 2, Module 6 (Attention & the Transformer Architecture) — you need the full forward-pass architecture solid before reasoning about how it gets trained at scale.
- Part 2, Module 3 (Backpropagation & Training) — gradient descent, loss functions, and overfitting/generalization intuition.
- Comfortable with the general shape of a supervised learning problem (inputs, labels, loss, optimizer) from Module 2/3.

---

## 3. Estimated Study Time

**7–9 hours** (2–3 hours theory, 3–4 hours implementation/fine-tuning, 1–2 hours exercises).

---

## 4. Difficulty

★★★★☆ (4/5) — Conceptually this module has fewer new mathematical mechanisms than Module 6, but it requires holding several distinct training phases in your head simultaneously and reasoning correctly about what each one changes and why — that's where the difficulty lives.

---

## 5. Why This Matters

You now understand exactly *what* a transformer computes on a forward pass. This module answers a question you've been deferring since Module 2: **where do the billions of parameters in `W_Q`, `W_K`, `W_V`, the feed-forward layers, and the embedding tables actually come from, and why does a "next word prediction" task produce a model that can write code, explain physics, and follow multi-step instructions?**

This matters immediately and practically, not just academically. As an AI engineer, you will almost never train a model from scratch (pretraining costs tens to hundreds of millions of dollars and is done by a handful of labs — OpenAI, Anthropic, Google, Meta). What you *will* do constantly is: (a) decide whether to use a general-purpose model as-is via prompting (Part 3), (b) fine-tune a smaller open model for a specific task when prompting isn't enough, or (c) understand *why* a model behaves the way it does — refuses certain requests, follows instructions well or poorly, hallucinates — all of which trace directly back to decisions made during pretraining and fine-tuning, not decisions you can override with a clever prompt alone.

There's also a direct, concrete parallel to your backend engineering intuition worth naming up front: **pretraining is like building and shipping a large, general-purpose platform** (think: a huge, generically-useful internal library or framework, trained on "everything") — expensive, done rarely, by a small team, optimized for broad general capability. **Fine-tuning is like configuring or lightly extending that platform for one specific application** — cheap, done often, by many teams, optimized for a narrow, specific outcome. You already understand this distinction instinctively from choosing when to build vs. configure vs. extend an existing framework; this module gives you the precise ML version of that same judgment call.

---

## 6. Theory

### 6.1 Pretraining: The Objective That Builds a World Model by Accident

**The task, precisely:** given a massive corpus of text (trillions of tokens, scraped from the web, books, code repositories, and more), train a decoder-only transformer (Module 6) to do exactly one thing: **predict the next token, given all previous tokens.** That's it. There are no labels beyond "the actual next word in real text that already exists" — this is called **self-supervised learning**, because the supervision signal (the correct answer) comes for free from the structure of the data itself, exactly the same way word2vec's "labels" in Module 4 were just the surrounding words in real sentences, requiring no human annotation.

**Why does this simple task produce broad capability?** Consider what it actually takes to predict the next token with high accuracy across a trillion-token corpus spanning news articles, novels, Wikipedia, Python code, mathematical proofs, and casual forum posts. To correctly predict the next token after "The mitochondria is the ___," the model needs to have learned biology facts. To correctly predict the next token in a half-finished Python function, it needs to have learned programming syntax and logic. To correctly complete "2 + 2 = ___," it needs arithmetic. **Next-token prediction, applied at sufficient scale and over sufficiently diverse data, forces the model to implicitly learn an enormous amount of world knowledge, reasoning patterns, and structure — not because anyone designed a "learn facts" objective, but because accurate prediction is, quite literally, impossible without that knowledge.** This is the exact same "the task forces the structure" argument from Module 4's distributional hypothesis and Module 6's attention specialization — it is, at this point, a running theme worth naming explicitly: **modern AI capability overwhelmingly comes from designing a training objective simple enough to apply at massive scale, and then letting scale do the rest, rather than from hand-engineering capability directly.**

**The result of pretraining is called a "base model."** A base model is extremely good at continuing text in a statistically plausible way — but critically, it has **no built-in notion of "being a helpful assistant."** If you prompt a base model with "What's the capital of France?", it might complete it with more questions ("What's the capital of Germany? What's the capital of Italy?") because that's a statistically very plausible continuation of a list-of-questions-style document it saw during training — it has no learned preference for "answer directly and helpfully" over any other statistically plausible continuation. This is the single most important practical fact in this module, and it directly motivates everything that follows.

### 6.2 Supervised Fine-Tuning (SFT): Teaching the Model to Behave Like an Assistant

**The fix for the base-model problem above:** take the pretrained base model and continue training it — with the same next-token-prediction mechanism, same architecture, same loss function — but now on a much smaller (thousands to low millions of examples, vs. trillions of tokens), carefully curated dataset of **(instruction, ideal response)** pairs, written or selected specifically to demonstrate the *behavior* you want: answering questions directly, following multi-step instructions, refusing genuinely harmful requests, formatting output as requested, and so on.

Mechanically, this is not a new algorithm — it's the exact same gradient descent over the exact same next-token-prediction loss you learned in Module 3 and saw applied at scale in 6.1. What's different is purely the **data distribution**: instead of "predict the next token in a random web page," it's "predict the next token in a demonstrated ideal assistant response, given an instruction." Because the model already has broad world knowledge and language competence from pretraining, SFT needs vastly less data and compute than pretraining — this is the exact same principle as transfer learning in any ML context: don't relearn everything from scratch, adapt an already-capable model to a specific behavior distribution.

**This is the first point in the pipeline where "prompt engineering" (Part 3) and "model behavior" genuinely diverge as separate concerns.** A model's *tendency* to be concise, to refuse certain requests, to format code in markdown blocks — these are shaped substantially by SFT data, not purely by whatever you type into the prompt at inference time. Understanding this will make you a better prompt engineer in Part 3: you'll know which behaviors are "trainable defaults you're working with or against" versus which are genuinely steerable via prompting alone.

### 6.3 RLHF and Preference-Based Alignment: Beyond "Imitate the Ideal Answer"

SFT has a real limitation: it only teaches the model to imitate the *specific* ideal responses humans wrote down. But often, judging *which of several possible responses is better* is much easier for a human than *writing* the single ideal response from scratch — think of how much easier it is to rank three draft emails by quality than to write the "perfect" one from a blank page. **Reinforcement Learning from Human Feedback (RLHF)** exploits exactly this asymmetry.

The process, precisely, in three stages:

**Stage 1 — Collect preference data.** Given a prompt, generate multiple candidate responses from the SFT model, and have human raters (or, increasingly, another well-calibrated AI model — "RLAIF," Constitutional AI-style approaches) rank or compare them (typically pairwise: "is response A or response B better?").

**Stage 2 — Train a reward model.** Using this preference data, train a separate model (often initialized from the same base architecture) whose job is: given a prompt and a response, output a single scalar score predicting how much a human would prefer this response. This is a genuinely standard supervised learning problem (very similar in spirit to the classification tasks from Module 2) — the "labels" are the human preference comparisons from Stage 1.

**Stage 3 — Optimize the original model against the reward model, via reinforcement learning.** Now, instead of training directly on human-written text (as in SFT), the model generates a response, the reward model scores it, and a reinforcement learning algorithm (commonly PPO — Proximal Policy Optimization) updates the model's parameters to make higher-reward responses more likely in the future. Critically, this stage typically includes a penalty term that keeps the fine-tuned model's behavior from drifting too far from the original SFT model's distribution (measured via KL-divergence) — without this, a model can learn to "game" the reward model with degenerate outputs that score artificially high but are actually low-quality or bizarre (a real, well-documented failure mode called "reward hacking," directly analogous to Goodhart's Law — "when a measure becomes a target, it ceases to be a good measure" — which you may already recognize from backend metrics/KPI design pitfalls).

**You do not need to be able to derive or implement PPO from scratch to be an effective AI engineer** — this is genuinely specialized research/training-infrastructure work done by a small number of teams at frontier labs. What you *do* need is the mechanically honest understanding above: alignment training is a real, deliberate, multi-stage process that shapes model behavior beyond raw next-token prediction, it's driven by human (or AI-assisted) preference judgments rather than single "correct" answers, and it's this stage — not pretraining, not SFT alone — most responsible for the specific "personality," refusal behavior, and helpfulness calibration you experience when using a production chat assistant.

**A brief, honest note on Constitutional AI and newer approaches:** Anthropic's Constitutional AI approach (and related methods used across the field) reduces reliance on large volumes of human preference labels by having a model critique and revise its own outputs against a written set of principles, then using *that* self-generated preference data to train the reward model — worth knowing as a named alternative to pure human-labeled RLHF, without needing full mechanistic depth here.

### 6.4 Parameter-Efficient Fine-Tuning: LoRA, and Why It's What You'll Actually Use

Full fine-tuning — updating *every single parameter* of a multi-billion-parameter model — requires storing full-precision gradients and optimizer state for every parameter, which for a modestly sized model can require far more GPU memory than most individuals or small teams have access to. **Low-Rank Adaptation (LoRA)** is the practical, dramatically cheaper alternative you will actually use.

**The core insight:** instead of updating a large weight matrix `W` (shape `d x d`) directly, LoRA freezes `W` entirely (no gradients computed for it at all) and instead learns a small *update* to it, expressed as the product of two much smaller matrices:

```
W_new = W_frozen + (A @ B)

where A is (d x r), B is (r x d), and r (the "rank") is small — e.g., 8, 16, or 64,
compared to d, which might be 4096 or larger.
```

Because `r << d`, the number of trainable parameters in `A` and `B` combined is a tiny fraction of `W`'s total parameter count (often under 1% of the original model's parameters need to be trained). The claim this rests on — and it's empirically well-supported for large pretrained models — is that **the *update* needed to adapt a large, already-capable pretrained model to a new task tends to have low "intrinsic rank,"** meaning it doesn't require the full expressive power of a dense `d x d` update to capture the necessary behavioral change. This is directly analogous to a compression argument you already have good intuition for from Module 5's tokenization discussion: you don't need to represent every possible update densely if the *actual* updates that matter live in a much smaller effective subspace.

**Practical consequence for you as an engineer:** LoRA fine-tuning of even a 7–13 billion parameter model is achievable on a single consumer or prosumer GPU, in hours rather than days, and produces a small "adapter" file (megabytes, not gigabytes) that can be loaded on top of the frozen base model at inference time — or even swapped in and out dynamically, letting one base model serve multiple fine-tuned "personalities" or task-specializations without needing multiple full copies of the model in memory. This is the exact tool you'll use in this module's fine-tuning project, and the same tool used constantly in real production AI engineering work whenever prompting alone (Part 3) isn't sufficient for a specific, narrow task.

### 6.5 Inference: What's Actually Different From Training

You've now covered three training phases. **Inference** — actually using a trained model to generate a response to a real prompt — is mechanically much simpler, and it's worth being precise about exactly what does and doesn't happen:

- **No gradient computation, no backpropagation, no weight updates.** Inference is a pure forward pass (Module 6's mechanism) through the frozen, already-trained network. This is why inference is dramatically cheaper than training, per token — no need to store activations for a backward pass, no optimizer state.
- **Generation is autoregressive and sequential, one token at a time**, exactly as you implemented in `toy-transformer`'s generation loop (Module 6, mini-project): the model computes a probability distribution over the next token, a token is sampled (or picked greedily) from that distribution, appended to the input, and the entire (now one-token-longer) sequence is fed back in to predict the *next* next token. This is precisely why longer generations take proportionally longer and cost proportionally more — it's a genuinely sequential, non-parallelizable-across-steps process at inference time, even though the underlying architecture parallelizes beautifully *across positions within a single forward pass* during training (this resolves the nuance flagged in Module 6's interview question 8 about parallelization).
- **Sampling strategy is a genuine, tunable design decision** — parameters like *temperature* (controls how "peaked" vs. "flat" the next-token probability distribution is before sampling — low temperature ≈ always pick the most likely token, high temperature ≈ more random/creative), *top-k* and *top-p (nucleus) sampling* (restrict sampling to only the most probable subset of tokens, to avoid occasionally sampling a wildly implausible token from the distribution's long tail) are all inference-time knobs you control directly via API parameters — a preview of Part 3's prompt/parameter engineering, and worth flagging now because these are *not* training decisions, they're choices you make every time you call the model.

### 6.6 Putting the Whole Pipeline Together

```
PRETRAINING                    SFT                          RLHF / Alignment              INFERENCE
─────────────                  ───                          ─────────────────              ─────────

Trillions of tokens,     →     Thousands–millions of   →    Preference comparisons,   →    A single frozen
next-token prediction,         curated (instruction,        reward model, RL             model, forward
self-supervised, no             ideal response) pairs,       (PPO) optimization,           pass only,
human labels needed              same loss mechanism,        keeps close to SFT             autoregressive
                                 much less data/compute       via KL penalty                 generation,
                                                                                              sampling params
     ↓                               ↓                             ↓                             ↓
"Base model" —              "Instruct/SFT model" —        "Chat/aligned model" —         What you actually
predicts plausible          follows instructions,          calibrated helpfulness,        call via API in
continuations, no            but not yet optimized          refusal behavior,              Part 3 — e.g.,
assistant behavior            for nuanced preference         personality shaped by          Claude, GPT-4,
                                                              human/AI feedback              production LLMs
```

Every model you will ever call via a production API (Part 3 onward) has been through all of these stages already, by the provider — your job as an AI engineer is almost always to work skillfully with the result (via prompting, Part 3) or, occasionally, to add one more small, targeted layer on top (via LoRA fine-tuning, this module's project) — essentially never to redo pretraining or full RLHF yourself.

---

## 7. Mental Models

1. **"A simple objective at massive scale beats a clever objective at small scale."** Next-token prediction is almost embarrassingly simple; it's the trillions of tokens of diverse data that force real capability to emerge.
2. **"Pretraining teaches knowledge; fine-tuning teaches behavior."** A base model *knows* a huge amount; SFT and RLHF teach it *how to act* on that knowledge as a helpful assistant.
3. **"Ranking is easier than writing — RLHF exploits that asymmetry."** Humans (or AI raters) comparing two responses is a cheaper, more scalable signal than humans writing the single ideal response every time.
4. **"When a measure becomes a target, it stops being a good measure."** Reward hacking in RLHF is the ML-training instance of Goodhart's Law — the same failure mode you already know from poorly designed backend KPIs and metrics.
5. **"You will almost never pretrain; you will occasionally fine-tune (cheaply, via LoRA); you will constantly prompt."** Calibrate your engineering effort accordingly.

---

## 8. Visual Explanation

**Diagram 1 — Base Model vs. Instruct/Chat Model, Same Prompt**

```
Prompt: "What's the capital of France?"

BASE MODEL (pretraining only) might continue:
  "What's the capital of France? What's the capital of Germany?
   What's the capital of Italy? ..."
  (statistically plausible continuation of a list-style document —
   no learned preference for "just answer the question")

INSTRUCT/CHAT MODEL (after SFT + RLHF) responds:
  "The capital of France is Paris."
  (directly, helpfully — this behavior was SHAPED by training data
   and preference optimization, not just prompting)
```

**Diagram 2 — The Three-Stage RLHF Pipeline**

```
SFT model generates          Human (or AI) rater          Reward model trained
several candidate     ──>    ranks/compares       ──>     to predict preference
responses to a prompt         the candidates                score from (prompt,
                                                              response) pairs
                                                                    │
                                                                    ▼
                                                          SFT model further
                                                          optimized (via PPO)
                                                          to maximize reward
                                                          model's score, with
                                                          a KL penalty keeping
                                                          it close to SFT
                                                          behavior (prevents
                                                          reward hacking)
```

**Diagram 3 — Full Fine-Tuning vs. LoRA**

```
FULL FINE-TUNING:
  Every parameter in every weight matrix is updated.
  Requires full gradient + optimizer state for the ENTIRE model.
  Memory cost: very high. Practical for large labs only.

LoRA:
  Original weight matrix W is FROZEN (no gradient computed for it).
  Two small matrices A (d x r) and B (r x d) are learned instead,
  where r << d.

     W_frozen  (d x d, untouched)
        +
     A (d x r) @ B (r x d)   <-- only these tiny matrices are trained
        =
     W_effective (used at inference time)

  Memory cost: a small fraction of full fine-tuning.
  Practical on a single GPU, in hours.
```

**Diagram 4 — Training vs. Inference, Compute Shape**

```
TRAINING (any phase):                    INFERENCE:
  Forward pass                             Forward pass ONLY
  + Backward pass (gradients)              (no backward pass, no
  + Optimizer step (update weights)         weight updates, ever)
  Expensive, done rarely                   Cheap per call, done constantly,
  (by the model provider)                  by every engineer building on
                                            top of the model (you)

  Processes MANY sequences in              Generates tokens ONE AT A TIME,
  parallel, many passes over data           autoregressively, feeding each
  (epochs)                                  new token back in as input
```

---

## 9. Recommended Resources

1. **[OpenAI — "Training language models to follow instructions with human feedback" (InstructGPT paper, 2022)](https://arxiv.org/abs/2203.02155)** — The paper that made the SFT → RLHF pipeline widely understood and adopted; read this as the primary source for sections 6.2–6.3 above.
2. **[Anthropic — "Constitutional AI: Harmlessness from AI Feedback" (2022)](https://arxiv.org/abs/2212.08073)** — Anthropic's own account of an alternative to pure human-labeled RLHF; directly relevant given the labs you're targeting for the platform-engineering track.
3. **[Hu et al. — "LoRA: Low-Rank Adaptation of Large Language Models" (2021)](https://arxiv.org/abs/2106.09685)** — The original LoRA paper; short, readable, and directly the algorithm you'll implement in this module's fine-tuning project.
4. **[Hugging Face — PEFT library documentation](https://huggingface.co/docs/peft/index)** — The production-grade library implementing LoRA and related parameter-efficient fine-tuning methods; this is the actual tool you'll use in the project below.
5. **[Andrej Karpathy — "State of GPT" (talk)](https://www.youtube.com/watch?v=bZQun8Y4L2A)** — An excellent, precise walkthrough of the full pretraining → SFT → RLHF pipeline from someone who's built all of it; a strong capstone resource for this entire module.
6. **[Hugging Face — "Illustrating Reinforcement Learning from Human Feedback (RLHF)"](https://huggingface.co/blog/rlhf)** — A clear, visual companion piece to the InstructGPT paper if the reward-model/PPO stage needs more repetition to click.

---

## 10. Exercises

1. **Base vs. instruct, empirically.** If you have access to both a base and an instruct/chat variant of the same open model family (e.g., via Hugging Face), prompt both with an identical direct question (not a chat-formatted prompt, just raw text) and compare the completions. Write two sentences on what you observe and connect it to section 6.1's explanation.
2. **Design an SFT dataset entry.** Write 3 example (instruction, ideal response) pairs you'd include in an SFT dataset for a hypothetical "customer support assistant for a food delivery app" fine-tuning task (a direct nod to your DoorDash interview prep context) — and explain, for each, what specific behavior you're trying to instill.
3. **Reward hacking, imagined.** Invent a plausible reward-hacking scenario for a hypothetical reward model that scores responses on "helpfulness + conciseness" — describe a degenerate model output that would score artificially high on this reward model while actually being low quality, and explain why the KL penalty in stage 3 of RLHF helps prevent this.
4. **LoRA rank trade-off.** Explain, in your own words, what you'd expect to happen to (a) fine-tuning quality/capability and (b) trainable parameter count and memory usage, as you increase the LoRA rank `r` from 4 to 64 to 256. At what point would you expect diminishing returns, and why?
5. **Temperature and sampling, hands-on.** Using any available LLM API or open model, generate completions for the same prompt at temperature 0 (or near-0), 0.7, and 1.5. Compare the outputs and connect the differences back to section 6.5's explanation of what temperature actually does to the probability distribution before sampling.
6. **Pipeline attribution.** For each of the following observed chat-model behaviors, identify which stage of the pipeline (pretraining, SFT, or RLHF) is most directly responsible, and justify your answer: (a) the model knows who wrote *Hamlet*, (b) the model answers a direct question directly instead of continuing it as a list, (c) the model consistently refuses to help write malware even when asked persistently and creatively, (d) the model's overall "tone" feels calibrated to be genuinely helpful rather than curt or evasive.

---

## 11. Mini-Project: `pipeline-attribution-lab`

A short, focused analysis project (not a training project — save the from-scratch training for the production project below) designed to cement the conceptual model from sections 6.1–6.3.

**Requirements:**
- Using an available base model and its corresponding instruct/chat variant from the same model family, design and run at least 8 varied prompts (factual recall, direct instructions, multi-step reasoning, a mildly ambiguous or borderline request) through both models.
- For each pair of completions, write a short structured note: what differs, and which training stage (pretraining knowledge vs. SFT instruction-following vs. RLHF-shaped refusal/tone calibration) most plausibly explains the difference.
- Produce a small summary table (prompt | base model behavior | instruct model behavior | most likely responsible stage) as your deliverable — this becomes a genuinely useful personal reference document for reasoning about model behavior for the rest of the handbook.

---

## 12. Production Project: `lora-finetune-toolkit`

This is a new, standalone but reusable artifact — the first project in the handbook that actually adapts a real pretrained model's weights, rather than only calling a model via API or building supporting infrastructure around one.

**What to build:**

1. **A reusable LoRA fine-tuning pipeline** (Python, using Hugging Face `transformers` + `peft` + `trl` or an equivalent stack) that takes: a base open model (something small enough to fine-tune on affordable hardware, e.g., in the 1–3B parameter range), a dataset of (instruction, ideal response) pairs in a simple JSONL format, and LoRA hyperparameters (rank, target modules, learning rate) as configuration — and produces a trained LoRA adapter as output.
2. **Curate a genuinely useful small SFT dataset** for a specific, narrow task relevant to your own interests — a strong, on-theme choice: a dataset of (question, ideal answer) pairs for backend/system-design interview question answering (directly reusable for your DoorDash prep), or a dataset teaching the model to respond in a specific structured format (e.g., always returning valid JSON matching a schema — directly previews Part 3's structured output content). Aim for at least 100–300 examples; quality and consistency matter far more than raw volume at this scale.
3. **Fine-tune, then rigorously evaluate the before/after difference**: run a fixed set of held-out test prompts (not seen during fine-tuning) through both the original base model and your LoRA-adapted version, and score the improvement — reuse the `eval-harness-preview` scaffolding from Part 0, Module 10 as the evaluation harness, extending it if needed to score against your specific task.
4. **Package the trained adapter for reuse**: save it in a way that can be loaded on top of the frozen base model at inference time (standard `peft` adapter save/load), and write a short README documenting exactly what the adapter does, what data it was trained on, and its measured before/after evaluation results.
5. **Add a cost/time log**: record how long fine-tuning took and on what hardware, and compare this concretely against the (well-documented, publicly available) cost of full fine-tuning or pretraining at the same model scale — a genuinely useful, memorable data point for your own future decision-making about when LoRA is (almost always) the right tool.

**Explicitly designed for extension:** `lora-finetune-toolkit` is the tool you'll reach for again any time later parts of this handbook (particularly Part 5's agents and Part 8's production systems) identify a narrow, well-defined task where prompting a general-purpose model isn't cost-effective or reliable enough, and a small, cheaply fine-tuned specialist model is the better engineering choice.

---

## 13. Practice & Interview Questions

1. Explain, precisely, why next-token prediction on internet-scale text produces a model with broad world knowledge, without anyone explicitly designing a "learn facts" objective.
2. What specific problem does Supervised Fine-Tuning solve that pretraining alone does not? Give a concrete example of base-model behavior that SFT is designed to fix.
3. Walk through the three stages of RLHF from memory: what data is collected, what's trained in each stage, and why the process ends with a KL-penalty-constrained optimization rather than unconstrained reward maximization.
4. What is "reward hacking," and how does it relate to Goodhart's Law? Give an original example, not one from this module's text.
5. Explain the core mathematical idea behind LoRA (low intrinsic rank of the necessary weight update) and why it makes fine-tuning dramatically cheaper than full parameter updates.
6. What computation happens during inference that does *not* happen during training, and vice versa? Be specific about gradients, optimizer state, and the autoregressive generation loop.
7. What does "temperature" control at inference time, mechanically? What would you expect to happen with temperature set to exactly 0?
8. If you were advising a startup with a narrow, well-defined text-classification-style task and a limited budget, would you recommend prompting a large general-purpose model, LoRA fine-tuning a smaller open model, or full fine-tuning/pretraining? Justify your answer with the cost/capability trade-offs from this module.
9. Why is a "base model" generally unsuitable to expose directly to end users in a consumer chat product, even though it has strong underlying language and knowledge capability?

---

## 14. Common Mistakes

- **Conflating "the model knows X" with "the model will tell you X helpfully."** Knowledge comes overwhelmingly from pretraining; helpful, direct, well-formatted delivery of that knowledge is substantially an SFT/RLHF behavior, not a given.
- **Assuming prompting can always fully override behavior shaped by SFT/RLHF.** Some behavioral tendencies are deep training-time defaults; prompting can steer *within* the range that training allows, but it isn't a substitute for retraining when the desired behavior is fundamentally outside that range.
- **Reaching for full fine-tuning by default instead of LoRA**, when LoRA is very often sufficient and dramatically cheaper — a real, common overengineering mistake once teams have GPU budget and forget to check whether they need it.
- **Treating temperature=0 as "the model has no randomness at all, guaranteed reproducible."** In practice, floating-point non-determinism and provider-side batching can still introduce small variability even at temperature 0 — a subtlety worth knowing before you build tests or systems that assume perfect reproducibility.
- **Designing an SFT or preference dataset without enough diversity or with inconsistent labeling standards.** Just like any supervised learning problem from Module 2, garbage or inconsistent training data produces a model that learns the *wrong* pattern confidently, not "no pattern" — a subtle trap because the resulting model will still look plausible on casual inspection.
- **Forgetting that RLHF-style optimization is trained against a *reward model*, not against literal human judgment at generation time.** The reward model is itself an imperfect proxy — all the usual "your model is only as good as your evaluation signal" caveats from Module 3 apply here too, one level up.

---

## 15. Debugging Exercise

You've fine-tuned a small open model using your `lora-finetune-toolkit` on a dataset of (backend interview question, ideal answer) pairs, hoping to produce a model that gives crisp, direct, well-structured interview answers. Training loss decreased nicely and the fine-tuning completed without errors. But when you evaluate the fine-tuned model on your held-out test prompts, it now confidently gives *wrong* technical answers with the same crisp, confident formatting you were hoping for — worse than the base model was before fine-tuning, which at least hedged appropriately on questions it wasn't sure about.

You're given a summary of your training data pipeline:

```
- 220 (question, answer) pairs, scraped from a mix of public interview-prep
  forum posts and personal notes
- No deduplication or source-quality filtering was applied
- No held-out validation split was used during training — all 220 examples
  were used for training, and training was run for 8 epochs
- Final training loss: very low, near-zero
```

**Your task:** Diagnose the likely root cause(s) of this failure using concepts from this module and Module 3 (not just "the data was bad" — be specific about the *mechanism*). Consider: what does "near-zero training loss after 8 epochs on only 220 examples with no validation split" imply, in the exact same vocabulary you used for overfitting/generalization in Module 3? Why would an overfit fine-tune specifically produce *confident, well-formatted, wrong* answers, rather than just "worse" answers in some vaguer sense? Propose at least two concrete fixes to your `lora-finetune-toolkit` pipeline that would have caught this problem before you shipped a broken evaluation result.

---

## 16. Checklist

- [ ] I can clearly distinguish pretraining, SFT, RLHF, and inference — what data each uses, what each optimizes, and what compute profile each has.
- [ ] I can explain why next-token prediction at scale produces broad world knowledge as a side effect, using the same "the objective forces the structure" logic from Modules 4 and 6.
- [ ] I can explain the three stages of RLHF from memory and explain reward hacking via the Goodhart's Law analogy.
- [ ] I can explain LoRA's core insight (low intrinsic rank of the necessary update) and why it's the practical fine-tuning tool for individual engineers.
- [ ] I understand precisely what is and isn't happening computationally during inference vs. training, including the autoregressive generation loop and sampling parameters like temperature.
- [ ] I've completed `pipeline-attribution-lab` and can point to concrete before/after examples illustrating base vs. instruct model behavior.
- [ ] I've built and run `lora-finetune-toolkit` end to end: curated a dataset, fine-tuned a real small model, and rigorously evaluated the before/after difference with a proper held-out test set.
- [ ] I can answer all 9 interview questions without notes.

---

## 17. Summary

A production LLM is the product of a multi-stage pipeline, not a single training run: pretraining on internet-scale text via simple next-token prediction produces a "base model" with broad, emergent world knowledge but no assistant-like behavior; Supervised Fine-Tuning on a much smaller, curated dataset of ideal (instruction, response) pairs teaches direct, instruction-following behavior; RLHF (or Constitutional-AI-style alternatives) further shapes helpfulness, tone, and refusal behavior by optimizing against a learned reward model built from human or AI preference judgments, constrained to avoid reward hacking. As an engineer, you will almost never redo pretraining or full RLHF — but you will regularly reach for LoRA, a parameter-efficient fine-tuning method that exploits the low intrinsic rank of task-specific weight updates to make fine-tuning affordable on modest hardware. Inference itself is a comparatively simple, purely-forward-pass, sequential, autoregressive process, with genuinely tunable knobs (temperature, top-k/top-p) that belong to inference time, not training time. You put all of this into practice by attributing specific model behaviors to specific pipeline stages, and by fine-tuning a real small model end-to-end with `lora-finetune-toolkit`, complete with rigorous before/after evaluation.

---

## 18. Next Steps

**Next module: Part 2, Module 8 — "Evaluating Models."** You now understand how models are trained and adapted — but you still need a rigorous answer to the question every one of your production decisions will depend on: *is this model, or this fine-tune, or this prompt change, actually good, and better than the alternative?* Module 8 covers benchmark design and its pitfalls, the LLM-as-judge pattern, task-specific evaluation harnesses, and the statistical traps (contamination, overfitting to benchmarks, small-sample noise) that make evaluation one of the most commonly underrated skills in applied AI engineering — directly extending `eval-harness-preview` (Part 0, Module 10) and the evaluation work you just did in this module's production project into a full, general-purpose evaluation framework you'll reuse for the rest of the handbook.

---

Reply "continue" for Module N, or flag anything to go deeper on.
