# Part 5, Module 6 (Capstone): Agent Evaluation & Production Readiness

> Closes out Part 5 exactly the way Part 4, Module 9 closed out Part 4:
> not a new mechanism, but the disciplined assembly, seam-auditing, and
> production-hardening of everything built across Modules 1–5.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Trace the **actual runtime execution order** of a full agent run —
   single-agent and multi-agent — across all five prior modules, and
   name the places that order is easy to get wrong.
2. Audit at least five **cross-component seams** across the Part 5
   stack (loop ↔ reflection, reflection ↔ memory, memory ↔
   orchestration, orchestration ↔ specialized domains, and the
   provenance chain end-to-end) and name the specific bug each
   produces when violated.
3. Assemble five independently-built regression drills
   (`loop-stress-test`, `reflection-agreement-eval`, `poisoning-drill`,
   `correlated-failure-drill`, and Module 5's domain-specific tests)
   into one coordinated, tiered CI acceptance gate.
4. Ship `agent-core` + `agent-orchestrator` as one versioned,
   documented package with a stable public interface, ready for Part
   11's capstone projects to build on directly.
5. Write the ADRs, runbook, and API reference a new engineer would need
   to safely operate and extend this system.
6. Give an honest, specific account of Part 5's limitations —
   distinguishing "not yet built" from "architecturally out of scope"
   — and name exactly which future Part addresses each.

## 2. Prerequisites

- Part 5, Modules 1–5, completed, with all artifacts on disk and
  runnable: `agent-core` v1.2 (loop, reflection, memory/provenance),
  `agent-orchestrator` (multi-agent), and at least one specialized
  agent plus `domain-risk-map` from Module 5.
- Part 4, Module 9 — you have already done exactly this kind of
  integration audit once, one layer down the stack; this module is
  that discipline applied to agents.

## 3. Estimated Study Time

13–16 hours across 3 sessions.

## 4. Difficulty

★★★★★ (5/5) — Capstone. As with Part 4's capstone, nothing here is
conceptually new; the difficulty is entirely in proving, not assuming,
that five independently-correct modules compose into one correct
system.

---

## 5. Why This Matters

Every module in Part 5 was validated against its own regression drill,
in isolation: `loop-stress-test` proved the loop terminates correctly.
`poisoning-drill` proved memory consolidation resists self-poisoning.
`correlated-failure-drill` proved the orchestrator doesn't mistake
correlated failure for corroboration. But — the same lesson as Part 4,
Module 9, one layer up — proving each component correct in isolation
does not prove the assembled system is correct, because seam defects
are by construction invisible to any single component's own tests.

Concretely, here is the kind of bug this module exists to catch: Module
2's reflection correctly flags a step low-confidence. Module 3's memory
correctly requires resolution before writing an episodic record. Module
4's orchestrator correctly refuses same-model agreement as
corroboration. All three guarantees can be individually true, and the
system can *still* leak a poisoned, unresolved, `agent_derived`
conclusion into shared semantic memory — if the seam between "reflection
flagged this as unresolved" and "memory's write trigger fires anyway"
is wired wrong, because nobody happens to unit-test the exact boundary
between those two modules' state representations. This is precisely why
the module ends with a seam audit before any new code is written, the
same discipline as Part 4's capstone.

This is also the module where Part 5 becomes a citable portfolio
artifact with a specific, defensible claim: not "I built an agent
framework," but "I designed a bounded, provenance-aware, multi-agent
system with a measured, adversarially-tested resistance to memory
poisoning and correlated failure, assembled behind one versioned
interface, ready to be handed a retrieval tool or a new specialized
domain without touching its internals." That claim is what a system
design interview at an AI lab is actually probing for.

---

## 6. Theory

### 6.1 The full runtime timeline, across all five modules

A single query into a multi-agent system, tracing every module's
contribution in the order it actually executes (not the order it was
built):

```
 1. task arrives at the orchestrator (an agent-core instance itself,
    Module 4)
 2. orchestrator's think() checks its bounded-loop guards (Module 1:
    iteration cap, cost budget, repetition fingerprint) BEFORE
    considering any action, including "dispatch to worker"
 3. orchestrator retrieves relevant episodic/semantic memory (Module 3)
    via rag-engine, with provenance-labeled framing in the prompt
 4. orchestrator's think() decides: single-agent path, or dispatch to
    worker(s) (Module 4) — "dispatch" is itself just an enumerated
    action, subject to the same requires_approval/reflection_policy
    discipline as any other action
 5. IF dispatching: each worker agent runs its OWN full loop
    (steps 2-9 recursively, one level down) inside its own bounded
    guards, its own domain-specific action space (Module 5), and its
    own episodic memory scope
 6. worker's action executes (Module 1's execute()) -- for a
    specialized domain, this is where Module 5's domain-specific
    boundary lives: sandbox (coding), input-guardrail-filtered
    observation (browser), rag-engine call (research), or
    latency-restructured guard timing (voice)
 7. worker's reflection (Module 2) runs per its configured policy
    (confidence-only vs. plan-altering, sync vs. async per Module 5's
    voice-specific restructuring)
 8. worker's result returns to the orchestrator as a schema-validated
    message, tagged provenance = peer_agent_derived (Module 4)
 9. orchestrator's own reflection evaluates the worker's result against
    the sub-goal it was dispatched to serve (Module 2, one level up)
10. orchestrator's memory-write trigger (Module 3) fires ONLY if:
    task completed, OR a low-confidence reflection was subsequently
    resolved, OR a human correction occurred -- and consolidation to
    SHARED semantic memory additionally requires Module 4's
    corroboration check (different models, or a non-agent-derived
    source)
11. any requires_approval=True action anywhere in this tree pauses
    for human approval before executing, using a channel-appropriate
    mechanism (Module 5: click-based for text/browser, voice-native
    confirmation or deferred-channel for voice)
12. final result returned; full trace (loop decisions, reflections,
    memory writes, provenance tags) logged for the CI gate's Tier 3
    analysis
```

The critical thing to notice, and the source of the most common seam
bug: **step 5 is recursive.** A worker agent is not a simplified,
guard-free version of an agent — it is a full `agent-core` instance
with its own copy of every guard in steps 2, 3, 7, 9, 10, 11. A common
implementation shortcut — building a lightweight "worker" that skips
memory or reflection to save complexity, on the reasoning that "the
orchestrator handles the important stuff" — silently reintroduces every
failure mode Modules 1–3 closed, just one level down, where the
orchestrator's own guards have no visibility into it.

### 6.2 Seam 1: reflection's "unresolved" state vs. memory's write trigger

Module 2 draws a distinction between confidence-only and plan-altering
reflection, and Module 3 specifies that episodic memory writes should
capture *resolved* low-confidence outcomes, not raw unresolved flags.
The seam risk: if reflection's output schema doesn't have an explicit
`resolved: bool` field distinct from `satisfied: bool`, the memory-write
trigger has no reliable way to distinguish "this step's uncertainty was
subsequently addressed" from "this step is still uncertain and the loop
just moved on anyway" (e.g., because it hit the iteration cap). Writing
the latter to episodic memory as if it were the former is exactly the
seed of the poisoning scenario in Section 5's opening example — and
`poisoning-drill` alone won't catch it, because that drill tests
whether *already-written* `agent_derived` records get over-promoted; it
does not test whether an *unresolved* flag gets written as if resolved
in the first place. **This module's audit requires you to check: does
your reflection schema expose `resolved` as a field distinct from
`satisfied`, and does the memory-write trigger actually check it?**

### 6.3 Seam 2: worker provenance vs. orchestrator's own provenance framing

Module 4 introduces `peer_agent_derived` for worker results. The seam
risk: when the orchestrator's own reflection (step 9 above) evaluates a
worker's result and produces its own conclusion about it, what
provenance does *that* conclusion carry going forward? If the
orchestrator's reflection output is itself written to shared memory
without re-tagging, it's easy for a `peer_agent_derived` input to get
silently re-labeled as `agent_derived` (the orchestrator's own
reflection, technically true) — losing the fact that it ultimately
traces back to an unverified worker. **Audit requirement:** provenance
should be a property that can only be *added to*, never *replaced* —
a record's full provenance chain (which agents touched it, in what
role) should be inspectable, not collapsed to the single most recent
agent's label.

