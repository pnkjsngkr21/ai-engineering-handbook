# Part 3, Module 12 (Capstone): A Finished, Production-Grade Conversational AI Service

> This is where eleven modules of individually-built, individually-
> evaluated components become one thing: a real, deployable
> conversational AI service you could put in front of actual users
> tomorrow. Nothing new is invented here. This module is entirely about
> assembly, integration testing, documentation, and holding the whole
> system to the same evidence-based standard every component was held to
> individually — and closing Part 3 with a system, not a pile of parts.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Assemble eleven previously-independent `llm-client-core` components
   (provider adapters, structured output, tool calling + MCP, memory,
   pipeline evaluation, guardrails, context budgeting, caching, latency/
   cost optimization, hallucination reduction, conversation policy) into
   one coherent, correctly-ordered request-handling pipeline.
2. Identify and resolve integration-level interactions between
   components that don't show up when each was tested in isolation —
   e.g., how context budgeting (Module 7) and caching (Module 8)
   interact, or how routing (Module 9) affects hallucination-reduction
   policy (Module 10).
3. Define and justify a complete acceptance-test suite for a production
   conversational AI service, spanning every evaluation surface built
   across Part 3 (single-call, pipeline-trace, guardrail effectiveness,
   cache correctness, routing accuracy, hallucination rate, multi-turn
   conversation quality).
4. Produce production-grade documentation (architecture decision
   records, a runbook, an API reference) for a system of this
   complexity, at the standard expected of a real engineering team
   handoff.
5. Conduct a structured architecture review of the finished system,
   identifying its genuine limitations and explicitly naming which
   later Parts of the handbook (RAG, Agents, Infrastructure) address
   each one — rather than presenting the system as complete or final.
6. Deliver `llm-client-core` as a finished, versioned, documented
   package — the running artifact that Parts 4 through 8 will build on
   directly, exactly as originally promised when this package was first
   introduced in Part 1, Module 2.

---

## 2. Prerequisites

- **All of Part 3, Modules 1-11.** This module assumes every prior
  production project in Part 3 is complete and passing its own tests;
  it does not reteach any of them.
- **Part 1, Module 1** (Clean Architecture) — you'll be reviewing
  `llm-client-core`'s overall structure against the layering principles
  established there, since eleven modules of incremental extension is
  exactly the scenario where architectural drift accumulates if not
  periodically checked.
- **Part 1, Module 8** (CI/CD) — the capstone's full-system CI pipeline
  extends the tiered fast/slow pipeline discipline from that module.

---

## 3. Estimated Study Time

**12–16 hours** (this is explicitly a larger, integration-focused
module — budget more time than a typical module; most of it is
integration testing, documentation, and architecture review rather than
new theory).

---

## 4. Difficulty

★★★★☆ (4/5)

No single piece here is conceptually new. The difficulty is entirely
integration complexity: eleven components that were each individually
correct can still interact in ways that produce genuinely new failure
modes, and finding those requires the same systematic, evidence-driven
discipline this whole part of the handbook has been building toward.

---

## 5. Why This Matters

This is the module that answers the question every hiring manager and
every senior engineer actually cares about: not "do you understand
prompting, tool calling, and RAG individually" but "can you assemble
these into something that actually works, that you'd be willing to put
your name on, that's documented well enough for someone else to operate
and extend." A portfolio of eleven isolated demos is meaningfully weaker
evidence of engineering ability than one integrated, evaluated,
documented system — and for your specific goals, this capstone is
exactly the artifact you'd want to walk through in a DoorDash-style
system-design interview or describe in a platform-engineering
application at an AI lab, because it demonstrates the full arc from
mechanistic understanding (Part 2) through disciplined, evidence-based
systems engineering (Part 3), not just familiarity with API surfaces.

---

## 6. Theory

### 6.1 Assembly order: why the sequence you build in isn't the sequence requests flow through

Across Modules 1-11 you built components roughly in the order a
developer needs them (get calls working, then make them structured,
then let them act, then give them memory, then evaluate, then guard,
then optimize context/cost, then reduce hallucination, then govern the
conversation). **The actual request-handling pipeline has a different,
specific execution order**, and getting this order wrong is a real
integration bug class, not a theoretical concern:

