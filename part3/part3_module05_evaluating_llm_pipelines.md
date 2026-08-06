# Part 3, Module 5: Evaluating Full LLM-Powered Pipelines

> Modules 1-4 built four distinct components: prompting, structured
> output, tool calling, and memory. Each was evaluated in isolation as
> you built it. Production systems chain all four together into one
> pipeline — and a pipeline's failure modes are not the union of its
> components' individual failure modes. This module builds the
> discipline for evaluating the whole thing, where errors compound
> silently across stages in ways no single-component eval can catch.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain why per-component evaluation (each stage scoring well in
   isolation) does not imply pipeline-level correctness, and identify the
   specific mechanism — error compounding across sequential stages —
   that causes this gap.
2. Distinguish evaluating a single LLM call (Part 2, Module 8's original
   scope) from evaluating a multi-stage pipeline (prompt → tool calls →
   memory retrieval → final generation), and design evals that target
   the pipeline's actual failure surface, not just its final output.
3. Build pipeline-level golden datasets that specify expected behavior
   at intermediate stages, not only the final answer, so failures can be
   localized to the stage that caused them.
4. Apply LLM-as-judge evaluation correctly to full pipeline outputs,
   while explicitly correcting for the bias patterns already
   characterized in `judge-bias-lab` (Part 2, Module 8).
5. Design and interpret regression testing for LLM pipelines — detecting
   when a change to one stage (a prompt tweak, a new tool, a different
   compaction strategy) silently degrades a different stage's behavior.
6. Extend `eval-framework` into a genuine pipeline-evaluation framework
   capable of scoring the entire `llm-client-core` stack (Modules 1-4)
   end-to-end, with stage-level attribution of failures.

---

## 2. Prerequisites

- **Part 2, Module 8** (`eval-framework` and `judge-bias-lab`) — this
  module extends that framework directly; you need its golden-dataset,
  pluggable-scorer, and bias-mitigated LLM-judge architecture fresh.
- **Part 3, Modules 1-4** — the pipeline under evaluation in this
  module's production project is exactly the accumulated
  `llm-client-core` stack: prompting, structured output, tool calling,
  and memory.
- **Part 1, Module 10** (Performance & Profiling) — the profiling
  mindset (isolate where in a system a problem originates before fixing
  it) transfers directly to stage-level failure attribution here.

---

## 3. Estimated Study Time

**7–9 hours** (3 hours theory/reading, 4–6 hours hands-on).

---

## 4. Difficulty

★★★★☆ (4/5)

Not conceptually novel individually, but genuinely hard in practice:
designing golden datasets that pin down intermediate-stage expectations
(not just final answers) requires real discipline, and diagnosing
compounding errors requires the same systematic isolation skill as
debugging any multi-stage distributed system.

---

## 5. Why This Matters

A depressingly common production pattern: each individual component (the
prompt template, the tool-selection logic, the memory retrieval) passed
its own unit-level eval with a high score, and the assembled system still
produces a wrong or unsafe answer in the field, because a small,
individually-tolerable imprecision in one stage fed a slightly-wrong
input into the next stage, which amplified it, which fed an even-more-
wrong input into the stage after that. This is exactly the systems-level
failure mode that separates "I can prompt a model well" from "I can
build and maintain a reliable AI-powered system in production" — which
is precisely the bar senior AI engineering roles and interviews
(including DoorDash-style system-design rounds) actually probe for.

It's also the direct foundation for Part 3's next module (Guardrails):
you cannot design a sensible guardrail without first knowing, with
evidence rather than intuition, where your pipeline actually tends to
fail.

---

## 6. Theory

### 6.1 Why per-stage evaluation doesn't guarantee pipeline correctness

