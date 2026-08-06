# Part 9, Module 3 — Proposals, Scoping & Client Communication

## 1. Learning Objectives

By the end of this module you will be able to:

1. Write a proposal that leads with a scoped solution to the client's specific problem rather than a list of your capabilities, applying Module 1's problem-first case-study discipline to a live sales document.
2. Scope an AI engineering engagement honestly, distinguishing what's genuinely knowable up front (integration work, well-understood infra) from what requires an explicit discovery phase (model/prompt behavior against the client's actual data, which this handbook has established since Part 2 as something that can't be reasoned about in the abstract).
3. Apply Part 6, Module 4's "declare your accuracy bar before seeing results" discipline to client-facing scoping — defining success criteria for an engagement before starting it, not backfilling a definition of success after the work is done.
4. Design a discovery-phase structure (a small, paid, bounded engagement before a full commitment) specifically for AI projects, and explain why this de-risks both sides in a way that's less necessary for more predictable traditional software work.
5. Communicate technical uncertainty to a non-technical client honestly — particularly the inherent non-determinism and evaluation-dependence of LLM-based systems (Part 2, Module 8; Part 3, Module 5) — without either overwhelming them with caveats or overselling certainty you don't have.
6. Handle scope creep and mid-engagement client requests using a clear, pre-agreed change process, rather than either rigidly refusing all change or silently absorbing unbounded extra work.
7. Produce a real, reusable proposal template and a discovery-phase offering, ready to send to a qualified lead from Module 2.

## 2. Prerequisites

- Part 9, Modules 1–2 (Positioning & Portfolio; Finding Clients) — this module's proposals reference your case studies directly and are triggered by leads that passed Module 2's qualification filter.
- Part 2, Module 8 (Evaluating Models) and Part 3, Module 5 (Evaluating Full LLM-Powered Pipelines) — the statistical/evaluation discipline this module applies to client-facing success-criteria definition.
- Part 6, Module 4 (Quantization) — specifically its "set the bar before seeing results" principle, reapplied here to scoping.

## 3. Estimated Study Time

8–11 hours (theory + exercises: ~2.5 hours; mini-project: ~2 hours; production project: ~3.5–6.5 hours).

## 4. Difficulty

★★★☆☆ (3/5) — Moderate difficulty, concentrated in the honest-uncertainty-communication skill (§6.5), which is a genuinely different discipline than anything purely technical in this handbook and doesn't have a clean right answer the way a code review does.

## 5. Why This Matters

An AI engineering engagement has a scoping problem that most traditional software engagements don't: you often cannot know, with full confidence, how well a system will perform against a client's actual data and actual query patterns until you've actually built and evaluated something against them — this isn't a scoping failure, it's a structural fact this handbook has built its entire evaluation discipline around since Part 2. A proposal that promises a fixed, fully-specified outcome ("95% accuracy," delivered with total confidence, before ever seeing the client's real data) is either overselling a certainty you don't have, or it's quietly building in enough slack to always hit the number regardless of real quality — neither of which serves the client, and both of which erode trust the moment reality diverges from the promise.

This module exists because scoping and communicating an AI engagement honestly, while still being commercially confident and closeable, is a genuinely hard skill this handbook hasn't taught yet — every technical module assumed you already had a defined problem to solve; this module is about defining that problem's boundaries and communicating uncertainty about it in a way that still gets you hired, rather than either overselling or scaring the client away with excessive hedging.

## 6. Theory

### 6.1 A proposal leads with a scoped solution, not a capabilities list

