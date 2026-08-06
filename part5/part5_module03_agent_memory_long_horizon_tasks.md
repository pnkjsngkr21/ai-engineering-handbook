# Part 5, Module 3: Agent Memory & Long-Horizon Tasks

> Extends `agent-core`'s scratch-pad (short-lived, single-run state)
> with the long-term memory read/write/consolidate mechanism from Part
> 3, Module 4 — so an agent can carry resolved uncertainty and reusable
> context across separate runs, not just across iterations of one run.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Distinguish three distinct memory scopes an agent needs — scratch-pad
   (within one run), episodic (across runs, "what happened last time"),
   and semantic (across runs, "what's generally true") — and explain why
   collapsing them into one store causes specific, predictable bugs.
2. Explain why Part 5, Module 2's low-confidence reflection outcomes are
   the correct trigger for episodic memory writes, rather than writing
   everything or writing nothing.
3. Implement a memory-write policy that reuses Part 3, Module 4's
   write/retrieve/consolidate mechanism, extended with agent-specific
   triggers instead of conversation-turn-boundary triggers.
4. Identify and defend against **memory poisoning** — the agent-specific
   version of Part 1, Module 9's privilege-separation concern, where an
   agent's own faulty conclusion gets written to long-term memory and
   then treated as trusted fact in a future run.
5. Design a long-horizon task (one that spans multiple separate agent
   invocations, possibly with human interruption in between) and
   implement the state-resumption mechanism that makes it possible.
6. Extend `agent-core` into v1.2 with a memory layer, evaluated for both
   usefulness (does it actually help later runs) and poisoning
   resistance (does a bad conclusion get contained rather than
   compounding).

## 2. Prerequisites

- Part 5, Modules 1–2 (`agent-core` v1.1), completed.
- Part 3, Module 4 (Memory) — this module is a direct, structural
  extension of that write/retrieve/consolidate mechanism; re-read it
  before starting.
- Part 4's `rag-engine` — long-term agent memory is, mechanically, a
  retrieval problem, and this module reuses `rag-engine`'s retrieval
  layer rather than building a second one.

## 3. Estimated Study Time

10–12 hours across 2–3 sessions.

## 4. Difficulty

★★★★☆ (4/5) — The memory-scope distinctions are conceptually
straightforward, but the poisoning-resistance design (Section 6.4) is
genuinely hard to get right and easy to convince yourself you've solved
without actually testing the adversarial case.

---

## 5. Why This Matters

Part 3, Module 4 established that conversational memory has no shortcut
— it's all context engineering, and the model has no memory of its own.
The same is true for agents, but the stakes are different: a chat
session that forgets something asks the user again. A long-horizon
agent task — "monitor this system and take corrective action over the
next week," "work through this multi-day research project across
several separate invocations" — cannot re-derive everything from
scratch every time it's invoked, and cannot simply keep an ever-growing
scratch-pad, because scratch-pad state (per Part 3, Module 7) is bounded
by context and does not survive between separate invocations anyway.

This matters especially because of a failure mode that's specific to
agents and didn't really exist in Part 3's chat-memory setting: an agent
that writes its own conclusions to memory, and later treats those
conclusions as trusted ground truth, can manufacture false confidence
out of nothing — it derived a wrong answer once, wrote it down, and now
every future run "confirms" it because the wrong answer is sitting in
memory looking like an established fact. This is the sharpest version
yet of the privilege-separation problem from Part 1, Module 9: the
agent's own past output must not automatically inherit the trust level
of verified external information, or the system can poison itself with
no external attacker required at all.

---

## 6. Theory

### 6.1 Three memory scopes, kept structurally separate

- **Scratch-pad** (already built, Part 5 Module 1): the
  thought/action/observation/critique history of the *current* run.
  Lives only as long as the run does. This is what `agent-core`'s
  `state` already is.
- **Episodic memory**: a record of *specific past runs* — "on this
  date, given this task, the agent did X, Y, Z, and the outcome was
  W." Retrieved when a new run's task resembles a past one, to avoid
  repeating the same mistakes or re-deriving the same plan from
  scratch. This is the direct agent-specific analog of Part 3, Module
  4's long-term memory, and reuses the same write/retrieve/consolidate
  mechanism, just triggered by different events (Section 6.2).
