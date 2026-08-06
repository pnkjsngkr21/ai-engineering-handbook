# Part 3, Module 7: Context Engineering — Budget Management & Prompt Compression

> Every module so far has been adding content to the context window:
> tool definitions (Module 3), memory retrievals (Module 4), guardrail
> instructions (Module 6), and an accumulating conversation history. This
> module makes explicit and disciplined what's been implicit since
> Module 4 — the context window is a finite, shared, costed resource, and
> managing it well is its own distinct engineering skill, not a side
> effect of getting everything else right.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain the context window as a genuinely finite, multi-tenant
   resource that every pipeline stage (prompt template, tool
   definitions, memory retrieval, guardrail instructions, conversation
   history) competes for, and articulate why treating it as effectively
   unlimited is a common and costly design mistake even as context
   windows have grown large.
2. Quantify the real cost and latency impact of context size, extending
   Part 2, Module 5's tokenization/cost-estimation work into an ongoing
   operational budget rather than a one-time calculation.
3. Explain the "lost in the middle" phenomenon — degraded attention to
   information placed in the middle of a long context — mechanistically,
   and use it to make concrete placement decisions rather than treating
   all context positions as equivalent.
4. Apply prompt compression techniques (selective inclusion,
   summarization, reference-instead-of-inclusion, structured over
   free-text representations) and correctly judge which technique fits
   which content type.
5. Design a context budget allocation policy across competing consumers
   (system/tool definitions, retrieved memory, conversation history,
   guardrail scaffolding, the actual user query) rather than letting any
   one consumer grow unchecked.
6. Build a `ContextBudgetManager` in `llm-client-core` that enforces
   this allocation policy across every existing component (Modules 1-6)
   and is evaluated for both cost and quality impact via
   `eval-framework`.

---

## 2. Prerequisites

- **Part 2, Module 5** (Tokenization) — you need working tokenizer
  tooling and cost-estimation logic; this module turns that into an
  ongoing budget-management discipline.
- **Part 3, Module 4** (Memory) — the sliding-window/summarization/
  hybrid compaction strategies are a direct special case of this
  module's more general budget-allocation problem; you're generalizing
  what you already built.
- **Part 3, Module 3** (Tool Calling) — tool definitions occupy real
  context budget, and this module accounts for that cost explicitly for
  the first time.
- **Part 3, Module 6** (Guardrails) — guardrail scaffolding/instructions
  also consume budget, and any latency/cost accounting from that module
  is extended here into the full context picture.

---

## 3. Estimated Study Time

**6–8 hours** (2–3 hours theory/reading, 4–5 hours hands-on).

---

## 4. Difficulty

★★★☆☆ (3/5)

Conceptually a generalization and formalization of work you've already
done piecemeal since Module 4 — the main new difficulty is designing a
genuinely cross-cutting budget allocator that touches every existing
component without becoming an intrusive rewrite of all of them.

---

## 5. Why This Matters

As context windows have grown to hundreds of thousands of tokens, a
common and expensive misconception has taken hold: "context is basically
free now, just put everything in." This is wrong on two independent
axes that this module addresses directly — **cost** (every token you
send is billed, at every single call, and a bloated context multiplies
that cost across every turn of every conversation at production scale)
and **quality** (Section 6.3's "lost in the middle" effect means a
larger context window doesn't mean uniformly reliable attention across
all of it — stuffing in more content can measurably *degrade* answer
quality even when it technically all "fits"). Getting context
engineering right is a direct, quantifiable lever on both your unit
economics and your system's actual reliability — exactly the kind of
lever that separates engineers who've internalized production AI
economics from those who haven't, which matters both for your DoorDash-
style interview prep (systems that must be cost-efficient at scale) and
for platform-engineering roles where context/cost efficiency is a first-
class concern.

---

## 6. Theory

### 6.1 The context window as a finite, contested, multi-tenant resource

By this point in Part 3, your `llm-client-core` pipeline assembles a
prefix (Module 1, Section 6.1) from *at least* these distinct sources,
every single call:

- The system prompt / persona (Module 1).
- Tool definitions available for this call (Module 3) — and every tool
  you expose costs real tokens on *every* call, whether or not it ends
  up being used.
- Retrieved long-term memory (Module 4).
- The running conversation summary plus the raw sliding window (Module 4).
- Any guardrail-related scaffolding or few-shot examples used to steer a
  model-based check (Module 6).
- The actual current user message.

**None of these consumers has an inherent claim to unlimited space.**
Treat the context window explicitly as a shared, finite budget that
these consumers compete for, the same way you'd think about shared
memory or CPU budget across services in a resource-constrained backend
system (a direct, deliberate analogy to your existing systems-design
intuition) — except that here, "overspending" the budget doesn't crash
the process, it silently degrades quality (Section 6.3) and inflates
cost (Section 6.2), which makes it *more*, not less, dangerous, because
there's no hard failure forcing you to notice.

### 6.2 The ongoing cost of context: not a one-time calculation

Part 2, Module 5 taught you to estimate token cost for a single call.
This module's shift is recognizing that in a production system, the
*same* base context (system prompt, tool definitions, a chunk of
conversation history) gets re-sent on *every single call* in a
multi-turn or tool-calling loop (Module 3), because — recall Module 1,
Section 6.1 — every call is stateless and must re-supply its entire
prefix from scratch. A tool-calling loop with 4 rounds (Module 3, Section
6.6) doesn't send your system prompt and tool definitions once; it sends
them, in full, on every one of those 4 calls. This multiplies your
effective per-conversation token cost by the number of rounds, which is
exactly why context bloat compounds in ways that a single-call cost
estimate systematically understates.

**Provider-side prompt caching** (available on both Anthropic and OpenAI,
covered in more operational depth in a later Part 3 latency/cost module)
mitigates the *cost* of resending an identical prefix across calls, but
it does not change the *quality* dynamics in Section 6.3 — a cached
prefix is still fully present in the model's attention computation, so
caching is a cost optimization, not a substitute for actually deciding
what belongs in the context in the first place. Don't let prompt caching
become an excuse to skip budget discipline.

### 6.3 "Lost in the middle": why position, not just presence, matters

Empirical research on long-context model behavior has repeatedly found
that models are measurably more reliable at using information placed at
the *beginning* or *end* of a long context than information placed in
the *middle*, even when every piece of information is, strictly
speaking, "within the context window" and technically available to
attend to. This connects directly back to Module 1, Section 6.1's
salience point (information closer to the point of generation tends to
have outsized influence) generalized across the whole context, not just
the final instruction — content buried in the middle of a very long
prefix competes with much more surrounding material for the attention
mechanism's effective "focus," and empirically tends to lose that
competition more often than content at the edges.

**Concrete design consequence**: don't just ask "does this information
fit in the context window?" — ask "where in the context should this
information sit, given how load-bearing it is for the current turn?"
Critical, must-not-be-missed information (an explicit constraint, the
current user's actual question, a just-retrieved memory directly
relevant to answering it) belongs near the *end* of the prefix, closest
to generation (Module 1, Section 6.6's prompt anatomy already
recommended this for the task instruction specifically — this module
generalizes the principle to every content type competing for context
space). Lower-priority background material is more tolerant of a
middle position, and might be a good candidate for compression or
omission entirely rather than being included at reduced effectiveness
anyway.

### 6.4 Compression techniques and when each applies

**Selective inclusion (the first and cheapest lever)**: not every
available tool, memory, or piece of conversation history needs to be in
every call's context. Before reaching for any compression technique,
ask whether some content simply shouldn't be included at all for this
specific call — e.g., only including tool definitions actually relevant
to the current turn's likely needs (if you can determine this cheaply),
rather than every tool your system has ever registered. This is the
compression technique with zero information-loss risk, because you're
not compressing anything — you're correctly deciding it wasn't needed in
the first place.

**Summarization (already built in Module 4)**: lossy, generation-based
compression — apply Module 4, Section 6.3's structured-extraction
discipline rather than bare summarization, for exactly the reasons
already established there.