Consider the `llm-client-core` pipeline as it now stands: a user message
arrives, a `PromptTemplate` renders it (Module 1), the model may select
and call one or more tools (Module 3), a `LongTermMemoryStore` retrieval
may inject prior context (Module 4), and a structured or free-text final
response is generated and validated (Module 2). Each of those four
stages, evaluated independently, might score 90%+ on its own golden
dataset. **The probability that a random end-to-end trace passes through
all four stages without any compounding error is not simply the product
of the individual pass rates treated as independent events** — in
practice it is often *worse* than that naive product, because errors are
frequently correlated (a genuinely ambiguous user query is simultaneously
harder for prompt-following, tool selection, *and* memory retrieval, so
failures cluster on the same difficult inputs rather than distributing
independently) and because errors compound directionally: a slightly
mis-parsed tool argument (a near-miss, still schema-valid per Module 2's
distinction between syntactic and semantic validity) doesn't just fail
independently at the tool stage — it actively corrupts the input to
every subsequent stage, which then can only fail or, at best, luckily
recover.

**The correct mental model**: pipeline evaluation is not "average the
per-stage scores." It requires tracing complete, realistic multi-stage
executions end-to-end and checking both the final output *and* the
correctness of the intermediate handoffs between stages, so that when a
failure is found, you can localize which stage introduced it rather than
only knowing the final answer was wrong.

### 6.2 Golden datasets for pipelines: specify intermediate expectations, not just final answers

A single-call golden dataset (Part 2, Module 8) specifies `(input,
expected_output)`. A pipeline golden dataset needs to specify expected
behavior at each meaningful stage boundary, because a final-answer-only
dataset cannot distinguish "the whole pipeline is right" from "two
separate mistakes happened to cancel out and produce a right-looking
final answer for the wrong reason" — the latter is a ticking time bomb
that a final-answer-only eval will never surface, and a classic reason
systems that "pass all their evals" still fail unpredictably in
production.

A well-designed pipeline test case for `llm-client-core` should specify:

```
{
  "input": "What's the status of order #4471, and email me a summary
             if it's delayed?",
  "expected_tool_calls": [
    {"tool": "get_order_status", "args": {"order_id": "4471"}}
  ],
  "expected_conditional_behavior": "only call send_email if status
             indicates a delay",
  "expected_memory_retrieval": null,  // no relevant stored memory expected
  "expected_final_output_schema": "OrderStatusResponse",
  "expected_final_output_constraints": {
      "must_mention": ["order #4471"],
      "must_not_call_send_email_if": "status == 'on_time'"
  }
}
```

Note what this buys you that a final-answer-only test doesn't: if the
final output looks correct but the trace shows `send_email` was called
even though the order was on time, you've caught a real bug (an
over-eager or incorrectly-conditioned tool call) that a final-answer
check alone might have missed entirely if the email tool's actual side
effect wasn't visible in the returned text.

### 6.3 LLM-as-judge at the pipeline level: same bias risks, higher stakes

`judge-bias-lab` (Part 2, Module 8) already demonstrated, hands-on, that
an LLM judge has measurable position and verbosity biases. At the
pipeline level, judging becomes harder, not easier, because you are now
often asking a judge to assess something more holistic — "did this
multi-step interaction accomplish the user's actual intent, including
correctly deciding *not* to take an action" — which is exactly the kind
of judgment call where an under-specified judge prompt reverts to
generic, surface-level heuristics (fluency, length, tone) rather than
the specific correctness criteria you actually care about.

**The fix is the same discipline `eval-framework` already established,
applied more rigorously**: give the judge model a structured rubric
tied directly to the golden dataset's specified expectations (Section
6.2), not an open-ended "did this look right?" prompt — this is
Module 2's schema-constrained-output thinking applied to the judge
itself, forcing it to score against explicit, enumerated criteria
(correct tool called? correct arguments? correct conditional decision?
final answer schema-valid?) rather than an unconstrained holistic
impression that inherits all of `judge-bias-lab`'s known bias patterns
at a larger, harder-to-audit scale.

### 6.4 Regression testing: catching cross-stage side effects of local changes

A change that looks purely local — tightening a tool's description
(Module 3, Section 6.3) to fix an ambiguous-selection bug, or adjusting
the compaction token budget (Module 4) — can silently change behavior in
an entirely different stage, because every stage shares the same
underlying conversation-state prefix (Module 1, Section 6.1). A tool
description edit can shift *which* tool the model selects for borderline
queries elsewhere in your golden set; a compaction change can alter what
context is available when the model makes a *later* tool-call decision.

