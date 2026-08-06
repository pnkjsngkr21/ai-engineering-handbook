# Part 4, Module 8: RAG Evaluation — End-to-End

> Modules 1-7 each built and individually evaluated one stage: chunking
> and embedding (Module 1), vector search (Module 2), hybrid retrieval
> (Module 3), re-ranking (Module 4), metadata filtering (Module 5),
> knowledge graphs (Module 6), and context compression (Module 7). This
> module does for the full RAG pipeline exactly what Part 3, Module 5
> did for the LLM pipeline: prove that passing every stage individually
> doesn't guarantee the assembled system actually answers questions
> correctly, and build the trace-level, stage-attributed evaluation
> that catches what isolated testing cannot.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain why RAG-specific end-to-end evaluation is necessary beyond
   Part 3, Module 5's general pipeline-evaluation framework — RAG
   introduces a genuinely new failure surface (retrieval correctness)
   that sits upstream of and compounds with every failure mode Module 5
   already covered.
2. Distinguish and separately measure the three genuinely different
   questions a RAG evaluation must answer: was the right content
   retrieved, was the generated answer faithful to what was retrieved,
   and was the generated answer actually correct/relevant to the
   question — and explain why conflating these three hides which stage
   caused a given failure.
3. Apply RAGAS-style or equivalent RAG-specific metrics (context
   precision/recall, faithfulness, answer relevance) and understand what
   each one actually measures and what it explicitly does not.
4. Design multi-stage RAG golden datasets analogous to Part 3, Module
   5's trace-level pipeline datasets, specifying expected retrieval
   behavior and expected generation behavior as distinct, separately-
   checkable expectations.
5. Extend `eval-framework`'s `TraceScorer` (Part 3, Module 5) to
   RAG-specific traces, attributing a given wrong answer to a specific
   upstream stage (chunking, retrieval, re-ranking, filtering, graph
   traversal, compression, or generation) rather than treating "RAG
   didn't work" as a single, undifferentiated failure.
6. Build a comprehensive, coordinated RAG acceptance-test suite,
   assembling every evaluation tool built across Part 4 (`chunking-
   eval-bench`, `ann-tuning-bench`, `hybrid-search-eval-bench`,
   `rerank-eval-bench`, `access-control-security-bench`, `graph-
   extraction-eval-bench`, `retrieval-compression-eval-bench`) into one
   release-readiness gate.

---

## 2. Prerequisites

- **Part 4, Modules 1-7 in full** — this module's evaluation suite
  assembles and extends the evaluation tooling from every prior Part 4
  module; all of them must be working.
- **Part 3, Module 5** (Evaluating Full LLM-Powered Pipelines) — you need
  the trace-level, stage-attributed evaluation discipline and the
  correlated/compounding-error argument fresh, since this module applies
  the identical argument to the retrieval stack.
- **Part 3, Module 10** (Hallucination Reduction, Section 6.3 on
  grounding faithfulness) — the faithfulness metric in this module
  directly reuses that module's citation-verification discipline.

---

## 3. Estimated Study Time

**8–10 hours** (3 hours theory/reading, 5–7 hours hands-on).

---

## 4. Difficulty

★★★★☆ (4/5)

Not conceptually new relative to Part 3, Module 5 — but genuinely
demanding in practice, because a RAG pipeline has more stages and more
compounding-error surface than the LLM pipeline Module 5 originally
evaluated, and correctly attributing a failure to the right one of
seven-plus stages requires real discipline.

---

## 5. Why This Matters

"RAG isn't working well" is one of the most common and least actionable
complaints in production AI systems, precisely because a RAG pipeline
has so many stages that a wrong answer could stem from almost any of
them — bad chunking, a wrong embedding model, retrieval that missed the
right document, a re-ranker that demoted the right chunk, an
over-aggressive metadata filter, a knowledge-graph extraction error, or
compression that dropped the critical detail. Without stage-attributed
evaluation, every "RAG isn't working" report turns into an unfocused,
expensive fishing expedition across all seven-plus stages. This module
is what converts that fishing expedition into a systematic diagnostic
process — exactly the kind of evaluation maturity that separates a RAG
demo from a RAG system someone can actually operate, debug, and improve
over time.

---

## 6. Theory

### 6.1 Why RAG needs its own evaluation layer, not just Part 3, Module 5's framework reused as-is

