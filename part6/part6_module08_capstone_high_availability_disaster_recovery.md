# Part 6, Module 8 (Capstone): High Availability & Disaster Recovery for AI Infrastructure

> Closes out Part 6 the way Part 4, Module 9 and Part 5, Module 6 closed
> out their Parts: assembling every artifact built across Modules 1–7
> into one integration-tested, documented infrastructure stack —
> including a real failure-injection audit of `llm-gateway`'s
> newly-introduced single-point-of-failure risk from Module 7.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Trace the **actual runtime dependency graph** of a single inference
   request across the full Part 6 stack — capacity planning, adapter
   selection, gateway routing, fleet routing, autoscaling — and
   identify every point where a failure anywhere in that graph could
   affect request success.
2. Audit at least five **cross-component seams** across Modules 1–7 and
   name the specific production incident each produces when violated.
3. Design and execute a real **failure-injection exercise** against
   `llm-gateway` (Module 7) and `gpu-fleet-gateway` (Module 5),
   measuring actual system behavior under provider outage, instance
   failure, and gateway failure — not assuming resilience mechanisms
   work because they were configured.
4. Define concrete availability targets (SLOs) for AI-specific failure
   modes — model-serving latency SLOs distinct from typical web-service
   SLOs, given Module 1's prefill/decode asymmetry — and design
   monitoring and alerting against them.
5. Assemble every Part 6 eval/benchmark tool
   (`quantization-eval-bench`, `batching-benchmark`,
   `scaling-curve-report`, `warmup-lag-report`,
   `gateway-vs-adapter-benchmark`) into one coordinated operational
   readiness gate.
6. Ship the full Part 6 stack as documented, versioned infrastructure,
   with an honest limitations review naming what's still missing and
   which future Part addresses each.

## 2. Prerequisites

- Part 6, Modules 1–7, completed, with all artifacts on disk and
  runnable.
- Part 4, Module 9 and Part 5, Module 6 — you have already done this
  kind of integration audit twice; this module is the same discipline
  applied to infrastructure instead of application logic.
- Part 1, Module 13 (System Design Fundamentals) — general
  high-availability vocabulary (redundancy, failover, SLOs) that this
  module applies specifically to AI infrastructure.

## 3. Estimated Study Time

13–16 hours across 3 sessions.

## 4. Difficulty

★★★★★ (5/5) — Capstone. As with the prior two capstones, the
difficulty is entirely in proving — through actual failure injection,
not code review — that the assembled stack behaves correctly under
real failure conditions.

---

## 5. Why This Matters

Modules 1–7 each solved a specific problem correctly and, mostly,
in isolation: capacity planning, batching, local development, precision
trade-offs, fleet routing, automated scaling, multi-provider gateways.
But an infrastructure stack's real test isn't whether each piece works
under normal conditions — it's whether the *assembled system* degrades
gracefully, or catastrophically, when something inevitably fails: a
cloud provider's GPU capacity runs out, a hosted API has an outage, the
gateway itself goes down, or an autoscaling event doesn't complete in
time. This is the infrastructure-specific version of the exact lesson
Part 4 and Part 5's capstones taught: component-level correctness does
not imply system-level correctness, and the only way to actually know
whether your resilience mechanisms work is to break things on purpose
and watch what happens — not to read your own circuit-breaker code and
conclude it must be fine.

This matters acutely because Module 7 introduced a new, load-bearing
dependency — `llm-gateway` — that every other component now potentially
routes through, and its own availability was explicitly flagged there
as unaddressed. An infrastructure engineer who ships a gateway without
auditing what happens when the gateway itself fails has moved a
single point of failure, not eliminated one — this module is where that
gets fixed, or at minimum, honestly measured and documented rather than
assumed away.

---

## 6. Theory

### 6.1 The full runtime dependency graph

Tracing a single inference request through the entire Part 6 stack, in
the order it actually executes:

```
 1. calling application (an agent-core agent, rag-engine, or any
    Part 3-5 component) calls llm-client-core's configured adapter
 2. adapter is pointed at llm-gateway's endpoint (per Module 7's
    "realistic combination" default) -- or, for a small number of
    latency-critical exceptions (e.g. the voice agent, per Module 7's
    debugging exercise), directly at a specific backend, bypassing
    the gateway
 3. IF routed through llm-gateway: gateway applies cross-provider
    fallback policy (Module 7) -- decides target provider (hosted API,
    self-hosted fleet, Ollama) based on health/cost/policy
 4. IF routed to the self-hosted fleet: gpu-fleet-gateway (Module 5)
    applies occupancy-aware routing across current instances, with
    its own circuit breaker (memory-pressure trigger) operating
    WITHIN this layer, independent of llm-gateway's cross-provider
    fallback
 5. request lands on a specific vLLM instance, running at whatever
    quantization level was chosen and validated in Module 4
 6. IF current fleet occupancy is high: gpu-infra's custom-metric
    autoscaler (Module 6) may already be provisioning a new instance
    -- subject to the warm-up lag characterized in
    warmup-lag-report
 7. instance serves the request (or fails: OOM, timeout, error)
 8. on failure at step 7: gpu-fleet-gateway's circuit breaker and
    retry logic (Module 5) attempt a different healthy instance
    WITHIN the fleet first
 9. on failure of the ENTIRE fleet (not just one instance): this
    escalates to llm-gateway's cross-provider fallback (step 3),
    which should route to a different provider entirely (hosted API
    or Ollama) -- a DIFFERENT layer than step 8's within-fleet retry
10. response returns to the calling application, or the request fails
    entirely if every fallback layer is exhausted
```

The critical structural fact, and the source of this module's most
important audit: **there are two distinct, nested failure-handling
layers — step 8's within-fleet retry and step 9's cross-provider
fallback — and they must be tested separately, because a bug in either
one can be masked by the other during normal testing.** If you only
ever test "does the system recover from a failure," without
distinguishing which layer actually caught it, you can ship a broken
step 8 (fleet-internal retry) that happens to always get caught by
step 9 (cross-provider fallback) in your tests — meaning every single
degraded request pays the (usually much higher) cost of failing over
to a different provider entirely, when a correctly-working fleet-
internal retry should have caught most failures cheaply and locally.

### 6.2 Seam 1: does capacity planning account for the gateway's added latency?

Module 1's `gpu-capacity-planner` and Module 7's
`gateway-vs-adapter-benchmark` were built at different times, by design,
against different concerns. **Audit requirement:** confirm your
capacity-planning numbers (used for SLO-setting in Section 6.5) account
for whichever path (Section 6.1, step 2) a given workload actually
takes — a latency budget calculated assuming direct-adapter routing
will be systematically wrong for any workload actually routed through
`llm-gateway`, and vice versa. This seam is easy to miss because both
tools individually report correct numbers for what they measured; the
bug is in failing to compose them per-workload.

### 6.3 Seam 2: are the two failure-handling layers (fleet-internal vs. cross-provider) actually tested independently?

Directly following from Section 6.1's structural point: **audit
requirement** — write a test that fails exactly one instance within a
healthy fleet (not the whole fleet, not the gateway) and confirms
recovery happens at step 8 (fleet-internal), without ever engaging step
9 (cross-provider fallback). Separately, write a test that fails the
entire self-hosted fleet and confirms recovery engages step 9
specifically. If your only test is "kill something and confirm the
system eventually recovers," you cannot tell these two seams apart, and
a broken fleet-internal retry can hide behind a working cross-provider
fallback indefinitely, in production, at real extra cost and latency,
until the day the fallback provider is *also* unavailable.

### 6.4 Seam 3: does autoscaling's warm-up lag interact correctly with the circuit breaker?

Module 5's circuit breaker and Module 6's autoscaler operate on
related but distinct signals (memory pressure/health vs. occupancy
level) and on different timescales (a circuit breaker trips in the
time of one bad request; autoscaling takes the warm-up-lag-measured
time from `warmup-lag-report`). **Audit requirement:** confirm that
while a new instance is warming up (Module 6, Section 6.2), the
circuit breaker doesn't incorrectly treat the *existing*, healthy
fleet's genuinely-high (but not failing) occupancy as a health failure
and unnecessarily escalate to cross-provider fallback — a false
escalation here means paying the fallback provider's cost for load that
proactive scaling was already correctly, if slowly, addressing.

### 6.5 Seam 4: are SLOs defined per-phase (prefill/decode), not just aggregate?

Module 1's prefill/decode distinction has a direct SLO consequence that's
easy to skip: a single aggregate "p99 latency" SLO, applied uniformly
across all requests, obscures whether latency problems are
concentrated in prefill (compute-bound, sensitive to instance
contention) or decode (memory-bandwidth-bound, sensitive to concurrent
occupancy) — the same information-hiding problem Module 4's aggregate
quantization benchmark had, now applied to latency monitoring.
**Audit requirement:** confirm your monitoring dashboards
(`observability-stack`, Part 1 Module 4) break out latency by phase,
not just in aggregate, so an on-call engineer diagnosing a latency SLO
violation can immediately tell whether the problem is prefill-side
(likely a scheduling/contention issue) or decode-side (likely an
occupancy/quantization issue) rather than starting from an
undifferentiated number.

