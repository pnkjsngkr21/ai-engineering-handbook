# Part 2, Module 6: Attention & the Transformer Architecture

---

## 1. Learning Objectives

By the end of this module, you will be able to:

- Explain, precisely and mechanically, what "attention" computes and why it solves the specific limitation of static embeddings identified in Module 4.
- Derive scaled dot-product attention from first principles: queries, keys, values, the softmax, and the scaling factor — and explain why each piece exists.
- Implement multi-head self-attention from scratch in NumPy/PyTorch and verify it against a known-correct reference.
- Explain positional encoding and why attention needs it (attention alone is permutation-invariant — a fact most explanations skip).
- Assemble a full minimal transformer block (attention + feed-forward + residual connections + layer norm) and explain the purpose of every sub-component.
- Distinguish encoder-only, decoder-only, and encoder-decoder architectures, and know which modern LLMs (GPT, Claude, BERT, T5) use which and why.
- Extend `embedding-service` into `contextual-embedding-service`, replacing static lookup with real contextual embeddings from a small pretrained transformer.

---

## 2. Prerequisites

- Part 2, Module 4 (Embeddings) — you must deeply understand static embeddings and *why* they fall short for polysemy; this module is the direct fix.
- Part 2, Module 5 (Tokenization) — you need to think in terms of token sequences, not words.
- Part 2, Module 3 (Backpropagation & Training) — matrix calculus and gradient flow intuition, since this is the most architecturally complex module so far.
- Comfortable with matrix multiplication shapes (`(n, d) @ (d, m) = (n, m)`) — this module lives and dies by getting tensor shapes right.

---

## 3. Estimated Study Time

**10–14 hours** — this is the single densest module in Part 2. Budget 3–4 hours for theory (re-read it more than once), 5–6 hours for implementation, 2 hours for exercises. Do not rush this one; everything from Module 7 onward assumes you deeply, not superficially, understand this.

---

## 4. Difficulty

★★★★★ (5/5) — This is the hardest module in the handbook so far, by a clear margin. The individual operations (matrix multiply, softmax) are things you already know. The difficulty is entirely in holding the *whole mechanism* in your head at once and understanding *why* each piece is shaped the way it is.

---

## 5. Why This Matters

Every single production AI system you will ever call — Claude, GPT-4, Gemini, LLaMA, every embedding model, every reranker — is a transformer, or a close variant of one. This is not an exaggeration for pedagogical effect; it is the literal current state of the field. Attention, introduced in the 2017 paper "Attention Is All You Need," is the mechanism that made the current era of AI possible, and it is squarely the reason the field moved as fast as it did after 2017.

Recall the unresolved promise from Module 4: static embeddings give "bank" exactly one vector, blurred across every sense of the word it ever appeared in during training. We said contextual embeddings fix this by computing a different vector "on the fly," conditioned on the surrounding sentence, via attention. This module is where you find out *exactly* how that "on the fly" computation works — not as a black box, but as a small number of matrix multiplications you can trace by hand.

There's also a direct, practical payoff for your backend/system-design brain: attention is fundamentally a **soft, differentiable, learned lookup/routing mechanism** — and once you see it that way, an enormous amount of "AI engineering" (RAG's retrieval step, agent tool selection, multi-head reasoning) reveals itself to be the same underlying pattern applied at different scales: *given a query, figure out which of many candidates are relevant, weight them accordingly, and combine them.* You already do a version of this every time you write a routing layer or a fan-out aggregator (`resilient-gateway`, Part 0 Module 13) that decides which backend services to call and how to weight/combine their responses — attention is that same intuition, made differentiable and trainable.

Get this module solid, and Modules 7–9 (training/fine-tuning, evaluation, reasoning models) — plus all of Parts 3 through 6 — become dramatically easier, because they're all built on top of this one mechanism.

---

## 6. Theory

### 6.1 The Problem, Restated Precisely