Part 3, Module 5 built pipeline-level evaluation for a system with
stages like prompt rendering, tool calling, and memory retrieval — none
of which involve a genuinely large, external, imperfectly-searchable
corpus. RAG adds an entirely new failure surface Module 5's original
framework never had to account for: **retrieval correctness against a
large corpus is itself fallible in ways a memory lookup or a tool call
generally isn't** — the right document might exist in the corpus but
never get retrieved (a Module 2/3 recall failure), might get retrieved
but demoted by re-ranking (a Module 4 failure), might get incorrectly
excluded by an overly strict metadata filter (a Module 5 failure), or
might require relational reasoning no chunk-based mechanism can provide
at all (a Module 6 gap). **This retrieval-correctness failure surface
sits entirely upstream of generation**, and it compounds with every
downstream failure mode Module 5 already covered (structured-output
failures, hallucination, guardrail issues) — meaning RAG evaluation must
extend, not replace, Module 5's trace-level discipline, adding an
entirely new set of upstream stages to attribute failures to.

### 6.2 Three genuinely different questions a RAG evaluation must answer

Following the RAGAS framework's well-established decomposition (and
directly extending Part 3, Module 5's "specify intermediate expectations,
not just final answers" argument), a complete RAG evaluation must
separately measure:

- **Context precision/recall — was the right content retrieved?** This
  is exactly `chunking-eval-bench`'s precision@k/recall@k (Module 1),
  now measured as part of the full pipeline trace rather than in
  isolation — did the retrieval stack (vector, hybrid, re-ranking,
  filtering, graph traversal) actually surface the chunks/facts
  genuinely needed to answer the query?
- **Faithfulness — is the generated answer actually grounded in what was
  retrieved?** This is distinct from correctness: a generated answer can
  be unfaithful (making a claim not actually supported by the retrieved
  content, i.e., the exact "unfaithfulness" risk flagged in Part 3,
  Module 10, Section 6.3) even when the correct content *was* retrieved
  — the retrieval stage did its job, but generation failed to use it
  properly, exactly the "right retrieval, wrong generation" case that
  parallels Part 3, Module 5's "right answer, wrong reason" pattern.
- **Answer relevance/correctness — does the generated answer actually
  address the question, and is it correct?** This is the final,
  user-facing quality check — but critically, a *high* answer-relevance
  score alongside *low* context precision/recall or faithfulness is
  exactly the "right answer, wrong reason" case (a lucky guess, or a
  correct answer that happened to come from the model's parametric
  memory rather than genuine grounding) that a final-answer-only
  evaluation would incorrectly count as a full success.

**These three metrics answer genuinely different questions, and a
failure in any one — or a suspicious mismatch between them (e.g., high
answer-relevance despite low context-recall) — is diagnostic
information about *which* stage needs attention**, precisely the
stage-attribution discipline this whole module exists to formalize for
RAG.

### 6.3 Faithfulness evaluation: reusing citation-verification, not reinventing it

Measuring faithfulness — does the generated answer's claims actually
follow from the retrieved content — is architecturally identical to Part
3, Module 10, Section 6.6's verification-pass mechanism: extract
individual factual claims from the generated answer (schema-constrained,
Part 3 Module 2), then for each claim, check whether it's actually
supported by the retrieved chunks/graph facts that were supplied as
context, using a narrow, rubric-based judge call (never an open-ended
"does this seem faithful" judgment, for exactly the reasons Part 3,
Module 5 and Module 6 established for rubric-based judging generally).
**Do not build a separate faithfulness-checking mechanism from
scratch** — this module's faithfulness scorer is a direct, specific
application of infrastructure you already built, applied to RAG's
retrieved-content-as-grounding-material case specifically.

### 6.4 Stage-attributed RAG trace evaluation

Extending Part 3, Module 5's `TraceScorer`/`PipelineGoldenDataset`
structure, a RAG-specific golden test case should specify expected
behavior at each of these stage boundaries, not just a final answer:

```
{
  "query": "...",
  "expected_retrieved_chunk_ids": [...],       // Module 1-3: was the
                                                 // right content found?
  "expected_reranking_top_result": "...",      // Module 4: did re-ranking
                                                 // correctly prioritize it?
  "expected_metadata_filter_behavior": "...",  // Module 5: were the
                                                 // right access/recency
                                                 // constraints applied?
  "expected_graph_traversal_path": [...],      // Module 6, if applicable:
                                                 // was relational reasoning
                                                 // correctly invoked?
  "expected_compression_preserved_facts": [...], // Module 7: did
                                                    // compression keep
                                                    // the critical detail?
  "expected_faithfulness": "all claims grounded",
  "expected_answer_content": "..."
}
```

A wrong final answer against this test case can now be attributed to a
*specific* stage — did the wrong chunks get retrieved at all (Module
1-3), did the right chunk get retrieved but demoted (Module 4), did a
filter wrongly exclude it (Module 5), did compression drop the critical
detail (Module 7), or did generation fail to faithfully use correctly-
supplied content (Module 10's faithfulness check)? This is exactly the
diagnostic value Section 6.1 argued RAG evaluation specifically needs,
and it directly answers the "RAG isn't working, why" question with a
specific, actionable stage rather than an undifferentiated shrug.

### 6.5 Compounding errors across a longer pipeline — the same argument, more stages

Part 3, Module 5's core argument — that per-stage pass rates don't
predict end-to-end correctness, because errors compound and correlate on
hard inputs rather than failing independently — applies with even more
force here, since RAG's pipeline now has significantly more stages than
the original LLM pipeline Module 5 evaluated. **The same discipline
applies without modification**: measure end-to-end, stage-attributed
trace correctness directly, never infer it from the average of Module
1-7's individual evaluation-tool pass rates, and explicitly test for the
"right final answer despite a wrong intermediate stage" case (a lucky
recovery that will not generalize) using the same rigor Part 3, Module 5
insisted on for the LLM pipeline.

---

## 7. Mental Models

1. **"RAG evaluation extends pipeline evaluation with an entirely new
   upstream failure surface: retrieval correctness against a large,
   imperfectly-searchable corpus."** Everything Part 3, Module 5 already
   established still applies; this module adds stages, not a new
   philosophy.
2. **"Context precision/recall, faithfulness, and answer relevance are
   three different questions — a failure or mismatch among them is
   diagnostic, telling you which stage to look at."** Never collapse
   them into one aggregate "RAG quality" score.
3. **"Faithfulness checking is Part 3, Module 10's verification-pass
   mechanism, reused — not reinvented."** Don't build parallel
   infrastructure for a problem you already solved.
4. **"A wrong final answer in RAG could stem from any of seven-plus
   stages — stage-attributed evaluation is what makes 'RAG isn't working'
   into an actionable diagnosis instead of a shrug."**

---

## 8. Visual Explanation

**Diagram 1 — RAG adds an upstream failure surface to Part 3's pipeline**

```
PART 3's PIPELINE (Module 5):
  [prompt] → [tools] → [memory] → [generation] → [guardrails]
       (failure surface: prompting, tool selection, memory retrieval,
        generation, guardrail correctness)

PART 4's RAG PIPELINE (this module):
  [chunking+embedding] → [vector/hybrid search] → [re-ranking] →
  [metadata filtering] → [graph traversal, if applicable] →
  [context compression] →  ⤵
                            [PART 3's PIPELINE, now fed by retrieval]
                            [prompt] → [generation] → [guardrails]

  NEW upstream failure surface (this module's focus):
  chunking, embedding, retrieval recall, re-ranking precision,
  filter correctness, graph extraction/traversal, compression fidelity
  — ALL compound with everything Part 3, Module 5 already covers.
```

**Diagram 2 — Three questions, three metrics, diagnostic when they diverge**

```
                Context           Faithfulness      Answer
                precision/recall                    relevance/correctness
                (was the right    (is the answer     (does the answer
                 content found?)   grounded in it?)   actually address
                                                       the question?)

Case A: HIGH        HIGH               HIGH           → genuine success
Case B: LOW         —                  —              → RETRIEVAL failed
                                                          (Modules 1-6)
Case C: HIGH        LOW                —              → GENERATION didn't
                                                          faithfully use
                                                          correct content
                                                          (unfaithfulness)
Case D: LOW         —                  HIGH            → "right answer,
                                                          wrong reason" —
                                                          lucky parametric-
                                                          memory guess,
                                                          NOT genuine RAG
                                                          success
```

**Diagram 3 — Stage-attributed trace evaluation**

```
Golden test case specifies expectations at EVERY stage boundary:

  [chunking/embed] → expected chunks present in corpus?      ✓/✗
  [retrieval]       → expected chunk IDs in top-k?            ✓/✗
  [re-ranking]      → expected chunk in top position?         ✓/✗
  [metadata filter] → correct inclusion/exclusion applied?    ✓/✗
  [graph traversal] → correct path followed, if applicable?   ✓/✗
  [compression]     → critical facts preserved?               ✓/✗
  [faithfulness]    → generated claims grounded?               ✓/✗
  [final answer]    → matches expected content?                ✓/✗

A single ✗ anywhere in this chain PINPOINTS the failing stage —
instead of one undifferentiated "wrong answer" result.
```

---

## 9. Recommended Resources

1. **The RAGAS framework paper and documentation** (search for the exact
   title/repo) — read directly for the context precision/recall,
   faithfulness, and answer relevance metric definitions, since Section
   6.2's decomposition follows this well-established framework closely,
   and reading the source is more valuable than a secondary summary.
2. **Your own Part 3, Module 5 (`TraceScorer`/`PipelineGoldenDataset`)
   and Module 10 (verification-pass/faithfulness) codebases** — the
   direct infrastructure this module extends; review both before
   building the RAG-specific stage-attribution scorer.
3. **A well-cited industry writeup on RAG evaluation challenges in
   production** (search for a recent, well-regarded engineering blog
   post on evaluating production RAG systems) — read for real-world
   accounts of the stage-attribution difficulty this module addresses.
4. **Every evaluation tool you've built across Part 4** (`chunking-
   eval-bench`, `ann-tuning-bench`, `hybrid-search-eval-bench`,
   `rerank-eval-bench`, `access-control-security-bench`, `graph-
   extraction-eval-bench`, `retrieval-compression-eval-bench`) — the
   actual primary resource for this module is assembling your own prior
   work, not new external material.

---

## 10. Exercises

1. **Construct a "right answer, wrong reason" RAG case, deliberately.**
   Build a test query where the model, using only its parametric memory
   (no actual grounding needed), would produce a correct-looking answer
   even if retrieval completely failed. Confirm your evaluation suite,
   if it only checked the final answer, would incorrectly count this as
   a full success — then confirm your context-precision/recall and
   faithfulness metrics correctly flag it as a retrieval/faithfulness
   failure despite the superficially correct answer.
2. **Construct an unfaithfulness case, deliberately.** Build a test query
   where the correct content genuinely is retrieved, but deliberately
   check whether generation might still produce an unsupported claim
   (reusing Part 3, Module 10's techniques for inducing this). Confirm
   your faithfulness scorer catches it even though context precision/
   recall look healthy.
3. **Build one full stage-attributed golden test case**, per Section
   6.4's structure, covering every stage boundary from chunking through
   final answer, for a real query against your corpus.
4. **Deliberately break one specific stage and confirm correct
   attribution.** Introduce a deliberate bug at one specific stage (e.g.,
   an overly strict metadata filter excluding a needed document) and
   confirm your stage-attributed evaluation correctly identifies that
   specific stage as the failure point, rather than only reporting a
   generic wrong-answer result.
5. **Run the full compounding-error experiment for RAG.** Using 15-20
   deliberately difficult/ambiguous queries, measure each Part 4 module's
   evaluation tool pass rate independently, then measure genuine
   end-to-end stage-attributed trace correctness. Compare the naive
   product of independent pass rates against the measured end-to-end
   result and quantify the gap, exactly replicating Part 3, Module 5's
   original experiment for the (longer) RAG pipeline.

---

## 11. Mini-Project: `rag-trace-visualizer`

A small tool, extending Part 3, Module 5's `trace-recorder`/
`pipeline-trace-visualizer` pattern, that renders a full RAG request
trace stage-by-stage — chunking/retrieval hits, re-ranking scores,
filter decisions, graph traversal path (if applicable), compression
decisions, faithfulness check results, and final answer — as a readable,
inspectable visualization, directly analogous to Part 3, Module 12's
`pipeline-trace-visualizer` but extended for RAG's additional upstream
stages. This is both a genuinely useful debugging tool for this module's
integration work and a strong portfolio artifact.

---

## 12. Production Project: Comprehensive RAG Evaluation Framework

### 12.1 What you're building

1. **A `RAGTraceScorer`**, extending Part 3, Module 5's `TraceScorer`,
   supporting stage-attributed golden test cases per Section 6.4's
   structure, covering every Part 4 stage (chunking/embedding, vector/
   hybrid retrieval, re-ranking, metadata filtering, graph traversal,
   compression, faithfulness, final answer).

2. **A `FaithfulnessScorer`**, directly reusing Part 3, Module 10's
   verification-pass/claim-extraction mechanism, applied specifically to
   RAG-retrieved grounding material.

3. **A comprehensive RAG golden dataset**, including at minimum: cases
   testing pure semantic-relevance retrieval, exact-match retrieval
   (Module 3), genuine multi-hop relational queries (Module 6),
   deliberately-constructed "right answer, wrong reason" cases (Exercise
   1), and deliberately-constructed unfaithfulness cases (Exercise 2).

4. **A coordinated acceptance-test suite**, assembling every evaluation
   tool built across Part 4 — `chunking-eval-bench`, `ann-tuning-bench`,
   `hybrid-search-eval-bench`, `rerank-eval-bench`, `access-control-
   security-bench` (as a CI-blocking security gate, per Part 4, Module
   5's severity), `graph-extraction-eval-bench`, `retrieval-compression-
   eval-bench`, and this module's `RAGTraceScorer`/`FaithfulnessScorer`
   — into one release-readiness CI gate, directly extending Part 3,
   Module 12's capstone acceptance-test assembly pattern.

5. **`rag-trace-visualizer` integration** for debugging and portfolio
   use.

### 12.2 What this sets up for later modules

- **Part 4's Production Architecture capstone** uses this comprehensive
  evaluation framework as its primary release-readiness gate, exactly as
  Part 3, Module 12's capstone used its assembled acceptance-test suite.
- **Part 5 (Agents)**, when it incorporates retrieval as one of an
  agent's available tools, will extend this same stage-attribution
  discipline to agent-invoked retrieval calls.

### 12.3 Definition of done

- `RAGTraceScorer` correctly attributes failures to specific stages on a
  test set including at least one deliberately-injected, single-stage
  bug (Exercise 4), verified to correctly identify that stage.
- `FaithfulnessScorer` correctly catches the deliberately-constructed
  unfaithfulness case (Exercise 2) despite healthy context precision/
  recall.
- The comprehensive golden dataset includes semantic, exact-match,
  multi-hop, "right answer wrong reason," and unfaithfulness test
  categories.
- The full, coordinated acceptance-test suite (all Part 4 evaluation
  tools plus this module's additions) runs as one CI gate, with
  `access-control-security-bench` treated as a blocking security check.
- `rag-trace-visualizer` correctly renders a full RAG trace across every
  stage.

---

## 13. Practice & Interview Questions

1. Explain why RAG evaluation needs to extend, not just reuse unchanged,
   Part 3, Module 5's general pipeline-evaluation framework.
2. Describe context precision/recall, faithfulness, and answer
   relevance as three distinct questions, and give a concrete example
   where a system could score well on one while failing another.
3. Why is faithfulness evaluation architecturally the same mechanism as
   Part 3, Module 10's verification pass, and why shouldn't you build a
   separate mechanism for it?
4. Describe the "right answer, wrong reason" failure mode specifically
   in a RAG context, and explain why a final-answer-only evaluation
   cannot catch it.
5. If a RAG system's answer is wrong, walk through, in order, the
   specific upstream stages you'd check using stage-attributed
   evaluation, and what a failure at each stage would look like.
6. Why does the compounding-error argument from Part 3, Module 5 apply
   with even more force to a RAG pipeline than to the original LLM
   pipeline it was built for?

---

## 14. Common Mistakes

- **Evaluating RAG systems only on final-answer correctness**, missing
  both the "right answer, wrong reason" case and the specific stage
  responsible for any genuine failure.
- **Treating "RAG isn't working" as one undifferentiated problem**
  instead of using stage-attributed evaluation to localize the actual
  failing stage.
- **Building a separate, from-scratch faithfulness-checking mechanism**
  instead of reusing Part 3, Module 10's already-built verification-pass
  infrastructure.
- **Assuming high context precision/recall implies a good final
  answer**, ignoring that generation can still fail to faithfully use
  correctly-retrieved content.
- **Never constructing deliberate "right answer, wrong reason" or
  unfaithfulness test cases**, meaning your evaluation suite may never
  actually exercise these specific, important failure modes.
- **Running each Part 4 module's evaluation tool independently without
  ever assembling them into one coordinated, comprehensive suite**,
  missing the whole-pipeline compounding-error risk Section 6.5
  describes.

---

## 15. Debugging Exercise

Your comprehensive RAG acceptance-test suite has been passing
consistently, including the faithfulness and stage-attribution checks.
A production incident reveals a case where the system gave a
confidently wrong answer to a query type not represented anywhere in
your golden dataset — a query combining an exact-match term (Module 3)
with a genuine multi-hop relational requirement (Module 6)
simultaneously, a combination your test categories never covered
together.

Write down at least two concrete hypotheses for why this specific
combination might fail even though each capability (exact-match
retrieval, multi-hop graph traversal) passes its own individual tests
(consider: is this a routing-layer failure — does your Module 6 routing
classifier correctly recognize a query needing *both* mechanisms
simultaneously, or does it force a binary choice between chunk-based and
graph-based retrieval when the query actually needs both? is this
simply a golden-dataset coverage gap for combined query types, echoing
the same pattern from nearly every debugging exercise in this
handbook?), and describe concretely how you'd extend your comprehensive
golden dataset and routing logic to handle queries requiring multiple
retrieval mechanisms simultaneously, rather than assuming routing is
always a mutually-exclusive choice.

---

## 16. Checklist

- [ ] I can explain why RAG evaluation extends, rather than replaces,
      Part 3, Module 5's pipeline-evaluation framework.
- [ ] I can distinguish context precision/recall, faithfulness, and
      answer relevance as three separate questions and give an example
      where they diverge.
- [ ] I understand why faithfulness checking reuses Part 3, Module 10's
      verification-pass mechanism rather than requiring new
      infrastructure.
- [ ] I can describe the "right answer, wrong reason" failure mode for
      RAG specifically and why final-answer-only evaluation misses it.
- [ ] `RAGTraceScorer` correctly attributes a deliberately-injected
      single-stage failure to the correct stage.
- [ ] `FaithfulnessScorer` correctly catches a deliberately-constructed
      unfaithfulness case.
- [ ] The comprehensive golden dataset covers semantic, exact-match,
      multi-hop, "right answer wrong reason," and unfaithfulness
      categories.
- [ ] The full, coordinated acceptance-test suite runs as one CI gate,
      with security checks treated as blocking.
- [ ] `rag-trace-visualizer` correctly renders a full RAG trace.

---

## 17. Summary

RAG introduces a genuinely new upstream failure surface — retrieval
correctness against a large, imperfectly-searchable corpus — that
compounds with every failure mode Part 3, Module 5's pipeline evaluation
already covered, which is why RAG needs its own evaluation layer rather
than simply reusing that framework unchanged. Context precision/recall,
faithfulness, and answer relevance are three distinct questions that
must be measured separately, because a system can score well on one
while failing another — a "right answer, wrong reason" case (a lucky
parametric-memory guess despite failed retrieval) or an unfaithful
answer (correct retrieval, ungrounded generation) are both real,
distinct failure modes a final-answer-only evaluation cannot catch.
Stage-attributed trace evaluation, extending Part 3, Module 5's
structure across every Part 4 module's stages, converts "RAG isn't
working" from an undifferentiated shrug into a specific, actionable
diagnosis. `eval-framework` now has a comprehensive `RAGTraceScorer` and
`FaithfulnessScorer`, a golden dataset covering every important failure
category, and a coordinated acceptance-test suite assembling every Part
4 evaluation tool into one release-readiness gate — the direct
foundation Part 4's production-architecture capstone will use as its
primary quality bar.

---

## 18. Next Steps

**Next: Part 4, Module 9 (Capstone) — Production RAG Architecture.**
This closing module of Part 4 assembles chunking and embedding, vector
and hybrid search, re-ranking, metadata filtering, knowledge graphs,
context compression, and this module's comprehensive evaluation
framework into one finished, integration-tested, documented production
RAG system — following the same assembly, cross-component-seam-auditing,
and honest-scoping discipline Part 3, Module 12 established for the LLM
engineering capstone, now applied to the complete retrieval-augmented
generation stack.

Reply "continue" for the Part 4 capstone module, or flag anything to go
deeper on.
