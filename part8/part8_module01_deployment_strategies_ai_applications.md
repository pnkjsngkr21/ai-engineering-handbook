# Part 8, Module 1 — Deployment Strategies for AI-Powered Applications

## 1. Learning Objectives

By the end of this module you will be able to:

1. Explain why deploying a change to an AI-powered application is a strictly harder problem than deploying a typical backend service — identify the specific new axes of change (model version, prompt version, retrieval corpus version, agent policy version) that don't exist in a traditional deployment and can change independently of code.
2. Apply blue-green and canary deployment patterns to each of those axes separately, rather than treating "deploy the app" as one atomic event the way Part 1, Module 8's CI/CD pipeline did for `auth-gateway`.
3. Design a rollback strategy that accounts for AI-specific non-determinism — recognize why "the old version worked, the new one doesn't" can be much noisier to detect than in a deterministic service, and what statistical discipline (reusing `eval-framework`, Part 2, Module 8) is needed to tell a real regression from ordinary variance.
4. Use feature flags to decouple *shipping* code from *enabling* a model/prompt/policy change, and explain why this separation matters more here than in most backend systems.
5. Design a staged-rollout process for a prompt change that reuses `eval-framework`'s golden datasets as a pre-deployment gate, not just a development-time check.
6. Identify what's genuinely new in this module's deployment story versus what's a direct reuse of Part 1, Module 8's CI/CD pipeline and Part 6's infra tooling — and correctly avoid rebuilding either from scratch.
7. Extend `ai-api-platform`'s existing CI/CD pipeline (Part 1, Module 8) with AI-specific deployment gates, producing a single pipeline that handles both ordinary code changes and model/prompt/policy changes with appropriate rigor for each.

## 2. Prerequisites

- Part 1, Module 8 (CI/CD & GitHub Actions) — the tiered fast/slow pipeline for `auth-gateway`; this module extends it rather than replacing it.
- Part 1, Module 11 (Rate Limiting & API Design capstone) — `ai-api-platform` as the assembled target of this module's pipeline work.
- Part 2, Module 8 (Evaluating Models) — `eval-framework`'s golden datasets, pluggable scorers, and confidence intervals; you'll reuse this directly as a deployment gate.
- Part 3, Module 5 (Evaluating Full LLM-Powered Pipelines) — trace-level golden datasets; relevant to what "did the new version actually work" means for a full pipeline, not just a single model call.
- Part 6, Module 4 (Quantization) — you already built a "set an accuracy bar before seeing results" discipline there for a model-precision change; this module generalizes that discipline to every kind of AI-specific change.

## 3. Estimated Study Time

10–13 hours (theory + exercises: ~3 hours; mini-project: ~2 hours; production project: ~5–8 hours).

## 4. Difficulty

★★★☆☆ (3.5/5) — The deployment mechanics themselves (blue-green, canary, feature flags) are standard, well-documented patterns; the difficulty is entirely in correctly identifying the AI-specific axes of change and applying rigorous statistical discipline to rollback decisions rather than eyeballing a dashboard.

## 5. Why This Matters

`ai-api-platform` already has a working CI/CD pipeline from Part 1, Module 8. It's tempting to assume that pipeline is "done" and just needs its target updated as the platform grew through Parts 2–7 — new code paths, sure, but still fundamentally the same kind of deploy. That assumption is wrong in a way that matters: a code deploy and a *prompt* deploy are different events with different risk profiles, even when the underlying infrastructure change is identical (a config value changes, a container restarts). A bad code deploy tends to fail loudly and deterministically — a null pointer exception, a 500 error, a broken endpoint. A bad prompt change, or a bad model version swap, often fails quietly and probabilistically — slightly worse accuracy on a specific input category, a subtly higher hallucination rate, a small faithfulness regression that only shows up on 3% of queries. Reusing a deployment strategy built for the first kind of failure to guard against the second kind is exactly the gap this module closes.

