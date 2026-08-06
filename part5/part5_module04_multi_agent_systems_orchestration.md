# Part 5, Module 4: Multi-Agent Systems & Orchestration

> Composes multiple `agent-core` instances — each with its own bounded
> action space (Module 1), reflection policy (Module 2), and episodic
> memory scope sharing a conservatively-gated semantic store (Module 3)
> — under an orchestrator, extending this Part's provenance discipline
> to govern trust *between* agents, not just across an agent's own past
> runs.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. State the precise reason to use multiple agents instead of one
   larger agent with a bigger action space — a decomposition argument,
   not a "more agents is more powerful" assumption.
2. Distinguish **orchestrator-worker** (centralized dispatch) from
   **peer-to-peer** (agents communicating directly) topologies, and
   choose correctly based on how much coordination the task actually
   needs.
3. Design an inter-agent message contract that is exactly as
   structured and validated as the tool-call contract from Part 3,
   Module 3 — because an inter-agent message is a tool call, from the
   orchestrator's perspective.
4. Extend Module 3's provenance model to a fourth value,
   `peer_agent_derived`, and explain why a message from another agent
   deserves neither full trust nor the same distrust as arbitrary
   external input.
5. Identify the specific new failure mode multi-agent systems introduce
   — cascading/correlated failure across agents that share the same
   underlying model and therefore the same blind spots — and design
   against it.
6. Extend `agent-core` into a multi-agent orchestration layer,
   `agent-orchestrator`, with a concrete worked example (a two-agent
   research-and-write pipeline) that reuses every prior Part 5 artifact.

## 2. Prerequisites

- Part 5, Modules 1–3 (`agent-core` v1.2), completed.
- Part 1, Module 9 (Security Fundamentals) — privilege separation is
  about to be tested across a trust boundary you haven't faced yet: not
  human vs. model, not model vs. tool output, but agent vs. agent.
- Part 3, Module 3 (Tool Calling & MCP) — inter-agent messaging is
  designed here as a direct extension of the tool-call schema
  discipline.

## 3. Estimated Study Time

11–14 hours across 2–3 sessions.

## 4. Difficulty

★★★★☆ (4/5) — Individually, none of the pieces are new; composing them
without silently reintroducing the open-ended-trust mistakes Modules 1–3
worked hard to close off is the actual difficulty.

---

## 5. Why This Matters

It's tempting to reach for multiple agents whenever a task feels big,
the same way it's tempting to reach for microservices whenever a
codebase feels big — and the correct caution is the same in both cases:
splitting into multiple coordinating processes adds real coordination
cost (message-passing overhead, partial-failure handling, consistency
questions) that a single, well-scoped process doesn't have. Multi-agent
systems are worth that cost for a specific, statable reason — genuine
task decomposition into sub-tasks that benefit from different action
spaces, different models, or true parallelism — not because "multi-agent"
sounds more sophisticated than "agent."

This matters for the interview-readiness thread running through this
whole Part: a system-design interviewer asking about multi-agent
architecture is testing whether you reach for decomposition because a
specific coordination problem demands it, or because you've absorbed
"agents talking to agents" as an unexamined best practice. It also
matters operationally, because multi-agent systems introduce a failure
mode that single-agent systems structurally cannot have: several agents
built on the same underlying model, given similar framing of the same
underlying problem, can all make the *same* mistake independently and
then "agree" with each other about it — manufacturing false consensus
that looks like independent verification but isn't, for exactly the
same reason Module 3's self-corroboration trap wasn't real corroboration.

---

## 6. Theory

### 6.1 The actual argument for multiple agents

Three legitimate reasons to split into multiple agents, and only these
three — if none applies, prefer one well-scoped agent with a larger
action space over multiple agents:

- **Genuine sub-task specialization.** Different sub-tasks benefit from
  different, non-overlapping action spaces (Module 1's enumerated-action
  discipline) — a research agent with read-only retrieval tools and a
  writing agent with document-editing tools shouldn't share one
  undifferentiated action space, both because it bloats each `think()`
  call's decision surface and because it weakens the security benefit
  of a narrow action space (Module 1, Section 6.5) for both roles at
  once.
- **True parallelism.** Independent sub-tasks with no data dependency
  between them can run concurrently as separate agent instances,
  which a single sequential loop cannot do — this is a real throughput
  argument, not a cosmetic one, and should be justified the same way
  you'd justify any concurrency decision (Part 0, Module 12).
