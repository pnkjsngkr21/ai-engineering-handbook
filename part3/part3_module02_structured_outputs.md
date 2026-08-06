# Part 3, Module 2: Structured Outputs

> Module 1 got real provider calls working through `llm-client-core` and
> established the token-prefix mental model. That's enough to get *good*
> text back. This module is about getting *reliable, machine-parseable*
> data back — the difference between a chatbot and a system another
> program can safely call.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain why LLMs producing malformed or inconsistent structured output
   is an expected consequence of autoregressive generation, not a random
   bug — and why this means "just ask nicely for JSON" is an incomplete
   solution.
2. Distinguish the three layers at which structured-output reliability
   can be enforced — prompt-level formatting instructions, provider-level
   constrained decoding (JSON mode / structured outputs / tool-schema
   forcing), and application-level validation/repair — and know which to
   reach for and when.
3. Explain, mechanistically, how constrained decoding (grammar-constrained
   / schema-constrained sampling) actually works at the token level, and
   why it eliminates a category of failure that pure prompting cannot.
4. Use Pydantic models as the single source of truth for an expected
   output shape, and generate both the provider-facing schema and the
   Python-side validation from that one definition.
5. Design a robust parse → validate → repair-or-retry pipeline for
   structured LLM output, including what to do when validation fails.
6. Extend `PromptTemplate` and both provider adapters in `llm-client-core`
   (Module 1) with first-class structured-output support, and wire
   correctness checks through `eval-framework` (Part 2, Module 8).

---

## 2. Prerequisites

- **Part 3, Module 1** — you need `AnthropicAdapter`, `OpenAIAdapter`,
  and `PromptTemplate` working, since this module extends all three
  directly. You also need the token-prefix mental model fresh.
- **Part 2, Module 6** (Attention & the Transformer Architecture) — the
  token-by-token sampling loop is central to understanding constrained
  decoding in Section 6.3.
- Comfort with Pydantic (Part 0, Module 1/9 territory — `FastAPI` request/
  response models already use it) and with JSON Schema basics.
- **Part 2, Module 8** (`eval-framework`) — you'll route correctness
  checks for this module's outputs through it.

---

## 3. Estimated Study Time

**7–9 hours** (2–3 hours theory/reading, 5–6 hours hands-on).

---

## 4. Difficulty

★★★☆☆ (3/5)

Conceptually straightforward once you see the three-layer model in
Section 6.2, but the production project requires careful cross-provider
handling (Anthropic and OpenAI expose meaningfully different mechanisms
for this), which is where the real engineering effort goes.

---

## 5. Why This Matters

The moment an LLM's output needs to be consumed by code rather than read
by a human — populating a database record, triggering a downstream API
call, rendering a UI component, feeding a tool-calling loop (Module 3,
next) — "usually valid JSON" is not good enough. A pipeline that fails to
parse 2% of the time is a pipeline that silently drops or corrupts 2% of
your production traffic, and that failure mode is exactly the kind that
doesn't show up in a demo and does show up at 2am in an on-call page.

This is also a very commonly probed interview topic for AI-engineering
roles specifically because it separates people who've only used a
chatbot from people who've shipped an LLM-backed API: "how do you
guarantee valid structured output from an LLM in production" is close to
a canonical question, and "I add 'respond only in JSON' to the prompt" is
the answer that ends the interview early.

---

## 6. Theory

### 6.1 Why raw prompting for JSON is unreliable — the mechanism

Recall from Module 1, Section 6.1: the model is doing next-token
prediction over a learned distribution, one token at a time, with no
separate "JSON mode" wired into its generation loop by default. When you
instruct "respond only with valid JSON," you are only shifting the
*probability distribution* over next tokens toward JSON-shaped text — you
are not eliminating the possibility that at some token position, the
model samples something that breaks JSON validity (an unescaped quote
inside a string, a trailing comma, prose leaking in before or after the
object, a truncated object if the generation hits a length limit
mid-structure).

Three concrete, mechanistically-grounded failure modes to expect:

- **Preamble/postamble leakage**: SFT training (Part 2, Module 7) biases
  the model toward "helpful assistant" framing text ("Sure, here's the
  JSON you requested:") even when instructed not to include it, because
  that framing pattern is heavily represented in the fine-tuning
  distribution and competes with your specific instruction.
- **Subtle schema drift**: the model may produce syntactically valid JSON
  that doesn't match your *intended* schema — wrong field names, wrong
  types (a string where you needed a number), extra or missing fields —
  because nothing about "respond in JSON" tells the model your exact
  shape unless you specify it, and even specifying it in the prompt is
  still just a soft distributional nudge, not an enforced constraint.
- **Truncation at token limits**: if `max_tokens` is reached mid-object,
  you get syntactically invalid, truncated JSON — a purely mechanical
  failure with nothing to do with the model's "understanding" of the
  task.

None of these are more likely to be solved by *more forceful prompt
wording*. They're better solved by moving enforcement to a layer where
it's actually guaranteed, which brings us to the three-layer model.

### 6.2 The three-layer model for structured output reliability

| Layer | What it does | Guarantees |
|---|---|---|
| **1. Prompt-level instruction** | Tells the model, in natural language, the desired shape (e.g., embeds a JSON Schema or example in the prompt) | Improves the *odds* of correct output; guarantees nothing |
| **2. Provider-level constrained decoding** | The provider's inference server restricts, at each sampling step, the set of tokens the model is even allowed to sample to those consistent with a supplied grammar/schema | Guarantees syntactic validity (and, for full schema-constrained modes, schema conformance); does not guarantee the *content* is semantically correct |
| **3. Application-level validation/repair** | Your code parses the (guaranteed-syntactically-valid, or best-effort) output against a schema, and handles failures via retry, repair, or fallback | Guarantees your application never proceeds with data that fails your own correctness checks |

**The critical realization**: these layers are complementary, not
substitutes. Layer 2 eliminates an entire category of failure (malformed
syntax) essentially for free, which makes Layer 3's job dramatically
easier (you're now only handling semantic/business-logic validation
failures, not JSON parse errors) — but Layer 2 does not replace the need
for Layer 3, because a schema can be syntactically satisfied while still
being *wrong* (e.g., a confidence score of `1.5` in a field typed as
"float," or a summary field with an empty string). And Layer 1 still
matters even when Layer 2 is available, because the constrained decoder
only guarantees the *shape*; the *quality* of what fills that shape is
still governed by ordinary prompting principles (Module 1).

### 6.3 How constrained decoding actually works, mechanistically

This is where you connect straight back to `toy-transformer`'s sampling
loop (Part 2, Module 6). Recall the standard sampling step: the model
produces a probability distribution over the entire vocabulary for the
next token; you then sample (or take the argmax, or apply
temperature/top-p) from that distribution to pick the actual next token.

**Constrained/grammar-guided decoding modifies exactly this step**: before
sampling, the decoding process computes — from the current partial output
and a formal grammar (a JSON Schema compiled down to something like a
finite-state machine or a context-free grammar, depending on
implementation) — the *subset* of vocabulary tokens that would keep the
output a valid continuation of that grammar. Every other token's
probability is masked to (effectively) zero before sampling. The model
still assigns preferences over the *allowed* subset — its actual learned
distribution among valid continuations is preserved — but it can no
longer sample a token that would break syntactic validity, because that
token was never in the eligible set to begin with.

This is why constrained decoding gives you a hard guarantee that prompt
instructions never can: it's not making invalid tokens *less likely*,
it's making them *impossible to sample*, at the same layer where
`toy-transformer` did unconstrained argmax/multinomial sampling over the
full vocabulary. You're literally restricting the support of the sampling
distribution at each step.

**Provider-specific mechanisms** (this is where the real engineering
differences show up, and why a naive single code path across providers
breaks):

- **Anthropic**: exposes this primarily through **tool use / tool
  schemas** — you define an input schema for a "tool" and force the model
  to call it, which constrains the output to match that schema. There
  isn't a separate freestanding "JSON mode" distinct from tool calling;
  structured output and tool calling share the same underlying mechanism
  in Anthropic's API (this is a deliberate design choice worth noting —
  it previews Module 3 directly).
- **OpenAI**: exposes both a `response_format: json_object`/`json_schema`
  mode (structured outputs) directly on chat completions, *and* a
  separate function/tool-calling mechanism — these are two distinct
  paths that happen to converge on similar guarantees.

Your adapter abstraction (Module 1) needs to hide this divergence behind
one interface, which is exactly this module's production project.

