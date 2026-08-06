# Part 10, Module 2 — System Design Interviews: From First Principles to a 45-Minute Performance

## 1. Learning Objectives

By the end of this module you will be able to:

1. Run a disciplined system-design interview framework (requirements → scale estimation → high-level design → deep dives → trade-offs) that fits a real 45-minute interview window, converting the deep, unhurried system-design instincts this handbook built across eight parts into a time-boxed performance.
2. Correctly triage which parts of a system deserve deep-dive time in an interview setting, given that a real interview can't cover a system to the depth this handbook's capstones did — recognizing that *choosing what to go deep on* is itself a signal being evaluated.
3. Apply this handbook's actual production experience — fan-out/aggregation (Part 1, Module 13), caching strategies (Part 1, Module 5), rate limiting (Part 1, Module 11), and, distinctively, real AI-system design (RAG architecture, agent orchestration, GPU serving economics) — as concrete, opinionated answers rather than generic textbook patterns.
4. Design for a DoorDash-style delivery/marketplace system specifically (order dispatch, driver-order matching, real-time tracking) using fan-out/aggregation and geospatial patterns, connecting directly to Module 1's algorithmic pattern-recognition work.
5. Design for an AI-system-specific prompt (design a RAG-powered support system; design a scalable LLM-serving platform) using this handbook's own built artifacts (`rag-engine`, `ai-infra-stack`) as a direct, credible source of real opinions — a genuine differentiator most system-design interview candidates don't have.
6. Communicate trade-offs the way this handbook's ADRs have been written since Part 1 — explicit alternatives considered and explicit reasons for rejecting them — as the primary signal an interviewer is actually listening for, more than any single "correct" architecture.
7. Run a full, timed mock system-design interview and self-evaluate against a realistic rubric.

## 2. Prerequisites

- Part 1, Module 13 (System Design Fundamentals) and Module 11 (Rate Limiting & API Design capstone) — this module's core patterns are direct reuse, not new content.
- Part 6 (AI Infrastructure) and Part 4 (RAG), broadly — the AI-specific system-design content in this module draws directly on real decisions made across both parts.
- Part 10, Module 1 — the pattern-recognition and disciplined-communication discipline from that module transfers directly to system-design communication here.

## 3. Estimated Study Time

12–16 hours (theory + framework internalization: ~3 hours; mini-project: ~2 hours; production project: ~7–11 hours, since multiple full mock interviews are the core of real readiness here).

## 4. Difficulty

★★★★☆ (4/5) — The individual technical content is largely review of this handbook's own prior work; the difficulty is entirely in compression — fitting genuine depth into a 45-minute window without either running out of time on breadth or going too shallow to demonstrate real understanding.

## 5. Why This Matters

You have something most system-design interview candidates don't: you didn't learn these patterns from a study guide, you built and audited them — `resilient-gateway`'s circuit breaker actually caught a real class of cascading failure (Part 1, Module 13), `rag-engine`'s access-control fix actually closed a real rank-leak vulnerability (Part 4, Module 9), `ai-infra-stack`'s capacity planning actually reasoned through the real VRAM/KV-cache ceiling (Part 6, Module 1). This is a genuine, rare advantage in a system-design interview — most candidates are reciting memorized patterns from a course; you have real, specific, opinionated answers backed by actual design decisions and actual documented trade-offs.

The risk this module exists to manage is almost the opposite of a typical candidate's problem: having built these systems in genuine depth, the temptation is to go too deep, too early, on a favorite topic (say, spending fifteen minutes on `rag-engine`'s access-control audit when the interviewer asked a broader question about a delivery-tracking system) and running out of time before covering the breadth an interviewer needs to see. This module's real content, beyond the technical patterns themselves, is the discipline of *time-boxed triage* — knowing what to compress, what to skip, and what to go deep on, given a fixed 45 minutes and an interviewer who's evaluating breadth-then-depth, not depth alone.

## 6. Theory

### 6.1 A time-boxed framework, adapted from this handbook's own unhurried discipline

A real system-design interview needs a fixed structure to avoid running out of time on any one phase — roughly:

```
 0-5 min:   Clarify requirements & scope (functional + non-functional)
 5-10 min:  Scale estimation (back-of-envelope: QPS, storage, bandwidth)
 10-20 min: High-level design (major components, data flow)
 20-35 min: Deep dive (interviewer-directed or candidate-chosen, 1-2 components)
 35-42 min: Trade-offs, failure modes, what you'd do differently at 10x scale
 42-45 min: Summary, open questions
```

This isn't a rigid script — interviewers redirect — but having an internalized default budget prevents the single most common interview failure mode: spending 25 minutes on requirements and high-level design and having no real time left for the deep dive, which is usually where the actual signal is generated.

### 6.2 Requirements clarification — the same discipline as Module 3's proposal scoping, compressed

Part 9, Module 3 built a discipline for separating what's genuinely knowable from what needs discovery, in a client-scoping context. The same discipline, compressed to interview timescale: ask about functional scope (what does the system actually need to do — read-heavy or write-heavy, real-time or eventually-consistent, what's explicitly out of scope) and non-functional requirements (expected scale, latency expectations, consistency requirements) *before* proposing any architecture, exactly the way Module 1's problem-solving loop insists on clarifying constraints before writing code. Skipping this and jumping straight to a remembered architecture is the system-design equivalent of Module 1's "coding without clarifying" mistake — it risks solving the wrong problem confidently.

### 6.3 Real patterns from this handbook, positioned as opinionated defaults

Where a typical candidate reaches for a generic, memorized pattern, you have real, tested opinions — use them, stated with the same specificity this handbook's ADRs have modeled throughout, not as generic textbook recall:

- **Fan-out/aggregation** (Part 1, Module 13): the direct pattern for a DoorDash-style "notify N drivers, aggregate responses, pick the best" dispatch flow — and you have real, audited experience with the circuit-breaker and timeout-budget discipline that makes fan-out safe under partial failure, not just the naive version of the pattern.
- **Caching with stampede protection** (Part 1, Module 5; extended in Part 3, Module 8 for LLM-specific caching) — a real, deeper answer than "add a cache," including the probabilistic-early-expiration and versioned-key discipline most candidates never mention.
- **Rate limiting and API design** (Part 1, Module 11) — `ai-api-platform`'s actual synthesis of Parts 0–1 gives you a concrete, complete answer for the "how do you protect this API from abuse" follow-up that many candidates handle only superficially.
- **Geospatial matching** (a DoorDash-specific extension not directly built in this handbook's prior parts, but a natural combination of Module 1's heap/graph pattern recognition with this module's system-design framing) — worth explicitly preparing as a named gap, since driver-order geospatial matching is a DoorDash-characteristic system-design question this handbook's existing artifacts don't directly cover, and preparing it deliberately (see Exercise 4) closes that gap before a real interview does.

### 6.4 AI-system-specific design — a genuine differentiator

A meaningful fraction of system-design interviews at AI-focused companies (and increasingly at companies like DoorDash building AI-powered features) now include an AI-system design prompt — "design a RAG-powered support system," "design a scalable LLM-serving platform," "design an AI agent that can take real actions safely." Here, you have a decisive advantage: you've actually built and audited exactly these systems, with real trade-offs documented, not textbook recall.

- **"Design a RAG-powered support system"** → you have `rag-engine`'s actual architecture: chunking/embedding strategy (Part 4, Module 1), hybrid search + re-ranking (Modules 3–4), access-control-as-code enforced before fusion, not after (Module 9's audited seam), and faithfulness verification (Module 8) — a substantially deeper and more specific answer than most candidates can give, because most haven't built and debugged the access-control rank-leak seam themselves.
- **"Design a scalable LLM-serving platform"** → you have `ai-infra-stack`'s real economics: prefill/decode's opposite utilization profiles (Part 6, Module 1), continuous batching and PagedAttention (Module 2), the quantization accuracy/speed trade-off with an evaluation discipline attached (Module 4), and occupancy-based load balancing with a dedicated OOM circuit breaker (Module 5) — genuinely rare depth for an interview setting.
- **"Design an AI agent that can take real, consequential actions"** → you have `agent-core`'s structural `requires_approval` gate (Part 5, Module 1) and the poisoning/correlated-failure defenses (Modules 3–4) as concrete, tested answers to the safety questions a good interviewer will ask, rather than a hand-wavy "we'd add some safety checks."

