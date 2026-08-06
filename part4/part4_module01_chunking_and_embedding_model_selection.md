# Part 4, Module 1: Chunking Strategies & Embedding Model Selection

> Part 3 built grounding as a mechanism — supplying explicit context so
> generation is traceable and checkable (Module 10) — but only ever
> applied it to a single tool result or a handful of memory items. Part
> 4 generalizes that mechanism to its full, intended scale: retrieval
> over a large, external document corpus. This first module builds the
> two foundational decisions every RAG system depends on — how you split
> documents into retrievable units, and which embedding model represents
> those units as vectors — because getting either wrong silently caps
> the ceiling on everything Part 4 builds afterward.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain why RAG is architecturally the grounding mechanism from Part
   3, Module 10 generalized to a large, external corpus — retrieve
   relevant material, supply it as explicit context, generate against
   it — and why the two new problems this scale introduces (what to
   retrieve, and how to search efficiently over huge collections) are
   what Part 4 actually adds.
2. Explain, mechanistically, why chunk size and boundary placement
   directly affect both retrieval precision and generation quality, using
   Part 2, Module 4's embedding-geometry intuition and Part 3, Module
   7's "lost in the middle" argument.
3. Compare fixed-size, recursive/structure-aware, and semantic chunking
   strategies, and choose correctly based on document type and downstream
   retrieval/generation requirements.
