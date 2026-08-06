# Part 7, Module 4 — The Operator Dashboard: Surfacing Infra Health & SLOs

## 1. Learning Objectives

By the end of this module you will be able to:

1. Articulate why an operator dashboard is a structurally different product from the end-user `chat-shell` UI built in Modules 1–3, not a "more detailed" version of it, and design its information architecture accordingly.
2. Surface `ai-infra-stack`'s (Part 6, Module 8) phase-specific SLOs — prefill vs. decode latency, per Part 6 Module 8's own hidden-regression lesson — as a live dashboard rather than an aggregate number that hides exactly the regressions Part 6 built tooling to catch.
3. Render `gpu-fleet-gateway`'s (Part 6, Module 5) real-occupancy routing signal and circuit-breaker state live, distinguishing a normal fallback from a fleet-wide degradation event.
4. Build the raw-provenance operator view deferred in Module 2 — the audience-appropriate counterpart to the end-user reduced provenance signal from Module 2's §6.5.
5. Design an alerting/escalation surface that reuses `ai-infra-readiness-gate`'s (Part 6, Module 8) tiered severity model instead of inventing a new one, so a CI-time gate and a live dashboard alert mean the same thing when they fire.
6. Correctly scope authentication and authorization for an operator-only surface, distinct from `chat-shell`'s end-user auth (Part 1, Module 7), and justify why reusing the exact same auth boundary would be a mistake.
7. Extend the `chat-shell` monorepo with a second, separately-deployed application (`ops-console`) rather than bolting operator views onto the end-user shell, and articulate the trade-off that decision represents.

## 2. Prerequisites

- Part 7, Modules 1–3 — the multi-stream SSE/reducer architecture is reused here, applied to infra telemetry instead of chat/trace/citation events.
- Part 6, Module 1 (GPU Fundamentals) — prefill vs. decode, VRAM/KV-cache ceiling.
- Part 6, Module 5 (Load Balancing) — occupancy-based routing, GPU OOM as a distinct failure shape, the memory-pressure circuit breaker.
- Part 6, Module 8 (HA/DR capstone) — the 5 audited seams, especially the fleet-retry-vs-cross-provider-fallback independence requirement and the phase-specific SLO split; `ai-infra-readiness-gate`'s tiered severity model.
- Part 1, Module 7 (Authentication & Authorization) — you'll be designing a second, distinct auth boundary against this same foundation.

## 3. Estimated Study Time

10–13 hours (theory + exercises: ~3 hours; mini-project: ~2 hours; production project: ~6–8 hours).

## 4. Difficulty

★★★☆☆ (3.5/5) — No new hard algorithmic problem like Module 2's approval gate; the difficulty here is almost entirely in information architecture and correctly wiring a genuinely different audience's auth boundary — getting the *scope* right matters more than any single component's implementation.

## 5. Why This Matters

Part 6's capstone ended with an honest limitation: "no real operator dashboard UI." Everything Part 6 built — phase-specific SLOs, occupancy-based routing, the memory-pressure circuit breaker, the tiered `ai-infra-readiness-gate` — has been observable only through logs, Grafana panels assembled ad hoc, or CI output. That's a real gap, and it's not a cosmetic one: Part 6, Module 8's sharpest lesson was that fleet-internal retry and cross-provider fallback are two *independent* layers that must be tested independently, because a broken one can hide behind a working one until both fail together. An aggregate "system healthy" dashboard number hides exactly that failure mode — it's the UI equivalent of the aggregate-latency-SLO mistake Module 8 itself named and fixed with phase-specific splitting. If the dashboard you build here repeats that mistake at the UI layer, you've quietly undone the fix.

This module is also the first time in Part 7 the audience genuinely changes. Every prior module built for an end user who wants an answer and, at most, occasional visibility into how it was produced. An operator wants the opposite default: full visibility, rare need for a curated summary, and — critically — access to information (raw provenance tags, per-node occupancy, real cost figures) that would be actively wrong to show an end user. Getting this audience split right, deliberately, rather than by reusing `chat-shell`'s components and just adding more detail, is the actual skill this module teaches.

