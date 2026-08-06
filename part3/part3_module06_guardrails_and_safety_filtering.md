# Part 3, Module 6: Guardrails & Safety Filtering

> Module 5 gave you evidence — trace-level attribution of exactly where
> your pipeline fails and how often. This module uses that evidence to
> place guardrails precisely where they earn their cost, rather than
> scattering generic safety checks everywhere. Guardrails are not a
> replacement for the security and evaluation discipline built in
> Modules 3 and 5 — they're the enforcement layer that makes both of
> those disciplines actionable at runtime.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Define what a guardrail actually is at the systems level: a runtime
   check — separate from the model's own generation — that inspects
   input, output, or an intended action and decides whether to allow,
   block, modify, or escalate it.
2. Classify guardrails by where they sit in the pipeline (input
   filtering, output filtering, action-level/tool-call filtering) and
   explain why each layer catches a distinct class of failure that the
   others structurally cannot.
3. Distinguish rule-based/deterministic guardrails from model-based
   (classifier or LLM-judge) guardrails, and choose correctly between
   them based on the failure mode you're targeting and the latency/cost
   budget available.
4. Explain why guardrails are necessary specifically *because* prompting
   alone cannot guarantee behavior (directly extending Module 1's
   prompt-injection argument and Module 3's tool-calling security
   argument) — a guardrail is the code-level enforcement those modules
   said security must live in.
5. Design a layered guardrail architecture using Module 5's trace-level
   failure evidence to decide placement, avoiding both under-protection
   (gaps at high-risk stages) and over-guarding (redundant, latency-
   costly checks at low-risk stages).
6. Evaluate guardrail effectiveness rigorously — false-negative rate
   (harmful content that slips through) and false-positive rate
   (legitimate content wrongly blocked) — using `eval-framework`, since
   an unmeasured guardrail is exactly as unreliable as an unmeasured
   prompt.
7. Implement a working guardrail layer in `llm-client-core`, placed at
   the stages Module 5's evidence identifies as highest-risk, with full
   evaluation coverage.

---

## 2. Prerequisites

- **Part 3, Module 5** (Evaluating Full LLM-Powered Pipelines) — this
  module's guardrail placement decisions are driven directly by that
  module's trace-level failure attribution; you need `TraceScorer` and
  `PipelineGoldenDataset` working.
- **Part 3, Module 3** (Tool Calling & MCP) — the action-level guardrail
  layer in this module is the direct, formalized version of the
  least-privilege/allow-list enforcement you already built there.
- **Part 3, Module 1** — the prompt-injection discussion (Section 6.5)
  is the motivating problem this module's guardrails partially address.
- **Part 1, Module 9** (Security Fundamentals) — the OWASP-style
  checklist and threat-modeling habits transfer directly.

---

## 3. Estimated Study Time

**7–9 hours** (2–3 hours theory/reading, 5–6 hours hands-on).

---

## 4. Difficulty

★★★★☆ (4/5)

The individual guardrail mechanisms (a regex check, a classifier call, a
schema-based tool allow-list) are each simple. The difficulty is in the
judgment: correctly modeling the false-positive/false-negative tradeoff,
avoiding guardrail sprawl that adds latency without adding real
protection, and correctly measuring effectiveness rather than assuming it.

---

## 5. Why This Matters

Guardrails are the most commonly *both* underbuilt and overbuilt part of
production LLM systems — underbuilt because teams assume "the system
prompt says not to do that" is sufficient (Module 1, Section 6.5 already
told you it isn't), and overbuilt because teams bolt on generic
third-party "safety" classifiers everywhere without evidence they target
the failure modes that actually occur in that specific system, paying
latency and cost for protection that doesn't match the real risk profile.
Getting this right — evidence-driven placement, rigorous effectiveness
measurement, layered rather than single-point defense — is a direct,
practical expression of the security maturity that senior/staff-level
engineering roles and safety-conscious labs (a category that includes
your named platform-engineering targets) explicitly look for.

---

## 6. Theory

### 6.1 What a guardrail actually is

