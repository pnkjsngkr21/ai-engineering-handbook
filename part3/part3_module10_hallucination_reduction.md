# Part 3, Module 10: Hallucination Reduction Techniques

> Every module in Part 3 has been building infrastructure around a
> problem this module finally confronts directly: LLMs produce
> confident, fluent, plausible-sounding statements that are sometimes
> false, with no structural difference in how a true and a false
> statement are generated. This module explains why that's not a bug to
> be patched away, and builds the concrete, evidence-based techniques —
> grounding, citation enforcement, calibrated uncertainty, and
> verification passes — that reduce its frequency and consequence,
> directly building on Module 5's evaluation and Module 6's guardrail
> infrastructure.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain hallucination mechanistically as a direct consequence of
   next-token generation over a learned probability distribution
   (Module 1, Section 6.1) — there is no separate "make things up" mode
   and no separate "state only true things" mode; both true and false
   completions are sampled from the same underlying process, and nothing
   in the generation mechanism itself distinguishes them.
2. Distinguish the several distinct causes that get lumped under
   "hallucination" — genuine knowledge gaps, confident extrapolation
   beyond training data, confabulation under compaction/summarization
   (Module 4), and instruction-following failures — since each has a
   different, targeted mitigation rather than one universal fix.
3. Apply grounding techniques (requiring the model to base claims on
   explicitly supplied context rather than parametric knowledge) and
   explain why this reduces, but does not eliminate, hallucination risk.
4. Design citation/attribution requirements using Module 2's
   structured-output mechanism, and explain why a cited claim is not
   automatically a correct claim (citation can itself be
   confabulated) — connecting directly to the copyright-citation
   discipline you've been applying throughout the handbook's own writing.