**Reference-instead-of-inclusion**: instead of including a large piece
of content (a long document, a large tool-result payload) directly in
context, include a short reference/identifier and give the model a tool
to fetch the full content only if actually needed for the current turn.
This defers the cost until the content is genuinely required, and — as
a side benefit — reduces exposure to Section 6.3's middle-of-context
degradation for content that turns out not to be needed at all this
turn. This is architecturally the seed of Part 4's RAG approach (retrieve
relevant chunks on demand, rather than stuffing an entire corpus into
context) applied here at the level of a single tool result or memory
item rather than a full document corpus.

**Structured over free-text representations**: a table or compact
structured format (closer to Module 2's schema-constrained thinking)
often conveys the same information in meaningfully fewer tokens than
equivalent prose, because natural-language phrasing carries redundant
connective/grammatical overhead a structured format doesn't need. Where
the content's actual use is programmatic or lookup-style rather than
narrative, prefer the structured form — it's both cheaper and, per
Section 6.3, often easier for the model to reliably extract specific
values from than an equivalent paragraph.

### 6.5 Designing a budget allocation policy

Given multiple context consumers (Section 6.1) and a finite total token
budget for a given call, you need an explicit allocation policy, not an
implicit "whatever's left over" approach that silently starves whichever
consumer happens to render last. A workable policy:

1. **Reserve a fixed floor for the irreducible essentials**: the system
   prompt/persona and the current user message are rarely compressible
   without direct quality loss, so budget for them first, as a fixed
   cost.
2. **Allocate a capped, prioritized budget to variable consumers**: tool
   definitions (only the relevant subset, per Section 6.4), retrieved
   memory (ranked by relevance, Module 4), and conversation history
   (per Module 4's hybrid strategy) each get an explicit, bounded
   allocation — not unlimited inclusion, and not zero — tuned based on
   which consumer's marginal contribution to answer quality is highest
   for your specific system (measured via `eval-framework`, not
   guessed).
3. **Enforce with hard checks, not soft intentions**: your
   `ContextBudgetManager` should actively measure the rendered token
   count of each consumer (reusing `embedding-service`'s tokenizer,
   Part 2 Module 5) and truncate/compress/omit according to the policy
   when the total would exceed budget — a policy that's only ever
   "aspirational" and not actually enforced in code will drift the
   moment any single consumer's typical size grows.

---

## 7. Mental Models

1. **"The context window is shared, finite, and contested — budget it
   like any other scarce resource."** Every pipeline stage from Modules
   1-6 is a consumer competing for the same limited space.
2. **"A larger context window doesn't mean everything in it is equally
   usable."** Position matters — content near the middle of a long
   prefix is measurably less reliably used than content at the edges.
3. **"Compress by not including, before compressing by summarizing."**
   Selective inclusion is lossless; summarization is lossy — reach for
   the lossless lever first.
4. **"An unenforced budget policy is not a policy."** If your allocation
   rules aren't measured and enforced in code, they will silently drift
   as any one consumer grows.

---

## 8. Visual Explanation

**Diagram 1 — Context window as contested shared resource**

```
┌──────────────────────── TOTAL CONTEXT BUDGET ─────────────────────────┐
│ [system prompt] [tool defs] [retrieved memory] [conv. history] [query] │
│    fixed           variable      variable          variable    fixed  │
│    reserve         (capped,      (capped,           (capped,   reserve│
│                    filtered)     ranked)            compacted)         │
└─────────────────────────────────────────────────────────────────────┘
        every consumer above competes for the SAME finite space —
        none has an inherent unlimited claim
```

**Diagram 2 — "Lost in the middle" and placement strategy**

```
Reliability of use, by position in a long context:

HIGH  ┤██                                                    ██
      │██                                                    ██
MED   │██                                                    ██
      │  ██                                                ██
LOW   │    ██████████████████████████████████████████████
      └──────────────────────────────────────────────────────►
       start of context      middle of context        end of context

Design consequence: put the MOST load-bearing content (the actual
current question, an explicit critical constraint, a just-retrieved
directly-relevant memory) near the END, closest to generation.
Lower-priority background material is more tolerant of a middle position
— or is a good compression/omission candidate.
```

**Diagram 3 — Compression technique selection**

```
                    Is this content needed
                    for THIS specific call?
                          │
                 ┌────────┴────────┐
                NO                YES
                 │                 │
          [OMIT — cheapest,   Is it large and only
           zero info loss]   conditionally needed?
                                    │
                          ┌─────────┴─────────┐
                         YES                   NO
                          │                     │
                 [REFERENCE, fetch      Is it more efficiently
                  on demand — Part 4's   structured than prose?
                  RAG pattern, applied           │
                  at small scale]        ┌───────┴───────┐
                                        YES              NO
                                         │                │
                              [STRUCTURED format   [SUMMARIZE via
                               — Module 2 schema]   structured
                                                     extraction —
                                                     Module 4 §6.3]
```

---

## 9. Recommended Resources

1. **Anthropic and OpenAI — prompt/context caching documentation**
   (docs.claude.com, platform.openai.com/docs) — read for the current,
   accurate mechanics of provider-side caching, since this directly
   affects the cost side of Section 6.2's analysis and changes over time.
2. **The "lost in the middle" research** (search for the specific paper
   once you locate the exact title and authors via the vendor docs or a
   direct search — it's a widely-cited empirical study on long-context
   LLM position effects) — read the experimental methodology directly,
   since the specific magnitude and shape of the effect is worth seeing
   first-hand rather than taking as an assertion.
3. **Anthropic — "Long context prompting tips"** (docs.claude.com) — the
   vendor's own operational guidance on structuring long-context prompts
   for reliability, which should directly inform your
   `ContextBudgetManager`'s placement policy (Section 6.3).
4. **Your own `embedding-service` and tokenizer tooling (Part 2, Module
   5)** — the practical foundation for every cost calculation in this
   module; review its `/estimate-cost` endpoint before building the
   budget manager.

---

## 10. Exercises

1. **Multiply the real cost.** Take one representative multi-round
   tool-calling conversation from Module 3's production project. Compute
   the total tokens actually sent across *all* rounds (not just one
   call), accounting for the fact that the system prompt and tool
   definitions are resent every round. Compare this to a naive
   single-call estimate and quantify the multiplier.
2. **Reproduce "lost in the middle," empirically.** Construct a long
   context (many thousands of tokens of filler/background content) with
   one specific, checkable fact placed at the beginning, then re-run
   with the same fact placed in the middle, then at the end. Query for
   that fact each time and compare accuracy/reliability across the three
   placements.
3. **Selective inclusion, measured.** Take a call currently sending all
   registered tool definitions regardless of relevance. Implement a
   cheap relevance filter (even a simple keyword-based one) that
   includes only likely-relevant tools for a given query, and measure
   both the token savings and whether tool-selection accuracy (Module
   3) changes.
4. **Structured vs. prose token cost.** Take one piece of content
   currently represented as free-text prose in your pipeline (e.g., a
   retrieved memory or tool result) and rewrite it as a compact
   structured format. Measure the token-count difference and check
   whether downstream answer quality is affected either way.
5. **Design a budget allocation table.** For your actual
   `llm-client-core` pipeline, write out a concrete token-budget
   allocation (fixed reserves plus capped variable allocations) across
   every consumer identified in Section 6.1, and justify each number —
   not as an arbitrary split, but tied to which consumer's marginal
   quality contribution you'd measure as highest via `eval-framework`.

---

## 11. Mini-Project: `context-cost-profiler`

A small standalone tool that, given a representative set of real
pipeline calls (using `trace-recorder` from Module 5), breaks down the
total token cost per call by consumer (system prompt, tool definitions,
memory, conversation history, query) and reports which consumer
dominates the budget across your golden test set — the direct empirical
input to designing the budget allocation table in Exercise 5.

---

## 12. Production Project: `ContextBudgetManager` in `llm-client-core`

### 12.1 What you're building

1. **A `ContextBudgetManager`** that sits in front of `PromptTemplate`
   rendering (Module 1) and the tool-calling loop's message assembly
   (Module 3), measuring the token cost of each consumer (Section 6.1)
   using `embedding-service`'s tokenizer, and enforcing an explicit,
   configurable allocation policy (Section 6.5) — including hard
   truncation/compression/omission when a consumer would exceed its
   allocation, not just logging a warning.

2. **Placement-aware assembly**: the manager should place the highest-
   priority, most load-bearing content (the current user query, any
   just-retrieved directly-relevant memory) nearest the end of the
   assembled prefix, and lower-priority background material earlier,
   directly applying Section 6.3's placement principle rather than
   simply concatenating consumers in an arbitrary or purely
   chronological order.

3. **Selective tool-definition inclusion**: extend Module 3's tool
   registration so that, for a given call, only a relevant subset of
   available tools is included in context (Section 6.4's first, cheapest
   compression lever), using at minimum a simple relevance heuristic —
   reducing wasted budget from irrelevant tool definitions on every call.

4. **`eval-framework` integration measuring both axes**: for a
   representative golden set, measure (a) total token cost/latency
   before and after `ContextBudgetManager` is applied, and (b) answer
   quality/correctness before and after — explicitly checking that
   compression saves cost *without* silently degrading quality, using
   the same trace-level evaluation discipline from Module 5.

5. **`context-cost-profiler` integrated into `observability-stack`**
   (Part 1, Module 4), so per-consumer token cost is visible on an
   ongoing basis, not just measured once during development.

### 12.2 What this sets up for later modules

- **Part 3's Caching Strategies module** (extending `smart-cache`) will
  build directly on this module's understanding of which context
  consumers are stable/cacheable (system prompt, tool definitions)
  versus inherently dynamic (conversation history, retrieved memory).
- **Part 3's Latency/Cost Optimization module** uses this module's
  per-consumer cost breakdown as its primary diagnostic input.
- **Part 4 (RAG)** directly reuses the reference-instead-of-inclusion
  pattern (Section 6.4) at a much larger scale — retrieving relevant
  chunks from a full document corpus rather than a single tool result.

### 12.3 Definition of done

- `ContextBudgetManager` enforces a real, configurable allocation policy
  across all identified consumers, with hard truncation/compression when
  a consumer exceeds its allocation.
- Placement-aware assembly is verified by a test confirming high-
  priority content lands near the end of the rendered prefix.
- Selective tool-definition inclusion measurably reduces per-call token
  cost on a golden set with tool-selection accuracy unaffected (verified
  via `eval-framework`, not assumed).
- Both cost and quality impact are measured, before/after, and neither
  regresses without an explicit, documented tradeoff decision.
- Per-consumer token cost is visible in `observability-stack` on an
  ongoing basis.

---

## 13. Practice & Interview Questions

1. Explain why "our context window is huge now, so we don't need to
   worry about prompt size" is an incomplete argument, addressing both
   the cost and the quality dimension separately.
2. Why does a multi-round tool-calling loop multiply your effective
   per-conversation token cost well beyond what a single-call estimate
   suggests?
3. Explain the "lost in the middle" phenomenon and its concrete design
   consequence for where you place a critical piece of retrieved
   information in a long context.
4. Rank, in order of preference, the compression techniques covered in
   this module, and justify the ordering in terms of information loss
   risk.
5. Design a context budget allocation policy for a system with a large
   tool registry, a long-term memory store, and long-running
   conversations. Be specific about what gets a fixed reserve versus a
   capped, prioritized allocation.
6. Why is prompt/context caching a cost optimization but not a
   substitute for context budget discipline?

---

## 14. Common Mistakes

- **Assuming a larger context window removes the need for budget
  discipline.** It doesn't remove the cost multiplier (Section 6.2) or
  the position-reliability effect (Section 6.3) — both persist regardless
  of window size.
- **Undercounting the real cost of multi-round tool-calling loops** by
  estimating cost from a single representative call instead of the full
  round-trip sequence.
- **Ignoring position when assembling context**, treating "it's included
  somewhere in the window" as sufficient regardless of where.
- **Reaching for summarization before selective inclusion.** Lossy
  compression should be a later resort, not the first lever, when simply
  not including irrelevant content is available and lossless.
- **An allocation policy that exists only as a comment or design doc**,
  not enforced in code — this drifts silently as any one consumer's
  typical size grows over time.
- **Optimizing cost without measuring quality impact.** Compression that
  reduces token spend but silently degrades answer correctness is a net
  loss, and must be checked via `eval-framework`, not assumed safe.

---

## 15. Debugging Exercise

Your `ContextBudgetManager` has been deployed for several weeks. Cost per
conversation has dropped as expected, but a specific class of user query
— ones that depend on a memory retrieved several turns earlier —
has started producing noticeably worse answers than before the manager
was introduced, even though your golden-set evals didn't catch a
regression at deployment time.

Write down at least two concrete hypotheses grounded in Sections 6.3-6.5
(consider: could the budget manager's placement policy be pushing
older-but-still-relevant retrieved memories toward the middle of the
context, where Section 6.3's effect degrades their reliability, even
though they're technically still included? could your golden set have
underrepresented this specific multi-turn-memory-dependent query
pattern, similar to the coverage gap in Module 5's debugging exercise?)
and describe concretely how you'd use `trace-recorder` and
`context-cost-profiler` together to confirm which hypothesis is correct
before changing the placement policy.

---

## 16. Checklist

- [ ] I can explain the context window as a finite, shared resource
      competed for by every pipeline stage, and why "context is basically
      free now" is an incomplete argument.
- [ ] I can explain why a multi-round tool-calling loop multiplies
      real token cost beyond a single-call estimate.
- [ ] I can explain "lost in the middle" and apply it to make concrete
      placement decisions.
- [ ] I can rank compression techniques by information-loss risk and
      choose the right one for a given content type.
- [ ] I can design an explicit, justified budget allocation policy
      across competing context consumers.
- [ ] `context-cost-profiler` reports real, per-consumer token cost
      breakdowns from actual pipeline traces.
- [ ] `ContextBudgetManager` enforces a real allocation policy with
      placement-aware assembly, verified by tests.
- [ ] Selective tool-definition inclusion measurably reduces cost without
      degrading tool-selection accuracy, verified via `eval-framework`.
- [ ] Per-consumer token cost is visible in `observability-stack` on an
      ongoing basis.

---

## 17. Summary

The context window is not an unlimited resource just because it's grown
large — it's a finite, shared budget that every pipeline stage from
Modules 1 through 6 competes for, and both the cost of sending it
(multiplied, often substantially, across every round of a multi-turn
tool-calling loop) and the reliability of using it (degraded for content
buried in the middle of a long context, regardless of technical
"fitting") demand explicit engineering discipline rather than an
assumption that bigger windows solved the problem. Compression should be
reached for in order of information-loss risk — selective inclusion
first (lossless), then reference-instead-of-inclusion, then structured
representations, then lossy summarization last — and load-bearing content
belongs near the end of the assembled prefix, closest to generation.
`llm-client-core` now has a `ContextBudgetManager` that enforces this
policy across every existing component, measured for both cost and
quality impact, with ongoing per-consumer cost visibility — directly
setting up the caching and latency/cost optimization modules that follow.

---

## 18. Next Steps

**Next: Part 3, Module 8 — Caching Strategies for LLM Calls.** Having
just identified which context consumers are stable and reusable across
calls (system prompts, tool definitions) versus inherently dynamic
(conversation history, retrieved memory), this module extends
`smart-cache` (Part 1, Module 5) with LLM-specific caching patterns —
provider-side prompt caching, response caching for repeated/similar
queries, and embedding caching for memory/retrieval lookups — built
directly on this module's consumer-stability analysis.

Reply "continue" for Module 8, or flag anything to go deeper on.