### 6.4 Seam 3: does the orchestrator's cost budget account for worker costs?

Module 1's cost budget guard was designed for a single loop. In the
recursive structure from Section 6.1, a worker's entire loop — its own
iterations, its own reflection calls — happens "underneath" one
orchestrator action (`dispatch_to_worker`). **Audit requirement:**
confirm the orchestrator's cost budget is charged for the *worker's
total run cost*, not just the cost of the dispatch message itself —
otherwise a single expensive worker run can silently blow through a
budget the orchestrator's own guard believes is still healthy, because
the guard was only ever wired to see its own direct LLM calls.

### 6.5 Seam 4: does a specialized domain's guard restructuring (Module 5) actually reach the shared regression suite?

Module 5 deliberately restructured guard *timing* for voice (async
reflection, deferred approval) without changing the guarantee. The seam
risk: `loop-stress-test`, `reflection-agreement-eval`, etc. were written
against the *synchronous* Modules 1–4 stack's assumptions. If a voice
agent's async-reflection path is never actually exercised by the
existing drills — because they assume reflection completes before the
loop proceeds — you can have a voice agent that looks fully compliant
by code review while its regression suite is silently only testing the
text-agent code path. **Audit requirement:** confirm each specialized
domain built in Module 5 is actually exercised by the Tier assignments
in Section 6.7's CI gate, not just present in the codebase.