```
1. Input guardrail (Module 6)          — reject/sanitize before anything else runs
2. Session-boundary check (Module 11)  — is this a new session? does memory need
                                          to fire from the PRIOR session first?
3. Long-term memory retrieval (Module 4) — before assembling context, not after
4. Context budget allocation (Module 7)  — decide what fits, in what order,
                                            BEFORE rendering the prompt
5. Prompt rendering (Module 1)           — using the budgeted, ordered content
6. Routing decision (Module 9)           — which model tier handles this call
7. Cache check (Module 8)                — before calling the model at all,
                                            if this call type is cacheable
8. Model call (with structured output    — Module 2, tool calling Module 3
   and/or tool-calling loop)
9. Action guardrail (Module 6)           — before executing any requested tool
10. Hallucination-reduction checks       — grounding/citation verification
    (Module 10)                            (Module 10) on the generated claims
11. Output guardrail (Module 6)          — final check before returning
12. Clarification/repair policy          — is this response itself a
    (Module 11)                            clarifying question? does it need
                                            conversation-repair handling?
13. Long-term memory write trigger       — only if a session-boundary signal
    (Module 4, gated by Module 11)         fired in this turn
```

Notice several things this ordering makes explicit that weren't
necessarily explicit when each module was built in isolation: memory
retrieval (step 3) must happen *before* context budgeting (step 4), not
after, because the budget allocator needs to know what memory content
exists in order to allocate space for it — building these in the wrong
order in Module 4 vs. Module 7 could easily produce a system where
memory is retrieved but never actually makes it into the budgeted
prompt. Caching (step 7) happens *after* routing (step 6) decides which
model tier's cache to check, not before — a cache hit is specific to a
model tier, and checking before routing risks returning a
wrong-tier-cached response. This kind of ordering dependency is exactly
the class of bug that per-module isolated testing (which each of
Modules 1-11 correctly did) cannot catch, and is the primary reason this
capstone module exists as genuine integration work, not just a victory
lap.

### 6.2 Cross-component interactions worth auditing explicitly

Beyond ordering, several pairs of components interact in ways worth
deliberately checking, because each was designed with the other's
*existence* in mind but not necessarily tested against it directly:

- **Context budgeting (Module 7) × caching (Module 8)**: if the budget
  allocator truncates or reorders content differently between two
  calls that should otherwise hit the same cache entry, the cache key
  (which should reflect the actual rendered prefix) may incorrectly
  miss — or worse, incorrectly hit despite the content actually being
  different. Verify the cache key is computed *after* budget allocation
  is applied, not before.
- **Routing (Module 9) × hallucination reduction (Module 10)**: a
  request routed to a cheaper model tier may have a higher baseline
  hallucination rate than one handled by the capable tier — verify
  your `hallucination-eval-suite` (Module 10) is run *per routing
  tier*, not only in aggregate, echoing Module 9's own per-tier
  accuracy discipline directly.
- **Session boundaries (Module 11) × memory retrieval (Module 4)**: if
  a session-boundary-triggered memory write hasn't completed before the
  *next* session's retrieval step runs (a genuine race condition risk
  if writes are processed asynchronously via `job-processor`, Part 1
  Module 6), the next session could retrieve stale memory. Verify write
  completion is confirmed, or explicitly accounted for as eventually
  consistent, before the next session's retrieval depends on it.
- **Guardrails (Module 6) × streaming (Module 9)**: an output guardrail
  that needs the complete response to make its decision cannot operate
  on a token-by-token streamed response the same way — verify your
  streaming implementation correctly buffers whatever the output
  guardrail needs before surfacing content, rather than streaming
  ungated content directly to the user.

None of these are new mechanisms to build — they're integration tests
to write, specifically targeting the seams between components that
were each independently correct.

### 6.3 What "production-grade" documentation actually requires

A system this complex, handed to someone else (a teammate, a future
you, an interviewer reading your portfolio), needs three distinct
documentation artifacts, each serving a different reader and purpose —
skipping any one of them is a real gap, not a nice-to-have:

- **Architecture Decision Records (ADRs)**, extending the practice from
  Part 1, Module 1: for each major cross-cutting decision made across
  Modules 1-11 (why hybrid compaction over pure sliding window, why
  escalation-based routing over upfront classification, why three
  distinct caching layers rather than one), a short, dated record of
  the decision, the alternatives considered, and why this one was
  chosen — this is what lets a future engineer (including future-you)
  understand *why* the system looks the way it does, not just what it
  does.
