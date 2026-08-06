# Part 8, Module 4 — Full-Stack Disaster Recovery for AI Applications

## 1. Learning Objectives

By the end of this module you will be able to:

1. Extend Part 6, Module 8's failure-injection discipline — built and proven at the GPU/infra layer only — to the full application stack: `ai-api-platform`'s API layer, `rag-engine`'s vector store, `agent-core`'s memory stores, `ai-secrets-governance`'s credential systems, and Part 7's three frontend shells.
2. Identify which components of the stack are stateful in ways that make disaster recovery genuinely hard (agent memory, paused `approval_pending` runs, vector store indices) versus stateless components that can simply be redeployed.
3. Design and test a recovery point objective (RPO) and recovery time objective (RTO) per component, recognizing that a uniform RPO/RTO across the whole stack is almost always wrong — a lost RAG corpus snapshot and a lost in-flight agent run have very different acceptable-loss profiles.
4. Apply Module 3's unified monitoring/alerting to DR specifically: distinguish a genuine disaster-triggering event from a false alarm, using the same cross-layer diagnostic discipline rather than a separate DR-specific detection system.
5. Design and run a full-stack DR drill that goes beyond Part 6, Module 8's GPU-layer drills — testing whether an in-flight, paused `agent-core` run can actually survive and correctly resume after a real regional failover, not just whether the infra comes back up.
6. Extend the credential dual-key rotation discipline (Module 2) to a DR context specifically: what happens to an in-flight rotation during a failover, and how the secrets system itself needs its own DR story.
7. Produce a complete, tested DR runbook and a named, honest assessment of what's still not covered — following the exact pattern of every capstone-style module this handbook has used before.

## 2. Prerequisites

- Part 6, Module 8 (HA/DR capstone) — the GPU/infra-layer failure-injection discipline, the 5 audited seams, and `ai-infra-readiness-gate`; this module is the direct extension of that module's own named limitation ("failure-injection discipline not yet extended past the GPU layer").
- Part 5, Module 3 (Agent Memory) and Module 6 (Agent capstone) — the four memory stores and the resumption guarantee for paused `approval_pending` runs; this module tests whether that guarantee survives an actual DR event, not just an ordinary pause.
- Part 8, Modules 1–3 (Deployment; Secrets; Monitoring) — all three are inputs to this module's DR design: deployment rollback, credential rotation, and cross-layer alerting all need their own DR-specific behavior.
- Part 4, Module 2 (Vector Databases & ANN Search) — the vector store's own persistence/backup characteristics, relevant to RAG corpus recovery.

## 3. Estimated Study Time

12–15 hours (theory + exercises: ~3.5 hours; mini-project: ~2.5 hours; production project: ~6–9 hours).

## 4. Difficulty

★★★★★ (4.5/5) — The hardest module in Part 8. Not because any single mechanism is novel, but because a full-stack DR drill genuinely has to exercise every stateful system this handbook has built, simultaneously, under an actual simulated failure — there's no way to shortcut the breadth this requires.

## 5. Why This Matters

Part 6, Module 8 built real disaster-recovery discipline — but scoped, deliberately and honestly, to the GPU/infra layer only, and named that scoping as an explicit limitation to be picked up later. Since then, the system has grown a great deal of state that a GPU-layer DR drill never touched: `agent-core`'s four memory stores (Part 5, Module 3), potentially long-paused `approval_pending` runs (Part 5, Module 6; tested for ordinary pauses, never for a real regional failure), `rag-engine`'s vector indices (Part 4), and now `ai-secrets-governance`'s credential rotation state (Module 2). A DR plan that only covers GPU fleet failover and calls the system "recoverable" is making a claim about 20% of what actually needs to survive a real incident.

This module also directly tests one of this handbook's most important running guarantees under its hardest possible condition. Every module since Part 5 has assumed that a paused `approval_pending` run can resume correctly — tested against ordinary pauses (Part 5, Module 6), against a page refresh (Part 7, Module 2), against a mid-flight credential rotation (Module 2). None of those tests simulated an actual regional failover, where the entire backend a paused run depends on might be recreated from a backup in a different region, potentially with a slightly stale memory-store snapshot. If the resumption guarantee doesn't survive *that*, it was never really proven — only proven under conditions gentle enough to let it pass.

