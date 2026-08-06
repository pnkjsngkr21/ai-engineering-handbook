# Part 3, Module 11: Multi-Turn Conversation Design

> Every prior Part 3 module built a component: prompting, structured
> output, tool calling, memory, evaluation, guardrails, context
> engineering, caching, latency/cost optimization, hallucination
> reduction. This module is where those components stop being separate
> pieces and become one coherent conversational experience across many
> turns — clarification-seeking, error recovery, session boundaries, and
> the turn-taking design decisions that determine whether a system feels
> reliable or frustrating to actually talk to. It is the direct bridge
> into Part 3's capstone.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain why multi-turn conversation design is a genuinely distinct
   engineering concern from any single component built so far — it's
   about the *policy* governing how those components interact turn over
   turn, not a new mechanism of its own.
2. Design clarification-seeking behavior: when a system should ask a
   follow-up question versus proceed on a reasonable assumption, and how
   to make that decision reliable rather than inconsistent.
3. Design conversation-repair patterns: how a system should recover
   gracefully when it has misunderstood the user, produced a wrong tool
   call, or been corrected — using the guardrail, evaluation, and
   hallucination-reduction infrastructure already built rather than
   inventing new error-handling from scratch.
4. Reason correctly about session/conversation boundaries — when a
   "conversation" should be considered ended, how that interacts with
   Module 4's short-term compaction and long-term memory write triggers,
   and why getting this boundary wrong silently corrupts memory quality.
5. Apply turn-taking design patterns for tool-calling-heavy
   conversations (Module 3) so that multi-step tool sequences feel
   coherent to a user rather than confusing or opaque.
6. Design and run a genuine multi-turn user-experience evaluation — not
   just per-message correctness (Module 5) but conversation-level
   coherence, appropriateness of clarification behavior, and recovery
   quality — as the direct precursor to Part 3's capstone acceptance
   criteria.

---

## 2. Prerequisites

- **Part 3, Module 5** (Evaluating Full LLM-Powered Pipelines) — this
  module's conversation-level evaluation extends trace-level evaluation
  to span multiple turns rather than one call or one pipeline execution.
- **Part 3, Module 4** (Memory) — session-boundary decisions directly
  determine when compaction and long-term-memory-write triggers fire.
- **Part 3, Module 3** (Tool Calling) — turn-taking design for
  tool-heavy conversations builds directly on the multi-turn loop
  already implemented there.
- **Part 3, Module 6** (Guardrails) and **Module 10** (Hallucination
  Reduction) — conversation-repair patterns reuse both, rather than
  introducing new error-handling infrastructure.

---

## 3. Estimated Study Time

**6–8 hours** (2 hours theory/reading, 4–6 hours hands-on).

---

## 4. Difficulty

★★★☆☆ (3/5)

Mechanically, this module is mostly policy and integration work across
components you've already built — the difficulty is in the design
judgment (when to clarify vs. assume, where a session boundary actually
belongs) rather than in any new, hard mechanism.

---

## 5. Why This Matters

A system with excellent individual components — reliable structured
output, correct tool calling, good memory, low hallucination rate — can
still feel unpleasant or untrustworthy to use if the conversation-level
policy is wrong: an assistant that asks unnecessary clarifying questions
for every trivial request feels tedious; one that never asks and
confidently proceeds on wrong assumptions feels careless; one that can't
gracefully recover when corrected feels brittle. This is exactly the
layer where "technically correct components" and "actually good
product" diverge, and it's the layer most engineers underinvest in
because it doesn't map to a single clean technical mechanism the way
tool calling or caching do — which is precisely why it deserves its own
module rather than being treated as an afterthought before the capstone.

---

## 6. Theory

### 6.1 Why this is a policy layer, not a new mechanism

Every technical capability needed for good multi-turn conversation
design already exists in `llm-client-core` from prior modules:
`PromptTemplate` (Module 1) can render a clarifying question exactly like
any other response; `ConversationManager` (Module 4) already manages
state across turns; guardrails (Module 6) and hallucination-reduction
checks (Module 10) already run per-response. **What's missing is the
policy that decides, turn by turn, which behavior is appropriate given
the conversation's actual state** — and that policy is this module's
entire subject matter. This is worth stating explicitly because it's
tempting to treat "conversation design" as requiring some new
mechanism; it doesn't. It requires disciplined decision rules applied
consistently across the components you've already built.