**This is exactly the same discipline as regression testing in
conventional software** (Part 1, Module 8's CI/CD content) applied to a
system whose "unit tests" are probabilistic rather than deterministic.
The practical consequence: your pipeline eval suite must run on every
change to *any* stage, not just the stage that was directly modified,
and you should track per-stage pass rates over time (via
`observability-stack`, Part 1 Module 4) to catch a slow, silent
regression in one stage caused by a change made for a different stage's
benefit — this is a real, recurring failure pattern in production LLM
systems and one of the main reasons ad hoc, manual "looks good to me"
testing of prompt/tool changes is insufficient at any real scale.

### 6.5 Statistical rigor at the pipeline level

Recall from `eval-framework` (Part 2, Module 8) that single-call
evaluation already required confidence intervals, not just point
estimates, because LLM outputs are stochastic. This matters even more at
the pipeline level, because compounding stochastic stages produce wider
variance in end-to-end outcomes than any single stage alone — a pipeline
that "passed" on one run of your golden set is meaningfully less
informative than a single-call eval passing once, precisely because more
independent (and correlated) sources of randomness are stacked together.
Run your pipeline golden set multiple times per evaluated change and
report pass rate with a confidence interval, exactly as `eval-framework`
already does for single calls — do not treat a pipeline eval's single-run
result as more certain than it actually is.

---

## 7. Mental Models

1. **"Passing every stage doesn't mean passing the pipeline."** Errors
   compound directionally and correlate on hard inputs — treat
   pipeline correctness as its own thing to measure, not an inference
   from component scores.
2. **"A pipeline golden dataset specifies the trace, not just the
   destination."** If you only check the final answer, two wrongs can
   look like a right, and you'll never find out until production.
3. **"A holistic judge prompt inherits every bias `judge-bias-lab`
   already taught you about — at a scale that's harder to audit."** Give
   the judge an explicit rubric tied to your golden dataset's specified
   stage expectations, not an open-ended "did this seem fine?"
4. **"A local change can cause a non-local regression."** Any edit to
   any stage requires re-running the *entire* pipeline eval suite, not
   just the stage you touched.

---

## 8. Visual Explanation

**Diagram 1 — Compounding error across stages**

```
Stage 1 (prompt/intent parse): 92% correct
Stage 2 (tool selection):      90% correct
Stage 3 (memory retrieval):    88% correct
Stage 4 (final generation):    91% correct

NAIVE (wrong) intuition: pipeline ≈ high 80s%, "should be fine"

REALITY: errors correlate on hard/ambiguous inputs, and a Stage 2
mistake corrupts the INPUT to Stage 3 and 4 (not an independent draw) —
actual end-to-end correctness is frequently well below the naive
product, and must be MEASURED end-to-end, never estimated from parts.
```

**Diagram 2 — Trace-level vs. final-answer-only evaluation**

```
FINAL-ANSWER-ONLY EVAL:
  input ──► [ pipeline: prompt→tools→memory→generate ] ──► output
                                                              │
                                                    score output only
                    (misses: right output, WRONG reason — e.g. lucky
                     tool-call cancellation, described in 6.2)

TRACE-LEVEL EVAL:
  input ──► [prompt] ──► [tool calls] ──► [memory] ──► [generate] ──► output
               │              │              │             │           │
            check          check          check         check       check
          (matches       (matches      (matches       (matches    (matches
           golden          golden        golden         golden      golden
           intent)         tool set)     retrieval)     schema)     answer)
```

**Diagram 3 — Regression testing across stages**

```
Change made to: Tool description (Module 3 stage)
                        │
                        ▼
Must re-run: ENTIRE pipeline golden set
                        │
         ┌──────────────┼──────────────┬───────────────┐
         ▼              ▼              ▼               ▼
     Prompt stage   Tool stage    Memory stage    Generation stage
     (unaffected     (directly      (check for      (check for
      — verify)       intended       unintended       unintended
                       fix, verify)   side effects)   side effects)
```

---

## 9. Recommended Resources

1. **Your own `eval-framework` and `judge-bias-lab` codebases (Part 2,
   Module 8)** — the primary resource here is genuinely your own prior
   work; re-read the scorer architecture and bias-mitigation design
   before extending it, since this module's goal is extension, not
   replacement.
2. **Anthropic — "Building effective agents" or equivalent engineering
   guidance on evaluating multi-step LLM systems** (anthropic.com/
   engineering or docs.claude.com) — read specifically for how the
   vendor frames evaluation of systems with tool use and multi-step
   behavior, since this directly informs Section 6.2's trace-level
   design.
3. **OpenAI Evals framework documentation and repository**
   (github.com/openai/evals) — read the architecture of a real,
   widely-used open-source eval framework specifically for how it
   handles multi-turn/multi-step test cases, and compare its design
   choices against your own `eval-framework`'s extension.
4. **Literature or vendor documentation on "LLM-as-judge" best
   practices** (search for recent guidance from Anthropic, OpenAI, or a
   well-cited academic source on calibrating LLM judges) — read
   specifically for rubric-based judging design (Section 6.3), since
   this is the part most teams get wrong by using unstructured judge
   prompts.
5. **Part 1, Module 8's CI/CD pipeline design** (your own prior work) —
   review how you built tiered fast/slow CI for `auth-gateway`, since the
   production project below effectively builds the LLM-pipeline
   equivalent of that same tiered-testing discipline.

---

## 10. Exercises

1. **Prove the compounding-error effect, empirically.** Using your
   actual `llm-client-core` stack, construct 15-20 deliberately
   borderline/ambiguous test inputs (not easy ones). Score each of the
   four stages independently, then score the full end-to-end trace.
   Compare the naive product of stage pass rates against your measured
   end-to-end pass rate and quantify the gap.
2. **Build one trace-level golden test case by hand**, following the
   structure in Section 6.2, for a scenario involving a conditional tool
   call (a tool that should only be called under certain conditions).
   Run it against your pipeline and confirm your eval correctly
   distinguishes "right final answer, wrong intermediate behavior" from
   genuine correctness.
3. **Design a rubric-based judge prompt** for scoring an end-to-end trace
   against your Exercise 2 test case, structured per Section 6.3 (Module
   2-style schema-constrained scoring, not open-ended). Compare its
   verdicts against your own manual judgment on 10 traces and check for
   disagreement patterns.
4. **Induce and catch a regression, deliberately.** Make a real change to
   one stage (e.g., tighten or loosen a tool description) that you
   suspect could have a side effect elsewhere. Run your full pipeline eval
   suite before and after, and confirm whether it catches a regression in
   an unrelated stage — if it doesn't catch one you expected, diagnose
   why your golden set didn't cover that case, and add coverage.
5. **Confidence intervals at the pipeline level.** Run your full pipeline
   golden set 5 times (allowing for the stochasticity across all stages)
   and report a pass-rate confidence interval, not a single point
   estimate. Compare its width against a single-call eval's confidence
   interval from Part 2, Module 8's work, and explain why it's wider.

---

## 11. Mini-Project: `trace-recorder`

A small standalone tool that instruments a full `llm-client-core`
pipeline run and produces a structured, inspectable trace object
recording every stage's input and output (prompt rendering, tool-call
requests/results, memory retrievals, final validated output) — not just
the final answer. This trace format is the direct prerequisite for the
production project's pipeline eval framework, since you cannot check
intermediate-stage correctness (Section 6.2) without first being able to
capture what happened at each stage.