You have a sequence of token embeddings from Module 4/5 (let's say a sentence with `n` tokens, each an embedding of dimension `d`): a matrix `X` of shape `(n, d)`. Right now, each row of `X` is *context-free* — `X[3]` (the embedding for "bank") is identical whether the sentence is "I sat by the river bank" or "I deposited cash at the bank."

**We want a function that takes `X` (shape `(n, d)`) and produces a new matrix `X'` (same shape `(n, d)`), where each row `X'[i]` is now a blend of the *entire sequence*, weighted by how relevant every other token is to token `i`.** For "bank" in "river bank," we want that blend to pull heavily from "river" (nearby, relevant); for "bank" in "deposited cash at the bank," we want it to pull from "deposited" and "cash" instead. Crucially, we want the model to **learn** what "relevant" means for a given task, not hand-code it.

That single sentence is the entire goal of attention. Everything below is *how*.

### 6.2 Queries, Keys, and Values — The Core Analogy, Made Precise

The now-famous framing: attention is a **soft dictionary/database lookup**. In a normal `HashMap<Key, Value>` lookup, you provide a `Query`, it's compared for *exact equality* against every `Key`, and you get back the `Value` of the single matching key. Attention generalizes this in two ways: (1) the comparison between query and key is a *continuous similarity score*, not exact match, and (2) instead of returning one value, you return a **weighted average of every value**, weighted by how similar each key was to the query.

Concretely, for each token in the sequence, we compute three different vectors by multiplying its embedding by three different learned weight matrices:

```
Q = X @ W_Q     (n, d) @ (d, d_k) = (n, d_k)     -- "what am I looking for?"
K = X @ W_K     (n, d) @ (d, d_k) = (n, d_k)     -- "what do I contain / offer?"
V = X @ W_V     (n, d) @ (d, d_v) = (n, d_v)     -- "what do I actually pass along if selected?"
```

`W_Q`, `W_K`, `W_V` are learned parameter matrices (exactly the same kind of thing as `W_in`/`W_out` in word2vec — matrices whose values are found via gradient descent, not hand-designed). Note that **Q, K, and V all come from the exact same input `X`** — this is why it's called *self*-attention: the sequence is attending to itself, generating its own queries and keys and values from its own tokens, rather than one sequence querying a *different* sequence (which is called cross-attention, and shows up in encoder-decoder architectures — see section 6.7).

### 6.3 Scaled Dot-Product Attention, Step by Step

This is the exact formula from the original paper, and every step has a specific mechanical purpose:

```
Attention(Q, K, V) = softmax( (Q @ K^T) / sqrt(d_k) ) @ V
```

Let's take this apart term by term, because glossing over any one piece is where most people's understanding stays permanently fuzzy.

**Step 1 — `Q @ K^T`, shape `(n, d_k) @ (d_k, n) = (n, n)`.** This produces a full `n x n` matrix of raw similarity scores — literally the dot product between every query and every key. `scores[i][j]` answers: "how relevant is token `j`'s key to token `i`'s query?" This is precisely the same dot-product-as-similarity operation you used for cosine similarity in Module 1 and Module 4, just not yet normalized to a unit vector.

**Step 2 — divide by `sqrt(d_k)` (the "scaled" part of "scaled dot-product attention").** As `d_k` (the dimension of Q and K) grows, dot products between random vectors grow in magnitude roughly proportional to `sqrt(d_k)` (a direct consequence of summing `d_k` independent terms — variance adds). Large-magnitude scores pushed through a softmax produce an extremely "peaked" (near one-hot) distribution, which in turn produces near-zero gradients almost everywhere (recall the vanishing gradient discussion from Module 3 — a saturated softmax is exactly this problem). Dividing by `sqrt(d_k)` counteracts this growth, keeping scores in a range where softmax gradients remain healthy and training is stable. This single division is a small detail with an outsized practical impact — it's the kind of thing that looks like an arbitrary "magic number" until you see the variance argument, at which point it's obviously necessary.

**Step 3 — `softmax(...)` over each row.** This converts each row of raw, scaled scores into a proper probability distribution (non-negative, sums to 1) over *which other tokens to attend to*. Row `i` of this matrix is literally "how much attention token `i` pays to every other token in the sequence, including itself" — and you can print this matrix and look directly at it; it's fully interpretable, which is why "attention visualization" is a common and genuinely useful debugging/interpretability tool (unlike most of a neural network's internals).

**Step 4 — multiply by `V`, shape `(n, n) @ (n, d_v) = (n, d_v)`.** Each output row is now a weighted average of *every value vector in the sequence*, weighted exactly by the attention distribution computed in step 3. This is the "soft lookup" completing: instead of returning one exact-matched value, you get a blend, weighted by learned relevance.

**The result is your contextual embedding.** Token `i`'s output vector is no longer just its own static embedding — it's a learned, weighted combination of every token in the sequence, where the weights were computed based on genuine content-based relevance (via Q/K dot products), not fixed rules. This is precisely how "bank" gets a different final vector depending on whether "river" or "deposited" is nearby: the attention weights, computed fresh for every input sentence, will differ, and so will the resulting blend.