### 6.2 Clarification-seeking: when to ask, when to assume

**The core tension**: asking a clarifying question every time there's
any ambiguity produces a tedious, over-cautious experience; never asking
and always proceeding on an assumption produces a system that
confidently acts on wrong guesses — directly compounding Module 3's
tool-calling risk (a wrong assumption feeding a real tool-call argument)
and Module 10's hallucination risk (a wrong assumption stated with
unwarranted confidence).

**A workable decision rule**, connecting directly to concepts already
built:

- **If the ambiguity affects only response *style* or minor framing**
  (and doesn't affect a tool-call argument, a stored-memory write, or a
  factual claim's correctness) — proceed on a reasonable default
  assumption, and state the assumption briefly rather than silently
  guessing (a direct application of the proactivity principle this
  handbook itself has followed throughout: state assumptions, don't ask
  when you can reasonably proceed).
- **If the ambiguity affects a consequential tool-call argument or a
  memory write** (Module 3's least-privilege framing: an action with
  real, possibly hard-to-reverse effects) — clarify before proceeding,
  because the cost of a wrong assumption here (Module 3, Section 6.5;
  Module 4's confabulation risk) is categorically higher than the cost
  of one extra clarifying turn.
- **If the ambiguity is resolvable from already-available context**
  (retrieved memory, Module 4; earlier conversation turns) — resolve it
  silently using that context rather than asking the user to repeat
  information the system should already have; asking anyway feels
  exactly like a memory-system failure to the user, even if it's
  technically a conversation-design choice, and should be evaluated
  with the same severity as a genuine memory-retrieval bug (Module 4).

**Make this a schema-constrained decision, not a soft prompt
instruction** (Module 2's discipline, applied here): a response schema
with an explicit `requires_clarification: bool` and
`clarification_question: Optional[str]` field forces a checkable
decision point rather than relying on the model's free-text response to
reliably signal whether it's asking a genuine clarifying question or
just hedging within a normal answer.

### 6.3 Conversation repair: recovering from misunderstanding gracefully

Even with good clarification policy, the system will sometimes act on a
wrong assumption, call the wrong tool, or otherwise misunderstand the
user — and how it responds when corrected matters as much as how often
it's initially wrong. **Repair is not a new mechanism**; it's the
disciplined reuse of infrastructure already built:

- **When a user correction arrives**, it should be treated, mechanically,
  as new information entering the conversation state (Module 1, Section
  6.1) — nothing exotic, just a new turn — but the *policy* response
  matters: acknowledge the correction explicitly rather than silently
  changing course with no reference to the prior error, since silent
  correction reads as inconsistency rather than responsiveness.
- **If the misunderstanding already resulted in a tool call** (Module 3),
  the repair policy must account for whether that tool call had a real
  side effect — a read-only tool call (Module 3's least-privilege
  design) can simply be redone correctly; a consequential one (Module
  3, Section 6.5) may require an explicit compensating action or
  escalation, not just a corrected next attempt, which is exactly why
  Module 3's least-privilege tool design pays off again here: narrow,
  read-only tools are cheap to recover from; broad, consequential ones
  are not.
- **If the correction contradicts a stored long-term memory** (Module
  4), it should trigger that module's consolidation logic (superseding
  the stale memory), not just correct the current turn's behavior while
  leaving the underlying stale memory untouched for the next session —
  this is a direct, easy-to-miss integration point between conversation
  repair and Module 4's write/consolidation policies.

### 6.4 Session boundaries: where memory and compaction policy actually gets triggered

Module 4 built compaction (short-term) and write/consolidation
(long-term) mechanisms, but largely deferred the question of *when*
those triggers actually fire in relation to a real, bounded
"conversation." This module confronts that directly, because getting it
wrong has real, silent consequences: **triggering long-term memory
writes too eagerly** (e.g., after every single turn) risks writing
premature, provisional, or since-superseded information as if it were a
finalized fact; **triggering it too late or not at all** (e.g., only at
an explicit "goodbye," which many real conversations never say) risks
losing durable information that should have been captured.

**A workable session-boundary policy**: define explicit signals for
"this conversational session has meaningfully concluded or paused" —
an explicit close (the user says goodbye or the client disconnects), an
inactivity timeout (no new turn within a defined window), or a topic
shift the system itself detects (the user's next message is
unrelated to the ongoing thread) — and trigger long-term memory
write/consolidation (Module 4) at those points rather than continuously
or arbitrarily. This is a genuine design decision requiring its own
evaluation (Section 6.6), not a default you can skip.

### 6.5 Turn-taking design for tool-heavy conversations

Module 3's tool-calling loop can run several rounds silently before
producing a final text response — from the user's perspective, this can
feel like an unexplained pause (if not streamed, per Module 9) or a
confusing "how did it know to do that" experience if the tool-call
sequence isn't surfaced at all. **Conversation design decisions here**:
whether and how to surface intermediate tool-use activity to the user
(a brief "checking your order status..." type indicator, reusing Module
9's streaming infrastructure for perceived responsiveness during a
multi-round tool sequence), and how to handle a tool-calling round that
fails or times out — surfacing a clear, honest statement of what
happened rather than either silently retrying indefinitely (Module 3's
max-turns bound already prevents infinite loops, but the *user-facing*
communication of a bounded failure is a conversation-design decision,
not just a backend safety bound) or presenting a generic, uninformative
error.

### 6.6 Evaluating conversation-level quality, not just per-message correctness

Module 5's `TraceScorer` evaluates a single pipeline execution (one
user turn through prompt → tools → memory → generation). **Conversation-
level evaluation is a further extension**: golden test cases now span
*multiple* turns, checking whether the *sequence* of behaviors is
appropriate — did the system correctly decide to clarify at the right
point (Section 6.2) and not at trivial points? did it correctly recover
when corrected (Section 6.3), including correctly triggering memory
consolidation if applicable? did session-boundary-triggered memory
writes (Section 6.4) actually fire at the right point and capture the
right information?

This requires genuinely multi-turn golden datasets (a full scripted
conversation, not a single input/output pair), scored with the same
rubric-based, schema-constrained judge discipline Module 5 and Module 6
already established — evaluating "did this conversation, across all its
turns, behave appropriately" rather than "was this one response
correct," which is a distinct and necessary evaluation surface this
module adds to `eval-framework`.

---

## 7. Mental Models

1. **"Conversation design is policy over existing mechanisms, not a new
   mechanism."** Every capability needed already exists in
   `llm-client-core`; this module is entirely about the decision rules
   governing when each capability is invoked.
2. **"Clarify when the cost of a wrong assumption is high; assume and
   state it when the cost is low."** Tie the clarify/assume decision to
   the actual downstream consequence (a tool-call argument, a memory
   write) rather than a blanket policy in either direction.
3. **"Repair is old infrastructure, new discipline."** Recovering from a
   misunderstanding reuses tool-calling, memory consolidation, and
   guardrail machinery already built — the new part is applying it
   consistently and acknowledging the correction explicitly.
4. **"A session boundary is a real design decision with real memory
   consequences, not a formality."** Too-eager or too-late memory
   triggers silently corrupt long-term memory quality either direction.

---

## 8. Visual Explanation

**Diagram 1 — The clarify/assume decision, tied to downstream stakes**

```
Ambiguity detected in current turn
             │
             ▼
   Does resolving it wrong affect:
   - a tool-call argument (Module 3)?
   - a long-term memory write (Module 4)?
   - a factual claim's correctness (Module 10)?
             │
      ┌──────┴──────┐
     YES            NO
      │              │
  CLARIFY        Is it resolvable from
  (ask, don't    already-available context
   guess)        (memory, earlier turns)?
                       │
               ┌───────┴───────┐
              YES              NO
               │                │
        RESOLVE SILENTLY   ASSUME + STATE
        (don't re-ask       the assumption
         what's already      briefly, proceed
         known)
```

**Diagram 2 — Session boundary triggers and memory consequences**

```
Ongoing conversation ──► [explicit close]      ──► trigger Module 4
                     ──► [inactivity timeout]  ──► write/consolidation
                     ──► [detected topic shift]──► NOW, not before,
                                                     not never

Trigger TOO EARLY (every turn):  writes provisional/unfinalized info
Trigger TOO LATE (never/rare):   loses durable info that should persist
Trigger AT THE RIGHT POINT:      captures finalized, durable information
                                  once, at the right granularity
```

**Diagram 3 — Multi-turn evaluation vs. single-turn evaluation**

```
SINGLE-TURN (Module 5's TraceScorer):
  [one input] ──► [one pipeline execution] ──► score THIS response

MULTI-TURN (this module's extension):
  [turn 1] ──► [turn 2: ambiguous] ──► [turn 3: correction] ──► [close]
     │              │                        │                    │
   score:        score: did it          score: did it         score: did
   normal        correctly              acknowledge the       memory write
   response      CLARIFY here,          correction AND        trigger here,
                 not assume?            trigger consolidation  correctly?
                                        if memory-relevant?

              ──────────► one score for the WHOLE sequence ◄──────────
```

---

## 9. Recommended Resources

1. **Anthropic — guidance on building conversational assistants /
   multi-turn dialogue design** (docs.claude.com, anthropic.com) — read
   for the vendor's own framing of conversation-level design
   considerations, since this is closer to product design guidance than
   a pure API mechanism, and vendor documentation often captures this
   most concretely.
2. **Classic conversational-UX/dialogue-systems literature (pre-LLM)**
   (search for foundational work on clarification-seeking dialogue
   systems and mixed-initiative interaction) — read for the underlying
   HCI/dialogue-design principles, which predate LLMs but transfer
   directly to Section 6.2's clarify/assume framework, since the
   underlying UX tradeoff is not new even though the generation
   mechanism is.
3. **Your own `eval-framework`/`TraceScorer` codebase (Part 2 Module 8,
   Part 3 Module 5)** — the primary technical resource for this module
   is extending your own multi-turn evaluation capability, not new
   external material.

---

## 10. Exercises

1. **Design the clarify/assume decision table for your own system.**
   For 8-10 realistic ambiguous requests in your domain, classify each
   using Section 6.2's decision rule (affects a tool-call argument or
   memory write? resolvable from existing context? otherwise low-stakes?)
   and write the expected system behavior for each.