- **Different model/cost tiers per sub-task.** Some sub-tasks need a
  more capable (and more expensive) model; others don't. Routing
  cheaper sub-tasks to a cheaper model, per-agent, is a direct
  extension of Part 3, Module 9's model-routing discipline, now applied
  at the agent-role level instead of the single-call level.

If your actual motivation is "the task felt too big for one agent," ask
first whether it's too big because it needs decomposition for one of
the three reasons above, or whether it's too big because the action
space or the plan is under-scoped — the fix for the latter is a better
single agent, not more agents.

### 6.2 Two topologies, and the same test as Module 1's plan-then-execute question

- **Orchestrator-worker (centralized):** one orchestrating agent (or
  even non-agentic orchestration code) decomposes the task, dispatches
  sub-tasks to worker agents, and integrates their results. Workers do
  not talk to each other directly. This is easier to reason about,
  easier to log/trace centrally, and — critically for this module's
  security thread — easier to enforce provenance and trust boundaries
  on, because every inter-agent message passes through one
  chokepoint.
- **Peer-to-peer (decentralized):** agents exchange messages directly,
  without central coordination. More flexible, but the trust-boundary
  and traceability problems in Section 6.4 get substantially harder,
  because there's no single point where you can validate or filter
  inter-agent messages.

The choice mirrors Module 1's plan-then-execute question almost
exactly: **if you can specify, in advance, what each worker needs and
what it should return, a centralized orchestrator can dispatch it
cleanly.** Peer-to-peer coordination is only worth its added complexity
when sub-tasks' need for each other genuinely can't be predicted up
front — and even then, prefer starting with a centralized orchestrator
and only decentralizing the specific interaction that demonstrably
needs it, rather than defaulting to peer-to-peer for the whole system.
This module's Production Project uses orchestrator-worker for exactly
this reason: it is the correct default, not just the simpler option to
teach.

### 6.3 An inter-agent message is a tool call

