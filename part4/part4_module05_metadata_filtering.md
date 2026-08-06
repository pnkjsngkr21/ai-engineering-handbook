# Part 4, Module 5: Metadata Filtering & Structured Retrieval Constraints

> Everything built in Modules 1-4 answers "what's semantically and
> lexically relevant to this query?" This module answers a genuinely
> different question that relevance scoring alone cannot: "of the
> relevant content, what is this specific user actually allowed to see,
> and what structural constraints (recency, source authority, document
> type) must the result satisfy regardless of how well it scores?"
> Getting this wrong isn't a quality problem — when it involves
> authorization, it's a security incident.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Distinguish relevance filtering (what Modules 1-4 do) from
   constraint filtering (what this module does), and explain precisely
   why a highly-relevant result can still be an *incorrect* result to
   return if it violates a structural constraint.
2. Design metadata schemas for chunks (extending Module 1's ingestion
   pipeline) that support the specific filter types production RAG
   systems actually need: access-control/authorization, recency/date
   ranges, source authority/trust tier, and document type.
3. Explain the two architectural approaches to combining metadata
   filtering with vector/hybrid search — pre-filtering (restrict the
   searchable set before similarity search runs) and post-filtering
   (search first, then filter results) — and the correctness and
   performance tradeoffs between them.
4. Identify and prevent the specific, serious failure mode where
   post-filtering silently returns fewer results than requested (or none
   at all) because otherwise-top-ranked results get filtered out after
   the fact, and design around it correctly.
5. Treat access-control metadata filtering as a genuine security
   boundary — directly extending Part 3, Module 3 and Module 6's
   discipline that security enforcement belongs in code, never in a
   prompt or in the LLM's own judgment — and design tests that verify
   this boundary holds under adversarial conditions.
6. Extend `llm-client-core`'s retrieval pipeline with a real, tested
   metadata-filtering layer, positioned correctly relative to
   `HybridRetriever` and `Reranker`.

---

## 2. Prerequisites

- **Part 4, Module 1** — chunk metadata (source, location, document
  type) was captured at ingestion; this module extends that schema and
  finally puts it to full use.
- **Part 4, Modules 2-4** (`VectorStore`, `HybridRetriever`, `Reranker`)
  — this module's filtering layer integrates with all three.
- **Part 1, Module 7** (Authentication & Authorization) — the
  access-control filtering in this module is a direct, corpus-retrieval-
  specific application of that module's authorization principles.
- **Part 3, Module 3, Section 6.5** (Tool-Calling Security) — the
  "security belongs in code, never in a prompt" argument applies with
  full force here, since a retrieval-level authorization failure is
  exactly as serious as a tool-calling authorization failure.

---

## 3. Estimated Study Time

**6–8 hours** (2 hours theory/reading, 4–6 hours hands-on).

---

## 4. Difficulty

★★★★☆ (4/5)

The individual filtering mechanisms are simple. The difficulty is
entirely in getting the security-relevant cases exactly right — an
access-control filtering bug in a RAG system is a real data-leak
vulnerability, not a quality bug, and this module holds itself to that
standard throughout.

---

## 5. Why This Matters

A RAG system deployed in any real organizational context — a company
internal-knowledge assistant, a customer-support system with tiered
access, a multi-tenant SaaS product — almost always needs to retrieve
from a corpus where not every document is visible to every user. Getting
metadata/access-control filtering wrong at the retrieval layer is one of
the most serious, concrete classes of real-world RAG security incident:
a system that retrieves and grounds an answer in a document the current
user shouldn't have access to has leaked that document's content through
the generated response, regardless of how well-designed every other part
of the pipeline (guardrails, hallucination reduction, evaluation) is.
This is exactly the kind of unglamorous-sounding but genuinely
consequential engineering discipline that separates a demo RAG system
from one that's safe to deploy against real, permissioned data — a
distinction that matters enormously for freelance and enterprise AI
engineering work specifically.

---

## 6. Theory

### 6.1 Relevance filtering vs. constraint filtering — a genuinely different question

