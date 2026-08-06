# Part 4, Module 9 (Capstone): Production RAG Architecture

> Closes out Part 4 the way Part 3, Module 12 closed out Part 3: not a new
> technique, but the disciplined assembly, seam-auditing, and
> production-hardening of everything built across Modules 1–8.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Trace the **actual runtime execution order** of a full RAG request
   across ingestion-time and query-time components, and explain why that
   order is not the same as the order the modules were built in.
2. Audit at least five **cross-component seams** in a RAG pipeline —
   places where one module's output assumptions must exactly match
   another module's input assumptions — and name the specific bug each
   seam produces when violated.
3. Assemble eight independently-built evaluation benches into a single
   coordinated CI acceptance gate, with a security bench that blocks
   merges rather than merely warning.
4. Ship a versioned, documented retrieval package (`rag-engine`) that
   extends `llm-client-core`, with a stable public interface that a future
   agent (Part 5) can call as a tool without knowing any of its internals.
5. Write the ADRs, runbook, and API reference that a new engineer (or you,
   in six months) would need to safely operate and extend this system.
6. Give an honest, specific account of this system's limitations —
   distinguishing "not yet built" from "architecturally out of scope" —
   and name exactly which future Part addresses each gap.

## 2. Prerequisites

- Part 4, Modules 1–8, completed, with all eight artifacts on disk and
  runnable: `chunking-eval-bench`, `VectorStore`, `ann-tuning-bench`,
  `HybridRetriever`, `hybrid-search-eval-bench`, `Reranker`,
  `rerank-eval-bench`, `MetadataFilter`, `access-control-security-bench`,
  `KnowledgeGraphStore`, `graph-extraction-eval-bench`, the RAG-specific
  `ContextBudgetManager` extension, `retrieval-compression-eval-bench`,
  `RAGTraceScorer`, `FaithfulnessScorer`, and `rag-trace-visualizer`.
- Part 3, Module 12 (you have already done exactly this kind of
  integration audit once — this module is that discipline applied one
  layer up the stack).
- `llm-client-core` v1.0, since `rag-engine` will be shipped as an
  extension package to it.

## 3. Estimated Study Time

14–18 hours across 3–4 sessions. This is the densest module in Part 4 —
budget a full session just for the seam audit before writing any
integration code.

## 4. Difficulty

★★★★★ (5/5) — Capstone. Nothing here is conceptually new; the difficulty
is entirely in getting eight independently-correct components to compose
into one correct system, which is a different (and harder) skill than
building any one of them.

---

## 5. Why This Matters

Every module in Part 4 was validated in isolation. `Reranker` was proven
correct against `rerank-eval-bench`. `MetadataFilter` was proven correct
against `access-control-security-bench`. But "each component is correct
in isolation" does not imply "the composed system is correct" — this is
the same lesson from Part 3, Module 5 (per-stage pass rates don't predict
pipeline correctness), now applied to retrieval instead of generation.

The specific failure mode this module exists to prevent: two components
that are each individually well-tested, but whose **assumptions about
each other are silently wrong**. `MetadataFilter` assumes it runs before
scoring. `HybridRetriever`'s fusion assumes it's fusing over the full
candidate pool. If you wire them in the wrong order, both unit test
suites stay green, and you ship a security hole or a silent
under-return bug that nothing catches until a customer notices — because
no single component's test suite is positioned to catch a *seam* defect.

This is also the module where the retrieval layer becomes a real,
citable artifact in your portfolio: not "I did a RAG tutorial" but "I
designed, integration-tested, and documented a production retrieval
system with a measured false-positive rate on access control, a measured
recall/latency tradeoff at each retrieval stage, and a CI gate that would
have caught the specific seam bugs described below." That sentence is
what a DoorDash or AI-lab system design interviewer is listening for.

---

## 6. Theory

### 6.1 Build order vs. runtime order