A guardrail is code — deterministic or model-based, but always something
your application executes and controls — that inspects something at
runtime (an incoming user message, a model's generated output, a
requested tool call) and makes an allow/block/modify/escalate decision,
*independent of whatever the model itself "decided" to do*. This
independence is the entire point: recall from Module 1, Section 6.5 and
Module 3, Section 6.5 that a model's behavior is shaped by soft,
probabilistic prompt-level signals, which can be overridden by
sufficiently adversarial input (prompt injection) or can simply fail
probabilistically even with no adversary present (the model makes an
honest mistake). A guardrail doesn't ask the model to behave — it
verifies, from outside the model's own reasoning, whether the actual
input/output/action satisfies a defined policy, and can act even when
the model itself was fooled or wrong.

This is precisely why guardrails must live in code (this module) rather
than in the prompt (Module 1) — a guardrail that itself only exists as
a prompt instruction ("please don't do X") inherits exactly the same
unenforceability Module 1, Section 6.5 already established. A real
guardrail's decision must be made by something the adversarial or
mistaken input cannot influence merely by being persuasive text within
the model's context — a regex match, a classifier score, a fixed
allow-list lookup, or a *separate* model call whose own output is
narrowly constrained (Module 2) and treated as a policy decision, not as
free-form generation to be trusted blindly.

### 6.2 The three placement layers, and what each catches