Modules 1-4 built mechanisms answering: **given this query, which chunks
are most semantically and lexically relevant?** This is a *ranking*
question — every chunk gets a score, and the top-scoring ones win.
**Metadata/constraint filtering answers a different, prior question:
given this query and this specific requesting context (user, tenant,
current date), which chunks are even eligible to be considered at all,
regardless of how relevant they might score?** This is not a ranking
question — it's a binary, categorical inclusion/exclusion decision that
must be applied *independently* of relevance scoring, because a
highly-relevant chunk that fails an eligibility constraint (the
requesting user lacks access, the document has been superseded by a
newer version, the document type is explicitly excluded for this query
context) is not a "lower-quality" result — it is an **incorrect result
that must never be returned**, full stop, regardless of its relevance
score.

This distinction matters because conflating the two — e.g., treating an
access-restricted document as merely "less relevant" and hoping it
scores low enough to not surface — is exactly the kind of soft,
probabilistic reasoning Part 3, Module 3 and Module 6 already warned
against for tool-calling authorization. **Constraint filtering must be a
hard, deterministic gate, never a soft scoring adjustment.**

### 6.2 Designing chunk metadata for real filtering needs

Extending Module 1's basic source/location metadata, production RAG
systems typically need at least these metadata dimensions captured at
ingestion time, since retrofitting them onto an already-ingested corpus
is exactly the kind of expensive, late correction Module 1 already
warned about for chunking/embedding decisions:

- **Access-control metadata**: which users, roles, or tenants are
  permitted to see this specific chunk — directly reusing Part 1,
  Module 7's authorization model (roles, permissions, tenant IDs) rather
  than inventing a separate access-control scheme just for retrieval.
- **Recency/temporal metadata**: when the source document was created,
  last updated, or (if applicable) superseded by a newer version —
  needed both for date-range filtering and, more subtly, for excluding
  now-outdated content even when it would otherwise score as relevant
  (a policy document that changed last month shouldn't ground an answer
  about current policy, even if the old version is semantically closest
  to the query).
- **Source authority/trust tier**: whether a document comes from an
  authoritative, verified source versus a lower-trust source (user-
  submitted content, an external unverified document) — relevant when a
  corpus blends sources of varying reliability, connecting directly to
  Part 3, Module 10's hallucination/grounding-quality concerns, since
  grounding in a low-trust source carries different risk than grounding
  in a verified one.
- **Document type/category**: enabling filters like "only search
  official documentation, not meeting transcripts" for query contexts
  where that distinction matters.

### 6.3 Pre-filtering vs. post-filtering: correctness and performance tradeoffs

**Pre-filtering** applies metadata constraints *before* similarity search
runs, restricting the searchable candidate set to only eligible chunks
from the start — e.g., a vector database query that includes both a
similarity search and a metadata filter clause evaluated together,
searching only within the filtered subset. **Post-filtering** runs
similarity search first against the *entire* corpus, then removes
ineligible results from the returned top-k afterward.

**The critical correctness risk with post-filtering**: if a query
requests the top-5 results and similarity search returns its top-5 from
the *entire* corpus, but 3 of those 5 get removed by the post-filter
(access-restricted, superseded, wrong type), **you're left with only 2
results — not the top-5 *eligible* results**, because the similarity
search never had visibility into the constraint and therefore never
looked past its initial top-5 to find the 6th, 7th, 8th-ranked results
that might have been eligible. This is a genuine, silent under-return
bug, not a theoretical edge case — it happens routinely whenever
metadata constraints are even moderately selective (e.g., most users
only have access to a subset of a large corpus), and it's exactly the
kind of failure that "looks fine" in a demo against an unrestricted test
account and only manifests once real, permissioned access patterns are
exercised.

**Pre-filtering avoids this entirely** by ensuring similarity search only
ever considers eligible chunks, so the top-k it returns are genuinely
the top-k *eligible* results. **The tradeoff**: pre-filtering requires
the underlying vector database/search infrastructure to support
efficient combined filter-plus-similarity queries (most modern vector
databases do, per Module 2's architecture options, but this must be
explicitly verified, not assumed, for your chosen database) — and highly
restrictive filters (a filter matching only a tiny fraction of the
corpus) can, in some ANN index implementations, hurt search-quality
guarantees if the index wasn't built with such selective filtering in
mind, which is worth verifying empirically for your specific database
and access-pattern distribution rather than assumed universally safe.