### 6.4 Why Pydantic is the right single source of truth

You already use Pydantic for FastAPI request/response validation (Part 0,
Module 9). The same tool does double duty here, and for a principled
reason: a Pydantic model is simultaneously

1. a **JSON Schema generator** (`model.model_json_schema()`) — which is
   exactly the artifact both providers' constrained-decoding mechanisms
   consume (Section 6.3), and
2. a **runtime validator** (`Model.model_validate_json(...)`) — which is
   exactly Layer 3's enforcement mechanism (Section 6.2).

Defining your expected output shape once, as a Pydantic model, and
deriving both the provider-facing schema and your own validation from
that single definition eliminates an entire class of bugs where the
prompt's described shape, the schema sent to the provider, and the
application's parsing logic silently drift apart from each other over
time — a direct application of the DRY/single-source-of-truth principle
from Part 1's Clean Architecture module.

### 6.5 The parse → validate → repair pipeline

Even with Layer 2 constrained decoding active, you still need a
disciplined Layer 3 pipeline, because:

- Not every provider/model configuration supports full schema-constrained
  decoding for every use case (sometimes you'll only have prompt-level
  instruction available, e.g., older models, some streaming
  configurations, or providers without the feature).
- Schema-*valid* is not the same as *semantically correct* (Section 6.2)
  — a `confidence: float` field constrained to be a valid float can still
  be `-4.2`, which is syntactically fine and semantically nonsense if
  your domain requires `0.0–1.0`.

**Recommended pipeline shape:**

```
raw_output
   │
   ▼
[1] Strip known non-JSON wrapping artifacts (markdown code fences, etc.)
   │
   ▼
[2] Attempt json.loads() / Pydantic model_validate_json()
   │
   ├── success ──► [3] Domain validation (Pydantic validators: ranges,
   │                    enums, cross-field constraints)
   │                    │
   │                    ├── success ──► done, return validated object
   │                    └── failure ──► [4] Repair-or-retry strategy
   │
   └── failure ──► [4] Repair-or-retry strategy
                        │
                        ├── Re-prompt with the exact parse/validation
                        │   error appended, asking the model to fix it
                        │   (cheap, often works — the model can see its
                        │   own mistake concretely)
                        ├── Retry generation from scratch (for transient
                        │   sampling issues, e.g., truncation)
                        └── After N attempts: fail loudly to the caller
                            rather than silently passing bad data downstream
```