## 6. Theory

### 6.1 Stateful versus stateless — the real DR scoping question

Not every component needs the same DR treatment, and treating them uniformly wastes effort on the easy parts while under-investing in the hard ones. Categorize honestly:

- **Stateless, trivially recoverable**: `ai-api-platform`'s API layer, `llm-client-core`'s adapters, `agent-core`'s execution logic itself (the code, not its data) — these can simply be redeployed from the last known-good version (Module 1's deployment pipeline) in a new region with no data-loss concern at all.
- **Stateful, batch-recoverable**: `rag-engine`'s vector store (Part 4, Module 2) and knowledge graph (Module 6) — these have real data that must be backed up, but the acceptable staleness is usually measured in hours, since a corpus snapshot from a few hours ago is still broadly correct, and a modest RPO here is a reasonable, well-understood trade-off.
- **Stateful, high-stakes, hard to batch-recover**: `agent-core`'s memory stores (scratch-pad, episodic, semantic, resumable task-state — Part 5, Module 3) and any currently-paused `approval_pending` run. A stale snapshot of *this* category isn't a minor staleness trade-off — it can mean a paused run resumes into a world where its own prior reasoning steps are missing or, worse, where a memory write it made right before the disaster is silently lost and later contradicted by its own next action, which is a subtler and more dangerous failure than an outright crash.
- **Stateful, security-sensitive**: `ai-secrets-governance`'s credential state (Module 2), especially any in-flight dual-key rotation — recovering into a world where the "active" credential in the recovered snapshot has already been revoked in the primary region would be a correctness failure with security implications, not just an availability one.

This categorization is the module's foundational move: **RPO/RTO targets should be set per category, not as one number for the whole stack**, and the third and fourth categories deserve most of this module's actual engineering effort, precisely because they're the ones a naive, uniform DR plan would under-invest in.

### 6.2 RPO/RTO, set deliberately per category — reusing Module 1's "declare before you see results" discipline

Following the same discipline Module 1 established for rollback thresholds and Part 6, Module 4 established for quantization accuracy bars: **RPO/RTO targets are declared in advance, per category, before a drill is run** — not backfilled after seeing what a drill happens to achieve, which would just be describing your current capability rather than setting a real target.

```python
rpo_rto_targets = {
    "api_layer":            {"rpo": "0 (stateless)",  "rto": "5 minutes"},
    "rag_vector_store":      {"rpo": "4 hours",         "rto": "30 minutes"},
    "agent_memory_stores":   {"rpo": "5 minutes",        "rto": "15 minutes"},
    "paused_approval_runs":  {"rpo": "0 (must not lose an in-flight decision)", "rto": "15 minutes"},
    "credential_state":      {"rpo": "0 (must not resurrect a revoked key)",     "rto": "10 minutes"},
}
```

Notice that `paused_approval_runs` and `credential_state` both get an RPO of effectively zero — this is a deliberate, considered choice, not an unrealistic aspiration: losing or corrupting either isn't a "some data loss is acceptable" situation the way a few hours of RAG corpus staleness is, because the failure modes (a stranded human decision; a resurrected revoked credential) are correctness and security failures, not availability ones.

### 6.3 Testing the resumption guarantee under a real failover — not just an ordinary pause

This is the module's central new drill, and the reason this module exists rather than simply extending Part 6, Module 8's existing drills in place. The test: deliberately pause an `agent-core` run in `approval_pending` state, then trigger a real simulated regional failover — not a mocked pause/resume cycle, but an actual failover of the backend the run depends on, recovering from the most recent backup per the RPO targets in §6.2 — and confirm the run resumes correctly once approved, in the failed-over region, with its memory-store state intact per the declared RPO.

The specific things this drill needs to catch, each corresponding to a way the resumption guarantee could quietly fail under real DR conditions that an ordinary-pause test would never exercise:

- **Does the failover correctly preserve the `approval_pending` state itself**, or does a restored-from-backup system come back up believing the action was never proposed at all (a state that existed only after the backup was taken)?
- **If the memory-store backup is slightly stale** (per the declared RPO), does the resumed run correctly recognize that some of its own recent reasoning might be missing, rather than confidently proceeding as if nothing happened — this is a sharper version of the provenance-integrity concern from Part 5, Module 3, now triggered by infrastructure failure rather than by another agent's untrusted output, but requiring the same kind of care.
- **Does the credential the resumed run needs to complete its action still exist and remain valid** in the failed-over region, given Module 2's dual-key rotation discipline — or could a rotation that completed in the primary region before the disaster have left the failed-over backup still referencing the now-revoked prior credential?

### 6.4 DR detection reuses Module 3's monitoring, doesn't invent a separate system

It's tempting to build DR-specific detection and alerting as its own system. Don't — Module 3 already built a unified, cross-layer monitoring and alerting taxonomy specifically so that every subsystem's severity language means the same thing. A DR-triggering event is, definitionally, one of the most severe things Module 3's taxonomy can classify — it should be detected and paged through the exact same unified alerting pipeline, at the top severity tier, rather than through a parallel DR-specific monitoring stack that could disagree with the primary system about whether an event is actually happening.

### 6.5 Honest scope — what full-stack DR still doesn't cover