### 6.6 Seam 5: does a human-approval pause correctly preserve resumable state?

Module 1 introduced the approval pause; Module 3 introduced resumable
task-state snapshots for long-horizon tasks. The seam risk: an approval
pause that isn't itself a snapshot point — if the process restarts while
awaiting approval (a real production scenario, not an edge case) and the
pending-approval state wasn't captured by Module 3's snapshot mechanism,
the task silently loses track of exactly the decision it was waiting on
a human for. **Audit requirement:** confirm every `requires_approval`
pause is a valid resumption point, tested by an actual kill-and-resume
cycle during an active approval pause, not just during ordinary
mid-loop execution as Module 3's original exercise tested.

### 6.7 Assembling the CI gate

Mirroring Part 4, Module 9's tiering exactly, now for agents:

- **Tier 1 (blocking, every PR, fast):** contract tests for the seams
  in Sections 6.2–6.6 (schema/field-presence checks, cheap to run),
  `poisoning-drill`, `correlated-failure-drill`.
- **Tier 2 (blocking, every PR, slower):** `loop-stress-test`,
  `reflection-agreement-eval`, and — critically per Section 6.5 — each
  specialized domain's own regression tests from Module 5, run for
  real, not skipped because they're "domain-specific."
- **Tier 3 (nightly, non-blocking but alerting):** full end-to-end
  multi-agent runs with complete trace logging, reviewed for drift in
  reflection-agreement rates and provenance-chain integrity over time.

As in Part 4, poisoning and correlated-failure resistance are treated
as security-class failures and belong in the hard-blocking tier, not
alongside general quality regressions.

---

## 7. Mental Models

1. **"A worker agent is a full agent, with every guard — not a
   lightweight helper the orchestrator watches instead."**
2. **"Provenance only ever accumulates; the newest agent's label never
   erases where a fact actually came from."**
3. **"A cost budget that doesn't see a worker's real spend isn't
   protecting you — it's an assumption wearing a guard's uniform."**
4. **"If a guard's timing changed for one domain, the regression suite
   has to actually exercise that changed timing, or it's testing a
   system you no longer ship."**

---

## 8. Visual Explanation