The interview discipline here isn't learning new content — it's *retrieving and compressing* real, already-built knowledge quickly under time pressure, which itself needs practice (Exercise 2).

### 6.5 Trade-offs as the primary signal — the ADR discipline, spoken aloud

This handbook has insisted, since Part 1, that an ADR's value is in the alternatives considered and rejected, not just the final decision. A system-design interview rewards exactly this same discipline, spoken aloud in real time: for every major design decision, briefly state at least one alternative you considered and why you didn't choose it ("I could use a queue-based async pattern here instead of synchronous fan-out, but given the sub-second latency requirement we clarified earlier, synchronous fan-out with a tight timeout budget is a better fit"). This single habit — stating the rejected alternative, not just the chosen path — is one of the most reliable signals separating a strong system-design interview performance from an average one, because it proves the design was reasoned rather than recited.

## 7. Mental Models

- **"Breadth first, depth where it's asked for — the biggest interview failure mode is running out of time before the deep dive even starts."**
- **"You didn't learn these patterns from a course — you built and audited them. Use the real specificity that gives you, don't flatten it into generic textbook language."**
- **"State the rejected alternative, not just the chosen path — that's the actual signal an interviewer is listening for."**
- **"AI-system design questions are where your real advantage lives — most candidates are reciting; you're recalling real, tested trade-offs."**

## 8. Visual Explanation

**The 45-minute time budget, as a default (adjustable to interviewer redirection):**

```
 0-5   │ requirements & scope
 5-10  │ scale estimation
 10-20 │ high-level design
 20-35 │ deep dive (biggest chunk — this is where signal is generated)
 35-42 │ trade-offs, failure modes, 10x-scale discussion
 42-45 │ summary
```

**Where this handbook's real artifacts map onto common prompts:**

```
 "design a delivery dispatch system"      → Part 1 Mod 13 fan-out/aggregation
                                              + geospatial matching (prepare separately)
 "design a RAG support system"             → rag-engine (Part 4), full architecture
 "design a scalable LLM-serving platform"  → ai-infra-stack (Part 6), real economics
 "design a safe AI agent"                  → agent-core (Part 5), requires_approval gate
```

## 9. Recommended Resources

1. **Alex Xu — *System Design Interview* (Vol. 1 and 2)** — the standard reference for interview-format pacing and common prompts; use it to calibrate the time-boxed framework (§6.1) against real interview norms, not as a source of technical content you don't already have from this handbook.
2. **Part 1, Module 13 and Part 4, Module 9 and Part 6, Module 8 (this handbook)** — re-read all three directly before mock-interviewing; these are your actual source material for this module's deep dives, and having them fresh matters more than any external resource.
3. **DoorDash Engineering Blog** (blog.doordash.com) — specifically posts on dispatch/logistics architecture, useful for calibrating realistic scale numbers and real constraints for §6.3's geospatial-matching preparation.
4. **"Designing Data-Intensive Applications" (Kleppmann)** — already a resource throughout this handbook since Part 1; worth a targeted re-read of the partitioning/replication chapters specifically for interview-pace fluency on those topics if they come up as a deep dive.

## 10. Exercises

