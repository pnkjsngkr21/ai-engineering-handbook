# Part 3, Module 4: Memory — Short-Term Context Management & Long-Term Persistent Memory

> Module 3's tool-calling loop accumulates conversation state turn after
> turn without limit. This module confronts what happens when that state
> keeps growing — inside one conversation, and across many — and builds
> the two distinct memory systems every production LLM application
> eventually needs: a disciplined way to manage what fits in the context
> window *right now*, and a durable store of what should persist *after*
> the window closes.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain precisely why "memory" for an LLM is a misnomer at the model
   level — the model has no persistent state between calls — and that
   everything called "memory" in an LLM application is actually context
   management or external storage engineered by your application code.
2. Distinguish short-term context management (fitting a single, possibly
   long conversation into a finite context window) from long-term memory
   (carrying information across separate conversations/sessions), and
   explain why they require different mechanisms.
3. Implement and compare the standard short-term strategies: sliding
   window, summarization-based compaction, and hybrid approaches, with a
   grounded understanding of what each trades away.
4. Explain why naive summarization can silently destroy information the
   system will later need, and design compaction strategies that
   preserve retrieval-critical detail.
5. Design a long-term memory system with explicit write, retrieval, and
   consolidation policies, distinguishing it clearly from RAG over
   static documents (previewing Part 4) even though the retrieval
   mechanics overlap.
6. Evaluate memory-system quality using `eval-framework` (Part 2, Module
   8) rather than by inspection, including catching the specific failure
   mode of confidently-wrong recalled "memories."
7. Extend `llm-client-core`'s conversation-state handling (Module 3) with
   a working short-term compaction strategy and a minimal long-term
   memory store, wired through `smart-cache` (Part 1, Module 5) and
   `embedding-service` (Part 2, Module 4/5).

---

## 2. Prerequisites

- **Part 3, Module 3** (Tool Calling & MCP) — you need the conversation-
  state accumulation pattern (Section 6.6 of that module) as your
  starting point; this module is largely about managing that state's
  growth and its lifespan beyond one session.