### 6.4 Multi-Head Attention — Why One Attention Computation Isn't Enough

A single attention computation forces the model to squeeze *every kind of relevant relationship* (syntactic — "this is the subject of that verb," semantic — "this pronoun refers to that noun," positional — "this token is right before that one") into one `d_k`-dimensional similarity space. That's a lot to ask of one subspace.

**Multi-head attention** runs several independent attention computations in parallel, each with its *own* smaller `W_Q`, `W_K`, `W_V` matrices (so each "head" can specialize in a different kind of relationship), then concatenates the results and projects back down to the original dimension:

```
head_i = Attention(X @ W_Q_i, X @ W_K_i, X @ W_V_i)    for i = 1..h heads
MultiHead(X) = concat(head_1, ..., head_h) @ W_O
```

If you have 8 heads and a model dimension `d = 512`, each head typically works with `d_k = d_v = d / h = 64` — so the total compute is comparable to one big attention computation, but the parameters are split into 8 independently-learnable "attention subspaces." Empirically (and somewhat interpretably, via attention visualization), different heads really do learn to specialize — some heads consistently attend to the immediately preceding word, some to the subject of the sentence, some to punctuation boundaries. Nobody designs this specialization — like everything else in this handbook so far, it's an emergent consequence of the training objective rewarding useful, non-redundant representations across heads.

This should feel structurally familiar from your systems background: it's the same principle as running several independent, specialized analyzers over the same input in parallel and combining their outputs (think: an ensemble of specialized microservices analyzing the same request from different angles, then merging results — the "fan-out and aggregate" pattern from `resilient-gateway`, applied inside a single neural network layer.)

### 6.5 The Missing Piece: Attention Has No Sense of Order

Here is a genuinely important fact that most casual explanations skip: **attention, as described so far, is completely permutation-invariant.** If you shuffled the rows of `X` (the input tokens) and correspondingly shuffled the rows/columns of the output, you'd get exactly the same relative computation — attention treats the input as an unordered *set* of tokens, not a *sequence*. There is nothing in `Q @ K^T` that knows token 3 comes before token 5.

This is obviously wrong for language — "dog bites man" and "man bites dog" contain the identical set of tokens but mean opposite things. The fix: **positional encoding** — before tokens are fed into any attention layer, we add a vector to each token's embedding that encodes its position in the sequence:

```
X_input = X_embedding + PositionalEncoding(position)
```

