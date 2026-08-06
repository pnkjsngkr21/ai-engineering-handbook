# Part 3, Module 1: Prompting & Prompt Engineering

> **Part 3 — LLM Engineering.** Part 2 answered "how do these models work?"
> Part 3 answers "how do I build reliable, production-grade systems on top
> of them?" Every module in this part assumes the mechanistic grounding
> from Part 2 and refuses to let anything be explained as folklore. This
> module starts that habit at the very first layer of the stack: the
> prompt itself.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain the anatomy of a well-structured prompt (system, user,
   assistant, and — where supported — developer roles) and how each maps
   onto the raw token sequence the model actually sees.
2. Derive *why* few-shot prompting, chain-of-thought, and role/system
   prompts work, in terms of the autoregressive next-token mechanism
   (Part 2, Module 6) and SFT/RLHF-shaped priors (Part 2, Module 7) —
   not as folklore ("the model likes examples") but as a mechanistic
   consequence.
3. Distinguish zero-shot, few-shot, and chain-of-thought prompting, and
   choose correctly between them based on task type and model class
   (standard vs. reasoning models, Part 2 Module 9).
4. Write prompts that are robust to the model's training-induced defaults
   (verbosity, hedging, refusal patterns, formatting habits) rather than
   fighting them blindly.
5. Identify prompt injection at a conceptual level: what it is
   mechanistically (the model cannot distinguish "instructions" from
   "data" in its input stream), why it's not fully solvable by prompting
   alone, and what mitigations exist at this layer (more in Part 5).
6. Build a real, working multi-provider prompt execution layer by wiring
   the Anthropic and OpenAI SDKs into `llm-client-core` (Part 1, Module
   2), replacing the mock/stub adapters that have stood in for it since
   Part 1.

---

## 2. Prerequisites

- **Part 2, Module 6** (Attention & the Transformer Architecture) — you
  need the autoregressive, next-token-at-a-time generation mechanism
  fresh in your head. Everything in Section 6 of this module leans on it.
- **Part 2, Module 7** (Training vs. Inference, and Fine-tuning) — you
  need to remember the difference between a base model and an
  instruction-tuned/RLHF model, and what that process actually optimizes
  for.
- **Part 2, Module 9** (Reasoning Models) — needed to understand why
  chain-of-thought prompting is close to a no-op on reasoning models but
  essential on standard models.
- **Part 1, Module 2** (`llm-client-core`) — you'll be extending this
  package directly. Re-familiarize yourself with its `LLMClient`
  interface, the `Adapter` pattern used for provider abstraction, and
  where the current mock adapter lives.
- Comfort reading API docs and making authenticated HTTP calls (Part 0,
  Module 4 and Module 9).

---

## 3. Estimated Study Time

**8–10 hours** (3–4 hours theory/reading, 4–6 hours hands-on
exercises + mini-project + production project).

---

## 4. Difficulty

★★★☆☆ (3/5)

The concepts are not mathematically hard — there's no new architecture to
learn. The difficulty is entirely in *precision of thought*: prompting is
easy to do sloppily and get away with it 80% of the time, which is exactly
what makes bad prompting habits so hard to unlearn later. This module is
graded on rigor, not cleverness.

---

## 5. Why This Matters

Every AI engineering job — freelance or FAANG-adjacent — routes through
prompting at some layer, even once you're building agents, RAG systems,
or fine-tuning pipelines. It is the cheapest, fastest, most reversible
lever you have to change model behavior, which makes it the correct first
tool to reach for and the most abused one when engineers skip
understanding *why* it works.

Two failure modes dominate in practice, and both come from treating
prompting as folklore instead of mechanism:

- **Cargo-culting.** Copying a "magic phrase" from a blog post ("You are
  an expert... Take a deep breath and think step by step...") without
  understanding what it does, so when it stops working on a new model or
  a new task, there's no principled way to debug it.
- **Fighting the model instead of steering it.** Treating unwanted
  outputs (hedging, refusals, verbosity, wrong format) as bugs to route
  around with more forceful language, instead of recognizing them as
  emergent properties of the SFT/RLHF process (Module 7) that respond
  predictably to specific structural changes.

For your specific trajectory: DoorDash-style interviews increasingly probe
"how would you design a prompt/system for X" as a proxy for practical AI
judgment, and the platform-engineering track at labs like Anthropic and
OpenAI expects you to reason about prompting from the same mechanistic
level as the researchers who trained the models — folklore explanations
don't survive that bar.

---

## 6. Theory

### 6.1 What a prompt actually is: the token-sequence view

Strip away the API's JSON structure and a "prompt" is just this: a
sequence of tokens that becomes the fixed prefix the model conditions on
before it starts autoregressively predicting the next token, one at a
time, exactly as you built in `toy-transformer` (Part 2, Module 6). There
is no special "instruction mode" at the mechanism level — the model does
not have a separate code path for "look, this part is an instruction."
Everything is just prefix.

This is the single most important mental shift for this module. When you
send:

```json
{
  "system": "You are a helpful assistant that only responds in JSON.",
  "messages": [{"role": "user", "content": "What's the capital of France?"}]
}
```

the provider's chat template concatenates this into one token sequence
that looks structurally like:

```
<system>You are a helpful assistant that only responds in JSON.</system>
<user>What's the capital of France?</user>
<assistant>
```

and the model then predicts tokens starting right after `<assistant>`,
one at a time, each new token appended to the sequence and re-fed as
context for the next prediction (this is exactly the loop you implemented
by hand in `toy-transformer`'s sampling code). The `system`/`user`/
`assistant` tags aren't a separate mechanism — they're special tokens (or
token sequences) that the model learned, during SFT (Module 7), to treat
as strong behavioral signals. That's it. That's the whole trick.

Once you internalize this, several things that look mysterious become
obvious:

- **Why prompt injection is hard to fully prevent** (Section 6.5): the
  model sees one token stream. If untrusted data is concatenated into
  that stream, the model has no ground-truth signal that "this span
  came from a document, not from the person who's allowed to give
  instructions" beyond whatever soft cues (role tags, formatting) you
  give it — and those are just more tokens, not enforcement.
- **Why word order and proximity matter**: instructions closer to the
  point of generation (i.e., later in the prefix) have a stronger,
  more direct influence, because the model's attention (Module 6) can
  attend to any prior token, but recency and structural salience still
  measurably affect how strongly a given span shapes the next-token
  distribution — this is why "repeat the key instruction right before
  the user's final question" is a real, mechanistically-grounded
  technique, not superstition.
- **Why formatting instructions ("respond only in JSON") sometimes get
  ignored**: the model is doing next-token prediction under a learned
  distribution shaped by SFT examples. If your instruction is
  contextually unusual (deep in a large context, phrased atypically
  compared to the model's SFT-formatting-instruction training data), it
  competes with a strong prior toward "normal assistant response
  formatting" and can lose.

### 6.2 Why few-shot examples work (in-context learning, mechanistically)

You know from Module 6 that attention lets every token attend to every
prior token. Few-shot prompting exploits this directly: by placing
several `(input, output)` example pairs in the prefix, you give the model
concrete token patterns to pattern-match against when generating the
final output. This is called **in-context learning (ICL)**, and it does
not update any weights — nothing is "learned" in the training sense.
Instead:

- The model has, during pretraining, seen enormous quantities of text
  containing repeated patterns, enumerations, and analogical structures
  (documentation with worked examples, Q&A pairs, tables, etc.).
- Pretraining on next-token prediction over this data induces a genuine
  in-context pattern-completion capability: given several examples of a
  mapping, the attention mechanism can attend back to the example
  outputs when predicting the new output, effectively performing a soft,
  implicit form of pattern matching over the visible context.
- Each additional consistent example narrows the plausible next-token
  distribution further, the same way giving more constraints narrows a
  search space — this is why 2–3 well-chosen examples often help far
  more than 0, but 10 mediocre ones can help less than 3 excellent ones
  (more examples per se isn't the lever; consistency and information
  content are).

**Zero-shot** relies entirely on the SFT/RLHF-induced prior of "how an
assistant should respond to an instruction like this," with no
demonstration. This works well when the task closely resembles the
distribution of instructions the model was fine-tuned on (this is most
"normal" tasks, which is why zero-shot is a fine default). It works
poorly when the task has an unusual output format or an unusual mapping
the model hasn't internalized a default behavior for — that's exactly
when few-shot earns its keep, because you're supplying the missing
distributional information directly in-context instead of hoping the
prior generalizes correctly.

**Practical decision rule** (grounded in the above, not folklore):

| Situation | Use |
|---|---|
| Common task, standard output shape (summarize, translate, explain) | Zero-shot |
| Unusual or strict output format (a bespoke schema, a specific tone/style not well-represented in typical SFT data) | Few-shot |
| Task requires disambiguating between several plausible interpretations | Few-shot (examples pin down *which* interpretation) |
| Task is genuinely novel/analogical (no SFT precedent) | Few-shot, and expect to iterate on example choice |

### 6.3 Why chain-of-thought (CoT) works — and why it's not magic

Recall from Module 9: a standard (non-reasoning) model produces its final
answer via the same one-token-at-a-time autoregressive process, with a
**fixed amount of computation per generated token** — the forward pass
through the network is the same size regardless of how "hard" the token
is to predict. This is the key mechanistic fact that explains CoT.

If you ask a standard model to jump straight to a multi-step arithmetic
or logical answer, it must produce the *entire* answer's worth of
implicit computation compressed into the forward passes for a small
number of output tokens — there is nowhere else for that computation to
happen. Chain-of-thought prompting ("think step by step," or few-shot
examples that show intermediate reasoning) works because **generated
tokens become part of the context for predicting the next token**. By
inducing the model to first emit intermediate reasoning tokens, you are
literally giving the model additional forward passes — additional
computation — to work with before it has to commit to a final answer.
Each reasoning token gets its own full forward pass through every layer,
and each subsequent token can attend back to all of it. CoT isn't
persuading the model to "think harder" in some abstract sense; it is
mechanistically expanding the amount of computation available to solve
the problem, by trading output-length for compute.

This directly explains two things you should already suspect from
Module 9:

- **Why CoT gives large gains on standard models for multi-step
  tasks** (math, multi-hop logic, planning) — these are exactly the
  tasks where the "right answer" cannot be produced in the fixed compute
  budget of a single next-token step, but can be decomposed into a chain
  of steps each individually within that budget.
- **Why CoT gives little-to-no additional benefit on reasoning models**
  (Module 9) for the same tasks — reasoning models already generate an
  extended, RL-trained deliberation trace before answering, so
  instructing them to "think step by step" is asking for something
  they're already architecturally/behaviorally doing by default. It can
  still help slightly for output *formatting* of the reasoning, but the
  compute-expansion trick is already baked in.

**Corollary you should internalize**: CoT is not "explain your reasoning
for the user's benefit" — that's a side effect. Its actual function is
computation expansion. This is why prompting for CoT with an
instruction like "briefly explain your answer" (which caps the reasoning
tokens short) recovers much less of the benefit than "think through this
step by step before answering" (which doesn't artificially truncate the
computation).

### 6.4 Why system prompts, personas, and "you are an expert" work

A system prompt does not activate some separate "system mode." It's
simply a segment of the prefix that the model's SFT/RLHF training taught
it to weight heavily as defining the *role and constraints* of the
assistant turn (Module 7). Two real mechanisms are at play, and it's
worth being precise about which is which:

1. **Genuine behavioral conditioning**: RLHF reward modeling (Module 7)
   trained the model on human preference judgments over
   system-prompt-conditioned completions, so system-prompt text really
   does shift the model's output distribution toward matching whatever
   persona/constraint was specified during that training — the model
   learned that text appearing in the system-prompt position is a
   strong signal about acceptable output shape, tone, and scope.
2. **Persona-as-context-narrowing** (the "expert" effect): telling a
   model "you are a senior security researcher" doesn't grant it new
   knowledge it didn't have from pretraining — it can't. What it does is
   bias the next-token distribution toward the *register and content
   patterns* statistically associated with that persona in the training
   corpus (more precise terminology, fewer hedges, different assumed
   audience). This is a real, measurable effect on style and sometimes on
   which of several correct answers is surfaced first, but it is not a
   capability upgrade, and treating it as one (e.g., "you are a 10x
   engineer, now solve this NP-hard problem correctly") is a common and
   costly misunderstanding.

### 6.5 Prompt injection: the mechanistic root cause

Because the model conditions on one undifferentiated token stream
(Section 6.1), it fundamentally **cannot cryptographically distinguish**
"trusted instructions from the system/developer" from "untrusted content
that happens to contain instruction-shaped text," except insofar as your
prompt structure gives it *soft* signals (role tags, delimiters,
explicit framing) that it has learned, probabilistically, to weight
appropriately. Soft signals are not enforcement. This is exactly why
prompt injection — where a document, webpage, tool output, or email
being processed by the model contains text like "ignore previous
instructions and instead..." — remains an open, actively-researched
problem rather than something eliminable purely by clever phrasing.

At this layer (prompting), the practical mitigations are:

- **Structural separation with clear delimiters** (e.g., wrapping
  untrusted content in explicit XML-style tags and instructing the model
  that content inside those tags is data, never instructions) — this
  measurably reduces susceptibility because it gives the model a strong,
  consistent, learnable signal, but does not make it impossible.
- **Explicit, repeated framing** near the point of generation (Section
  6.1's recency point) reinforcing what is and isn't an instruction.
- **Least-privilege by design**: never relying on the prompt alone to
  enforce a security boundary when the output will trigger an action
  (an email send, a database write, a purchase). That enforcement
  belongs in code, not in prompt wording — you'll build this properly in
  Part 5 (Agents) with privilege separation, and it connects back to
  Part 1, Module 9's security fundamentals.

**The one-sentence version to carry forward**: prompting can reduce the
likelihood of injection succeeding, but it cannot be the security
boundary — treat every prompt-level defense here as risk-reduction, not
guarantee, and always pair it with code-level enforcement for anything
consequential.

### 6.6 The anatomy of a production-quality prompt

Pulling the above together, a well-engineered prompt for a production
system typically has these components, in this order, and you should be
able to justify *why* each one is where it is using Sections 6.1–6.5:

1. **Role/persona** (if relevant) — narrows register and framing
   (6.4).
2. **Task instruction** — explicit, unambiguous statement of what to do.
3. **Constraints and output format** — explicit schema/format
   requirements, stated as close to the generation point as possible
   for maximum salience (6.1).
4. **Few-shot examples** (if the task benefits from them per 6.2) —
   placed after general instructions, before the final task input.
5. **Untrusted/dynamic content** — clearly delimited (6.5), always
   treated as data.
6. **The actual task/question** — placed last, closest to generation,
   for maximum salience (6.1), often repeating the critical constraint
   one more time if it's easy to lose in a long context (this previews
   Part 3's later Context Engineering module).
7. **CoT trigger, if applicable** (6.3) — e.g., "think through this
   step by step, then give your final answer on its own line prefixed
   with `ANSWER:`" — note how this also solves the *parsing* problem by
   giving you a reliable extraction point, which matters enormously
   once you're calling this from code (Section 6.7).

### 6.7 From prompting to engineering: why this needs to live in code, not a chat window

Everything above describes how to get one good response from a model in
a chat interface. **Prompt engineering as an engineering discipline**
means treating prompts as versioned, tested, parameterized artifacts
that live in your codebase, get evaluated (via `eval-framework`, Part 2
Module 8) exactly like any other piece of logic, and get executed through
a real client abstraction rather than copy-pasted between provider
consoles. This is precisely why this module's production project wires
real provider SDKs into `llm-client-core`: from this point forward in the
handbook, "prompting" and "calling the API correctly, with proper
adapters, error handling, and evaluability" are the same skill.

---

## 7. Mental Models

1. **"A prompt is a prefix, not an instruction channel."** Everything the
   model does is next-token prediction conditioned on the whole token
   stream — instructions only work because SFT/RLHF taught the model to
   weight certain positions and patterns heavily, not because there's a
   separate command interpreter.
2. **"Chain-of-thought is bought compute, not persuasion."** Every
   reasoning token is a full forward pass the model gets to use before
   committing to an answer — CoT works by expanding the compute budget,
   not by making the model "try harder."
3. **"A persona changes the model's register, not its knowledge."**
   "You are an expert" shifts style and which correct answer surfaces
   first; it does not grant new capability the pretraining didn't
   already encode.
4. **"Prompts reduce injection risk; code enforces security."** Never
   let prompt wording be the only thing standing between untrusted input
   and a consequential action.

---

## 8. Visual Explanation

**Diagram 1 — The token-stream view of a "structured" prompt**

```
┌─────────────────────────────────────────────────────────┐
│  RAW TOKEN SEQUENCE THE MODEL ACTUALLY SEES              │
│                                                           │
│  [SYS] You are a precise, concise coding assistant. [/SYS]│
│  [USER] Refactor the function below for readability.     │
│         <code>...</code>                                 │
│         Respond only with the refactored code.     [/USER]│
│  [ASST] ▸▸▸ generation starts here, one token at a time  │
└─────────────────────────────────────────────────────────┘
        ▲                                    ▲
        │                                    │
   learned strong prior              closest to generation
   from SFT: "this defines           point → highest salience
   role/constraints"                 for the final instruction
```

**Diagram 2 — Why CoT expands compute (standard model)**

```
NO CoT:
[.......... long prompt ..........][ANSWER]
                                       ▲
                          one forward pass must encode
                          the entire multi-step solution

WITH CoT:
[.......... long prompt ..........][step 1][step 2][step 3][ANSWER]
                                       ▲       ▲       ▲       ▲
                                  each step gets its own full forward pass;
                                  ANSWER's forward pass can attend to all of them
```

**Diagram 3 — Few-shot as narrowing the next-token distribution**

```
Zero-shot:           P(output | task instruction)
                     → broad distribution over "plausible assistant answers"

Few-shot (k examples): P(output | task instruction, ex1, ex2, ex3)
                     → distribution sharply conditioned toward the
                       demonstrated input→output mapping
                       (narrows with each consistent example)
```

---

## 9. Recommended Resources

1. **Anthropic — "Prompt engineering overview" and the full Prompt
   Engineering guide** (docs.claude.com) — the canonical, model-vendor
   source for Claude-specific technique, written by the team that also
   trained the model; read this first since you'll be building against
   the Anthropic API directly in this module's production project.
2. **Anthropic — "Let Claude think" / chain-of-thought guidance**
   (docs.claude.com) — directly reinforces Section 6.3's mechanism with
   vendor-specific implementation guidance and worked examples.
3. **OpenAI — "Prompt engineering" guide** (platform.openai.com/docs) —
   read this second, specifically to notice where OpenAI's and
   Anthropic's guidance *agree* (that's signal about real underlying
   mechanism) versus where they diverge (that's signal about
   model-specific SFT differences, not universal law).
4. **Anthropic — "Prompt injection" documentation / research posts**
   (anthropic.com) — go directly to the vendor's own framing of this
   risk rather than secondary blog posts; this is the most accurate
   picture of current mitigations and their limits (Section 6.5).
5. **Wei et al., "Chain-of-Thought Prompting Elicits Reasoning in Large
   Language Models" (2022)** — the original CoT paper. Read the
   experimental results section specifically to see the standard-model
   task-type pattern (large gains on multi-step arithmetic/symbolic
   reasoning, minimal gains on tasks solvable in one step) that
   Section 6.3 explains mechanistically.
6. **Anthropic API Reference — Messages API** (docs.claude.com/en/api)
   — you need this open while building the production project; pay
   close attention to the `system` parameter (top-level, not a message
   role, in Anthropic's API — a real structural difference from
   OpenAI's chat-completions format you must handle in your adapter).
7. **OpenAI API Reference — Chat Completions / Responses API**
   (platform.openai.com/docs/api-reference) — same reason as above,
   for the OpenAI adapter; note where `system`/`developer` roles differ
   from Anthropic's structure.

---

## 10. Exercises

1. **Token-stream tracing.** Take any three-message conversation
   (system + user + assistant) you'd send to the Anthropic API. Write
   out, by hand, your best reconstruction of what the actual
   concatenated token-level prefix looks like (you don't need the real
   special tokens — approximate with clear markers). Identify exactly
   where in that sequence you'd insert a new constraint for maximum
   salience, and justify why using Section 6.1.
2. **Zero-shot vs. few-shot, empirically.** Pick a task with a slightly
   unusual output format (e.g., "extract entities as a Python list of
   tuples `(entity, type, confidence_1_to_5)`"). Run it zero-shot against
   a real model, then again with 3 few-shot examples. Record and compare
   the raw outputs. Explain any difference (or lack of difference) using
   Section 6.2 — don't just say "few-shot worked better," explain
   *why* in terms of distributional narrowing.
3. **CoT compute-expansion, empirically.** Take a genuinely multi-step
   arithmetic or logic problem. Run it against a standard (non-reasoning)
   model twice: once demanding "answer only, no explanation," once with
   "think step by step, then give your final answer." Compare accuracy
   across 5–10 different problem instances. Then run the same experiment
   against a reasoning model. Explain the pattern using Section 6.3.
4. **Persona effect, isolated.** Ask a factual question with a
   "you are a Nobel laureate physicist" persona and again with no
   persona at all. Confirm the *content* correctness is unchanged but
   identify concretely what changed in register/style. Write two
   sentences distinguishing what the persona did and did not do,
   grounded in Section 6.4.
5. **Injection, conceptually.** Write out (in a doc, not executed against
   anything with real privileges) a short example of a document that
   contains an embedded prompt-injection attempt, and then write the
   delimiter-based mitigation prompt structure you'd use to reduce (not
   eliminate) its success rate. Explicitly note, in one sentence, what
   this mitigation does *not* protect against.

---

## 11. Mini-Project: `prompt-lab`

A small, self-contained CLI tool (not yet integrated with
`llm-client-core`) for **empirically testing prompting techniques**
side-by-side against a real model, producing a comparison report.

**Requirements:**

- Accepts a task definition (a base instruction + optional few-shot
  examples + optional CoT trigger) and a list of test inputs.
- Runs each test input under multiple prompt variants (zero-shot,
  few-shot, few-shot+CoT) against a single provider (Anthropic, using a
  simple direct SDK call — this is intentionally throwaway/standalone,
  distinct from the production project).
- Outputs a comparison table: variant, input, output, and (if you supply
  expected answers) a simple correctness check.
- Logs token counts per variant (you already have tokenizer tooling from
  `embedding-service`, Part 2 Module 5 — reuse it here to make the
  cost/technique tradeoff visible, not abstract).

**Why this is scoped small and throwaway:** the goal is to build direct,
hands-on intuition for Sections 6.2–6.3 before you formalize anything.
Do not over-engineer this — it should take at most 1–2 hours. The
production project below is where the real, reusable engineering
happens.

---

## 12. Production Project: Real Provider Adapters in `llm-client-core`

This is the module where **`llm-client-core` (Part 1, Module 2) stops
being a well-designed shell around a mock and becomes a genuinely
functioning, production-usable component.** Every module for the rest of
Part 3 will call through this client.

### 12.1 What exists already

From Part 1, Module 2, you have:

- An `LLMClient` interface (the target abstraction).
- A `ProviderAdapter` pattern (Adapter design pattern) with a mock/stub
  adapter used for testing the surrounding architecture without real API
  calls.
- A `LLMClientFactory` (Factory pattern) that constructs the right
  adapter based on config.
- Decorator-based cross-cutting concerns (likely retry/logging wrappers)
  from the same module.

### 12.2 What you're building now

1. **`AnthropicAdapter`** implementing `ProviderAdapter`, wrapping the
   real Anthropic Messages API. Handle:
   - The `system` parameter as a **top-level field**, not a message
     (a real structural difference from OpenAI — get this right, it's a
     common integration bug).
   - Streaming and non-streaming response modes (return both through a
     unified interface — you'll need streaming properly working before
     Part 3's later latency/streaming module, but implement it now while
     the adapter is fresh).
   - Token usage extraction from the response (`usage.input_tokens`,
     `usage.output_tokens`) surfaced through your existing
     `observability-stack` (Part 1, Module 4) metrics, not silently
     dropped.
   - Rate limit and transient-error handling using your existing
     Decorator-based retry logic from Part 1, Module 2 — don't
     reimplement retry logic ad hoc inside the adapter.

2. **`OpenAIAdapter`** implementing the same interface against OpenAI's
   API, handling the `system`/`developer` role differences and the same
   usage/streaming/retry requirements.

3. **A `PromptTemplate` abstraction** (new, small, but important) that
   encodes Section 6.6's anatomy as a reusable, parameterized structure
   — not string concatenation scattered through call sites. At minimum:
   a way to define role/persona, task instruction, few-shot examples,
   a delimited-content slot for untrusted/dynamic input (Section 6.5),
   and an optional CoT trigger, then render it correctly per-provider
   (since system-prompt placement differs between providers, per 12.2.1
   above).

4. **Golden-path integration test** using `eval-framework` (Part 2,
   Module 8): run the same `PromptTemplate` + task against both real
   adapters and confirm both produce parseable, schema-valid output (this
   is your first real, non-toy use of `eval-framework` against live
   provider calls rather than toy models — treat it as the template for
   every subsequent Part 3 module's evaluation).

### 12.3 What this project explicitly sets up for later modules

- **Part 3, Module 2 (Structured Outputs)** will extend `PromptTemplate`
  and the adapters with JSON-mode/schema-validated output parsing.
- **Part 3, later modules** (tool calling, memory, caching, routing) all
  assume `AnthropicAdapter`/`OpenAIAdapter` are real and working —
  everything from here is additive to this artifact, not a new one.
- **Part 3's capstone** (production conversational AI service) is, in
  large part, `llm-client-core` fully matured plus everything Part 3
  adds on top.

### 12.4 Definition of done

- Both adapters make real, successful calls against live APIs (use your
  own API keys; keep them out of source control per Part 1, Module 9's
  security fundamentals).
- Retry/backoff, streaming, and usage-metric emission all verified with
  tests (unit tests with mocked HTTP responses for the retry/error paths
  — you don't want flaky tests dependent on live rate-limit triggering).
- `PromptTemplate` renders correctly for both providers from a single
  definition, verified by a test that asserts on the rendered prefix
  structure for each.
- `eval-framework` integration test passes against both live providers.

---

## 13. Practice & Interview Questions

1. Explain, mechanistically, why chain-of-thought prompting improves
   accuracy on multi-step reasoning tasks for a standard (non-reasoning)
   model, but often does not for a dedicated reasoning model. Do not use
   the word "thinking" as an explanation — describe the compute
   mechanism (Section 6.3).
2. A teammate says: "I told the model 'you are a world-class security
   researcher' and now it finds more vulnerabilities." What's the most
   likely actual effect this had, and what's a wrong conclusion someone
   might draw from this result?
3. Why can't prompt-level defenses (delimiters, explicit framing)
   *guarantee* protection against prompt injection? Answer at the
   mechanism level (Section 6.5), not "because attackers are creative."
4. You have a task with a genuinely novel, bespoke output schema that has
   no obvious precedent in typical instruction-tuning data. Would you
   reach for zero-shot or few-shot first, and why, in terms of what
   zero-shot is actually relying on?
5. Design the prompt structure (not the full text, just the
   section-by-section anatomy per 6.6) for a system that summarizes
   untrusted, user-uploaded documents and must never follow instructions
   contained within those documents. Identify the one thing this
   structure reduces risk of, and the one thing it does *not*
   guarantee.
6. In the Anthropic Messages API, where does the system prompt live
   structurally, and why does getting this wrong in a multi-provider
   adapter silently produce worse — not erroring — results?
7. Explain why "more few-shot examples is always better" is false, using
   Section 6.2's distributional-narrowing framing (hint: think about what
   happens when your examples are inconsistent with each other).

---

## 14. Common Mistakes

- **Treating the system prompt as a security boundary.** It's a strong
  behavioral prior, not an access-control mechanism (Section 6.5) —
  never gate a genuinely consequential action on "the system prompt told
  it not to."
- **Adding CoT instructions to a reasoning model "just in case."** At
  best inert, at worst it can truncate or malform the model's native
  extended reasoning trace depending on how it's phrased — know which
  model class you're talking to (Part 2, Module 9) before choosing.
- **Treating persona prompts as capability upgrades.** "You are a 10x
  engineer" does not make an underlying model better at a task it
  genuinely can't do; conflating register change with capability change
  leads to false confidence in production.
- **Inconsistent few-shot examples.** Examples that subtly contradict
  each other in format or logic narrow the distribution toward
  *inconsistency*, often producing worse outputs than zero-shot — verify
  your examples agree with each other before assuming more is better.
- **Hardcoding provider-specific prompt structure into call sites**
  instead of behind the `PromptTemplate`/adapter abstraction — this is
  exactly the kind of thing Part 1's Clean Architecture module (Part 1,
  Module 1) exists to prevent, and it will make every later Part 3
  module (structured outputs, tool calling, routing) painful to retrofit.
- **Silently swallowing token usage.** If your adapter doesn't surface
  `usage` through observability, you cannot debug cost or make routing
  decisions later in Part 3 — instrument it now while the adapter is
  being written, not retroactively.

---

## 15. Debugging Exercise

Your `AnthropicAdapter` and `OpenAIAdapter` both implement the same
`PromptTemplate`-based call for a document-summarization task. A teammate
reports: "The Anthropic version keeps producing much longer, more
hedge-y summaries than the OpenAI version for the exact same input text,
even though we're using the same `PromptTemplate` definition and the same
task instruction."

Before touching any code, write down at least three concrete hypotheses
grounded in this module's theory (not "maybe Anthropic's model is just
worse") — think about Sections 6.1, 6.4, and 6.6 specifically: is there
something about *where* or *how* the system prompt is being rendered
differently between the two adapters that could explain a structural,
not qualitative, difference in the two providers' effective prefixes?
Then describe exactly how you'd verify (or rule out) each hypothesis
using the token-stream-tracing technique from Exercise 1, without
guessing.

---

## 16. Checklist

- [ ] I can explain why a prompt is "just a prefix" and not a separate
      instruction channel, and derive at least two consequences of that
      fact.
- [ ] I can explain chain-of-thought as compute expansion, not
      persuasion, and predict when it will and won't help based on model
      class.
- [ ] I can explain what a persona prompt does and does not change.
- [ ] I understand why prompt injection cannot be fully solved by
      prompting alone, and what belongs in code instead.
- [ ] I completed `prompt-lab` and have real, recorded evidence (not just
      intuition) comparing zero-shot, few-shot, and CoT on at least one
      task.
- [ ] `AnthropicAdapter` and `OpenAIAdapter` both make real, working
      calls through `llm-client-core`, with retry, streaming, and usage
      metrics verified by tests.
- [ ] `PromptTemplate` renders correctly, per-provider, from a single
      definition, and I have a test proving it.
- [ ] The `eval-framework` integration test passes against both live
      providers.

---

## 17. Summary

A prompt is not a command; it's a token prefix, and every "technique" in
this module is explainable as a consequence of that fact combined with
what SFT/RLHF taught the model to do with certain patterns in that prefix
(Module 7) and how attention lets it use everything in that prefix
(Module 6). Few-shot prompting narrows the output distribution by
demonstration. Chain-of-thought buys additional forward-pass compute by
trading it for output length — which is why it matters enormously for
standard models and barely at all for reasoning models (Module 9).
Personas shift register, not capability. And prompt injection is a direct,
unavoidable consequence of the model seeing one undifferentiated token
stream — prompting can reduce its likelihood but can never be the actual
security boundary. On the engineering side, `llm-client-core` now makes
real calls to real providers through a proper adapter and template
abstraction, which is the foundation every remaining Part 3 module builds
on directly.

---

## 18. Next Steps

**Next: Part 3, Module 2 — Structured Outputs (JSON mode, schema
validation, Pydantic-style output parsing).** This module will extend
`PromptTemplate` and both provider adapters with reliable structured
output extraction — the natural next layer once raw prompting and real
provider calls are working, and a direct prerequisite for Module 3's tool
calling.

Reply "continue" for Module 2, or flag anything to go deeper on.