## 6. Theory

### 6.1 Why an operator dashboard is a different product, not a more detailed view

It's tempting to treat `ops-console` as `chat-shell` with an "advanced" toggle. Resist this for three concrete reasons, each already established elsewhere in the handbook and now converging:

- **Different trust model.** Module 2 designed the end-user trace UI around *reducing* what's shown, because a user's attention is scarce and most internal detail would erode trust or just be noise. An operator's job is the literal opposite: their attention is the reason they're looking at the dashboard at all, and hiding detail from them is a liability, not a courtesy.
- **Different auth boundary.** `chat-shell` end users are authenticated per Part 1, Module 7's identity model and should see only their own conversations/data. An operator needs cross-user visibility (fleet health affects every user at once) but should almost never need per-user conversation content. Reusing one auth system for both is the same category of mistake as Part 4, Module 5 warned against for retrieval access control: conflating two different authorization questions ("can this identity see this content" vs. "can this identity see this infrastructure state") into one boundary makes it easy to accidentally grant one via the other.
- **Different deploy/availability requirement.** An operator dashboard needs to stay usable *during* the incidents it's meant to help diagnose — including scenarios where `chat-shell`'s backend (`ai-api-platform`) is degraded. If `ops-console` is bolted onto the same deployment as `chat-shell`, a fleet-wide outage can take down the exact tool an operator needs to diagnose that outage. This is a direct echo of Part 6, Module 8's redundancy lesson applied to the dashboard itself: `ops-console` needs its own resilience story, ideally reading from infra telemetry through a path independent of the primary serving path it's monitoring.

Concretely: `ops-console` is built as a **separate application**, sharing `chat-shell`'s component library and reducer patterns (Modules 1–3's architecture is genuinely reusable — same discipline: never render unconfirmed state, structurally independent reducers per data source) but deployed independently, authenticated independently, and reading from telemetry sources (`ai-infra-stack`'s metrics, `ai-infra-readiness-gate`'s results) rather than from `ai-api-platform`'s primary request path.

### 6.2 Phase-specific SLOs — the dashboard's central discipline

Part 6, Module 8's sharpest fix was splitting latency SLOs by prefill/decode phase instead of reporting an aggregate, because aggregation hides exactly the regression that phase-specific measurement catches (a decode-phase KV-cache-bandwidth regression can be fully masked by an unrelated prefill improvement in the same window). A dashboard that reports one "p99 latency: 340ms" number at the top has thrown that fix away.

The operator dashboard's primary panel is therefore not a single number but a small matrix:

```
                    p50      p95      p99      SLO target   status
prefill latency     45ms     120ms    210ms    <250ms       ✓ within SLO
decode latency/tok  12ms     31ms     58ms     <40ms        ⚠ breaching p99
```

This is a direct, literal rendering of data `ai-infra-stack` already computes (Part 6, Module 8) — the module's contribution isn't new backend logic, it's refusing to compress it back into an aggregate the way a naive dashboard would, exactly the discipline Module 8 fought to establish in the first place.

### 6.3 Fleet state — distinguishing "working as designed" from "degraded"

`gpu-fleet-gateway` (Part 6, Module 5) routes by real occupancy and has a dedicated memory-pressure circuit breaker distinct from ordinary timeout-based breaking, because a GPU OOM event takes down every in-flight request on that instance at once — a different failure shape than a typical service timeout. A dashboard that shows fleet nodes as a flat "healthy/unhealthy" list loses this distinction entirely, and loses it at exactly the moment it matters most: mid-incident, when an operator needs to immediately tell "one node tripped its OOM breaker and traffic correctly rerouted" (working as designed, low urgency) apart from "occupancy is climbing across the fleet and there's nowhere left to reroute to" (a real capacity emergency).