---

## 12. Production Project: Pipeline-Level Extension of `eval-framework`

### 12.1 What you're building

1. **A `PipelineGoldenDataset` format** extending `eval-framework`'s
   existing golden-dataset structure (Part 2, Module 8) to support the
   trace-level specification from Section 6.2: expected tool calls,
   expected conditional behavior, expected memory-retrieval behavior, and
   expected final-output constraints, per test case.

2. **A `TraceScorer`** that consumes a `trace-recorder` output and a
   `PipelineGoldenDataset` test case, and produces a stage-attributed
   score: which stage(s) passed, which failed, and — critically — cases
   where the final answer was correct but an intermediate stage was
   wrong (the "right for the wrong reason" case from Section 6.2), which
   must be flagged distinctly, not silently counted as a pass.

3. **A rubric-based `PipelineJudge`** (Section 6.3) using
   schema-constrained scoring (Module 2) against the same
   `PipelineGoldenDataset` expectations, explicitly designed to avoid
   the open-ended holistic-judgment pattern, and cross-validated against
   `judge-bias-lab`'s known bias checks (does the pipeline judge still
   exhibit position/verbosity bias when scoring full traces? verify, don't
   assume it's fixed just because the rubric is more structured).

4. **A pipeline-level CI integration**, extending Part 1, Module 8's
   tiered CI design: the full pipeline eval suite runs on every change to
   any of `llm-client-core`'s stages (prompting, structured output, tool
   calling, memory), with per-stage and end-to-end pass rates (with
   confidence intervals, per Section 6.5) tracked over time via
   `observability-stack`.