1. Practice the 0–10 minute portion (requirements + scale estimation) for three different prompts, strictly timed, without proceeding to design — the goal is making this phase fast and automatic so it doesn't eat into deep-dive time.
2. Give yourself 90 seconds to summarize `rag-engine`'s access-control architecture (Part 4, Module 9) as if answering a deep-dive question — practice the compression, not just recall.
3. Practice explicitly stating a rejected alternative for three different design decisions across any system, following §6.5's discipline, until it becomes a natural habit rather than an effortful add-on.
4. Prepare a geospatial driver-order matching design specifically (§6.3's named gap) — grid-based or hash-based spatial indexing, nearest-neighbor matching, and how you'd combine it with fan-out/aggregation for a real dispatch flow.
5. Design a rejected-alternative-rich answer for "design a scalable LLM-serving platform," explicitly contrasting at least two alternatives you'd consider and reject (e.g., naive static batching vs. continuous batching; a single large model vs. model routing by task difficulty) using Part 6's real content.

## 11. Mini-Project

Run one full, strictly-timed 45-minute system-design interview on a DoorDash-style prompt ("design an order dispatch and driver-matching system"), self-recorded or with a peer, following §6.1's time budget exactly. Afterward, review: did you actually reach the deep-dive phase with enough time remaining, and did you state at least two rejected alternatives per §6.5? This produces your first real calibration data point for the Production Project.

## 12. Production Project: A Practiced System-Design Interview Repertoire

Build genuine, timed fluency across both DoorDash-style and AI-system-design prompts.

**Scope:**

1. **At least three full, timed mock interviews** covering: one DoorDash-style logistics/marketplace prompt, one RAG-system prompt, and one LLM-serving-infrastructure or agent-safety prompt — each strictly timed to 45 minutes, self-recorded or peer-run.
2. **A prepared deep-dive bank**: for each of the four artifact areas in §6.3–6.4 (fan-out/aggregation, caching, RAG architecture, LLM-serving economics, agent safety), a practiced, time-boxed (roughly 90-second) summary ready to deliver on demand, plus a longer (5–10 minute) deep-dive version for when an interviewer directs you there.
3. **Geospatial matching preparation** (Exercise 4), since this is a real, DoorDash-characteristic gap this handbook's prior artifacts don't directly cover.
4. **A self-evaluation rubric**, scoring each mock interview on: time-budget adherence, requirements-clarification quality, rejected-alternatives stated, and depth-appropriateness of the deep dive (neither too shallow nor over-running on a single component at breadth's expense).
5. **Iterative refinement**: after each mock interview, identify the specific phase (per §6.1's budget) that ran over or under, and adjust practice accordingly — the same targeted, data-driven practice discipline Module 1 built for DSA problems, now applied to system-design pacing specifically.

**Documentation**: a short running note per mock interview, tracking phase timing and rejected-alternatives count, continuing the practice-tracking discipline established in Module 1.

**Hands off to:** Part 10's remaining modules (AI/ML-specific interview preparation covering topics like ML system evaluation questions distinct from full system design; behavioral interviews and negotiation), all of which assume this module's practiced repertoire is available as real, on-demand fluency, not something to re-derive from scratch in a later module or a real interview.

## 13. Practice & Interview Questions

1. Why is running out of time before the deep-dive phase the most common and costly system-design interview failure mode, and how does a fixed time budget (§6.1) protect against it?
2. What's the risk, specific to a candidate with your actual depth of production AI-system experience, of going too deep too early on a favorite topic?
3. Explain why stating a rejected alternative for a design decision is a stronger signal to an interviewer than stating only the chosen approach, even when the chosen approach is correct.
4. Walk through how you'd use `rag-engine`'s real access-control audit (Part 4, Module 9) as a deep-dive answer to "how would you prevent this system from leaking information a user isn't authorized to see" — using the real seam you found, not a generic answer.
5. Design the geospatial matching component for a DoorDash-style dispatch system, and explain how it would combine with the fan-out/aggregation pattern from Part 1, Module 13 in a full architecture.
6. In a mock scenario: an interviewer stops you mid-high-level-design to redirect you to a specific deep dive earlier than your own time budget planned for. How do you adapt without losing the discipline of the framework?

## 14. Common Mistakes

- **Jumping to architecture before clarifying requirements and scale.** Risks confidently solving the wrong problem, the system-design equivalent of Module 1's "coding before clarifying constraints" mistake.
- **Spending too much time on a favorite, deeply-known topic at the expense of breadth.** A real risk specifically for a candidate with this handbook's depth — genuine expertise can become a time-management liability if not deliberately budgeted.
- **Presenting only the chosen design without stating any rejected alternatives.** Loses the single strongest signal of reasoned, rather than recited, design thinking.
- **Flattening real, hard-won, specific experience (the RAG access-control audit, the OOM circuit breaker) into generic textbook language.** Wastes your actual differentiator; use the real specificity.
- **Treating AI-system-design prompts as requiring new preparation, rather than recognizing you already have real, built answers.** The preparation needed here is compression and retrieval practice, not new learning.
- **Skipping deliberate preparation for DoorDash-characteristic gaps** (like geospatial matching) that this handbook's prior artifacts don't directly cover, assuming general system-design fluency will improvise a good answer live.

## 15. Debugging Exercise

In a timed mock interview, you deliver a strong, detailed high-level design and a genuinely excellent deep dive on `rag-engine`'s access-control architecture. Afterward, a peer reviewing the recording notes that you never actually asked what scale the system needed to support, and your deep dive's caching strategy assumed a scale that may not match what was actually needed.

Form a hypothesis before reading further:

<details>
<summary>Hint 1</summary>
Re-read §6.1 and §6.2. The requirements/scale-estimation phase exists specifically to ground every later decision. What happens to a technically excellent later phase if the earlier phase it should have been built on was skipped?
</details>

<details>
<summary>Hint 2</summary>
Consider why this mistake might be specifically easy for a candidate with your depth to make. Is there a temptation to skip a phase that feels like "throat-clearing" when you're eager to get to the deep content you know well?
</details>

<details>
<summary>Likely root cause</summary>
This is very likely a case of genuine expertise creating its own blind spot: because you have real, deep, ready-to-deliver content for the design and deep-dive phases, there's a natural pull to move through the earlier "clarify requirements and scale" phase quickly or superficially, since it doesn't showcase what you actually know — but skipping it means every subsequent decision, however excellent in isolation, is ungrounded in the actual problem being asked, exactly the same failure mode Module 1's problem-solving loop names for skipping constraint clarification before coding. A brilliant caching strategy for the wrong scale is a worse interview answer than a simpler, correctly-scoped one, because it signals that requirements-gathering discipline is missing under pressure — a real concern for an interviewer evaluating whether you'd do the same thing on a real ambiguous work problem. The fix isn't more technical preparation; it's deliberately practicing the discipline of spending the full, budgeted 5–10 minutes on requirements and scale, every time, treating it as non-negotiable process rather than negotiable overhead you can skip when you're confident about what comes next — the same "declare and follow the process even when you're confident about the outcome" discipline this handbook has insisted on since Part 6, Module 4's "set the bar before seeing results" principle.
</details>

## 16. Checklist

- [ ] Internalized 45-minute time budget (§6.1), adhered to in at least three timed mock interviews
- [ ] Requirements and scale-estimation phase run fully and deliberately, every time, not compressed or skipped even when confident about the design
- [ ] 90-second and 5–10 minute deep-dive versions prepared and practiced for each major artifact area (fan-out, caching, RAG, LLM-serving, agent safety)
- [ ] At least two rejected alternatives stated per mock interview, as a practiced habit
- [ ] Geospatial matching design prepared specifically as a DoorDash-characteristic gap
- [ ] Self-evaluation rubric applied to every mock interview, tracking phase timing specifically
- [ ] Real, specific artifact experience (not flattened, generic textbook language) used in deep-dive answers
- [ ] At least one mock interview run on an AI-system-design prompt, using this handbook's real built systems as source material

## 17. Summary

You don't need to learn new system-design content for this module — you need to compress, retrieve, and pace content you've already built and audited across eight parts of real engineering work, and to practice the specific discipline of a 45-minute time budget that a real interview imposes and this handbook's own unhurried capstone work never had to. The genuine risk this module manages isn't a knowledge gap; it's the opposite — real depth becoming a pacing liability if requirements-clarification gets rushed past in eagerness to reach a favorite deep dive, exactly what this module's debugging exercise caught. The discipline that actually wins a system-design interview is the same one that's run through this entire handbook since Part 1: state your reasoning, name the alternative you rejected and why, and don't skip a step just because you're confident about what comes next.

## 18. Next Steps

Reply "continue" for Module 3, or flag anything to go deeper on.