Following directly from Module 1's case-study discipline: a proposal that opens with "I have experience in RAG, agents, and production AI infrastructure" is making the same mistake a broad positioning statement makes — asking the reader to do the work of connecting your capabilities to their specific problem. An effective proposal opens by restating the client's problem in their own language (proof you listened, not a template), moves immediately into a scoped, specific solution outline, and only then supports that outline with relevant case-study evidence (Module 1's artifact tracks) — capabilities in service of the solution, never capabilities as the pitch itself.

### 6.2 Honest scoping — what's knowable up front versus what needs discovery

Separate an AI engagement's scope into two genuinely different categories, and say so explicitly in the proposal rather than presenting the whole thing as equally certain:

- **Knowable up front**: integration work (connecting to the client's existing systems, per Part 1's API/backend disciplines), infrastructure requirements (Part 6), and well-understood engineering patterns this handbook has built repeatedly (a RAG pipeline's component structure, an agent's approval-gate architecture). These can be scoped with real confidence and a real fixed estimate.
- **Requires discovery**: how well retrieval and generation will actually perform against the client's specific corpus and query patterns, what the right chunking/embedding strategy is for their content (Part 4, Module 1), what accuracy/faithfulness bar is realistically achievable given their data quality — none of this is honestly knowable without actually evaluating against real data, which is the entire reason Part 2, Module 8's evaluation discipline exists in the first place. A proposal that promises a specific accuracy number for this category, sight unseen, is either guessing or padding.

The practical proposal structure: **quote the knowable-up-front work with real confidence and a fixed or near-fixed estimate; scope the discovery-dependent work as an explicit, bounded discovery phase (§6.4) with its own deliverable being a realistic assessment, not a guaranteed outcome.**

### 6.3 Setting success criteria before the work starts — reapplying Module 4's discipline

Part 6, Module 4 established a rule that's proven useful throughout this handbook: set your acceptance bar before you see results, or you invite motivated reasoning. Client engagements need the identical discipline, and skipping it is a very common, very costly freelance mistake: agreeing to "build a good RAG system" without ever pinning down what "good" means, then discovering at delivery time that the client's bar (implicit, never stated) doesn't match what was built, with neither side able to point to an agreed prior definition to resolve the disagreement.

The fix: before starting real work, agree explicitly, in writing, on the evaluation methodology (reusing `eval-framework`'s golden-dataset/scorer discipline from Part 2, Module 8, translated into client-legible terms) and the specific, measurable bar that constitutes success — ideally with an agreed process for how the bar itself gets set if it's genuinely unknowable before discovery (§6.4 exists partly to solve this: the discovery phase's deliverable can explicitly include *proposing* a realistic, evidence-based success bar for the full engagement, rather than promising one blind).

### 6.4 The discovery phase — a small, paid, bounded engagement that de-risks both sides

Given §6.2's honest split between knowable and discovery-dependent work, a standalone discovery phase is the module's central structural recommendation: a small, explicitly time-boxed, paid engagement (days, not weeks) whose deliverable is not "the finished system" but a concrete, evidence-based assessment — a working prototype against a sample of the client's real data, an honest read on achievable performance, and a scoped, confident proposal for the full engagement now grounded in real evidence rather than a pre-discovery guess.

This structurally mirrors Part 8, Module 1's shadow-traffic-before-canary discipline: don't commit to a full rollout (a full engagement) before testing against real conditions (the client's actual data) on a small, contained, reversible scale first. It de-risks the client (a small, bounded spend before a large commitment) and de-risks you (avoiding a fixed-price commitment to an outcome you couldn't have honestly predicted), and it directly resolves §6.2 and §6.3's core tension: you can genuinely, honestly set a specific success bar for the full engagement, because by the time you propose one, you've actually seen real evidence.

### 6.5 Communicating uncertainty honestly — neither overselling nor over-hedging

A genuinely hard communication skill this module has to name directly: a non-technical client generally doesn't want to hear "well, LLMs are stochastic, so we can't really promise anything specific" (true, but unhelpfully hedged, and it reads as a lack of confidence) and shouldn't be told a confidently specific number you can't actually back before discovery (false certainty that erodes trust the moment it's wrong). The resolution is to be specific about *what kind* of uncertainty exists and *how it will be resolved*, rather than either hiding it or drowning the client in it: "I can't promise a specific accuracy number before seeing your actual documents, because retrieval quality genuinely depends on your content's structure — but the discovery phase will give us a real, measured number within a week, evaluated against your own data, not a generic benchmark." This gives the client something concrete to hold onto (a process and a timeline) rather than either a false guarantee or an anxiety-inducing wall of caveats.

### 6.6 Scope creep — a pre-agreed change process, not a rigid boundary or silent absorption

Client requests that fall outside the original scope are close to inevitable in a multi-week engagement, and freelancers tend to fail in one of two opposite directions: rigidly refusing anything not in the original document (damages the relationship, feels adversarial) or silently absorbing extra work indefinitely (erodes margin and, eventually, resentment, without the client ever being aware a boundary was even crossed). The fix, agreed in the proposal itself before work starts: a simple, named change process — a new request gets a quick scoped estimate (reusing §6.1's discipline at a smaller scale) and an explicit yes/no/defer decision from the client, rather than either extreme. This is the same principle as Part 8, Module 1's per-axis versioning and rollback discipline, reapplied to project scope: change is expected and fine, as long as it's an explicit, tracked, agreed event rather than an untracked, silent one.

## 7. Mental Models

- **"A proposal proves you understood their problem before it proves you can build things — capabilities support the solution, they don't lead it."**
- **"Some scope is honestly knowable up front; some genuinely isn't until you've seen real data — say which is which, out loud, rather than pretending both are equally certain."**
- **"Set the success bar before the work starts, exactly like you set a canary threshold before you see results — or you're inviting a disagreement neither side can resolve later."**
- **"Uncertainty handled well gives the client a concrete process and timeline; uncertainty hidden or over-hedged gives them either false confidence or unwarranted anxiety."**

## 8. Visual Explanation

**Proposal structure, in order:**

```
1. Restated problem (their words, proof you listened)
2. Scoped solution outline (specific, not a capabilities list)
3. Supporting case-study evidence (Module 1's tracks, in service of #2)
4. Explicit split: knowable-now scope (fixed estimate) vs. discovery-needed scope
5. Success criteria — agreed now, or explicitly deferred to discovery's output
6. Change process for anything outside this scope
```

**Discovery phase, structurally mirroring Part 8's canary discipline:**

```
 small, paid, time-boxed discovery
        │
        ▼
 real evidence: working prototype vs. client's actual data
        │
        ▼
 scoped, evidence-based proposal for full engagement
        │           (success bar now set from real data,
        ▼            not a pre-discovery guess)
 full engagement — now genuinely de-risked for both sides
```

## 9. Recommended Resources

1. **Blair Enns — *Pricing Creativity* / *Win Without Pitching*** — a strong, freelancer/consultant-specific treatment of proposal structure and the discovery-phase idea, directly relevant to §6.1 and §6.4.
2. **Part 2, Module 8 and Part 6, Module 4 (this handbook)** — re-read both directly before drafting a real proposal; this module is a direct translation of their statistical/evaluation-discipline arguments into client-facing language.
3. **Jonathan Stark — "Diagnose before you prescribe"** (public writing/podcast, if accessible) — a concise, freelancer-specific framing of why a discovery/diagnostic phase before a full proposal builds trust rather than signaling hesitation.
4. **A sample AI/ML consulting statement of work (SOW) template** (search for publicly-shared examples from consulting firms in this space) — useful to see how the knowable-vs-discovery split (§6.2) is handled in real, professionally-drafted documents, cross-checked critically rather than copied wholesale.

## 10. Exercises

1. Take one of your Module 1 case studies and draft the opening two paragraphs of a proposal for a hypothetical client with that exact problem, following §6.1's problem-restated-first structure.
2. For a hypothetical RAG engagement, write out the explicit knowable-up-front/discovery-needed split (§6.2) — name at least three items in each category and justify the split for each.
3. Draft the specific, written success-criteria language you'd propose for an agentic-automation engagement (per `agent-core`'s work, Part 5), including how you'd propose the bar be set if it can't be fully known before discovery.
4. Write the exact client-facing paragraph you'd use to explain non-determinism/evaluation-dependence (§6.5) for a prospective client asking "can you guarantee X% accuracy?" before any discovery work has happened.
5. Design your own change-process language (§6.6) for scope creep — specific enough that a client reading it up front understands exactly what happens when a new request arrives mid-engagement.

## 11. Mini-Project

Draft a complete discovery-phase offer (§6.4) as a standalone, sendable document: what it costs, how long it takes, exactly what deliverable the client gets, and exactly how it de-risks the decision to move into a full engagement. Keep it to one page — this is meant to be a low-friction, easy-to-say-yes-to offer, and length itself is a design constraint here, not just a nice-to-have.

## 12. Production Project: A Real Proposal Template & Discovery Offer

Build a complete, reusable proposal system, ready to send to a real qualified lead from Module 2.

**Scope:**

1. **A reusable proposal template**, structured per §6.8's ordering, with placeholders for the problem restatement, scoped solution, relevant case-study references (pulled directly from Module 1's tracks), the knowable/discovery split, success criteria, and the change process — genuinely reusable, not a one-off document.
2. **The discovery-phase offer** from the Mini-Project, finalized and ready to send as a standalone document.
3. **Success-criteria language**, per Exercise 3, tailored to each of your artifact tracks, ready to adapt per engagement rather than drafted from scratch each time.
4. **Uncertainty-communication language**, per Exercise 4, refined into a short, reusable paragraph (or a small set of variants for different scenarios) you're genuinely comfortable sending to a real prospective client.
5. **Change-process language**, per Exercise 5, included in the proposal template as a standing clause.
6. **A real send**: use this template on at least one real, qualified lead from Module 2's pipeline (or a close simulation if you're not yet at that stage), and record what happens — what the client questioned, pushed back on, or responded well to — feeding directly back into refining the template.

**Documentation**: a short internal note (continuing the pattern from Modules 1–2) recording what worked and what needs revision after the first real use, treating the proposal template itself as another testable hypothesis rather than a finished artifact.

**Hands off to:** Part 9, Module 4 (Pricing & Contracts), which will formalize the actual numbers behind the fixed-scope and discovery-phase pricing this module only scoped structurally; and any real engagement this proposal wins, which becomes live practice for Part 10's interview-preparation material — a well-scoped, honestly-communicated engagement is itself a strong story for later interviews.

## 13. Practice & Interview Questions

1. Why does a proposal that opens with a capabilities list make the same mistake as a broad positioning statement, and what should it open with instead?
2. Explain the honest scoping split between knowable-up-front and discovery-dependent work in an AI engagement, and why promising a specific accuracy number for the discovery-dependent category, before discovery, is dishonest even if well-intentioned.
3. How does a discovery phase structurally resemble the shadow-traffic-before-canary pattern from Part 8, Module 1? What risk does each de-risk, for which party?
4. Design the language you'd use to answer a client's "can you guarantee 95% accuracy?" question honestly, without either overselling certainty or burying them in unhelpful hedging.
5. Why is a pre-agreed change process for scope creep better than either rigidly refusing all new requests or silently absorbing them?
6. In a mock scenario: a client pushes back on your discovery-phase proposal, wanting a single fixed-price quote for the entire engagement up front. How would you respond, using this module's reasoning rather than simply refusing?

## 14. Common Mistakes

- **Opening a proposal with a capabilities list instead of the client's restated problem.** Repeats the same reader-does-the-work mistake as a broad positioning statement.
- **Promising a specific accuracy/quality number before any discovery work against the client's real data.** Either overselling a certainty you don't have, or quietly padding the number to guarantee it's hit regardless of real quality — neither serves the client.
- **Skipping an explicit, written success-criteria agreement before starting work.** Invites an unresolvable disagreement at delivery time, with neither side able to point to an agreed prior definition.
- **Over-hedging on uncertainty to the point of sounding unconfident.** A client wants a concrete process and timeline for resolving uncertainty, not a wall of caveats with no path forward.
- **Silently absorbing scope creep, or rigidly refusing any change at all.** Both damage the engagement over time; a pre-agreed, lightweight change process avoids both extremes.
- **Treating the proposal template as a one-time document rather than a refined, reusable system.** The first real proposal's outcome is data, exactly like Module 2's lead-tracking data — use it to improve the template, not just to close or lose one deal.

## 15. Debugging Exercise

You send a well-structured discovery-phase offer, per this module's template, to a promising, qualified lead from Module 2. The client responds positively to the problem restatement and case studies but declines specifically because of the discovery phase, saying "we just need a straightforward quote, not another sales step before we can even get a price."

Form a hypothesis before reading further:

<details>
<summary>Hint 1</summary>
Re-read §6.4 and §6.5. Was the *reasoning* behind the discovery phase — why it exists, what specific risk it removes for the client — actually communicated, or did the client only see "an additional paid step before the real work," without understanding what problem it solves for them specifically?
</details>

<details>
<summary>Hint 2</summary>
Consider whether "discovery phase" was framed primarily as something that benefits you (de-risking your own estimate) or something that benefits them (a genuine, evidence-based answer to "will this actually work for us" before they commit real budget) — and whether the proposal made that framing explicit or assumed it was obvious.
</details>

<details>
<summary>Likely root cause</summary>
The discovery phase is a genuinely sound structural idea, but if the proposal presented it primarily as a process step ("first we do discovery, then we scope the full engagement") rather than explicitly as a client benefit ("this gives you a real, evidence-based answer — grounded in your own data, not a guess — before you commit to the larger engagement, and costs a small fraction of the full project to get"), a client focused on getting a number quickly will reasonably read it as friction rather than value, especially if they've received competing proposals offering an immediate, if less honest, fixed quote. This is §6.5's uncertainty-communication lesson recurring at the proposal-structure level: it's not enough to have the right process; the client needs to understand *why* it exists and what it does for *them*, stated explicitly rather than assumed self-evident. The fix isn't to abandon the discovery phase — doing so reintroduces exactly the honest-scoping problem §6.2 and §6.3 exist to solve — but to lead the discovery-phase pitch with the client-facing benefit far more explicitly, and, if a client still insists on a single fixed quote, to be honest about the trade-off in writing (a fixed quote without discovery necessarily means wider margins or padded assumptions to cover the unknowns) rather than either refusing the client outright or quietly abandoning the discipline this module is built around.
</details>

## 16. Checklist

- [ ] Proposal template opens with the client's restated problem, not a capabilities list
- [ ] Knowable-up-front and discovery-dependent scope are explicitly separated and labeled as such
- [ ] Success criteria are defined in writing before work starts, or explicitly deferred to discovery's evidence-based output
- [ ] Discovery-phase offer exists as a standalone, sendable, low-friction document, with its client-facing benefit stated explicitly, not just its process
- [ ] Uncertainty-communication language is specific about what's uncertain and how it will be resolved, avoiding both overselling and over-hedging
- [ ] A pre-agreed change process for scope creep is included as a standing clause
- [ ] At least one real send has occurred, with the outcome recorded and fed back into template refinement

## 17. Summary

An AI engagement's scoping problem is structural, not a communication failure to be smoothed over: real performance against a client's actual data genuinely can't be known with confidence before you've evaluated against it, which is the entire premise Part 2's evaluation discipline was built on. This module's contribution is a proposal and discovery-phase structure that's honest about that fact while still being commercially confident — separating what's genuinely knowable from what needs real evidence, setting success criteria before work starts rather than after, and communicating uncertainty as a concrete, resolvable process rather than either false certainty or unhelpful hedging. The debugging exercise's finding — that a sound discovery-phase structure can still fail commercially if its client-facing benefit isn't stated explicitly — is the module's sharpest lesson: being right about scoping and being persuasive about why you're right are two different skills, and this module exists to build both.

## 18. Next Steps

Reply "continue" for Module 4, or flag anything to go deeper on.