**The practical rule**: prefer pre-filtering by default, especially for
any access-control constraint (where under-returning results is a
correctness bug, and where the constraint is often known before search
even runs, e.g., from the authenticated request context) — reserve
post-filtering only for constraints that are cheap to check and rarely
selective enough to trigger Section 6.3's under-return problem, and
always test explicitly for the under-return failure mode regardless of
which approach you use.

### 6.4 Access-control filtering as a genuine security boundary

Directly extending Part 3, Module 3, Section 6.5 and Module 6's core
argument: **access-control metadata filtering is enforcement that must
live in your retrieval-layer code, verified deterministically against
the actual authenticated request context — never inferred from the
LLM's own judgment about what it should or shouldn't mention, and never
implemented as a prompt instruction ("please don't reveal information
the user isn't authorized to see").** The LLM has no reliable, verifiable
way to know what a given user is authorized to see beyond whatever
you tell it in-context, and — exactly per Module 1's prompt-injection
argument — a soft, prompt-level instruction is not enforcement.

**The retrieval-layer filter must run *before* any chunk content ever
reaches the model's context at all** — if an access-restricted chunk is
retrieved, included in the prompt, and the model is merely instructed
not to repeat its contents, you've already lost: the sensitive content is
in the model's context and its behavior around not repeating it is
governed by the same soft, overridable instruction-following reliability
Module 1 established from the very first module of Part 3. The only
real security boundary is preventing the restricted chunk from ever
being retrieved into context in the first place — which is exactly why
this must be a hard, code-level, pre-filtering-preferred gate, not a
downstream instruction.

**Test this like a real security control, not a feature**: construct
adversarial test cases — a user without access to a specific document
issuing a query specifically designed to surface that document's content
(a query using the document's exact likely phrasing) — and verify the
restricted content never appears anywhere in retrieved results or the
final generated response, treating any failure here with the same
severity as Part 3, Module 3's indirect-injection security test.

---

## 7. Mental Models

1. **"Relevance asks 'how well does this match'; constraint filtering
   asks 'is this even eligible to be considered.'** These are genuinely
   different questions, and constraint filtering must be a hard,
   deterministic gate, never folded into relevance scoring.
2. **"Post-filtering can silently under-return results — this is a real
   bug, not an edge case."** If a constraint is even moderately
   selective, post-filtering after search routinely leaves you with
   fewer than the requested top-k eligible results.
3. **"Access-control filtering is a security boundary, and it must live
   in code, before content ever reaches the model's context."** A prompt
   instruction telling the model not to reveal something is not
   enforcement — by the time that instruction matters, the sensitive
   content is already in context.
4. **"Test access-control filtering adversarially, with the same rigor
   as any other security control."** A query deliberately designed to
   surface restricted content is the correct test, not an incidental
   afterthought.

---

## 8. Visual Explanation

**Diagram 1 — Relevance filtering vs. constraint filtering**

```
RELEVANCE (Modules 1-4):           CONSTRAINT (this module):
  "How well does this chunk          "Is this chunk even ELIGIBLE
   match the query semantically       to be considered, regardless
   and lexically?"                    of how well it matches?"

  → produces a SCORE,                → produces a BINARY gate,
    higher = better                    eligible / not eligible

A chunk can score PERFECTLY on relevance and still be INELIGIBLE
(wrong access tier, superseded document, wrong type) —
these are answered independently, and constraint ALWAYS wins.
```

**Diagram 2 — Post-filtering's silent under-return bug**

```
Query requests: top-5 results

POST-FILTERING (WRONG):
  Similarity search over ENTIRE corpus → top-5 by score:
  [Doc A][Doc B][Doc C][Doc D][Doc E]
              │
     apply access-control filter AFTER
              │
  User lacks access to B, D, E ──► removed
              │
              ▼
  RETURNED: [Doc A][Doc C]  ← only 2 results, NOT top-5 eligible!
  (Docs F, G, H — ranked 6th, 7th, 8th — were NEVER EVEN CONSIDERED,
   even though the user may have access to them)

PRE-FILTERING (CORRECT):
  Similarity search ONLY over chunks the user is eligible for
              │
              ▼
  RETURNED: [Doc A][Doc C][Doc F][Doc G][Doc H]  ← true top-5 ELIGIBLE
```

**Diagram 3 — Where access-control enforcement must live**

```
WRONG (enforcement as a prompt instruction):
  [restricted chunk RETRIEVED INTO CONTEXT]
              │
  prompt: "please don't reveal this to unauthorized users"
              │
  ⚠ sensitive content is ALREADY in the model's context —
    enforcement is a soft, overridable instruction (Module 1's
    argument) — this has ALREADY failed as a security boundary

CORRECT (enforcement in retrieval-layer code, pre-filtering):
  [access-control filter runs BEFORE similarity search]
              │
  restricted chunk NEVER ENTERS the candidate set at all
              │
  ✓ sensitive content never reaches the model's context —
    genuine, code-level enforcement, not a request to the model
```

---

## 9. Recommended Resources

1. **A major vector database's documentation on metadata filtering and
   pre-filtering vs. post-filtering support** (e.g., the official docs
   for your chosen Module 2 vector database) — read specifically for how
   your actual chosen database implements combined filter-plus-
   similarity queries, since this directly determines your architecture.
2. **OWASP — access control and authorization guidance** (owasp.org) —
   read for general access-control-testing methodology, applicable
   directly to the adversarial test design in Section 6.4.
3. **Your own Part 1, Module 7 (Authentication & Authorization)
   codebase** — the direct model this module's access-control metadata
   schema should reuse rather than reinvent.
4. **Your own Part 3, Module 3, Section 6.5 and Module 6's security-test
   patterns** — review before designing this module's adversarial
   access-control tests, since the discipline transfers directly.

---

## 10. Exercises

1. **Reproduce the post-filtering under-return bug, deliberately.**
   Implement post-filtering first (search the full corpus, then filter).
   Construct a query where a meaningfully selective access-control
   filter (e.g., restricting to a small subset of your corpus) is
   applied, and confirm you get fewer than the requested top-k eligible
   results, even though more eligible results exist further down the
   unfiltered ranking.
2. **Fix it with pre-filtering.** Re-implement the same scenario using
   pre-filtering (restrict the searchable set before similarity search
   runs) and confirm you now correctly get the true top-k eligible
   results.
3. **Design and run an adversarial access-control test.** Construct a
   query using the exact likely phrasing of a specific access-restricted
   document, issued by a user without access to it. Verify the
   restricted content never appears in retrieved results or in any
   generated response grounded in retrieval.
4. **Test the wrong-layer-enforcement failure mode, deliberately.**
   Implement access control *only* as a prompt instruction (deliberately
   reproducing the "wrong" pattern from Diagram 3) and confirm, using
   the same adversarial query from Exercise 3, whether the restricted
   content leaks through despite the instruction — building direct,
   first-hand evidence for why this approach fails.
5. **Design a recency-filtering test.** Construct a scenario with two
   versions of a document (an older, superseded one and a current one)
   both present in your corpus. Verify your metadata filtering correctly
   excludes or deprioritizes the superseded version even when it might
   otherwise score as more semantically relevant to a specific phrasing
   of the query.

---

## 11. Mini-Project: `access-control-security-bench`

A small standalone tool, extending your Part 3, Module 3/6 security-test
patterns, that runs a suite of adversarial retrieval queries — each
designed to attempt to surface a specific access-restricted, superseded,
or wrong-type document — against your retrieval pipeline, and reports
pass/fail per test case, with any leak treated as a critical failure, not
a quality score. This is the direct security regression suite the
production project below makes permanent.

---

## 12. Production Project: A Real Metadata-Filtering Layer in `llm-client-core`

### 12.1 What you're building

1. **An extended chunk metadata schema**, building on Module 1's
   ingestion pipeline, adding access-control (reusing Part 1, Module 7's
   authorization model), temporal/recency, source-authority, and
   document-type fields to every ingested chunk.

2. **A `MetadataFilter` component** integrated into `VectorStore`/
   `HybridRetriever` (Modules 2-3), implementing pre-filtering by default
   for access-control constraints specifically (per Section 6.3's
   practical rule), with the filter constraints derived from the actual
   authenticated request context (never from the LLM's own output or
   judgment).

3. **Correct pipeline placement**: `MetadataFilter` must run before
   `Reranker` (Module 4) ever sees a candidate set, and — critically —
   before any chunk content is assembled into the generation prompt via
   `ContextBudgetManager` (Part 3, Module 7), verified by an integration
   test tracing the actual execution order.

4. **`access-control-security-bench` as a permanent CI gate**: the
   adversarial test suite from the mini-project becomes a required,
   passing part of pipeline CI (Part 3, Module 5's discipline), treated
   with the same severity as any other security regression test in the
   handbook — a single failure here should block deployment, not merely
   lower a quality score.

5. **Observability**: emit metadata-filter rejection rates and (via
   `access-control-security-bench`) ongoing security-test pass/fail
   status via `observability-stack`, distinct from general
   retrieval-quality metrics given the different severity of a failure
   here.

### 12.2 What this sets up for later modules

- **Part 4's Knowledge Graphs module** will extend structured retrieval
  constraints further, incorporating relationship-based (not just
  attribute-based) filtering.
- **Part 4's capstone production architecture** assembles
  `MetadataFilter` as a non-optional, security-critical stage of the
  complete RAG pipeline.
- **Part 5 (Agents)** will need the same access-control discipline
  applied to any agent capability that reads from this same corpus.

### 12.3 Definition of done

- Chunk metadata schema includes access-control, temporal, source-
  authority, and document-type fields, correctly populated at ingestion.
- `MetadataFilter` uses pre-filtering for access-control constraints, with
  the under-return bug (Exercise 1) explicitly tested against and
  proven absent.
- Pipeline execution order is verified: `MetadataFilter` runs before
  `Reranker` and before any content reaches context assembly.
- `access-control-security-bench`'s full adversarial suite passes and is
  a required, blocking part of pipeline CI.
- Metadata-filter rejection rates and security-test status are visible
  in `observability-stack`.

---

## 13. Practice & Interview Questions

1. Explain the difference between relevance filtering and constraint
   filtering, and why a highly-relevant result can still be an incorrect
   result to return.
2. Describe the post-filtering under-return bug precisely, with a
   concrete numeric example, and explain why pre-filtering avoids it.
3. Why is "instruct the model not to reveal unauthorized content" an
   insufficient access-control mechanism, even if the model reliably
   follows the instruction most of the time?
4. Design an adversarial test for verifying that access-control metadata
   filtering actually prevents restricted content from ever reaching a
   generated response.
5. What metadata dimensions, beyond access control, does a production
   RAG system typically need, and give a concrete scenario where
   ignoring recency/supersession metadata produces a wrong answer even
   with perfect relevance scoring.
6. When would post-filtering be an acceptable choice over pre-filtering,
   and what would you need to verify before choosing it?

---

## 14. Common Mistakes

- **Treating access-restricted content as merely "less relevant"**
  instead of a hard, categorical exclusion applied independently of
  relevance scoring.
- **Implementing access control via post-filtering without testing for
  the under-return bug**, silently returning fewer results than
  requested whenever the filter is selective.
- **Relying on a prompt instruction to prevent the model from revealing
  restricted content**, when the actual failure has already occurred the
  moment that content entered the model's context.
- **Never adversarially testing access-control filtering**, treating it
  as a feature to be verified only with "happy path" queries rather than
  deliberately crafted attempts to surface restricted content.
- **Retrofitting metadata fields onto an already-ingested corpus**,
  rather than designing the schema comprehensively at ingestion time per
  Module 1's "expensive to fix late" argument.
- **Conflating recency filtering with relevance scoring** — a superseded
  document should be excluded or clearly deprioritized as a constraint,
  not left to compete on pure semantic similarity with its replacement.

---

## 15. Debugging Exercise

Your `access-control-security-bench` suite has been passing consistently.
A recent audit reveals that a specific document, marked as
access-restricted in your metadata, was nonetheless referenced (not
directly quoted, but clearly summarized) in a response generated for a
user without access to it — and your existing adversarial test suite
didn't catch this specific case because the audit's triggering query
used substantially different phrasing than any of your existing
adversarial test cases.

Write down at least two concrete hypotheses for how a document with
correct access-control metadata could still leak through your
pre-filtering layer (consider: is there a code path — e.g., a fallback
retrieval mode, a cache layer from Part 3, Module 8, or a different
entry point into the pipeline — that bypasses `MetadataFilter` entirely,
rather than the filter itself being broken? could this be less a
filtering bug and more a coverage gap in your adversarial test suite,
similar to the pattern seen in nearly every debugging exercise across
Parts 3 and 4 — i.e., your test suite's specific phrasings didn't happen
to trigger the actual leak path, even though the underlying
vulnerability existed all along?), and describe concretely how you'd
audit every code path that can produce a generated response to confirm
`MetadataFilter` is genuinely unbypassable, not just untested against
this specific phrasing.

---

## 16. Checklist

- [ ] I can explain the difference between relevance filtering and
      constraint filtering and why constraint filtering must be a hard
      gate.
- [ ] I can describe the post-filtering under-return bug with a concrete
      example and explain why pre-filtering avoids it.
- [ ] I can explain why access-control enforcement must live in
      retrieval-layer code, before content reaches the model's context —
      never as a prompt instruction.
- [ ] I can design an adversarial test suite for access-control
      filtering, treating any leak as a critical failure.
- [ ] `access-control-security-bench` is built, covers pre- and
      post-filtering comparison, and passes against my real pipeline.
- [ ] The extended chunk metadata schema includes access-control,
      temporal, source-authority, and document-type fields.
- [ ] `MetadataFilter` uses pre-filtering for access control and is
      verified free of the under-return bug.
- [ ] Pipeline execution order is verified: filtering happens before
      re-ranking and before context assembly.
- [ ] `access-control-security-bench` is a required, blocking part of
      pipeline CI.
- [ ] Metadata-filter rejection rates and security-test status are
      visible in `observability-stack`.

---

## 17. Summary

Relevance scoring (Modules 1-4) and constraint filtering (this module)
answer genuinely different questions — how well something matches versus
whether it's even eligible to be considered — and conflating them, or
implementing eligibility as a soft scoring adjustment rather than a hard
gate, is a real correctness and security risk. Post-filtering after
similarity search has a specific, serious failure mode: it can silently
return fewer than the requested top-k eligible results whenever a
constraint is even moderately selective, because search never had
visibility into the constraint while forming its initial ranking —
pre-filtering avoids this by restricting the searchable set before
search runs. Access-control filtering specifically is a genuine security
boundary that must be enforced in retrieval-layer code, before any
restricted content ever reaches the model's context, because a prompt
instruction telling the model not to reveal something is not
enforcement — by the time that instruction would matter, the sensitive
content is already present and the failure has already occurred.
`llm-client-core` now has a real `MetadataFilter` layer, using
pre-filtering for access control, correctly positioned before re-ranking
and context assembly, with a permanent, CI-blocking adversarial security
test suite treating any leak with the severity it deserves.

---

## 18. Next Steps

**Next: Part 4, Module 6 — Knowledge Graphs for Structured Retrieval.**
With attribute-based metadata filtering in place, this module extends
structured retrieval further into relationship-based reasoning — when a
query requires understanding connections *between* entities (not just
matching content about a single entity), a knowledge graph layered
alongside vector/hybrid retrieval can answer questions that pure
similarity search structurally cannot, extending this module's
constraint-filtering discipline into genuinely structured, relational
retrieval.

Reply "continue" for Module 6, or flag anything to go deeper on.
