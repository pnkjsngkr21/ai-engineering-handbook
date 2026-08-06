# Part 4, Module 6: Knowledge Graphs for Structured Retrieval

> Every retrieval mechanism built so far — vector similarity, keyword
> search, re-ranking, metadata filtering — answers questions about
> individual chunks: is this chunk relevant, does it match this term,
> is it eligible. None of them can answer a question that's fundamentally
> about the *relationship* between two entities described in different
> documents. This module adds a structurally different retrieval
> mechanism — a knowledge graph — specifically for the class of query
> that requires traversing relationships, not just matching content.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain precisely why vector/hybrid retrieval, however well-tuned,
   structurally cannot answer multi-hop relational queries (questions
   requiring information connected across two or more documents via an
   explicit relationship) — grounding the explanation in what a chunk
   embedding actually represents versus what a relational query
   actually asks.
2. Explain the core knowledge-graph data model — entities (nodes) and
   relationships (edges) — and how it differs fundamentally from a
   chunk-based vector index.
3. Design an entity/relationship extraction pipeline that builds a
   knowledge graph from your existing document corpus, using
   structured-output extraction (Part 3, Module 2) rather than
   unstructured, unreliable free-text parsing.
4. Explain graph traversal/query mechanisms (multi-hop relationship
   queries) at a level sufficient to design when and how to invoke them
   from `llm-client-core`'s retrieval layer.
5. Design a combined retrieval architecture where vector/hybrid search
   and knowledge-graph traversal are used for genuinely different query
   types, with an explicit routing decision (extending Part 3, Module
   9's routing discipline) rather than one replacing the other.
6. Evaluate knowledge-graph retrieval's actual contribution rigorously,
   using a golden query set specifically constructed with multi-hop
   relational questions, rather than assuming graph-based retrieval is
   automatically superior for a general corpus.

---

## 2. Prerequisites

- **Part 4, Modules 1-5** — you need the full vector/hybrid/re-ranking/
  metadata-filtering retrieval stack working, since this module's core
  argument is about what that stack *cannot* do, not a replacement for
  it.
- **Part 3, Module 2** (Structured Outputs) — entity/relationship
  extraction in this module reuses schema-constrained output directly,
  rather than unreliable free-text parsing.
- **Part 3, Module 9** (Latency & Cost Optimization, routing) — the
  routing discipline (match the mechanism to the query type, verified by
  evidence) applies directly to deciding when to use graph traversal
  versus vector/hybrid retrieval.
- **Part 2, Module 8** (`eval-framework`) — knowledge-graph retrieval's
  contribution must be measured with the same rigor as every other
  retrieval mechanism in Part 4.

---

## 3. Estimated Study Time

**8–10 hours** (3 hours theory/reading, 5–7 hours hands-on).

---

## 4. Difficulty

★★★★☆ (4/5)

The conceptual case for knowledge graphs is straightforward; the real
difficulty is in reliable entity/relationship extraction from
unstructured documents (a genuinely hard, error-prone step) and in
honestly evaluating whether the added complexity is worth it for your
specific corpus and query patterns, rather than adopting it because it
sounds more sophisticated.

---

## 5. Why This Matters

A specific, real class of query defeats vector/hybrid retrieval no
matter how well it's tuned: "who reports to the person who approved this
project?", "which vendors does our supplier rely on that are also used
by our competitor?", "what's the chain of documents that led to this
policy change?" — questions where the answer requires connecting facts
that live in *different* documents via an explicit relationship, not
questions a single retrieved chunk (however relevant) can answer on its
own. Recognizing this class of query, and knowing when a knowledge
graph is the right tool versus when it's unnecessary complexity for a
corpus that doesn't actually need relational reasoning, is a genuine
mark of mature RAG system design — and, done honestly, requires the same
measured, evidence-based restraint this handbook has insisted on for
every other "more sophisticated" technique introduced in Part 4.

---

## 6. Theory

### 6.1 Why vector/hybrid retrieval cannot answer multi-hop relational queries

Recall the fundamental unit every mechanism in Modules 1-5 operates on:
a **chunk** — a bounded span of text, embedded as a single vector
(Module 1) or matched via keyword terms (Module 3), then filtered
(Module 5) and re-ranked (Module 4). Every one of these mechanisms scores
or filters chunks **independently of each other** — there is no
mechanism, anywhere in that stack, for combining information *across*
two separately-retrieved chunks into a single answer that depends on how
they relate to each other.

Consider a concrete multi-hop query: "who is the manager of the person
who wrote the Q3 budget proposal?" This requires first identifying who
wrote the Q3 budget proposal (fact 1, likely findable in one chunk),
then identifying that person's manager (fact 2, likely findable in a
*different* chunk — an org chart or HR document that has no semantic or
lexical overlap with the budget proposal at all). **No single chunk
contains the answer**, and no amount of tuning chunk size, embedding
model choice, hybrid fusion, or re-ranking (Modules 1-4) changes this,
because the answer isn't "in" any one chunk — it's in the *relationship*
between two facts that live in structurally unrelated documents. This is
not a retrieval-quality problem to be optimized away; it's a fundamental
mismatch between what chunk-based retrieval computes (relevance of
independent spans of text) and what the query actually requires
(traversal across an explicit relationship).