- **A runbook**: for someone operating this system in production —
  what each `observability-stack` metric family means (per-consumer
  token cost, cache-hit rates, guardrail false-positive/negative rates,
  hallucination rate, per-tier routing accuracy), what a concerning
  value looks like for each, and what to check first when something
  looks wrong (directly reusing the debugging-exercise patterns built
  throughout Part 3 as the actual on-call playbook).
- **An API reference**: for someone building against this system —
  what `llm-client-core`'s public interface actually looks like now
  (after eleven modules of extension), with the `PromptTemplate`,
  `ToolDefinition`, `ConversationManager`, `ContextBudgetManager`, and
  guardrail/hallucination-reduction integration points all documented
  as a coherent public API, not just as scattered internal
  implementation details.

### 6.4 Architecture review: naming the system's real limitations honestly

A genuinely mature capstone doesn't present the finished system as
"done" — it explicitly names what it doesn't yet handle, because that
honesty is itself evidence of engineering maturity, and because it sets
up the rest of the handbook correctly. At minimum, your architecture
review should explicitly name:

- **No retrieval over a large, external document corpus** — the system
  grounds against tool results and memory, but has no general RAG
  capability yet; **Part 4 addresses this directly**, extending the
  grounding mechanism from Module 10 into a full retrieval architecture.