4. Explain what an embedding model actually optimizes for (Part 2,
   Module 4's contrastive/similarity training objective) and use that
   understanding to evaluate embedding models on dimensions beyond a
   single leaderboard score — domain fit, dimensionality/cost tradeoffs,
   and multilingual/code-specific needs.
5. Design and run a rigorous, retrieval-specific evaluation (precision@k,
   recall@k) for a chunking + embedding-model combination, rather than
   assuming a "better" embedding model or a "smarter" chunking strategy
   is actually better for your specific corpus without evidence.
6. Build a batch document-ingestion pipeline — chunking, embedding, and
   storage — extending `job-processor` (Part 1, Module 6) and
   `contextual-embedding-service` (Part 2, Module 6), as the foundation
   every later Part 4 module builds on directly.

---

## 2. Prerequisites

- **Part 2, Module 4** (Embeddings) and **Module 6** (Attention &
  Transformers, for `contextual-embedding-service`) — you need the
  contrastive-training/similarity-geometry intuition fresh, since
  embedding model selection in this module is evaluated against that
  mechanism, not just a benchmark leaderboard.
- **Part 2, Module 10** (`multimodal-rag-preview`) — this is the direct
  architectural seed for everything in Part 4; you're now building out
  its retrieval side at full scale.
- **Part 3, Module 10** (Hallucination Reduction, Section 6.3 on
  grounding) — Part 4 is this concept generalized; you need that
  argument fresh.
- **Part 3, Module 7** (Context Engineering) — the "lost in the middle"
  argument and token-budget discipline apply directly to how much
  retrieved content you can usefully include.
- **Part 1, Module 6** (`job-processor`) — this module's ingestion
  pipeline is a new batch workload type running through that existing
  infrastructure.
- **Part 2, Module 8** (`eval-framework`) — retrieval evaluation in this
  module extends the precision@k scorer pattern already built for
  `multimodal-rag-preview`.

---

## 3. Estimated Study Time

**8–10 hours** (3 hours theory/reading, 5–7 hours hands-on).

---

## 4. Difficulty

★★★☆☆ (3.5/5)

The mechanisms (splitting text, calling an embedding API, storing
vectors) are simple. The difficulty is entirely in developing calibrated
judgment about chunking/embedding tradeoffs for a specific corpus and
task — a judgment that requires measurement, not intuition, and this
module insists on building that measurement discipline from day one
rather than deferring it to a later evaluation module.

---

## 5. Why This Matters

Nearly every "RAG isn't working well" complaint in production traces
back to one of exactly two root causes: chunks that don't correspond to
coherent, retrievable units of meaning, or an embedding model that
doesn't actually capture similarity the way the task needs it to. Both
are decided at ingestion time, before a single query is ever run — which
means mistakes here are expensive to discover late (you have to
re-chunk and re-embed an entire corpus to fix them) and are exactly the
kind of foundational decision worth getting right through measurement
rather than reflexive default choices. This is also precisely the RAG
knowledge depth that separates "I called a vector database's default
example" from genuine RAG engineering competence, which is squarely
within what senior AI engineering interviews and freelance RAG-system
engagements actually probe.

---

## 6. Theory

### 6.1 RAG as grounding, generalized — and what's genuinely new at this scale

Part 3, Module 10, Section 6.3 established grounding: supply explicit
context so generation is traceable to it rather than relying on
parametric memory. RAG is architecturally that same idea, applied when
the relevant context could be *anywhere in a large corpus* rather than a
single, already-identified tool result. This introduces exactly two new
problems Part 3 never had to solve, because its grounding material was
always small and already selected by the calling code:

- **What to retrieve** — out of potentially millions of candidate
  passages, which ones are actually relevant to the current query? This
  is fundamentally a search problem, not a generation problem, and it's
  solved by embedding-based similarity search (reusing Part 2, Module
  4's mechanism directly) rather than anything the LLM itself does.
- **How to search efficiently at that scale** — brute-force similarity
  search over millions of vectors doesn't scale the way `similarity-
  search-toy`'s (Part 2, Module 1) toy implementation did; this requires
  the vector database and indexing structures covered in the next
  module.

This module addresses the *input* side of the "what to retrieve"
problem: before you can search well, you need to have split your corpus
into sensible, embeddable units (chunking) and represented them well as
vectors (embedding model selection) — get either wrong, and no amount of
search sophistication in later modules can retrieve something that was
never chunked or embedded coherently in the first place.

### 6.2 Why chunk boundaries matter mechanistically

An embedding vector (Part 2, Module 4) is a single, fixed-size
representation of *whatever text you feed into the embedding model* —
recall that the model was trained (via a contrastive objective) to place
semantically similar texts near each other in vector space. **A chunk
that spans two unrelated ideas produces an embedding that's a blurred
average of both** — not clearly close to queries about either idea
individually, which directly degrades retrieval precision, because the
chunk's vector no longer sits cleanly near the region of vector space a
relevant query would land in. Conversely, **a chunk that's too small
(a sentence fragment, a single bullet point stripped of its surrounding
context) may embed coherently on its own but lack enough information for
the *generation* step to actually answer the query once retrieved** —
this is a distinct failure from a retrieval failure: the chunk gets
correctly retrieved, but doesn't contain enough to ground a good answer,
directly connecting to Part 3, Module 7's context-budget concern about
what's actually useful once included.

**The core tension to hold explicitly**: chunk size trades off against
two different things in two different directions — smaller chunks
generally improve retrieval *precision* (a more focused embedding, less
blurring) but risk insufficient context for generation once retrieved;
larger chunks generally improve the *sufficiency* of retrieved context
for generation but risk retrieval imprecision (a blurred, less
discriminative embedding) and reintroduce Module 7's "lost in the
middle" risk if several large chunks are concatenated into the final
prompt. There is no universally correct chunk size — it's a genuine,
measurable tradeoff specific to your corpus and task, which is exactly
why Section 6.5's evaluation discipline is non-negotiable rather than a
nice-to-have.

### 6.3 Chunking strategies compared

**Fixed-size chunking** (split every N tokens/characters, often with a
small overlap between consecutive chunks to reduce boundary-cutting
information loss): simplest to implement, fast, and works acceptably
for relatively uniform, prose-heavy text. **What it trades away**: no
awareness of the document's actual structure, so it will just as
happily cut a chunk boundary through the middle of a sentence, a table
row, or a code function as between two genuinely distinct ideas.

**Recursive/structure-aware chunking** (split along the document's
natural structural boundaries first — headings, paragraphs, code
function/class boundaries, list items — falling back to a fixed-size
split only within a structural unit that's still too large): respects
the document's actual semantic units, producing chunks that are far more
likely to correspond to one coherent idea. **What it requires**:
document-type-specific parsing logic (Markdown headers, code AST
boundaries, PDF layout structure) — more implementation effort than
fixed-size, but the effort pays off directly in Section 6.2's boundary-
coherence argument.

**Semantic chunking** (use embedding similarity itself, or a lighter
model, to detect where the *topic* actually shifts within a document,
and place chunk boundaries at those detected shift points, rather than
at any fixed structural or size-based rule): the most sophisticated
option, directly targeting Section 6.2's core concern (avoid chunks that
blur two unrelated ideas) at the cost of real additional computation
during ingestion (an extra embedding-similarity pass over the document)
and added implementation complexity. **When it earns its cost**:
long-form, loosely-structured prose (a lengthy blog post or transcript
with no clean headings) where structure-aware chunking has nothing
reliable to key off of.

**Practical decision rule**, grounded in the above rather than a
one-size-fits-all default:

| Document type | Lean toward |
|---|---|
| Well-structured docs (Markdown, API references with clear headings) | Recursive/structure-aware |
| Source code | Recursive, respecting function/class boundaries (never fixed-size for code — cutting mid-function is almost always worse than cutting mid-paragraph in prose) |
| Long, loosely-structured prose/transcripts with no clean headings | Semantic chunking, if the added cost is justified by measured quality gains |
| Short, relatively uniform documents (FAQ entries, product descriptions) | Fixed-size is often perfectly adequate — don't over-engineer where simple already works |

### 6.4 What an embedding model actually optimizes for, and how to evaluate one beyond a leaderboard score

Recall from Part 2, Module 4: an embedding model is trained via a
contrastive objective to place texts with *similar meaning* near each
other in vector space, and texts with *different meaning* far apart —
critically, "similar meaning" as defined by whatever training data and
objective the specific model was trained on. This is why blindly
picking "the top-ranked model on a general leaderboard" is an incomplete
strategy: a leaderboard score reflects performance on the benchmark's
specific tasks and domains, which may or may not resemble your actual
corpus and query patterns.

**Dimensions worth evaluating explicitly, beyond a single aggregate
score**:

- **Domain fit**: a model trained predominantly on general web text may
  underperform on a highly specialized corpus (legal documents, medical
  literature, your own codebase) compared to a domain-adapted or
  code-specific embedding model — this is directly analogous to Part 2,
  Module 7's fine-tuning discussion: a model's training distribution
  shapes what it's actually good at, and a general-purpose model's
  "average" competence can mask poor performance on your specific
  distribution.
- **Dimensionality and cost**: higher-dimensional embeddings often
  capture finer-grained distinctions but cost more to store and search
  (directly relevant to Module 2's vector-database indexing tradeoffs,
  next) — a smaller, cheaper embedding model that's "good enough" for
  your actual precision requirements may be the better engineering
  choice than the largest available model, exactly the same
  capability-matched-to-need argument Part 3, Module 9 made for LLM
  routing.
- **Multilingual/code-specific needs**: if your corpus includes source
  code or multiple languages, a general-purpose text embedding model may
  perform meaningfully worse than a model specifically trained on code
  or multilingual data — verify this explicitly for your actual corpus
  rather than assuming a general model transfers adequately.
- **Query-document asymmetry**: some embedding models are specifically
  trained with an asymmetric objective (a "query" encoding and a
  "document" encoding that are related but distinct) rather than a
  single symmetric similarity function — using the wrong encoding side
  for a query versus a document with such a model measurably degrades
  retrieval quality, and this is a common, easy-to-miss integration bug.

### 6.5 Evaluation is not optional: precision@k and recall@k for your specific corpus

Given the genuine tradeoffs in both Sections 6.3 and 6.4, the only
reliable way to choose a chunking strategy and embedding model
combination is to measure retrieval quality directly against a golden
dataset built from your actual corpus and realistic queries — exactly
the discipline `multimodal-rag-preview` (Part 2, Module 10) already
established for cross-modal retrieval, now applied here as the
foundational evaluation for the entire RAG system Part 4 builds.

**Precision@k**: of the top-k retrieved chunks for a query, what
fraction are actually relevant? **Recall@k**: of all the chunks that
are actually relevant to a query, what fraction appear in the top-k
retrieved results? These two metrics answer different questions and can
trade off against each other (a very large k trivially improves recall
while diluting precision), which is why both need to be reported, not
just one.

**Build the golden dataset from real documents in your actual corpus**,
with manually verified (query, relevant-chunk-IDs) pairs — not
synthetic or generic test queries, because chunking and embedding
performance is corpus-specific in exactly the ways Sections 6.3-6.4
describe, and a generic benchmark cannot substitute for evidence about
your specific documents and the queries your system will actually face.

---

## 7. Mental Models

1. **"RAG is Part 3's grounding mechanism, generalized to corpus scale
   — with two genuinely new problems: what to retrieve, and how to
   search efficiently."** Everything else about grounding (traceability,
   faithfulness risk) still applies unchanged.
2. **"A chunk boundary is a decision about what counts as one coherent
   idea."** Get it wrong and you either blur distinct ideas into one
   imprecise vector, or fragment one idea into insufficient pieces.
3. **"An embedding model's leaderboard rank is not the same question as
   'is this the right model for my corpus.'** Domain fit, dimensionality/
   cost, and query-document asymmetry all matter and aren't captured by
   one aggregate score.
4. **"Chunking and embedding decisions are foundational and expensive to
   fix late."** Measure with precision@k/recall@k against your own
   corpus before committing, because re-chunking and re-embedding an
   entire production corpus later is a real, costly undertaking.

---

## 8. Visual Explanation

**Diagram 1 — RAG as grounding, generalized**

```
PART 3's GROUNDING (Module 10):
  [already-known tool result / memory item] ──► supplied directly
  as context ──► generation grounded in it

PART 4's RAG (this module onward):
  [huge corpus, millions of chunks] ──► WHAT TO RETRIEVE?
         │                                    (this module: chunk +
         │                                     embed well; next module:
         │                                     search efficiently)
         ▼
  [top-k relevant chunks] ──► supplied as context ──► generation
  grounded in them (SAME mechanism as Part 3, Module 10, from here on)
```

**Diagram 2 — The chunk-size tradeoff**

```
TOO SMALL                  JUST RIGHT              TOO LARGE
(sentence fragment)        (one coherent idea)     (spans multiple ideas)

Retrieval: sharp,           Retrieval: sharp,        Retrieval: blurred,
 focused embedding           focused embedding         averaged embedding
                                                       — imprecise

Generation: often            Generation: sufficient   Generation: sufficient
 insufficient context         context to answer         context, but risks
 once retrieved                                         "lost in the middle"
                                                         if several are
                                                         concatenated
```

**Diagram 3 — Evaluating an embedding model beyond a leaderboard score**

```
                  Leaderboard rank (general benchmark)
                              │
                    NOT SUFFICIENT ALONE — also check:
                              │
      ┌───────────┬───────────┼───────────┬───────────┐
      ▼           ▼           ▼           ▼           ▼
  Domain fit   Dimension-   Multilingual/  Query-doc    Measured
  (your        ality/cost    code-specific  asymmetry    precision@k/
  corpus,       tradeoff     needs (your    handling     recall@k on
  not generic   (Module 2's  actual         (encoding    YOUR golden
  web text)     indexing)    corpus)        side bug     dataset
                                             risk)        (Section 6.5)
```

---

## 9. Recommended Resources

1. **A well-documented chunking library's strategy comparison** (e.g., a
   widely-used RAG-framework's documentation on text splitters — read
   its comparison of fixed-size, recursive, and semantic splitting
   directly from the source) — read specifically for how a real,
   maintained library implements the recursive/structure-aware strategy
   for different document types (Markdown, code, PDF).
2. **The MTEB (Massive Text Embedding Benchmark) leaderboard and
   methodology documentation** (huggingface.co/spaces or the official
   MTEB paper/repo) — read the methodology section specifically, so you
   understand exactly what a leaderboard rank does and doesn't tell you,
   directly informing Section 6.4's "beyond the leaderboard" argument.
3. **Anthropic or a major embedding-model provider's documentation on
   embedding model selection** (docs.claude.com or the relevant
   provider's docs) — read for the vendor's own guidance on
   dimensionality/cost tradeoffs and any query-document asymmetric
   encoding requirements specific to their models.
4. **Your own `embedding-service`/`contextual-embedding-service`
   codebase (Part 2, Modules 4 and 6)** — the direct technical
   foundation this module extends; review its architecture before
   building the ingestion pipeline.
5. **Your own `multimodal-rag-preview` precision@k scorer (Part 2,
   Module 10)** — review its implementation before extending it into
   this module's corpus-scale retrieval evaluation.

---

## 10. Exercises

1. **Reproduce chunk-boundary blurring, empirically.** Take a real
   document containing two clearly distinct topics in adjacent sections.
   Chunk it with a fixed-size splitter that deliberately cuts across the
   topic boundary, embed the resulting chunk, and compare its similarity
   to queries about each topic individually versus a correctly-boundary-
   respecting chunk's similarity to the same queries.
2. **Compare all three chunking strategies on real documents.** Take a
   representative sample from your actual target corpus (ideally
   spanning at least two document types — e.g., structured docs and
   loosely-structured prose). Chunk each sample with fixed-size,
   recursive/structure-aware, and (for the loosely-structured sample)
   semantic chunking. Build a small golden query set and measure
   precision@k/recall@k for each strategy.
3. **Evaluate at least two embedding models on your corpus.** Using the
   same golden query set from Exercise 2, embed your chunks with two
   different embedding models (ideally one general-purpose, one more
   specialized to your domain if available) and compare precision@k/
   recall@k directly — don't rely on either model's leaderboard rank
   alone.
4. **Deliberately trigger the query-document asymmetry bug.** If your
   chosen embedding model supports asymmetric query/document encoding,
   deliberately embed a query using the document-side encoding (or vice
   versa) and measure the retrieval quality degradation, to build
   concrete intuition for why this integration detail matters.
5. **Cost/dimensionality tradeoff, quantified.** For your two embedding
   models from Exercise 3, compute the actual storage cost difference
   (dimensionality × corpus size) and compare it against the measured
   precision@k/recall@k difference — make an explicit, justified
   recommendation rather than defaulting to "bigger is better."

---

## 11. Mini-Project: `chunking-eval-bench`

A small standalone tool, extending `eval-framework`'s precision@k/
recall@k scorer pattern from `multimodal-rag-preview` (Part 2, Module
10), that takes a document set, a golden query dataset, and a
(chunking strategy, embedding model) combination, and reports
precision@k and recall@k for that combination — built once, to be reused
for every chunking/embedding decision across the rest of Part 4, and the
direct tool used to justify the production project's final choices.

---

## 12. Production Project: A Real Document Ingestion Pipeline

### 12.1 What you're building

1. **A document-ingestion pipeline** running through `job-processor`
   (Part 1, Module 6) — a new batch workload type alongside its existing
   jobs — that takes raw documents (at minimum, Markdown/structured text
   and source code; PDF support can extend this later using the `pdf`
   skill's extraction capabilities), applies the chosen chunking
   strategy (per document type, using Section 6.3's decision rule), and
   produces chunk records ready for embedding.

2. **Batch embedding generation** extending `embedding-service`/
   `contextual-embedding-service` (Part 2, Modules 4 and 6), using
   `job-processor`'s existing idempotency and dead-lettering guarantees
   so a failed or interrupted ingestion run can be safely retried
   without duplicating or corrupting already-embedded chunks.

3. **A `chunking-eval-bench`-justified selection**: for your actual
   target corpus (whatever real documents you're building this RAG
   system against — your own project documentation, a codebase, or a
   domain corpus relevant to your freelancing/interview-prep goals),
   run the comparison from Exercises 2-3 and select a chunking strategy
   and embedding model with documented, evidence-based justification —
   not a default choice.

4. **Storage of chunks and embeddings** in a form ready for the next
   module's vector-database work — at minimum, structured records
   containing the chunk text, its source document/location metadata (for
   later citation/attribution per Part 3, Module 10's discipline), and
   its embedding vector, even if the actual similarity-search
   infrastructure is built in Module 2.

5. **Observability**: emit ingestion pipeline metrics (documents
   processed, chunks produced, embedding failures/retries, cost per
   corpus) via `observability-stack` (Part 1, Module 4).

### 12.2 What this sets up for later modules

- **Part 4, Module 2 (Vector Databases)** will take this module's chunk/
  embedding output and build the actual efficient similarity-search
  index over it.
- **Part 4's hybrid search, re-ranking, and metadata-filtering modules**
  all assume this ingestion pipeline's chunk metadata (source, location,
  document type) is present and well-structured from the start.
- **Part 4's evaluation module** will extend `chunking-eval-bench`
  into a comprehensive, whole-RAG-pipeline evaluation, analogous to how
  Part 3, Module 5 extended single-call evaluation into pipeline
  evaluation.

### 12.3 Definition of done

- The ingestion pipeline correctly processes at least two distinct
  document types from your real target corpus through `job-processor`,
  with idempotent, safely-retriable execution.
- `chunking-eval-bench` reports precision@k/recall@k for at least two
  chunking strategies and two embedding models on your actual corpus and
  golden query set.
- A final chunking strategy and embedding model are selected with
  documented, evidence-based justification (not a default assumption).
- Chunk records include source/location metadata sufficient for later
  citation.
- Ingestion metrics are visible in `observability-stack`.

---

## 13. Practice & Interview Questions

1. Explain why RAG is architecturally a generalization of the grounding
   mechanism from Part 3, and name the two genuinely new problems this
   scale introduces.
2. Explain, mechanistically, why a chunk spanning two unrelated ideas
   produces a less useful embedding than two separately-chunked ideas.
3. Describe the chunk-size tradeoff between retrieval precision and
   generation sufficiency, and explain why there's no universally
   correct chunk size.
4. Why is a general embedding-model leaderboard rank insufficient
   evidence for choosing an embedding model for a specific, specialized
   corpus?
5. What is query-document asymmetric encoding, and what concrete bug can
   arise from mishandling it?
6. Design a golden evaluation dataset for testing chunking strategies on
   a real corpus, and explain why it must be built from your actual
   documents rather than a generic benchmark.

---

## 14. Common Mistakes

- **Defaulting to fixed-size chunking for everything**, including source
  code and well-structured documents, where it needlessly cuts across
  natural boundaries that a structure-aware approach would respect.
- **Choosing an embedding model purely by leaderboard rank**, without
  checking domain fit, dimensionality/cost tradeoffs, or asymmetric
  encoding requirements against your actual corpus.
- **Skipping retrieval evaluation entirely**, assuming a "reasonable-
  sounding" chunking/embedding choice is adequate without measuring
  precision@k/recall@k on real data.
- **Mishandling query-document asymmetric encoding** for models that
  require it, silently degrading retrieval quality in a way that's easy
  to miss without direct evaluation.
- **Treating chunk size as a one-time, global setting** rather than a
  per-document-type decision — code, structured docs, and loose prose
  often warrant genuinely different strategies within the same corpus.
- **Deferring evaluation until after a large corpus has already been
  ingested**, making a later strategy change expensive to apply
  retroactively.

---

## 15. Debugging Exercise

Your `chunking-eval-bench` results show strong precision@k for a
particular chunking strategy and embedding model combination on your
golden query set, but real user queries against the deployed system are
producing noticeably worse retrieval results than the eval suggested.

Write down at least two concrete hypotheses for this discrepancy
(consider: does your golden query set actually represent the phrasing
and specificity of real user queries, or is it too close to the
document's own phrasing — a coverage-gap pattern you've now seen
repeatedly across Part 3's debugging exercises, applied here to
retrieval instead of generation? could real documents in the live
corpus differ structurally from the sample documents used to build the
golden set, e.g., containing document types or formatting your chunking
strategy wasn't tested against?), and describe concretely how you'd
expand `chunking-eval-bench`'s golden set to close this gap.

---

## 16. Checklist

- [ ] I can explain RAG as a generalization of Part 3's grounding
      mechanism and name the two new problems it introduces.
- [ ] I can explain mechanistically why chunk boundaries affect
      embedding quality and articulate the chunk-size tradeoff.
- [ ] I can compare fixed-size, recursive/structure-aware, and semantic
      chunking and choose correctly by document type.
- [ ] I can evaluate an embedding model on dimensions beyond a
      leaderboard rank: domain fit, dimensionality/cost, and asymmetric
      encoding requirements.
- [ ] I understand why precision@k/recall@k must be measured on my own
      corpus and golden query set, not assumed from a generic benchmark.
- [ ] `chunking-eval-bench` is built and has produced real comparative
      results across at least two chunking strategies and two embedding
      models on my actual target corpus.
- [ ] The ingestion pipeline correctly and idempotently processes real
      documents through `job-processor`.
- [ ] A final chunking strategy and embedding model are selected with
      documented, evidence-based justification.
- [ ] Ingestion metrics are visible in `observability-stack`.

---

## 17. Summary

RAG is Part 3's grounding mechanism generalized to a large, external
corpus, introducing two genuinely new problems: deciding what to
retrieve, and searching efficiently at scale. This module addressed the
foundation of the first problem — how documents get split into
retrievable units (chunking) and how those units get represented as
vectors (embedding model selection) — because both decisions are made
once, at ingestion time, and are expensive to fix retroactively across
an entire corpus. Chunk boundaries matter because an embedding blurs
whatever text it's given, so a chunk spanning multiple ideas produces an
imprecise vector, while a chunk that's too small may lack sufficient
context once retrieved — there's no universal right size, only a
measurable tradeoff specific to your corpus. Embedding model selection
requires looking past a single leaderboard score to domain fit,
dimensionality/cost tradeoffs, and asymmetric-encoding requirements, all
verified with precision@k/recall@k against your own documents and
queries, not assumed. The ingestion pipeline built here — chunking,
batch embedding through `job-processor`, and evidence-based strategy
selection via `chunking-eval-bench` — is the direct foundation Module 2
builds real, efficient similarity search on top of.

---

## 18. Next Steps

**Next: Part 4, Module 2 — Vector Databases.** With chunks embedded and
stored, this module covers how to actually search efficiently over
potentially millions of vectors — indexing structures (approximate
nearest-neighbor algorithms), vector database selection and tradeoffs,
and integrating real similarity search into `llm-client-core`'s
retrieval layer, replacing the brute-force approach `similarity-search-
toy` (Part 2, Module 1) used at toy scale.

Reply "continue" for Module 2, or flag anything to go deeper on.
