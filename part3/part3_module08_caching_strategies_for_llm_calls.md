# Part 3, Module 8: Caching Strategies for LLM Calls

> Module 7 mapped which context consumers are stable (system prompts,
> tool definitions) versus inherently dynamic (conversation history,
> retrieved memory). This module turns that stability analysis into real
> cost and latency savings — three distinct caching mechanisms, each
> targeting a different layer of the pipeline, built as a direct
> extension of `smart-cache` (Part 1, Module 5) rather than a bolt-on.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Distinguish three genuinely different caching mechanisms relevant to
   an LLM pipeline — provider-side prompt/prefix caching, application-
   level response caching, and embedding caching — and explain why each
   requires a different cache-key design and has a different validity
   lifetime.
2. Explain, mechanistically, how provider-side prompt caching works (what
   gets cached, at what granularity, and why prefix stability matters)
   and connect it directly to Module 7's consumer-stability analysis.
3. Design cache keys for LLM response caching that correctly account for
   semantic equivalence versus exact-match requirements, and recognize
   when naive exact-match caching under- or over-generalizes.
4. Apply `smart-cache`'s existing stampede-protection and probabilistic-
   early-expiration mechanisms (Part 1, Module 5) to the LLM-specific
   case, including the added risk of caching a non-deterministic
   generation as if it were deterministic.
5. Reason correctly about cache invalidation for LLM-related caches,
   including the specific risk of a cached response becoming stale
   relative to updated tools, updated retrieved memory, or updated system
   prompts.
6. Extend `smart-cache` and `llm-client-core` with all three caching
   layers, measured for real cost/latency impact and correctness (no
   silently stale or wrongly-shared cached responses) via
   `eval-framework` and `observability-stack`.

---

## 2. Prerequisites

- **Part 1, Module 5** (`smart-cache`) — this module extends that exact
  package; you need its stampede-protection, versioned-key, and
  probabilistic-early-expiration mechanisms fresh.
- **Part 3, Module 7** (Context Engineering) — the consumer-stability
  analysis (which context pieces are stable versus dynamic) directly
  determines what's cacheable and at what layer.
- **Part 2, Module 4/5** (Embeddings & Tokenization) — `embedding-
  service`'s embedding generation is what embedding caching targets.
- **Part 3, Module 4** (Memory) — the `LongTermMemoryStore`'s retrieval
  layer already uses `smart-cache` for embedding lookups; this module
  generalizes and formalizes that usage.

---

## 3. Estimated Study Time

**6–8 hours** (2 hours theory/reading, 4–6 hours hands-on).

---

## 4. Difficulty

★★★☆☆ (3/5)

You already built the hard parts of caching correctness (stampede
protection, versioned keys) in Part 1, Module 5. The new difficulty here
is entirely domain-specific: correctly reasoning about what "the same
request" means for a stochastic, context-dependent system, which is a
genuinely different problem than caching a deterministic database query.

---

## 5. Why This Matters

LLM API calls are the single most expensive per-operation cost in most
AI-powered systems, by a wide margin compared to a typical database query
or cache hit — and unlike a typical backend service, that cost recurs
per-token, per-call, at a scale where even a modest cache-hit rate
translates into substantial savings at production traffic volumes.
Getting caching wrong in the other direction — serving a stale or
incorrectly-matched cached response — is also uniquely damaging for an
LLM system specifically, because a cached response that no longer
matches the actual current context (updated tools, updated retrieved
memory, a changed system prompt) can produce confidently wrong output
with no visible sign that anything is stale, unlike a cached database
row that's at least recognizably the same *kind* of data. This module is
where your Part 1 caching discipline meets the specific correctness
hazards of caching probabilistic, context-sensitive generation.

---

## 6. Theory

### 6.1 Three distinct caching layers — do not conflate them