```
                    orchestrator's think()
                              │
             ⚠ SEAM 4: cost budget must include
                worker's TOTAL run cost, not just
                the dispatch message
                              │
                              ▼
                dispatch_to_worker (Module 4)
                              │
              ┌───────────────┴────────────────┐
              ▼   worker = full agent-core       ▼
      instance, own full loop (Modules 1-3),
      own domain-specific boundary (Module 5)
              │
              ▼
        worker's own reflection (Module 2)
              │
   ⚠ SEAM 2: result tagged peer_agent_derived --
      provenance ACCUMULATES, never replaced,
      even after orchestrator's own reflection
      touches it
              │
              ▼
      orchestrator's reflection on the worker's
      result (Module 2, one level up)
              │
   ⚠ SEAM 1: reflection schema needs `resolved`
      distinct from `satisfied` -- memory write
      trigger (Module 3) checks resolution,
      not mere presence of a flag
              │
              ▼
        memory write + consolidation
   ⚠ SEAM 3: consolidation requires genuine
      corroboration (Module 4) -- agreement
      alone is not enough
              │
              ▼
   requires_approval pause anywhere in this tree
   ⚠ SEAM 5: must be a valid resumption point
      (Module 3 snapshot mechanism), tested by an
      actual kill-and-resume DURING the pause
```

---

## 9. Recommended Resources

1. **Your own Part 4, Module 9** — reread it immediately before starting
   this module's seam audit; the discipline (runtime order first, seam
   audit before new code, tiered CI gate, honest limitations table) is
   identical, one layer up the stack.
2. **Anthropic — "How we built our multi-agent research system"**
   (revisit) — specifically for how a production system handles the
   recursive cost/guard-propagation problem in Section 6.4.
3. **Martin Fowler — "Contract Tests"** (revisit, Part 4 Module 9's
   resource) — the seam-audit tests in this module are the same
   pattern, applied to reflection/memory/orchestration interfaces
   instead of retrieval interfaces.
4. **Your own Module 5 `domain-risk-map`** — the most relevant resource
   for Section 6.5's audit is your own design document; check it
   against what actually got tested, not what you intended to test.

---

## 10. Exercises

1. Write out, from memory, the 12-step recursive runtime timeline from
   Section 6.1, including the fact that step 5 recurses into a full
   copy of steps 2–9. Compare against the original and note every
   divergence.
2. For each of the five seams in Sections 6.2–6.6, write one contract
   test that would fail if that seam were violated, independent of
   testing any single module's internal correctness.
3. Instrument your orchestrator's cost budget to log worker-attributed
   cost separately from direct cost. Run one multi-agent task and
   confirm the total matches what the budget guard actually checked
   against.
4. Simulate a process restart during an active `requires_approval`
   pause (not during ordinary execution). Confirm the task resumes
   correctly at the pending approval, not from a stale or lost state.
5. Pick one specialized domain from Module 5. Confirm, by actually
   running it, that its regression test is present and passing in your
   Tier 2 CI gate — not merely present in the repository.

---

## 11. Mini-Project

**`agent-seam-audit-report`**: the direct sequel to Part 4, Module 9's
Mini-Project. Manually trace three real multi-agent runs (at least one
involving a `requires_approval` pause, at least one involving a
low-confidence reflection that gets resolved) end-to-end through the
12-step timeline, recording at each step what state was present and
what provenance tag it carried. Flag any step where resolution status,
provenance, or cost attribution silently disappeared or got
overwritten.

---

## 12. Production Project: `agent-core` v2.0 (Part 5 capstone package)

### Scope

Ship a single versioned package, extending `llm-client-core` and
composing `rag-engine`, that:

- Merges `agent-core` (Modules 1–3) and `agent-orchestrator` (Module 4)
  into one coherent public interface:

```python
class Agent:
    """A single bounded agent -- the worker AND the orchestrator
    are both instances of this class; an orchestrator differs only
    in having 'dispatch to another Agent' in its action space."""

    async def run(self, task: str, identity: RequesterIdentity) -> AgentResult:
        """
        Enforces, structurally, in order:
        loop guards (1) -> memory retrieval (3) -> think/act (1) ->
        domain-specific execution boundary (5) -> reflection (2),
        with `resolved` tracked distinctly from `satisfied` ->
        memory write + provenance-accumulating consolidation (3, 4) ->
        approval gating with resumable snapshot support (1, 3) at
        every requires_approval boundary, including nested worker
        calls.
        """
```

- Fixes all five seams from Sections 6.2–6.6 structurally, not by
  convention: `resolved` as a first-class reflection field,
  provenance as an append-only chain, worker cost propagated into the
  dispatching agent's budget, specialized-domain guard timing actually
  exercised by CI, and every approval pause as a valid snapshot point.