- **Semantic memory**: generalized, resolved facts distilled *from*
  episodic memory over time — "the internal API for X always returns
  paginated results even when the docs don't say so" — the kind of
  durable, reusable knowledge that would otherwise be relearned the
  hard way in every single run. This is consolidation (Part 3, Module
  4's term) applied specifically to cross-run agent experience.

Collapsing these three into one undifferentiated store produces a
specific, predictable bug: scratch-pad-style verbose, run-specific
detail (which should decay and not persist) gets treated with the same
weight as hard-won, validated semantic facts (which should persist and
generalize), so retrieval at the start of a new run either drowns in
irrelevant specifics or fails to surface the one durable fact that
actually mattered. Keep them as separate stores with separate retention
and retrieval policies, exactly as Part 3, Module 4 kept short-term
compaction and long-term storage structurally distinct.

### 6.2 What triggers a write, for agents specifically

Part 3, Module 4 established session-boundary triggers for
conversational long-term memory. Agents need different, more specific
triggers, because a "session boundary" isn't always well-defined for a
long-horizon task:

- **Task completion** (successful or not) — always write an episodic
  record: task, plan, outcome, and (from Module 2) any steps flagged
  low-confidence by reflection.
- **Low-confidence reflection outcomes specifically** — this is the
  direct payoff of Module 2's critique logging: a step reflection
  flagged as uncertain, once resolved (by a human, by a later
  successful retry, or by explicit escalation), is exactly the kind of
  resolved uncertainty worth persisting, so a future run facing the
  same uncertainty doesn't have to re-derive the resolution from
  scratch. An episodic write here should carry the *resolution*, not
  just the original uncertain observation — writing the unresolved
  confusion itself back to memory would just be re-injecting noise.
- **Human correction** — if a human overrides or corrects an agent's
  action (via the approval gate from Module 1, or after-the-fact
  feedback), that correction is a strong, validated signal and should
  be written with a higher trust weight than the agent's own
  unsupervised conclusions (Section 6.4 explains why this weighting
  matters).

Notice what's *not* a trigger: routine, successful, low-ambiguity steps
that reflection didn't flag. Writing every single scratch-pad entry to
long-term memory reproduces Part 3, Module 4's "bare summarization
isn't extraction" mistake at agent scale — indiscriminate write volume
buries the few genuinely reusable facts under routine noise, and
retrieval quality degrades exactly the way Part 4's chunking-quality
arguments predict for any retrieval store.

### 6.3 Retrieval is `rag-engine`'s job, not a new mechanism

Once episodic and semantic memory exist as stores, *retrieving* the
relevant subset for a new run's task is structurally identical to Part
4's retrieval problem: given a query (the new task description),
retrieve the most relevant prior records. There is no reason to build a
second retrieval stack for this. `agent-core` v1.2's memory layer should
call `RAGEngine.retrieve()` against a memory-specific collection,
reusing chunking, hybrid search, re-ranking, and metadata filtering
exactly as built in Part 4 — the only new work is deciding what gets
written into that collection (Section 6.2) and how much trust to assign
what's retrieved (Section 6.4), not how it's retrieved.

### 6.4 Memory poisoning — the agent-specific privilege-separation problem

Here is the failure mode that makes this module hard, stated precisely:
if an agent's own conclusion — including a wrong one — is written to
episodic or semantic memory with the same trust level as a
human-verified correction or an externally-sourced fact, then a future
run's `think()` step retrieves that wrong conclusion as if it were
established ground truth, builds on it, possibly reaches the same wrong
conclusion again (now with more apparent corroboration, because it
"remembers" reaching it before), and writes it again. Nothing external
attacked the system; it poisoned itself through its own memory loop —
this is Part 1, Module 9's privilege-separation lesson (don't let
untrusted content acquire the trust level of verified information)
turned inward, against the agent's own unverified output.

**Mitigation, concretely — trust levels are a required field, not a
convention:**

