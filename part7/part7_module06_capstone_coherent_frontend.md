# Part 7, Module 6 — Capstone: One Coherent Frontend for the Whole Stack

## 1. Learning Objectives

By the end of this module you will be able to:

1. Trace the actual runtime relationship between `chat-shell`, `ops-console`, and `voice-shell` — three separately-deployed applications built across Modules 1–5 — and identify where they genuinely share infrastructure versus where they only *appear* to, because sharing a component library is not the same as sharing a runtime.
2. Audit the seams between the five independent reducers built across this part (`tokenReducer`, `traceReducer`, `citationReducer`, `turnReducer`, plus `ops-console`'s telemetry reducers) and identify where an assumption that held in isolation breaks when two of them run against the same live session simultaneously.
3. Assemble every stress/regression test built across Part 7 (`streaming-render-stress-test`, `agent-trace-render-stress-test`, `citation-access-control-test`, the synthetic-page drill, `voice-interaction-stress-test`) into one tiered, CI-blocking `frontend-ci-gate`, following the same tiered-severity discipline established by `agent-ci-gate` (Part 5), `rag-ci-gate` (Part 4), and `ai-infra-readiness-gate` (Part 6).
4. Produce full architecture documentation for the assembled frontend surface: an ADR log, a cross-application data-flow diagram, and a "what a new engineer needs to know before touching any of the three shells" onboarding doc.
5. Conduct an honest limitations review — name what Part 7 has deliberately left unaddressed, and map each limitation to the specific future Part or module that will close it, in the same spirit as every prior capstone's limitations section.
6. Make and defend one final, previously-deferred architectural decision: whether `chat-shell`, `ops-console`, and `voice-shell` should share a single deployment/build pipeline going forward, or remain genuinely separate — with a real trade-off analysis, not a default.

## 2. Prerequisites

- Part 7, Modules 1–5, all of them — this capstone is a direct assembly and audit of everything built across the part, not new independent content.
- Part 5, Module 6 and Part 6, Module 8 (the two most recent capstones) — re-read their audited-seams sections immediately before starting; this module follows their exact methodology, applied to the frontend layer.

## 3. Estimated Study Time

12–16 hours (runtime tracing and seam audit: ~4 hours; test assembly: ~3 hours; documentation: ~3 hours; final architectural decision and write-up: ~2–3 hours).

## 4. Difficulty

★★★★☆ (4/5) — Individually, nothing here is harder than what Modules 1–5 already built. The difficulty is entirely in the audit: finding the places where three independently-reasonable designs interact badly, which requires holding all five modules' assumptions in your head at once.

## 5. Why This Matters

Every capstone so far in this handbook has made the same discovery: components that are each individually correct can still combine into a system with real gaps at the seams, and those gaps are invisible until you deliberately go looking for them with the *system* in mind rather than any one component. Part 5's capstone found that reflection needs `resolved` tracked distinctly from `satisfied`, or memory writes could be fooled — a bug invisible to Module 2 in isolation, only visible once Module 3's memory system had to interact with Module 2's reflection output. Part 6's capstone found that fleet retry and cross-provider fallback needed to be tested independently, not because either was individually broken, but because their interaction masked failure. Part 7 has now built five modules' worth of frontend surface, each individually solid — this capstone is where you find out what actually happens when a `requires_approval` action is pending in `chat-shell` while an operator is simultaneously watching `ops-console`'s live provenance stream for the same run, or when a user starts a session in `voice-shell` and continues it in `chat-shell` mid-conversation.

There's also a genuinely new kind of question this capstone raises that none of the underlying AI-engineering parts needed to ask: three separate applications sharing one component/event architecture is a real product-organization decision, not just a technical one. Getting this right is exactly the kind of judgment that separates a demo from something a team could actually operate and extend — which is the whole point of every "production project" this handbook has built toward.

## 6. Theory

### 6.1 Tracing the actual runtime relationship between three shells

Before auditing seams, get precise about what's actually shared versus merely similar:

**Genuinely shared:**
- The `AgentEvent`/`StreamEvent` contract (Module 2, extended in Modules 3 and 5) — one versioned schema, all three shells decode it the same way.
- The reducer *pattern* (structurally independent reducers per data source, never rendering unconfirmed state) — a design discipline, reused, not a shared runtime.
- Component implementations where explicitly reused (`<ToolCallCard>` in both `chat-shell` and `voice-shell`'s escalation path, per Module 5, §6.5).

**Deliberately NOT shared:**
- Deployment (each shell deploys independently — `ops-console` explicitly so it survives `ai-api-platform` outages, per Module 4, §6.1).
- Authentication/authorization boundaries (`chat-shell`'s end-user auth vs. `ops-console`'s operator auth, per Module 4).
- Runtime process (a user's `chat-shell` session and an operator's `ops-console` session watching the same `run_id` are two separate client connections to the same backend event stream, not two views into one shared client state).

This distinction matters because the seam audit in §6.2 specifically targets the *shared* surfaces (the event contract, the reducer assumptions) — that's where a bug in one shell can propagate to another. The deliberately-unshared surfaces are, by design, isolated from each other's failures; confirming that isolation actually holds is itself one of the audit items.

### 6.2 Five audited seams

Following the exact methodology of Part 5 and Part 6's capstones — trace the full cross-module runtime interaction, then name the seams that only become visible once every piece runs together:

**Seam 1 — Concurrent viewers of one `run_id` (chat-shell + ops-console).** An end user's `chat-shell` session and an operator's `ops-console` session can both be subscribed to the same `run_id`'s event stream simultaneously — the end user sees the reduced trace and citation view (Modules 2–3), the operator sees the raw provenance stream (Module 4). Each was built and tested against a single-viewer assumption. The seam: does the backend's event stream support multiple concurrent subscribers per `run_id` without one subscriber's reconnect/resume logic (Module 2, §6.3's reconnection requirement) affecting the other's? This must be tested explicitly — a naive single-subscriber assumption in the event-delivery layer would mean an operator's dashboard refresh could inadvertently disrupt the end user's live session, or vice versa.

**Seam 2 — A voice session handed off to a text session mid-conversation.** `voice-shell` and `chat-shell` both implement the tool-call lifecycle (Module 2's `traceReducer`), but `voice-shell` adds `turnReducer` state and a distinct confirmation grammar (Module 5) on top. If a user starts a `requires_approval` interaction by voice and then, mid-confirmation, switches to typing in `chat-shell` for the same session (a real scenario — someone starts hands-free, then picks up their phone), does the pending approval state transfer correctly? The seam: `approval_pending` state must live entirely in the backend's execution layer (as Part 5, Module 1 always specified), never assumed to be tracked correctly only within whichever client happened to initiate it — a design that was implicitly correct since Module 2 but never actually tested against a cross-shell handoff until now.

**Seam 3 — `ops-console`'s independent data path, actually exercised during a real `chat-shell`/`voice-shell` degradation.** Module 4 designed `ops-console` to read telemetry through a path independent of `ai-api-platform`'s primary serving path specifically so it survives outages of the systems it monitors. This was a design decision, not yet a proven one — the seam: has anyone actually tested `ops-console` staying live while `chat-shell` and `voice-shell` are both artificially degraded, the direct frontend analog of Part 6, Module 8's fleet-retry-vs-fallback independence test? An untested independence claim is exactly the kind of thing that looks fine until the one incident where it's needed.

**Seam 4 — Citation and provenance rendering consistency across shells for the same underlying event.** A single `provenance` event (Module 2) is rendered as a reduced two-state signal in `chat-shell`, as a raw tag in `ops-console` (Module 4), and — per Module 5 — potentially escalated to a full visual card in `voice-shell`. All three renderings must trace back to exactly one underlying event and classification, never three independently-computed interpretations that could drift out of sync if, say, `chat-shell`'s reduction logic and `ops-console`'s "what counts as a pattern-worthy block" logic (Module 4, Exercise 5) are implemented as separate, divergent client-side computations instead of both consuming one already-classified server-side signal. The seam: confirm all three shells consume the same server-computed classification rather than each doing their own reduction of raw data — otherwise a fix to one shell's display logic can silently create a discrepancy with another's.

**Seam 5 — The `frontend-ci-gate`'s tests must actually exercise cross-shell scenarios, not just each shell in isolation.** Every stress test built in Modules 1–5 was written and validated against its own shell. Assembling them into one CI gate (§6.3) is not automatically the same as testing the seams above — a CI gate that runs five independent test suites in parallel and calls it done has not actually tested Seams 1–4, only re-confirmed each shell still works alone. This is the frontend version of exactly the mistake Part 6, Module 8 flagged: assembling existing tools into a gate is necessary but not sufficient; the gate needs dedicated cross-component tests, not just a union of pre-existing ones.

### 6.3 Assembling the `frontend-ci-gate`

Following the same tiered-severity model as `agent-ci-gate`, `rag-ci-gate`, and `ai-infra-readiness-gate`:

```
Tier 1 (hard blocker — never merge if failing):
  - citation-access-control-test (Module 3)          — security
  - voice-interaction-stress-test: no-execute-before-approval case (Module 5) — safety
  - Seam 2 cross-shell approval-handoff test (new, this module)             — safety

Tier 2 (must pass before deploy, can be waived with explicit sign-off):
  - streaming-render-stress-test (Module 1)
  - agent-trace-render-stress-test (Module 2)
  - Seam 1 concurrent-viewer test (new, this module)
  - Seam 4 cross-shell rendering-consistency test (new, this module)

Tier 3 (monitored, alert on regression, doesn't block merge):
  - synthetic-SLO-breach-page drill (Module 4)
  - Seam 3 ops-console-independence drill (new, this module)
```

Notice the tier placement mirrors the underlying stakes exactly as every prior gate has: anything touching security (access control) or a structural safety guarantee (no execution before approval) is Tier 1; rendering-correctness and consistency issues that are important but not safety-critical are Tier 2; operational-resilience drills that need periodic real exercise more than every-commit blocking are Tier 3 — the same reasoning Part 6, Module 8 used for its own gate's tiers.

### 6.4 The deferred architectural decision — one deployment pipeline, or three?

Module 4 deliberately built `ops-console` as a separate deployment specifically so it survives outages of what it monitors. That reasoning doesn't automatically extend to whether `chat-shell` and `voice-shell` — which don't have `ops-console`'s "must survive the outage of what it depends on" requirement, since they *are* the primary user-facing surface, not a diagnostic tool for it — should share a deployment pipeline with each other.

The honest trade-off:

- **Shared pipeline for `chat-shell` + `voice-shell`**: simpler versioning (the `AgentEvent` contract, shared components, and reducer patterns stay in lockstep automatically), one release process to reason about, lower operational overhead for a small team.
- **Separate pipelines**: independent release cadence (a voice-specific bug fix doesn't need to wait for or risk the text shell's release train), and — the sharper argument — different risk profiles: `voice-shell`'s real-time audio constraints (Module 5) make it more sensitive to certain classes of regression (a barge-in cancellation timing bug) that `chat-shell` would never surface, so bundling their releases means `chat-shell` changes could accidentally introduce risk to `voice-shell`'s much less forgiving margin, or vice versa.

This handbook's recommendation, stated as a recommendation rather than a rule (this is a genuine judgment call that depends on team size and release cadence in practice): keep `chat-shell` and `voice-shell` on **separate deployment pipelines**, sharing the component library and event contract as versioned dependencies rather than a monorepo-deploys-together setup — mirroring exactly the reasoning already applied to `ops-console` in Module 4, now extended for a different but analogous reason (different risk/regression profile rather than different-outage-survival requirement). `ops-console` remains separate for its own, already-established reason. The unifying principle across all three: **shared architecture, independent blast radius** — the same phrase used, in different words, every time this handbook has separated a shared library from a deployed service (`llm-client-core` vs. its adapters' individual deployments, `rag-engine`'s single public interface vs. its internal components).

## 7. Mental Models

- **"Sharing a component library is not sharing a runtime — audit the seams between deployments, not just between files."**
- **"An untested independence claim is a hope, not a guarantee"** — Seam 3's `ops-console` survival story needed an actual drill, the same way every prior capstone's claims needed one.
- **"Assembling existing tests into a gate finds nothing new by itself — the seams need their own tests, written for the seam, not inherited from either side of it."**
- **"Shared architecture, independent blast radius"** — the one-line summary of every deployment-separation decision this handbook has made, now applied a third time to the frontend layer.

## 8. Visual Explanation

**What's shared vs. deployed independently across the three shells:**

```
┌─────────────── shared (versioned dependency) ───────────────┐
│  AgentEvent/StreamEvent contract · reducer pattern discipline │
│  <ToolCallCard>, <CitationText> (reused where applicable)     │
└───────────────────────────────────────────────────────────────┘
         │                    │                     │
         ▼                    ▼                     ▼
   ┌───────────┐        ┌───────────┐         ┌────────────┐
   │ chat-shell│        │voice-shell│         │ops-console │
   │(own deploy)│        │(own deploy)│         │(own deploy,│
   │ end-user  │        │ end-user  │         │ independent│
   │   auth    │        │   auth    │         │data path,  │
   │           │        │           │         │operator auth)│
   └───────────┘        └───────────┘         └────────────┘
         └──────────┬──────────┘                     │
                     ▼                                ▼
            same backend run_id event stream, multiple concurrent
            subscribers (Seam 1) — proven independent, not assumed
```

**The five audited seams, at a glance:**

```
Seam 1: chat-shell + ops-console, same run_id, concurrent           → subscriber isolation
Seam 2: voice-shell → chat-shell, mid-approval handoff              → backend-owned state
Seam 3: ops-console alive during chat-shell/voice-shell degradation → path independence
Seam 4: one provenance event, three renderings                     → one server-side source
Seam 5: assembled gate must test seams, not just union pre-existing suites
```

## 9. Recommended Resources

1. **Part 5, Module 6 and Part 6, Module 8 (this handbook)** — re-read both capstones' seam-audit sections back to back immediately before starting this module's audit; the methodology is deliberately identical and worth having freshly in mind.
2. **Martin Fowler — "Micro Frontends"** (martinfowler.com) — the closest published framing for the "shared architecture, independent deployment" pattern this module formalizes for `chat-shell`/`voice-shell`/`ops-console`; useful for calibrating the deployment-separation trade-off in §6.4 against established practice rather than reasoning from scratch.
3. **Google SRE Book — "Testing for Reliability" (ch. 17)** — directly relevant to §6.2's Seam 3 audit: the chapter's argument for testing failure independence claims with real drills, not code review, rather than assuming a design intention was actually achieved.
4. **Part 1, Module 13 and Part 6, Module 5 (System Design Fundamentals; Load Balancing)** — worth a brief re-read for the general pattern of auditing seams between independently-correct components under concurrent load, which Seam 1 is a frontend instance of.

## 10. Exercises

1. Design the concrete test for Seam 1: two simulated clients (one `chat-shell`-shaped, one `ops-console`-shaped) subscribed to the same `run_id`, with one deliberately disconnecting and reconnecting mid-session. What specifically should you assert about the other client's state during that disruption?
2. Write out the exact backend state transition Seam 2 depends on: where, precisely, does `approval_pending` state live, and what does a client (any client, `chat-shell` or `voice-shell`) actually do when it connects mid-approval — fetch current state, or assume its own local state is authoritative? Justify against Module 2, §6.3's original resumption requirement.
3. Design the Seam 3 drill: what does "artificially degrade `chat-shell`/`voice-shell`'s serving path" concretely mean in a test environment, and what specific assertion proves `ops-console` remained usable throughout, not just that it didn't crash?
4. Take Seam 4 and trace one `provenance` event through all three shells' rendering logic. Where exactly does each shell's rendering diverge from a shared, server-computed classification (if anywhere) versus where does each shell correctly just render a pre-classified signal differently for its audience?
5. Argue the *other* side of §6.4's recommendation: make the best case for a single shared deployment pipeline across all three shells, and identify the specific condition (team size, release cadence, or something else) under which that argument would actually win.

## 11. Mini-Project

Write the Seam 1 concurrent-viewer test described in Exercise 1, against a minimal mocked event-stream server (a small local script simulating one `run_id`'s event stream with multiple subscriber connections). Confirm a disconnect/reconnect from one simulated client never drops, duplicates, or delays events delivered to the other simulated client. This isolates the specific seam most likely to hide a real bug, before assembling the full gate in the Production Project.

## 12. Production Project: `frontend-ci-gate` v1 and the Cross-Shell Architecture Doc

Assemble and document the complete Part 7 frontend surface.

**Scope:**

1. **Five new cross-shell seam tests** (§6.2, Exercises 1–4), each written specifically to exercise the interaction the corresponding seam describes — not inherited or repurposed from any single shell's existing test suite.
2. **`frontend-ci-gate`**, tiered per §6.3, assembling all of Modules 1–5's individual stress/regression tests plus the five new seam tests, with tier placement matching the stakes reasoning already established by every prior gate in the handbook.
3. **Seam 3 independence drill**, run as an actual scheduled/periodic exercise (not just a one-time proof) — following Part 6, Module 8's own lesson that an untested independence claim decays into a false assumption over time if never re-exercised.
4. **Cross-application data-flow diagram**, showing precisely what's shared (contract, patterns, specific reused components) versus independently deployed (per §8's diagram), as a living architecture document.
5. **ADR log** consolidating every architectural decision made across Part 7 (the `AgentEvent`/`TraceStep` separation from Module 2, the claim-tiering design from Module 3, the separate-app/separate-auth decision from Module 4, the stakes-classifier and visual-escalation decision from Module 5, and this module's deployment-pipeline decision from §6.4) into one indexed, cross-referenced document.
6. **Onboarding doc**: "what a new engineer needs to know before touching any of the three shells" — covering the shared event contract, the reducer discipline, which auth boundary applies where, and a pointer to each of the five audited seams so a new engineer doesn't have to rediscover them by causing the same bug.
7. **Honest limitations review** (§13): explicitly named, each mapped to the specific future part/module that addresses it.

**Documentation**: this module's entire deliverable *is* documentation and testing infrastructure — no new user-facing feature is being built, matching the shape of every prior capstone in this handbook, where the capstone's job was integration and honesty about what remains, not new functionality.

**Hands off to:** Part 8 (Production AI), which extends Part 6, Module 8's failure-injection discipline — and now, transitively, this module's Seam 3 drill methodology — to the full application stack rather than just the GPU/infra layer; and Part 11's capstone projects, which will be the first place all three shells (`chat-shell`, `ops-console`, `voice-shell`) plus every backend system from Parts 3–6 are exercised together in a single end-to-end product build.

## 13. Honest Limitations (named explicitly, mapped to future work)

1. **No automated visual-regression testing across the three shells.** `frontend-ci-gate` catches logical/state-machine regressions but not visual/CSS drift. Reasonable to defer — Part 8's broader CI/CD hardening is the natural place to add this, once the application stack's full deployment pipeline (not just the frontend) is being formalized.
2. **The Seam 3 independence drill is currently a manually-scheduled exercise, not yet a fully automated, continuously-re-verified one.** Directly analogous to the gap Part 6, Module 8 named for its own drills — Part 8's failure-injection-discipline extension is where this gets formalized handbook-wide, across both infra and frontend.
3. **No load testing of concurrent-viewer scenarios (Seam 1) at realistic operator/end-user ratios.** The seam test proves correctness for a small number of simulated concurrent subscribers; it hasn't been proven at real production concurrency. Reasonable to defer to Part 8, where realistic load profiles for the full stack are established.
4. **The single-shared-pipeline-vs-three-pipelines decision (§6.4) was made as a recommendation based on general trade-off reasoning, not validated against this specific team's actual release cadence or incident history**, because neither exists yet at this stage of the handbook. Revisit once Part 9/10's freelancing and interview-prep context, or a real deployed project, gives an actual cadence to reason from.
5. **No internationalization/localization considered anywhere in Part 7** — every UI text, confirmation phrase, and voice grammar (Module 5) has been designed and tested in English only. This is a real, named gap, not an oversight to gloss over, and would need dedicated design work (voice confirmation grammars especially don't translate directly) before any real multi-language deployment.

## 14. Practice & Interview Questions

1. Explain the difference between "these three applications share a component library" and "these three applications share a runtime," and why conflating the two is a common architectural mistake.
2. Walk through Seam 2 (voice-to-text handoff mid-approval) end to end. Why does this seam only become visible once both `voice-shell` and `chat-shell` exist, even though each shell's approval logic was independently correct?
3. Why is assembling five pre-existing test suites into one CI gate insufficient to claim the seams between components are tested, and what does an actual seam test need to look like instead?
4. Make the case for keeping `ops-console` on a separate deployment pipeline from `chat-shell`/`voice-shell`, and then make the (different) case for why `chat-shell` and `voice-shell` might reasonably be kept separate from *each other* too, given they don't share `ops-console`'s specific "must survive what it monitors" requirement.
5. In a system design interview: you're asked to design three related client applications (e.g., a customer app, an internal ops tool, and a voice assistant) that share significant backend infrastructure. Walk through how you'd decide what to share as a library versus what to keep genuinely separate as deployments, using this module's "shared architecture, independent blast radius" principle.
6. Why is an untested claim that one system is "independent" of another's failures worth treating as false until proven, rather than assumed true because the architecture was designed with that intention?

## 15. Common Mistakes

- **Assuming shared component code means shared runtime behavior.** The seams in §6.2 exist precisely because three deployments running the same reducer *pattern* can still disagree in practice about shared backend state.
- **Calling a CI gate "assembled" once every individual shell's tests pass in it.** Without dedicated seam tests, this only reconfirms each shell works alone — exactly Seam 5's warning.
- **Treating `ops-console`'s independence design as self-evidently achieved** rather than something that needs an actual, periodically-repeated drill to keep believing.
- **Letting each shell compute its own reduction/classification of a shared event** (Seam 4) instead of consuming one server-computed classification — an easy way for three renderings of "the same fact" to quietly drift out of agreement over time as each shell's display logic evolves independently.
- **Defaulting to a single shared deployment pipeline for convenience without weighing the different regression/risk profiles** each shell actually has (§6.4) — convenience is a real factor, but it should be an explicit trade-off, not an unexamined default.
- **Skipping the honest limitations section, or writing it as a formality**, rather than genuinely naming what's untested and where it gets addressed — every prior capstone's credibility rested on this section being real.

## 16. Checklist

- [ ] All five seams (§6.2) have dedicated, purpose-built tests — not repurposed single-shell tests
- [ ] `frontend-ci-gate` assembled with tiers matching the stakes reasoning of `agent-ci-gate`/`rag-ci-gate`/`ai-infra-readiness-gate`
- [ ] Seam 3 independence drill scheduled as a recurring exercise, not a one-time proof
- [ ] Cross-application data-flow diagram distinguishes shared architecture from independent deployment precisely
- [ ] Full ADR log consolidated and cross-referenced across all of Part 7's modules
- [ ] Onboarding doc written, pointing a new engineer to all five audited seams explicitly
- [ ] Honest limitations section written with real specificity, each mapped to a named future part/module
- [ ] Deployment-pipeline decision (§6.4) documented as a reasoned recommendation, not an unexamined default
- [ ] Every shell verified to consume one server-computed event classification rather than independently re-deriving it (Seam 4)

## 17. Summary

Part 7 built three genuinely different frontends — a text chat shell, an operator dashboard, and a voice interface — each earning its own module because each had a real, distinct constraint the others didn't: `chat-shell`'s job was rendering unconfirmed state safely; `ops-console`'s was serving a different audience with a different trust model and surviving the outages it monitors; `voice-shell`'s was surviving a latency and interruption budget none of the others had to answer for. This capstone's job wasn't to build a fourth thing — it was to prove, the same way every capstone before it has, that three individually-correct designs don't automatically compose correctly, and to find the five seams where they didn't until audited directly. The result is `frontend-ci-gate`, a single tiered gate that — like every gate before it in this handbook — encodes not just "does each piece work" but "do the pieces actually agree with each other under the conditions where agreement is hardest to fake." Part 7 closes having answered every honest limitation Parts 5 and 6 left open about their missing frontends, and having left its own honestly, for Part 8 to pick up.

## 18. Next Steps

Part 7 (Frontend for AI) is now complete. Reply "continue" to begin Part 8 (Production AI), or flag anything from Part 7 to go deeper on first.