The original transformer paper used a fixed, non-learned sinusoidal encoding (a fixed function of position using sine/cosine waves of different frequencies per dimension — chosen partly because it lets the model generalize to sequence lengths not seen during training, and because relative positions can be recovered via a linear function of the absolute encodings). Many modern models instead use **learned positional embeddings** (a simple lookup table exactly like Module 4's word embedding table, but indexed by position instead of token identity), or more recent schemes like **RoPE (Rotary Position Embeddings)**, which bakes relative position directly into the attention score computation itself rather than adding a separate vector — RoPE is what most current-generation open models (LLaMA, and many others) actually use, precisely because it generalizes better to long contexts. You don't need to implement RoPE by hand to understand this module's core mechanism, but you should know it exists and why it's currently preferred — it will come up in any serious discussion of long-context model design (a preview of Part 4's long-context RAG considerations).

### 6.6 Assembling a Full Transformer Block

Multi-head self-attention alone is not the whole story — a real transformer layer ("block") wraps it with a few more essential components:

```
def transformer_block(x):
    # Sub-layer 1: Multi-head self-attention, with residual connection + layer norm
    attn_out = multi_head_attention(x)
    x = layer_norm(x + attn_out)          # residual connection, then normalize

    # Sub-layer 2: Position-wise feed-forward network, with residual + layer norm
    ff_out = feed_forward(x)               # two linear layers with a nonlinearity between
    x = layer_norm(x + ff_out)

    return x
```

**Residual connections** (`x + attn_out` instead of just `attn_out`) are a direct, deliberate fix for the vanishing-gradient problem you empirically observed in Module 3's `training-loop-visualizer`: they give gradients a direct, unobstructed path backward through the network (the gradient of `x + f(x)` with respect to `x` always includes a clean `+1` term, regardless of how small `f`'s gradient becomes), which is precisely what makes it possible to stack dozens of transformer blocks (real models stack anywhere from a dozen to over a hundred) without training collapsing.

**Layer normalization** rescales activations to have stable mean/variance at each layer, which — similar in spirit to the softmax-scaling argument in 6.3 — keeps training numerically stable as you stack many layers deep. Think of it as the transformer-block-level equivalent of the good input-normalization hygiene you'd apply to any numerical feature before feeding it into a traditional ML model, just applied repeatedly, layer after layer, rather than once at the input.

**The position-wise feed-forward network** is a small, ordinary two-layer neural network (identical in kind to the `neural-net-from-scratch` you built in Module 2), applied *independently to each token's vector* after attention has done its job of mixing information across tokens. If attention is "gather relevant information from across the sequence," the feed-forward layer is "now do some additional per-token processing/transformation on the gathered result." The two sub-layers have genuinely complementary jobs: attention mixes *across* positions, the feed-forward network transforms *within* a position.

A full transformer model is simply **this block, stacked N times** (GPT-3: 96 layers; smaller models: as few as 6–12), with a token embedding + positional encoding layer at the very bottom, and a task-specific output layer at the very top (for a language model: a final linear layer projecting back to vocabulary size, followed by softmax, to predict the next token — this is literally your Module 4 skip-gram output layer's exact structure, reused at the top of a vastly more sophisticated architecture).

### 6.7 Encoder-Only, Decoder-Only, and Encoder-Decoder — Why the Split Exists

The original 2017 transformer paper was designed for machine translation and used **both** an encoder and a decoder. Understanding why later architectures split apart into "just the encoder" or "just the decoder" clarifies a huge amount of confusing terminology you'll see in the wild.

**Encoder-only (e.g., BERT):** every token can attend to *every other token in the sequence, including tokens after it* — this is called **bidirectional** attention. This is ideal when you want a rich, holistic understanding of a complete piece of text you already fully have — exactly the use case for producing contextual embeddings for search/retrieval/classification, which is precisely why encoder-only models are the standard choice for embedding models (including the `sentence-transformers` model you already used in `embedding-service`, which is BERT-derived).

**Decoder-only (e.g., GPT, Claude, LLaMA):** each token can only attend to itself and *tokens before it* in the sequence — this is enforced via a **causal mask** (a modification to step 1 of section 6.3: before the softmax, positions corresponding to "future" tokens have their score set to negative infinity, so softmax assigns them exactly zero probability). This asymmetric restriction is essential for **autoregressive generation** — a model generating text one token at a time cannot be allowed to "see" tokens it hasn't generated yet, or the training objective becomes trivial/cheating (predicting the next word by literally looking at the next word). This is why every modern chat-style LLM you call via API — the entire subject of Part 3 of this handbook — is a decoder-only transformer.

**Encoder-decoder (e.g., the original transformer, T5):** an encoder processes a full input sequence bidirectionally (e.g., a sentence in French), then a decoder generates an output sequence autoregressively (e.g., the English translation), using **cross-attention** — a variant of attention where the decoder's tokens generate the Queries, but the Keys and Values come from the *encoder's* output, not the decoder's own tokens. This architecture remains the natural fit for genuine sequence-to-sequence tasks (translation, summarization-as-rewriting) but has become less dominant than decoder-only models for the general-purpose chat/agent use cases this handbook focuses on.

**The practical takeaway for you as an AI engineer:** when you see "embedding model" → assume encoder-only, bidirectional. When you see "chat model" / "LLM" / "generation" → assume decoder-only, causal-masked, autoregressive. This single distinction resolves a lot of otherwise-confusing architecture diagrams you'll encounter.

---

## 7. Mental Models

1. **"Attention is a soft, differentiable, learned database lookup."** Query asks a question, Keys advertise what each token offers, softmax over dot products decides how much to trust each answer, Values are what actually gets blended in.
2. **"Scaling by `sqrt(d_k)` exists purely to keep the softmax from saturating."** Every "mysterious constant" in a well-designed architecture has a variance/gradient-flow reason — go looking for it before assuming it's arbitrary.
3. **"Residual connections are a gradient highway; layer norm is the guardrail that keeps traffic at a sane speed."** Together, they're *why* you can stack 96 transformer blocks without the whole thing collapsing during training — a direct, deliberate answer to Module 3's vanishing gradient problem.
4. **"Attention mixes information across tokens; the feed-forward layer transforms information within a token."** Two sub-layers, two complementary jobs, repeated N times.
5. **"Bidirectional (encoder) sees the whole page before answering; causal (decoder) reads left to right and can never peek ahead."** This one distinction tells you almost everything about whether an architecture is meant for understanding (embeddings) or generation (chat).

---

## 8. Visual Explanation

**Diagram 1 — Scaled Dot-Product Attention, Full Pipeline**

```
Input X  (n tokens, d dims each)
   │
   ├──> X @ W_Q ──> Q  (n, d_k)
   ├──> X @ W_K ──> K  (n, d_k)
   └──> X @ W_V ──> V  (n, d_v)

           Q @ K^T            (n, n)  -- raw similarity scores
              │
        divide by sqrt(d_k)   -- prevent softmax saturation
              │
          softmax(rows)       (n, n)  -- attention weights, each row sums to 1
              │
              ▼
        attn_weights @ V      (n, d_v)  -- weighted blend of Values
              │
              ▼
      Contextual output for each token
```

**Diagram 2 — Attention Weight Matrix for a Real Sentence (illustrative)**

```
Sentence: "The animal didn't cross the street because it was tired"
Attending FROM token "it" (row), TO every token (columns):

              The  animal  didn't  cross  the  street  because  it  was  tired
"it" row:    0.02   0.71    0.01   0.02  0.01   0.04     0.03   0.10 0.03 0.03
                     ▲▲▲▲
             most attention mass goes to "animal" — the model has learned
             that "it" refers back to "animal," not "street," purely from
             the training objective, with no explicit coreference labels.
```

**Diagram 3 — Multi-Head Attention (parallel, specialized heads)**

```
Input X
   │
   ├──> Head 1 (own W_Q,W_K,W_V) ──> maybe learns "attend to previous word"
   ├──> Head 2 (own W_Q,W_K,W_V) ──> maybe learns "attend to sentence subject"
   ├──> Head 3 (own W_Q,W_K,W_V) ──> maybe learns "attend to punctuation"
   ├──> ...
   └──> Head h (own W_Q,W_K,W_V) ──> maybe learns something uninterpretable
              │
       concat all heads' outputs
              │
           @ W_O  (project back to model dimension d)
              │
              ▼
      Multi-head attention output
```

**Diagram 4 — One Full Transformer Block**

```
        x  (n, d)
        │
        ├────────────────┐
        │                │
        ▼                │
  Multi-Head Attention    │ (residual)
        │                │
        ▼                │
       (+) <──────────────┘
        │
        ▼
    Layer Norm
        │
        ├────────────────┐
        │                │
        ▼                │
  Feed-Forward Network    │ (residual)
        │                │
        ▼                │
       (+) <──────────────┘
        │
        ▼
    Layer Norm
        │
        ▼
   output x' (n, d)  -- same shape as input, stack N of these blocks
```

**Diagram 5 — Encoder-Only vs. Decoder-Only Attention Masks**

```
ENCODER (bidirectional) — every token sees every token:
        The  cat  sat  on  the  mat
  The  [ x    x    x   x    x    x  ]
  cat  [ x    x    x   x    x    x  ]
  sat  [ x    x    x   x    x    x  ]
  ...  all cells allowed (no masking)

DECODER (causal) — each token only sees itself and earlier tokens:
        The  cat  sat  on  the  mat
  The  [ x    -    -   -    -    -  ]
  cat  [ x    x    -   -    -    -  ]
  sat  [ x    x    x   -    -    -  ]
  on   [ x    x    x   x    -    -  ]
  the  [ x    x    x   x    x    -  ]
  mat  [ x    x    x   x    x    x  ]
        '-' = masked to -infinity before softmax → 0 probability
```

---

## 9. Recommended Resources

1. **[Vaswani et al. — "Attention Is All You Need" (2017)](https://arxiv.org/abs/1706.03762)** — The original transformer paper. Read this now, even if some notation is initially unfamiliar — you have every prerequisite concept already, and reading the primary source that started this entire era is directly worth the friction.
2. **[Jay Alammar — "The Illustrated Transformer"](https://jalammar.github.io/illustrated-transformer/)** — The best visual companion to the paper; read this alongside or immediately after the paper if any diagram in this module needs more repetition to click.
3. **[Andrej Karpathy — "Let's build GPT: from scratch, in code, spelled out" (video)](https://www.youtube.com/watch?v=kCc8FmEb1nY)** — Karpathy builds a small decoder-only transformer completely from scratch, line by line, in a single sitting; this is the single best resource for turning this module's theory into working code, and directly informs this module's mini-project.
4. **[Stanford CS224N, Lecture on Transformers (Chris Manning)](https://web.stanford.edu/class/cs224n/)** — The rigorous academic treatment; watch after your implementation is working, to validate and deepen your mental model.
5. **[Hugging Face — "The Transformer Model Family" (official docs)](https://huggingface.co/docs/transformers/model_summary)** — The clearest official reference for which real-world models are encoder-only, decoder-only, or encoder-decoder, and why — useful as a living reference you'll return to throughout the rest of this handbook.
6. **[RoFormer paper — Su et al., "RoPE: Enhanced Transformer with Rotary Position Embedding" (2021)](https://arxiv.org/abs/2104.09864)** — Optional deeper dive once the core module is solid; read this if you want to understand exactly how modern long-context models handle position, beyond the sinusoidal scheme covered here.

---

## 10. Exercises

1. **Shape-trace by hand.** For a sequence of `n=6` tokens, model dimension `d=64`, 4 attention heads (so `d_k = d_v = 16` per head), write out the exact shape of every intermediate tensor (`Q`, `K`, `K^T`, `Q@K^T`, attention weights, `V`, the attention output, the concatenated multi-head output, and the final `W_O` projection).
2. **Manual mini-attention computation.** Using a toy example with 3 tokens and `d_k = 2`, pick simple numeric values for `Q`, `K`, `V` (small integers) and manually compute the full scaled dot-product attention output, including the softmax step, by hand or with a calculator.
3. **The scaling factor, empirically.** Implement scaled dot-product attention twice — once with the `/sqrt(d_k)` scaling and once without — using random `Q`/`K` matrices with a large `d_k` (e.g., 512). Print the softmax output in both cases and observe how much more "peaked" (close to one-hot) the unscaled version becomes. Connect this back to Module 3's vanishing gradient discussion.
4. **Causal mask implementation.** Given an `(n, n)` raw attention score matrix, write the masking code that converts it into a valid causal (decoder-style) attention score matrix, and confirm via `softmax` that masked positions receive exactly zero probability.
5. **Positional encoding sanity check.** Implement the sinusoidal positional encoding formula from the original paper for a sequence of length 20 and dimension 16. Plot or print it, and confirm that positions close to each other produce similar (but not identical) encoding vectors, while distant positions produce more different ones.
6. **Head specialization inspection.** Using a small pretrained transformer (via Hugging Face `transformers`, with `output_attentions=True`), run a real sentence through it and visualize/print the attention weight matrices for at least 2 different heads in the same layer. Write two sentences on any interpretable pattern you notice (e.g., a head that consistently attends to the previous token, or to a specific punctuation mark).

---

## 11. Mini-Project: `toy-transformer`

Following Karpathy's approach as inspiration (but write your own, don't just transcribe his), implement a minimal decoder-only transformer from scratch in PyTorch.

**Requirements:**
- A `MultiHeadSelfAttention` module implementing exactly the mechanism from Theory sections 6.2–6.4, with a causal mask option.
- A `TransformerBlock` module combining attention + feed-forward + residual connections + layer norm, per section 6.6.
- A small full model: token embedding + learned positional embedding + N stacked `TransformerBlock`s + final linear projection to vocabulary size.
- Train it as a character-level language model (reuse the character-level tokenization idea from Module 5's theory, section 6.1, for simplicity — no need for full BPE here) on a small text corpus (a few thousand characters — a short story or a chapter of public domain text works well) to predict the next character.
- After training, generate text by sampling one character at a time from the model's output distribution, feeding each generated character back in as input (autoregressive generation — you'll recognize this is the exact same mechanical loop you'll use for real LLM APIs in Part 3, just with a tiny model of your own).
- Verify qualitatively: the generated text should not be random noise — it should show recognizable structure (correct-looking word shapes, plausible punctuation placement) even if it isn't coherent English, proving your attention mechanism is genuinely learning sequence structure.

---

## 12. Production Project: `contextual-embedding-service`

This is a direct architectural upgrade of `embedding-service` (Part 2, Modules 4–5) — same service, same API contract where possible, but now you *understand and can inspect* exactly what's happening inside it, and you add capability that only makes sense once you understand attention.

**What to build:**

1. **Swap in explicit access to attention internals.** Using Hugging Face `transformers` with `output_attentions=True` on the underlying sentence-embedding model, add a new endpoint:
   ```
   POST /inspect
   { "text": "I sat by the river bank" }
   →
   { "tokens": ["I", "sat", "by", "the", "river", "bank"],
     "attention_summary": { "bank": {"river": 0.61, "the": 0.09, ...} },
     "embedding": [0.03, -0.18, ...] }
   ```
   This endpoint should surface, for a chosen target word, which other tokens it attended to most strongly — a genuinely useful debugging tool for anyone building on top of this service later (a direct, practical instance of the "attention weights are interpretable" property from section 6.3).
2. **Add a polysemy demonstration test suite**: pick 3–4 genuinely ambiguous words (bank, bat, spring, bass) and, for each, two sentences using different senses. Assert, via cosine similarity, that the *contextual* embedding of the word differs meaningfully between the two sentences (proving real context-sensitivity), while a *static* word2vec embedding (reuse `toy-word2vec` from Module 4) for the same word does not change — a directly observable, testable proof that the upgrade from Module 4 to now is real and measurable, not just theoretical.
3. **Add a `/causal-vs-bidirectional` documentation note and a small demo endpoint** contrasting how a decoder-only model (causal-masked) versus your encoder-based embedding model (bidirectional) would each process the same sentence differently — reinforcing section 6.7's distinction with a runnable example, since this distinction becomes directly load-bearing again in Part 3 (you'll exclusively be calling decoder-only chat models via API).
4. **Keep the tokenizer-aware caching and cost-estimation endpoints from Module 5 intact and passing all existing tests** — this module adds capability, it does not regress the previous one.

**Explicitly designed for extension:** `contextual-embedding-service`'s `/inspect` endpoint and polysemy test suite become the debugging toolkit you'll reach for throughout Part 4 (RAG) whenever retrieval quality looks wrong and you need to understand *why* two pieces of text did or didn't end up similar in embedding space. The causal-vs-bidirectional demo directly previews Part 3, where every model you call is decoder-only.

---

## 13. Practice & Interview Questions

1. Explain, from scratch, what Query, Key, and Value represent and why all three are needed — what would be lost if you tried to build attention with just Query and Key (no separate Value)?
2. Why do we divide by `sqrt(d_k)` in scaled dot-product attention? Be precise about the mechanism, not just "for stability."
3. Explain why attention alone (without positional encoding) is permutation-invariant, and why that's a problem for language modeling.
4. What is the purpose of multi-head attention, as opposed to one larger single-head attention computation with the same total dimensionality?
5. Explain the role of residual connections and layer normalization in making deep transformer stacks trainable. Connect this explicitly to the vanishing gradient problem from Module 3.
6. What is the difference between self-attention and cross-attention? Where does cross-attention appear in an encoder-decoder architecture?
7. Why is BERT (encoder-only, bidirectional) a good fit for producing embeddings, while GPT-style models (decoder-only, causal) are a good fit for text generation? Could you use a decoder-only model to produce embeddings? What would you lose?
8. A teammate says "attention lets the model look at the whole input at once, which is why transformers are faster to train than RNNs." Evaluate this claim — is it accurate, and if so, precisely why (hint: think about parallelization across sequence positions during training vs. the sequential nature of RNNs)?
9. Explain what a causal mask does, mechanically (what values change, and where, in the computation), and why it's necessary for training a model to generate text autoregressively.

---

## 14. Common Mistakes

- **Treating attention as "the model looking at important words," without understanding it's a precise, learned weighted average computed via dot products.** This vague framing will fail you the moment an interviewer asks a follow-up question about the actual mechanism.
- **Forgetting the `sqrt(d_k)` scaling when implementing attention from scratch**, leading to a working-but-numerically-unstable implementation that trains poorly on anything beyond toy examples — a classic "it works on my tiny example but not at scale" bug.
- **Assuming positional information is unnecessary "because the model sees the whole sequence anyway."** Seeing the whole sequence is not the same as knowing the *order* of that sequence — these are genuinely separate facts a model must be given.
- **Confusing encoder-only and decoder-only architectures when choosing a model for a task** — e.g., trying to use a chat-style decoder-only model naively as an embedding model without understanding that its causal masking means later tokens never influence earlier tokens' representations, which is often not what you want for a holistic sentence embedding.
- **Believing multi-head attention means "the model looks at multiple sentences at once."** It means multiple *parallel attention computations over the same single sequence*, each potentially specializing in a different kind of relationship — a common and understandable but incorrect first guess.
- **Assuming deeper is strictly better without residual connections and layer norm doing real work to make that depth trainable** — depth alone, without these stabilizing components, reintroduces the vanishing/exploding gradient problems from Module 3.

---

## 15. Debugging Exercise

You've implemented your own scaled dot-product attention for `toy-transformer`, and training loss is stuck — it decreases very slightly for the first few steps, then completely flatlines, no matter how long you train or what learning rate you try (you've tried several, per your Module 3 experience with learning rate tuning).

Here is your attention implementation:

```python
def attention(Q, K, V):
    scores = Q @ K.transpose(-2, -1)  # (n, n)
    weights = softmax(scores, dim=-1)  # <-- bug is near here
    return weights @ V
```

**Your task:** Identify the missing step (directly covered in this module's theory) and explain, mechanistically, *why its absence specifically produces the "learns a tiny bit, then flatlines" symptom* rather than, say, an outright crash or obviously garbage output from step one. (Hint: think about what happens to the softmax output's "peakedness" as `d_k` grows, and what that peakedness does to the gradient flowing backward through the softmax during backpropagation — this connects directly back to the vanishing gradient intuition from Module 3.) Fix the implementation, retrain, and confirm loss now decreases smoothly to a much lower value.

---

## 16. Checklist

- [ ] I can derive scaled dot-product attention from the "soft database lookup" analogy, explaining precisely what Q, K, and V represent.
- [ ] I can explain, with the variance argument, exactly why the `sqrt(d_k)` scaling factor exists.
- [ ] I understand and can explain why attention is permutation-invariant without positional encoding, and how positional encoding fixes this.
- [ ] I can explain why multi-head attention exists and what "specialization across heads" means in practice.
- [ ] I understand the role of residual connections and layer normalization in making deep transformers trainable, and can connect this to Module 3's vanishing gradient problem.
- [ ] I can clearly distinguish encoder-only, decoder-only, and encoder-decoder architectures, and correctly classify BERT, GPT/Claude/LLaMA, and T5.
- [ ] I've implemented `toy-transformer` from scratch, trained it as a character-level language model, and generated recognizably structured (if not coherent) text via autoregressive sampling.
- [ ] I've upgraded `embedding-service` into `contextual-embedding-service`, with a working `/inspect` endpoint and a passing polysemy test suite proving real context-sensitivity.
- [ ] I can answer all 9 interview questions without notes.

---

## 17. Summary

Attention is a soft, differentiable, fully-learned database lookup: Queries ask what a token needs, Keys advertise what every token (including itself) offers, a scaled softmax over their dot products decides how much to trust each candidate, and Values are blended accordingly to produce a genuinely context-aware output vector — finally delivering on Module 4's promise of contextual, not static, embeddings. Multi-head attention runs several of these lookups in parallel, letting different heads specialize in different kinds of relationships. Because attention alone has no sense of sequence order, positional encoding is added separately. A full transformer block wraps multi-head attention and a per-token feed-forward network in residual connections and layer normalization — a direct, deliberate architectural answer to the vanishing gradient problem you first encountered in Module 3, and precisely what makes it possible to stack dozens to hundreds of these blocks into today's large language models. Whether a model uses bidirectional (encoder-only) or causal (decoder-only) attention determines whether it's suited to holistic understanding tasks like embeddings, or autoregressive generation tasks like chat — a distinction that will matter constantly for the rest of this handbook. You implemented every piece of this from scratch in `toy-transformer`, and used your newfound understanding to give `embedding-service` a genuine, inspectable, testable contextual-embedding upgrade.

---

## 18. Next Steps

**Next module: Part 2, Module 7 — "Training vs. Inference, and Fine-tuning."** You now understand the full forward-pass architecture of a transformer. Module 7 steps back to the training *process* at the scale real LLMs are actually trained (pretraining on internet-scale corpora, the compute/data/parameter scaling relationships, and why it works at all), then covers how a pretrained base model gets turned into a genuinely useful assistant via fine-tuning (supervised fine-tuning and a first look at RLHF/preference-based alignment) — the exact process that turns "a model that can autocomplete the internet" into "a model that can follow your instructions," which is the model you'll actually be calling via API throughout Part 3.

---

Reply "continue" for Module N, or flag anything to go deeper on.