2. **Script and test a repair scenario.** Construct a 4-5 turn scripted
   conversation where the system reasonably misunderstands an ambiguous
   early request, is corrected by the user, and (if the misunderstanding
   involved a tool call) must recover. Run it against your actual
   pipeline and evaluate whether the repair behavior (acknowledgment,
   correct recovery, memory consolidation if relevant) is appropriate.
3. **Test session-boundary triggering under all three signals.**
   Construct three separate test conversations, each ending via a
   different boundary signal (explicit close, simulated inactivity
   timeout, detected topic shift). Confirm long-term memory
   write/consolidation (Module 4) fires correctly at each, and doesn't
   fire prematurely mid-conversation.
4. **Design tool-heavy turn-taking feedback.** For a 2-3 round
   tool-calling sequence (Module 3), design what, if anything, gets
   surfaced to the user during the intermediate rounds (reusing
   streaming from Module 9), and justify the choice against the
   alternative of surfacing nothing until the final response.
5. **Build one multi-turn golden test case** per Section 6.6's format,
   specifying expected behavior across the full sequence (not just the
   final turn), and confirm your evaluation extension correctly scores
   both a "did everything right" run and a deliberately-broken run (e.g.,
   one where you disable clarification) as pass/fail respectively.

---

## 11. Mini-Project: `conversation-repair-sim`