5. Elicit and use calibrated uncertainty expression (the model
   indicating confidence or explicitly stating it doesn't know) and
   understand why naive "tell me if you're not sure" instructions are
   often unreliable, requiring more structural approaches.
6. Design and implement a verification pass (a second, targeted model
   or retrieval call that checks a first response's factual claims) and
   understand its cost/latency tradeoff using Module 9's optimization
   framework.
7. Extend `eval-framework` with a genuine hallucination-rate scorer and
   apply the techniques above to a real, measured reduction in
   `llm-client-core`'s pipeline.

---

## 2. Prerequisites

- **Part 3, Module 5** (Evaluating Full LLM-Powered Pipelines) — you
  need trace-level evaluation infrastructure to measure hallucination
  rate meaningfully, not just spot-check it.
- **Part 3, Module 4** (Memory) — the confabulation-under-compaction
  failure mode (Section 6.3 of that module) is a specific, already-
  identified instance of the broader problem this module addresses.
- **Part 3, Module 2** (Structured Outputs) — citation/attribution
  enforcement reuses schema-constrained output directly.
- **Part 3, Module 9** (Latency & Cost Optimization) — verification
  passes have a real cost/latency tradeoff you'll reason about using
  that module's framework.

---

## 3. Estimated Study Time

**7–9 hours** (3 hours theory/reading, 4–6 hours hands-on).

---

## 4. Difficulty

★★★★☆ (4/5)

The theory is conceptually approachable but genuinely humbling in
practice: no technique here eliminates hallucination, and internalizing
that honestly — designing for reduction and detection rather than a
false sense of solved-ness — is the actual difficulty.

---

## 5. Why This Matters

Hallucination is the single most commonly cited concern about
production LLM deployment, and for good reason: a system that states
false information with the same fluent confidence as true information is
uniquely dangerous compared to a system that fails loudly, because there
is no natural signal to the user (or to your own monitoring) that
something is wrong. This module is where the handbook's running
mechanistic-grounding commitment pays off most directly for your
long-term credibility as an AI engineer: understanding *why*
hallucination happens (not "the model sometimes lies," but "the model
has no mechanism to distinguish confident-and-true from confident-and-
false at generation time") is exactly the depth of understanding that
separates engineers who can design real mitigations from those who can
only apply folklore ("just tell it to be careful") that doesn't survive
contact with a genuinely adversarial or ambiguous production case.

---

## 6. Theory

### 6.1 Why hallucination isn't a bug to be patched — it's a structural property of generation

Return one final time to Module 1, Section 6.1: at every token position,
the model produces a probability distribution over the vocabulary and
samples (or takes the highest-probability token) from it. **There is no
separate mechanism, anywhere in this process, that checks a candidate
token or sentence against ground truth before allowing it to be
generated.** A false statement about a historical date and a true one
are produced by exactly the same sampling process, differing only in
how the model's learned distribution happens to weight the relevant
tokens — which is shaped by what patterns were reinforced during
pretraining and SFT/RLHF (Part 2, Modules 6-7), not by any real-time
fact-checking step.

This reframes the goal of this entire module correctly: you are not
trying to "turn off" hallucination, because there's no hallucination
switch to turn off — false and true statements come from the identical
mechanism. You are trying to **shift the model's effective behavior**
toward stating true things and toward recognizing and expressing when
it doesn't have reliable grounds for a claim, using techniques that
change what the model conditions on (grounding), what format its claims
must take (citation), and what independent checks run against its output
(verification) — none of which touch or "fix" the underlying generation
mechanism itself, because that mechanism cannot be fixed in this sense.

### 6.2 Disentangling the causes — one word, several distinct problems

"Hallucination" as commonly used lumps together at least four distinct
failure modes, and conflating them leads to applying the wrong
mitigation:

- **Genuine knowledge gaps**: the model was never trained on information
  needed to answer correctly (a fact past its training cutoff, an
  obscure or private fact), and generates a plausible-sounding guess
  rather than recognizing and stating the gap. **Mitigation target**:
  grounding (Section 6.3) — give the model the actual information it
  needs rather than relying on parametric memory.
- **Confident extrapolation**: the model has partial, related knowledge
  and extrapolates beyond what it actually knows with confidence,
  producing a specific-sounding but unverified detail (a fabricated
  citation, an invented specific number). **Mitigation target**:
  citation requirements (Section 6.4) that force explicit sourcing,
  making unsupported extrapolation visible rather than seamlessly
  blended into fluent prose.
- **Confabulation under compaction**: already identified specifically in
  Module 4, Section 6.3 — summarization/extraction steps can invent
  detail that wasn't in the original source. **Mitigation**: already
  addressed there via structured extraction; this module generalizes
  the same principle.
- **Instruction-following failure producing a fabricated-looking
  answer**: the model was explicitly instructed to say "I don't know"
  when uncertain but didn't reliably follow that instruction (Module 1,
  Section 6.1's point that instructions are soft, probabilistic
  nudges, not enforced behavior). **Mitigation target**: this is exactly
  why calibrated uncertainty (Section 6.5) needs more structural support
  than a bare instruction, and why verification passes (Section 6.6)
  matter as an independent check rather than relying on the same call
  to self-report its own uncertainty reliably.

### 6.3 Grounding: conditioning on supplied facts instead of parametric memory

**Grounding** means structuring the prompt so the model's claims are
based on information explicitly present in the context (retrieved
documents, tool results, supplied data) rather than solely on whatever
it learned during pretraining (its "parametric knowledge"). This
directly leverages Module 1's in-context mechanism (Section 6.2 of that
module): information present in the prefix has a much more direct,
traceable influence on generation than the diffuse, compressed knowledge
baked into the model's weights, and — critically — grounded generation
gives you something checkable: you can verify a generated claim against
the supplied source text directly, which is structurally impossible for
a claim drawn purely from parametric memory with no external reference
point.

**This is the single biggest lever available**, and it's exactly what
Part 4's entire RAG curriculum is built around at scale — retrieving
relevant, verified documents and grounding generation in them rather
than relying on the model's parametric memory for facts. This module
covers grounding at the conceptual/mechanism level; Part 4 covers it as
a full production architecture. **What grounding does not do**: it
doesn't prevent the model from ignoring the supplied grounding context
and still generating an ungrounded claim (a real, measurable failure
mode called "unfaithfulness" to the source), nor does it help at all if
the supplied grounding material is itself wrong — grounding shifts
where the risk lives, from "the model's parametric memory might be
wrong" to "the model might not faithfully use the correct material you
gave it, or the material itself might be wrong," both of which are
narrower, more checkable risks than the original problem.

### 6.4 Citation and attribution: making unsupported claims visible

Requiring the model to cite a specific source for each factual claim
(implemented via Module 2's schema-constrained output — e.g., a
structured response format with a `claim` field paired with a
`source_reference` field) serves a genuinely useful purpose even though
it doesn't guarantee correctness: **it makes ungrounded claims visible
and checkable rather than seamlessly blended into fluent, equally
confident-sounding prose.** A claim with no valid citation, or a citation
that doesn't actually support the claim when checked, is a concrete,
detectable signal you can act on (flag it, suppress it, route it to
verification) — compared to an uncited claim, which offers no such
signal at all.

**The critical caveat, worth stating explicitly**: citation itself can be
confabulated — the model can generate a citation-shaped reference that
looks plausible (a real-sounding source name, a real-sounding page
number) but doesn't actually support the claim, or doesn't exist at all.
This is precisely the same generation mechanism (Section 6.1) applied to
the citation text itself, not a separate, more trustworthy process.
**Never treat the mere presence of a citation as proof of correctness**
— a citation must itself be checked against the actual source (this is
exactly what a verification pass, Section 6.6, is for), or citation
becomes a false signal of rigor that's arguably worse than no citation
at all, since it invites misplaced trust.

### 6.5 Calibrated uncertainty: why "tell me if you're not sure" isn't enough

The naive approach — instructing the model, "if you're not confident,
say you don't know" — is a Module 1, Section 6.1-style soft prompt
instruction, and inherits exactly the reliability limits that implies:
it competes with the SFT/RLHF-trained default toward providing a
helpful-sounding, complete answer (Part 2, Module 7), and there's no
guarantee the model's internal "confidence" (to the extent that's even
a coherent concept for a next-token-sampling process) correlates
reliably with the actual correctness of its claim.