Following this handbook's consistent capstone discipline: naming what's genuinely out of scope is as important as what's covered. This module's drill tests a single-region failover for the core stateful systems (§6.1's categories three and four) plus credential-state consistency (§6.3's third bullet). It deliberately does not yet cover: a simultaneous multi-region failure (a much rarer, higher-severity scenario reasonably deferred), DR for Part 7's three frontend shells' own state (largely stateless from the frontend's own perspective, since all durable state lives server-side, but not yet explicitly drilled), or a "partial" disaster where only one of several stateful subsystems fails while others remain healthy (a scenario Part 6, Module 8's own audited seams suggest is at least as likely and dangerous as a clean, total regional failure, and a natural candidate for a future, deeper DR module if this handbook's scope ever extends that far).

## 7. Mental Models

- **"Not every piece of state deserves the same RPO — a stale RAG snapshot is a quality trade-off; a lost paused approval is a correctness failure."**
- **"An ordinary pause-and-resume test proves nothing about what happens when the whole backend under that pause gets recreated from a backup."**
- **"DR detection isn't a separate alarm system — it's the top severity tier of the one alerting taxonomy you already built."**
- **"Set the RPO/RTO target before the drill, exactly like you set a rollback threshold before the canary — otherwise you're just describing what you got, not what you needed."**

## 8. Visual Explanation

**Four state categories, four different DR postures:**

```
 stateless (API layer, adapters)        → redeploy, RPO/RTO trivial
 batch-recoverable (RAG vector store)   → hours-scale RPO acceptable
 hard-to-batch-recover (agent memory,   → near-zero RPO required —
   paused approval_pending runs)          correctness failure if violated
 security-sensitive (credential state)  → near-zero RPO required —
                                            security failure if violated
```

**The failover drill, tracing a paused run through a real regional recreation:**

```
 run paused: approval_pending  ──▶  [simulated regional failure]
                                            │
                                   backend recreated from backup
                                   (per declared RPO for its category)
                                            │
              human approves  ──▶  does resumed run in NEW region:
                                     - correctly see the pending decision?
                                     - correctly detect any memory staleness?
                                     - authenticate with a still-valid credential?
```

## 9. Recommended Resources

1. **Part 6, Module 8 (this handbook)** — re-read directly and completely before starting; this module is explicitly the extension of its named limitation, not new methodology.
2. **AWS Well-Architected Framework — "Reliability Pillar," DR strategies section** — a solid practical reference for RPO/RTO terminology and standard failover patterns (backup/restore, pilot light, warm standby, multi-site), useful for calibrating §6.2's per-category targets against industry-standard categories.
3. **Google SRE Workbook — "Disaster Recovery Testing"** — directly relevant to designing the failover drill in §6.3 as a real, periodically-repeated exercise rather than a one-time proof, echoing the same "untested independence is a hope, not a guarantee" lesson from Part 7's capstone.
4. **Part 5, Module 3 (Agent Memory)** — re-read the provenance and consolidation-policy sections immediately before designing §6.3's memory-staleness detection; the mechanism you need here is closely related to, though triggered by a different cause than, the poisoning-detection work already built there.
5. **Netflix Technology Blog — "Chaos Engineering" posts (any era)** — useful general framing for designing a drill that actually injects a real failure rather than a convenient simulation, relevant to avoiding the trap this module's own debugging exercise illustrates.

## 10. Exercises

1. For each of the four state categories in §6.1, name the specific existing artifact from this handbook that holds that category's state (e.g., which module built the RAG vector store, which built the agent memory stores) and confirm your RPO/RTO targets from §6.2 against the artifact's actual persistence characteristics.
2. Design the memory-staleness detection mechanism referenced in §6.3's second bullet: what signal would tell a resumed agent run that its own memory-store snapshot might be missing recent writes, and what should it do differently as a result (proceed cautiously, refuse to proceed, alert a human)?
3. Walk through Module 2's dual-key rotation state during a regional failover: if a rotation completed in the primary region 30 seconds before the disaster, and the backup used for failover is 5 minutes old, what state does the failed-over region believe about which credential is "active," and what's the fix?
4. Design the "partial disaster" scenario named as out of scope in §6.5 in enough detail to scope a future module: what's a plausible single-subsystem failure (say, just the vector store, with everything else healthy) and why might it be more dangerous than a clean total regional failure?
5. Argue whether Part 7's three frontend shells (`chat-shell`, `ops-console`, `voice-shell`) genuinely need their own DR drill, given they hold no durable state themselves — under what specific condition would you conclude they need one anyway?

## 11. Mini-Project

Build a small simulation harness (a script, not real infrastructure) modeling one `agent-core` run's state machine (per Part 5, Module 1/6) paused in `approval_pending`, and simulate a "backup taken, then failure, then restore from that backup" sequence with a configurable staleness gap. Confirm your harness can detect the three failure conditions named in §6.3 (lost pending state, undetected memory staleness, stale credential reference) at least in the abstract, before wiring this logic against a real backend in the Production Project.

## 12. Production Project: `full-stack-dr-drill` — Extending Part 6's DR Discipline

Extend Part 6, Module 8's GPU/infra-layer DR work to the full application stack.

**Scope:**

1. **Per-category RPO/RTO targets** (§6.2) declared and documented for every stateful component identified in §6.1 (Exercise 1's mapping), before any drill is run.
2. **Full-stack failover drill** (§6.3) implemented against a real (or realistically simulated, e.g., a staging environment with genuine backup/restore mechanics) regional failover, covering: `rag-engine`'s vector store restoration, `agent-core`'s four memory stores' restoration, and a live `approval_pending` run's survival and correct resumption through the entire failover.
3. **Memory-staleness detection** (Exercise 2), extending Part 5, Module 3's provenance/consolidation machinery to recognize a post-restore staleness gap as a distinct signal, triggering cautious behavior (at minimum, surfacing the staleness to a human via `ops-console`, Part 7, Module 4, rather than proceeding silently).
4. **Credential-state consistency check** (Exercise 3), extending Module 2's dual-key rotation drill to explicitly cover the failover scenario — confirming a resumed run never authenticates with a credential that's actually been revoked in the (now-primary) failed-over region.
5. **DR alerting integrated into Module 3's unified taxonomy** (§6.4) — no separate DR-specific alert system; a DR-triggering event is simply the highest severity tier of the existing pipeline.
6. **`full-stack-dr-drill` as a recurring, scheduled exercise**, not a one-time proof — directly following Part 6, Module 8 and Part 7, Module 6's shared lesson that an untested resilience claim decays into an unfounded assumption if it's never re-run.
7. **Honest scope documentation** (§6.5): explicitly named out-of-scope scenarios (multi-region simultaneous failure, partial/single-subsystem disasters, frontend-shell-specific DR), each with a brief note on why it's reasonably deferred and what would need to happen for it to become in-scope.

**Documentation**: a full DR runbook covering every stateful component's recovery procedure, cross-referenced to the specific handbook module that built it; and an ADR documenting the per-category RPO/RTO decisions with their rationale, explicitly contrasting them against a rejected "one uniform RPO/RTO" alternative.

**Hands off to:** Part 9 and Part 10 (Freelancing; Interview Preparation), where a full-stack DR story — especially the honest, scoped limitations section — is itself a strong, concrete answer to "tell me about a system you've made resilient" in a senior interview context; and, if the curriculum extends further, any future module addressing the partial-disaster and multi-region scenarios explicitly named as out of scope here.

## 13. Practice & Interview Questions

1. Why is a single, uniform RPO/RTO target across an entire AI application stack almost always the wrong design choice? Give a concrete example of two components in this stack with legitimately different acceptable-loss profiles.
2. Explain why testing a paused `approval_pending` run's resumption against an ordinary pause/reconnect (Part 7, Module 2) is not sufficient to claim it survives a real disaster-recovery event, and what specifically differs.
3. Walk through the credential-consistency failure mode in §6.3's third bullet: how could a dual-key rotation (Module 2) that worked correctly under normal operation still produce an incorrect outcome during a regional failover?
4. Why should a DR-triggering event be detected through the same unified monitoring/alerting taxonomy (Module 3) rather than a dedicated, separate DR detection system?
5. In an interview: you're asked to design disaster recovery for a stateful, multi-component AI system (agent memory, vector search, paused human-in-the-loop actions). Walk through how you'd categorize components by recovery difficulty and set RPO/RTO targets differently for each.
6. Why is it important to name what a DR plan explicitly does *not* cover (§6.5), rather than presenting a drilled scenario as if it proves the whole system is disaster-proof?

## 14. Common Mistakes

- **Setting one uniform RPO/RTO for the whole stack.** Either wastes engineering effort protecting low-stakes state to an unnecessarily strict standard, or dangerously under-protects high-stakes state (paused approvals, credential consistency) by holding it to the same lenient bar as a RAG corpus snapshot.
- **Treating Part 6, Module 8's GPU-layer DR drills as proof the whole system is disaster-recoverable.** That module explicitly scoped itself to infra and named the gap; treating it as complete coverage ignores its own honest limitations section.
- **Testing resumption only against ordinary pauses, never against an actual simulated failover.** Passes a much easier test and calls the harder, real guarantee proven when it hasn't been.
- **Building a separate DR-specific alerting/detection system.** Duplicates Module 3's unified taxonomy and risks the two systems disagreeing about whether a disaster is actually occurring.
- **Treating a passed DR drill as a permanent guarantee rather than a snapshot in time.** The same decay-without-re-testing lesson this handbook has now made in Part 6, Part 7, and here — schedule the drill to recur.
- **Skipping the honest out-of-scope section, or writing it as a formality.** A DR plan that doesn't name what it doesn't cover invites false confidence exactly where it matters most — during a real incident that falls into an uncovered gap.

## 15. Debugging Exercise

The full-stack DR drill passes cleanly in staging: a paused `approval_pending` run survives a simulated regional failover, resumes correctly, and authenticates with a valid credential. Three months later, a real regional incident occurs, and the equivalent real production run fails to resume — the resumed system reports the action as never having been proposed at all.

Form a hypothesis before reading further:

<details>
<summary>Hint 1</summary>
The staging drill and the real incident both simulate/experience "a regional failover." What's different about how the staging drill's backup was taken versus how backups are actually scheduled and taken in the real production environment?
</details>

<details>
<summary>Hint 2</summary>
Consider timing: the staging drill likely takes a backup, immediately pauses a run, then immediately fails over — a tight, convenient sequence. Real backups run on a fixed schedule (say, every N minutes) independent of when a real approval happens to get paused. What's the actual worst-case gap between "run entered `approval_pending`" and "the most recent backup that would be used for a real failover" in production?
</details>

<details>
<summary>Likely root cause</summary>
The staging drill's sequence — take a backup, then pause a run, then fail over — guarantees the backup used always postdates the pause, so the drill can never actually exercise the realistic worst case: a run enters `approval_pending` *after* the most recent scheduled backup was taken, and a real disaster strikes before the next scheduled backup captures that pending state at all. In that real-world ordering, restoring from the most recent actual backup correctly reflects a world where the pending action never happened, because it hadn't happened yet when that backup was taken — which is exactly the reported symptom, and is not a bug in the failover logic itself, but a genuine RPO violation the drill's convenient sequencing never tested. This is the DR-drill version of a mistake named earlier in this same module (§6.2's "declare before you see results" discipline) turned inside out: the drill's own test sequence was implicitly declaring a best-case backup timing rather than the declared RPO's actual worst case. The fix is to redesign the drill to pause a run at a random point *between* two scheduled backups, specifically including the interval right before the next backup is due, so the test genuinely exercises the declared RPO's boundary — and, separately, to reconsider whether `paused_approval_runs`' stated RPO of "0 (must not lose an in-flight decision)" is actually achievable given a backup-on-a-schedule mechanism at all, or whether it requires a different mechanism entirely (e.g., synchronously replicating `approval_pending` state changes to a secondary region in real time, rather than relying on periodic backups for this specific category, exactly the way its near-zero RPO target in §6.2 should have implied from the start).
</details>

## 16. Checklist

- [ ] Every stateful component categorized (stateless / batch-recoverable / hard-to-batch-recover / security-sensitive) with an honest mapping to the specific artifact from this handbook that holds its state
- [ ] Per-category RPO/RTO targets declared before any drill is run, not backfilled from drill results
- [ ] Full-stack failover drill exercises a real (or realistically simulated) regional failover, not just a mocked pause/resume cycle
- [ ] Memory-staleness detection implemented and surfaced to a human via `ops-console`, not silently ignored on resumption
- [ ] Credential-state consistency verified across a failover specifically, not just under Module 2's ordinary rotation drill
- [ ] DR events detected and paged through Module 3's unified alerting taxonomy, at its top severity tier — no separate DR-specific alert system
- [ ] Drill scheduled as a recurring exercise, with the backup-timing gap explicitly randomized per §15's finding, not run only at a convenient, best-case sequence
- [ ] Honest out-of-scope section names multi-region and partial-disaster scenarios explicitly, with reasoning for their deferral
- [ ] ADR documents per-category RPO/RTO rationale against the rejected uniform-target alternative

## 17. Summary

Part 6, Module 8 proved disaster recovery at the infrastructure layer and was honest that it stopped there. This module picks up exactly where that honesty left off, and the hardest thing it proves isn't a new mechanism — it's that a guarantee tested under gentle conditions (an ordinary pause, a convenient backup-then-pause-then-fail sequence) isn't the same claim as a guarantee tested under the real, inconvenient timing a genuine disaster will actually produce. The debugging exercise's finding is the module's real thesis: a drill that always arranges for the backup to be freshly ahead of the failure it's testing will pass every time and prove almost nothing about the worst case a declared RPO is actually supposed to cover. Categorizing state honestly by how hard it is to recover, setting a real per-category target before testing against it, and then testing against that target's actual worst-case timing rather than its most convenient one — that's the discipline this module adds, and it's the same discipline, applied one more time, that every capstone in this handbook has ultimately come down to: don't let a passing test tell you more than it actually proved.

## 18. Next Steps

Reply "continue" for Module 5, or flag anything to go deeper on. (Note: Part 8's scope — deployment, monitoring, secrets, and now full-stack DR — is now substantially covered; the next module is a natural place to check whether a Part 8 capstone is warranted before moving to Part 9, or whether one further module, e.g. on compliance/data-privacy for AI systems, belongs first. Flag your preference, or say "continue" for the default next step.)