Every memory record carries an explicit `provenance` field with one of
a small enumerated set of values: `human_verified`,
`externally_sourced` (e.g., retrieved from `rag-engine`'s document
corpus, already access-controlled and attributable per Part 4, Module
5), or `agent_derived` (the agent's own conclusion, unverified). At
retrieval time, `think()`'s prompt must present `agent_derived` records
with explicit uncertainty framing ("in a previous run, the agent
concluded X, but this was not independently verified") — never as flat
fact. And critically: an `agent_derived` record should never, by
itself, be promoted to semantic memory (the durable, generalized store)
without at least one corroborating `human_verified` or
`externally_sourced` record backing it — this is the structural
circuit-breaker that stops a wrong conclusion from compounding across
runs purely by being repeated.

### 6.5 Long-horizon tasks: resumption, not re-derivation

A long-horizon task — one that legitimately spans multiple separate
invocations, possibly with a human closing their laptop in between —
needs a **resumable task state**, distinct from both scratch-pad
(too ephemeral, discarded at run end) and episodic memory (a record of
*past* runs, not the *current*, still-in-progress one). This is a
fourth, deliberately narrow store: a serialized snapshot of exactly
where the task's plan stands (per Section 6's plan-then-execute vs.
interleaved distinction — whichever was in use), which steps are done,
which are pending, and any `requires_approval` actions still awaiting a
human response. Resuming a run means loading this snapshot into a fresh
scratch-pad and continuing the loop — not re-planning from the original
task description as if starting over, and not treating the snapshot as
retrievable "memory" mixed in with episodic records from unrelated past
tasks.

---

## 7. Mental Models

1. **"Scratch-pad forgets on purpose; episodic memory remembers
   specific pasts; semantic memory keeps only what generalized."**
2. **"Write what reflection resolved, not what reflection merely
   flagged — resolution is the reusable part."**
3. **"An agent's own unverified conclusion is untrusted input to its
   future self, exactly like an untrusted document is untrusted input
   to itself right now."**
4. **"Resuming a long-horizon task is loading a snapshot, not
   re-deriving the plan from the original prompt."**

---

## 8. Visual Explanation

```
                    single agent run
      ┌───────────────────────────────────────┐
      │           scratch-pad (Module 1)         │
      │    thought/action/observation/critique   │
      │    (Module 2) — lives only this run       │
      └───────────────┬───────────────────────┘
                       │  write triggers (6.2):
                       │  task completion,
                       │  resolved low-confidence
                       │  reflection, human correction
                       ▼
      ┌───────────────────────────────────────┐
      │        episodic memory (per-run record)  │
      │   provenance: agent_derived /            │
      │   human_verified / externally_sourced    │
      └───────────────┬───────────────────────┘
                       │  consolidation, ONLY when
                       │  corroborated by a non-
                       │  agent_derived record (6.4)
                       ▼
      ┌───────────────────────────────────────┐
      │        semantic memory (durable,          │
      │        generalized, cross-task facts)     │
      └───────────────┬───────────────────────┘
                       │
                       │  retrieval at start of a NEW
                       │  run, via rag-engine (6.3),
                       │  provenance-labeled in prompt
                       ▼
              new run's think() call

      separate, narrow store — NOT mixed into the above:
      ┌───────────────────────────────────────┐
      │     resumable task-state snapshot        │
      │  (long-horizon tasks only — Section 6.5) │
      │  loaded directly into a fresh            │
      │  scratch-pad on resumption               │
      └───────────────────────────────────────┘
```

---

## 9. Recommended Resources

1. **Park et al., "Generative Agents: Interactive Simulacra of Human
   Behavior" (2023 paper)** — the most-cited treatment of the
   episodic-memory/reflection/retrieval loop for agents; read critically
   for how (or whether) it addresses provenance and poisoning, since
   that's this module's central concern and not the paper's main focus.
2. **Your own Part 3, Module 4 code** — the write/retrieve/consolidate
   mechanism is being reused, not reinvented; re-read your own
   implementation before writing the agent-specific triggers.
3. **Your own Part 1, Module 9 and Part 3, Module 6 material** — the
   provenance/trust-level framing in Section 6.4 is a direct
   application of the privilege-separation principle you already
   established; treat this module's memory-poisoning defense as that
   principle's most concrete test case yet.
4. **Anthropic — "Building Effective Agents"** (revisit again) — the
   sections on state and context management for longer-running agent
   workflows are directly relevant to Section 6.5's resumption design.

---

## 10. Exercises

1. Design the schema for an episodic memory record, including the
   `provenance` field. Write two example records — one
   `agent_derived`, one `human_verified` — for the same underlying
   fact, and write the exact prompt text `think()` would use to present
   each differently.
2. Simulate three consecutive runs of the same recurring task, where
   run 1 produces a wrong `agent_derived` conclusion. Trace whether
   your consolidation rule (Section 6.4) correctly prevents that
   conclusion from being promoted to semantic memory after runs 2 and 3
   simply repeat it, absent any external corroboration.
3. Implement the resumable task-state snapshot for a long-horizon task
   with at least one `requires_approval` action pending. Kill the
   process, restart it, and confirm the task resumes from the snapshot
   rather than re-planning from scratch.
4. Using `rag-engine`'s existing retrieval, retrieve semantic memory
   for a new task and confirm the retrieved records are correctly
   labeled with provenance in the assembled prompt — not silently
   flattened into undifferentiated context.
5. Decide, for your own action space from Module 1, exactly which
   write triggers from Section 6.2 apply to which action types, and
   justify any action type you deliberately exclude from ever writing
   to memory.

---

## 11. Mini-Project

**`poisoning-drill`**: deliberately seed episodic memory with a wrong
`agent_derived` conclusion, run the agent on the same or a similar task
three more times without any human correction, and confirm — by
inspecting semantic memory afterward — that the wrong conclusion was
never promoted past episodic scope. This is the empirical proof that
Section 6.4's consolidation rule actually holds under the specific
adversarial condition it was designed for, rather than something you
just implemented and assumed works.

---

## 12. Production Project: `agent-core` v1.2 (Memory extension)

### Scope

Extend `agent-core` (v1.1, with reflection) with:

- Episodic and semantic memory as two structurally distinct
  collections, each with its own write policy per Section 6.2 and its
  own retrieval call through `RAGEngine.retrieve()` (Part 4) — no new
  retrieval infrastructure.
- A required `provenance` field
  (`human_verified` / `externally_sourced` / `agent_derived`) on every
  memory record, enforced at the type level exactly as
  `requires_approval` (Module 1) and `reflection_policy` (Module 2)
  were — no silent defaults.
- A consolidation function that only promotes an `agent_derived` record
  to semantic memory when corroborated by at least one
  non-`agent_derived` record, implementing Section 6.4's circuit
  breaker structurally, not as a documented convention.
- A resumable task-state snapshot mechanism (Section 6.5), separate from
  both memory stores, used specifically for long-horizon tasks spanning
  multiple invocations.
- `poisoning-drill` (Mini-Project) as a standing regression test, run
  alongside `loop-stress-test` (Module 1) and
  `reflection-agreement-eval` (Module 2), so this module's guarantee is
  continuously verified, not proven once and forgotten.

### Explicit extension point

**Part 5's multi-agent modules** will give each agent in a multi-agent
system its own episodic memory scope, while sharing a common,
more conservatively-gated semantic memory store across agents — this
module's provenance/consolidation design is precisely what makes shared
semantic memory safe across multiple independent agents rather than a
shared poisoning surface.

---

## 13. Practice & Interview Questions

1. Why does an agent need three (or four, counting task-state)
   structurally separate memory scopes instead of one undifferentiated
   store?
2. What specific trigger from Module 2's reflection mechanism feeds
   directly into episodic memory writes, and why is writing the
   *resolution* more valuable than writing the original flagged
   uncertainty?
3. Explain memory poisoning in an agent system, in terms of Part 1,
   Module 9's privilege-separation principle, without using the word
   "poisoning" until your final sentence.
4. Why should an agent's own past conclusion never be promoted to
   durable semantic memory without external corroboration, even after
   several runs appear to "confirm" it?
5. Why does long-term agent memory retrieval reuse `rag-engine` rather
   than needing its own retrieval stack?
6. What's structurally different about resuming a long-horizon task
   versus retrieving relevant episodic memory for a completely new
   task?

---

## 14. Common Mistakes

- **One undifferentiated memory store** — mixes ephemeral scratch-pad
  detail, specific episodic history, and generalized semantic facts,
  degrading retrieval quality for all three.
- **Writing every scratch-pad entry to long-term memory** — the
  agent-scale version of "bare summarization isn't extraction";
  drowns genuinely reusable facts in routine noise.
- **Treating `agent_derived` conclusions as equally trustworthy as
  verified information** — the direct cause of memory poisoning; always
  a required, surfaced field, never an assumption.
- **Allowing self-corroboration to count as corroboration** — three
  runs all producing the same wrong `agent_derived` conclusion is not
  external validation; consolidation must require a genuinely
  independent, non-agent-derived source.
- **Building a second retrieval stack for memory** — reinventing what
  `rag-engine` already does well, instead of reusing it with a
  memory-specific collection.
- **Conflating episodic memory with resumable task state** — a
  long-horizon task's in-progress snapshot is not "a past run to learn
  from," it's the current run's paused state; mixing the two stores
  causes both retrieval noise and incorrect resumption.

---

## 15. Debugging Exercise

Across five recurring monthly runs of the same agent task, semantic
memory now contains a "fact" that turns out to be wrong, and no human
ever explicitly verified it. Using Section 6.4 and `poisoning-drill`,
walk through: (a) the most likely gap in the consolidation rule that let
this happen (hint: check whether "corroboration" was actually checking
for a genuinely independent source, or just counting repeated
occurrences), (b) the specific test from the Mini-Project that should
have caught this before it reached production, and (c) the fix, stated
as a change to the consolidation function's logic, not as a process
reminder to "be more careful."

---

## 16. Checklist

- [ ] I maintain scratch-pad, episodic memory, semantic memory, and (for
      long-horizon tasks) resumable task state as four structurally
      distinct stores, not one.
- [ ] Episodic memory writes are triggered specifically by task
      completion, resolved low-confidence reflection, and human
      correction — not by every scratch-pad entry.
- [ ] Every memory record carries an explicit, required `provenance`
      field with no silent default.
- [ ] My consolidation function requires genuine non-`agent_derived`
      corroboration before promoting anything to semantic memory, and
      `poisoning-drill` proves this holds under a real adversarial
      test, not just by inspection.
- [ ] Memory retrieval reuses `RAGEngine.retrieve()` against a
      memory-specific collection, with provenance surfaced (not
      flattened) in the assembled prompt.
- [ ] Resumable task-state snapshots are a separate mechanism from
      episodic/semantic memory, and I've tested an actual kill-and-
      resume cycle.
- [ ] `loop-stress-test`, `reflection-agreement-eval`, and
      `poisoning-drill` all pass together as one regression suite.

---

## 17. Summary

Agent memory is Part 3, Module 4's write/retrieve/consolidate mechanism,
kept in three structurally separate scopes (episodic, semantic,
scratch-pad) plus a fourth, narrow resumable-task-state store for
long-horizon work — with agent-specific write triggers built directly on
Module 2's reflection critiques. The genuinely hard, new problem this
module solves is memory poisoning: an agent's own unverified conclusion
must never acquire the trust level of verified information, or the
system can confidently compound its own mistakes with no external
attacker required. The structural fix — a required `provenance` field
and a consolidation rule that demands independent corroboration — is
this module's real contribution, proven not by design review but by a
deliberate adversarial drill.

---

## 18. Next Steps

Next: **Part 5, Module 4 — Multi-Agent Systems & Orchestration**, where
multiple `agent-core` instances, each with their own bounded action
space (Module 1) and now their own episodic memory scope sharing a
common, conservatively-gated semantic store (this module), are composed
under an orchestrator — with this module's provenance discipline
extended to govern what one agent is allowed to treat as trusted when
it comes from a *different* agent, not just from its own past runs.

Reply "continue" for Module 4, or flag anything to go deeper on.
