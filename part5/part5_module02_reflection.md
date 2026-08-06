# Part 5, Module 2: Reflection

> Extends `agent-core` (Part 5, Module 1) with a self-critique step
> inserted into the existing loop — reusing the scratch-pad and
> termination guards you already built, not a parallel mechanism.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Define reflection precisely as an additional, distinct LLM call
   inside the loop — one that scores/critiques the *last action's
   outcome against the task*, separate from the call that decides the
   *next* action.
2. Explain why reflection is a special case of Part 3, Module 5's
   pipeline-evaluation lesson ("right answer, wrong reason") applied
   at every intermediate step of an agent run, not just at the final
   output.
3. Distinguish reflection that changes the *plan* from reflection that
   only changes *confidence/logging*, and implement both.
4. Identify the specific new failure mode reflection introduces
   (self-reinforcing false confidence) and the concrete mitigation for
   it.
5. Decide, with a stated cost/benefit argument, when reflection is worth
   its extra LLM call and when it's waste — reflection is not free, and
   this module treats "should every agent reflect after every step" as
   an empirical question, not a default.
6. Extend `agent-core` into `agent-core` v1.1 with a configurable
   reflection step, measured against `loop-stress-test` and a new
   reflection-specific eval.

## 2. Prerequisites

- Part 5, Module 1 (`agent-core`), completed and passing
  `loop-stress-test`.
- Part 3, Module 5 (Evaluating Full LLM-Powered Pipelines) — the
  "per-stage pass rates don't predict pipeline correctness" and
  "right answer, wrong reason" lessons are the direct conceptual parent
  of this module.
- Part 3, Module 10 (Hallucination Reduction) — the verification-pass
  mechanism you built there is the same mechanical pattern this module
  reuses for self-critique.

## 3. Estimated Study Time

8–10 hours across 2 sessions.

## 4. Difficulty

★★★☆☆ (3/5) — Small in code surface (this is a few hundred lines
added to an existing loop), but the judgment calls about when
reflection helps versus when it's theater require careful empirical
thinking, not just implementation.

---

## 5. Why This Matters

Without reflection, `agent-core`'s loop treats every observation as
face-value evidence that the last action succeeded: a tool call
returns something, the loop appends it to the scratch-pad, and moves
on. But "the tool call returned without erroring" and "the tool call
actually advanced the task" are different claims — the same gap Part 3,
Module 5 identified between "did each stage run" and "did the pipeline
actually work." An agent that searched for the wrong thing, retrieved an
irrelevant document, or misread a numeric result will confidently build
its next three steps on that mistake unless something explicitly checks
whether the last step's outcome actually matches what the plan needed.

This matters for exactly the reason multi-step systems are more fragile
than single-turn ones: errors compound. A single-turn system that's
wrong once is wrong once. A ten-step agent that's subtly wrong at step
2 and never checks itself carries that error through steps 3–10,
often amplifying it, and by the time a human notices the final output
is bad, there's no cheap way to find out which of ten steps was the
actual point of failure. Reflection is the mechanism that catches this
early, at the step where it happened, instead of at the end where it's
expensive to diagnose.

---

## 6. Theory

### 6.1 Reflection is a second, distinct LLM call — not a bigger prompt

It's tempting to fold reflection into the existing `think()` call by
asking the model to "consider whether the last step worked" as part of
deciding the next action. Resist this, for the same reason Part 3,
Module 2 separated generation from validation: a single call that's
asked to both critique the past and plan the future is a call with two
different jobs, and models (like people) do a worse job at self-critique
when it's bundled with "and also keep moving forward, don't dwell."
Separating them into two calls — `reflect(state) -> critique` then
`think(state, critique) -> next_action` — costs one extra LLM call per
iteration but produces a critique that isn't silently biased toward
"looks fine, let's continue" by the pressure to keep the plan moving.

### 6.2 What reflection actually checks