A small standalone tool that scripts several multi-turn conversation
scenarios (a clean clarify-then-proceed scenario, a misunderstanding-
then-correction scenario, and a session-boundary scenario) against your
actual pipeline, logging the full turn-by-turn trace and flagging
whether each scenario's expected behavior (per Sections 6.2-6.4) was
observed — the direct precursor to formalizing multi-turn golden
datasets in the production project.

---

## 12. Production Project: Conversation-Level Policy Layer + Multi-Turn Evaluation Extension

### 12.1 What you're building

1. **A schema-constrained clarification decision** integrated into
   `ConversationManager` (Module 4) and `PromptTemplate` (Module 1): an
   explicit `requires_clarification`/`clarification_question` output
   field (Section 6.2), with the decision rule (tool-call/memory-write
   stakes vs. resolvable-from-context vs. low-stakes) implemented as
   real logic, not left to unconstrained model judgment alone.

2. **A conversation-repair policy** that explicitly handles user
   corrections: detecting a correction (via a targeted classification
   step, reusing Module 2's schema-constrained approach), acknowledging
   it in the response, and triggering the appropriate downstream action —
   redoing a read-only tool call, escalating for a consequential one
   (Module 3), or triggering memory consolidation (Module 4) — based on
   what was actually affected by the original misunderstanding.

3. **Explicit session-boundary detection and triggering**: implement all
   three boundary signals from Section 6.4 (explicit close, inactivity
   timeout, detected topic shift via a lightweight classification check),
   each correctly triggering `LongTermMemoryStore`'s write/consolidation
   at the right point, verified by tests covering all three signals
   independently.

4. **Tool-heavy turn-taking UX**: surface appropriate intermediate
   status during multi-round tool-calling sequences (Module 3), reusing
   streaming infrastructure (Module 9), with a clear, honest user-facing
   message when a tool-calling round fails or hits its max-turns bound.

5. **Multi-turn golden datasets and evaluation**: extend
   `eval-framework`/`TraceScorer` (Module 5) to support and score
   multi-turn scripted conversations against expected behavior across
   the *entire* sequence (Section 6.6), including at least the three
   scenario types built in the `conversation-repair-sim` mini-project,
   now formalized as permanent regression tests.

### 12.2 What this sets up for later modules

- **Part 3's capstone** (the next module) directly assembles this
  conversation-level policy layer, alongside every other Part 3
  component, into the finished production conversational AI service —
  this module's multi-turn evaluation extension becomes the capstone's
  primary acceptance-test framework.
- **Part 5 (Agents)** will extend the repair and turn-taking patterns
  here to longer-horizon, more autonomous multi-step agent behavior.

### 12.3 Definition of done

- Clarification decisions are schema-constrained and correctly follow
  the stakes-based decision rule on your test set (Exercise 1).
- Conversation repair correctly acknowledges corrections and triggers
  the appropriate downstream action (tool redo, escalation, or memory
  consolidation) based on what the original misunderstanding affected.
- All three session-boundary signals correctly trigger long-term memory
  write/consolidation at the right point, verified by dedicated tests.
- Tool-heavy turn-taking surfaces appropriate intermediate status and
  honest failure messaging.
- Multi-turn golden datasets exist for all three
  `conversation-repair-sim` scenario types and are part of the permanent
  pipeline CI (Module 5) regression suite.

---

## 13. Practice & Interview Questions

1. Why is multi-turn conversation design considered a policy layer
   rather than requiring new underlying mechanisms, given everything
   already built in Modules 1-10?
2. Design the clarify-vs-assume decision rule for a system that manages
   a user's calendar, and justify at least one case where you'd clarify
   and one where you'd proceed on an assumption.
3. A user corrects a misunderstanding that had already resulted in a
   consequential tool call (e.g., an email was sent to the wrong
   recipient). Walk through what your system needs to do, beyond simply
   answering correctly on the next turn.
4. Why does triggering long-term memory writes after every single turn
   risk degrading memory quality, and what's a better session-boundary
   policy?
5. How would you evaluate whether a system's clarification behavior is
   well-calibrated (neither too eager nor too reluctant) using
   `eval-framework`, rather than by subjective impression alone?
6. Design a multi-turn golden test case that would fail if a system's
   session-boundary detection incorrectly triggered a memory write
   mid-conversation instead of at an appropriate boundary.

---

## 14. Common Mistakes

- **Treating clarification as a uniform policy (always ask, or never
  ask)** instead of tying the decision to the actual downstream stakes
  of getting it wrong.
- **Silent course-correction without acknowledgment** when the user
  corrects a misunderstanding — technically fixing the next response
  while reading as inconsistent or unresponsive to the user.
- **Triggering long-term memory writes on every turn**, capturing
  provisional or since-superseded information as if finalized.
- **Never triggering memory writes without an explicit "goodbye,"**
  losing durable information from conversations that simply trail off.
- **Leaving multi-round tool-calling sequences completely opaque to the
  user**, producing a confusing experience even when the underlying
  logic (Module 3) is entirely correct.
- **Evaluating only single-turn correctness** and never building
  genuinely multi-turn golden datasets, missing the entire class of
  conversation-level policy bugs this module addresses.

---

## 15. Debugging Exercise

Your conversation-repair policy has been working correctly in manual
testing, but `conversation-repair-sim`'s automated multi-turn golden
tests reveal that in one specific scenario — a correction that arrives
several turns after the original misunderstanding, rather than
immediately following it — the system fails to recognize it as a
correction at all and treats it as an unrelated new request.

Write down at least two concrete hypotheses for why correction-detection
might fail specifically when there's a gap between the misunderstanding
and the correction (consider: does your correction-detection
classification step, per Module 2's schema-constrained approach, only
look at the immediately preceding turn, missing corrections referencing
something further back? could Module 4's short-term compaction have
already summarized away the specific original misunderstanding by the
time the correction arrives, making it undetectable even in principle
from the compacted context?), and describe how you'd use
`trace-recorder` (Module 5) to confirm which is happening before
redesigning the correction-detection step.

---

## 16. Checklist

- [ ] I can explain why multi-turn conversation design is a policy layer
      over existing mechanisms rather than a new mechanism itself.
- [ ] I can design a clarify-vs-assume decision rule tied to downstream
      stakes (tool-call arguments, memory writes) rather than a blanket
      policy.
- [ ] I can describe conversation repair as disciplined reuse of
      tool-calling, memory, and guardrail infrastructure, not new
      error-handling.
- [ ] I understand why session-boundary triggering matters for memory
      quality and can name at least three concrete boundary signals.
- [ ] I can design appropriate turn-taking UX for a multi-round
      tool-calling sequence.
- [ ] `conversation-repair-sim` is built and covers clarify, repair, and
      session-boundary scenarios against my real pipeline.
- [ ] Schema-constrained clarification is integrated into
      `ConversationManager`/`PromptTemplate` and follows the stakes-based
      decision rule on my test set.
- [ ] Conversation repair correctly triggers the right downstream action
      based on what the original misunderstanding affected.
- [ ] All three session-boundary signals correctly and only trigger
      memory write/consolidation at the appropriate point.
- [ ] Multi-turn golden datasets for all three scenario types are part of
      permanent pipeline CI.

---

## 17. Summary

Multi-turn conversation design is where every component built across
Part 3 — prompting, structured output, tool calling, memory, evaluation,
guardrails, context engineering, caching, latency/cost optimization, and
hallucination reduction — gets governed by an explicit, evidence-tested
policy rather than left to per-turn improvisation. Clarification should
be tied to the actual downstream stakes of getting an ambiguity wrong,
not applied uniformly; conversation repair should acknowledge corrections
explicitly and reuse existing tool-calling, memory, and guardrail
infrastructure rather than inventing new error-handling; session
boundaries are a genuine design decision that determines when long-term
memory write/consolidation actually fires, with real quality
consequences if triggered too eagerly or too rarely; and tool-heavy
turn-taking needs its own honest, UX-aware treatment rather than
remaining opaque to the user. `llm-client-core` now has a schema-
constrained clarification policy, a working repair mechanism, explicit
session-boundary detection correctly wired into Module 4's memory
triggers, and a multi-turn evaluation extension to `eval-framework` that
scores entire conversation sequences rather than single turns — every
piece the capstone module needs to assemble Part 3's finished production
conversational AI service.

---

## 18. Next Steps

**Next: Part 3, Module 12 — Capstone: A Finished, Production-Grade
Conversational AI Service.** This closing module of Part 3 synthesizes
every component built across Modules 1-11 — real provider adapters,
structured output, tool calling with a live MCP connection, short- and
long-term memory, pipeline-level evaluation, layered guardrails, context
budget management, three-layer caching, streaming/batching/routing,
hallucination reduction, and conversation-level policy — into one
finished, tested, documented, deployable service, with acceptance
criteria drawn directly from this module's multi-turn evaluation
framework.

Reply "continue" for the Part 3 capstone module, or flag anything to go
deeper on.