**Layer 1 — Provider-side prompt/prefix caching.** Both Anthropic and
OpenAI offer a mechanism where a stable prefix portion of your prompt
(system prompt, tool definitions, a long stable document) can be cached
*on the provider's infrastructure* across calls, so that subsequent
calls sharing that exact prefix skip re-processing it, reducing both
cost and latency for the cached portion. **What it caches**: the
*computed representation* of a token prefix, not the final output — the
model still generates a fresh response each time; only the cost of
processing the shared prefix is amortized. **Key requirement**: the
cached prefix must be **byte-for-byte identical** across calls up to the
cache boundary — this is exactly why Module 7's consumer-stability
analysis matters: your system prompt and tool definitions (Module 7's
"fixed reserve" and largely-stable consumers) are excellent candidates;
your conversation history and retrieved memory (dynamic per-call) are
not, and placing them *before* the cache boundary in your prefix
assembly would break the cache on every call.

**Layer 2 — Application-level response caching.** This is what
`smart-cache` (Part 1, Module 5) already does for other kinds of data,
applied here to full LLM responses: if you've already generated a
response for effectively the same request, serve the cached response
instead of calling the model again, skipping cost and latency entirely
for that call. **What it caches**: the *final output*, keyed on some
representation of the request. **Key requirement, and the hard part**:
correctly defining what "effectively the same request" means (Section
6.3) — this is the layer where naive caching goes wrong most often.

**Layer 3 — Embedding caching.** Already partially built in Module 4's
`LongTermMemoryStore`: caching the embedding vector for a given piece of
text, since re-embedding identical text repeatedly is wasted
computation, and embeddings (unlike LLM generations) are deterministic
for a fixed model version, which makes this the *simplest* of the three
layers to reason about correctness for.