5. **A regression-catch demonstration**: deliberately reproduce
   Exercise 4's induced regression scenario as a permanent test in your
   suite, so it's provable — not just claimed — that this framework
   catches cross-stage regressions.

### 12.2 What this sets up for later modules

- **Part 3, Module 6 (Guardrails)** will use this module's trace-level
  failure attribution directly to decide *where* in the pipeline
  guardrail checks should be inserted, based on evidence of where
  failures actually cluster rather than guesswork.
- **Part 3's capstone** will use this full pipeline eval framework as its
  primary quality gate.
- **Part 4 (RAG)** and **Part 5 (Agents)** will both extend
  `PipelineGoldenDataset`/`TraceScorer` for their own, more complex
  multi-stage pipelines, rather than building parallel evaluation
  infrastructure from scratch.

### 12.3 Definition of done

- `PipelineGoldenDataset` and `TraceScorer` work against at least 15-20
  real test cases covering the full `llm-client-core` stack, with
  explicit stage-level attribution in every result.
- At least one "right final answer, wrong intermediate stage" case is
  present in your golden set and correctly flagged, not silently passed.
- `PipelineJudge` is rubric-based and has been checked (not assumed)
  against `judge-bias-lab`'s bias patterns at the pipeline-trace scale.
- Pipeline CI runs the full suite on any stage change and reports
  confidence-interval pass rates, tracked in `observability-stack`.
- The Exercise 4 regression scenario exists as a permanent, passing
  regression test that would have failed before the fix.

---

## 13. Practice & Interview Questions

1. Explain why four pipeline stages each individually scoring 90%+ does
   not imply the end-to-end pipeline scores anywhere near 90%, using the
   correlation and error-compounding arguments from Section 6.1.
2. What does a pipeline golden dataset need to specify that a
   single-call golden dataset does not, and why does skipping this leave
   a specific class of bug undetected?
3. Describe the "right answer for the wrong reason" failure mode
   concretely, with an example involving a tool call, and explain why a
   final-answer-only eval cannot catch it.
4. Why does an LLM-as-judge evaluating a full multi-step trace need a
   more structured, rubric-based prompt than one scoring a single
   response, given what you already know from `judge-bias-lab`?
5. A teammate tightens a tool's description to fix an unrelated
   selection-ambiguity bug and doesn't re-run the full pipeline eval
   suite, reasoning "I only touched one tool's description." What
   concretely could go wrong, and why is their reasoning insufficient?
6. Why does pipeline-level evaluation need wider confidence intervals
   (or more repeated runs) than single-call evaluation to reach the same
   level of statistical confidence?

---

## 14. Common Mistakes

- **Inferring pipeline quality from averaged component scores** instead
  of measuring end-to-end trace correctness directly — the single most
  common mistake this module exists to correct.
- **Final-answer-only golden datasets**, which cannot distinguish genuine
  correctness from lucky error-cancellation, and therefore give false
  confidence.
- **Open-ended, unstructured LLM-judge prompts at the pipeline level**,
  inheriting `judge-bias-lab`'s known biases at a scale that's much
  harder to manually audit than single-response judging.