There's a direct link back to Part 6, Module 4's quantization work: that module already discovered that an aggregate benchmark can hide an uneven, task-specific accuracy regression, and that the fix was setting an accuracy bar *before* seeing results, against your own system's actual dependent capabilities. That was one specific instance (a precision change) of a much more general problem this module now names explicitly: *any* AI-specific change — model version, prompt version, retrieval corpus, agent policy — needs the same discipline, and needs it built into the deployment pipeline itself, not left to a one-off evaluation exercise someone remembers to run.

## 6. Theory

### 6.1 The new axes of change — what can change independently of code

A traditional backend service has essentially one axis of deployable change: the code (and its dependencies). `ai-api-platform`, as it now stands after Parts 2–7, has at least four:

- **Code** — the same axis every prior CI/CD module has handled.
- **Model version** — swapping which underlying model `llm-client-core`'s adapter calls (Part 6, Modules 2–3's `VLLMAdapter`/`OllamaAdapter`, or a provider's model string), independent of any code change.
- **Prompt/template version** — a `PromptTemplate` (Part 3, Module 1) can change wording, few-shot examples, or output schema without touching a line of application code.
- **Retrieval corpus / knowledge base version** — `rag-engine`'s underlying indexed documents (Part 4) can be re-ingested or updated independently of both code and model.
- **Agent policy / tool configuration** — `agent-core`'s available tools, `requires_approval` thresholds, or reflection calibration (Part 5) can change independently too.

Each of these axes can be the *actual cause* of a production regression while the code deploy log shows nothing changed. A deployment strategy that only versions and rolls back code is blind to four of its five real change vectors. This module's central design move is to make each axis a first-class, independently versioned, independently rollback-able unit — not folded into "the app version."

```python
# ai-api-platform deployment manifest — each axis versioned independently
deployment_config = {
    "code_version": "v2.14.0",
    "model_version": "claude-sonnet-4-6",       # Part 6/3 territory
    "prompt_version": "support-agent-prompt-v7", # Part 3 territory
    "corpus_version": "kb-snapshot-2026-08-01",  # Part 4 territory
    "policy_version": "agent-policy-v3",         # Part 5 territory
}
```

### 6.2 Blue-green and canary, applied per axis

Blue-green and canary are already familiar patterns for code (implicitly, from Part 1, Module 8's CI/CD work and Part 6's infra modules). The insight this module adds is that they apply *independently* to each axis in §6.1, and mixing them up is a real source of confusion in practice — for instance, canarying a code change while also swapping the model version in the same rollout makes it impossible to attribute a regression to either cause cleanly.

- **Blue-green for a prompt/model swap**: run both the old and new prompt/model version simultaneously behind a router, with 100% of traffic still going to "blue" (the current version) while "green" (the candidate) receives shadow traffic — real production requests, duplicated, with green's outputs logged but never shown to users. This lets you evaluate the candidate against real, current traffic distribution before any user is exposed to it, which is a meaningfully stronger signal than a golden dataset alone (Part 2, Module 8's datasets are necessarily a fixed, dated snapshot; shadow traffic reflects what's actually happening right now).
- **Canary for a prompt/model swap**: after shadow evaluation looks acceptable, route a small percentage (1–5%) of *real* user traffic to the candidate, with `eval-framework`'s scorers (Part 2, Module 8) running against canary traffic's outputs continuously, not just at the shadow stage — because a change that looked fine on shadow traffic can still reveal an issue once its outputs are actually acted on by users (e.g., a subtly worse tool-call formatting that only breaks a downstream parser in production, not in an eval harness).
- **Independent rollback per axis**: because each axis is versioned separately (§6.1), a canary failure on the prompt axis rolls back only the prompt version, leaving code, model, corpus, and policy versions untouched — critically important, because a combined rollback (reverting everything to "the last known-good deploy") throws away unrelated changes that were working fine and re-introduces whatever bugs *they* were fixing.

### 6.3 Rollback under non-determinism — the statistical discipline this module insists on