**More structural approaches, in increasing rigor**:

- **Explicit refusal categories in a schema** (Module 2): rather than
  free-text "I don't know," provide a schema field like
  `answer_status: Literal["answered", "insufficient_information",
  "requires_verification"]`, forcing the model to make an explicit,
  checkable categorical decision rather than hoping natural-language
  hedging happens reliably.
- **Grounding-dependent confidence**: if a claim cannot be traced to
  supplied grounding material (Section 6.3) at all, that absence itself
  is a strong, checkable signal (independent of the model's own
  self-reported confidence) that the claim should be flagged — this
  reframes "confidence" from an unreliable self-report into an
  externally-verifiable property of whether grounding material actually
  supports the claim.
- **Cross-checking via a second, independent call** (the verification
  pass, Section 6.6) — the most rigorous option, because it doesn't rely
  on the original generation call's own self-assessment at all.

### 6.6 Verification passes: an independent second check

A **verification pass** is a second, distinct model or retrieval call
that checks a first response's factual claims against grounding material
or external sources, rather than trusting the first call's own
self-reported confidence. This is architecturally similar to Module 6's
guardrail pattern (an independent check separate from the primary
generation), applied specifically to factual-claim verification rather
than policy compliance.

**A concrete design**: extract individual factual claims from the first
response (using Module 2's structured extraction, exactly as in Module
4's compaction work), then for each claim, issue a targeted, narrowly-
scoped verification call asking specifically "does the supplied
grounding material support this specific claim?" — a much narrower,
more checkable question than "is this whole response correct?", and one
where a schema-constrained yes/no/partially answer (Module 2) is far
more reliable than an open-ended judgment (directly echoing Module 5 and
Module 6's rubric-based-judge discipline).

**The unavoidable cost/latency tradeoff** (Module 9's framework applies
directly): a verification pass means at least one additional model call
per response (or per claim, if verifying claim-by-claim), directly
increasing cost and total latency. This is exactly the kind of
explicit, per-call-type policy decision Module 8 and Module 9 already
established the discipline for — verification passes are worth their
cost for high-stakes claims (medical, legal, financial, anything
feeding an irreversible action per Module 3's tool-calling risk
framing) and often not worth it for low-stakes, easily-correctable
conversational content. Decide this explicitly, per use case, rather
than applying verification uniformly or never.

### 6.7 Measuring hallucination rate — the same discipline as everywhere else in Part 3

Consistent with every module in this part: **an unmeasured hallucination-
reduction technique is exactly as unreliable as an unmeasured prompt,
guardrail, or routing policy.** Build a golden dataset (`eval-framework`)
containing test cases specifically designed to probe for hallucination —
questions the model genuinely cannot know the answer to, questions where
correct grounding material is supplied alongside distractors, and
questions probing whether cited sources actually support their claims —
and measure hallucination rate as a first-class metric alongside
accuracy, exactly as Module 5 insisted on trace-level metrics and Module
6 insisted on false-positive/negative rates for guardrails. Report it
with confidence intervals (Module 5, Section 6.5's discipline), and
track it over time via `observability-stack` the same as every other
pipeline quality metric in this part.

---

## 7. Mental Models

1. **"Hallucination isn't a bug in the generation mechanism — true and
   false statements come from the identical process."** You're not
   fixing a broken step; you're shifting behavior via what the model
   conditions on, what format it must use, and what independently checks
   it.
2. **"'Hallucination' is at least four different problems wearing one
   name."** Knowledge gaps, confident extrapolation, compaction
   confabulation, and instruction-following failure each need a
   different, targeted mitigation.
3. **"A citation is not proof — it's a checkable claim, and it must
   itself be checked."** Confabulated citations look exactly like real
   ones, because they're generated by the same mechanism as any other
   claim.
4. **"Verification passes trade cost for confidence, deliberately, and
   only where the stakes justify it."** Apply the same explicit,
   per-use-case policy discipline as caching and routing — never
   uniformly, never never.

---

## 8. Visual Explanation

**Diagram 1 — Hallucination as one mechanism producing two labels**

```
Generation mechanism (Module 1, §6.1): sample next token from
learned distribution, repeat.

                    ┌─────────────────────┐
                    │  SAME MECHANISM       │
                    │  produces both:       │
                    └─────────┬────────────┘
                              │
                ┌─────────────┴─────────────┐
                ▼                           ▼
        "The capital of         "The capital of
         France is Paris"        France is Lyon"
              TRUE                    FALSE
        (no structural difference in HOW either was generated —
         only in how the learned distribution happened to weight
         the relevant tokens)
```

**Diagram 2 — Four causes, four targeted mitigations**

```
Cause                          Targeted mitigation
──────────────────────────────────────────────────────────
Genuine knowledge gap     →    Grounding (supply the fact)
Confident extrapolation   →    Citation requirements
                                (force explicit sourcing)
Compaction confabulation  →    Structured extraction
                                (Module 4, §6.3 — already built)
Instruction-following     →    Schema-constrained refusal
failure                         categories + verification pass
                                (don't rely on self-report alone)
```

**Diagram 3 — Verification pass architecture**

```
[First response] ──► [extract individual claims] (Module 2 schema)
                              │
              ┌───────────────┼───────────────┐
              ▼               ▼               ▼
      claim 1 + ground   claim 2 + ground  claim 3 + ground
      material  ──►         material ──►      material ──►
      [narrow verify    [narrow verify    [narrow verify
       call: yes/no/      call]            call]
       partial]
              │               │               │
              └───────────────┼───────────────┘
                              ▼
                  [aggregate: flag unsupported
                   claims before returning to user]

Cost: +1 model call per claim verified — apply per Module 9's
      explicit, use-case-specific policy discipline, not uniformly.
```

---

## 9. Recommended Resources

1. **Anthropic and OpenAI — published guidance on reducing
   hallucinations** (docs.claude.com, platform.openai.com/docs, or
   engineering blog posts from either) — read for the vendors' own
   current, evolving recommended practices, since this is an actively
   developing area.
2. **Research on retrieval-augmented generation and faithfulness**
   (search for well-cited papers on "faithfulness" and "attribution" in
   grounded generation — you'll have direct hands-on RAG experience
   soon in Part 4, so read this now as motivation) — read specifically
   for how "faithfulness to source" is measured, since this is exactly
   the property grounding (Section 6.3) targets and citation
   verification (Section 6.6) checks.
3. **Any well-cited benchmark or evaluation suite specifically targeting
   factuality/hallucination** (search for a current, actively-maintained
   factuality benchmark) — read its methodology for how it constructs
   test cases probing genuine knowledge gaps versus confident
   extrapolation, since this directly informs your own golden-dataset
   design in Section 6.7.
4. **Your own `judge-bias-lab` and `eval-framework` work (Part 2, Module
   8)** — the rubric-based, schema-constrained judging discipline
   transfers directly to designing the narrow verification-call prompts
   in Section 6.6; re-read before building them.

---

## 10. Exercises

1. **Induce each of the four causes, deliberately.** Design one test
   prompt targeting each of Section 6.2's four distinct causes (a
   genuine knowledge gap — ask about something clearly outside training
   data; a confident-extrapolation trigger — ask for a specific,
   checkable detail the model likely doesn't actually know; a
   compaction-confabulation trigger — reuse Module 4's Exercise 3; an
   instruction-following-failure trigger — instruct "say you don't know
   if unsure" on a genuinely unanswerable question and see if it
   complies). Confirm you can reproduce each distinct failure mode.
2. **Measure grounding's real effect.** Run the genuine-knowledge-gap
   test case from Exercise 1 both without any supplied grounding
   material and with correct, relevant grounding material supplied.
   Compare accuracy and note whether the model still occasionally
   ignores the supplied grounding (the "unfaithfulness" risk from
   Section 6.3).
3. **Break citation, deliberately.** Ask the model a question likely to
   trigger confident extrapolation, requiring a citation in a
   schema-constrained format (Module 2). Manually verify each returned
   citation against a real source. Record how often a citation is
   present but doesn't actually support the claim (or references a
   source that doesn't exist).
4. **Build and test a schema-constrained refusal.** Implement the
   `answer_status` schema field from Section 6.5 for a set of questions
   spanning clearly-answerable, genuinely-unanswerable, and ambiguous
   cases. Measure how often the model correctly selects
   `insufficient_information` versus fabricating an answer.
5. **Build and cost out a verification pass.** Implement the claim-by-
   claim verification architecture from Section 6.6 for one high-stakes
   test case. Measure the additional latency/cost incurred and the
   fraction of claims correctly flagged as unsupported, and make an
   explicit judgment (per Module 9's framework) about whether that
   tradeoff is worth it for this specific use case.

---

## 11. Mini-Project: `hallucination-eval-suite`

A small standalone tool, built against `eval-framework`, containing
golden test cases for each of Section 6.2's four causes plus a
citation-faithfulness check (does the cited source actually support the
claim), reporting hallucination rate per cause category with confidence
intervals — the direct measurement discipline Section 6.7 requires,
built once and reusable as a permanent quality gate for the production
project below and for Part 4's RAG work later.

---

## 12. Production Project: Hallucination Reduction Layer in `llm-client-core`

### 12.1 What you're building

1. **Grounding enforcement in `PromptTemplate`**: extend Module 1's
   template anatomy to support an explicit "grounding material" slot
   (distinct from few-shot examples or conversation history) with a
   prompt instruction requiring claims to be traceable to that material,
   and a schema-constrained `answer_status` field (Section 6.5) as a
   standard part of any grounded-response schema.

2. **Citation enforcement extending Module 2**: a reusable
   `CitedClaim` Pydantic model (`claim: str`, `source_reference: str`)
   usable anywhere in the pipeline that needs attributed output, plus a
   citation-verification checker (comparing a cited claim against the
   actual supplied grounding material, not just checking the citation
   field is non-empty).

3. **A verification-pass component**, implementing Section 6.6's
   claim-extraction-plus-narrow-verification architecture, with an
   explicit, documented per-call-type policy (Module 9's discipline)
   for when it's enabled, reusing `llm-client-core`'s existing adapters
   for the verification calls themselves.

4. **`hallucination-eval-suite` integration**: measure hallucination rate
   before and after applying grounding, citation, and verification
   individually and in combination, on the same golden set, so you have
   real evidence of each technique's individual and combined
   contribution — not just an assumption that "we added mitigations, so
   it must be better."

5. **Observability**: emit hallucination-rate and verification-pass
   cost/latency metrics via `observability-stack`, tracked over time as
   a first-class pipeline quality metric alongside everything else
   Part 3 has instrumented.

### 12.2 What this sets up for later modules

- **Part 3's capstone** integrates this hallucination-reduction layer as
  a non-optional quality component of the finished production service.
- **Part 4 (RAG)** is, in substantial part, the grounding mechanism from
  Section 6.3 built out into a full retrieval architecture — you're
  building the conceptual and schema foundation for it here.
- **Part 5 (Agents)** will need verification passes specifically before
  any consequential tool-calling action, directly extending this
  module's cost/benefit framework for when verification is worth its
  overhead.

### 12.3 Definition of done

- `hallucination-eval-suite` measures hallucination rate across all four
  cause categories plus citation-faithfulness, with confidence
  intervals.
- Grounding, citation enforcement, and verification passes are each
  individually measurable in their contribution to hallucination-rate
  reduction on the golden set.
- The verification-pass component has an explicit, documented
  enable/disable policy per call type, justified by cost/benefit.
- Hallucination-rate and verification-cost metrics are visible in
  `observability-stack`, tracked as an ongoing quality signal.

---

## 13. Practice & Interview Questions

1. Explain why hallucination cannot be "fixed" at the generation
   mechanism level, and what the available mitigations actually change
   instead.
2. Name the four distinct causes commonly lumped under "hallucination"
   and the targeted mitigation for each.
3. Why is the presence of a citation insufficient evidence that a claim
   is correct, and what does this imply about how citations must be
   used in a production system?
4. Why is "just tell the model to say it doesn't know if it's not sure"
   an unreliable mitigation on its own, and what more structural
   alternative would you use instead?
5. Design a verification-pass architecture for a high-stakes use case
   (e.g., a medical-information assistant) and justify, using Module 9's
   framework, why the added cost/latency is worth it there specifically.
6. How does grounding relate to Part 4's RAG architecture, and what does
   grounding not solve even when correctly implemented?

---

## 14. Common Mistakes

- **Treating any single technique (grounding, citation, or verification)
  as "solving" hallucination.** Each reduces risk for a specific cause;
  none eliminates the underlying mechanism.
- **Trusting citation presence as proof of correctness** without
  verifying the citation actually supports the claim — this is exactly
  the confabulated-citation risk Section 6.4 warns about.
- **Relying solely on a bare "say you don't know if unsure" instruction**
  instead of a more structural, schema-constrained refusal mechanism.
- **Applying verification passes uniformly (always or never)** instead
  of an explicit, cost-justified, per-use-case policy.
- **Measuring "we added mitigations" as success**, without actually
  measuring hallucination rate before and after via
  `hallucination-eval-suite` — precisely the unmeasured-technique
  mistake this entire part of the handbook has repeatedly warned
  against.
- **Conflating grounding-material quality with grounding-faithfulness.**
  Supplying correct grounding material doesn't guarantee the model
  actually uses it faithfully — both must be checked separately.

---

## 15. Debugging Exercise

Your `hallucination-eval-suite` shows a low overall hallucination rate
in aggregate, but a recent incident involved the assistant confidently
citing a specific document section that, upon manual check, does not
actually contain the claimed information — the citation format was
perfectly well-formed and passed your schema validation (Module 2), but
the underlying claim-to-source match was wrong.

Write down at least two concrete hypotheses for why your evaluation
suite's aggregate hallucination rate didn't surface this (consider: does
your citation-faithfulness check actually verify semantic support for
the claim, or does it only verify that the `source_reference` field is
non-empty and schema-valid — a distinction Section 6.4 explicitly
warned about? does your golden set have adequate coverage for this
specific document/section, or is this another instance of the
coverage-gap pattern already seen in Module 5 and Module 9's debugging
exercises?), and describe concretely how you'd strengthen the
citation-faithfulness check to catch this class of failure going
forward.

---

## 16. Checklist

- [ ] I can explain why hallucination is a structural property of
      generation rather than a fixable bug, and what mitigations actually
      change instead.
- [ ] I can name and distinguish the four distinct causes commonly
      lumped under "hallucination" and their targeted mitigations.
- [ ] I can explain why citation presence alone is not proof of
      correctness, and how confabulated citations arise from the same
      mechanism as any other claim.
- [ ] I can design a schema-constrained refusal mechanism more reliable
      than a bare "say you don't know" instruction.
- [ ] I can design a verification-pass architecture and justify its
      cost/benefit for a specific, high-stakes use case.
- [ ] `hallucination-eval-suite` is built and measures hallucination rate
      across all four cause categories plus citation faithfulness.
- [ ] Grounding, citation enforcement, and verification passes are each
      implemented in `llm-client-core` with individually-measured
      contributions to hallucination reduction.
- [ ] The verification-pass component has an explicit, cost-justified,
      per-call-type enable/disable policy.
- [ ] Hallucination-rate and verification-cost metrics are visible in
      `observability-stack`.

---

## 17. Summary

Hallucination is not a separate, faulty generation mode — true and false
statements are produced by the identical next-token sampling mechanism
established all the way back in Module 1, which means no technique in
this module "fixes" generation itself; each instead shifts what the
model conditions on, what format its claims must take, or what
independently checks its output. The word "hallucination" covers at
least four genuinely distinct causes — knowledge gaps, confident
extrapolation, compaction confabulation, and instruction-following
failure — each requiring a different, targeted mitigation rather than
one universal fix. Grounding reduces reliance on unreliable parametric
memory but introduces its own faithfulness risk; citation makes
unsupported claims visible and checkable but can itself be confabulated
and must be independently verified; calibrated uncertainty needs
structural, schema-constrained support rather than a bare instruction;
and verification passes provide the most rigorous independent check, at
an explicit, deliberately-decided cost. `llm-client-core` now has a
measured, evidence-based hallucination-reduction layer — grounding
support, citation enforcement with faithfulness checking, and a
cost-justified verification-pass component — directly setting up Part
4's RAG architecture, which is grounding built out to full production
scale.

---

## 18. Next Steps

**Next: Part 3, Module 11 — Multi-Turn Conversation Design.** With
prompting, structured output, tool calling, memory, evaluation,
guardrails, context engineering, caching, latency/cost optimization, and
hallucination reduction all now built and measured as individual
components, this module addresses the product/UX-adjacent design layer
that ties them together across a real, extended conversation — turn-
taking patterns, clarification-seeking behavior, conversation repair
after a misunderstanding, and session boundaries — directly preceding
Part 3's capstone module, which synthesizes everything built so far into
one finished production conversational AI service.

Reply "continue" for Module 11, or flag anything to go deeper on.