### 6.6 Seam 5: is `llm-gateway` itself covered by an availability guarantee?

Module 7 explicitly flagged this as unaddressed. **Audit requirement,
and this module's core new content:** `llm-gateway` needs its own
redundancy — running multiple gateway instances behind a simple,
lower-level load balancer (not the occupancy-aware routing from Module
5, which is specific to GPU-backed instances; a standard load balancer
suffices here since the gateway itself isn't GPU-bound) — and a defined
behavior for what calling applications do if every gateway instance is
unreachable (a last-resort direct-adapter fallback path, bypassing the
gateway entirely for critical workloads, configured explicitly per
Module 7's debugging exercise pattern, not left as an unhandled
failure). Prove this with an actual test: kill every gateway instance
and confirm calling applications degrade to direct-adapter routing
rather than failing outright.

### 6.7 Assembling the operational readiness gate

Mirroring the CI-gate structure from Part 4 and Part 5's capstones, now
for infrastructure readiness rather than code correctness:

- **Tier 1 (blocking before any production deployment change):**
  the seam contract tests from Sections 6.2–6.6, and a fresh run of
  `quantization-eval-bench` if the model or precision changed.
- **Tier 2 (blocking before scaling/capacity changes):**
  `scaling-curve-report` and `warmup-lag-report`, re-run against the
  new configuration, not assumed still valid from a prior run.
- **Tier 3 (scheduled, e.g. monthly, non-blocking but alerting):**
  a full failure-injection drill — provider outage simulation, fleet
  instance failure, and gateway failure — with results compared
  against the SLOs from Section 6.5.

---

## 7. Mental Models

1. **"Fleet-internal retry and cross-provider fallback are two
   different layers — test them separately, or a broken one can hide
   behind a working one indefinitely."**
2. **"A capacity number is only correct for the routing path it assumed
   — compose your tools per actual workload, don't average them."**
3. **"An aggregate latency SLO hides whether the problem is prefill or
   decode — split it, the same way you'd never trust an aggregate
   quantization benchmark."**
4. **"Every new cross-cutting layer is a new thing that can go down —
   the gateway needed its own redundancy plan the moment it existed."**

---

## 8. Visual Explanation

```
   TWO DISTINCT FAILURE-HANDLING LAYERS -- test independently:

   single fleet instance fails
        │
        ▼
   ⚠ SEAM 3: gpu-fleet-gateway circuit breaker (Module 5)
        │  fleet-internal retry -- should catch this HERE,
        │  cheaply, without engaging cross-provider fallback
        ▼
   [recovered within fleet]  -- TEST THIS PATH SEPARATELY

   ══════════════════════════════════════════════

   entire self-hosted fleet fails (not just one instance)
        │
        ▼
   ⚠ SEAM 2: escalates to llm-gateway cross-provider
        │  fallback (Module 7) -- routes to hosted API/Ollama
        ▼
   [recovered via different provider] -- TEST THIS PATH SEPARATELY,
                                          confirm it does NOT mask
                                          a broken fleet-internal path

   ══════════════════════════════════════════════

   ⚠ SEAM 6: llm-gateway itself is unreachable
        │
        ▼
   redundant gateway instances (this module's new content)
        │  all instances down?
        ▼
   last-resort: calling app falls back to DIRECT adapter routing,
   bypassing the gateway entirely for critical workloads
   (per Module 7's debugging exercise pattern, now made systematic)
```

---

## 9. Recommended Resources

1. **Your own Part 4, Module 9 and Part 5, Module 6** — reread both
   immediately before this module's seam audit; the discipline is
   identical, applied to infrastructure.
2. **Google SRE Book — "Service Level Objectives" chapter** — the
   authoritative, general SLO-design reference; read specifically for
   how to translate this module's phase-specific latency argument
   (Section 6.5) into a formally-defined SLO.
3. **Netflix's "Chaos Engineering" writing / Principles of Chaos
   Engineering** — the general discipline behind this module's
   failure-injection methodology; the specific failures you'll inject
   (provider outage, instance failure, gateway failure) are this
   module's AI-infrastructure-specific instantiation of that general
   practice.
4. **Your own Part 1, Module 13 (`resilient-gateway`) and Module 4
   (`observability-stack`)** — this module's redundancy and monitoring
   work extends both directly.

---

## 10. Exercises

1. Write out, from memory, the 10-step runtime dependency graph from
   Section 6.1, marking which steps are fleet-internal (Module 5) and
   which are cross-provider (Module 7). Compare against the original.
2. Write the two independent failure tests from Section 6.3 — single
   instance failure and whole-fleet failure — and confirm each engages
   only the layer it should.
3. Simulate an autoscaling warm-up period (Module 6) concurrent with
   sustained high fleet occupancy. Confirm the circuit breaker doesn't
   incorrectly escalate to cross-provider fallback while proactive
   scaling is still in progress but has not yet failed.
4. Break out your latency monitoring dashboard by prefill vs. decode
   phase specifically, if it isn't already, and confirm you can
   diagnose a simulated latency SLO violation faster with the
   phase-specific view than with the aggregate one alone.
5. Deploy at least two `llm-gateway` instances behind a simple load
   balancer, kill all of them, and confirm calling applications degrade
   to direct-adapter fallback rather than failing outright.

---

## 11. Mini-Project

**`infra-seam-audit-report`**: the third installment of this
handbook's "manually trace, then automate" capstone discipline (after
Part 4 and Part 5's versions). Manually trace three real request
scenarios — one normal, one single-instance failure, one full-fleet-
plus-gateway failure — end-to-end through the 10-step dependency
graph, recording which layer actually handled each failure and at what
latency/cost. Flag any scenario where the wrong layer caught a failure,
or where recovery took longer than your SLO allows.

---

## 12. Production Project: Part 6 capstone — `ai-infra-stack`

### Scope

Assemble every Part 6 artifact into one documented, versioned,
operationally-ready infrastructure stack:

- All five seam fixes from Sections 6.2–6.6 implemented and covered by
  independent contract tests (Section 6.3's two-layer-independence test
  is the most important single test in this module).
- Gateway redundancy (Section 6.6): multiple `llm-gateway` instances,
  a simple load balancer in front of them, and a documented,
  tested last-resort direct-adapter fallback path for critical
  workloads if the gateway layer is entirely unreachable.
- Phase-specific (prefill/decode) SLOs, defined explicitly and wired
  into `observability-stack`'s dashboards and alerting — not a single
  aggregate latency number.
- `ai-infra-readiness-gate`: the three-tier operational readiness gate
  from Section 6.7, assembling `quantization-eval-bench`,
  `batching-benchmark`, `scaling-curve-report`, `warmup-lag-report`,
  and `gateway-vs-adapter-benchmark` into one coordinated suite, run on
  the schedule appropriate to each tier.
- Full documentation:
  - **ADRs** for: the seam fixes themselves (why fleet-internal and
    cross-provider recovery are kept as separate layers), the gateway
    redundancy design, and the phase-specific SLO definitions.
  - **Runbook**: what an on-call engineer does for each class of
    incident (single instance failure, fleet failure, gateway failure,
    autoscaling lag), keyed to which dashboard signal indicates each.
  - **API reference**: unchanged from Part 5's perspective — this is
    the proof point that all of Part 6's work was genuinely
    encapsulated behind stable interfaces (`llm-client-core`'s
    adapters, `Agent`'s calls into them) the whole time; no calling
    code from Part 3 or Part 5 needs to change for any of this
    module's redundancy work to take effect.

### Explicit extension point

**Part 7 (Frontend for AI)** will build real operator-facing dashboards
on top of this module's phase-specific SLO monitoring, replacing raw
metrics views with an actual UI. **Part 8 (Production AI)** will extend
this module's failure-injection discipline to the full application
stack, not just the GPU-serving layer, and will formalize the
scheduled Tier 3 drills from Section 6.7 into a recurring operational
practice.

---

## 13. Practice & Interview Questions

1. Why must fleet-internal retry and cross-provider fallback be tested
   as separate failure paths, rather than one combined "does the system
   recover" test?
2. What specific bug can hide behind a working cross-provider fallback
   if the fleet-internal retry layer is actually broken, and why would
   normal testing likely miss it?
3. Why does introducing a centralized gateway (Module 7) create an
   obligation to design that gateway's own redundancy, and what's the
   concrete fallback behavior if every gateway instance is down?
4. Why should latency SLOs be defined per inference phase (prefill vs.
   decode) rather than as a single aggregate number?
5. How would you design a failure-injection exercise to validate that
   an autoscaler's warm-up lag doesn't get incorrectly treated as a
   circuit-breaker-triggering failure?
6. What's the practical difference between "each component passed its
   own tests" and "the assembled infrastructure stack is production-
   ready," and how does this module's operational readiness gate close
   that gap?

---

## 14. Common Mistakes

- **Testing recovery only end-to-end, never isolating which layer
  handled it** — the single most consequential mistake in this module;
  hides broken fleet-internal retry behind a working cross-provider
  fallback until the fallback provider is also unavailable.
- **Capacity-planning without accounting for which routing path a
  workload actually takes** — producing latency budgets that are
  correct for the tool that generated them but wrong for the actual
  composed path.
- **Treating autoscaling's warm-up lag and the circuit breaker as
  unrelated** — risking unnecessary, expensive cross-provider
  escalation while proactive scaling is still correctly, if slowly,
  in progress.
- **Monitoring only aggregate latency** — hiding exactly which phase
  (prefill or decode) an SLO violation traces back to, slowing incident
  diagnosis.
- **Introducing a gateway without a redundancy plan** — moving a
  single point of failure rather than eliminating one, and finding out
  only during a real outage.

---

## 15. Debugging Exercise

During a real hosted-API provider outage, your system recovered — no
user-visible errors — but your infrastructure bill for that day was
far higher than normal, and your on-call notes show every single
request during the outage window routed through `llm-gateway`'s
cross-provider fallback to a more expensive backup provider, even
though your self-hosted fleet was healthy and had spare capacity the
whole time.

Using Section 6.3 and `infra-seam-audit-report`, walk through: (a) the
most likely explanation (hint: this is Section 6.1's exact structural
risk — check whether the fleet-internal retry layer was actually
functioning, or whether every failure was skipping straight to
cross-provider fallback regardless of whether the fleet itself was
healthy), (b) the specific independent test from Exercise 2 that would
have caught this before the incident, and (c) the fix — stated as a
correction to the escalation logic between step 8 and step 9 of the
dependency graph, not as a monitoring or alerting improvement alone.

---

## 16. Checklist

- [ ] I can draw the full 10-step runtime dependency graph from memory,
      correctly marking fleet-internal vs. cross-provider layers.
- [ ] I have independent, separate tests proving single-instance
      failures are caught by fleet-internal retry, and whole-fleet
      failures are caught by cross-provider fallback — not one combined
      test.
- [ ] I've confirmed autoscaling's warm-up lag doesn't cause
      unnecessary cross-provider escalation while proactively scaling.
- [ ] My latency monitoring is broken out by prefill/decode phase, not
      just aggregate.
- [ ] `llm-gateway` runs redundantly, with a tested last-resort
      direct-adapter fallback if every gateway instance is unreachable.
- [ ] `ai-infra-readiness-gate` assembles all five Part 6 benchmark
      tools into one coordinated, tiered operational readiness process.
- [ ] I've written the ADRs, runbook, and confirmed the API reference
      required zero changes to any Part 3/5 calling code.
- [ ] I can name Part 6's remaining limitations and which future Part
      addresses each.

---

## 17. Summary

Part 6 built seven independently correct pieces of AI infrastructure.
This capstone proved, through explicit seam audits and real failure
injection rather than code review, that they compose into one
system that degrades gracefully rather than catastrophically: fleet-
internal recovery and cross-provider fallback are kept as separate,
independently-tested layers so a broken one can't hide behind a
working one; capacity numbers are composed per actual routing path,
not assumed generically correct; autoscaling's warm-up lag is
reconciled with the circuit breaker so proactive scaling isn't mistaken
for failure; latency SLOs are split by inference phase so incidents are
diagnosable, not just detectable; and the gateway introduced in Module
7 finally gets the redundancy plan its own existence as a new
cross-cutting dependency demanded.

The honest limitations, and where each is addressed:

| Limitation | Addressed in |
|---|---|
| No real operator-facing dashboard UI (raw metrics only) | Part 7 — Frontend for AI |
| Failure-injection discipline not yet extended to the full application stack | Part 8 — Production AI |
| Tier 3 drills are manual/scheduled, not yet a formalized recurring practice | Part 8 — Production AI |

---

## 18. Next Steps

Part 6 is complete. `ai-infra-stack` is your platform-engineering
capstone artifact, sitting underneath every application-layer package
built so far (`llm-client-core`, `rag-engine`, `agent-core`) without
requiring any of them to change.

Part 7 (Frontend for AI) begins next: extending `chat-shell` from Part
0, Module 8 into a real user-facing interface for the systems built
across Parts 3–6 — including, per this module's first named
limitation, a real operator dashboard for the phase-specific SLOs and
infrastructure health signals this capstone established, replacing raw
metrics views with an actual UI.

---

Reply "continue" for Module N, or flag anything to go deeper on.