A traditional service's rollback trigger is usually a clean, deterministic signal: error rate crosses a threshold, latency crosses a threshold. An AI-specific rollback trigger is much noisier, because model outputs are stochastic even for identical inputs (Part 3, Module 8's caching-stochastic-generation discussion is directly relevant background here) — a single bad response from the candidate version doesn't necessarily mean a regression; it might just be an unlucky sample from a distribution that hasn't actually shifted.

The fix, directly reusing Part 2, Module 8's `eval-framework` machinery: **never roll back on a single anecdotal failure; require a confidence-interval-based comparison against the baseline version's known performance distribution**, computed from the same scorers used throughout the handbook (faithfulness, tool-call accuracy, task-specific accuracy per Part 6, Module 4's lesson). Set the rollback threshold — what constitutes a statistically meaningful regression, not just "one weird output" — *before* the canary starts, exactly like Module 4 insisted on setting the accuracy bar before seeing quantization results. Retroactively deciding "that looks bad enough to roll back" after eyeballing a few canary outputs is the deployment-time version of the same mistake: it invites motivated reasoning and inconsistent standards from one deploy to the next.

```python
# Rollback decision — set BEFORE canary starts, not after eyeballing results
rollback_policy = {
    "min_canary_sample_size": 500,       # enough for meaningful CI, per eval-framework's stats
    "scorer": "faithfulness_score",       # from Part 4, Module 8 / Part 3, Module 10
    "max_acceptable_regression": 0.03,    # 3 percentage points, decided in advance
    "confidence_level": 0.95,
}
# Canary is rolled back only if the observed regression's confidence interval
# excludes zero at the pre-declared confidence level AND exceeds the
# pre-declared acceptable-regression threshold — not on a raw eyeballed sample.
```

### 6.4 Feature flags — decoupling "shipped" from "enabled"

A feature flag lets code for a new prompt/model/policy version be deployed (present in the running system, testable internally) without being *enabled* for any real traffic. This decoupling matters more here than in most backend systems for one specific reason: an AI-specific change's blast radius is harder to predict in advance than a typical code change's, precisely because its failure modes are probabilistic rather than deterministic (§6.3). Being able to instantly flip a flag off — reverting to the prior prompt/model/policy version for 100% of traffic, without a redeploy — is a much faster and lower-risk rollback path than reverting a deployment, and should be the default rollback mechanism for anything on the four AI-specific axes, with a full deployment rollback reserved for code-axis regressions.

### 6.5 What's genuinely new here versus reused

Being explicit about this, because conflating "new deployment pattern" with "new infrastructure" leads to needless rebuilding: the actual mechanics of blue-green/canary routing, the CI/CD pipeline's trigger/gate structure, and the underlying infra (`gpu-fleet-gateway`, `ai-infra-stack`) are **entirely reused from Part 1, Module 8 and Part 6** — nothing new needs to be built there. What's new is:

1. Versioning the four AI-specific axes independently (§6.1) rather than folding them into "the code version."
2. Wiring `eval-framework`'s scorers into the canary-evaluation stage as an automated gate (§6.2–6.3), rather than a manual step someone remembers to run.
3. The pre-declared, confidence-interval-based rollback policy (§6.3) as a formal, code-enforced rule rather than an on-call engineer's judgment call under pressure.
4. Feature-flag-first rollback for AI-axis changes specifically (§6.4), as a faster default than a full deployment rollback.

## 7. Mental Models

- **"Code fails loudly and deterministically; a bad prompt fails quietly and probabilistically — your deployment pipeline needs a different alarm for each."**
- **"Four axes, four independent rollbacks — a prompt regression shouldn't cost you your last working model version."**
- **"Set the rollback bar before the canary starts, not after you've seen a few bad outputs you didn't like."**
- **"A feature flag is a faster rollback than a redeploy — reach for it first on any AI-axis change."**

## 8. Visual Explanation

**Four independently versioned axes, each with its own rollback:**

```
┌ code_version: v2.14.0 ────────────────┐  rollback: standard deploy revert
├ model_version: claude-sonnet-4-6 ─────┤  rollback: feature flag → prior model
├ prompt_version: support-agent-v7 ─────┤  rollback: feature flag → prior prompt
├ corpus_version: kb-2026-08-01 ────────┤  rollback: re-point to prior snapshot
└ policy_version: agent-policy-v3 ──────┘  rollback: feature flag → prior policy
```

**Canary flow for a prompt/model change:**

```
 100% traffic ──▶ blue (current)
        │
        ├── shadow traffic (duplicated, not shown to users) ──▶ green (candidate)
        │                                                          │
        │                                              eval-framework scorers
        │                                              (pre-declared thresholds)
        │                                                          │
        ▼                                                  pass? ──▶ canary: 1-5% real traffic
   still 100% on blue                                                       │
                                                                 continuous scoring,
                                                                 CI-based rollback rule
                                                                       │
                                                            pass? ──▶ ramp to 100%
                                                            fail? ──▶ flag off, back to blue
```

## 9. Recommended Resources

1. **Martin Fowler — "Feature Toggles (aka Feature Flags)"** (martinfowler.com) — the canonical reference for flag categories (release, experiment, ops, permission toggles); the AI-axis flags in this module are closest to his "release toggle" category, worth reading to avoid conflating them with longer-lived experiment toggles.
2. **Google — "Canary Analysis" documentation / Kayenta project docs** — practical reference for automated statistical canary analysis, directly relevant to §6.3's confidence-interval-based rollback rule rather than manual dashboard-watching.
3. **Part 2, Module 8 (Evaluating Models) and Part 6, Module 4 (Quantization)** — re-read both immediately before building; this module is close to a direct application of their statistical discipline to a new context (deployment) rather than new methodology.
4. **LaunchDarkly or similar feature-flag platform documentation** (whichever tool `ai-api-platform`'s stack integrates with) — for the practical mechanics of flag-based instant rollback, if not building a bespoke flag system in-house.
5. **Weaveworks/GitOps community docs — "Progressive Delivery"** — a good survey of canary/blue-green terminology and tooling landscape (Flagger, Argo Rollouts) if you want a broader view before deciding on the Production Project's implementation approach.

## 10. Exercises

1. Design the shadow-traffic duplication mechanism for a prompt-version canary: how do you duplicate real production requests to a candidate version without that candidate's (possibly slower, possibly erroring) response ever affecting the real user-facing response path?
2. A canary shows a 1.5 percentage-point faithfulness regression with a confidence interval that includes zero. Per §6.3's pre-declared policy, should this trigger a rollback? What if the same regression had a confidence interval that excluded zero but was still under the pre-declared 3-point threshold?
3. Explain why a combined rollback ("just revert to the last full deploy") is worse than an axis-specific rollback in a scenario where the code axis fixed a real security bug in the same release that introduced a prompt regression.
4. Design the feature-flag schema for the four AI-specific axes (§6.1) — what would the flag's "value" actually be for a model-version flag versus a corpus-version flag, given they're structurally different kinds of change?
5. `agent-core`'s policy version (Part 5) includes `requires_approval` thresholds. Argue whether a policy-version canary should ever be allowed to run on real user traffic without a human approval step in the loop, given that a policy regression's failure mode might be exactly "silently skips an approval it shouldn't have."

## 11. Mini-Project

Build a small deployment-config validator (a CLI tool or a small script) that takes a `deployment_config` (per §6.1's schema) and a `rollback_policy` (per §6.3's schema) and validates: every axis has an independently specified version, the rollback policy's thresholds are all declared (not left as placeholder/TODO values), and the minimum canary sample size is large enough to detect the declared `max_acceptable_regression` at the declared confidence level (a basic power-analysis check). This isolates the "is this rollout actually configured rigorously" question from the harder job of building the real canary infrastructure in the Production Project.

## 12. Production Project: `ai-deployment-pipeline` — Extending `ai-api-platform`'s CI/CD

Extend `ai-api-platform`'s existing CI/CD pipeline (Part 1, Module 8) with AI-specific deployment gates.

**Scope:**

1. **Four independently versioned axes** (§6.1) wired into `ai-api-platform`'s deployment manifest, each with its own version-tracking and rollback path.
2. **Shadow-traffic infrastructure** (Exercise 1) for prompt/model candidate evaluation, built on `resilient-gateway`'s (Part 1, Module 13) existing fan-out capability rather than a new duplication mechanism from scratch.
3. **Canary routing** with `eval-framework` (Part 2, Module 8) and `RAGTraceScorer`/`TraceScorer` (Parts 3–4) wired in as automated, continuous scorers against canary traffic — not a manual evaluation step.
4. **Pre-declared rollback policy enforcement**: the pipeline should reject a canary configuration that doesn't specify its rollback thresholds in advance (the Mini-Project's validator, now enforced as a CI check, not just a standalone tool).
5. **Feature-flag-first rollback** (§6.4) for all four AI-specific axes, with a full deployment rollback reserved for code-axis issues only.
6. **Policy-version human-in-the-loop canary gate** (Exercise 5): any `agent-core` policy-version canary affecting `requires_approval` thresholds requires an explicit human sign-off before ramping past shadow traffic, regardless of automated scorer results — a deliberate, documented exception to full automation, justified by the specific failure mode named in Exercise 5.
7. **`deployment-regression-drill`**: a test harness that intentionally deploys a known-bad prompt/model/policy version through the full pipeline and confirms the pre-declared rollback policy actually fires and actually reverts via feature flag, not just that the scorer correctly detected the regression in isolation.

**Documentation**: an ADR extending Part 1, Module 8's original CI/CD ADR to cover the four new axes, explicitly naming why code-axis and AI-axis changes need different rollback mechanisms; and a runbook for an on-call engineer covering "a canary just failed — what do you do," distinguishing the automated feature-flag rollback path from situations that need a human decision (the policy-version exception from Exercise 5).

**Hands off to:** Part 8's later modules on monitoring/alerting (which will need to distinguish a canary-triggered rollback event from a genuine incident) and secrets/DR (which will extend this module's per-axis versioning discipline to configuration and credential rotation).