**Input guardrails** — inspect the user's (or any upstream system's)
input *before* it reaches the model. Catch: known-bad patterns (explicit
jailbreak attempts, disallowed topics per your product policy, PII that
shouldn't be sent to a third-party model at all), and can reject or
sanitize before any model call is even made — cheapest possible
intervention point, since a blocked input costs nothing in model
inference.

**Output guardrails** — inspect the model's *generated response* before
it's returned to the user or acted upon. Catch: policy-violating content
the model produced despite input-side checks passing (the model can
still generate something problematic from a completely benign-looking
input, since generation is stochastic — Module 1, Section 6.1), and
schema/format violations that slipped past Module 2's constrained
decoding for whatever provider-specific reason.

**Action-level guardrails** — inspect a *requested tool call* (Module 3)
before your application actually executes it. Catch: the tool-calling-
specific risk category Module 3, Section 6.5 already identified —
authorized-looking but actually unauthorized or unsafe actions,
including those driven by indirect prompt injection via a tool's own
result content. This layer is non-negotiable for any tool with real-world
side effects, because it is the *last* point before an irreversible
action occurs, and therefore carries the highest consequence of any
guardrail layer in the pipeline.

**Why all three, and why none is a substitute for another**: each layer
inspects a different artifact at a different point in the pipeline, and
each catches failures that occur *after* the previous layer's check
point — an input guardrail cannot catch a problem the model itself
introduces during generation (that requires an output guardrail), and
neither input nor output guardrails can catch a tool-call-specific risk
that only manifests in the structured call itself, independent of how
benign the surrounding generated text looks (that requires an
action-level guardrail). This directly explains why Module 3's
allow-list/authorization check was correctly placed at the tool-execution
point rather than earlier — it's this module's action-level guardrail,
already built, now being named and formalized as part of a complete
layered system.

### 6.3 Rule-based vs. model-based guardrails: choosing correctly

**Rule-based (deterministic) guardrails** — regex/keyword matching,
fixed allow-lists, schema validation, rate limits, exact-match PII
patterns (a well-formed SSN or credit-card-number pattern). **Strengths**:
zero latency overhead relative to a model call, perfectly reproducible,
trivially auditable, immune to prompt-injection-style manipulation
because there's no model in the decision path at all. **Weaknesses**:
brittle against paraphrase or novel phrasing — a rule tuned to catch one
specific jailbreak phrasing catches nothing once the phrasing changes
even slightly, since it has no semantic understanding, only pattern
matching.

**Model-based guardrails** — a purpose-built classifier (fast,
narrow-scope, often much smaller than your main model) or an LLM-judge
call (Module 2's structured-output mechanism, applied to a policy
question: "does this content violate policy X? respond with a
schema-constrained yes/no plus confidence") scoring content for policy
violation. **Strengths**: generalizes across paraphrase and novel
phrasing far better than fixed rules, can capture genuinely semantic
judgments ("is this medical advice too specific to be safe" is not a
regex-expressible property). **Weaknesses**: real latency and cost
overhead (an extra model call per guardrail check), and — critically —
inherits the exact same stochasticity and bias risks as any other model
call (`judge-bias-lab`'s lessons apply directly: an unstructured
"is this bad?" judge prompt is exactly as unreliable here as it was for
pipeline evaluation in Module 5).

**Practical decision rule**: use rule-based guardrails for anything with
a genuinely fixed, enumerable pattern (PII formats, an explicit
disallowed-word list, schema/format conformance — much of which you
already built via Module 2's validation layer) — these are cheap and
should be your first line of defense wherever applicable. Reserve
model-based guardrails for genuinely semantic judgments that resist
enumeration (topic-level policy violations, tone/intent classification,
jailbreak-attempt detection against novel phrasing) — and when you use
them, apply the same rubric-based, schema-constrained discipline Module
5 established for pipeline judges, not an open-ended prompt.

### 6.4 Evidence-driven placement: using Module 5's trace data

The naive approach to guardrails is uniform coverage — a generic content
filter on every input and every output, everywhere, regardless of actual
risk. This is both expensive (every check adds latency, and model-based
checks add real cost) and, paradoxically, often less effective than
targeted coverage, because attention and tuning effort get spread thin
across many low-risk checkpoints instead of concentrated on the specific
stages where your own pipeline's trace-level evaluation (Module 5)
showed failures actually cluster.

**The correct process**: before adding any guardrail, look at your
`TraceScorer` results across your golden dataset and any production
incident data. Where do failures actually concentrate? If failures
cluster specifically around ambiguous tool-argument construction after
memory retrieval (a real pattern Module 5's debugging exercise surfaced),
that's exactly where an action-level guardrail belongs, and a generic
input-side content filter would do nothing to address it. This turns
guardrail design from "add safety checks because that seems responsible"
into "add specifically the checks that address failure modes you have
evidence for" — a direct, evidence-based extension of the debugging
discipline from Part 1, Module 10 (Performance & Profiling): don't
optimize/guard blindly, profile first, then act on the actual bottleneck
or risk.

### 6.5 Measuring guardrail effectiveness: false negatives and false positives

A guardrail is itself a classifier (even a rule-based one, formally), and
must be evaluated as one — reporting only "we added a guardrail" without
measuring its actual false-negative rate (harmful/policy-violating
content that still gets through) and false-positive rate (legitimate,
benign content wrongly blocked) is exactly the unmeasured-prompt mistake
Module 1 already warned against, applied to safety infrastructure instead
of a feature prompt.

**False negatives** are the more commonly discussed risk (harmful content
slipping through) but **false positives carry real, underweighted cost**:
a guardrail that blocks legitimate user requests too aggressively
directly damages product usability, and — this is easy to miss —
*trains users to route around your product* rather than trust it,
which is its own kind of long-term failure. Build a golden dataset
(`eval-framework`, Part 2 Module 8) containing both genuine
policy-violation cases *and* legitimate-but-superficially-similar cases
(the classic "how do pharmacists store controlled substances safely"
vs. an actual harmful request pattern), and report both rates explicitly,
with confidence intervals (Module 5, Section 6.5's discipline applies
here identically) — never tune a guardrail against only one side of this
tradeoff.

---

## 7. Mental Models

1. **"A guardrail's decision must be made by something the input cannot
   talk its way past."** If a guardrail's enforcement is itself just a
   prompt instruction, it inherits Module 1's unenforceability problem —
   real guardrails live in code the model doesn't control.
2. **"Input, output, and action guardrails each catch a failure the
   others structurally cannot."** None is a substitute for the others;
   layering is not redundancy, it's coverage of genuinely distinct
   failure points.
3. **"Rules for the enumerable, models for the semantic."** Reach for
   deterministic checks first wherever the pattern is genuinely fixed;
   reserve model-based judgment for what resists enumeration.
4. **"Guard where the evidence says to guard."** Trace-level failure data
   from your own pipeline evaluation should drive placement, not a
   generic checklist applied uniformly.
5. **"A false positive is not a free, safe default."** Over-blocking has
   a real cost — usability damage and erosion of trust — and must be
   measured with the same rigor as the harm you're trying to prevent.

---

## 8. Visual Explanation

**Diagram 1 — The three guardrail layers in the pipeline**

```
 [user input] ──►[INPUT GUARDRAIL]──►[model: prompt→tools→memory→gen]
                        │                         │
                  block/allow/               emits tool_call
                  sanitize                   request
                        │                         │
                        │                  [ACTION GUARDRAIL]
                        │                         │
                        │                  block/allow before
                        │                  real execution
                        │                         │
                        ▼                         ▼
                  (rejected              (tool executes, result
                   here never             flows back into context)
                   reaches model)
                                                    │
                                          [model generates final answer]
                                                    │
                                          [OUTPUT GUARDRAIL]
                                                    │
                                          block/allow/modify
                                                    │
                                                    ▼
                                            [response to user]
```

**Diagram 2 — Rule-based vs. model-based tradeoff**

```
                 RULE-BASED                MODEL-BASED
Latency:         ~0ms                      real (extra call)
Cost:            ~$0                       real (per-check tokens)
Robust to        NO — brittle to           YES — generalizes
paraphrase:      novel phrasing            across phrasing
Auditability:    trivial, deterministic    harder — inherits
                                           model stochasticity
Best for:        enumerable patterns       genuinely semantic
                 (PII format, schema,      judgments (topic/
                 fixed word lists)         intent classification)
```

**Diagram 3 — Evidence-driven placement using Module 5's trace data**

```
TraceScorer results across golden set + production incidents:

Stage           Failure rate    Guardrail priority
─────────────────────────────────────────────────
Prompt/intent        3%          low  → light input check
Tool selection       4%          low  → covered by Module 3's allow-list
Tool ARGUMENTS       14%         HIGH → add action-level guardrail HERE
Memory retrieval     2%          low  → skip for now
Final generation     5%          med  → output schema check (have it)

           ▲
  Guardrail investment concentrated where evidence shows
  the actual failure cluster is — not spread uniformly.
```

---

## 9. Recommended Resources

1. **OWASP — "LLM Top 10" full list** (owasp.org GenAI security project)
   — beyond Module 3's prompt-injection entry, read the broader list
   (insecure output handling, excessive agency, sensitive information
   disclosure) since several map directly onto this module's three
   guardrail layers.
2. **Anthropic — content-classification and safety-tooling
   documentation** (docs.claude.com, anthropic.com) — read for the
   vendor's own framing of moderation/classification tooling available
   at the API level, since this may reduce how much you need to
   hand-build for common cases.
3. **NIST AI Risk Management Framework (or equivalent government/
   standards-body guidance)** (nist.gov) — read the risk-categorization
   approach for a more rigorous vocabulary around false-negative/
   false-positive tradeoffs (Section 6.5) than most informal engineering
   blog posts provide.
4. **A well-documented open-source guardrails library's architecture
   docs** (e.g., a project specifically built for LLM input/output
   validation and policy enforcement, found via its GitHub README) —
   read specifically for how a real, maintained library structures
   rule-based vs. model-based checks, and compare its layering against
   Section 6.2's three-layer model.
5. **Your own `judge-bias-lab` and `eval-framework` work (Part 2, Module
   8)** — re-read before designing this module's model-based guardrails,
   since the bias-mitigation discipline transfers directly and you
   should not rediscover those lessons from scratch.

---

## 10. Exercises

1. **Classify a real incident (or a plausible one) into a layer.** Take
   any specific failure scenario from Module 5's debugging exercise (the
   tool-argument-construction issue) and determine precisely which
   guardrail layer (input, output, action) would actually have caught
   it, and explain why the other two layers structurally could not have.
2. **Rule-based guardrail, built and measured.** Implement a
   deterministic input guardrail for one enumerable pattern relevant to
   your system (e.g., a PII-format check). Measure its false-positive
   rate against a golden set containing legitimate content that
   superficially resembles the pattern (e.g., a phone-number-shaped
   string that's actually a product SKU).
3. **Model-based guardrail, rubric-constrained.** Implement one
   semantic, model-based guardrail (e.g., detecting an attempted
   jailbreak/instruction-override in user input) using a structured,
   schema-constrained judge prompt (Module 2 discipline), not an
   open-ended one. Measure both its false-negative and false-positive
   rates against a golden set containing both real attempts and
   superficially-similar-but-legitimate content.
4. **Layer bypass test.** Deliberately construct an input designed to
   pass your input guardrail but produce a policy-violating *output* —
   confirm your output guardrail catches it, proving the layers are
   genuinely independent and non-redundant, not just doing the same
   check twice.
5. **Evidence-driven placement, on your own pipeline.** Using your actual
   `TraceScorer` results from Module 5, identify your pipeline's
   highest-failure-rate stage and design (don't necessarily implement in
   full) the guardrail you'd place there, explicitly justifying the
   choice with the trace data rather than intuition.

---

## 11. Mini-Project: `guardrail-bench`

A small standalone tool, built against `eval-framework`, that takes a
guardrail implementation (rule-based or model-based) and a golden
dataset containing both true-positive (genuine violation) and
true-negative (legitimate-but-similar) cases, and reports false-negative
rate, false-positive rate, and latency overhead — the direct measurement
tool this module's theory insists on (Section 6.5), built once and
reused for every guardrail you add to the production system.

---

## 12. Production Project: A Layered, Evidence-Placed Guardrail System in `llm-client-core`

### 12.1 What you're building

1. **An input guardrail stage**, combining rule-based checks (PII
   pattern detection, an explicit disallowed-content list relevant to
   your system's domain) and one model-based, rubric-constrained check
   (jailbreak/instruction-override attempt detection), integrated as a
   pre-generation step in `llm-client-core`'s call path.

2. **An output guardrail stage**, checking generated responses against
   both your existing Module 2 schema/format validation (already built —
   this module formalizes it as this layer's rule-based component) and
   one additional model-based semantic check appropriate to your
   system's domain.

3. **A formalized action guardrail stage**, explicitly extending Module
   3's tool-execution allow-list/authorization check into this module's
   layered framework — same mechanism, now named and evaluated as part
   of a complete system rather than a standalone security feature.

4. **Placement justified by Module 5 evidence**: before finalizing where
   each check lives and how strict its threshold is, run your
   `TraceScorer` across your existing golden dataset plus this module's
   new guardrail-specific test cases, and document, in writing, which
   observed failure pattern each guardrail addresses — no guardrail
   should exist in the final system without a stated, evidence-backed
   reason for its placement.

5. **`guardrail-bench` evaluation for every guardrail added**, reporting
   false-negative rate, false-positive rate, and latency overhead for
   each, integrated into the pipeline CI (Module 5, Section 12) so a
   future change that degrades guardrail effectiveness is caught
   automatically, not discovered in production.

### 12.2 What this sets up for later modules

- **Part 3's Context Engineering module** will need to account for
  guardrail-check token/latency overhead as part of its budget
  management.
- **Part 5 (Agents)** applies this exact layered guardrail architecture,
  with the action-level layer becoming significantly more load-bearing
  once agents can take longer, more autonomous action sequences.
- **Part 3's capstone** integrates this guardrail system as a
  non-optional component of the finished production service.

### 12.3 Definition of done

- All three guardrail layers (input, output, action) are implemented and
  integrated into `llm-client-core`'s call path.
- Every guardrail's placement is documented with a specific reference to
  the `TraceScorer`/production-evidence pattern it addresses.
- `guardrail-bench` reports false-negative rate, false-positive rate, and
  latency overhead for every guardrail, with results tracked in
  `observability-stack`.
- The layer-bypass test (Exercise 4) passes, proving the layers are
  independently effective, not redundant.
- Guardrail evaluation is part of pipeline CI (Module 5) and would catch
  a regression in guardrail effectiveness on a future change.

---

## 13. Practice & Interview Questions

1. Define what a guardrail is at the systems level, and explain why a
   guardrail whose enforcement is itself only a prompt instruction fails
   to actually guard anything, using Module 1's argument.
2. Name the three guardrail placement layers and give a concrete example
   of a failure each one catches that the other two structurally cannot.
3. When would you choose a rule-based guardrail over a model-based one,
   and vice versa? Ground your answer in the enumerable-vs-semantic
   distinction, not just "rules are faster."
4. Why is false-positive rate an equally important guardrail metric as
   false-negative rate, and what real cost does an over-aggressive
   guardrail impose that's easy to underweight?
5. Describe how you would decide where to place a new guardrail in an
   existing pipeline, using evidence rather than intuition.
6. A teammate proposes adding a generic third-party content-safety
   classifier to every input and every output "just to be safe," with no
   reference to any specific observed failure pattern. What's your
   response, grounded in Section 6.4's evidence-driven placement
   argument?

---

## 14. Common Mistakes

- **Treating a prompt-level instruction as a guardrail.** "The system
  prompt tells it not to do X" is not enforcement — it's exactly the
  soft, overridable signal Module 1 and Module 3 already warned about.
- **Uniform, undifferentiated guardrail coverage** applied everywhere
  regardless of actual, evidenced risk — expensive and often less
  effective than targeted coverage.
- **Measuring only false-negative rate**, ignoring false-positive cost
  and the real product/trust damage over-blocking causes.
- **Using an open-ended, unstructured LLM-judge prompt as a guardrail**,
  inheriting all of `judge-bias-lab`'s known unreliability without the
  rubric-based discipline that mitigates it.
- **Guarding without evidence.** Adding checks because they seem
  responsible, rather than because trace-level data (Module 5) shows a
  specific, real failure cluster they address.
- **Never re-evaluating guardrails after pipeline changes.** A guardrail
  tuned against one version of your prompt/tool set can silently degrade
  in effectiveness as the surrounding system evolves, if it isn't part of
  ongoing pipeline CI.

---

## 15. Debugging Exercise

Your action-level guardrail correctly blocked a batch of clearly
malicious tool-call attempts during testing. After deployment, you
discover — via `guardrail-bench`'s ongoing monitoring — that its
false-positive rate has crept up over several weeks, now blocking a
noticeable fraction of legitimate tool calls, even though the guardrail's
code hasn't changed.

Write down at least two concrete hypotheses for why a guardrail's
effectiveness could drift over time with no code changes to the
guardrail itself (consider: has the *surrounding* pipeline changed in a
way that shifts the distribution of legitimate tool-call arguments the
guardrail now sees, e.g., a Module 4 memory-retrieval change altering
typical argument phrasing? has genuine user behavior shifted in a way
your original golden set didn't anticipate?), and describe how you'd use
`TraceScorer` and `guardrail-bench` together to distinguish between these
hypotheses before retuning the guardrail.

---

## 16. Checklist

- [ ] I can define a guardrail precisely and explain why its enforcement
      must live outside the model's own controllable context.
- [ ] I can name the three placement layers and give a distinct failure
      example for each.
- [ ] I can choose correctly between rule-based and model-based
      guardrails for a given failure mode and justify the choice.
- [ ] I can explain why guardrail placement should be evidence-driven
      (via `TraceScorer`) rather than uniform or intuition-based.
- [ ] I can explain why false-positive rate matters as much as
      false-negative rate, with a concrete cost example.
- [ ] `guardrail-bench` is built and reports both error rates plus
      latency overhead for a real guardrail.
- [ ] All three guardrail layers are implemented in `llm-client-core`,
      each with a documented, evidence-based justification for its
      placement.
- [ ] The layer-bypass test proves the three layers are independently
      effective.
- [ ] Guardrail evaluation is integrated into pipeline CI and would catch
      a future regression in effectiveness.

---

## 17. Summary

A guardrail is code-level enforcement, independent of the model's own
generation, that inspects input, output, or a requested action and makes
an allow/block/modify decision — it exists precisely because Module 1
and Module 3 already established that prompt-level instructions alone
cannot guarantee behavior, especially under adversarial or merely
mistaken conditions. Input, output, and action-level guardrails each
catch a distinct class of failure the others structurally cannot, so
layering is coverage, not redundancy. Rule-based checks handle enumerable
patterns cheaply and reliably; model-based checks handle genuinely
semantic judgments at real latency/cost, and must use the same
rubric-based discipline `judge-bias-lab` already proved necessary for
LLM judges elsewhere in the stack. Placement and tuning should be driven
by Module 5's trace-level evidence of where failures actually
concentrate, not uniform coverage or intuition, and every guardrail must
be measured on both false-negative and false-positive rate — an
unmeasured guardrail is exactly as unreliable as an unmeasured prompt.
`llm-client-core` now has a full, evidence-placed, measured three-layer
guardrail system, directly setting up Part 5's heavier reliance on the
action-level layer once agents take longer, more autonomous action
sequences.

---

## 18. Next Steps

**Next: Part 3, Module 7 — Context Engineering (Context Window Budget
Management and Prompt Compression).** With prompting, structured output,
tool calling, memory, evaluation, and guardrails all now built and
measured, this module formalizes the token-budget discipline that's been
implicit since Module 4's compaction work into a general-purpose
practice covering the entire pipeline's context assembly — directly
extending Part 2, Module 5's tokenization/cost foundations.

Reply "continue" for Module 7, or flag anything to go deeper on.
