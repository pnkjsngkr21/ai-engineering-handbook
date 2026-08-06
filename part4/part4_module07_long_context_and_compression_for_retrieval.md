# Part 4, Module 7: Long Context & Context Compression for Retrieval

> Part 3, Module 7 established the context window as a finite, contested
> resource and built `ContextBudgetManager` to allocate it across a
> handful of pipeline consumers. Part 4's retrieval stack — chunk-based
> hits, re-ranked results, and now knowledge-graph traversal facts — can
> easily produce far more genuinely relevant material than any context
> window can productively hold. This module confronts that volume
> problem directly: how much retrieved content is actually worth
> including, and how do you compress what doesn't fit without losing
> what matters.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain why RAG retrieval specifically stresses the context-budget
   problem harder than any other Part 3 consumer — multiple retrieval
   mechanisms (Modules 2-6) can each legitimately surface material, and
   "just increase k" is not a free way to improve recall once Part 3,
   Module 7's "lost in the middle" effect is accounted for.
2. Apply the diminishing-returns logic of retrieval — additional
   retrieved chunks contribute progressively less marginal value while
   still incurring the same marginal context cost — to make a principled
   decision about how many results to actually include, rather than
   defaulting to a fixed k.
3. Design and apply retrieval-specific compression techniques —
   extractive summarization of retrieved chunks, deduplication of
   overlapping/redundant retrieved content, and selective sub-chunk
   extraction — distinct from Part 3, Module 4's conversation-compaction
   techniques, and understand when each applies.
4. Reason correctly about long-context models as a partial, not
   complete, alternative to careful retrieval-and-compression — using
   Part 3, Module 7's "lost in the middle" evidence to avoid the
   "just use a bigger context window" trap for RAG specifically.
5. Evaluate context-compression strategies for RAG using both
   `chunking-eval-bench`-style retrieval metrics and downstream
   generation-quality metrics, since compression here risks the same
   omission/confabulation failure modes as any other lossy step in the
   pipeline.
6. Extend `ContextBudgetManager` (Part 3, Module 7) with a RAG-specific
   allocation and compression policy, integrated with `HybridRetriever`,
   `Reranker`, and `KnowledgeGraphStore`.

---

## 2. Prerequisites

- **Part 3, Module 7** (Context Engineering) — this module is a direct,
  RAG-specific extension; you need `ContextBudgetManager` and the "lost
  in the middle" argument fresh.
- **Part 4, Modules 2-6** — this module allocates budget across every
  retrieval mechanism built so far (vector, hybrid, re-ranking, metadata
  filtering, knowledge-graph traversal); you need all of them producing
  real output.
- **Part 3, Module 4, Section 6.3** (structured-extraction discipline
  for compaction) — retrieval-specific compression reuses this exact
  discipline rather than bare summarization.
- **Part 3, Module 10** (Hallucination Reduction) — compression is a
  lossy step, and this module's evaluation discipline reuses that
  module's confabulation-detection argument directly.

---

## 3. Estimated Study Time

**6–8 hours** (2 hours theory/reading, 4–6 hours hands-on).

---

## 4. Difficulty

★★★☆☆ (3.5/5)

Conceptually a direct, disciplined extension of Part 3, Module 7's
already-established framework — the difficulty is in correctly measuring
diminishing returns and compression quality specifically for retrieval
content, rather than assuming the general principles transfer without
verification.

---

## 5. Why This Matters

A common, avoidable RAG failure pattern: a system retrieves generously
(a large k, "just in case"), concatenates everything into the prompt,
and either blows the context budget entirely or — worse, and harder to
notice — silently degrades answer quality via Part 3, Module 7's "lost
in the middle" effect, because the one genuinely load-bearing chunk out
of fifteen retrieved ones ends up buried in the middle of an overlong
context. Getting retrieval volume and compression right is the specific
place where Part 4's retrieval sophistication (hybrid search,
re-ranking, knowledge graphs) actually translates into better generated
answers — sophisticated retrieval that then gets diluted by
undisciplined context assembly wastes everything built in the prior six
modules.

