# Part 6, Module 7: LiteLLM & Multi-Provider Gateways

> Unifies hosted APIs, self-hosted vLLM, and Ollama behind one routing
> layer with consistent fallback and cost-tracking, extending
> `llm-client-core`'s adapter roster (Modules 2–3) with a proxy-layer
> alternative — and asking directly when a proxy gateway is the right
> architecture versus when the in-process adapter pattern already built
> is sufficient.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain the architectural difference between an **in-process
   adapter** (what `llm-client-core` already is) and an **out-of-process
   proxy gateway** (what LiteLLM's proxy server is), and state precisely
   what capability the proxy model buys you that the adapter model
   structurally cannot.
2. Deploy LiteLLM's proxy server unifying hosted APIs, `VLLMAdapter`'s
   underlying vLLM endpoint, and Ollama behind one OpenAI-compatible
   interface.
3. Configure cross-provider fallback routing (if provider A fails or
   rate-limits, retry against provider B) at the gateway level, and
   explain why this is a different, complementary layer from
   `resilient-gateway`'s circuit breaker, not a replacement for it.
4. Implement centralized, cross-team cost tracking and budget
   enforcement at the gateway layer — a capability that's awkward to
   get right when every service embeds its own adapter directly.
5. Make an explicit, justified architectural decision for your own
   system: keep `llm-client-core`'s adapters as the integration layer,
   adopt a proxy gateway instead, or run both in a specific, deliberate
   combination — and defend that choice, rather than defaulting to
   whichever is more novel.

## 2. Prerequisites

- Part 6, Modules 2–3 (`VLLMAdapter`, `OllamaAdapter` in
  `llm-client-core`) — this module is a direct architectural comparison
  against that existing work, not a fresh integration exercise.
- Part 1, Module 7 (Authentication & Authorization, `auth-gateway`) —
  a proxy gateway is architecturally similar to `auth-gateway`'s
  centralized-cross-cutting-concern pattern, applied to LLM routing
  instead of identity.
- Part 3, Module 9 (Latency & Cost Optimization, model routing) —
  this module's cross-provider routing is a direct extension of that
  routing discipline, now spanning providers instead of model tiers
  within one provider.

## 3. Estimated Study Time

7–9 hours across 1–2 sessions.

## 4. Difficulty

★★★☆☆ (3/5) — Mostly configuration and architectural-comparison work;
the genuine intellectual content is the in-process-vs-proxy trade-off
analysis, not new mechanism.

---

## 5. Why This Matters

By this point you have a working, adapter-based multi-provider
integration in `llm-client-core` — hosted APIs, vLLM, and Ollama are
all already accessible behind one interface, from within your
application's own process. It would be reasonable to ask why this
module exists at all. The answer is architectural, not
capability-based: an in-process adapter is a great fit when one
codebase (or a small number of tightly-coordinated services sharing a
package) needs multi-provider access. It is a poor fit the moment you
have **multiple independent services or teams**, each with their own
codebase, that all need consistent provider routing, fallback, rate
limiting, and — critically — centralized cost visibility across all of
them. Re-embedding `llm-client-core` correctly in every service, and
keeping routing/fallback policy consistent across all of them as
requirements change, is a coordination problem that a shared, in-process
package can't solve by itself; that's precisely the problem a
standalone proxy gateway is built to solve, the same way `auth-gateway`
(Part 1, Module 7) solved authorization as a cross-cutting concern
once you had more than one service that needed it.

This matters for interview-readiness because "should this be a shared
library or a standalone gateway service" is a genuine, recurring
system-design question — for LLM routing specifically as much as for
authentication, rate limiting, or any other cross-cutting concern — and
being able to answer it with a specific, principled criterion (not "the
gateway approach is more modern") is exactly the kind of judgment an
infrastructure interview is testing for.

---

## 6. Theory

### 6.1 In-process adapter vs. out-of-process proxy: the actual distinction

`llm-client-core`'s adapters (Part 3, Module 1; extended in Modules
2–3) run **inside** your application's process — every service that
wants multi-provider routing imports the package and configures it
itself. LiteLLM's proxy server runs as a **separate, standalone
service** that any application talks to via a standard (typically
OpenAI-compatible) HTTP interface, with the actual provider-routing,
fallback, and rate-limiting logic living entirely in the proxy, not in
each calling application.

The precise capability difference: an in-process adapter's
configuration and behavior is scoped to whatever process imports it —
consistent behavior across multiple services requires each service to
import the same version of the package with the same configuration,
which is a real coordination burden as the number of independent
services grows. A proxy gateway's configuration lives in exactly one
place, and every calling service — regardless of what language or
framework it's written in — gets identical routing, fallback, and cost-
tracking behavior automatically, simply by pointing at the gateway's
HTTP endpoint. This is the same architectural trade-off as a shared
library versus a dedicated microservice for any cross-cutting concern
(Part 1, Module 7's authorization discussion, and more generally Part
0, Module 13's system-design fundamentals) — not a new pattern, but a
new, concrete instance of a pattern you already understand.

### 6.2 What a proxy gateway buys you that an in-process adapter structurally cannot

- **Cross-service consistency without re-deployment coordination.**
  Updating fallback policy or adding a new provider in the gateway
  takes effect for every calling service immediately, without each
  service needing to update its own dependency version and redeploy —
  a real operational win once you have more than a couple of
  independent services.
- **Centralized cost tracking and budget enforcement.** Because every
  request from every service passes through one gateway, you get a
  single, authoritative place to track spend per team/project/API key
  and enforce budget caps — genuinely awkward to do consistently when
  cost-tracking logic is duplicated (or worse, inconsistently
  implemented) across many services' individual adapter
  configurations.
- **Language/framework independence.** A proxy gateway with a
  standard HTTP interface can be called from any language — relevant if
  your organization's services aren't all built on the same stack that
  `llm-client-core` targets.

### 6.3 What you lose, or what stays the same

The proxy model is not strictly better — it introduces its own costs:
an additional network hop (real latency cost, worth measuring rather
than assuming negligible, especially relevant given Part 5's voice
agent latency sensitivity), an additional piece of infrastructure to
deploy, monitor, and keep highly available (if the gateway goes down,
every service depending on it loses LLM access, a new single point of
failure that didn't exist when each service had its own in-process
adapter), and less type-safety/compile-time checking than an in-process
package can offer within a single codebase's own language ecosystem.

**Fallback routing at the gateway layer is complementary to, not a
replacement for, `resilient-gateway`'s circuit breaker (Part 1, Module
13; Part 6, Module 5):** the gateway's cross-provider fallback handles
"provider A is down, route to provider B" — a different concern from
`gpu-fleet-gateway`'s intra-fleet circuit breaking, which handles "one
specific self-hosted instance within provider B's own fleet is
unhealthy." A well-designed system keeps both layers: `gpu-fleet-
gateway`'s occupancy-aware routing and memory-pressure circuit breaking
operate *within* the self-hosted vLLM fleet, and LiteLLM's proxy
routing operates *across* that fleet and other providers (hosted APIs,
Ollama) as a whole.

### 6.4 The decision criterion, stated explicitly

- **Stay with `llm-client-core`'s in-process adapters alone** if you
  have one codebase, or a small number of tightly-coordinated services
  that can reasonably share a package version and configuration, and
  the operational burden of an additional standalone service isn't
  justified yet.
- **Adopt a proxy gateway (LiteLLM or equivalent)** once you have
  multiple independent services/teams needing consistent routing and
  cost visibility, where the coordination cost of keeping an in-process
  package consistently configured across all of them exceeds the
  operational cost of running and maintaining a gateway.
- **Run both, in combination** — the realistic answer for most
  systems at real scale: `llm-client-core` remains the interface each
  service's code calls (preserving its Adapter-pattern benefits within
  each codebase), but its adapters are configured to point at the
  LiteLLM gateway's endpoint rather than directly at each provider —
  giving you in-process ergonomics for application developers and
  centralized cross-cutting control for platform/infrastructure
  operators, simultaneously.

This mirrors the exact "measure and justify, don't default" discipline
Module 3 established for Ollama-vs-vLLM and Module 6 established for
Kubernetes automation — the proxy gateway is a real architectural
upgrade with a real cost, adopted when a specific, statable coordination
problem justifies it, not because it's the more sophisticated-sounding
option.

---

## 7. Mental Models

1. **"An in-process adapter is configured per-service; a proxy gateway
   is configured once, for every service that calls it — the trade-off
   is coordination cost versus a new operational dependency."**
2. **"Gateway-level fallback handles cross-provider failures; your
   fleet's circuit breaker handles within-provider failures — keep
   both, they're not the same layer."**
3. **"A shared library and a standalone gateway are the same
   cross-cutting-concern trade-off you already know from
   authentication — now applied to LLM routing."**
4. **"The realistic answer at scale is usually both: adapters for
   application-code ergonomics, gateway for centralized operator
   control."**

---

## 8. Visual Explanation

```
   IN-PROCESS ADAPTER ONLY (Modules 2-3):

   service A ──[imports llm-client-core]──► provider X, Y, Z
   service B ──[imports llm-client-core]──► provider X, Y, Z
   service C ──[imports llm-client-core]──► provider X, Y, Z
        (each service configures its OWN routing/fallback/cost
         tracking -- consistency requires coordinated deployment)

   ──────────────────────────────────────────────────

   PROXY GATEWAY (LiteLLM):

   service A (any language) ──┐
   service B                   ├──► LiteLLM proxy ──► provider X, Y, Z
   service C                   ┘     (ONE place: routing, fallback,
                                       cost tracking, budget caps --
                                       consistent for every caller,
                                       no per-service redeploy needed)

   ──────────────────────────────────────────────────

   REALISTIC COMBINATION (Section 6.4):

   service A ──[llm-client-core, adapter now points AT the gateway]──┐
   service B ──[llm-client-core, adapter points at the gateway]──────┼──► LiteLLM proxy ──► provider X, Y, Z
   service C ──[llm-client-core, adapter points at the gateway]──────┘
        application code keeps clean in-process ergonomics;
        platform team gets centralized operator control
```

---

## 9. Recommended Resources

1. **LiteLLM official documentation** — the current, practical
   reference for proxy deployment, fallback configuration, and cost-
   tracking setup.
2. **Your own Part 1, Module 7 (`auth-gateway`) material** — re-read the
   shared-library-vs-gateway trade-off argument you already made for
   authentication; this module applies the identical reasoning to LLM
   routing, deliberately, so the comparison should feel familiar rather
   than new.
3. **Your own Part 3, Module 9 material** — the cross-provider routing
   this module implements at the gateway layer is the same discipline
   as that module's within-provider model-tier routing, one level up.

---

## 10. Exercises

1. Deploy LiteLLM's proxy server, configured to route to at least one
   hosted API provider, your `VLLMAdapter`'s vLLM endpoint, and your
   local Ollama instance — confirming all three are reachable through
   one unified interface.
2. Configure cross-provider fallback (e.g., if the hosted API rate-
   limits or errors, fall back to the self-hosted vLLM endpoint), and
   test it by deliberately triggering a failure on the primary
   provider.
3. Configure cost tracking with a budget cap for a simulated "team,"
   and confirm requests are blocked or flagged once the cap is
   exceeded.
4. Measure the added latency of routing through the proxy versus
   calling a provider directly through `llm-client-core`'s adapter, and
   assess whether that added latency matters for a latency-sensitive
   workload like Part 5's voice agent.
5. Using Section 6.4's criterion, write an explicit justification for
   your own project: in-process adapters only, a full gateway, or both
   in combination — and if "both," specify exactly which components
   point at the gateway and which don't.

---

## 11. Mini-Project

**`gateway-vs-adapter-benchmark`**: a short, measured comparison
report — latency added by routing through the proxy, and a clear
statement of which specific coordination problem (if any) your current
project actually has that would justify adopting the gateway. This is
meant to prevent the module's own content from becoming an unexamined
default; the deliverable is a decision, backed by your own numbers, not
an assumption that the more architecturally-elaborate option is
automatically better.

---

## 12. Production Project: `llm-gateway` (LiteLLM deployment)

### Scope

Deploy and integrate `llm-gateway`:

- LiteLLM proxy server configured to route across hosted APIs,
  `VLLMAdapter`'s vLLM fleet (behind `gpu-fleet-gateway`, Module 5), and
  Ollama (Module 3) — one unified endpoint for every provider this
  handbook has built.
- Cross-provider fallback policy, explicitly distinguished in
  documentation from `gpu-fleet-gateway`'s intra-fleet circuit breaking
  (Section 6.3) — both layers present, each handling its own scope.
- Centralized cost tracking with budget enforcement, configured per
  logical "team" (even if that's currently just yourself, structured as
  if it weren't, so it's ready to extend).
- `llm-client-core`'s adapters reconfigured, per Section 6.4's
  "realistic combination," to point at `llm-gateway`'s endpoint rather
  than directly at each provider — application code (every `agent-core`
  agent from Part 5, `rag-engine` from Part 4) is completely unchanged,
  because the Adapter pattern already abstracted the actual endpoint
  away.
- `gateway-vs-adapter-benchmark` (Mini-Project) as the documented,
  evidence-based justification for this architecture, attached to the
  deployment's ADR.

### Explicit extension point

**Part 6, Module 8 (Capstone: High Availability & Disaster Recovery)**
will treat `llm-gateway` itself as a critical dependency requiring its
own availability guarantees — since, per Section 6.3, it's now a
potential single point of failure for every service that depends on it
— and will design redundancy specifically for this new failure surface.

---

## 13. Practice & Interview Questions

1. What's the precise architectural difference between an in-process
   adapter and a standalone proxy gateway for multi-provider LLM
   routing?
2. Name one capability a proxy gateway provides that an in-process
   shared library structurally cannot, and explain why.
3. Why is gateway-level cross-provider fallback a different concern
   from a self-hosted fleet's internal circuit breaker, and why should
   a real system keep both?
4. What new failure mode does introducing a centralized proxy gateway
   create, and how would you mitigate it?
5. Give a concrete criterion for deciding whether your own system needs
   a proxy gateway, an in-process adapter, or both — not a general
   preference.

---

## 14. Common Mistakes

- **Adopting a proxy gateway because it's the more "production-grade"
  sounding option**, without a specific coordination problem it solves
  — paying real operational cost for no corresponding benefit.
- **Treating gateway-level fallback as a replacement for
  `gpu-fleet-gateway`'s circuit breaker** — they operate at different
  scopes and both are usually needed.
- **Not measuring the proxy's added latency** before deciding it's
  acceptable for a latency-sensitive workload — assumed-negligible
  latency additions are exactly what Part 6's "measure, don't assume"
  discipline exists to catch.
- **Introducing a gateway without planning for its own availability** —
  creating a new single point of failure for every dependent service
  without a corresponding redundancy plan.
- **Duplicating cost-tracking logic across services instead of
  centralizing it in the gateway** — the specific problem this module's
  architecture exists to solve, undermined if teams keep their own
  parallel tracking anyway.

---

## 15. Debugging Exercise

After introducing `llm-gateway`, your voice agent (Part 5, Module 5)
reports a small but noticeable latency regression compared to its
pre-gateway performance, even though the gateway's own dashboard shows
healthy routing and no fallback events triggered.

Using Section 6.3 and `gateway-vs-adapter-benchmark`, walk through:
(a) the most likely explanation (hint: a healthy gateway with no
fallback events can still add latency simply from the additional
network hop itself, independent of any routing logic), (b) the specific
measurement that would confirm this (comparing direct-adapter latency
against through-gateway latency for the exact same request, isolating
the hop's cost), and (c) the architectural decision this implies for
the voice agent specifically — should it be routed through the gateway
at all, given Part 5, Module 5's tight latency budget, or should it be
one of the components that talks to `VLLMAdapter` directly, bypassing
the gateway, while less latency-sensitive services still route through
it?

---

## 16. Checklist

- [ ] I can state the precise architectural difference between an
      in-process adapter and a proxy gateway, not just "one is newer."
- [ ] I have a working LiteLLM proxy unifying hosted APIs, vLLM, and
      Ollama behind one interface.
- [ ] I've configured and tested cross-provider fallback, and can
      explain why it's distinct from `gpu-fleet-gateway`'s circuit
      breaker.
- [ ] I've configured centralized cost tracking with a real budget cap
      test.
- [ ] I've measured the gateway's added latency myself, not assumed it
      negligible, and made an explicit decision about whether
      latency-sensitive workloads (the voice agent) should route
      through it.
- [ ] I have a written, criterion-based justification (Section 6.4) for
      my own project's architecture choice — adapter-only, gateway, or
      both.

---

## 17. Summary

A proxy gateway and an in-process adapter solve the same basic
problem — multi-provider LLM access — at different architectural
scopes, and the choice between them is the same shared-library-versus-
dedicated-service trade-off you already reasoned through for
authentication in Part 1. A gateway buys centralized, cross-service
consistency in routing, fallback, and cost tracking, at the real cost
of an added network hop and a new potential single point of failure.
The correct answer for most systems at real scale is both together:
`llm-client-core`'s adapters stay the interface application code calls,
now pointed at the gateway's endpoint — application ergonomics and
centralized operator control, simultaneously, decided by an explicit
criterion and a measured latency cost, not by which option sounds more
sophisticated.

---

## 18. Next Steps

Next: **Part 6, Module 8 (Capstone): High Availability & Disaster
Recovery for AI Infrastructure**, closing out Part 6 the way Part 4 and
Part 5's capstones closed out their Parts: assembling every artifact
built across Modules 1–7 (`gpu-capacity-planner`, `VLLMAdapter`/
`OllamaAdapter`, `quantization-eval-bench`, `gpu-fleet-gateway`,
`gpu-infra`, `llm-gateway`) into one integration-tested, documented
infrastructure stack — including a real failure-injection audit of
`llm-gateway`'s newly-introduced single-point-of-failure risk from this
module.

Reply "continue" for Module 8, or flag anything to go deeper on.