- **Part 2, Module 5** (Tokenization) — you need working token-counting
  tooling (`embedding-service`'s `/estimate-cost` endpoint) since context
  budget management is measured in tokens, not messages.
- **Part 2, Module 4** (Embeddings) — long-term memory retrieval reuses
  `embedding-service` directly.
- **Part 1, Module 5** (Caching Strategies) — `smart-cache` is extended
  here for memory storage; you need its stampede-protection and
  versioned-key concepts fresh.
- **Part 2, Module 8** (`eval-framework`) — used throughout this module
  to evaluate memory quality, not just inspect it manually.

---

## 3. Estimated Study Time

**8–10 hours** (3 hours theory/reading, 5–7 hours hands-on).

---

## 4. Difficulty

★★★☆☆ (3.5/5)

The individual mechanisms (truncate, summarize, embed-and-retrieve) are
each simple in isolation. The difficulty is in policy design: deciding
*what* to forget, *when* to consolidate, and *how* to verify that a
"remembered" fact is actually true and not a hallucinated consolidation
artifact.

---

## 5. Why This Matters

Every long-running conversational product — a coding assistant that
should remember your codebase's conventions across sessions, a customer
support bot that shouldn't ask the same identifying questions every time,
a personal assistant that should recall your preferences — needs memory
that outlives a single context window and a single conversation. Get this
wrong in either direction and you get a visibly broken product: too
aggressive forgetting produces an assistant that repeats itself and
forgets what it was just told; too permissive "remembering" produces an
assistant that confidently states things that never happened, which is a
uniquely damaging trust failure because it looks exactly like
overconfidence rather than a bug.

This is also squarely where "personal knowledge management / second
brain" territory (a system you've expressed direct interest in building)
and production AI-assistant engineering meet — the memory-system design
choices in this module are the actual mechanism behind any Telegram/
WhatsApp-accessible personal knowledge assistant, not a separate concern
from it.

---

## 6. Theory

### 6.1 The model has no memory — everything here is your application's job

Say this precisely, because sloppy language here causes sloppy
architecture: **a language model call is stateless.** Every single
generation call (Module 1, Section 6.1) conditions only on the token
prefix you send *in that call*. There is no hidden state carried forward
by the provider between API calls (leaving aside provider-side prompt
caching, which is a latency/cost optimization, not a memory mechanism —
covered properly in a later Part 3 module). If the model "remembers"
something from three turns ago, it is only because your application
included that information in the prefix of the current call — exactly
the conversation-state accumulation pattern from Module 3, Section 6.6.

This means both halves of this module's title are entirely your
application's engineering responsibility:

- **"Short-term memory"** is really: *what subset of the accumulated
  conversation state do you include in each new call's prefix, given a
  finite context window?*
- **"Long-term memory"** is really: *what information do you extract,
  store externally, and re-inject into a future, entirely separate
  conversation's prefix?*

Neither is memory in the sense a human has memory. Both are context
engineering with different time horizons.

### 6.2 Short-term context management: the three standard strategies

As a single conversation grows, eventually the accumulated messages
(including, per Module 3, every tool-call request and result) exceed
either the model's hard context-window limit or your practical budget
(cost and latency both scale with input tokens — a direct callback to
Part 2, Module 5's tokenization/cost work). You need a policy for what to
do about it.

**Strategy 1 — Sliding window (truncation).** Keep only the most recent
`N` messages (or the most recent `T` tokens), dropping the oldest.
Simplest to implement, cheapest computationally (no extra model calls),
and the correct default for tasks where older context genuinely stops
being relevant (e.g., a live customer-support session about the ticket
currently open). **What it trades away**: anything genuinely important
from early in the conversation is gone, with no graceful degradation —
it's a hard cliff, not a soft fade.

**Strategy 2 — Summarization-based compaction.** Periodically (e.g.,
every N turns, or when approaching a token budget threshold), replace
the oldest chunk of raw conversation history with a single LLM-generated
summary message, then continue appending new turns after that summary.
**What it trades away**: summarization is itself an LLM generation call,
subject to the same fallibility as any other generation — it can drop
details, subtly misstate something, or (worst case) hallucinate a detail
that wasn't in the original conversation, which then gets treated as
ground truth for the rest of the session. This is a real, not
hypothetical, failure mode — Section 6.3 covers it directly.

**Strategy 3 — Hybrid (windowed + running summary).** Maintain a running
summary of everything older than the window, plus the raw, unsummarized
sliding window of the most recent `N` messages. This is the practical
default for most production conversational systems: recent context stays
at full fidelity (important because the most recent turns are usually
most load-bearing for the current response, echoing Module 1's recency/
salience point), while older context degrades gracefully into summary
rather than disappearing entirely.

**The decision isn't "which is best" in the abstract** — it's a function
of the task's actual information-retention requirements, which is a
product/systems question you should answer explicitly, not default on:

| Task property | Lean toward |
|---|---|
| Only recent turns are ever relevant (live troubleshooting) | Sliding window |
| Early details occasionally matter but exact wording doesn't | Summarization/hybrid |
| Specific facts stated early must be recoverable verbatim later (e.g., a stated legal constraint, an exact number) | Neither alone — pair with structured extraction (Section 6.4) or long-term memory, not a lossy summary |

### 6.3 Why naive summarization silently loses (or invents) information

Recall Module 1, Section 6.1: summarization is generation, and
generation is sampling from a learned distribution — it is not a
lossless compression algorithm. Two concrete failure modes to design
around, not just be aware of:

- **Silent omission**: a summarization prompt with no explicit
  instruction about *what must be preserved* will, by default, compress
  toward what a "typical helpful summary" looks like in the SFT training
  distribution (Module 1, Section 6.4's persona-conditioning logic
  applies here too) — which is not necessarily what your application
  actually needs preserved. A summary that reads as fluent and complete
  can have silently dropped the one constraint (a budget limit, a
  deadline, an explicit "don't do X") that mattered most.
- **Confabulation under compaction**: because summarization is
  generation, not extraction, there is a real risk that the summary
  states something plausible-sounding but not actually present in the
  original conversation — and once that summary replaces the raw
  history, there's no way for a later turn to detect the discrepancy,
  because the "ground truth" the model would check against has itself
  been deleted.

**Mitigation, grounded directly in Module 1 and 2's tools**: don't
summarize with a bare "summarize this conversation" prompt. Use a
structured extraction prompt (Module 2's schema-constrained output)
that explicitly enumerates the categories of information that must be
preserved verbatim (stated facts, constraints, decisions, open
questions) versus what can be safely compressed (small talk, resolved
back-and-forth, retries). This turns compaction from "ask the model to
be a good summarizer and hope" into "extract a specific, schema-defined
set of retained fields plus a free-text compressed narrative for
everything else" — a direct, disciplined application of Module 2's
three-layer reliability thinking to the memory problem.

### 6.4 Long-term memory: write, retrieve, consolidate

Long-term memory is a genuinely different system from short-term context
management, because it must survive across conversations that don't
share a context window at all. The standard architecture has three
distinct policies, each of which is a real design decision, not a
default you can skip:

**Write policy — what gets stored, and when?** Options range from
"store everything, verbatim" (simple, but produces an unbounded,
low-signal store that's expensive and noisy to retrieve from later) to
"extract only specific, structured facts via schema-constrained output"
(Module 2) at defined trigger points (end of conversation, explicit
user request, detection of a stated preference/fact). Production
systems almost always want the latter: a disciplined extraction step,
not raw logging, because retrieval quality (Section 6.4, next) degrades
badly against a noisy, unfiltered store.

**Retrieval policy — how do you find relevant stored memories for a new
conversation?** This is architecturally identical to retrieval in a RAG
system (Part 4 will cover this in full depth): embed the stored memories
(`embedding-service`, Part 2 Module 4/5) and the current conversation's
relevant context, and retrieve by similarity. **The key distinction from
Part 4's RAG**: RAG retrieves from a largely static, curated document
corpus; long-term memory retrieves from a corpus your *own system wrote*,
incrementally, over time, about a *specific user or entity* — the
retrieval mechanics overlap heavily, but the write side (Section 6.4,
above) and the consolidation side (below) are unique to memory and don't
have a RAG equivalent, because RAG's source documents aren't being
actively summarized/merged/superseded by your own system over time.

**Consolidation policy — what happens when new information conflicts
with or supersedes old stored memories?** This is the piece most
memory-system implementations skip, and it's the piece that causes the
most damaging production failures — an assistant that recalls an
outdated preference the person explicitly changed weeks ago, presented
with full confidence, is worse than an assistant with no memory at all,
because it actively erodes trust. A workable policy requires: storing
memories with timestamps and (where applicable) explicit supersession
links, retrieving by recency-weighted relevance (not pure similarity),
and periodically (or on retrieval) running a consolidation pass that
resolves direct contradictions rather than surfacing both the old and
new fact with equal confidence.

### 6.5 Evaluating memory systems — this cannot be done by inspection alone

Memory-system bugs are exactly the kind that "look fine" in a handful of
manual tests and then fail systematically in production, because the
failure modes (silent omission, confabulation, stale-preference recall)
are individually rare per-turn but compound over a long-running system.
This is precisely the class of problem `eval-framework` (Part 2, Module
8) exists for: build a golden dataset of (conversation history →
required-retained-facts) pairs and score compaction against it
automatically; build a set of (stored-memory-set → query → expected
recall) pairs and score retrieval precision/recall the same way you did
for `multimodal-rag-preview`'s precision@k (Part 2, Module 10); and
critically, build an explicit **confabulation-detection eval** —
present the system with a query about something that was *never*
actually stated or stored, and verify it doesn't fabricate a confident
memory in response. This last check is not optional: it's the single
highest-value eval for this entire module, because confabulated memory
is the failure mode most likely to reach production undetected.

---

## 7. Mental Models

1. **"The model has no memory; your application has context management
   and storage."** Every use of the word "memory" in this module is
   shorthand for engineered context assembly, never a model-level
   capability.
2. **"Summarization is generation, not compression."** It's subject to
   the same sampling fallibility as any other model output — it can omit
   or invent, and you must design compaction prompts accordingly
   (structured extraction, not "just summarize").
3. **"Long-term memory needs a write policy, a retrieval policy, and a
   consolidation policy — skipping any one of the three breaks the
   system."** Most naive implementations only build retrieval and are
   surprised when stale or contradictory memories surface.
4. **"An assistant that confidently recalls something that never
   happened is worse than one with no memory at all."** Confabulation
   detection is not a nice-to-have eval; it's the primary risk this
   whole module exists to manage.

---

## 8. Visual Explanation

**Diagram 1 — Short-term strategies compared**

```
SLIDING WINDOW:
[msg1][msg2][msg3][msg4][msg5][msg6][msg7][msg8] ← current window (recent N)
 ▲▲▲▲ dropped entirely, no trace

SUMMARIZATION:
[msg1][msg2][msg3] ──► [SUMMARY: "user stated X, decided Y..."]
                              │
                        [SUMMARY][msg4][msg5][msg6][msg7][msg8]
                        (raw msgs continue appending after the summary)

HYBRID (recommended default):
[running summary of everything before the window]
        + 
[msg_(n-4)][msg_(n-3)][msg_(n-2)][msg_(n-1)][msg_n]  ← raw, full-fidelity window
```

**Diagram 2 — Long-term memory's three policies**

```
┌────────────┐      ┌─────────────┐      ┌───────────────┐
│   WRITE     │ ───► │  RETRIEVE    │ ───► │ CONSOLIDATE    │
│ (extract,   │      │ (embed query,│      │ (resolve       │
│  not log    │      │  similarity  │      │  conflicts,    │
│  everything)│      │  search over │      │  recency-weight,│
│             │      │  stored facts)│      │  supersede old) │
└────────────┘      └─────────────┘      └───────────────┘
      │                                          │
      └──────────────── feedback loop ───────────┘
        (consolidation can trigger re-writes / merges)
```

**Diagram 3 — The confabulation-detection eval**

```
Stored memories: ["user prefers dark mode", "user works in fintech"]

Query: "What car does the user drive?"
   │
   ▼
CORRECT behavior: "I don't have that information."
WRONG (confabulated) behavior: "You mentioned you drive a Tesla."
                                        ▲
                        this is the failure eval-framework's
                        confabulation check must catch
```

---

## 9. Recommended Resources

1. **Anthropic — "Context windows" and any published guidance on long
   conversations/context management** (docs.claude.com) — read for the
   vendor's own framing of context-window limits and any built-in
   context-management features, since these evolve and directly affect
   which strategies in Section 6.2 are even necessary at a given context
   size.
2. **A well-documented open-source conversational-memory library's
   design docs** (e.g., a memory-layer project's architecture
   documentation on GitHub) — read specifically for how a real system
   implements write/retrieve/consolidate (Section 6.4) rather than
   reinventing the policy design from nothing; compare their choices
   against Section 6.4's framework critically, not just adopt them.
3. **Anthropic or OpenAI cookbook examples on conversation summarization
   and long-context handling** (github.com/anthropics/anthropic-cookbook
   or platform.openai.com/docs cookbook equivalents) — concrete,
   runnable examples of summarization-based compaction to compare against
   your own implementation.
4. **Research on "lost in the middle" / long-context retrieval
   degradation** (search for the specific paper by name once you locate
   it via the vendor docs above) — relevant because it explains an
   additional reason large raw context (as opposed to well-compacted
   context) can silently degrade answer quality even when everything
   technically "fits" in the window, which strengthens the case for
   disciplined compaction over just relying on ever-larger context
   windows.
5. **Your own `eval-framework` codebase (Part 2, Module 8)** — the most
   important "resource" for this module is your own prior work; review
   its scorer architecture before designing the memory-specific scorers
   in Section 6.5, since you want to extend, not duplicate, that
   infrastructure.

---

## 10. Exercises

1. **Statelessness, proven directly.** Make two separate, freshly-
   initialized API calls (no shared conversation state at all) where the
   second asks the model to recall something only mentioned in the
   first. Confirm it cannot, and explain why in one sentence using
   Section 6.1.
2. **Sliding window vs. summarization, empirically.** Construct a 20+
   turn synthetic conversation where a specific fact is stated once early
   on and becomes relevant again at the end. Run it under a sliding
   window (short enough to drop the fact) and under summarization-based
   compaction. Compare whether the final answer correctly uses the early
   fact under each strategy.
3. **Induce a confabulation, deliberately.** Design a compaction prompt
   with a bare "summarize this conversation" instruction (no structured
   extraction) and run it against a conversation containing several
   specific numbers/names. Check the summary for any detail that doesn't
   actually match the original — you're trying to reproduce Section
   6.3's failure mode on purpose, not avoid it, so you know what it looks
   like.
4. **Fix it with structured extraction.** Redesign the same compaction
   prompt as a schema-constrained extraction (Module 2) with explicit
   fields for "stated facts," "decisions," and "open questions." Re-run
   against the same conversation and confirm accuracy improves.
5. **Consolidation conflict.** Manually construct a small stored-memory
   set containing a stated preference, then a later, contradicting
   statement (e.g., "I prefer email" then later "actually, text me
   instead"). Write a retrieval query and confirm whether your system
   (before you build consolidation logic) surfaces the stale preference,
   the new one, or both with equal confidence — then implement
   recency-weighted consolidation and re-test.

---

## 11. Mini-Project: `memory-eval-harness`

A small standalone tool, built directly against `eval-framework` (Part 2,
Module 8), that runs three specific evaluation suites and reports a
score for each:

- **Compaction fidelity**: given a set of (long conversation, required-
  retained-facts) pairs, score whether each compaction strategy
  (sliding window / summarization / hybrid) preserves the required facts.
- **Retrieval precision/recall**: given a stored-memory corpus and a set
  of (query, expected-relevant-memories) pairs, score retrieval quality,
  reusing the precision@k scorer pattern from `multimodal-rag-preview`
  (Part 2, Module 10).
- **Confabulation rate**: given queries about information that was
  never stored, score how often the system fabricates a confident
  answer instead of correctly reporting it doesn't know.

This harness is the direct evaluation backbone for the production
project below — build it first.

---

## 12. Production Project: Short-Term Compaction + Long-Term Memory Store in `llm-client-core`

### 12.1 What you're building

1. **A `ConversationManager`** extending Module 3's conversation-state
   handling, implementing the hybrid strategy (Section 6.2) by default:
   a configurable token budget (using `embedding-service`'s
   `/estimate-cost` tokenizer, Part 2 Module 5), a raw sliding window of
   the most recent turns, and a running summary maintained via
   structured extraction (Section 6.3) rather than bare summarization —
   triggered automatically as the budget threshold is approached.

2. **A `LongTermMemoryStore`** with explicit, separately-implemented
   write, retrieval, and consolidation policies (Section 6.4):
   - **Write**: schema-constrained extraction (Module 2) of durable
     facts/preferences from a conversation, triggered at session end or
     on explicit signal, stored with timestamps.
   - **Retrieve**: embed the current conversation's relevant context
     (`embedding-service`) and perform similarity search against stored
     memories, backed by `smart-cache` (Part 1, Module 5) for the
     embedding-lookup layer specifically (not for the memories'
     source-of-truth storage — be explicit in your design about which
     store is authoritative and which is a cache).
   - **Consolidate**: recency-weighted retrieval scoring, plus an
     explicit conflict-resolution step that detects when a newly
     extracted fact contradicts a stored one and marks the old one as
     superseded rather than leaving both live with equal weight.

3. **Full `memory-eval-harness` integration**: both the compaction
   strategy and the long-term store must pass their respective
   evaluation suites, including the confabulation-rate check, as part of
   your permanent test suite — this is not optional given Section 6.5's
   argument.

4. **Observability**: emit metrics via `observability-stack` (Part 1,
   Module 4) for compaction trigger frequency, retrieval hit rate, and
   consolidation-conflict frequency — this is the operational signal
   you'll need to tune write/retrieval policy over time in a real
   deployment.

### 12.2 What this sets up for later modules

- **Part 3's Context Engineering module** will formalize the token-
  budget management this module introduces into a general-purpose
  discipline covering prompt compression beyond just conversation
  history.
- **Part 4 (RAG)** will reuse this module's retrieval mechanics directly,
  applied to static document corpora instead of self-written memory —
  you're building the retrieval half of RAG here, ahead of schedule, in
  a narrower context.
- **Part 3's capstone** (production conversational AI service) will use
  both `ConversationManager` and `LongTermMemoryStore` as core
  components, alongside the tool-calling loop from Module 3.

### 12.3 Definition of done

- `ConversationManager` correctly triggers hybrid compaction at a
  configurable token budget, verified by `memory-eval-harness`'s
  compaction-fidelity suite.
- `LongTermMemoryStore` correctly writes, retrieves, and consolidates,
  verified by the retrieval-precision and confabulation-rate suites,
  with the confabulation rate at or near zero on your golden test set.
- Consolidation correctly resolves at least one deliberately-constructed
  conflicting-preference scenario (Exercise 5) without surfacing the
  stale fact with equal confidence.
- Compaction, retrieval, and consolidation metrics are visible in
  `observability-stack`.

---

## 13. Practice & Interview Questions

1. Explain precisely why "the model remembers our last conversation" is
   an inaccurate description of what's actually happening in a
   production system, and what's actually responsible for that behavior.
2. What's the difference between short-term context management and
   long-term memory, and why can't one mechanism serve both?
3. Why is naive "summarize this conversation" a risky compaction
   strategy for information your system will later depend on, and what's
   the concrete fix?
4. Design the write, retrieval, and consolidation policies for a memory
   system supporting a personal assistant that should remember stated
   preferences across sessions. Be explicit about what happens when a
   preference changes.
5. Why is a confabulation-rate eval more important than a
   retrieval-precision eval, if you could only build one? (Answer by
   reasoning about relative damage from each failure mode, not just
   asserting an ordering.)
6. How does long-term memory retrieval differ architecturally from RAG
   over a static document corpus, given that both use embedding-based
   similarity search?
7. A user says "actually, contact me by phone, not email" mid-
   conversation, superseding an earlier stated preference. Walk through
   exactly what your system needs to do, end to end, for this change to
   correctly propagate into a future, separate conversation.

---

## 14. Common Mistakes

- **Calling context-window truncation "memory loss" as if it's an
  inherent model limitation** rather than recognizing it as a direct
  consequence of your own context-management policy choice (or lack of
  one).
- **Using bare "summarize the conversation so far" prompts for
  compaction** without structured extraction, inviting exactly the
  silent-omission and confabulation failures in Section 6.3.
- **Storing everything verbatim as "long-term memory"** instead of a
  disciplined write policy — this doesn't scale and degrades retrieval
  quality with noise.
- **Building retrieval without consolidation**, leaving contradictory
  stored facts live simultaneously and letting similarity search
  arbitrarily surface either one.
- **Never testing for confabulation.** Teams commonly build and ship
  extensive retrieval-quality evals while never testing whether the
  system fabricates confident answers about things it was never told —
  exactly the highest-damage failure mode in this whole module.
- **Conflating long-term memory with RAG** and assuming Part 4's content
  will cover this — the retrieval mechanics overlap, but write and
  consolidation policies are unique to memory and must be designed now.

---

## 15. Debugging Exercise

Users report that your assistant, after several weeks of use, has
started giving responses that reference a preference the user says they
never stated — e.g., confidently claiming "as you mentioned, you prefer
weekly summaries" when the user insists they never said this.

Write down at least three concrete, mechanistically-grounded hypotheses
using Sections 6.1, 6.3, and 6.4 (consider: could this be a write-policy
bug that extracted something incorrectly from an earlier ambiguous
statement? A consolidation bug that failed to distinguish two different
users' data? A retrieval bug surfacing a low-similarity, irrelevant
memory with high confidence? A pure confabulation with no faulty stored
memory at all?) before proposing a fix, and describe exactly what you'd
inspect in your stored-memory data and retrieval logs to distinguish
between these hypotheses.

---

## 16. Checklist

- [ ] I can state clearly why the model itself has no memory and why
      everything in this module is application-engineered.
- [ ] I can distinguish short-term context management from long-term
      memory and explain why they need separate mechanisms.
- [ ] I can explain, with a concrete example, why naive summarization
      can omit or invent information, and how structured extraction
      mitigates this.
- [ ] I can describe write, retrieval, and consolidation as three
      distinct, necessary policies for any long-term memory system.
- [ ] `memory-eval-harness` is built and includes a working
      confabulation-rate check.
- [ ] `ConversationManager` implements hybrid compaction with a
      configurable token budget, verified against the eval harness.
- [ ] `LongTermMemoryStore` implements all three policies and correctly
      resolves a deliberately-constructed conflicting-preference test
      case.
- [ ] Compaction, retrieval, and consolidation metrics are visible in
      `observability-stack`.

---

## 17. Summary

Language models have no memory at the mechanism level — every call is
stateless, conditioned only on whatever prefix your application
assembles (Module 1, Section 6.1). "Memory" in a production system is
therefore always engineered: short-term context management decides what
subset of an ongoing, possibly very long conversation fits in each new
call's token budget (sliding window, summarization, or a hybrid of both),
while long-term memory is a genuinely separate system with its own
write, retrieval, and consolidation policies, needed because information
must survive across conversations that share no context window at all.
Both are vulnerable to the same underlying risk — summarization and
extraction are generation, not lossless compression, and can silently
omit or invent information — which is why confabulation-rate evaluation,
not just retrieval-precision evaluation, is the highest-priority check
this module introduces. `llm-client-core` now has a working
`ConversationManager` for short-term compaction and a `LongTermMemoryStore`
with all three policies implemented and evaluated, directly setting up
Part 4's RAG retrieval mechanics and Part 3's own upcoming Context
Engineering module.

---

## 18. Next Steps

**Next: Part 3, Module 5 — Evaluating Full LLM-Powered Pipelines.**
Having now built prompting, structured output, tool calling, and memory
as distinct components, the natural next step is evaluating them
*together*, as one end-to-end pipeline — extending `eval-framework`
beyond single-call scoring (Part 2, Module 8) into pipeline-level
evaluation that accounts for compounding errors across multiple stages,
directly setting up the Guardrails module that follows it.

Reply "continue" for Module 5, or flag anything to go deeper on.