**Why keeping these three distinct matters**: they have different
granularities (a prefix segment vs. a full response vs. a single
embedding vector), different validity lifetimes (a model-version-scoped
embedding cache can be very long-lived; a response cache tied to dynamic
retrieved memory should not be), and different correctness risks
(Section 6.2's determinism problem applies acutely to Layer 2, barely at
all to Layer 3, and not at all in the same way to Layer 1 since Layer 1
doesn't cache the output). Conflating them into a single generic
"LLM cache" invites exactly the wrong invalidation and key-design
decisions for at least one of the three layers.

### 6.2 The determinism problem: caching non-deterministic generation

`smart-cache`'s original design (Part 1, Module 5) assumed caching a
computation that, for a given input, has one correct, stable output — a
database query, a computed aggregate. **LLM generation is not that**:
even with an identical prompt, sampling temperature > 0 means repeated
calls can produce genuinely different valid outputs (Module 1, Section
6.1's sampling step). This creates a real design tension specific to
Layer 2 response caching: caching *does* mean you'll serve the exact same
response to what your key considers "the same request" every time,
collapsing the natural variation the model would otherwise produce. For
some use cases this is exactly what you want (a deterministic-feeling
FAQ-style answer, reproducibility for testing); for others (creative
generation, anything where response diversity itself has value) it's an
unwanted side effect you should recognize and explicitly decide about,
not stumble into by caching everything uniformly.

**Practical rule**: Layer 2 response caching is the right default for
low-temperature, fact-retrieval-style calls where a stable answer is
actually desirable, and a poor default for high-temperature, creative,
or intentionally-varied generation — make this an explicit, per-call-type
policy decision (extending the same "explicit policy, not blanket
default" discipline from Module 6's guardrail placement and Module 7's
budget allocation), not a system-wide default applied uniformly.

### 6.3 Cache key design: exact match vs. semantic equivalence

The naive Layer 2 cache key — hash the exact rendered prompt string —
under-generalizes badly: two user queries that are semantically
identical but phrased slightly differently ("what's your refund policy"
vs. "how do refunds work") produce completely different hashes and never
hit each other's cache entry, even though a cached answer to one would
serve the other perfectly well.

**The correction, using tools you already have**: for Layer 2 caching
where semantic equivalence matters, key on (or additionally check
against) an **embedding-based similarity** to existing cached
requests — reusing `embedding-service` (Part 2, Module 4) and Layer 3's
embedding cache directly — rather than an exact string hash alone. This
converts response caching from an exact-match lookup into a similarity-
threshold lookup: if a new request's embedding is sufficiently close to
a cached request's embedding (and, critically, the *rest* of the
context — system prompt version, available tools, any retrieved memory
— is also unchanged, per Section 6.4), serve the cached response;
otherwise, generate fresh.

**The risk in the other direction — over-generalizing**: an overly
loose similarity threshold will serve a cached response to a request
that's *superficially* similar but actually requires a different answer
(e.g., "what's the refund policy for electronics" vs. "what's the refund
policy for software" — textually and semantically close, but the
correct answers genuinely differ). Tune this threshold empirically, via
`eval-framework`, against a golden set specifically constructed to
include both true near-duplicates (should hit cache) and superficially-
similar-but-distinct requests (should not) — this is structurally the
same false-positive/false-negative discipline Module 6 already
established for guardrails, applied here to cache-hit decisions instead.

### 6.4 Invalidation: the LLM-specific staleness risks

Beyond ordinary cache-invalidation concerns (Part 1, Module 5's
versioned-key approach already covers "the underlying data changed"),
LLM response caching has staleness risks specific to this pipeline:

- **System prompt or tool-definition changes** (Module 1, Module 3)
  invalidate any Layer 2 cache keyed only on the user's query, because
  the *same* user query against a *different* system prompt or tool set
  can and should produce a different response — your cache key must
  incorporate a version identifier for these components, not just the
  user-facing query text.
- **Retrieved memory changes** (Module 4) — a cached response generated
  when a particular memory was retrieved is not valid once that memory
  has been superseded by Module 4's consolidation logic; a cache key
  that ignores which memories were part of the original context risks
  serving an answer based on now-outdated personal information.
- **Guardrail policy changes** (Module 6) — if a guardrail's behavior
  changes (a stricter or looser threshold), a previously-cached response
  that would now be blocked (or vice versa) should not continue being
  served from cache without re-evaluation.

**The general principle**: your Layer 2 cache key must include a version/
fingerprint of *every* upstream component that could change the correct
answer for the "same" user-facing query — not just the query text
itself. This is a direct, LLM-specific instance of `smart-cache`'s
existing versioned-key discipline (Part 1, Module 5), applied to a
wider and more implicit set of dependencies than a typical cached
database query has.

### 6.5 Stampede protection, still needed, same mechanism

Nothing about the LLM-specific context changes the core stampede-
protection problem `smart-cache` already solved (Part 1, Module 5): if a
popular query's cache entry expires and many concurrent requests arrive
simultaneously, without protection they'll all independently trigger
expensive (here, LLM-call-expensive rather than database-expensive, but
structurally identical) regeneration. Reuse `smart-cache`'s existing
locking/single-flight mechanism unchanged — this is exactly the kind of
component that shouldn't be reimplemented for a new domain when the
underlying problem is identical.

---

## 7. Mental Models

1. **"Three caches, three different things being cached."** Prefix
   caching caches a computed representation of stable input; response
   caching caches a final output; embedding caching caches a
   deterministic vector — conflating them leads to wrong key design and
   wrong invalidation logic for at least one.
2. **"Caching a stochastic process is a policy decision, not a free
   optimization."** Response caching collapses the model's natural
   variation — decide explicitly whether that's desirable for a given
   call type, per temperature and use case.
3. **"A cache key for LLM responses must capture everything upstream
   that could change the right answer."** Not just the query text — the
   system prompt version, the tool set, and any retrieved memory are all
   part of what makes two requests "the same."
4. **"Prefix caching rewards exactly the consumer-stability discipline
   Module 7 already taught."** Put stable content first and dynamic
   content after the cache boundary, or the cache never hits.

---

## 8. Visual Explanation

**Diagram 1 — Three caching layers at different pipeline points**

```
┌──────────────┐   ┌──────────────────┐   ┌───────────────┐
│  LAYER 1:     │   │  LAYER 3:         │   │  LAYER 2:      │
│  Prefix cache │   │  Embedding cache  │   │  Response cache│
│  (provider-   │   │  (text → vector,  │   │  (query+ctx → │
│   side)       │   │   deterministic)  │   │   final answer)│
├──────────────┤   ├──────────────────┤   ├───────────────┤
│ caches:        │   │ caches:            │   │ caches:        │
│ computed rep   │   │ embedding vectors  │   │ full generated │
│ of a STABLE    │   │ for repeated text  │   │ response, keyed│
│ prefix segment │   │ (memory lookups,   │   │ on query +     │
│                │   │  retrieval)        │   │ upstream deps  │
└──────────────┘   └──────────────────┘   └───────────────┘
   saves: cost/          saves: embedding        saves: the ENTIRE
   latency for the       compute for              generation call
   cached prefix          repeated text            (biggest win,
   portion only                                    highest risk)
```

**Diagram 2 — Response cache key must capture upstream dependencies**

```
NAIVE KEY:            hash(user_query)
                             │
                    MISSES changes in:
                    - system prompt version
                    - available tool set
                    - retrieved memory content
                    → serves STALE response after any of these change

CORRECT KEY:  hash(user_query_embedding_bucket,
                    system_prompt_version,
                    tool_set_version,
                    retrieved_memory_fingerprint)
                             │
                    correctly invalidates when
                    any real upstream dependency changes
```

**Diagram 3 — Similarity threshold tuning (false-positive/negative tradeoff)**

```
Threshold TOO LOOSE:
  "refund policy electronics" ≈ "refund policy software"  → WRONGLY
  cached answer served for the wrong product category (false positive
  cache hit — same risk shape as Module 6's guardrail false positives)

Threshold TOO STRICT:
  "what's your refund policy" ≠ "how do refunds work"  → cache MISS
  on a genuine near-duplicate, losing the savings entirely
  (false negative — under-utilized cache)

Tune empirically against a golden set with BOTH true near-duplicates
and superficially-similar-but-distinct pairs, exactly per Module 6's
discipline.
```

---

## 9. Recommended Resources

1. **Anthropic — "Prompt caching" documentation** (docs.claude.com) —
   the authoritative reference for Layer 1's exact mechanics
   (cache-breakpoint placement, minimum cacheable prefix length, cache
   lifetime) on the provider you're building against.
2. **OpenAI — prompt caching / "automatic caching" documentation**
   (platform.openai.com/docs) — the equivalent reference for OpenAI;
   read specifically for how its caching triggers differ from
   Anthropic's explicit-breakpoint approach, since your adapter needs to
   account for both.
3. **Your own `smart-cache` codebase (Part 1, Module 5)** — the primary
   resource for this module is genuinely your own prior work; review its
   stampede-protection and versioned-key implementation before extending
   it, since Section 6.5 explicitly reuses it unchanged.
4. **Anthropic or OpenAI cookbook examples on semantic caching /
   response caching** (github.com cookbook repos) — concrete, runnable
   patterns for Layer 2's embedding-similarity-based cache-hit logic to
   compare against your own design.

---

## 10. Exercises

1. **Measure Layer 1's real impact.** Take a call in your pipeline with
   a stable system prompt and tool-definition set (Module 3). Enable
   provider-side prefix caching and measure the actual latency/cost
   difference on repeated calls sharing that prefix, versus without
   caching enabled.
2. **Break Layer 1 on purpose.** Move one piece of dynamic content
   (e.g., the current timestamp, or a per-call unique ID) to *before*
   the cache boundary in your prefix assembly. Confirm the cache no
   longer hits, and explain precisely why using Section 6.1's "byte-for-
   byte identical" requirement.
3. **Build and tune Layer 2's similarity threshold.** Implement
   embedding-based response caching for a low-temperature, fact-
   retrieval-style call type. Construct a golden set with true
   near-duplicate query pairs and superficially-similar-but-distinct
   pairs (Section 6.3), and tune your similarity threshold against
   measured false-positive and false-negative cache-hit rates.
4. **Reproduce a staleness bug, deliberately.** Cache a response keyed
   only on user query text (the naive key from Diagram 2). Change the
   system prompt, re-issue the same query, and confirm you get a stale
   cached response. Fix the key design per Section 6.4 and confirm the
   cache correctly misses after the same change.
5. **Decide the determinism policy, explicitly.** For at least two
   different call types in your pipeline (one low-temperature/fact-
   style, one higher-temperature/creative-style), write down and justify
   whether Layer 2 response caching should be enabled for each,
   referencing Section 6.2's tradeoff directly rather than defaulting to
   "cache everything" or "cache nothing."

---

## 11. Mini-Project: `cache-hit-bench`

A small standalone tool, built against `eval-framework`, that measures
Layer 2 response-cache effectiveness on a golden set containing both true
near-duplicate query pairs and superficially-similar-but-distinct pairs,
reporting cache-hit rate, false-positive-hit rate (wrongly served a
mismatched cached response), and the actual cost/latency savings
achieved — the direct measurement tool for tuning the similarity
threshold in Exercise 3, built once and reusable for any future
similarity-threshold tuning need in the handbook.

---

## 12. Production Project: All Three Caching Layers in `llm-client-core` + `smart-cache`

### 12.1 What you're building

1. **Layer 1 — Prefix caching enabled and correctly structured**: ensure
   `PromptTemplate` rendering (Module 1) places stable content (system
   prompt, tool definitions) before dynamic content (conversation
   history, retrieved memory, current query) in the assembled prefix,
   with the provider-specific cache-breakpoint mechanism explicitly
   configured for both `AnthropicAdapter` and `OpenAIAdapter`.

2. **Layer 2 — Response caching extending `smart-cache`**: a new cache
   type using an embedding-similarity-based key (Section 6.3) that
   additionally incorporates version fingerprints for the system prompt,
   tool set, and any retrieved memory (Section 6.4), reusing
   `smart-cache`'s existing stampede-protection mechanism (Section 6.5)
   unchanged, with an explicit, per-call-type enable/disable policy
   (Section 6.2) rather than a blanket default.

3. **Layer 3 — Embedding caching formalized**: generalize Module 4's
   ad hoc embedding-cache usage in `LongTermMemoryStore` into a
   reusable, `smart-cache`-backed component usable by any future
   embedding-generating call in the system (directly setting up Part 4's
   RAG embedding needs).

4. **`cache-hit-bench` evaluation for Layer 2**: tune and verify the
   similarity threshold against your golden set, reporting hit rate,
   false-positive-hit rate, and cost/latency savings, integrated into
   pipeline CI (Module 5) so a future change doesn't silently degrade
   cache correctness.

5. **Observability**: emit per-layer cache-hit-rate and cost-savings
   metrics via `observability-stack` (Part 1, Module 4), distinct per
   layer so you can attribute savings (or staleness incidents) to the
   correct caching mechanism.

### 12.2 What this sets up for later modules

- **Part 3's Latency/Cost Optimization module** uses these three caching
  layers as core levers, alongside model routing and batching.
- **Part 4 (RAG)** reuses Layer 3's formalized embedding cache directly
  for corpus-scale embedding generation.
- **Part 3's capstone** integrates all three caching layers as
  non-optional cost/latency infrastructure in the finished production
  service.

### 12.3 Definition of done

- Layer 1 prefix caching is correctly configured and measurably reduces
  cost/latency on repeated-prefix calls for both provider adapters.
- Layer 2 response caching correctly incorporates all upstream
  dependency versions in its key, verified by the staleness-bug test
  from Exercise 4 (now a permanent regression test).
- `cache-hit-bench` reports hit rate, false-positive-hit rate, and
  savings, with an explicit, tuned similarity threshold.
- Layer 3 embedding caching is generalized into a reusable component.
- Per-layer cache metrics are visible in `observability-stack`.
- Explicit, documented per-call-type policy exists for whether Layer 2
  caching is enabled, per Section 6.2's determinism tradeoff.

---

## 13. Practice & Interview Questions

1. Name the three distinct LLM-related caching layers and explain, for
   each, what specifically gets cached and what its validity lifetime
   depends on.
2. Why is caching an LLM's generated response a fundamentally different
   correctness problem than caching a database query result?
3. A naive response-cache key is `hash(user_query)`. Describe two
   concrete ways this under- or over-generalizes, and how you'd fix each.
4. Why does provider-side prefix caching reward the exact content-
   ordering discipline established in the Context Engineering module?
5. For what kind of LLM call would you deliberately disable response
   caching, and why, using the determinism-tradeoff argument?
6. What upstream dependencies, beyond the literal user query text, must
   a response cache key account for in a pipeline with tool calling and
   long-term memory, and what happens if you omit one?

---

## 14. Common Mistakes

- **Treating "LLM caching" as one undifferentiated concept** instead of
  three distinct mechanisms with different keys, lifetimes, and
  correctness risks.
- **Caching creative/high-temperature generation by default**, silently
  collapsing intended response variation without an explicit policy
  decision.
- **Exact-string-match cache keys for response caching**, missing
  obvious near-duplicate hits that a semantic/embedding-based key would
  catch.
- **Cache keys that omit upstream dependency versions** (system prompt,
  tool set, retrieved memory), risking silently stale responses after
  any of those change — a uniquely hard-to-notice failure mode for LLM
  systems specifically, since the output still "looks like" a plausible
  answer.
- **Placing dynamic content before the prefix-cache boundary**, breaking
  Layer 1 caching without realizing it, since the cache simply misses
  silently rather than erroring.
- **Reimplementing stampede protection from scratch** for the LLM case
  instead of reusing `smart-cache`'s existing, already-correct mechanism.

---

## 15. Debugging Exercise

Your Layer 2 response cache's hit rate looks healthy in aggregate
metrics, but a support team reports a specific pattern: users who ask
about their account status shortly after updating a stored preference
(Module 4's `LongTermMemoryStore`) sometimes get an answer reflecting
their *old* preference, even though the underlying memory was correctly
updated and consolidated.

Write down at least two concrete hypotheses grounded in Sections 6.3 and
6.4 (consider: does your cache key's "retrieved memory fingerprint"
actually update promptly when consolidation supersedes an old memory, or
is there a timing/staleness gap between consolidation and the
fingerprint your cache key computes? could the similarity-threshold
matching in Layer 2 be matching this query against a cached response
from *before* the preference update, if the fingerprint component of
the key isn't granular enough to distinguish the two states?) and
describe how you'd use `cache-hit-bench` and your cache-key logging
together to confirm which is happening before changing the key design.

---

## 16. Checklist

- [ ] I can name and distinguish the three LLM-related caching layers and
      what each actually caches.
- [ ] I can explain why caching a stochastic generation is a policy
      decision, not a free-and-uniform optimization.
- [ ] I can design a response-cache key that correctly incorporates
      semantic similarity and all relevant upstream dependency versions.
- [ ] I understand why prefix-caching effectiveness depends directly on
      Module 7's consumer-stability/ordering discipline.
- [ ] `cache-hit-bench` is built and reports hit rate, false-positive-hit
      rate, and real savings for Layer 2.
- [ ] Layer 1 prefix caching is correctly configured for both provider
      adapters and measurably reduces cost/latency.
- [ ] Layer 2 response caching correctly incorporates upstream
      dependency versions, verified by a passing staleness-regression
      test.
- [ ] Layer 3 embedding caching is generalized into a reusable component.
- [ ] Per-layer cache metrics are visible in `observability-stack`.

---

## 17. Summary

LLM-related caching is genuinely three separate mechanisms, not one:
provider-side prefix caching amortizes the cost of a stable, byte-for-
byte-identical prompt segment (directly rewarding Module 7's consumer-
stability and ordering discipline); application-level response caching
serves an entire generated output for an equivalent request, which
requires confronting the fact that generation is stochastic (a policy
decision about whether collapsing that variation is desirable, not a
free optimization) and that "equivalent request" must be defined by
semantic similarity plus every upstream dependency that could change the
correct answer, not just the literal query text; and embedding caching
is the simplest of the three, since embeddings are deterministic. All
three reuse `smart-cache`'s existing stampede-protection mechanism
unchanged, because that problem is identical regardless of what's being
cached. `llm-client-core` and `smart-cache` now implement all three
layers, measured and tuned via `cache-hit-bench` for real, verified
cost/latency savings without introducing staleness — directly setting up
the latency/cost optimization module that follows.

---

## 18. Next Steps

**Next: Part 3, Module 9 — Latency & Cost Optimization (Streaming,
Batching, Model Routing).** With caching now handling the "don't
recompute what you don't have to" lever, this module covers the
remaining major cost/latency levers for calls that genuinely do need
fresh generation: streaming responses to reduce perceived latency,
batching requests where real-time response isn't required, and routing
different request types to different-sized models based on actual
difficulty — extending this module's cost-measurement discipline into a
complete cost/latency optimization toolkit.

Reply "continue" for Module 9, or flag anything to go deeper on.