A precise definition, so this doesn't collapse into vague
"self-awareness": reflection is a call that takes (a) the sub-goal the
last action was supposed to serve, and (b) the actual observation
returned, and answers one structured question: **did this observation
plausibly satisfy the sub-goal, and if not, why not?** This is
mechanically identical to Part 3, Module 10's verification-pass idea
(checking a generated claim against retrieved context) — here, the
"claim" is the assumption baked into the plan ("this search will find
the relevant document") and the "context to check against" is what
actually came back.

Two categories of reflection, worth keeping structurally separate:

- **Confidence-only reflection**: produces a score/flag attached to the
  scratch-pad entry, without altering the plan. Cheap to add, useful
  for the final `RAGTraceScorer`/`agent`-equivalent trace analysis
  (Section 6.5) even when you choose not to act on it mid-run.
- **Plan-altering reflection**: when the critique concludes the sub-goal
  was not satisfied, it feeds back into `think()` as an explicit signal
  ("the last action did not find what step 2 needed; consider
  retrying with different arguments, or replanning entirely") — this is
  where reflection earns its cost, because it can prevent the loop from
  building three more steps on a broken foundation.

### 6.3 The new failure mode: self-reinforcing false confidence

Reflection introduces a specific new risk that wasn't present in Module
1's loop: if the same model, with the same biases and the same
(possibly wrong) understanding of the task, is asked to critique its own
work, it can systematically produce confident-sounding approval of
outcomes that are actually wrong — because the same misunderstanding
that produced the flawed action also shapes the critique of it. This is
not a hypothetical: it is the direct analog of Part 3, Module 6's point
about model-based guardrails needing calibration against real failure
data, and of Part 2, Module 8's judge-bias-lab lesson that an LLM judge
can share the same blind spots as the model it's judging.

**Mitigation, concretely:** where the stakes justify it, use a
*different* model (or the same model with a substantially different,
adversarially-framed prompt — "find the flaw in this reasoning," not
"does this look okay?") for reflection than for the main planning calls.
This is the same principle as Part 2, Module 8's bias-mitigated judge
design, applied to self-critique instead of output evaluation. Where
stakes don't justify the extra cost/complexity of a second model,
at minimum make the reflection prompt structurally adversarial (asking
it to argue *against* the last action's success, not asking a neutral
"did this work?") — a critique prompt that defaults to affirmation will
tend to affirm.

### 6.4 When reflection is worth it — the cost/benefit question

Reflection roughly doubles the LLM-call count per iteration (one for
`think`, one for `reflect`). That is not free, and "reflect after every
single step, always" is not the correct default any more than "cache
everything" was the correct default in Part 3, Module 8. The right
question, per iteration: **how likely is this specific action to fail
silently (return without error, but not actually serve the sub-goal),
and how expensive is it to discover that three steps later instead of
now?**

Concretely:

- High-value reflection targets: retrieval steps (did the retrieved
  content actually address the sub-question, or just look
  superficially related — this is precisely Part 4, Module 8's
  faithfulness-vs-relevance distinction, now applied mid-agent-run
  instead of at final-answer time), and any step whose output feeds a
  branching decision later in the plan.
- Low-value reflection targets: mechanical, low-ambiguity steps where
  success/failure is already unambiguous from the tool's own return
  value (e.g., a well-typed API call that either errors or returns
  exactly the requested field) — reflecting on these adds cost without
  reducing meaningful uncertainty.

This is a judgment call to make explicitly per action type when
designing an agent's action space, not a blanket policy — and it should
be *measured*, not assumed, exactly as Part 3, Module 9 insisted latency
and cost tradeoffs be measured per-tier rather than only in aggregate.

### 6.5 Reflection data as the input to a real eval, not just a runtime signal

Every reflection critique, even confidence-only ones that don't alter
the plan, should be logged as part of the agent's trace. This gives you
a per-step, human-and-model-labeled signal for exactly the kind of
trace-level evaluation Part 3, Module 5 and Part 4, Module 8 built —
except now applied to *agent* traces instead of single-turn or
RAG-retrieval traces. A reflection log that says "step 4's retrieval was
flagged low-confidence, and the human reviewing the final output
independently found step 4's output was in fact wrong" is a validated
signal that your reflection mechanism has real predictive value, not
just theater — and is the dataset you'll use in this module's
Production Project to actually measure whether reflection helps.

---

## 7. Mental Models

1. **"Reflection checks whether the last step served its sub-goal — not
   whether it ran without error."**
2. **"A model critiquing its own work shares its own blind spots; make
   the critique adversarial, or use a second model, where it matters."**
3. **"Confidence-only reflection is cheap insurance; plan-altering
   reflection is where the real value — and the real cost — is."**
4. **"Reflect where silent failure is likely and expensive to discover
   later; don't reflect on steps that already fail loudly."**

---

## 8. Visual Explanation

```
              (continuing agent-core's loop, Module 1)
                              │
                    ┌─────────▼─────────┐
                    │   execute(action)    │
                    └─────────┬─────────┘
                              │ observation
                    ┌─────────▼─────────┐
                    │   reflect(state,      │  <- NEW: separate LLM
                    │   sub_goal,           │     call, not folded
                    │   observation)        │     into think()
                    │                       │
                    │   -> critique:        │
                    │      satisfied? bool  │
                    │      reasoning: str   │
                    │      confidence: float│
                    └─────────┬─────────┘
                              │
                 satisfied? ──┼── not satisfied?
                    │                    │
          confidence-only        plan-altering:
          (log, continue)        feed critique into
                    │             think()'s next call
                    │             explicitly
                    └──────────┬─────────┘
                              ▼
                    ┌─────────────────────┐
                    │   update(state, ...)   │
                    │   (append observation  │
                    │   AND critique to      │
                    │   scratch-pad/trace)   │
                    └─────────────────────┘
                              │
                        loop continues
```

---

## 9. Recommended Resources

1. **Shinn et al., "Reflexion: Language Agents with Verbal
   Reinforcement Learning" (2023 paper)** — the paper most directly
   behind this module's core idea; read it now to compare its framing
   of self-critique against your own first-principles version, and note
   where it does or doesn't address Section 6.3's false-confidence risk.
2. **Anthropic — "Building Effective Agents"** (revisit) — reread the
   section on evaluator/optimizer patterns, which is this module's
   plan-altering reflection under a different name.
3. **Your own Part 2, Module 8 (`judge-bias-lab`)** — the most relevant
   resource is your own prior work on LLM-judge bias; reflection is a
   judge that happens to be judging an agent's own intermediate step
   instead of a separate output, and the same mitigation techniques
   apply.
4. **Your own Part 3, Module 10 verification-pass code** — reuse the
   mechanical pattern directly rather than reinventing a critique
   mechanism from scratch.

---

## 10. Exercises

1. For three different action types in your Module 1 action space
   (e.g., retrieve, compute, send-notification), decide which need
   reflection and which don't, using Section 6.4's criterion. Justify
   each decision in one sentence.
2. Implement `reflect()` as a genuinely separate LLM call (separate
   prompt, ideally separate model or an adversarially-framed prompt per
   Section 6.3) and confirm via logging that it is not silently reusing
   `think()`'s call or context.
3. Deliberately inject a subtly wrong observation (e.g., a retrieval
   result that's topically related but doesn't actually answer the
   sub-question) into a test run. Confirm your reflection step flags it
   as unsatisfied, and confirm a *neutrally-framed* reflection prompt is
   more likely to miss it than an adversarially-framed one — measure
   this, don't assume it.
4. Log reflection critiques across 10 real agent runs. Compare
   reflection's "satisfied: true/false" against an independent human
   judgment of whether each step actually succeeded. Compute a rough
   agreement rate — this is your first evidence of whether reflection
   has real predictive value in your system.
5. Estimate the added cost (extra LLM calls, latency) of adding
   reflection to every step of a 10-step agent run, versus only
   reflecting on the high-value targets identified in Exercise 1. Is the
   selective policy worth the added design complexity?

---

## 11. Mini-Project

**`reflection-agreement-eval`**: using the logged data from Exercise 4,
build a small script that computes agreement between reflection's
self-assessment and an independent (human or separate-model) assessment
of the same step, across enough runs to be a meaningful sample. Report
not just an aggregate agreement rate but a breakdown by action type —
this is where you'll discover, empirically, whether your Section 6.4
judgment calls about which actions need reflection were correct.

---

## 12. Production Project: `agent-core` v1.1 (Reflection extension)

### Scope

Extend `agent-core` (Part 5, Module 1) with:

- A `reflect()` call inserted between `execute()` and `update()` in the
  existing loop, implemented as a genuinely separate LLM call from
  `think()` — reusing `llm-client-core` for the call itself, not new
  LLM-calling infrastructure.
- A per-action-type `reflection_policy` field (`none` /
  `confidence_only` / `plan_altering`), set explicitly by the developer
  configuring each action in the action space — mirroring the explicit,
  no-silent-default discipline Module 1 established for
  `requires_approval`.
- A configurable **reflection model**, allowed to differ from the
  planning model, with an adversarially-framed default critique prompt
  ("argue against this step's success; find the flaw") per Section 6.3.
- Full reflection logging into the same trace format `agent-core`
  already writes, extending it with `critique`, `satisfied`, and
  `confidence` fields per scratch-pad entry — not a separate logging
  system.
- The `reflection-agreement-eval` from the Mini-Project, run as a
  standing eval (not one-off), so that future modules extending
  `agent-core` further can re-run it and detect regressions in
  reflection quality.
- Regression-tested against `loop-stress-test` (Module 1) to confirm
  reflection doesn't break any of the three termination guards — in
  particular, confirm the cost budget guard correctly accounts for
  `reflect()`'s extra LLM calls, not just `think()`'s.

### Explicit extension point

**Part 5's later modules on memory and multi-agent systems** will
consume `agent-core` v1.1's critique/confidence trace data directly:
low-confidence steps become the natural trigger for a memory-write
decision ("this was uncertain, worth persisting the resolution for next
time") and, in multi-agent settings, the natural trigger for escalating
a sub-task to a different, more specialized agent rather than retrying
blindly.

---

## 13. Practice & Interview Questions

1. Why is reflection implemented as a separate LLM call rather than
   folded into the planning call that decides the next action?
2. What specific new failure mode does adding self-critique to an agent
   introduce, and what's the concrete mitigation?
3. Give an example of an action type where reflection is high-value and
   one where it's low-value, and state the criterion you used to decide.
4. How is reflection, as defined in this module, a special case of the
   "right answer, wrong reason" lesson from evaluating LLM pipelines?
5. How would you measure, empirically rather than by assumption, whether
   your reflection mechanism actually has predictive value in a given
   system?
6. Your agent's reflection step reports high confidence on a step that
   an independent human review later found to be wrong. What do you
   check first?

---

## 14. Common Mistakes

- **Folding reflection into the planning prompt** — a call juggling
  "critique the past" and "plan the future" tends to under-critique in
  favor of forward progress.
- **Neutral, non-adversarial critique prompts** — "does this look okay?"
  invites affirmation; "find the flaw" does not, and the difference is
  measurable, not just stylistic.
- **Reflecting on every action uniformly** — wastes cost on
  low-ambiguity steps where failure is already unambiguous from the
  tool's return value, while providing no more signal than the tool
  output already gave.
- **Trusting reflection without measuring it against ground truth** —
  shipping self-critique and assuming it's catching real errors, without
  ever running the equivalent of `reflection-agreement-eval` to check.
- **Letting the cost budget guard ignore reflection's extra calls** —
  a common regression when extending Module 1's loop: the budget check
  written before reflection existed may not have been updated to count
  `reflect()` calls, silently doubling real cost per iteration relative
  to what the budget assumes.

---

## 15. Debugging Exercise

Your agent's reflection step has a 95% "satisfied: true" rate across
recent runs, but user-reported task failures haven't decreased since you
added reflection. Using Section 6.3 and the Mini-Project's agreement
eval, walk through: (a) the most likely explanation for a
high-self-reported-satisfaction rate that doesn't correlate with actual
task success, (b) the specific experiment (from Exercise 3/Mini-Project)
you'd run to confirm it, and (c) the specific mitigation from Section
6.3 you'd apply, and why simply "reflecting more often" would not fix
this particular problem.

---

## 16. Checklist

- [ ] `reflect()` is implemented as a genuinely separate LLM call from
      `think()`, with its own prompt.
- [ ] Every action type in my action space has an explicit
      `reflection_policy`, with no silent default.
- [ ] My default reflection prompt is adversarially framed, not
      neutral, and I've measured (not assumed) that this catches more
      real errors than a neutral framing.
- [ ] Reflection critiques are logged into the same trace format as
      the rest of the agent's execution, with `satisfied`/`confidence`
      fields.
- [ ] `reflection-agreement-eval` has actually been run against real or
      simulated data, with a reported agreement rate broken down by
      action type.
- [ ] My cost-budget termination guard correctly accounts for
      `reflect()`'s LLM calls, verified by a test, not assumed.
- [ ] `loop-stress-test` from Module 1 still passes after this
      extension.

---

## 17. Summary

Reflection adds one disciplined thing to the agent loop: a separate,
adversarially-framed check of whether the last action's observation
actually served its sub-goal, logged alongside the observation itself.
It is Part 3, Module 5's "per-stage pass rates don't predict pipeline
correctness" lesson applied at every intermediate step of a multi-step
agent run, using the same mechanical pattern as Part 3, Module 10's
verification pass and Part 2, Module 8's bias-aware judge design. It is
not free — it roughly doubles per-iteration LLM calls — and its value
must be measured per action type, not assumed uniformly, or you risk
shipping self-critique that's really just self-congratulation.

---

## 18. Next Steps

Next: **Part 5, Module 3 — Agent Memory & Long-Horizon Tasks**, which
extends `agent-core`'s scratch-pad (short-lived, single-run state) with
the long-term memory read/write/consolidate mechanism from Part 3,
Module 4, so an agent can carry resolved uncertainty and reusable
context across separate runs — with low-confidence reflection outcomes
from this module as one of the concrete triggers for what gets written
to long-term memory.

Reply "continue" for Module 3, or flag anything to go deeper on.