- **No autonomous, multi-step planning** — tool calling here is
  reactive (responds to the current turn's needs), not a planning agent
  that decomposes a complex goal into a sequence of self-directed steps;
  **Part 5 addresses this directly**, building on this module's
  tool-calling and guardrail infrastructure.
- **No self-hosted or optimized model serving** — the system calls
  hosted provider APIs exclusively; it has no vLLM/quantization/
  self-hosted-inference story; **Part 6 addresses this directly**, and
  is Pankaj's specific platform-engineering track.
- **No dedicated frontend** — the system's conversational surface is
  API-level only; **Part 7 addresses this directly**.
- **Deployment/monitoring/DR maturity is foundational, not
  comprehensive** — `observability-stack` and CI are real, but a full
  production deployment story (secrets management, disaster recovery,
  staged rollout) is a distinct, deeper topic; **Part 8 addresses this
  directly**.

Naming these explicitly, with the specific future module that addresses
each, converts "here's a system with unstated gaps" into "here's a
system with a clear, deliberate scope boundary and an explicit roadmap"
— a materially stronger artifact for both a portfolio and your own
understanding of where you actually are in the curriculum.

---

## 7. Mental Models

1. **"The build order isn't the execution order."** Eleven modules built
   roughly in dependency order does not mean the finished request
   pipeline should execute in that same order — trace the actual
   runtime sequence explicitly and verify it, don't assume it.
2. **"Two correct components can still interact incorrectly."**
   Integration bugs live specifically at the seams between
   independently-tested pieces — audit those seams deliberately, don't
   assume passing isolated tests implies a passing integrated system
   (this is Module 5's entire argument, now applied at the whole-system
   scale).
3. **"Documentation is part of the deliverable, not an afterthought
   after the 'real' work."** ADRs, a runbook, and an API reference each
   serve a genuinely different reader; skipping any one leaves a real
   gap for whoever inherits this system next, including future-you.
4. **"A mature capstone names its own limitations explicitly."** Stating
   what the system doesn't yet do, and which future module addresses
   each gap, is stronger engineering communication than presenting the
   system as complete.

---

## 8. Visual Explanation

**Diagram 1 — The actual runtime request pipeline (execution order)**

```
[input guardrail]
      │
[session-boundary check] ──(if boundary)──► [prior-session memory write]
      │
[long-term memory retrieval]
      │
[context budget allocation]  ← knows about retrieved memory from above
      │
[prompt rendering]
      │
[routing decision]
      │
[cache check]  ← keyed on the ROUTED tier + BUDGETED, rendered prefix
      │
   (miss)
      │
[model call: structured output / tool-calling loop]
      │
[action guardrail]  ← before any tool actually executes
      │
[hallucination-reduction checks]  ← grounding/citation verification
      │
[output guardrail]
      │
[clarification/repair policy]
      │
[session-boundary-gated memory write, if applicable this turn]
      │
      ▼
  response to user
```

**Diagram 2 — Cross-component seams requiring explicit integration tests**

```
Module 7 (context budget) ──seam──► Module 8 (caching)
      "does the cache key reflect the POST-budget rendered prefix?"

Module 9 (routing) ──seam──► Module 10 (hallucination reduction)
      "is hallucination rate measured PER ROUTING TIER, not just aggregate?"

Module 11 (session boundary) ──seam──► Module 4 (memory retrieval)
      "does the NEXT session's retrieval wait for the PRIOR session's
       write to actually complete?"

Module 6 (guardrails) ──seam──► Module 9 (streaming)
      "does the output guardrail see the COMPLETE response before
       anything is streamed to the user?"
```

**Diagram 3 — Three documentation artifacts, three readers**

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│    ADRs      │    │   Runbook    │    │ API Reference│
│  (WHY it's   │    │ (HOW to      │    │  (WHAT it     │
│  built this   │    │  operate it  │    │  exposes to   │
│  way)         │    │  in prod)    │    │  build on)     │
├─────────────┤    ├─────────────┤    ├─────────────┤
│ reader: future│    │ reader: on-  │    │ reader:       │
│ engineer /    │    │ call /       │    │ someone       │
│ future-you    │    │ operator     │    │ building      │
│ maintaining   │    │ responding   │    │ against this  │
│ the system    │    │ to an alert  │    │ package       │
└─────────────┘    └─────────────┘    └─────────────┘
```

---

## 9. Recommended Resources

1. **Your own Part 1, Module 1 ADR practice and Part 1, Module 8 CI/CD
   design** — the primary resources for this module are genuinely your
   own prior work; review both before writing this capstone's ADRs and
   CI pipeline, since the goal is consistent extension, not a new
   documentation style.
2. **A well-regarded published system-design writeup of a real,
   production LLM-powered product** (search for engineering blog posts
   from companies that have published detailed architecture write-ups
   of their AI features) — read one closely for how they structure
   their own architecture documentation and named limitations, as a
   calibration point for your own.
3. **Google's "Site Reliability Engineering" book, runbook/playbook
   chapter** (or equivalent freely available material on writing
   operational runbooks) — read specifically for runbook structure and
   the "what to check first" pattern, which transfers directly to
   Section 6.3's runbook requirement.
4. **Your own accumulated `eval-framework`, `guardrail-bench`,
   `routing-bench`, `hallucination-eval-suite`, and
   `conversation-repair-sim` codebases** — the actual acceptance-test
   foundation for this capstone is your own Part 2 and Part 3 work in
   its entirety; there is no external resource that substitutes for
   reviewing all of it together before beginning integration.

---

## 10. Exercises

1. **Trace the real execution order against Section 6.1's diagram.**
   For your actual `llm-client-core` implementation, trace a single real
   request through every stage and confirm the true execution order
   matches Section 6.1 exactly — note and fix any place your
   implementation diverges (e.g., if caching currently happens before
   routing, this is a real bug to fix, not a stylistic choice).
2. **Write and run all four cross-component seam tests from Section
   6.2.** For each, construct a specific test case designed to fail if
   the seam is handled incorrectly, and confirm your system passes all
   four — fixing any that don't.
3. **Write a complete ADR for one major decision** (your choice — e.g.,
   the hybrid memory-compaction strategy, or escalation-based routing),
   following Part 1, Module 1's ADR format, including alternatives
   genuinely considered and why they were rejected.
4. **Write the runbook entry for one `observability-stack` metric
   family** (your choice — e.g., hallucination rate) specifying: what a
   healthy range looks like, what a concerning trend looks like, and the
   first three things you'd check if it degraded, referencing the
   specific debugging-exercise patterns from the module where that
   metric was introduced.
5. **Write the architecture-review section explicitly naming all five
   limitations from Section 6.4**, in your own words, each with a
   one-sentence explanation of which future Part addresses it and how.

---

## 11. Mini-Project: `pipeline-trace-visualizer`

A small tool that takes a `trace-recorder` (Module 5) output for a full
request and renders it as a readable, stage-by-stage visualization
matching Section 6.1's 13-step pipeline diagram — showing exactly which
stages fired, what each decided, and timing per stage. This is both a
genuinely useful debugging tool for the integration work in this module
and a strong artifact to include directly in your portfolio/interview
walkthroughs, since it makes the finished system's internals legible at
a glance rather than requiring a verbal explanation.

---

## 12. Production Project: `llm-client-core` v1.0 — Full Integration, Full Documentation

### 12.1 What you're building

This capstone's production project *is* the full integration and
delivery of everything built across Part 3:

1. **Reordered, verified execution pipeline** matching Section 6.1
   exactly, with the four cross-component seam tests from Section 6.2 as
   permanent regression tests in pipeline CI.

2. **A complete acceptance-test suite**, assembling every evaluation
   tool built across Part 3 into one coordinated CI run:
   `eval-framework`'s pipeline-trace scoring (Module 5), `guardrail-
   bench` (Module 6), `cache-hit-bench` (Module 8), `routing-bench`
   (Module 9), `hallucination-eval-suite` (Module 10), and multi-turn
   conversation evaluation (Module 11) — all run together as a single
   "is this system release-ready" gate, not as separate, disconnected
   test suites.

3. **Full documentation**: ADRs for every major cross-cutting decision
   made across Modules 1-11 (at minimum: memory compaction strategy,
   routing architecture, caching layer design, guardrail placement
   philosophy, hallucination-reduction technique selection), a complete
   runbook covering every `observability-stack` metric family
   introduced across Part 3, and an API reference documenting
   `llm-client-core`'s full public interface as it now stands.

4. **A written architecture review**, explicitly naming the five
   limitations from Section 6.4 (no RAG, no autonomous planning, no
   self-hosted serving, no dedicated frontend, foundational-not-
   comprehensive deployment maturity) and the specific future Part
   addressing each.

5. **`pipeline-trace-visualizer`**, integrated as both a debugging tool
   and a portfolio-ready artifact demonstrating the finished system's
   internals.

6. **Versioned release**: tag `llm-client-core` as v1.0, with a
   changelog summarizing what was added across all of Part 3's twelve
   modules — this is the artifact Part 4 onward will depend on as a
   stable, versioned foundation, exactly as promised when the package
   was first introduced back in Part 1, Module 2.

### 12.2 What this sets up for the rest of the handbook

- **Part 4 (RAG)** extends this exact package, adding retrieval over a
  large document corpus as a new grounding source (Module 10's
  grounding mechanism, generalized).
- **Part 5 (Agents)** extends the tool-calling and guardrail
  infrastructure (Modules 3 and 6) into autonomous, multi-step planning.
- **Part 6 (AI Infrastructure)** — Pankaj's platform-engineering
  track — adds self-hosted serving as an alternative or complement to
  the hosted-provider adapters built here.
- **Part 7 (Frontend for AI)** builds a real UI against this package's
  now-finalized API, extending `chat-shell` (Part 0, Module 8).
- **Part 8 (Production AI)** deepens the deployment/monitoring/DR story
  this module's runbook only foundationally covers.

### 12.3 Definition of done

- The full request pipeline executes in the exact order specified in
  Section 6.1, verified by a passing integration test tracing every
  stage.
- All four cross-component seam tests (Section 6.2) pass and are
  permanent CI regression tests.
- The complete acceptance-test suite (all six evaluation tools) runs as
  one coordinated CI gate and passes.
- ADRs, a runbook, and an API reference all exist, are complete, and
  would genuinely allow a new engineer to understand, operate, and build
  against the system without your direct explanation.
- The architecture review explicitly and honestly names all five system
  limitations with their corresponding future Part.
- `llm-client-core` is tagged v1.0 with a complete changelog.

---

## 13. Practice & Interview Questions

1. Walk through the complete request-handling pipeline for this system,
   in the correct execution order, and explain why at least two of the
   ordering decisions (not arbitrary sequence, but genuine dependencies)
   matter.
2. Describe a specific integration bug that could occur between two
   individually-correct components in this system, and how you'd design
   a test to catch it.
3. What are the three distinct documentation artifacts a system of this
   complexity needs, and what would go wrong operationally if you
   skipped the runbook specifically?
4. Present this system's architecture as if in a design-review
   interview: what does it do, and — just as importantly — what does it
   explicitly not yet do, and why is naming the gaps a strength rather
   than a weakness in this presentation?
5. If you had to cut this capstone down to the three most important
   acceptance tests to run before any production release, which three
   would you choose from across Part 3's tooling, and why?
6. How does this capstone's architecture directly set up Part 4's RAG
   work, specifically in terms of which existing mechanism gets
   generalized?

---

## 14. Common Mistakes

- **Assuming eleven passing module-level test suites imply a passing
  integrated system.** This is precisely the compounding-error argument
  from Module 5, now applying to the whole-system integration itself,
  not just within a single pipeline execution.
- **Skipping or shortchanging documentation** because it feels like
  "not real engineering work" compared to the components themselves —
  for a system this complex, documentation is part of the deliverable,
  not optional polish.
- **Presenting the system as finished/complete** rather than honestly
  scoped, which both overstates the system's actual maturity and misses
  a genuine opportunity to demonstrate engineering judgment about scope.
- **Treating the capstone as "just wire everything together"** without
  actually auditing the cross-component seams in Section 6.2 — these
  are real, specific integration risks, not a formality.
- **Building a new, twelfth test suite from scratch** instead of
  assembling and coordinating the six evaluation tools already built
  across Part 3 into one acceptance gate.

---

## 15. Debugging Exercise

During capstone integration testing, you discover a specific, narrow
failure: for conversations that (a) got routed to the cheaper model tier
(Module 9) *and* (b) triggered a session-boundary memory write (Module
11) *and* (c) the next session's opening message depends on that
just-written memory — the retrieved memory is sometimes stale or
missing, but only in this specific three-way combination; each pair of
these conditions tested independently (routing + memory write; memory
write + retrieval; routing + retrieval) works fine.

Write down at least two concrete hypotheses for why a three-way
interaction could fail when each pairwise interaction succeeds
(consider: is there a genuine race condition where the memory write,
processed asynchronously via `job-processor`, only manifests a timing
problem when the *specific* combination of a faster (cheap-tier) request
cycle and a real inter-session gap occurs — i.e., does routing tier
affect the timing window in a way that only becomes visible combined
with the async write path? could this be an instance of Module 5's
"pairwise-passing doesn't imply three-way-passing" compounding-error
argument, applied one level higher than pipeline stages — to
cross-cutting system properties instead?), and describe concretely how
you'd extend your acceptance-test suite to cover this specific
three-way combination going forward, rather than only pairwise
combinations.

---

## 16. Checklist

- [ ] I can trace my system's actual request-handling pipeline and
      confirm it matches the correct execution order from Section 6.1.
- [ ] I have identified and tested all four cross-component seams from
      Section 6.2, fixing any that failed.
- [ ] I can explain why a documentation artifact serving three distinct
      readers (ADRs, runbook, API reference) is necessary, not
      redundant.
- [ ] I can honestly name at least five specific limitations of the
      finished system and the future Part addressing each.
- [ ] `pipeline-trace-visualizer` correctly renders a full request trace
      matching the 13-step pipeline.
- [ ] The full acceptance-test suite (all six evaluation tools) runs as
      one coordinated CI gate.
- [ ] ADRs, a runbook, and an API reference are all complete and would
      genuinely support a new engineer without your direct explanation.
- [ ] The written architecture review honestly scopes the system's
      current limitations.
- [ ] `llm-client-core` is tagged v1.0 with a complete changelog covering
      all twelve Part 3 modules.

---

## 17. Summary

This capstone assembled eleven independently-built, independently-
evaluated components — real provider adapters, structured output, tool
calling with a live MCP connection, short- and long-term memory,
pipeline-level evaluation, layered guardrails, context budget
management, three-layer caching, streaming/batching/routing,
hallucination reduction, and conversation-level policy — into one
finished, integration-tested, fully-documented, honestly-scoped
production conversational AI service. The real engineering work here was
not building anything new, but auditing the execution order and
cross-component seams that isolated module-level testing structurally
cannot catch, exactly the compounding-error discipline Module 5
established, now applied at whole-system scale. `llm-client-core` ships
as v1.0: a stable, versioned, documented foundation — with its genuine
limitations named explicitly rather than hidden — that Parts 4 through 8
will build on directly for RAG, agents, infrastructure, frontend, and
production deployment, exactly as promised when this package was first
introduced in Part 1, Module 2.

---

## 18. Next Steps

**Part 3 is complete.** Next: **Part 4 — Retrieval-Augmented Generation
(RAG)**, beginning with chunking strategies and embedding-model
selection for a real document corpus — the direct generalization of
this capstone's grounding mechanism (Module 10) from single tool results
and memory items into full-scale retrieval over a large, external
knowledge base, extending `contextual-embedding-service` and
`multimodal-rag-preview` from Part 2, Module 10, and `job-processor`
(Part 1, Module 6) for batch embedding generation.

Reply "continue" for Part 4, Module 1, or flag anything to go deeper on.