- Assembles `agent-ci-gate`: the three-tier suite from Section 6.7,
  including every regression drill built across Modules 1–5, with
  `poisoning-drill` and `correlated-failure-drill` as hard CI blockers.

- Ships with:
  - **ADRs** for: the loop's termination-guard design, the
    reflection/memory resolution-tracking schema, the provenance
    accumulation model, and the orchestrator-worker vs. peer-to-peer
    default choice.
  - **Runbook**: agent-specific `observability-stack` metrics
    (iteration counts per run, reflection agreement rate over time,
    memory promotion rate and rejection rate, cost-per-run broken down
    by direct vs. worker-attributed), and what an on-call engineer does
    when each degrades.
  - **API reference** for `Agent.run()`, written for a consumer who has
    never seen the internals — the document Part 6 and Part 11 will
    use directly.

### Explicit extension point

**Part 6 (AI Infrastructure)** will replace the underlying model calls
inside `Agent` with self-hosted serving for latency-sensitive
deployments (directly relevant to Module 5's voice-agent timing
constraints), without changing `Agent.run()`'s public interface —
the actual test, as with Part 4's capstone, of whether the
encapsulation was done correctly. **Part 7 (Frontend for AI)** will
build the first real UI for the approval-gate mechanism, replacing
Module 1's CLI placeholder. **Part 11's capstone projects**
(Research Agent, AI Code Reviewer, Browser Agent, Voice Assistant,
Multi-Agent Platform) all build directly on `agent-core` v2.0's public
interface, using Module 5's domain-specific designs as their starting
point.

---

## 13. Practice & Interview Questions

1. Why does a worker agent in a multi-agent system need the exact same
   guards as a top-level agent, rather than a simplified version?
2. Explain the difference between a reflection outcome that's
   `satisfied` and one that's `resolved`, and why conflating them can
   lead to a memory-poisoning bug that `poisoning-drill` alone wouldn't
   catch.
3. Why should provenance be append-only rather than replaceable, and
   what specific bug does a replaceable provenance field introduce in a
   multi-agent system?
4. A multi-agent system's cost budget guard reports healthy usage while
   the actual bill is far higher than expected. What's the most likely
   architectural gap?
5. Why does changing a guard's *timing* (as Module 5 did for voice
   agents) require updating the regression suite, not just the
   implementation?
6. Design the resumption behavior for a process that restarts while a
   `requires_approval` action is pending. What state has to be
   captured, and by which existing mechanism?
7. What's the practical difference between "I built an agent
   framework" and the more specific, defensible claim this module's
   capstone lets you make in an interview?

---

## 14. Common Mistakes

- **Building lightweight, guard-free workers "for simplicity"** —
  silently reintroduces every Module 1–3 failure mode one level down,
  invisible to the orchestrator's own tests.
- **Conflating `satisfied` and `resolved` in reflection output** — the
  single most consequential schema gap in this module; memory writes
  need to know which one they're checking.
- **Replaceable provenance fields** — losing a fact's true origin the
  moment any agent's own reflection touches it.
- **Cost budgets scoped only to direct calls** — missing the dominant
  cost driver in any real multi-agent system, which is almost always
  worker fan-out, not the orchestrator's own `think()` calls.
- **Treating specialized-domain guard changes (Module 5) as
  implementation details that don't need dedicated CI coverage** —
  produces a regression suite that's green while testing a code path
  the system no longer actually uses for that domain.
- **Approval pauses that aren't resumption points** — assuming Module
  3's snapshot mechanism "just handles it" without testing the specific
  case of a restart during an active pause, which is a different code
  path from an ordinary mid-loop restart.

---

## 15. Debugging Exercise

A production incident: a multi-agent research-and-write pipeline
restarted mid-run (a routine deployment, not a crash) while a
`publish`-action approval was pending. On restart, the task resumed, but
the human's original approval request was lost — the agent proceeded as
if no approval had ever been required, and published without one.