- **Treating a stage-local code change as low-risk** and skipping full
  pipeline regression testing — this is exactly how silent, hard-to-
  trace production regressions get introduced.
- **Reporting a single pipeline-eval run's pass rate as if it were a
  stable, precise number**, ignoring the wider stochastic variance at
  the pipeline level (Section 6.5).
- **Building a second, parallel evaluation system for each new pipeline**
  (RAG, agents, etc.) instead of extending this module's
  `PipelineGoldenDataset`/`TraceScorer` abstractions — creates
  duplicated, drifting evaluation logic across the codebase.

---

## 15. Debugging Exercise

Your pipeline eval suite has been passing consistently for weeks. After a
routine change to the `LongTermMemoryStore`'s consolidation logic (Module
4), the suite still passes at the same aggregate rate — but a small
number of real user reports describe the assistant occasionally calling
a tool with subtly wrong arguments in ways that weren't happening before.

Write down at least two concrete hypotheses for why your existing
pipeline eval suite might be failing to catch this regression despite a
stable aggregate pass rate (consider: does your golden set have coverage
for the specific interaction between memory-retrieved context and
tool-argument construction? could this be a case where the failure rate
increased but stayed under your golden set's detection threshold because
the affected input pattern is underrepresented in your test cases?), and
describe concretely how you'd add trace-level coverage to close this gap
rather than simply lowering your pass-rate threshold to "catch more
things" arbitrarily.

---

## 16. Checklist

- [ ] I can explain, with the correlation/compounding argument, why
      per-stage pass rates don't predict end-to-end pipeline correctness.
- [ ] I can design a pipeline golden-dataset test case that specifies
      intermediate-stage expectations, not just a final answer.
- [ ] I can explain the "right answer, wrong reason" failure mode and why
      only trace-level evaluation catches it.
- [ ] I can design a rubric-based LLM-judge prompt for pipeline traces
      and explain why it's more resistant to `judge-bias-lab`'s known
      biases than an open-ended prompt.
- [ ] I understand why local stage changes require full-suite regression
      testing, not just testing the changed stage.
- [ ] `trace-recorder` produces a full, inspectable stage-by-stage trace
      for real pipeline runs.
- [ ] `PipelineGoldenDataset`/`TraceScorer` are built and correctly flag
      at least one "right for the wrong reason" case in my golden set.
- [ ] The pipeline CI integration runs the full suite on any stage change
      and reports confidence-interval pass rates via `observability-stack`.
- [ ] My deliberately-induced regression scenario exists as a permanent,
      passing regression test.

---

## 17. Summary

Evaluating each stage of an LLM pipeline in isolation — prompting,
structured output, tool calling, memory — is necessary but not
sufficient, because errors compound directionally across stages and
correlate on the same difficult inputs rather than failing independently,
which means end-to-end correctness must be measured directly, never
inferred from component scores. Pipeline golden datasets need to specify
expected behavior at intermediate stage boundaries, not just final
answers, or you'll never catch the specific and genuinely dangerous
"right answer for the wrong reason" failure mode. LLM-as-judge evaluation
at this scale needs the same rubric-based discipline `judge-bias-lab`
already proved necessary at the single-response scale, applied more
rigorously because holistic pipeline judgments revert to shallow
heuristics faster than single-response judgments do. `eval-framework` now
has trace-level golden datasets, stage-attributed scoring, a rubric-based
pipeline judge, and CI integration that catches cross-stage regressions
— this is the evidence-generating infrastructure the next module
(Guardrails) needs in order to place its checks where failures actually
happen, rather than where intuition guesses they might.

---

## 18. Next Steps

**Next: Part 3, Module 6 — Guardrails and Safety Filtering.** With
trace-level failure attribution now available from this module's
pipeline evaluation framework, Module 6 uses that evidence to design and
place guardrails — input filtering, output filtering, and action-level
safety checks — precisely at the pipeline stages where this module's
evals show failures actually concentrate, rather than applying uniform,
undifferentiated safety checks everywhere.

Reply "continue" for Module 6, or flag anything to go deeper on.