The cleanest way to design orchestrator-worker communication: from the
orchestrator's point of view, dispatching a sub-task to a worker agent
*is* a tool call, structurally identical to any action in Part 3,
Module 3's tool-calling loop — a structured, schema-validated request
with typed arguments, and a structured, schema-validated response. This
means every discipline you already have for tool calls applies for
free: the worker's response schema is validated before the orchestrator
trusts it (Part 3, Module 2's structured-output discipline), and the
orchestrator's `think()` step decides *whether* to dispatch to a worker
using exactly the same enumerated-action-space reasoning as Module 1 —
"dispatch to research_worker" is just another action in the
orchestrator's action space, with its own `requires_approval` and
`reflection_policy` settings, unchanged from how any other action is
configured.

### 6.4 Provenance extends: `peer_agent_derived` is a fourth trust level

Module 3 established three provenance values for memory:
`human_verified`, `externally_sourced`, `agent_derived`. A message
arriving from another agent needs a fourth: `peer_agent_derived`. It is
neither as trustworthy as `human_verified` or genuinely
`externally_sourced` content, nor should it be treated with the same
suspicion as arbitrary, potentially-adversarial external input (Part 1,
Module 9) — a worker agent operating inside your own system, with its
own bounded action space and its own guardrails, is a different threat
model than an untrusted web page. The correct treatment: the
orchestrator's `think()` call presents worker results with explicit
provenance framing ("worker agent X reported Y, derived via its own
process, not independently verified"), and — this is the important
part — a worker's result that gets *written to shared semantic memory*
must pass through the exact same consolidation rule as Module 3's
`agent_derived` records: it needs independent corroboration before
promotion, and "another agent said so" does not, by itself, count as
independent corroboration if both agents share the same underlying
model.

### 6.5 The new failure mode: correlated failure, not independent verification

This is the sharpest new risk in multi-agent design, and it's easy to
miss because it looks like the opposite of a risk. If two agents built
on the same underlying model, given similar prompts about the same
underlying problem, both reach the same wrong conclusion, an
orchestrator that treats "two agents agree" as corroboration has been
fooled by **correlated failure masquerading as independent
verification** — the multi-agent version of Module 2's
self-reinforcing-confidence problem, and the direct multi-agent
instance of Part 2, Module 8's judge-bias-lab lesson that a judge
sharing the underlying model's blind spots isn't independent scrutiny.

**Mitigation, concretely, mirroring Module 3's corroboration rule:**
agreement between two agents only counts as real corroboration for
promotion to shared semantic memory when at least one of the following
holds: the agents used genuinely different underlying models (not just
different prompts on the same model), or at least one agent's
contribution was itself `externally_sourced` or `human_verified`
rather than `agent_derived`/`peer_agent_derived`. "Two agents, same
model, same framing, same wrong answer, both confident" is not two
opinions — it's one opinion, computed twice, and treating it as two is
the multi-agent version of double-counting evidence.

### 6.6 Partial failure is the normal case, not the exception

In a single-agent loop, one failed action is one failed step in one
loop — recoverable by the guards already built (Modules 1–2). In a
multi-agent system, one worker failing, timing out, or returning a
low-confidence result is a *routine operating condition* the
orchestrator must have an explicit policy for, not an edge case: retry
with the same worker, escalate to a different worker/model tier
(Section 6.1's third justification for multi-agent, now paying off),
or degrade gracefully and report partial results to the human. This
policy should be an explicit part of the orchestrator's action space
design, decided per worker type at the same design stage as
`requires_approval` and `reflection_policy` — not handled ad hoc by a
generic try/except wrapped around every dispatch.

---

## 7. Mental Models

1. **"More agents only if the task genuinely decomposes — specialization,
   parallelism, or cost-tiering — never because the task just feels
   big."**
2. **"A dispatched sub-task is a tool call; validate the worker's
   response exactly like any other structured tool output."**
3. **"Two agents agreeing on the same model's mistake is one opinion
   computed twice, not independent verification."**
4. **"A worker failing is Tuesday, not an incident — the orchestrator
   needs an explicit policy for it, not a try/except."**

---

## 8. Visual Explanation

```
                          orchestrator agent
                   (its own agent-core instance:
                    scratch-pad, reflection, memory)
                                  │
                     think(): decide which worker(s)
                     to dispatch to — just another
                     action in an enumerated space
                                  │
              ┌───────────────────┼───────────────────┐
              ▼                                        ▼
     ┌──────────────────┐                    ┌──────────────────┐
     │  worker agent A     │                    │  worker agent B     │
     │  (research)          │                    │  (writing)           │
     │  own action space,   │                    │  own action space,   │
     │  own episodic scope, │                    │  own episodic scope, │
     │  possibly a cheaper  │                    │  possibly a          │
     │  model tier          │                    │  different model     │
     └─────────┬────────┘                    └─────────┬────────┘
               │  structured, schema-validated result   │
               │  (= a tool response, Part 3 M3/M2)      │
               └───────────────────┬─────────────────────┘
                                  ▼
                   orchestrator's think() receives
                   results with provenance =
                   peer_agent_derived, framed as
                   unverified in the prompt
                                  │
                    partial failure? explicit policy
                    (retry / escalate tier / degrade)
                                  │
                                  ▼
                shared semantic memory — promotion
                requires genuine corroboration:
                different models OR at least one
                externally_sourced/human_verified
                contribution (NOT just "both agents
                agreed")
```

---

## 9. Recommended Resources

1. **Anthropic — "How we built our multi-agent research system"**
   (Anthropic engineering blog) — a direct, official account of
   orchestrator-worker design at production scale; compare its stated
   coordination and failure-handling choices against Sections 6.2 and
   6.6.
2. **Anthropic — "Building Effective Agents"** (revisit a third time)
   — the multi-agent/orchestrator-worker pattern section; by now you
   should be able to read it critically, checking it against your own
   provenance and correlated-failure reasoning rather than taking its
   framing as given.
3. **Your own Part 2, Module 8 (`judge-bias-lab`)** — re-read
   specifically for the shared-blind-spot argument; Section 6.5 is that
   lesson's third and sharpest application in this handbook.
4. **Your own Part 1, Module 13 (System Design Fundamentals,
   `resilient-gateway`)** — the fan-out/circuit-breaker patterns you
   built there for backend fan-out apply directly to orchestrator
   dispatch and partial-failure handling; this module is that pattern
   one layer up, with agents instead of backend services.

---

## 10. Exercises

1. Take a task and argue, using Section 6.1's three criteria, whether it
   genuinely benefits from multiple agents or would be better served by
   one agent with a larger, well-organized action space. Do this for
   both a task that should split and one that shouldn't, and be honest
   about which is harder to argue.
2. Design the message schema for dispatching a sub-task to a worker
   agent and receiving its result, treating it explicitly as a Part 3,
   Module 3-style tool call/response pair, with full schema validation.
3. Simulate two worker agents, both on the same underlying model, both
   independently reaching the same wrong conclusion on an ambiguous
   sub-task. Confirm your orchestrator's consolidation rule (Section
   6.5) correctly refuses to treat their agreement as corroboration
   for promotion to shared semantic memory.
4. Design an explicit partial-failure policy for one worker type in
   your system: what happens on timeout, on a low-confidence
   (Module 2) result, and on an outright error — and justify each
   choice.
5. Compare, for the same two-agent task, an orchestrator-worker
   implementation against a peer-to-peer implementation. Which was
   easier to add tracing and provenance enforcement to, and why does
   that match Section 6.2's argument?

---

## 11. Mini-Project

**`correlated-failure-drill`**: the multi-agent analog of Module 3's
`poisoning-drill`. Deliberately construct a scenario where two
same-model worker agents are likely to make the same mistake (an
ambiguous sub-task with a plausible-but-wrong shortcut answer), run the
full orchestrator pipeline, and confirm — by inspecting what actually
gets promoted to shared semantic memory — that mere agreement between
same-model agents does not trigger promotion. This is the empirical
proof for Section 6.5, the same way `poisoning-drill` was for Section
6.4.

---

## 12. Production Project: `agent-orchestrator`

### Scope

Build `agent-orchestrator`, a package that:

- Wraps multiple `agent-core` (v1.2) instances under one orchestrator
  agent, itself also an `agent-core` instance — the orchestrator is not
  a special new kind of thing, it's an agent whose action space happens
  to be "dispatch to worker X."
- Implements a worked, real two-agent pipeline: a **research worker**
  (read-only, `rag-engine`-backed retrieval actions, `requires_approval:
  False` throughout — low stakes) and a **writing worker** (drafts a
  document from the research worker's findings, with a final
  `publish` action set to `requires_approval: True`, per Module 1's
  discipline for irreversible actions) — a direct, concrete instance of
  Section 6.1's specialization argument.
- Implements dispatch as schema-validated tool calls/responses per
  Section 6.3, with full validation on every worker result before the
  orchestrator's `think()` sees it.
- Extends Module 3's provenance enum with `peer_agent_derived`, and
  implements the corroboration rule from Section 6.5 (different models
  OR at least one non-agent-derived contribution) as the gate for
  promoting any worker result to shared semantic memory.
- Implements an explicit per-worker-type partial-failure policy
  (Section 6.6) — for the research worker: retry once, then degrade to
  reporting partial findings; for the writing worker: escalate to a
  human via the approval-gate mechanism rather than silently publishing
  an incomplete draft.
- Ships `correlated-failure-drill` as a standing regression test
  alongside `loop-stress-test`, `reflection-agreement-eval`, and
  `poisoning-drill` — all four now run together as one suite proving
  the full Part 5 artifact chain holds under adversarial conditions.

### Explicit extension point

This two-agent research-and-write pipeline, with its
`requires_approval`-gated `publish` action, is the direct seed for
**Part 11's capstone projects** (several of which — the Research Agent,
the Multi-Agent Platform — are explicitly this pattern generalized and
productionized), and its worker-model-tiering design is what **Part 6
(AI Infrastructure)** will optimize when self-hosted serving makes
per-worker model choice a real cost lever rather than just an API
parameter.

---

## 13. Practice & Interview Questions

1. What are the three legitimate reasons to split a task across
   multiple agents, and what's the tell that a task doesn't actually
   need to be split, even if it feels large?
2. How is dispatching a sub-task to a worker agent structurally the
   same operation as a tool call, and what discipline does that
   inheritance buy you for free?
3. Two worker agents built on the same model both report the same
   conclusion on an ambiguous sub-task. Why might this not be
   corroboration, and what would make it real corroboration?
4. Compare orchestrator-worker and peer-to-peer topologies on
   traceability and provenance enforcement specifically. Which is
   easier to secure, and why?
5. Design a partial-failure policy for a worker agent that times out.
   What are the options, and what determines which one is correct for
   a given worker's role?
6. How does this module's `peer_agent_derived` provenance value relate
   to Part 1, Module 9's original privilege-separation argument?

---

## 14. Common Mistakes

- **Splitting into multiple agents because the task "feels big"** —
  without one of Section 6.1's three genuine justifications, this adds
  coordination cost for no benefit and often makes debugging harder,
  not easier.
- **Defaulting to peer-to-peer for flexibility** — usually the wrong
  default; centralized orchestration is easier to trace, secure, and
  reason about, and should only be abandoned for the specific
  interaction that demonstrably needs decentralization.
- **Trusting worker results without schema validation** — treating a
  worker's response as if it were the orchestrator's own reasoning,
  instead of validating it exactly like any other tool response.
- **Counting agent agreement as corroboration** — the single most
  dangerous mistake in this module; same-model agents agreeing on a
  mistake is correlated failure, not independent verification, and
  promoting it to shared memory compounds the error across every future
  run and every other agent that reads shared semantic memory.
- **Ad hoc, inconsistent partial-failure handling** — a generic
  try/except around dispatch calls, instead of an explicit,
  per-worker-type policy decided at design time.
- **No `correlated-failure-drill`** — implementing the corroboration
  rule and assuming it works, instead of proving it against the
  specific adversarial scenario it exists to prevent.

---

## 15. Debugging Exercise

Your two-agent research-and-write pipeline has been running for weeks.
Shared semantic memory now contains several "facts" that neither agent
independently verified against an external source — they originated
when the research worker reported something with moderate confidence,
the writing worker's draft repeated it as established, and the
orchestrator's consolidation logic treated "both agents included this in
their output" as sufficient corroboration.

Using Section 6.5 and `correlated-failure-drill`, walk through: (a) why
"both agents included this" is not the same evidence as "two
independent agents corroborated this," given that both workers likely
share the same underlying model, (b) the specific check missing from
the consolidation rule (hint: it should be checking *what kind* of
agreement occurred, not just *that* agreement occurred), and (c) the
concrete fix to the corroboration gate.

---

## 16. Checklist

- [ ] I can justify my system's use of multiple agents against Section
      6.1's three criteria, specifically, not generically.
- [ ] I chose orchestrator-worker or peer-to-peer deliberately, using
      Section 6.2's "can you specify sub-tasks in advance" test, not by
      default.
- [ ] Every worker dispatch is a schema-validated tool call/response
      pair, validated before the orchestrator's `think()` sees it.
- [ ] My provenance enum includes `peer_agent_derived`, and worker
      results are framed with explicit unverified-provenance language
      in the orchestrator's prompt.
- [ ] My consolidation rule for promoting worker results to shared
      semantic memory checks for genuine corroboration (different
      models, or a non-agent-derived source) — not mere agreement.
- [ ] `correlated-failure-drill` passes, proving the corroboration rule
      holds under the specific adversarial scenario it's designed for.
- [ ] Each worker type has an explicit, designed-in-advance
      partial-failure policy, not a generic catch-all.
- [ ] `loop-stress-test`, `reflection-agreement-eval`,
      `poisoning-drill`, and `correlated-failure-drill` all pass
      together as one suite.

---

## 17. Summary

Multi-agent systems are a decomposition decision, justified by genuine
specialization, parallelism, or cost-tiering — not a default reached for
whenever a task feels large. Dispatching to a worker is structurally a
tool call, inheriting Part 3's validation discipline for free. The
module's real contribution is naming and defending against the failure
mode that's genuinely new at this scale: same-model agents agreeing on
a shared mistake is correlated failure, not independent corroboration,
and a consolidation rule that doesn't distinguish the two will let a
multi-agent system confidently poison its own shared memory faster than
a single agent ever could — proven, not assumed, by
`correlated-failure-drill`.

---

## 18. Next Steps

Part 5's core mechanisms — the loop, reflection, memory, and
orchestration — are now in place. Next: **Part 5, Module 5 —
Specialized Agents (Browser, Coding, Research, Voice)**, applying this
foundation to concrete agent categories with domain-specific action
spaces, each inheriting the full guard/reflection/memory/orchestration
stack built in Modules 1–4 rather than reinventing safety mechanisms
per domain.

Reply "continue" for Module 5, or flag anything to go deeper on.