---

## 6. Theory

### 6.1 Why RAG specifically stresses the context-budget problem

Part 3, Module 7 identified several context consumers (system prompt,
tool definitions, memory, conversation history) competing for a finite
budget. RAG introduces a consumer that's structurally different from
the others in one important way: **the amount of potentially-relevant
material is often much larger than what any reasonable k would capture**
— unlike a fixed system prompt or a bounded conversation history, a
large corpus can easily contain 20, 50, or 100+ chunks with genuine,
non-trivial relevance to a given query, especially once Module 6's
knowledge-graph traversal is added as an additional source of grounding
facts. This means the decision "how much retrieved content should I
include" is not a fixed allocation the way Part 3, Module 7 treated
tool definitions — it's a genuine, per-query optimization problem with
real diminishing returns that must be reasoned about explicitly.

### 6.2 Diminishing returns in retrieval: why "just increase k" isn't free

Consider retrieving the top-k chunks for a query, ranked by relevance
(via `HybridRetriever`/`Reranker`, Modules 3-4). **The marginal
relevance contribution of each additional chunk decreases** as k
increases — by construction, since chunks are ranked by decreasing
relevance, the 15th chunk is (on average, and assuming your ranking is
working correctly per Module 4's evaluation) less relevant than the
1st. **But the marginal context-budget cost of each additional chunk
stays roughly constant** — the 15th chunk costs about as many tokens as
the 1st. This means past some point, each additional retrieved chunk is
paying full token cost for shrinking marginal value — and, per Part 3,
Module 7, Section 6.3's "lost in the middle" argument, may actively
*displace* or dilute the effectiveness of the genuinely load-bearing
chunks that appear earlier in the ranking, since a longer assembled
context increases the odds any single piece of content sits in a
lower-reliability middle position.

**The practical consequence**: "increase k to improve recall" is not a
free lever — it's a real tradeoff between marginal recall gain and both
direct token cost and indirect quality risk from context dilution. The
correct k is not a fixed default (k=5, k=10) applied uniformly — it
should be determined empirically, per corpus and query type, by
measuring where the marginal relevance contribution genuinely drops off
using `chunking-eval-bench`'s precision@k/recall@k curves (Module 1),
combined with downstream generation-quality measurement (Section 6.5),
not retrieval metrics alone.

### 6.3 Retrieval-specific compression techniques

**Deduplication**: multiple retrieved chunks — especially when combining
vector, keyword, and knowledge-graph-derived sources (Modules 3, 6) —
frequently contain substantially overlapping or redundant content
(the same fact stated in slightly different chunks, or a fact already
covered by the knowledge-graph traversal path also appearing in a
separately-retrieved chunk). Deduplicating before context assembly is a
lossless compression step (Part 3, Module 7, Section 6.4's "compress by
not including, before compressing lossily" principle) that should always
run first, since it removes redundancy without removing information —
detecting near-duplicate content via embedding similarity between
retrieved chunks themselves (reusing the exact mechanism from Module 2)
is the standard, reliable approach.

**Selective sub-chunk extraction**: rather than including an entire
retrieved chunk when only a portion of it is actually relevant to the
specific query, extract just the relevant span — this differs from
Module 1's chunking decision (made once, at ingestion time, for general
retrievability) by being a per-query, post-retrieval refinement.
Implemented via schema-constrained extraction (Part 3, Module 2's
discipline, exactly as in Module 6's entity extraction and Part 3,
Module 4's memory compaction) rather than a bare "extract the relevant
part" instruction, for the same omission/confabulation reasons
established repeatedly throughout this handbook.

**Extractive summarization of retrieved content**: when a retrieved
chunk is broadly relevant but verbose relative to what's actually
needed, summarize it — but, per Part 3, Module 4, Section 6.3's
argument (which applies identically here, since summarization is
generation and therefore fallible in the same ways regardless of what's
being summarized), use structured extraction specifying exactly which
categories of information must be preserved verbatim, never a bare
"summarize this chunk" instruction. **This is a genuinely lossy step**
and should be reserved for cases where deduplication and sub-chunk
extraction alone don't bring the assembled context within budget, not
applied as a default first resort.

### 6.4 Long-context models: a partial answer, not a substitute for compression discipline

As context windows have grown, a natural question is whether large
context windows make retrieval compression unnecessary — "if the model
can hold 200K+ tokens, why not just include everything the retriever
finds?" **This is exactly the trap Part 3, Module 7, Section 6.3
already warned against, now specifically relevant to RAG**: a larger
context window increases what technically *fits*, but does not change
the "lost in the middle" reliability degradation for content placed away
from the context's edges. Including 50 retrieved chunks because the
window technically holds them does not mean all 50 are reliably used by
the model during generation — the genuinely load-bearing chunks may
still be effectively ignored if they land in a low-reliability middle
position amid a large volume of lower-relevance material.

**The correct framing**: a larger context window raises the *ceiling* on
how much you could include, but the diminishing-returns argument
(Section 6.2) and the placement discipline (Part 3, Module 7, Section
6.3) still apply at whatever scale you're operating — a bigger window
is not permission to skip the discipline of deciding what's actually
worth including and where it should sit in the assembled prompt. Verify
this for your own system rather than assuming it: measure generation
quality (Section 6.5) as you increase k on a genuinely large-context
model, and confirm whether quality actually improves, plateaus, or
degrades past some point — the empirical answer, not an assumption about
window size, should drive your k and compression policy.

### 6.5 Evaluating compression: retrieval metrics are necessary but not sufficient

`chunking-eval-bench`'s precision@k/recall@k (Module 1) measures whether
the *right chunks were retrieved* — it does not measure whether the
final assembled, possibly-compressed context actually produces a good
generated answer. **Both must be measured, and they can diverge**: a
compression strategy could preserve high precision@k/recall@k (the
right chunks are still "present" in some form) while still degrading
generation quality if the compression step itself omitted or
confabulated a specific detail (Part 3, Module 4 and Module 10's risk,
applied here to retrieval-content compression specifically).

**The required evaluation**: extend your RAG golden query set (Modules
1-6's evaluation infrastructure) with an explicit generation-quality
scorer — using the rubric-based, schema-constrained judge discipline
from Part 3, Module 5 — that checks whether the *final generated
answer*, produced from the compressed context, is correct and complete,
not just whether the retrieval metrics look healthy. Report both
retrieval-quality and generation-quality metrics side by side for each
candidate compression strategy, since a strategy that improves one while
silently degrading the other is exactly the kind of hidden regression
this handbook's evaluation discipline exists to catch.

---

## 7. Mental Models

1. **"More retrieved chunks isn't free — diminishing relevance meets
   constant marginal cost, and it gets worse, not better, past some
   point once context dilution is accounted for."** Choose k empirically,
   not by default.
2. **"Deduplicate first — it's lossless. Summarize last — it's
   lossy."** Apply retrieval-compression techniques in order of
   information-loss risk, exactly as Part 3, Module 7 established for
   general context compression.
3. **"A bigger context window raises the ceiling; it doesn't remove the
   need to decide what's worth including and where it sits."** The "lost
   in the middle" effect persists at any window size.
4. **"Retrieval metrics and generation-quality metrics can diverge —
   measure both, always."** Precision@k looking healthy doesn't
   guarantee the final generated answer is actually good once
   compression is applied.

---

## 8. Visual Explanation

**Diagram 1 — Diminishing relevance vs. constant marginal cost**

```
Relevance   │██
contribution│██
per chunk   │██  ██
            │██  ██  ██
            │██  ██  ██  ▓▓  ▓▓  ░░  ░░  ░░  ░░  ░░
            └──────────────────────────────────────► k (1st...10th chunk)
                    diminishing marginal relevance

Token cost  │██  ██  ██  ██  ██  ██  ██  ██  ██  ██
per chunk   │██  ██  ██  ██  ██  ██  ██  ██  ██  ██
            └──────────────────────────────────────► k
                    roughly CONSTANT marginal cost

Past some point (e.g., k=5-6 here), each additional chunk pays full
cost for shrinking benefit — AND risks "lost in the middle" dilution
of the genuinely load-bearing early chunks.
```

**Diagram 2 — Retrieval compression techniques, ordered by information-loss risk**

```
1. DEDUPLICATION (lossless)
   Remove near-identical content across vector/keyword/graph sources
        │
2. SELECTIVE SUB-CHUNK EXTRACTION (low loss, schema-constrained)
   Extract only the query-relevant span from an otherwise-broad chunk
        │
3. EXTRACTIVE SUMMARIZATION (lossy, structured — Module 4 §6.3 discipline)
   Summarize a verbose-but-relevant chunk, preserving specified
   categories of must-keep information verbatim
        │
        ▼
   Apply IN THIS ORDER — reach for later steps only when earlier
   ones don't bring the assembled context within budget.
```

**Diagram 3 — Retrieval quality vs. generation quality can diverge**

```
Compression Strategy A:          Compression Strategy B:
  precision@k: HIGH                 precision@k: HIGH
  recall@k:    HIGH                 recall@k:    HIGH
  (retrieval metrics identical — SAME chunks retrieved in both)

  generation quality: GOOD          generation quality: DEGRADED
  (compression preserved            (compression silently omitted
   the critical detail)              the one number/fact that mattered)

           ▲
  RETRIEVAL METRICS ALONE WOULD NOT HAVE CAUGHT B's REGRESSION —
  generation-quality scoring is REQUIRED, not optional.
```

---

## 9. Recommended Resources

1. **Your own Part 3, Module 7 (Context Engineering) codebase** — the
   direct foundation this module extends; review `ContextBudgetManager`
   and the "lost in the middle" argument before designing this module's
   RAG-specific policy.
2. **Anthropic — "Long context prompting tips"** (docs.claude.com) —
   reread specifically with a RAG lens this time, since the guidance on
   structuring long contexts applies directly to assembling multiple
   retrieved chunks.
3. **Research or engineering writeups specifically on RAG context
   optimization / retrieval compression** (search for recent work on
   "context compression for RAG" or "retrieval-augmented generation
   context optimization") — read for current, evidence-based approaches
   to the diminishing-returns and compression problem specific to RAG.
4. **Your own `chunking-eval-bench` and Part 3, Module 5's rubric-based
   judge infrastructure** — the direct evaluation tools this module
   combines for the required dual (retrieval + generation quality)
   measurement.

---

## 10. Exercises

1. **Measure the diminishing-returns curve directly.** For a
   representative set of real queries against your corpus, measure
   generation quality (using a rubric-based judge, Part 3 Module 5
   style) as you vary k from 1 to 15-20, holding the retrieval
   mechanism constant. Plot the curve and identify where marginal
   quality gain flattens or reverses.
2. **Quantify deduplication's real contribution.** For queries where
   your hybrid/graph retrieval (Modules 3, 6) produces overlapping
   content from different sources, measure the token savings from
   deduplication and confirm generation quality is unaffected (or
   improved) after removing the redundancy.
3. **Test selective sub-chunk extraction against bare inclusion.** For a
   set of queries where retrieved chunks are broad relative to what's
   needed, compare full-chunk inclusion against schema-constrained
   sub-chunk extraction, measuring both token cost and generation
   quality for each.
4. **Reproduce a lost-in-the-middle regression specifically for
   retrieval content.** Construct a scenario with one genuinely critical
   retrieved chunk and several lower-relevance ones. Place the critical
   chunk in the middle of the assembled context versus near the end, and
   measure the generation-quality difference — directly connecting Part
   3, Module 7's general finding to RAG-specific content.
5. **Test the "bigger window solves it" claim directly.** Using a model
   with a genuinely large context window, measure generation quality as
   you increase k well beyond your normal operating point (e.g., k=30-50).
   Confirm empirically whether quality actually continues improving,
   plateaus, or degrades — don't assume the answer.

---

## 11. Mini-Project: `retrieval-compression-eval-bench`

A small standalone tool, combining `chunking-eval-bench`'s retrieval
metrics with Part 3, Module 5's rubric-based generation-quality
scoring, that evaluates a given (k, compression strategy) combination
against both retrieval quality and final generation quality on a real
golden query set — the direct dual-metric evaluation tool Section 6.5
requires, and the tool used to justify the production project's final
policy.

---

## 12. Production Project: A RAG-Specific Context Assembly Policy in `llm-client-core`

### 12.1 What you're building

1. **A RAG-specific extension to `ContextBudgetManager`** (Part 3,
   Module 7): an explicit, empirically-justified k selection (per query
   type, if your routing layer from Module 6 distinguishes them) based
   on the diminishing-returns curve measured in Exercise 1, rather than
   a fixed default.

2. **A retrieval-compression pipeline**, applying deduplication,
   selective sub-chunk extraction, and (only as a last resort)
   structured extractive summarization, in that order, before content
   reaches `ContextBudgetManager`'s placement-aware assembly.

3. **Placement-aware integration**: retrieved and compressed content
   must be placed within the assembled prefix according to Part 3,
   Module 7, Section 6.3's principle — the most load-bearing retrieved
   content (typically the top-ranked, post-`Reranker` results) nearest
   the end of the prefix, closest to generation.

4. **`retrieval-compression-eval-bench` integration**, measuring both
   retrieval quality and generation quality for the final chosen (k,
   compression strategy) policy, with the decision explicitly justified
   by this dual-metric evidence.

5. **Observability**: emit per-query k, compression-technique usage
   (deduplication rate, sub-chunk extraction rate, summarization rate),
   and generation-quality trend via `observability-stack`, so the
   diminishing-returns curve and compression effectiveness remain
   auditable as the corpus and query patterns evolve over time.

### 12.2 What this sets up for later modules

- **Part 4's RAG Evaluation module** extends this dual-metric (retrieval
  + generation quality) discipline into the comprehensive, end-to-end
  RAG evaluation framework.
- **Part 4's Production Architecture capstone** assembles this
  context-assembly policy as a core stage of the complete RAG pipeline,
  directly before generation.

### 12.3 Definition of done

- k selection is empirically justified per query type, backed by a
  measured diminishing-returns curve, not a fixed default.
- The compression pipeline applies deduplication, sub-chunk extraction,
  and structured summarization in the correct, information-loss-ordered
  sequence.
- Placement-aware assembly correctly positions the most load-bearing
  retrieved content nearest the end of the prefix, verified by a test.
- `retrieval-compression-eval-bench` reports both retrieval and
  generation quality for the final chosen policy.
- Per-query k, compression-technique usage, and generation-quality trend
  metrics are visible in `observability-stack`.

---

## 13. Practice & Interview Questions

1. Explain why "just increase k" is not a free way to improve RAG
   recall, addressing both the direct cost and the indirect
   context-dilution risk.
2. Describe the correct order for applying retrieval-compression
   techniques and justify the ordering by information-loss risk.
3. Why doesn't a larger context window eliminate the need for
   retrieval-compression discipline in RAG specifically?
4. Why must retrieval quality (precision@k/recall@k) and generation
   quality be measured as separate, potentially-diverging metrics for a
   RAG system?
5. Design an experiment to determine the empirically-justified k for a
   specific corpus and query type, rather than defaulting to a common
   value like k=5.
6. A teammate proposes always including the top-20 retrieved results
   "to be safe" now that the team has switched to a much larger
   context-window model. What would you want to measure before agreeing
   this is a good idea?

---

## 14. Common Mistakes

- **Defaulting to a fixed k (e.g., always k=5) uniformly**, without
  measuring the actual diminishing-returns curve for the specific
  corpus and query type.
- **Assuming a larger context window removes the need for compression
  discipline**, ignoring that "lost in the middle" persists regardless
  of window size.
- **Reaching for summarization before deduplication and sub-chunk
  extraction**, applying the lossiest compression technique first
  instead of last.
- **Measuring only retrieval metrics (precision@k/recall@k) and assuming
  generation quality follows automatically**, missing compression-
  induced omission or confabulation that retrieval metrics alone cannot
  detect.
- **Concatenating retrieved content without deduplication**, wasting
  context budget on redundant material, especially once multiple
  retrieval sources (vector, keyword, graph) are combined.
- **Ignoring placement within the assembled prefix**, treating "the
  content is included somewhere" as sufficient regardless of where it
  sits relative to the context's edges.

---

## 15. Debugging Exercise

Your `retrieval-compression-eval-bench` results show that your current
(k, compression) policy performs well on your golden query set. In
production, however, a specific class of complex query — ones requiring
synthesis across 4-5 genuinely relevant chunks rather than 1-2 — shows
noticeably degraded generation quality compared to simpler, single-fact
queries, even though retrieval metrics for these complex queries look
healthy.

Write down at least two concrete hypotheses for why generation quality
specifically degrades for multi-chunk-synthesis queries despite healthy
retrieval metrics (consider: does your placement-aware assembly treat
all "relevant" chunks equally, potentially placing some of the 4-5
genuinely necessary chunks in a middle, lower-reliability position
simply because your policy only optimizes placement for the single
top-ranked chunk? does your golden query set actually contain enough
genuine multi-chunk-synthesis cases to have surfaced this pattern during
evaluation, or is this another instance of the coverage-gap pattern seen
repeatedly across this handbook's debugging exercises?), and describe
concretely how you'd extend both your placement policy and your golden
query set to address multi-chunk-synthesis queries specifically.

---

## 16. Checklist

- [ ] I can explain why RAG's retrieval volume stresses the
      context-budget problem differently than Part 3, Module 7's other
      consumers.
- [ ] I can explain the diminishing-returns tradeoff between marginal
      relevance and marginal context cost as k increases.
- [ ] I can order retrieval-compression techniques by information-loss
      risk and justify the ordering.
- [ ] I can explain why a larger context window doesn't eliminate the
      need for compression/placement discipline.
- [ ] I understand why retrieval quality and generation quality must be
      measured separately and can diverge.
- [ ] `retrieval-compression-eval-bench` is built and reports both
      metrics for my chosen (k, compression) policy.
- [ ] k selection is empirically justified via a measured
      diminishing-returns curve, not a fixed default.
- [ ] The compression pipeline applies techniques in the correct order,
      with placement-aware assembly verified by a test.
- [ ] Per-query k, compression usage, and generation-quality metrics are
      visible in `observability-stack`.

---

## 17. Summary

RAG retrieval stresses Part 3, Module 7's context-budget problem in a
way its other consumers don't, because a large corpus can surface far
more genuinely relevant material than any k should naively include —
additional retrieved chunks carry diminishing marginal relevance while
still costing roughly constant marginal tokens, and past some empirically-
measurable point, more chunks actively risk diluting the genuinely
load-bearing ones via the "lost in the middle" effect, regardless of how
large the context window technically is. Compression should be applied
in order of information-loss risk — deduplication first (lossless),
then selective sub-chunk extraction, then structured extractive
summarization only as a last resort — and both retrieval quality and
final generation quality must be measured, since they can diverge in
ways retrieval metrics alone cannot detect. `ContextBudgetManager` now
has a RAG-specific allocation and compression policy, with k and
compression-technique usage empirically justified via
`retrieval-compression-eval-bench` rather than defaulted, and
placement-aware assembly correctly prioritizing the most load-bearing
retrieved content.

---

## 18. Next Steps

**Next: Part 4, Module 8 — RAG Evaluation (End-to-End).** With
chunking, embedding, vector search, hybrid search, re-ranking, metadata
filtering, knowledge graphs, and context compression all built, this
module extends Part 3, Module 5's pipeline-evaluation discipline into a
comprehensive, end-to-end RAG evaluation framework — covering retrieval
quality, generation quality, and their interaction, as one coordinated
evaluation surface, directly setting up Part 4's final production
architecture capstone.

Reply "continue" for Module 8, or flag anything to go deeper on.