```
node-gpu-03   ● circuit OPEN (memory pressure)     traffic rerouted    [expected — no action needed]
node-gpu-07   ● occupancy 94%, climbing            2 of 6 nodes >90%   [⚠ fleet capacity risk]
```

The second state is the one worth an alert; the first, correctly, is not — it's the system doing exactly what Part 6, Module 5 designed it to do. A dashboard that alerts on both equally trains operators to ignore alerts, which is worse than not alerting at all.

Similarly, Module 8's independence requirement (fleet-internal retry and cross-provider fallback must be tested and observed as genuinely separate layers) needs to be visible as two separate status rows, not folded into one "requests succeeding: yes/no" indicator — otherwise a live incident where the second layer is silently unavailable while the first still masks it is exactly invisible until both fail together, the precise scenario Module 8 built drills to catch in testing. The dashboard's job is to make that scenario visible in production, not just in CI.

### 6.4 Raw provenance — the operator's counterpart to Module 2's reduced signal

Module 2 deliberately deferred the raw-provenance view: end users get a reduced two-state trust signal (verified / unmarked), because a raw tag like `agent_derived` reads as jargon or an unwarranted disclaimer to someone who isn't auditing the system. An operator is exactly the audience for whom that reduction is the wrong choice — their job is auditing the system, and the four raw tags (`human_verified` / `externally_sourced` / `agent_derived` / `peer_agent_derived`) are precisely the granularity that lets them catch a poisoning attempt (Part 5, Module 3) or correlated-failure false corroboration (Module 4) before it propagates.

```
memory write   run_id: a91f...   tag: agent_derived        consolidation: BLOCKED (needs corroboration)
memory write   run_id: b204...   tag: human_verified        consolidation: promoted to semantic memory
memory write   run_id: c552...   tag: peer_agent_derived    consolidation: BLOCKED (same-model correlation risk)
```

This view reuses the exact `AgentEvent` `provenance` events already emitted since Module 2 — no new backend work, just a second, unreduced rendering of the same event stream, gated behind `ops-console`'s separate auth boundary rather than `chat-shell`'s.

### 6.5 Alerting reuses the CI gate's severity model — it doesn't invent a new one

`ai-infra-readiness-gate` (Part 6, Module 8) already has a tiered severity model for CI: some checks (like the access-control and poisoning/correlated-failure drills) are hard blockers, others are warnings. It's tempting to design a fresh, dashboard-specific alert-severity scheme. Don't — if a CI gate calls something a hard blocker and the live dashboard calls the equivalent live condition merely a warning, operators learn that the two systems disagree about what matters, which erodes trust in both. Reuse the same severity taxonomy, and where a live metric corresponds to a check that's a hard blocker in CI (e.g., the access-control security test), treat its live-degraded state as page-worthy, not just dashboard-red.

## 7. Mental Models

- **"An operator dashboard's job is to not hide what an end-user dashboard correctly hides."** Same underlying data, opposite default visibility.
- **"A circuit breaker tripping correctly is not an incident — it's evidence the incident was contained. Don't alert on the fix."**
- **"If CI calls it a hard blocker, the live dashboard calls it page-worthy — one severity model, not two that can quietly disagree."**
- **"The tool you need during an outage can't live inside the thing that's having the outage."**

## 8. Visual Explanation

**Two audiences, one event stream, two renderings:**

```
              AgentEvent: provenance { tag: "agent_derived", ... }
                                │
                ┌───────────────┴───────────────┐
                ▼                                ▼
        chat-shell (end user)             ops-console (operator)
        reduced: unmarked                 raw: "agent_derived — consolidation BLOCKED"
        (Module 2, §6.5)                  (this module, §6.4)
```

**Dashboard top-level layout:**