You built Part 4 in this order: chunking → vector search → hybrid search
→ re-ranking → metadata filtering → knowledge graphs → context
compression → evaluation. That is a reasonable *pedagogical* order (each
module needed the previous one's concepts). It is **not** the runtime
order of a single query, and treating it as if it were is the single most
common integration bug in RAG systems.

There are actually two runtime timelines: **ingestion-time** (runs once,
or on a schedule, per document) and **query-time** (runs once per user
request). Confusing which components belong to which timeline is the
first seam bug.

**Ingestion-time pipeline** (Modules 1, 2, 6 build this):

```
raw document
  → chunking (Module 1)
  → embedding (Module 1)
  → entity/relationship extraction (Module 6, only for graph-eligible docs)
  → write to VectorStore (Module 2) + KnowledgeGraphStore (Module 6)
  → metadata attached at write time: source_id, access_control_labels,
    doc_version, chunk_position, extraction_confidence
```

This runs through `job-processor` (Part 1, Module 6), asynchronously, in
the background. It is *not* on the critical path of any user request.
This matters because it means chunking and embedding-model mistakes are
expensive to discover — they're baked into every downstream vector
until you re-ingest — which is precisely why Module 1's eval bench had
to exist before you scaled ingestion.

**Query-time pipeline** (this is the part with the real seam risk):

```
 1. user query arrives
 2. query classification / routing decision:
      chunk-based vs. graph-based vs. both (Module 6)
 3. metadata pre-filter constructed from the requester's identity
      (Module 5) — this narrows the candidate set BEFORE retrieval scores
      anything
 4. retrieval executes against the pre-filtered candidate set:
      - vector search (Module 2) -- if chunk-based path selected
      - BM25 search (Module 3)    -- if chunk-based path selected
      - RRF fusion combines the two (Module 3)
      - graph traversal (Module 6) -- if graph-based path selected, run
        in parallel with chunk-based retrieval, not instead of it, when
        the router flags the query as needing both
 5. re-ranking (Module 4) scores the fused/combined candidate set with a
      cross-encoder
 6. context compression (Module 7) reduces the re-ranked top-k to fit the
      context budget, in info-loss-risk order, while preserving per-chunk
      source metadata
 7. `ContextBudgetManager` (Part 3 Module 7, RAG-extended in Module 7)
      places the compressed context in the prompt, respecting placement
      effects ("lost in the middle")
 8. generation (the underlying LLM call, via `llm-client-core`)
 9. faithfulness / citation verification pass (Module 8, reusing Part 3
      Module 10's verification-pass mechanism) runs on the *generated
      answer against the retrieved-and-compressed context*
10. response returned to user, annotated with citations
11. (async, off critical path) trace recorded for `RAGTraceScorer`
```

Notice step 3 happens **before** step 4, not after. This is the seam this
module spends the most time on, because it is the one with security
consequences, not just quality consequences.

### 6.2 Why pre-filtering must precede scoring (the security seam)

Module 5 already taught you post-filtering's silent under-return bug: if
you retrieve top-k by relevance and *then* drop the ones the user isn't
authorized to see, you can return fewer than k results with no signal
that anything was hidden — a quality bug. But composed with
`HybridRetriever`'s fusion, there's a second, worse failure mode:

If metadata filtering runs *after* fusion instead of before it,
Reciprocal Rank Fusion computes ranks over a candidate pool that includes
documents the requester cannot see. Even if those documents are dropped
before the response is generated, **their presence has already changed
the ranks of the documents the requester *can* see** — RRF's score for a
visible document is a function of its rank position relative to
everything else in the pool, authorized or not. This means the ranking
the user experiences is silently influenced by content that was supposed
to be invisible to them. In an adversarial framing: a malicious actor
could infer something about the existence or ranking-relevance of
documents they can't access, purely from how it perturbs the ranks of
documents they can. That's an information-leak channel through a
statistical side effect, not a code bug — which is exactly the kind of
seam defect that no single component's unit tests can catch, because both
`MetadataFilter` and `HybridRetriever` are individually behaving exactly
as specified.

**Rule, stated once and enforced structurally:** `MetadataFilter`
constructs the *eligible candidate set* first. Every retrieval path —
vector, BM25, graph — receives that eligible set as its search space, not
the full corpus. This is the same "authorization in code before content
reaches the model" principle from Module 5, extended one level:
authorization must also happen before content reaches *any scoring
function*, not just before it reaches the model's context window.

### 6.3 Does `Reranker` see the correctly-filtered set?

Yes, structurally, if you enforce the ordering above — `Reranker` only
ever receives what `HybridRetriever`/`KnowledgeGraphStore` returned, and
those only ever searched within the pre-filtered eligible set. But there
is a second seam here worth naming explicitly: **the cross-encoder in
`Reranker` was trained/evaluated in Module 4 against relevance labels
that had no notion of access control.** This means `Reranker`'s scores
are a pure relevance signal, uncontaminated by eligibility — which is
correct, because eligibility was already decided in step 3. If you ever
find yourself wanting to add access-control signal into the re-ranking
score itself, that is a design smell: it means pre-filtering didn't
actually narrow the candidate set, and you're trying to patch the
symptom at the wrong layer.

### 6.4 Does compression run after re-ranking, and does it preserve citation metadata?

Order matters here for a pure efficiency reason, not a correctness one:
compression (Module 7) is comparatively expensive (sub-chunk extraction,
summarization calls), so you want to run it only on the small
re-ranked top-k that will actually reach the prompt, not on the larger
candidate set that fed the re-ranker. Running compression before
re-ranking would mean paying compression cost on results you're about to
discard.

The correctness seam here is different: Module 7's compression
techniques (dedup, sub-chunk extraction, structured summarization) must
each be audited to confirm they **carry forward** `source_id`,
`chunk_position`, and enough of the original text span to support
citation — because Module 8's `FaithfulnessScorer` needs to verify the
generated answer against *exactly what the model saw*, attributed back
to a specific source. If compression silently drops source attribution
while compressing content, `FaithfulnessScorer` has nothing to check
faithfulness against, and citations in the final answer become
unverifiable claims dressed up as verified ones — which is worse than no
citations at all, because it looks trustworthy without being
trustworthy. This is a direct instance of Part 3 Module 10's lesson:
citation is itself confabulatable and must be verified, not assumed
correct because it's present.

**Concretely:** every compression technique in Module 7's implementation
must take a `RetrievedChunk` (with metadata) as input and return a
`CompressedChunk` (with the same metadata fields, shorter `text`) as
output — never a bare string. If any compression function in your
Module 7 code returns `str` instead of `CompressedChunk`, that is a seam
bug, and this module's integration tests check for it explicitly.

### 6.5 Does knowledge-graph routing correctly combine with hybrid retrieval?

Module 8's debugging exercise surfaced this: some queries are genuinely
"needs both" (a multi-hop relational sub-question plus a semantic
lookup), and routing to *either* chunk-based or graph-based retrieval
alone under-serves them. The seam risk is in how the two result sets are
combined when the router selects "both": if you naively concatenate
graph-traversal results and hybrid-search results and hand the
concatenation straight to the re-ranker, the re-ranker's cross-encoder —
trained on chunk-vs-query relevance — has no principled way to score a
graph-traversal result (which may be a structured fact, not a text
chunk). The fix, which you should verify is actually implemented and not
just described: graph-traversal results get **rendered into a
chunk-shaped text representation** (a canonical sentence form of the
traversed fact, with the same metadata shape as a `RetrievedChunk`)
*before* they enter the shared candidate pool that the re-ranker scores.
This is the same "adapter pattern" idea from Part 1, Module 2, applied to
make two structurally different retrieval outputs commensurable to a
single downstream scorer.

### 6.6 What actually changes at "production" scale

Everything above is correctness. Two things are genuinely new at this
module and weren't forced by any single earlier module:

- **A single coordinated CI gate.** Each of the eight eval benches was
  built to answer one question in isolation. Composed into one gate,
  they must run in a defined order (fast/cheap checks first — chunking
  and access-control — expensive checks last — full `RAGTraceScorer`
  end-to-end runs), and the access-control bench must be a **hard
  blocker**, not a warning, exactly as established in Module 5: a
  regression in relevance quality degrades the product; a regression in
  access control is a security incident, and CI should treat those two
  classes of failure with different severities.
- **A single versioned public interface.** Everything built in Modules
  1–8 becomes an internal implementation detail behind one `RAGEngine`
  class. This is not gratuitous encapsulation — it is what makes the
  system usable as a tool by Part 5's agents, which should be able to
  call `rag_engine.retrieve(query, identity)` without knowing whether the
  answer came from a vector store, a graph, or both, and without being
  able to accidentally bypass the metadata filter by calling an internal
  component directly.

---

## 7. Mental Models

1. **"Each component's tests prove it's correct alone; only the seam
   audit proves the system is correct together."**
2. **"Authorization must happen before scoring, not just before
   generation — a rank is already a leak."**
3. **"If metadata isn't shaped the same going in and coming out of every
   stage, citation is a claim you can't back up."**
4. **"The public interface is the security boundary — anything callable
   that skips the filter is a vulnerability with a function signature."**

---

## 8. Visual Explanation

**Query-time pipeline, with the seam-audit points marked (⚠):**

```
                         user query + identity
                                 │
                                 ▼
                     ┌───────────────────────┐
                     │   Query Router          │
                     │  (chunk / graph / both) │
                     └──────────┬────────────┘
                                 │
                                 ▼
              ⚠ SEAM 1: filter BEFORE scoring
                     ┌───────────────────────┐
                     │   MetadataFilter        │
                     │  eligible candidate set │
                     └──────────┬────────────┘
                 ┌───────────────┼────────────────┐
                 ▼                                 ▼
        ┌────────────────┐              ┌──────────────────────┐
        │ HybridRetriever  │              │ KnowledgeGraphStore    │
        │ (vector + BM25   │              │ traversal → rendered   │
        │  + RRF fusion)   │              │ as chunk-shaped text   │
        └────────┬────────┘              └──────────┬────────────┘
                 └───────────────┬──────────────────┘
                                 ▼
              ⚠ SEAM 2: commensurable candidate pool
                     ┌───────────────────────┐
                     │       Reranker           │
                     │  cross-encoder scoring   │
                     └──────────┬────────────┘
                                 ▼
              ⚠ SEAM 3: metadata preserved through compression
                     ┌───────────────────────┐
                     │  Context Compression     │
                     │  (dedup → extract →      │
                     │   summarize)             │
                     └──────────┬────────────┘
                                 ▼
                     ┌───────────────────────┐
                     │  ContextBudgetManager    │
                     │  (placement-aware)       │
                     └──────────┬────────────┘
                                 ▼
                         LLM generation
                                 │
                                 ▼
              ⚠ SEAM 4: faithfulness checked against
                          what the model actually saw
                     ┌───────────────────────┐
                     │  FaithfulnessScorer      │
                     │  (blocking, pre-return)  │
                     └──────────┬────────────┘
                                 ▼
                    response + verified citations
                                 │
                                 ▼ (async, off critical path)
                        RAGTraceScorer logging
```

**Ingestion-time pipeline (separate timeline, no query-time seams):**

```
raw doc → chunk → embed → [extract entities/relations if graph-eligible]
        → write VectorStore + KnowledgeGraphStore, tagged with
          access_control_labels + doc_version + chunk_position
```

---

## 9. Recommended Resources

1. **Anthropic — "Building Effective Agents" / "Contextual Retrieval"
   engineering posts** — the closest official framing to "RAG as
   assembled system, not single technique"; useful cross-check against
   your own seam list.
2. **Kleppmann, *Designing Data-Intensive Applications*, Ch. 12
   ("The Future of Data Systems")** — the general theory of composing
   independently-correct subsystems into a correct whole; the
   derived-data/seam-consistency framing maps directly onto this
   module's problem even though the book predates RAG.
3. **OWASP — Access Control Testing guidance** — reread the sections on
   testing for authorization-bypass-through-inference now that you have
   a concrete example (Section 6.2's RRF-rank leak) of that exact
   category of bug.
4. **Your own Module 5 and Module 8 debugging exercises** — the most
   relevant "resource" here is re-reading your own prior write-ups of
   the under-return bug and the routing debugging exercise before
   writing this module's integration tests; you are extending both.
5. **Martin Fowler — "Contract Tests"** — the specific testing pattern
   this module needs: tests that pin down the *interface contract*
   between two components (e.g., "compression always returns
   `CompressedChunk`, never `str`"), independent of either component's
   internal correctness.

---

## 10. Exercises

1. Write out, from memory, the 11-step query-time execution order from
   Section 6.1, without looking back at this document. Compare against
   the original and note every place your memory diverged — those
   divergences are exactly where you'd introduce a seam bug under time
   pressure.
2. For each of the four seams marked ⚠ in the diagram, write one
   contract test (per Fowler's pattern) that would fail if that seam
   were violated, *without* testing either component's internal
   correctness.
3. Construct a query that should route to "both" (chunk + graph). Trace
   it manually through the pipeline and confirm the graph-traversal
   result actually gets rendered into chunk-shaped text before reaching
   the re-ranker. If your Module 6 implementation doesn't already do
   this, fix it now — this is a prerequisite for the Production Project.
4. Take one document with an `access_control_labels` field. Simulate two
   identities, one authorized and one not. Run the same query as both
   identities and diff the RRF ranks of the *authorized* documents
   between the two runs. If the ranks differ, you have Section 6.2's
   leak; find and fix the filtering order in your own code.
5. Pick any one of the eight eval benches. Estimate its runtime cost
   (API calls, wall-clock time) and decide where in a CI pipeline
   ordering (fast-blocking → slow-blocking → nightly-only) it belongs,
   and justify the placement.

---

## 11. Mini-Project

**`seam-audit-report`**: a single markdown document (not code) where you
manually trace three real queries from your own test corpus end-to-end
through the 11-step pipeline, recording at each step: what came in, what
went out, and which metadata fields were present. Flag any step where a
field silently disappeared. This is intentionally low-tech — the goal is
to force you to actually read your own code's data flow before writing
the automated integration tests in the Production Project, the same way
Part 3, Module 12 had you manually trace the 13-step conversational
pipeline before automating anything.

---

## 12. Production Project: `rag-engine` (Part 4 capstone artifact)

### Scope

Ship a single installable package, `rag-engine`, extending
`llm-client-core`, that:

- Wraps every Part 4 component (`VectorStore`, `HybridRetriever`,
  `Reranker`, `MetadataFilter`, `KnowledgeGraphStore` + router, the RAG
  `ContextBudgetManager` extension) behind **one public class**:

```python
class RAGEngine:
    def __init__(self, config: RAGEngineConfig): ...

    async def retrieve(
        self,
        query: str,
        identity: RequesterIdentity,
        k: int = 8,
    ) -> RetrievalResult:
        """
        The ONLY entry point. Enforces, in order:
        routing -> pre-filtering -> retrieval -> fusion/rendering
        -> re-ranking -> compression -> budget placement.
        Internal components are not exported from the package's
        public __init__.py — callers cannot bypass the filter by
        reaching for VectorStore directly.
        """

    async def generate(
        self,
        query: str,
        identity: RequesterIdentity,
        k: int = 8,
    ) -> RAGResponse:
        """
        retrieve() -> assemble via ContextBudgetManager -> call
        llm-client-core -> FaithfulnessScorer verification pass
        (blocking; on failure, either regenerate once with a
        stricter grounding instruction or return a flagged,
        lower-confidence response -- never silently return an
        unverified answer as if it were verified) -> trace logged
        async via RAGTraceScorer.
        """
```

- Enforces every ordering constraint from Section 6 structurally (via
  code organization, not just convention) — the eligible-set-first
  filter, the chunk-shaped rendering adapter for graph results, the
  `CompressedChunk`-typed compression contract, and the blocking
  faithfulness check before return.

- Includes an **integration test suite** distinct from each module's own
  unit tests: contract tests per Exercise 2, an authorization-leak test
  per Exercise 4 (this one runs in CI as part of the security gate), and
  at least one full end-to-end "both" routing test per Exercise 3.

- Assembles all eight eval benches into `rag-ci-gate`, a single runnable
  suite with three tiers:
  - **Tier 1 (blocking, every PR):** `access-control-security-bench`,
    contract tests.
  - **Tier 2 (blocking, every PR, slower):** `hybrid-search-eval-bench`,
    `rerank-eval-bench`, `retrieval-compression-eval-bench`.
  - **Tier 3 (nightly, non-blocking but alerting):**
    `chunking-eval-bench`, `ann-tuning-bench`,
    `graph-extraction-eval-bench`, full `RAGTraceScorer` end-to-end run.

- Ships with:
  - **ADRs** for: chunking/embedding model choice, vector DB
    architecture, hybrid fusion approach, re-ranking cost/benefit
    policy, and knowledge-graph usage scope (when to route there vs.
    not).
  - **Runbook**: RAG-specific `observability-stack` metrics to watch
    (retrieval latency per stage, fusion recall proxy, faithfulness
    score distribution, access-control-bench pass rate over time), and
    what an on-call engineer does when each degrades.
  - **API reference** for `RAGEngine.retrieve()` / `.generate()`,
    written for a consumer who has never seen the internals — this is
    literally the document Part 5's agent-building module will hand you.

### Explicit extension point

**Part 5 (AI Agents)** will import `rag-engine` and use `RAGEngine` as
one tool in an agent's tool set, exactly the way Part 3's tool-calling
loop consumed arbitrary tools — this module's API reference is written
so that hookup requires zero knowledge of Modules 1–8's internals.
**Part 6 (AI Infrastructure)** will replace the embedding, re-ranking,
and graph-extraction model calls inside this package with self-hosted
serving, without changing `RAGEngine`'s public interface — which is the
actual test of whether this module's encapsulation was done correctly.

---

## 13. Practice & Interview Questions

1. Walk through the runtime execution order of a RAG request and explain
   why it differs from the build/learning order. What's the concrete
   cost of getting this order wrong in production?
2. Explain, without using the word "leak" until the end, how running
   metadata filtering after fusion instead of before it can expose
   information about documents a user isn't authorized to see, even if
   those documents are never returned in the response.
3. You're given two independently-passing test suites — one for a
   re-ranker, one for a metadata filter — and a production incident
   where an unauthorized document influenced a response. Where do you
   look first, and why wouldn't either team's existing tests have caught
   it?
4. Design the public interface for a retrieval system that will be used
   as a tool by an autonomous agent you don't control the code of. What
   invariants does that interface need to guarantee, and why?
5. How would you decide which of eight evaluation benches should block a
   CI merge versus run nightly? What's the cost of getting that severity
   assignment wrong in each direction?
6. A compression step in your context pipeline returns a bare string
   instead of a typed object with source metadata. What downstream
   capability silently breaks, and why might it take a long time to
   notice?
7. Explain why a re-ranker trained purely on relevance labels should
   never also be asked to incorporate access-control signal, even though
   it would be architecturally easy to add that as an extra feature.

---

## 14. Common Mistakes

- **Wiring components in build order instead of runtime order** — the
  single most common integration bug; always draw the actual per-request
  timeline before wiring anything.
- **Treating each component's green test suite as sufficient evidence
  the system is correct** — per-component correctness and seam
  correctness are different claims requiring different tests.
- **Applying access control after scoring instead of before it** —
  produces both the quality bug (Module 5's under-return) and the more
  subtle security bug (Section 6.2's rank-leak), and the second one
  won't show up in a simple "are unauthorized docs in the response?"
  test — you have to specifically test for rank perturbation.
- **Letting compression functions return bare strings** — quietly kills
  citation verifiability; enforce a typed contract, don't rely on
  convention.
- **Making every eval bench a blocking CI gate at the same severity** —
  either slows every PR to a crawl (if everything blocks) or lets a
  security regression through with a warning nobody reads (if nothing
  blocks); severity must be assigned per failure class, per Module 5's
  established pattern.
- **Exporting internal components from the package** — if
  `VectorStore` or `HybridRetriever` are importable directly by a
  consumer of `rag-engine`, the "one entry point enforces the filter"
  guarantee is fiction; encapsulation has to be structural, not
  documented-but-optional.

---

## 15. Debugging Exercise

You receive this bug report: "Users report that search results feel
slightly different depending on who's logged in, even for queries about
public documents that everyone can see." No unauthorized document is
ever visible in any response. Nothing in the access-control test suite
fails.

Using Section 6.2 and Exercise 4, identify:
(a) the mechanism most likely responsible,
(b) the specific test you'd write to confirm it (hint: it's not a test
of *what's returned*, it's a test of *rank position* of what's returned,
diffed across two identities),
(c) the one-line structural fix, and
(d) why this bug is specifically dangerous — what could a sufficiently
motivated adversary infer from it, given enough queries?

---

## 16. Checklist

- [ ] I can draw the 11-step query-time pipeline from memory, correctly
      ordered, without notes.
- [ ] I can explain, in one sentence each, all four ⚠ seams and the
      specific bug each one produces when violated.
- [ ] My `MetadataFilter` runs before any retrieval scoring, verified by
      an actual rank-diff test across two identities, not just a
      "no unauthorized doc in the response" test.
- [ ] My graph-traversal results are rendered into chunk-shaped text
      before entering the shared re-ranking candidate pool.
- [ ] Every compression function in my pipeline returns a typed
      `CompressedChunk`, never a bare string, verified by a contract
      test.
- [ ] `FaithfulnessScorer` runs as a blocking pre-return check, not a
      post-hoc logging-only step.
- [ ] All eight eval benches are assembled into `rag-ci-gate` with an
      explicit, justified tiering (blocking vs. nightly).
- [ ] The access-control bench is a hard CI blocker, not a warning.
- [ ] `RAGEngine` is the only exported entry point from `rag-engine`;
      internal components are not importable by consumers.
- [ ] I've written the ADRs, runbook, and API reference — and the API
      reference alone (no internals knowledge required) is enough to
      hook this into an agent's tool set.
- [ ] I can name all four Part-4 limitations (no agentic retrieval
      behavior, no self-hosted serving, no frontend, foundational-only
      deployment maturity) and which future Part addresses each.

---

## 17. Summary

Part 4 built eight independently strong retrieval components. This
module didn't add a ninth technique — it proved, through explicit seam
audits rather than hope, that the eight compose into one system that
behaves correctly end-to-end: authorization narrows the candidate set
*before* anything scores it, structurally different retrieval outputs
(chunks and graph traversals) are made commensurable before a shared
re-ranker sees them, metadata survives compression so citations stay
verifiable, and faithfulness is checked against exactly what the model
saw before a response ever leaves the system. All of that is now
encapsulated behind one interface, `RAGEngine`, versioned and documented
well enough that Part 5's agents can use it as a tool without
understanding — or being able to bypass — any of Part 4's internals.

The honest limitations, and where each is addressed:

| Limitation | Addressed in |
|---|---|
| No autonomous multi-step research/retrieval agent behavior | Part 5 — AI Agents |
| No self-hosted embedding/re-ranking/extraction model serving | Part 6 — AI Infrastructure |
| No dedicated frontend for the RAG system | Part 7 — Frontend for AI |
| Foundational-only production deployment maturity | Part 8 — Production AI |

---

## 18. Next Steps

Part 4 is complete. `rag-engine` v1.0 is your second major shipped
package artifact, alongside `llm-client-core` v1.0 from Part 3 — and the
two are now composable: an LLM client with a retrieval tool attached is
most of what Part 5 needs on day one.

Part 5 (AI Agents) begins next: planning, reflection, memory-driven
autonomy, and multi-agent systems, applying Part 1 Module 9's
prompt-injection/privilege-separation content and Part 3's guardrails
work to systems that take multiple autonomous steps rather than
answering in one turn — with `rag-engine` as the first real tool such an
agent will call.

---

Reply "continue" for Module N, or flag anything to go deeper on.