Using Sections 6.6 and the Mini-Project, walk through: (a) the most
likely gap in how the approval pause interacted with Module 3's
resumable snapshot (hint: check whether the snapshot mechanism, tested
in Module 3 for ordinary mid-loop restarts, was ever specifically tested
for a restart *during* a pending approval, as distinct from a restart
during normal execution), (b) the specific test that should have caught
this before deployment, and (c) the structural fix — stated as a change
to what the snapshot captures and how `Agent.run()` checks it on
resumption, not as a process reminder to "test restarts more."

---

## 16. Checklist

- [ ] I can draw the full 12-step recursive runtime timeline from
      memory, including that worker execution recurses through the
      same steps as the top-level agent.
- [ ] I can explain, in one sentence each, all five seams from Sections
      6.2–6.6 and the specific bug each produces when violated.
- [ ] My reflection schema distinguishes `resolved` from `satisfied`,
      and my memory-write trigger checks the correct field.
- [ ] Provenance is append-only in my implementation, verified by a
      test that an orchestrator's own reflection on a worker's result
      doesn't erase the `peer_agent_derived` origin.
- [ ] My cost-budget guard is charged for a dispatched worker's total
      run cost, verified by an actual cost-accounting test, not
      assumed.
- [ ] Every specialized domain from Module 5 has its regression test
      actually running (not just present) in my Tier 2 CI gate.
- [ ] Every `requires_approval` pause is a tested, valid resumption
      point, including a real kill-and-resume test during an active
      pause specifically.
- [ ] `agent-ci-gate` runs all five prior drills plus this module's
      seam contract tests, with poisoning and correlated-failure
      resistance as hard blockers.
- [ ] `Agent` is the single public interface; internal loop, reflection,
      memory, and orchestration components are not separately
      importable by consumers.
- [ ] I've written the ADRs, runbook, and API reference — and the API
      reference alone is enough for Part 6 or Part 11 to build on this
      without reading the internals.
- [ ] I can name Part 5's limitations (no self-hosted low-latency
      serving, no real approval-gate frontend, foundational-only
      deployment maturity) and which future Part addresses each.

---

## 17. Summary

Part 5 built five independently strong mechanisms: a bounded loop,
reflection with calibrated confidence, provenance-aware memory
resistant to self-poisoning, orchestration resistant to correlated
failure, and domain-specific calibration for four real agent
categories. This module didn't add a sixth mechanism — it proved,
through explicit seam audits rather than hope, that the five compose
correctly end-to-end: a worker agent is a full agent with every guard,
not a lightweight helper; reflection's resolution state is tracked
distinctly from mere satisfaction so memory writes can't be fooled;
provenance only ever accumulates; cost budgets see a worker's true
cost; specialized-domain guard-timing changes are actually exercised by
CI; and every approval pause is a real, tested resumption point. All of
that is now behind one interface, `Agent`, versioned and documented well
enough that Part 6 can swap in self-hosted serving, Part 7 can build a
real approval-gate UI, and Part 11's capstones can build directly on it
without understanding — or being able to bypass — any of Part 5's
internals.

The honest limitations, and where each is addressed:

| Limitation | Addressed in |
|---|---|
| No self-hosted, latency-optimized model serving (directly relevant to Module 5's voice-agent constraints) | Part 6 — AI Infrastructure |
| No real frontend for approval gates or agent monitoring (CLI placeholder only) | Part 7 — Frontend for AI |
| Foundational-only production deployment maturity | Part 8 — Production AI |

---

## 18. Next Steps

Part 5 is complete. `agent-core` v2.0 is your third major shipped
package artifact, composable with `llm-client-core` (Part 3) and
`rag-engine` (Part 4) — together, the full stack Part 11's capstone
projects will assemble from directly.

Part 6 (AI Infrastructure) begins next: vLLM, Ollama, LiteLLM, GPU
fundamentals, quantization, and self-hosted model serving — Pankaj's
platform-engineering track, building on Part 0, Module 14's cloud
fundamentals and Part 1's `ai-api-platform`, and directly addressing
this module's first named limitation by giving Part 5's agents
(especially the voice agent from Module 5) a self-hosted, latency-
optimized serving path instead of a hosted API call.

---

Reply "continue" for Module N, or flag anything to go deeper on.