```
┌─ ops-console ──────────────────────────────────────────────┐
│ ┌ Latency (phase-split) ┐  ┌ Fleet state ┐  ┌ Fallback ┐    │
│ │ prefill: ✓ within SLO │  │ 6 nodes     │  │ layer 1: │    │
│ │ decode:  ⚠ p99 breach │  │ 1 breaker   │  │ ✓ live   │    │
│ └────────────────────────┘ │ open (exp.) │  │ layer 2: │    │
│                             │ 2 near cap. │  │ ⚠ untested│   │
│                             └─────────────┘  │  (14d)   │    │
│                                               └──────────┘    │
│ ▸ Raw provenance stream (live)                                │
│ ▸ Recent readiness-gate results (tiered, same as CI)          │
└──────────────────────────────────────────────────────────────┘
```

## 9. Recommended Resources

1. **Google SRE Book — "Monitoring Distributed Systems" (ch. 6)** — the four golden signals framing, and specifically its argument against aggregate-only dashboards; directly underwrites §6.2 and §6.3's phase/state-specific design.
2. **Grafana docs — "Alerting" and "Dashboard design best practices"** — even if you build `ops-console` as a custom app rather than a Grafana instance, its documented anti-patterns (single aggregate panels, alert fatigue from non-actionable alerts) are the right checklist for §6.3 and §6.5.
3. **Part 6, Module 8 (this handbook)** — re-read the 5 audited seams directly before building; this module is close to a literal UI translation of that module's findings and should be checked against it line by line.
4. **PagerDuty — "Incident Response" documentation** — practical reference for how severity tiers should map to actual paging behavior, relevant to §6.5's requirement that dashboard severity and CI severity agree.
5. **Part 1, Module 7 (Authentication & Authorization)** — revisit specifically to design `ops-console`'s separate auth boundary deliberately, rather than defaulting to reusing `chat-shell`'s.

## 10. Exercises

1. Design the authorization model for `ops-console`: what roles exist, what does each role see (full raw provenance? fleet state? both?), and why shouldn't every internal engineer automatically get every role?
2. Write out, in the tiered severity model from §6.5, which live dashboard conditions should page immediately, which should show as a dashboard warning only, and which shouldn't surface as an alert at all (a correctly-tripped circuit breaker being the canonical example of the last category).
3. `ops-console` is meant to stay usable during a `ai-api-platform` outage. Sketch its data-source architecture: which telemetry does it read directly from `ai-infra-stack`/infra-level sources, and which (if any) legitimately has to go through the primary serving path anyway? Justify any exception.
4. Take Module 8's independence requirement (fleet retry vs. cross-provider fallback tested independently) and design the exact two status indicators the dashboard needs to make that independence visible live, not just provable in CI drills.
5. A raw provenance tag of `agent_derived` with `consolidation: BLOCKED` is normal and expected most of the time. Design a rule for when a *pattern* of such blocks (not a single instance) should actually escalate to an alert — what threshold or trend would indicate an actual poisoning attempt in progress rather than ordinary operation?

## 11. Mini-Project