### 6.2 The knowledge-graph data model: entities and relationships, not chunks

A **knowledge graph** represents information as **entities** (nodes —
a person, a document, a project, an organization) connected by
**relationships** (edges — "wrote," "manages," "approved," "supersedes"),
each edge typically labeled with the specific relationship type. This is
a fundamentally different representation from a chunk index: instead of
"here is a span of text and its embedding," a knowledge graph says "here
is a specific entity, and here are its specific, typed connections to
other specific entities."

**Why this solves the multi-hop problem**: once "person who wrote the
Q3 budget proposal" is represented as an entity node with a "wrote" edge
to the "Q3 budget proposal" document node, and that same person entity
has a separate "reports to" edge to their manager's entity node,
answering "who is the manager of the person who wrote the Q3 budget
proposal" becomes a **graph traversal** — follow the "wrote" edge
backward to find the author entity, then follow that entity's "reports
to" edge forward to find the manager — a well-defined, mechanical
operation over explicit structure, categorically different from
similarity scoring over independent text spans.

### 6.3 Building a knowledge graph: extraction is the hard, error-prone part

The graph structure itself (nodes and typed edges) is conceptually
simple; **building an accurate graph from unstructured documents is the
genuinely difficult engineering problem**, because it requires reliably
extracting entities and their relationships from free text — exactly the
kind of task Part 3, Module 2's structured-output discipline exists for,
rather than attempting unreliable free-text parsing or regex-based
extraction.

**The extraction pipeline**: for each document (or chunk), use a
schema-constrained LLM call (Module 2) with an explicit schema for
`entities: List[Entity]` and `relationships: List[Relationship]` (each
relationship specifying a source entity, target entity, and typed
relationship label), rather than an open-ended "extract the entities and
relationships from this text" prompt — precisely the same discipline
Part 3, Module 4 established for memory-compaction extraction: structured
extraction with an explicit schema outperforms free-text generation for
exactly the same reasons (Part 3, Module 4, Section 6.3's confabulation
and omission risks apply identically here).