The re-prompt-with-the-error strategy deserves a mechanistic note: giving
the model its own malformed output plus the specific validation error as
new context works well because you're giving it a concrete, in-context
example of exactly what went wrong (an application of Module 1's
few-shot/in-context-learning mechanism — the "example" here is the
model's own failure) rather than asking it to guess what "be more
careful" means.

### 6.6 Cost and latency implications (a preview)

Structured output enforcement isn't free: constrained decoding can add
latency (the eligible-token computation is extra work per step,
implementation-dependent), and repair/retry loops add both latency and
token cost. This is a legitimate engineering tradeoff you'll formalize
properly in Part 3's later latency/cost-optimization module — for now,
the practical guidance is: **prefer Layer 2 (provider-level constraints)
whenever available**, since it eliminates the most common and costly
failure mode (retry loops from malformed syntax) at effectively zero
marginal prompting cost, and reserve Layer 3 repair loops for genuine
semantic/domain validation failures, which should be rarer.

---

## 7. Mental Models

1. **"Prompting nudges the distribution; constrained decoding masks it."**
   Instructions make good output more *likely*; schema-constrained
   sampling makes bad-syntax output *impossible* by removing invalid
   tokens from the eligible set before sampling, not by discouraging them.
2. **"Schema-valid is not the same as correct."** A constrained decoder
   guarantees shape, never meaning — domain validation is a separate,
   still-necessary job.
3. **"One schema, three consumers."** Define the shape once (Pydantic);
   let the prompt, the provider's constraint mechanism, and your parser
   all derive from that single definition instead of drifting apart.
4. **"Show the model its own mistake."** The most effective repair
   strategy is concrete: feed back the exact parse/validation error, not
   a vaguer "try again."

---

## 8. Visual Explanation

**Diagram 1 — Unconstrained vs. constrained sampling at one token step**

```
UNCONSTRAINED (Module 1's toy-transformer sampling):
vocabulary: [ "a", "{", "\"", "1", ..., "banana", ... ]  (full vocab, ~50k+)
probs:      [ 0.01, 0.15, 0.20, 0.02, ..., 0.001, ... ]
sample from the FULL distribution → any token possible, including ones
that would break JSON validity

CONSTRAINED (schema-guided decoding):
grammar state: "expect a string-opening quote next"
eligible tokens:  [ "\"" ]  ← everything else masked to 0 BEFORE sampling
sample from the MASKED distribution → syntactically invalid tokens are
not just unlikely, they are literally unreachable
```

**Diagram 2 — The three-layer stack**

```
┌────────────────────────────────────────────┐
│ Layer 1: Prompt instruction / embedded schema│  ← improves odds only
├────────────────────────────────────────────┤
│ Layer 2: Provider constrained decoding       │  ← guarantees syntax/shape
│          (Anthropic tool schema /            │
│           OpenAI json_schema mode)           │
├────────────────────────────────────────────┤
│ Layer 3: App-level parse+validate+repair     │  ← guarantees semantics
│          (Pydantic model_validate)           │
└────────────────────────────────────────────┘
        all three needed; none is a substitute for another
```

**Diagram 3 — Single Pydantic model, two derived artifacts**

```
              class ExtractionResult(BaseModel):
                  entity: str
                  entity_type: Literal["PERSON","ORG","LOCATION"]
                  confidence: float = Field(ge=0.0, le=1.0)
                         │
          ┌──────────────┴───────────────┐
          ▼                              ▼
  .model_json_schema()          .model_validate_json(raw_output)
  → sent to provider as          → used by your app to validate
    the schema constraint          the (already syntax-valid) output
```

---

## 9. Recommended Resources

1. **Anthropic — "Tool use" documentation** (docs.claude.com) — read this
   first even though this module isn't "about" tool calling yet, because
   Anthropic's structured-output mechanism *is* its tool-calling
   mechanism (Section 6.3); understanding this now makes Module 3 far
   easier.
2. **OpenAI — "Structured Outputs" guide** (platform.openai.com/docs) —
   covers `response_format: json_schema` directly; read specifically for
   how OpenAI documents its own guarantee boundary (what it does and
   does not guarantee), and compare that boundary to Anthropic's.
3. **Pydantic — official documentation on JSON Schema generation and
   validators** (docs.pydantic.dev) — you need `model_json_schema()`,
   custom validators (`field_validator`, `model_validator`), and
   `model_validate_json()` at a practical level for the production
   project.
4. **"Guidance" or "Outlines" library documentation** (their respective
   GitHub repos/docs) — open-source implementations of grammar-
   constrained decoding; reading how they implement the token-masking
   mechanism (Section 6.3) against open-weight models makes the
   closed-provider version much less mysterious, since you can actually
   read the masking code.
5. **JSON Schema — official specification site** (json-schema.org) — you
   don't need the whole spec, but skim the core keywords
   (`type`, `enum`, `required`, `properties`) since this is the
   interchange format both provider mechanisms and Pydantic all speak.

---

## 10. Exercises

1. **Break it on purpose.** Using only Layer 1 (prompt instruction, no
   provider constraint), write a prompt asking for a JSON object with at
   least 4 fields including one enum-typed field. Run it 15–20 times
   against a real model. Record the raw failure rate and categorize each
   failure using Section 6.1's three failure modes.
2. **Fix it with Layer 2.** Re-run the same task using your provider's
   schema-constrained mechanism (Anthropic tool schema or OpenAI
   `json_schema` mode). Confirm the syntactic failure rate drops to
   (near) zero. Explicitly check: did any *semantically* wrong outputs
   still get through despite being schema-valid?
3. **Design a repair loop.** Implement the parse → validate → repair
   pipeline from Section 6.5 for one task. Deliberately induce a
   validation failure (e.g., ask for a confidence score, then define your
   Pydantic model with an overly strict range) and confirm your repair
   step successfully re-prompts and recovers.
4. **Cross-provider schema parity.** Take one Pydantic model and generate
   the request payload for both Anthropic (as a tool schema) and OpenAI
   (as `response_format`). Diff the two derived schemas by hand and note
   any structural differences you had to account for.
5. **Latency/cost measurement.** Measure and compare: (a) average latency
   for unconstrained + Layer 3 retry-on-failure, vs. (b) Layer 2
   constrained decoding with no retries needed, across the same 15–20
   test inputs from Exercise 1. Record real numbers, not estimates.

---

## 11. Mini-Project: `schema-bench`

A small standalone tool that takes a Pydantic model and a set of test
inputs, runs the same extraction/generation task under all three layers
independently (prompt-only, Layer 2 constrained, Layer 2 + Layer 3
repair), and reports a comparison table of: syntactic validity rate,
semantic validity rate (against a supplied ground truth or rule-based
check), average latency, and average token cost per layer combination.

This is intentionally throwaway/standalone — like Module 1's
`prompt-lab`, its purpose is building calibrated, empirical intuition for
the three-layer tradeoffs (Section 6.2) before formalizing anything
inside the production system below.

---

## 12. Production Project: Structured Output Support in `llm-client-core`

### 12.1 What you're building

Extend Module 1's `PromptTemplate`, `AnthropicAdapter`, and
`OpenAIAdapter` with first-class structured-output capability:

1. **`PromptTemplate.with_output_schema(pydantic_model)`** — a new method
   that attaches an expected-output Pydantic model to a template. This
   should be usable by *any* future module that needs structured output
   (Module 3's tool calling shares this exact mechanism on Anthropic per
   Section 6.3, so build this generally, not narrowly for "JSON only").

2. **Per-adapter schema translation**:
   - `AnthropicAdapter`: translate the Pydantic-derived JSON Schema into
     an Anthropic tool definition, force tool use (`tool_choice`), and
     extract the structured result from the tool-call response block.
   - `OpenAIAdapter`: translate into OpenAI's `response_format:
     {"type": "json_schema", ...}` structure and extract from the
     standard message content.
   - Both paths must return the **same unified result type** from the
     client's perspective — callers should not need to know which
     provider they're talking to (this is the Adapter pattern from Part
     1, Module 2, applied correctly).

3. **The parse → validate → repair pipeline (Section 6.5)** implemented
   as a reusable component (not duplicated per-adapter): given a raw
   provider response and the target Pydantic model, attempt validation,
   and on failure, construct a repair re-prompt (reusing
   `PromptTemplate`'s existing rendering) that includes the specific
   error, up to a configurable max-retry count, after which it raises a
   typed exception rather than returning bad data.

4. **`eval-framework` integration**: add a scorer type (or extend an
   existing one from Part 2, Module 8) that checks schema conformance and
   optionally domain-specific correctness for structured-output tasks,
   and run it against both live adapters as your test suite.

5. **Observability**: emit metrics (via `observability-stack`, Part 1,
   Module 4) for schema-validation failure rate and repair-attempt count
   per call — this is exactly the kind of signal that later modules
   (evaluation of full pipelines, guardrails) will depend on existing
   already.

### 12.2 What this sets up for later modules

- **Part 3, Module 3 (Tool/Function Calling & MCP)** builds directly on
  this — you've already implemented Anthropic's tool-schema mechanism
  here; Module 3 extends it from "force exactly one schema-shaped
  response" to "let the model choose among multiple tools and call them
  in a loop."
- **Part 3's guardrails module** will use this same validation-failure
  signal as one of its input checks.
- **Part 4 (RAG)** will use structured output support here for
  citation/source-attribution formatting in generated answers.

### 12.3 Definition of done

- `with_output_schema()` works identically (from the caller's
  perspective) against both live adapters.
- Schema-conformance failure rate on a 20+ item test set is at or near
  0% (syntactic) with Layer 2 active; any residual failures are handled
  by the repair pipeline and logged, never silently swallowed.
- `eval-framework` scorer integration test passes against both
  providers.
- Metrics for validation failures and repair attempts are visible in
  `observability-stack`.

---

## 13. Practice & Interview Questions

1. Explain, at the token-sampling level, exactly what "constrained
   decoding" restricts and why this gives a guarantee that no amount of
   prompt wording can.
2. Why is "the model returned schema-valid JSON" not sufficient evidence
   that a structured-output pipeline is working correctly? Give a
   concrete example.
3. Anthropic and OpenAI implement structured output via meaningfully
   different mechanisms. Describe both, and describe how you'd design an
   adapter interface that hides this difference from calling code.
4. Design a repair-loop strategy for a validation failure and justify
   why re-prompting with the *specific* error text tends to outperform a
   generic "try again, be more careful" retry.
5. What are the cost/latency tradeoffs of constrained decoding vs.
   unconstrained-generation-plus-retry, and under what circumstances
   would you choose the latter deliberately?
6. A Pydantic model defines `confidence: float = Field(ge=0.0, le=1.0)`.
   The model still returns `confidence: 1.3` inside otherwise-valid JSON.
   At which layer did this fail, and at which layer is it caught?

---

## 14. Common Mistakes

- **Relying on prompt wording alone for anything consequential.**
  "Respond only in JSON" without Layer 2/3 enforcement is exactly the
  failure mode Section 6.1 describes — expect it to break in production
  at scale even if it looked fine in a handful of manual tests.
- **Assuming schema-valid means correct.** Skipping domain validation
  (range checks, cross-field consistency, enum sanity) because "it
  parsed fine" is a very common and costly shortcut.
- **Duplicating the schema definition** across the prompt text, the
  provider payload, and the parsing code instead of deriving all three
  from one Pydantic model — this drifts silently over time and is
  exactly the failure mode Section 6.4 exists to prevent.
- **Infinite or excessive repair retries** without a hard cap and loud
  failure — silently retrying forever (or many times) on a
  fundamentally malformed request hides a real bug and burns cost/latency.
- **Ignoring provider-specific mechanism differences** and trying to
  force one code path across both providers instead of translating
  correctly per adapter (Section 6.3) — this is where naive
  multi-provider structured-output code most often breaks.

---

## 15. Debugging Exercise

Your structured-output pipeline is passing all tests against Anthropic
but intermittently (roughly 1 in 30 calls) fails validation against
OpenAI for the *exact same* `PromptTemplate` and Pydantic model, with an
error indicating an extra, unexpected field in the returned JSON that
isn't in your schema.

Write down at least two concrete, mechanistically-grounded hypotheses
(Sections 6.1–6.3) for why this could be provider-specific rather than a
general flaw in your schema or template, and describe exactly how you'd
verify each — including what you'd inspect in the raw provider response
before assuming anything about "OpenAI being less reliable."

---

## 16. Checklist

- [ ] I can explain why prompting alone cannot guarantee valid structured
      output, at the token-sampling level.
- [ ] I can explain the three-layer model and correctly place any given
      technique (prompt instruction, provider constraint, app validation)
      into the right layer.
- [ ] I understand mechanistically how constrained decoding works
      (masking the eligible token set before sampling).
- [ ] I built `schema-bench` and have real comparative numbers across all
      three layers for at least one task.
- [ ] `PromptTemplate.with_output_schema()` works against both live
      provider adapters with a unified result type.
- [ ] The parse → validate → repair pipeline is implemented as a shared,
      reusable component with a hard retry cap and loud failure on
      exhaustion.
- [ ] `eval-framework` integration verifies schema conformance against
      both providers.
- [ ] Validation-failure and repair-attempt metrics are visible in
      `observability-stack`.

---

## 17. Summary

Getting an LLM to *usually* produce valid JSON is a prompting problem;
getting it to *always* produce valid, schema-conformant, and
domain-correct output is a systems problem with three distinct layers —
prompt instruction (odds, not guarantees), provider-level constrained
decoding (guarantees syntax/shape by masking invalid tokens out of the
sampling distribution entirely), and application-level validation and
repair (guarantees semantics). A single Pydantic model should be the one
source of truth that generates both the provider-facing schema and your
own runtime validation, preventing drift between what you asked for, what
you constrained, and what you check. `llm-client-core` now supports this
end-to-end across both providers, with a shared repair pipeline and
observability — the exact foundation Module 3's tool calling will extend
directly, since on Anthropic, structured output and tool calling are
already the same underlying mechanism you just implemented.

---

## 18. Next Steps

**Next: Part 3, Module 3 — Function/Tool Calling and MCP.** This module
extends the exact schema-constraint mechanism you just built from "force
one specific output shape" to "let the model choose among multiple
available tools/functions and call them, potentially in a multi-turn
loop" — plus an introduction to the Model Context Protocol (MCP) as a
standardized way of exposing those tools.

Reply "continue" for Module 3, or flag anything to go deeper on.
