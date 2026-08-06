# Part 8, Module 3 — Full-Stack Monitoring & Alerting for AI Applications

## 1. Learning Objectives

By the end of this module you will be able to:

1. Distinguish three monitoring layers that now exist in `ai-api-platform` — infrastructure health (`ai-infra-stack`, Part 6), application observability (`observability-stack`, Part 1, Module 4), and AI-quality signals (`eval-framework` and friends, Parts 2–4) — and explain why collapsing them into one dashboard or one alert queue hides exactly the failures each layer exists to catch.
2. Design a unified alerting taxonomy that spans all three layers without flattening their different severity models, extending Part 7, Module 4's "one severity model" principle from the frontend/infra boundary to the full stack.
3. Build AI-quality monitors that run continuously in production, not just at deployment time (Module 1's canary gates) — detecting slow drift in faithfulness, hallucination rate, or tool-call accuracy that a canary window is too short to catch.
4. Correctly attribute a production incident to one of several plausible causes spanning multiple layers — a spike in "bad" agent outputs might be an infra issue (Part 6), a deployment regression (Module 1), a credential problem (Module 2), or a genuine model/prompt drift — and design the diagnostic path that distinguishes them quickly.
5. Design alert fatigue prevention specific to AI systems, where "quality" metrics are inherently noisier than typical infra metrics (Module 1, §6.3's confidence-interval discipline) and a naive threshold-based alert will fire constantly on ordinary variance.
6. Extend `observability-stack`'s existing OpenTelemetry tracing (Part 1, Module 4) to carry AI-specific span attributes (model version, prompt version, token counts, retrieval scores) end to end through a request, so a single trace can answer "which of the five axes changed right before this went wrong."
7. Assemble a full-stack incident-response runbook that routes an alert to the correct owning team/system based on which layer and axis it implicates.

## 2. Prerequisites

- Part 1, Module 4 (Logging, Monitoring & Observability) — `observability-stack`'s OTel tracing, structured logs, and AI-specific metrics via EventBus; this module extends it directly.
- Part 6, Module 8 (HA/DR capstone) — `ai-infra-readiness-gate` and its tiered severity model, already reused once in Part 7, Module 4 and now extended again.
- Part 7, Module 4 (Operator Dashboard) — the phase-specific SLO panel and the "one severity model across CI and live dashboard" principle this module extends to the whole stack.
- Part 8, Modules 1–2 (Deployment Strategies; Secrets) — both are potential root causes this module's diagnostic path needs to distinguish from a genuine model/prompt-quality issue.

## 3. Estimated Study Time

10–13 hours (theory + exercises: ~3 hours; mini-project: ~2 hours; production project: ~5–8 hours).

## 4. Difficulty

★★★★☆ (3.5/5) — No single hard new mechanism, but correctly designing cross-layer incident attribution requires holding the full system (infra, app, AI-quality, deployment, secrets) in mind at once — closer in shape to a capstone-style integration challenge than a typical module.

## 5. Why This Matters

By this point, `ai-api-platform` has three genuinely different kinds of monitoring already built, each from a different part of the handbook, each solving a real problem in isolation: `observability-stack` (Part 1) traces requests and logs structured events; `ai-infra-stack`/`ai-infra-readiness-gate` (Part 6) watches GPU occupancy, phase-specific latency, and fleet health; `eval-framework` and its many extensions (Parts 2–4) score output quality against golden datasets and, since Module 1, against canary traffic. None of these three were ever unified, because each was built to solve the specific problem in front of it at the time. In production, an actual incident doesn't announce which layer it belongs to — a support-agent's answers getting subtly worse could be an infra problem (GPU contention degrading generation quality via a timeout-driven fallback to a weaker model), a deployment regression (Module 1's prompt-canary process was skipped or its threshold was too loose), a credential issue (Module 2's rotation stranded some fraction of runs), or a genuine, slow model/data drift that nothing else in the system was designed to catch because it's not a discrete deployment event at all.

This module exists because the cost of *not* unifying these three layers isn't redundant tooling — it's slow, confused incident response, where an on-call engineer has to manually check three separate systems, each with its own severity language, before even knowing which team or which of Part 8's other modules the problem actually belongs to. Getting this right is what turns three separately-excellent monitoring systems into something that's actually operable under real production pressure.

## 6. Theory

### 6.1 Three layers, three different failure signatures

Before unifying anything, get precise about what each layer actually catches, because conflating them is the root of most alert-fatigue and misattribution problems:

- **Infrastructure layer** (`ai-infra-stack`, Part 6): fails in ways that are usually fast and clearly attributable — a GPU OOM, a fleet node dropping out, a latency SLO breach on a specific phase (prefill/decode). Signal is typically clean and near-real-time.
- **Application layer** (`observability-stack`, Part 1): fails in traditional ways — an unhandled exception, a timeout, a broken downstream integration. Signal is also typically clean, deterministic, and near-real-time.
- **AI-quality layer** (`eval-framework` and its extensions): fails in slow, probabilistic, often statistically subtle ways — a faithfulness score drifting down a few points over days, a hallucination-rate uptick that only clears a confidence threshold after accumulating enough samples. Signal is inherently noisier and requires the statistical discipline Module 1 already established for canaries, now applied continuously rather than only during a deployment window.

The practical consequence: these three layers need different alerting latencies and different confidence requirements by design, not as an oversight to fix — infra and app alerts can and should fire fast on a clean threshold crossing; AI-quality alerts need the same confidence-interval-based discipline as a canary rollback decision, evaluated over a rolling window, or they'll fire constantly on ordinary output variance and get ignored.

### 6.2 Continuous AI-quality monitoring — beyond the canary window

Module 1's canary process catches a quality regression *at deployment time*, within a bounded evaluation window. It cannot catch a regression that develops *after* a version has been running safely for weeks — a slow drift in the actual distribution of user queries (a new, harder category of question becoming common), a subtle change in an upstream data source feeding `rag-engine`'s corpus, or degradation in a third-party provider's model behavior that Anthropic or OpenAI ships without you deploying anything at all.

The fix: **the same scorers used for canary evaluation (`eval-framework`, `RAGTraceScorer`, `TraceScorer`) run continuously in production, on a sampled slice of live traffic, with a rolling-window statistical comparison against a trailing baseline** — not a fixed pre-deployment baseline, since the "correct" baseline for detecting slow drift needs to be recent enough to not be stale, but stable enough to not just be re-measuring noise against itself. This is a genuinely new monitoring category, distinct from both the canary process (bounded, deployment-triggered) and from infra/app monitoring (fast, deterministic) — it runs indefinitely, on a schedule, independent of any deploy event.

```python
# Continuous quality monitor — independent of any deployment event
class QualityDriftMonitor:
    scorer: Scorer                    # reused from eval-framework
    sample_rate: float                # e.g., 2% of live traffic, sampled
    baseline_window: timedelta         # trailing window, e.g., last 14 days
    comparison_window: timedelta       # recent window being evaluated, e.g., last 24h
    min_sample_size: int
    confidence_level: float
    max_acceptable_drift: float        # same "declare before you see results" discipline as Module 1
```

### 6.3 Cross-layer incident attribution — the diagnostic decision tree

Given an observed symptom (say, "user-reported answer quality complaints spiked this morning"), the diagnostic path needs to check layers in a specific, principled order — cheapest and most deterministic signals first, because ruling out simple causes quickly prevents a slow statistical investigation into AI-quality drift when the real cause was something as mundane as a fleet node dropping out:

1. **Infra layer first**: check `ai-infra-stack`/`gpu-fleet-gateway` state (Part 6, Module 5; Part 7, Module 4's dashboard) — was there a circuit-breaker trip, a fallback to a degraded model tier, an occupancy spike? These are fast, deterministic checks and a common actual root cause masquerading as a "quality" problem (a fallback to a smaller/quantized model under load, per Part 6, Module 4's uneven-degradation lesson, looks exactly like a quality regression from the outside).
2. **Deployment/credential axis second**: check `ai-deployment-pipeline`'s (Module 1) recent axis versions and `ai-secrets-governance`'s (Module 2) recent rotation events — did any of the five axes (code, model, prompt, corpus, policy, credentials) change recently, and did a canary gate get bypassed or its threshold get loosened under pressure?
3. **AI-quality layer last**: only once 1 and 2 are ruled out, treat it as a genuine drift/quality issue requiring `QualityDriftMonitor`'s statistical investigation (§6.2) — the slowest and least deterministic layer to investigate, and therefore the last one to reach for, not the first.

This ordering is itself the module's central contribution — not a new detection mechanism, but a principled diagnostic sequence that prevents defaulting to "the model must be getting worse" (the hardest thing to prove or disprove) before checking two much cheaper, more common, and more fixable explanations first.

### 6.4 Extending OTel tracing with AI-specific span attributes

`observability-stack` (Part 1, Module 4) already traces requests end to end. To make §6.3's diagnostic sequence fast rather than requiring manual cross-referencing across three separate systems, every trace needs to carry the AI-specific context that could explain a quality change directly as span attributes: which model version served this request, which prompt version, which corpus snapshot, which policy version (Module 1's four axes, now attached to individual traces rather than just to deployment-level config), plus token counts, retrieval scores, and reflection/faithfulness outcomes where applicable.

```python
with tracer.start_as_current_span("agent_run") as span:
    span.set_attribute("ai.model_version", model_version)
    span.set_attribute("ai.prompt_version", prompt_version)
    span.set_attribute("ai.corpus_version", corpus_version)
    span.set_attribute("ai.policy_version", policy_version)
    span.set_attribute("ai.faithfulness_score", faithfulness_score)
    span.set_attribute("ai.retrieval_top_k_score", top_k_score)
```

This turns "which axis changed right before this went wrong" from a manual, cross-system investigation into a single trace query — directly serving §6.3's diagnostic sequence, and closing a real gap: without this, an on-call engineer diagnosing a quality complaint would have to separately check `ai-deployment-pipeline`'s version history and correlate it by timestamp with `observability-stack`'s traces, rather than reading the answer directly off one trace.

### 6.5 One severity taxonomy across all three layers — extending, not inventing

Part 7, Module 4 already established that a live dashboard's alert severity should reuse a CI gate's tier taxonomy rather than inventing a separate one. This module extends that same principle across all three monitoring layers and every gate built so far in the handbook (`agent-ci-gate`, `rag-ci-gate`, `ai-infra-readiness-gate`, `frontend-ci-gate`, `ai-deployment-pipeline`'s canary gate): **one severity taxonomy, consistently applied**, so that a Tier 1 finding in any gate and a Tier 1 live alert in production mean the same thing to whoever's on call, regardless of which layer or which historical module built the underlying check.

## 7. Mental Models

- **"Infra and app failures are fast and clean; AI-quality failures are slow and statistical — alert on each with the confidence discipline it actually deserves."**
- **"Check the cheap, deterministic layers before the slow, statistical one — a fallback to a weaker model under load looks exactly like a quality regression from the outside."**
- **"A canary catches a regression at deploy time; a drift monitor catches one that develops afterward — you need both, they're not the same tool."**
- **"One severity taxonomy across every gate and every dashboard this handbook has built — a Tier 1 finding means the same thing everywhere, or none of them mean anything."**

## 8. Visual Explanation

**Three monitoring layers, different failure signatures:**

```
 Infra layer         : fast, deterministic   (GPU OOM, node drop, SLO breach)
 Application layer   : fast, deterministic   (exception, timeout, integration failure)
 AI-quality layer     : slow, statistical     (faithfulness drift, hallucination uptick)
```

**Diagnostic sequence for an ambiguous quality complaint:**

```
 symptom reported
       │
       ▼
 1. Check infra (ai-infra-stack) ──found cause?──▶ yes: resolve here, stop
       │ no
       ▼
 2. Check deployment/credential axes (Modules 1-2) ──found cause?──▶ yes: resolve here, stop
       │ no
       ▼
 3. Investigate via QualityDriftMonitor (statistical, slowest)
```

**One trace, five axis attributes, answering "what changed":**

```
 span: agent_run
   ai.model_version    = claude-sonnet-4-6
   ai.prompt_version    = support-agent-v7
   ai.corpus_version    = kb-2026-08-01
   ai.policy_version     = agent-policy-v3
   ai.faithfulness_score = 0.71   ◀── compare against rolling baseline directly from the trace
```

## 9. Recommended Resources

1. **Part 1, Module 4 (this handbook)** — re-read directly before starting; this module's OTel extension work builds on `observability-stack` exactly as specified rather than as new infrastructure.
2. **Google SRE Book — "Monitoring Distributed Systems" (ch. 6) and "Alerting" appendix** — re-read specifically the "symptoms vs. causes" framing, directly relevant to §6.3's diagnostic ordering (check cheap/fast causes before slow/statistical ones).
3. **OpenTelemetry — "Semantic Conventions for Generative AI"** (opentelemetry.io) — the emerging standard for exactly the span-attribute extension in §6.4; worth checking your custom attribute names against the standard's evolving conventions rather than inventing your own naming scheme from scratch.
4. **Part 6, Module 4 (Quantization)** — re-read the section on uneven, task-specific degradation; directly relevant to why an infra-layer fallback (e.g., to a quantized model under load) can present identically to a genuine quality regression, motivating §6.3's "check infra first" ordering.
5. **Honeycomb — "Observability Engineering" (O'Reilly, if accessible) or their public blog on high-cardinality tracing** — useful for thinking through how to keep AI-specific span attributes (which can be high-cardinality, e.g., per-request retrieval scores) queryable at scale without overwhelming the tracing backend.

## 10. Exercises

1. Design the rolling-window comparison for `QualityDriftMonitor` (§6.2): what baseline window length and comparison window length would you choose, and what's the trade-off between a baseline that's too short (noisy, re-measures against itself) and one that's too long (stale, slow to reflect a real recent shift)?
2. Walk through §6.3's diagnostic sequence for a specific scenario: faithfulness scores drop 4 points over 6 hours. What are the first three concrete things you'd check, in order, before concluding it's a genuine quality drift requiring deeper investigation?
3. Write out the AI-specific OTel span attributes (§6.4) you'd add beyond the six shown in the example, and justify each addition against a specific diagnostic question it would help answer.
4. A Tier 1 finding in `rag-ci-gate` (Part 4) and a Tier 1 live alert in this module's unified taxonomy should mean the same thing. Pick one specific `rag-ci-gate` Tier 1 check and design its live-production equivalent alert, including what threshold would make it fire.
5. Design an alert-fatigue safeguard for the AI-quality layer specifically: given that even a correctly-implemented confidence-interval-based alert will occasionally fire on a real but minor and self-correcting fluctuation, what escalation policy (e.g., requiring the drift to persist across N consecutive windows before paging) would you add on top of the statistical threshold alone?

## 11. Mini-Project

Build a standalone `QualityDriftMonitor` simulator: feed it a mocked time series of daily faithfulness scores (mostly stable, with one deliberately injected slow drift over the final several days) and confirm it correctly distinguishes the injected drift from the stable baseline period's ordinary day-to-day noise, using a real confidence-interval calculation rather than a fixed threshold. This isolates the statistical core of continuous quality monitoring before wiring it to real production traffic sampling in the Production Project.

## 12. Production Project: `ai-observability-unified` — One Taxonomy, Three Layers

Unify `observability-stack` (Part 1), `ai-infra-stack`/`ai-infra-readiness-gate` (Part 6), and `eval-framework`'s production-facing extensions (Parts 2–4) into one coherent monitoring and alerting system for `ai-api-platform`.

**Scope:**

1. **AI-specific OTel span attributes** (§6.4) added to every relevant trace across `llm-client-core`, `rag-engine`, and `agent-core`'s existing instrumentation, extending `observability-stack` rather than introducing a parallel tracing system.
2. **`QualityDriftMonitor`** from the Mini-Project, deployed against real sampled production traffic, using `eval-framework`'s existing scorers, with the rolling-window design from Exercise 1 and the persistence-based escalation safeguard from Exercise 5.
3. **Unified severity taxonomy** (§6.5) applied consistently across `agent-ci-gate`, `rag-ci-gate`, `ai-infra-readiness-gate`, `frontend-ci-gate`, `ai-deployment-pipeline`'s canary gate, and this module's new live alerts — a single documented tier definition referenced by every one of them, not six independently-defined severity schemes that happen to use similar-sounding labels.
4. **Cross-layer diagnostic runbook** implementing §6.3's ordered sequence as an actual on-call procedure, with direct links from each step to the specific dashboard/tool that answers it (`ops-console`'s fleet-state panel for step 1, `ai-deployment-pipeline`'s version history and `ai-secrets-governance`'s rotation log for step 2, `QualityDriftMonitor`'s dashboard for step 3).
5. **`incident-attribution-drill`**: a test harness that deliberately injects one root cause at a time (a simulated fleet degradation, a simulated bad deployment, a simulated genuine quality drift) and confirms the diagnostic runbook correctly identifies each within its expected layer, and — critically — confirms the *other* two layers don't produce false-positive alerts when the true cause lies elsewhere.

**Documentation**: an ADR documenting the unified severity taxonomy as the single source of truth, explicitly superseding any layer-specific severity language that predates it (with a note that existing gates keep their tier assignments, just now under one shared definition); and the cross-layer runbook itself as a first-class, regularly-reviewed operational document.

**Hands off to:** Part 8's disaster-recovery module, which will need this module's unified alerting to correctly distinguish a real DR-triggering event from a false alarm across any of the three layers; and Part 9/10 (Freelancing; Interview Preparation), where the ability to articulate this kind of full-stack incident-response design cleanly is itself a direct interview asset, not just an operational one.

## 13. Practice & Interview Questions

1. Why do infrastructure, application, and AI-quality monitoring need different alerting latencies and confidence requirements, rather than one unified threshold-based alert rule applied everywhere?
2. Explain why a canary process (Module 1) and a continuous drift monitor (this module) are not redundant with each other — what does each catch that the other structurally cannot?
3. Walk through the diagnostic-ordering argument in §6.3: why check infra and deployment/credential causes before investigating a genuine AI-quality drift, even if quality drift turns out to be the actual cause more often in practice?
4. Why does extending OTel tracing with AI-specific span attributes (model/prompt/corpus/policy version, quality scores) meaningfully speed up incident diagnosis, compared to checking each system separately by timestamp correlation?
5. In an interview: you're asked to design monitoring for a production LLM application. Walk through the three layers, how their alerting differs, and how you'd prevent an on-call engineer from wasting time chasing a "quality regression" that was actually an infra fallback in disguise.
6. Why should every CI gate and every live-monitoring alert across a system share one severity taxonomy, rather than each subsystem defining its own?

## 14. Common Mistakes

- **Applying one uniform alert-threshold policy across infra, app, and AI-quality layers.** AI-quality signals are inherently noisier and need confidence-interval-based, rolling-window evaluation; a naive fixed threshold either fires constantly or misses real drift.
- **Treating the canary gate (Module 1) as sufficient ongoing quality monitoring.** It only evaluates a bounded window at deploy time and is structurally blind to drift that develops afterward.
- **Investigating AI-quality drift first when a symptom is reported, before checking infra and deployment/credential causes.** Wastes time on the slowest, least deterministic investigation when the actual cause is often a much cheaper, faster check away.
- **Building AI-specific tracing as a separate system from `observability-stack`'s existing OTel instrumentation.** Creates two disconnected sources of truth an on-call engineer has to manually correlate, defeating the point of the extension.
- **Letting each gate/dashboard invent its own severity taxonomy.** A Tier 1 in one system and a "critical" in another can mean subtly different things, undermining trust in whichever one an operator learns to distrust first — the same lesson Part 7, Module 4 already established, now at full-stack scale.
- **Skipping the incident-attribution drill and assuming the unified runbook works because each layer's individual tooling was already tested.** The same "assembled ≠ tested" trap this handbook has flagged at every capstone — cross-layer attribution needs its own explicit test.

## 15. Debugging Exercise

`QualityDriftMonitor` pages the on-call engineer for a statistically significant faithfulness drop. The engineer follows the runbook, checks infra (clean), checks deployment/credential history (no recent changes on any axis), and concludes it must be genuine model drift — but three days of investigation turn up nothing, and the "drift" quietly resolves on its own.

Form a hypothesis before reading further:

<details>
<summary>Hint 1</summary>
The runbook's step 2 checked `ai-deployment-pipeline`'s axis history and `ai-secrets-governance`'s rotation log. Are those the *only* two things that could change what kind of traffic the system is receiving, without touching any of the five versioned axes at all?
</details>

<details>
<summary>Hint 2</summary>
Think about §6.2's own caveat: "a slow drift in the actual distribution of user queries... becoming common." Is a shift in the *incoming query mix* itself something the diagnostic sequence in §6.3 explicitly checks for, or does it fall through the cracks between "deployment change" and "genuine model drift"?
</details>

<details>
<summary>Likely root cause</summary>
The diagnostic sequence in §6.3, as originally scoped, has a real gap: it checks whether any of the five *system* axes changed (infra, code/model/prompt/corpus/policy, credentials), but never checks whether the *input distribution* itself shifted — for instance, a marketing campaign or a seasonal pattern temporarily sending a much higher proportion of a harder query category that the system has always handled somewhat worse, without any deployment or genuine model-quality change at all. This isn't "drift" in the sense the monitor was built to catch (a change in the model/system's behavior) — it's a change in *what's being asked*, which resolves on its own once the traffic mix reverts, exactly matching the observed symptom. The fix is to add an explicit step to the diagnostic sequence — checking query-category distribution shifts, using whatever categorization `eval-framework`'s golden datasets already use (Part 2, Module 8) — positioned between step 2 (deployment/credentials) and step 3 (genuine quality investigation), since it's cheaper to check than a full quality investigation but wasn't part of the original three-layer framing. This is a good instance of a broader lesson: a diagnostic runbook, like any other system in this handbook, needs its own capstone-style audit once real incidents reveal a category of cause the original design didn't anticipate.
</details>

## 16. Checklist

- [ ] AI-specific OTel span attributes (model/prompt/corpus/policy version, quality scores) added across `llm-client-core`, `rag-engine`, `agent-core` traces
- [ ] `QualityDriftMonitor` running continuously against sampled production traffic, using rolling-window confidence-interval comparison, not a fixed threshold
- [ ] Persistence-based escalation (drift must hold across multiple windows) implemented to prevent alert fatigue on self-correcting fluctuations
- [ ] One documented severity taxonomy referenced by every existing gate and every new live alert, with no independently-invented alternative schemes remaining
- [ ] Cross-layer diagnostic runbook implements the infra-then-deployment-then-quality ordering, with direct links to the tool answering each step
- [ ] Query-distribution-shift check included in the diagnostic sequence, addressing the gap found in §15
- [ ] `incident-attribution-drill` confirms each of infra/deployment/quality root causes is correctly attributed, and confirms the other two layers don't false-positive
- [ ] ADR documents the unified severity taxonomy as superseding, not duplicating, prior layer-specific severity language

## 17. Summary

Three excellent, independently-built monitoring systems — infrastructure health, application observability, and AI-quality evaluation — don't automatically add up to a system that's operable under real incident pressure; they add up to three places an on-call engineer has to look, in an order nobody agreed on, using severity words that don't necessarily mean the same thing. This module's contribution is almost entirely about the seams between systems that were each already correct: an explicit, cost-ordered diagnostic sequence that checks fast/deterministic causes before slow/statistical ones, one severity taxonomy shared across every gate this handbook has built, and a genuinely new capability — continuous quality drift monitoring — that fills the real gap between "caught at deploy time" (Module 1's canary) and "never caught at all." The debugging exercise's finding (that even a well-designed diagnostic sequence can miss a category of cause until a real incident reveals the gap) is itself the module's most important lesson: full-stack monitoring is never actually finished, only as complete as the incidents that have tested it so far.

## 18. Next Steps

Reply "continue" for Module 4, or flag anything to go deeper on.