**A hard, unavoidable challenge specific to this extraction**: **entity
resolution** — recognizing that "J. Smith," "Jane Smith," and "the VP of
Engineering" appearing in different documents all refer to the *same*
underlying entity. Getting this wrong either fragments one real entity
into several disconnected graph nodes (breaking traversal, since the
graph won't know these are the same person) or incorrectly merges two
distinct entities into one (producing false relationships). This is a
genuinely hard sub-problem, worth explicit attention and evaluation
(Section 6.6), not an incidental detail — entity-resolution errors are
the single most common source of a knowledge graph silently producing
wrong answers despite each individual extraction being reasonable.

### 6.4 Graph traversal and query patterns

Once a graph is built, answering a query requires: (1) identifying the
relevant starting entity or entities (often via the same vector/hybrid
retrieval already built in Modules 1-4, applied to entity descriptions
rather than document chunks — these mechanisms are complementary, not
competing, exactly as Section 6.5 argues), and (2) traversing the graph
from that starting point following the relationship types the query
implies, up to some bounded number of "hops," collecting the entities
and facts encountered along the traversal path as the grounding
material for generation (Part 3, Module 10's grounding discipline,
applied to graph-derived facts instead of retrieved chunks).

**A genuinely useful design pattern**: use an LLM call (schema-
constrained, per Module 2) to translate a natural-language query into an
explicit graph-traversal specification (which starting entity, which
relationship types to follow, how many hops) — this is structurally
identical to Part 3, Module 3's tool-calling pattern: the model decides
*what* traversal is needed, your code *executes* the traversal
deterministically against the actual graph structure, never trusting the
model to "imagine" the traversal result itself.

### 6.5 When to use graph retrieval vs. vector/hybrid retrieval — routing, not replacement

Consistent with every "should I add this more sophisticated technique"
question in Part 4 so far: **knowledge-graph retrieval is not a
replacement for vector/hybrid retrieval — it answers a genuinely
different class of query**, and the two should be combined via an
explicit routing decision, directly extending Part 3, Module 9's
routing discipline. A query asking "what does our refund policy say
about damaged goods" is a pure relevance-matching question, well-served
by Modules 1-4's stack with no need for graph traversal. A query asking
"which of our current vendors were also flagged in last year's
compliance review" is a genuinely relational, multi-hop question, poorly
served (or entirely unanswerable) by chunk-based retrieval alone.

**Building this routing decision reliably** requires the same
difficulty-classification discipline as LLM routing (Part 3, Module 9,
Section 6.4) — a lightweight classification step (rule-based or a small,
fast model call) that determines whether a query's phrasing implies a
relational, multi-hop structure ("who," "which... that also," "the
person who... and then...") versus a pure content-relevance question,
routing accordingly, and — per Part 3, Module 9's core discipline —
**measuring the routing decision's accuracy explicitly**, not just
assuming the classifier gets it right.

### 6.6 Evaluating knowledge-graph retrieval — and being honest about the extraction-quality dependency

Knowledge-graph retrieval's value must be measured with a golden query
set specifically constructed with genuine multi-hop relational
questions (not reused from Modules 1-5's chunk-retrieval golden sets,
which were never designed to test this capability) — comparing whether
the graph-based path correctly answers these questions where the
chunk-based stack demonstrably cannot, exactly the "measure the specific
claimed benefit, don't assume it" discipline running throughout Part 4.

**Additionally, and specific to this module**: because the entire
graph's usefulness depends on Section 6.3's extraction quality, your
evaluation must separately measure **extraction accuracy** (are entities
and relationships correctly identified, with entity resolution working
correctly) as a distinct metric from **traversal/query accuracy** (given
a correctly-built graph, does the traversal mechanism correctly answer
the query) — conflating these hides whether a wrong answer stems from a
bad extraction (the graph itself is wrong) or a bad traversal query
(the graph is right, but the query-to-traversal translation failed) —
precisely the same stage-attribution discipline Part 3, Module 5
established for pipeline evaluation, applied here to the graph-
construction and graph-query stages specifically.

---

## 7. Mental Models

1. **"Chunk-based retrieval scores independent spans of text; knowledge
   graphs represent explicit relationships between entities."** These
   are structurally different data models solving structurally different
   problems — one is not a more-sophisticated version of the other.
2. **"No single chunk contains the answer to a genuinely multi-hop
   question — the answer lives in the relationship between two facts in
   different documents."** This is a category mismatch chunk-retrieval
   tuning cannot fix, no matter how good.
3. **"Extraction, not traversal, is the hard part."** Building an
   accurate graph from unstructured text — especially entity resolution
   — is the genuinely difficult engineering problem; querying an
   already-correct graph is comparatively straightforward.
4. **"Route between vector/hybrid and graph retrieval by query type,
   measured — don't assume graphs are a universal upgrade."** Most
   queries don't need relational traversal; use it specifically where
   evidence shows chunk-based retrieval structurally cannot answer.

---

## 8. Visual Explanation

**Diagram 1 — Why chunk retrieval fails multi-hop queries**

```
Query: "Who is the manager of the person who wrote the Q3 budget proposal?"

Chunk A (budget doc): "...prepared by Jane Smith, Finance..."
Chunk B (org chart):  "...Jane Smith reports to Raj Patel, VP Finance..."

VECTOR/HYBRID RETRIEVAL: scores Chunk A and Chunk B INDEPENDENTLY.
Neither chunk alone answers the query. No mechanism in Modules 1-5
COMBINES them via the "reports to" relationship — that relationship
is never represented anywhere in a chunk-based system.

KNOWLEDGE GRAPH:
  [Jane Smith] --wrote--> [Q3 Budget Proposal]
  [Jane Smith] --reports_to--> [Raj Patel]

Traversal: find entity that "wrote" the proposal (Jane Smith) →
           follow "reports_to" edge → Raj Patel.  ANSWER FOUND.
```

**Diagram 2 — Entity/relationship extraction pipeline**

```
[raw document] ──► [schema-constrained extraction, Module 2 style]
                            │
              Entity[]: {name, type, source_reference}
              Relationship[]: {source_entity, target_entity, type}
                            │
                            ▼
                  [ENTITY RESOLUTION]
          "J. Smith", "Jane Smith", "the VP of Finance's report"
                     ──► all map to ONE node
                            │
                            ▼
                    [Knowledge Graph]
             (nodes = resolved entities,
              edges = typed relationships)
```

**Diagram 3 — Routing between retrieval mechanisms**

```
Incoming query
      │
      ▼
[difficulty/type classifier — Part 3 Module 9 style]
      │
  ┌───┴────────────────────┐
  ▼                        ▼
"pure content-        "relational, multi-hop
 relevance" query      structure implied"
  │                        │
  ▼                        ▼
Vector/Hybrid/Rerank   Knowledge-graph traversal
(Modules 1-4)          (this module)
  │                        │
  └───────────┬────────────┘
              ▼
    grounding material for generation
    (Part 3, Module 10's mechanism, either source)
```

---

## 9. Recommended Resources

1. **A well-cited paper or technical report on GraphRAG / knowledge-
   graph-augmented retrieval** (search for a well-known GraphRAG
   architecture paper or technical report from a major AI lab) — read
   for a real, production-oriented architecture combining chunk-based
   and graph-based retrieval, directly relevant to Section 6.5's
   routing/combination argument.
2. **A well-documented graph database's official documentation** (e.g.,
   Neo4j's documentation on the property graph model and Cypher query
   language, or an equivalent graph database) — read for the concrete
   data model and query mechanics underlying Section 6.2 and 6.4.
3. **Research or engineering writeups on entity resolution / entity
   linking** (search for a well-cited paper or practical guide on named
   entity resolution) — read specifically for how real systems handle
   the "same entity, different surface forms" problem from Section 6.3,
   since this is the genuinely hard part of the whole module.
4. **Your own Part 3, Module 2 (Structured Outputs) and Module 4
   (Memory's structured-extraction discipline) codebases** — the direct
   technical foundation for this module's extraction pipeline; review
   both before implementing entity/relationship extraction.

---

## 10. Exercises

1. **Construct a genuine multi-hop query and prove chunk retrieval
   fails.** Using two real documents from your corpus that are related
   but share no meaningful lexical or semantic overlap (e.g., a project
   document and a separate org-chart-style document), construct a query
   requiring both. Run it against your full Modules 1-4 retrieval stack
   and confirm it cannot produce a correct answer, even with re-ranking
   and hybrid search enabled.
2. **Build a small knowledge graph by hand first.** Before automating
   extraction, manually construct a small graph (10-20 entities, their
   relationships) from a handful of real documents, to build direct
   intuition for the data model before trusting an automated pipeline.
3. **Implement schema-constrained entity/relationship extraction** and
   run it against the same documents from Exercise 2. Compare the
   automated extraction's entities/relationships against your manual
   version, specifically checking for entity-resolution errors (the same
   real entity represented as multiple nodes, or two distinct entities
   incorrectly merged).
4. **Answer Exercise 1's multi-hop query via graph traversal.**
   Implement the traversal mechanism against your constructed graph and
   confirm it correctly answers the query that defeated chunk-based
   retrieval in Exercise 1.
5. **Build and measure a query-type router.** Construct a golden query
   set mixing pure-relevance and genuine multi-hop questions. Implement
   a routing classifier deciding which retrieval mechanism to use, and
   measure its classification accuracy explicitly, not just the
   downstream answer quality.

---

## 11. Mini-Project: `graph-extraction-eval-bench`

A small standalone tool, extending `eval-framework`, that measures
extraction accuracy (entity/relationship correctness, including
entity-resolution correctness) separately from traversal/query accuracy,
against a small, manually-verified golden graph built from real corpus
documents — the direct evaluation discipline Section 6.6 requires,
distinguishing "the graph is wrong" from "the query against a correct
graph is wrong" as genuinely separate failure modes.

---

## 12. Production Project: Knowledge-Graph Retrieval in `llm-client-core`

### 12.1 What you're building

1. **An entity/relationship extraction pipeline**, running through
   `job-processor` (Part 1, Module 6) alongside Module 1's chunking/
   embedding ingestion pipeline, using schema-constrained extraction
   (Part 3, Module 2) with explicit entity-resolution logic (at minimum,
   a similarity-based or rule-based approach to merging likely-duplicate
   entities — acknowledging this is imperfect and measured, not assumed
   perfect, per Section 6.6).

2. **A `KnowledgeGraphStore`** — a graph database or equivalent structure
   storing the extracted, resolved entities and relationships, with a
   traversal query interface.

3. **A query-to-traversal translation component**, using schema-
   constrained output (Module 2) to convert a natural-language query
   into an explicit traversal specification (starting entity, relationship
   types, hop count), with your code executing the traversal
   deterministically against `KnowledgeGraphStore` — never trusting the
   model to imagine the traversal result.

4. **A routing layer** deciding between `HybridRetriever`/`Reranker`
   (Modules 3-4) and `KnowledgeGraphStore` traversal based on query type,
   extending Part 3, Module 9's routing/classification discipline, with
   explicit, measured classification accuracy.

5. **`graph-extraction-eval-bench` integration**, measuring extraction
   accuracy and traversal accuracy separately, plus a comparative
   evaluation showing genuine multi-hop queries are correctly answered
   via the graph path where the chunk-based path demonstrably fails
   (Exercise 1's proof, now formalized as a permanent regression test).

### 12.2 What this sets up for later modules

- **Part 4's Long Context / Context Compression module** will need to
  account for graph-traversal-derived grounding material alongside
  chunk-based material in the overall context budget.
- **Part 4's capstone production architecture** assembles
  `KnowledgeGraphStore` and its routing layer as an optional,
  evidence-justified component of the complete RAG pipeline — used only
  where your corpus and query patterns genuinely warrant it.
- **Part 5 (Agents)** may use graph traversal as one of several tools
  available to an autonomous agent for multi-step relational reasoning.

### 12.3 Definition of done

- The extraction pipeline processes real corpus documents through
  `job-processor`, producing entities and relationships with explicit,
  measured entity-resolution handling.
- `graph-extraction-eval-bench` reports extraction accuracy and
  traversal accuracy as separate, distinct metrics.
- At least one genuine multi-hop query (from Exercise 1) is correctly
  answered via graph traversal, formalized as a permanent regression
  test, with a documented comparison showing chunk-based retrieval
  cannot answer it.
- The routing layer's classification accuracy (graph vs. chunk-based) is
  explicitly measured, not assumed.
- A documented, evidence-based decision exists for whether and how
  extensively knowledge-graph retrieval is used for your specific
  corpus, given its real extraction-quality and maintenance costs.

---

## 13. Practice & Interview Questions

1. Explain precisely why vector/hybrid retrieval cannot answer a genuine
   multi-hop relational query, tracing the explanation to what a chunk
   embedding actually represents.
2. Describe the entity/relationship graph data model and how it differs
   from a chunk-based vector index.
3. Why is entity resolution the hardest part of building a knowledge
   graph from unstructured documents, and what goes wrong if it's done
   poorly?
4. Design a routing mechanism for deciding between chunk-based and
   graph-based retrieval for a given query, and explain how you'd
   measure whether that routing decision is actually accurate.
5. Why must extraction accuracy and traversal/query accuracy be measured
   as separate metrics, rather than one combined "did the graph answer
   correctly" score?
6. Under what circumstances would you recommend against building a
   knowledge graph for a RAG system, even though it's technically
   capable of answering a broader class of query?

---

## 14. Common Mistakes

- **Assuming knowledge-graph retrieval is a strict upgrade over chunk-
  based retrieval** and routing every query through it, rather than
  reserving it specifically for genuinely relational, multi-hop
  questions where chunk-based retrieval structurally cannot succeed.
- **Underestimating entity resolution's difficulty**, treating
  extraction as "mostly solved" once entities and relationships are
  extracted, without separately verifying that different surface forms
  of the same entity are correctly merged.
- **Using unstructured, free-text extraction prompts** instead of
  schema-constrained extraction (Module 2), reintroducing the omission/
  confabulation risks Part 3, Module 4 already warned against.
- **Trusting the model to "imagine" a graph traversal result** instead
  of translating the query into an explicit traversal specification and
  executing it deterministically against the actual graph — precisely
  the tool-calling discipline (model decides, code executes) from Part
  3, Module 3, applied here.
- **Conflating extraction accuracy and traversal accuracy into one
  metric**, hiding which stage is actually responsible for a wrong
  answer.
- **Building a knowledge graph for a corpus/query pattern that doesn't
  actually need relational reasoning**, adding real extraction and
  maintenance cost without evidenced benefit.

---

## 15. Debugging Exercise

Your knowledge-graph retrieval path correctly answers most multi-hop
test queries in `graph-extraction-eval-bench`, but a specific class of
query — ones involving a person referenced by a nickname or informal
title in some documents and their full name in others — consistently
fails, producing an "entity not found" or an incomplete traversal result.

Write down at least two concrete hypotheses for this specific failure
pattern (consider: is this an entity-resolution gap — does your
resolution logic actually handle nickname/informal-title variants, or
only near-exact string matches? could the query-to-traversal translation
step itself be failing to identify the correct starting entity when the
query uses yet another surface form of the person's name that appears in
neither document — a third variant your resolution logic has never
seen?), and describe concretely how you'd extend `graph-extraction-
eval-bench`'s golden dataset to explicitly test entity-resolution
robustness across multiple surface-form variants, rather than only
testing with a single canonical name per entity.

---

## 16. Checklist

- [ ] I can explain precisely why chunk-based retrieval cannot answer
      genuine multi-hop relational queries.
- [ ] I can describe the entity/relationship graph data model and how it
      differs from a chunk-based index.
- [ ] I understand why entity resolution is the hardest part of graph
      construction and what fails when it's done poorly.
- [ ] I can design a routing decision between chunk-based and graph-based
      retrieval, with explicitly measured classification accuracy.
- [ ] I understand why extraction accuracy and traversal accuracy must
      be measured as separate, distinct metrics.
- [ ] `graph-extraction-eval-bench` is built and reports both metrics
      separately against a real, manually-verified golden graph.
- [ ] The extraction pipeline runs through `job-processor` with explicit,
      measured entity-resolution handling.
- [ ] At least one genuine multi-hop query is correctly answered via
      graph traversal, formalized as a permanent regression test.
- [ ] The routing layer's accuracy is explicitly measured.
- [ ] A documented, evidence-based decision exists for the extent of
      knowledge-graph usage in my specific system.

---

## 17. Summary

Vector and hybrid retrieval score independent spans of text and
structurally cannot combine facts across documents via an explicit
relationship — a genuine, unfixable category mismatch for multi-hop
relational queries, not a tuning problem. A knowledge graph solves this
by representing entities and typed relationships explicitly, enabling
graph traversal to answer exactly this class of question — but building
an accurate graph from unstructured documents is the genuinely hard part,
particularly entity resolution (recognizing that different surface forms
refer to the same underlying entity), and this must be extracted with
the same schema-constrained discipline established in Part 3 rather than
unreliable free-text parsing. Knowledge-graph retrieval is not a
replacement for chunk-based retrieval — it answers a structurally
different class of query, and the two should be combined via an explicit,
measured routing decision, exactly like Part 3, Module 9's model-routing
discipline. `llm-client-core` now has a real extraction pipeline, a
`KnowledgeGraphStore` with deterministic traversal execution, and a
routing layer choosing between chunk-based and graph-based retrieval
based on measured evidence — with extraction accuracy and traversal
accuracy evaluated as genuinely separate metrics.

---

## 18. Next Steps

**Next: Part 4, Module 7 — Long Context & Context Compression for
Retrieval.** With multiple retrieval mechanisms now producing grounding
material (chunks, re-ranked results, graph-traversal facts), this module
confronts the question of how much of it can actually be usefully
included given Part 3, Module 7's context-budget constraints — extending
that module's compression techniques specifically for retrieval-heavy
RAG contexts, where the volume of potentially-relevant material often
exceeds what any context window can productively hold.

Reply "continue" for Module 7, or flag anything to go deeper on.