## 13. Practice & Interview Questions

1. Name the four AI-specific axes of change this module identifies, and explain why folding them all into "the code version" is a mistake.
2. Why does an AI-specific rollback decision need a confidence-interval-based statistical rule, where a traditional service's rollback trigger can often just be a clean threshold crossing?
3. Explain the difference between shadow traffic and canary traffic, and why shadow evaluation alone isn't sufficient before exposing a candidate version to real users.
4. Why should a feature flag be the default rollback mechanism for a prompt or model version change, rather than a full deployment revert?
5. In an interview: you're asked how you'd safely roll out a new prompt version for a production LLM feature. Walk through shadow traffic, canary percentage, the statistical gate, and the rollback mechanism, in that order, and explain why pre-declaring thresholds matters.
6. Why might a policy-version change (affecting an agent's `requires_approval` behavior) warrant a stricter rollout process than a prompt-version change, even though both are "AI-specific axes" in this module's framing?

## 14. Common Mistakes

- **Treating a model/prompt/policy/corpus swap as part of "the code deploy."** Loses independent rollback and independent risk attribution — a regression on one axis forces reverting everything.
- **Rolling back on a single bad canary output.** Confuses ordinary stochastic variance with a real regression; always require the pre-declared statistical threshold.
- **Declaring rollback thresholds after looking at canary results.** Invites motivated reasoning and inconsistent standards between deploys — set the bar before the canary starts, every time, per Part 6, Module 4's precedent.
- **Using shadow traffic evaluation alone as the sign-off, skipping a real canary stage.** Shadow traffic never actually gets acted on by users; some regressions only surface once real users interact with the candidate's outputs.
- **Fully automating a policy-version rollout affecting `requires_approval` thresholds.** The specific failure mode (a regression that silently skips an approval it shouldn't) is exactly the kind of thing automated scorers might not catch as a "regression" at all if it doesn't manifest as an obviously wrong text output.
- **Assembling scorers into a canary gate without a rollback drill proving the gate actually fires and actually reverts.** The same "assembled ≠ tested" trap named in Part 7's capstone, recurring here at the deployment layer.

## 15. Debugging Exercise

A prompt-version canary at 5% traffic runs for two days with all `eval-framework` scorers reporting well within threshold. On promotion to 100% traffic, faithfulness drops sharply within hours — a regression the canary never caught.

Form a hypothesis before reading further:

<details>
<summary>Hint 1</summary>
The canary sample was evaluated correctly against the scorers it had. What's different about the *traffic* a canary at 5% sees versus the traffic the full rollout sees at 100%, beyond just the size of the sample?
</details>

<details>
<summary>Hint 2</summary>
Think about traffic composition, not traffic volume. Is 5% of traffic, sampled however it was sampled, actually representative of the full traffic mix — including rare but important query categories?
</details>

<details>
<summary>Likely root cause</summary>
The canary's 5% sample was very likely drawn from overall traffic without stratification, meaning it was dominated by the most common query patterns and under-sampled a rarer-but-real category of query where the new prompt version actually regressed — the classic problem of a sample that's large enough in aggregate but not representative across the categories that matter, directly echoing Part 6, Module 4's quantization lesson that an aggregate benchmark can hide a category-specific regression. The fix is to stratify canary sampling by known important query categories (reusing whatever categorization `eval-framework`'s golden datasets already use, Part 2, Module 8) rather than sampling traffic uniformly, and to require the canary's scorer results be reported per category, not just in aggregate, before promotion — the same "evaluate against your own system's dependent capabilities, not an aggregate number" discipline the handbook has now applied to quantization, canary evaluation, and here, canary *sampling* itself.
</details>

## 16. Checklist

- [ ] All four AI-specific axes (model, prompt, corpus, policy) versioned independently from code and from each other
- [ ] Shadow-traffic evaluation stage precedes any real-user canary exposure
- [ ] Canary evaluation uses `eval-framework`/`RAGTraceScorer`/`TraceScorer` scorers continuously, not a one-time manual check
- [ ] Rollback thresholds declared before the canary starts, never adjusted after seeing results
- [ ] Rollback triggers only on a statistically meaningful regression (confidence interval excludes zero and exceeds the declared threshold), never on a single bad output
- [ ] Feature-flag rollback is the default mechanism for AI-axis changes; full deployment rollback reserved for code-axis issues
- [ ] Policy-version canaries affecting `requires_approval` require explicit human sign-off, not full automation
- [ ] Canary sampling is stratified by known important query categories, not uniform over aggregate traffic
- [ ] `deployment-regression-drill` proves the rollback policy actually fires and actually reverts, not just that a scorer detects a regression
- [ ] ADR and on-call runbook written, extending Part 1, Module 8's original CI/CD documentation

## 17. Summary

`ai-api-platform`'s existing CI/CD pipeline already knew how to deploy code safely. What it didn't know how to do — because nothing built before Part 2 needed it — was deploy a model version, a prompt, a retrieval corpus, or an agent policy, each of which can regress the system in a quiet, probabilistic way that a traditional deployment's deterministic failure signals were never built to catch. This module's real contribution is naming those four axes as first-class, independently versioned and independently rollback-able units, and insisting — with the same discipline Part 6, Module 4 first established for quantization — that the bar for "this is a regression" gets set before you look at the results, not after. The mechanics (blue-green, canary, feature flags) are standard; the discipline of applying them correctly to a system whose failures don't announce themselves the way a null pointer exception does is what's actually new, and it's the foundation the rest of Part 8's monitoring, secrets, and disaster-recovery work builds on.

## 18. Next Steps

Reply "continue" for Module 2, or flag anything to go deeper on.