Build a standalone `<PhaseLatencyPanel>` component (per §6.2's matrix) fed by a hardcoded, timer-driven mock metrics stream that occasionally pushes decode-phase p99 above its SLO target while prefill stays healthy. Confirm the panel correctly shows a breach on decode only, never rolling the two phases into one aggregate status — this isolates the exact discipline Part 6, Module 8 fought to establish, now proven at the render layer before wiring it to real `ai-infra-stack` telemetry in the Production Project.

## 12. Production Project: `ops-console` v1

Build `ops-console` as a new, separately-deployed application within the monorepo — reusing `chat-shell`'s component/reducer conventions (Modules 1–3) but not its deployment, its auth boundary, or its data sources.

**Scope:**

1. **Separate auth boundary** (Exercise 1), built on Part 1, Module 7's foundation but with its own role model (`operator`, `admin`) distinct from `chat-shell`'s end-user identity system — explicitly documented as a deliberate boundary, not a shortcut.
2. **`<PhaseLatencyPanel>`** from the Mini-Project, wired to real `ai-infra-stack` metrics (Part 6, Module 1/8), rendering prefill/decode SLO status live and never aggregated.
3. **Fleet-state panel** (§6.3), distinguishing correctly-tripped memory-pressure circuit breakers (expected, non-alerting) from genuine capacity-risk states (occupancy climbing fleet-wide), reading from `gpu-fleet-gateway` (Part 6, Module 5).
4. **Fallback-layer independence panel** (Exercise 4), showing fleet-internal retry and cross-provider fallback (`llm-gateway`, Part 6, Module 7) as two visibly separate status indicators, including a "last independently tested" timestamp sourced from `ai-infra-readiness-gate`'s drill history — surfacing Module 8's staleness risk (a fallback layer that hasn't actually been exercised recently) directly to an operator, not just to CI logs.
5. **Raw provenance stream** (§6.4), reusing the existing `AgentEvent` `provenance` events from Module 2 with the unreduced rendering, gated behind the operator auth boundary.
6. **Readiness-gate result feed**, surfacing `ai-infra-readiness-gate`'s (Part 6, Module 8) recent CI results live in the dashboard with the same tier labels used in CI — the literal implementation of §6.5's "one severity model" rule.
7. **Independent data-source architecture** (Exercise 3): `ops-console` reads infra telemetry through a path that doesn't depend on `ai-api-platform`'s primary serving path being healthy, so it remains usable during the exact incidents it's meant to help diagnose.
8. **Alert-pattern detection** (Exercise 5): a lightweight rule flagging a *pattern* of blocked consolidations (not single instances) as an actual escalation, distinct from ordinary operation.

**Documentation**: an ADR explicitly justifying the separate-application, separate-auth, separate-deploy decision (§6.1) against the simpler "add an admin toggle to chat-shell" alternative, naming the specific incident scenario that alternative would fail; and an operator runbook cross-referencing each dashboard panel back to the Part 6 module/seam it surfaces, so a new operator can trace any alert back to the design reasoning behind it.

**Hands off to:** the Part 7 capstone, which will need to decide how (or whether) `ops-console` and `chat-shell` share a single deployment story despite being architecturally separate, and the voice interface module, which will need its own answer to §6.1's audience-boundary question for a third, again-different interaction mode.

## 13. Practice & Interview Questions

1. Why shouldn't an operator dashboard reuse the end-user application's authentication and authorization boundary, even though both ultimately sit on the same underlying identity system?
2. Explain the failure mode a single aggregate latency panel would hide that a phase-specific panel catches, using Part 6, Module 8's own regression story as the example.
3. Why is a correctly-tripped circuit breaker the wrong thing to alert on, and what should the dashboard show instead to make that state legible without triggering unnecessary urgency?
4. Design the data-source architecture question from Exercise 3 as an interview answer: how would you ensure a monitoring dashboard remains available during the exact outage it's meant to help diagnose?
5. Why should dashboard alert severity and CI gate severity share one taxonomy rather than each having independently-designed thresholds?
6. What's the operator-facing argument for showing raw provenance tags, when the same tags were deliberately hidden from end users in Module 2? What's the underlying principle that makes both choices correct simultaneously?

## 14. Common Mistakes

- **Bolting operator views onto `chat-shell` as an "advanced" toggle.** Couples the diagnostic tool's availability to the health of the exact system it's meant to diagnose, and conflates two genuinely different auth boundaries.
- **Reporting an aggregate latency SLO on the dashboard.** Directly undoes Part 6, Module 8's phase-specific fix at the one layer — the live dashboard — where the regression would first become visible to a human.
- **Alerting on every circuit-breaker trip regardless of cause.** Trains operators to ignore alerts, which is strictly worse than under-alerting.
- **Folding fleet-retry and cross-provider-fallback status into one "requests succeeding" indicator.** Recreates exactly the invisible-until-both-fail scenario Part 6, Module 8 built drills specifically to catch — now invisible in production instead of just in CI.
- **Inventing a dashboard-specific severity scheme that disagrees with `ai-infra-readiness-gate`'s CI tiers.** Erodes trust in whichever system an operator learns to distrust first.
- **Showing raw provenance tags to end users, or reduced signals to operators.** Both are audience mismatches; each was the right choice for a different consumer, not a universal default.

## 15. Debugging Exercise

Two weeks after `ops-console` ships, there's a real incident: decode-phase p99 latency breaches SLO for eleven minutes. The dashboard's phase-latency panel correctly shows the breach in real time. But no page fires, and the on-call operator only notices by chance while looking at something else.

Form a hypothesis before reading further:

<details>
<summary>Hint 1</summary>
Re-read §6.5. The panel showing red correctly is a *rendering* success. Paging is a *separate* mechanism. What connects the two, and is it possible for one to work while the other silently doesn't?
</details>

<details>
<summary>Hint 2</summary>
Check whether the phase-latency panel's breach state was ever mapped into the same severity taxonomy as `ai-infra-readiness-gate`'s CI tiers, or whether it was built as "just a red indicator" without an explicit tier assignment feeding the alerting system.
</details>

<details>
<summary>Likely root cause</summary>
The `<PhaseLatencyPanel>` was built to correctly *render* SLO breaches (a purely visual/data concern) but was never wired into the alert-severity/paging pipeline described in §6.5 and Exercise 2 — it shows red, but showing red and triggering a page are two different code paths, and only the render path got built and tested. This is the dashboard equivalent of a common Part 6 mistake: building the metric without building the alert on the metric. The fix is to explicitly classify each panel's states against the shared severity taxonomy (§6.5) as a first-class, tested part of the panel's implementation — not an assumed side effect of the panel looking red — and to add exactly the kind of drill Part 6, Module 8 already established the philosophy for: periodically force a synthetic SLO breach and assert a real page fires, not just that a UI element changes color.
</details>

## 16. Checklist

- [ ] `ops-console` deployed as a separate application with its own auth boundary, distinct from `chat-shell`'s end-user identity system
- [ ] Latency panel shows prefill and decode SLO status separately, never as one aggregate figure
- [ ] Circuit-breaker trips distinguished visually from genuine fleet-capacity-risk states; only the latter is alert-worthy
- [ ] Fleet-internal retry and cross-provider fallback shown as two independent status indicators, including staleness of last independent test
- [ ] Raw provenance stream available to operators, reduced signal remains the only view end users see
- [ ] Dashboard alert severity reuses `ai-infra-readiness-gate`'s CI tier taxonomy, verified to agree, not independently invented
- [ ] `ops-console`'s telemetry data path verified independent of `ai-api-platform`'s primary serving path health
- [ ] Pattern-based (not single-instance) escalation rule implemented for blocked-consolidation provenance events
- [ ] A synthetic-breach drill exists proving a real page fires end-to-end, not just that a panel renders red
- [ ] ADR written justifying the separate-app/separate-auth/separate-deploy decision against the simpler alternative

## 17. Summary

An operator dashboard isn't a more detailed version of the end-user product — it's a different product for a different audience with a different trust model, a different auth boundary, and a different availability requirement, and every one of those differences has a concrete design consequence. `ops-console` earns its existence by refusing to repeat, at the UI layer, the exact mistakes Part 6 fought hard to fix at the infra layer: no aggregate SLOs where phase-specific ones caught a real regression, no alerting on a circuit breaker doing its job correctly, no folding two independently-important resilience layers into one status light. Reusing one severity taxonomy across CI and the live dashboard, and verifying the dashboard stays usable during the outages it's meant to diagnose, closes the "no real operator dashboard UI" limitation named at the end of Part 6's capstone — the last of Part 6's honestly-flagged gaps this handbook has now addressed.

## 18. Next Steps

Reply "continue" for Module 5, or flag anything to go deeper on.
