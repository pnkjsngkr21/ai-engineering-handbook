# PART 2 — AI Foundations
## Module 1: Linear Algebra & Probability Intuition for AI

---

### A Note Before We Begin

Part 2 assumes **zero prior ML knowledge**, as promised. If you've picked
up fragments of ML concepts before, that's fine — we're building from the
ground up regardless, so nothing here assumes you already know what an
embedding or a neural network is. Where useful, we'll anchor new ideas in
things you already know deeply from backend engineering (arrays, hash
maps, distributed systems), because the best way to learn genuinely new
concepts is to connect them to a solid existing mental scaffold.

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Explain what a vector, matrix, and matrix multiplication actually
  represent — geometrically and computationally — well enough to reason
  about embeddings and neural network layers later without hand-waving.
- Explain probability distributions, conditional probability, and
  Bayes' theorem intuitively, with the specific framing that makes
  "why do LLMs predict the next token this way" click.
- Understand dot products and cosine similarity precisely — the exact
  mathematical operation underlying every embedding-similarity search
  you'll build in Part 4.
- Read simple numpy code implementing these operations and connect it
  directly to the math, so later PyTorch/model code isn't a black box.
- Recognize which parts of this math you'll actually touch directly
  (rarely) versus which mental models you'll use constantly (often) —
  calibrating exactly how much of this to actually memorize.

### 2. Prerequisites
Part 0 Module 1 (Python) for the numpy code; no prior math beyond high
school algebra assumed.

### 3. Estimated Study Time
10–12 hours over 5–6 days.

### 4. Difficulty
⭐⭐⭐☆☆ (Medium — not because the math is hard, but because building
correct geometric intuition, rather than just memorizing formulas, takes
deliberate practice.)

### 5. Why This Matters
Every single AI concept from here forward — embeddings (Part 2/4),
attention (Part 2), training (Part 2), evaluation confidence (Part 2) —
is built directly on top of the handful of ideas in this module: vectors
as points/directions in space, matrix multiplication as a batched linear
transformation, and probability as the language models literally speak
in (an LLM's core operation is outputting a probability distribution over
possible next tokens). Getting genuine intuition here, not just formula
memorization, is what makes everything else in Part 2 feel like natural
extensions rather than arbitrary new facts.

---

### 6. Theory

**What is a vector, really? (beyond "an array of numbers")**

A vector is a list of numbers, **and** a point in space, **and** an
arrow/direction from the origin to that point — all three views are
simultaneously true and useful at different moments:
```python
v = [3, 4]   # a list of 2 numbers... which is ALSO the point (3, 4)...
             # which is ALSO an arrow from (0,0) to (3,4), with length
             # sqrt(3² + 4²) = 5 (Pythagorean theorem — this length is
             # called the vector's "magnitude" or "norm")
```
For AI specifically: an **embedding** (Part 2, later module) is just a
vector — a list of, say, 768 numbers — representing a word, sentence, or
document as a **point in a 768-dimensional space**, where "similar
meaning" is represented as "nearby points." You cannot visualize 768
dimensions directly, but every intuition you build in 2D/3D (distance,
direction, angle between vectors) transfers mathematically to
768 dimensions — this is the single biggest leap of faith worth making
peace with early: **high-dimensional geometry works the same way as the
2D/3D geometry you can actually picture, even though you can't visualize
it directly.**

**Dot product — the single most important operation in this entire
module, because it's the literal mechanism behind embedding similarity:**
```python
a = [1, 2, 3]
b = [4, 5, 6]
dot_product = a[0]*b[0] + a[1]*b[1] + a[2]*b[2]   # = 4 + 10 + 18 = 32
```
Geometrically, the dot product of two vectors relates directly to the
**angle between them**: it's large and positive when vectors point in
similar directions, near zero when they're roughly perpendicular
(unrelated), and negative when they point in opposite directions
(opposite meaning). This is *exactly* why embedding similarity search
(Part 4) works: two pieces of text with similar meaning produce embedding
vectors that point in similar directions, so their dot product (or the
closely related **cosine similarity**, which normalizes for vector length
so only direction matters) is high.

```python
import numpy as np
def cosine_similarity(a, b):
    return np.dot(a, b) / (np.linalg.norm(a) * np.linalg.norm(b))
```
**This exact function is the mathematical core of every vector database
search you'll build in Part 4** — worth genuinely internalizing now rather
than treating as a black-box library call later.

**Matrices and matrix multiplication — what a neural network layer
actually does:**

A matrix is a grid of numbers — and, crucially, **matrix multiplication
represents applying a linear transformation to every vector it's
multiplied with** (rotating, scaling, or projecting vectors into a
different space). A neural network layer (Part 2, next module) is,
mechanically, nothing more than: take an input vector, multiply it by a
weight matrix, add a bias vector, apply a simple nonlinear function —
that's the *entire* mechanical operation, repeated across many layers.
Understanding matrix multiplication as "transform every input vector into
a new space" is what makes "a neural network learns a representation"
stop being a vague phrase and start being a concrete, visualizable
operation.

```python
W = np.array([[0.5, 0.2], [0.1, 0.9]])   # a 2x2 "weight matrix"
x = np.array([3, 4])                       # an input vector
output = W @ x   # matrix-vector multiplication: transforms x into a new vector
```

**Probability — the language a model actually "speaks":**

An LLM's core mechanical output, at every single step, is a **probability
distribution over its entire vocabulary** — for a given context, it
computes "how likely is each possible next token," and then a
**sampling strategy** (Part 2/3) picks one. This reframes "how does an LLM
generate text" from something mysterious into something mechanically
precise: **repeated sampling from a learned probability distribution,
one token at a time**, where the distribution itself is what training
(Part 2, later module) shapes.

**Conditional probability, precisely (the concept underlying "next-token
prediction"):**
`P(next_token | context)` reads as "the probability of the next token,
**given** this specific context" — the vertical bar means "given," i.e.,
conditioned on everything that came before. This is *exactly* the
quantity a language model is trained to estimate, over and over, for
every position in a sequence.

**Bayes' theorem — the intuition, not just the formula:**
```
P(A | B) = P(B | A) * P(A) / P(B)
```
The genuinely useful intuition (beyond memorizing the formula): Bayes'
theorem is how you **update a belief given new evidence** — you start with
a prior belief (`P(A)`), observe evidence (`B`), and compute an updated,
"posterior" belief (`P(A | B)`). This exact framing reappears directly in
Part 2's evaluation content (how confident should you be that a model's
output is correct, given some evidence) and in Part 5's agent-reasoning
content (updating beliefs about the world given tool-call results).

---

### 7. Mental Models

**Model 1 — "An embedding is a point in high-dimensional space; 'similar
meaning' literally means 'nearby points' — and the math for measuring
'nearby' (dot product / cosine similarity) is exactly the 2D geometry you
already understand, just in more dimensions than you can picture."**

**Model 2 — "A neural network layer is 'transform this vector into a new
space via matrix multiplication, then apply a simple nonlinearity' —
repeated many times. Nothing mystical, just repeated, learned geometric
transformation."**

**Model 3 — "A language model's entire job, mechanically, is producing a
probability distribution over 'what comes next,' one token at a time —
everything else (creativity, reasoning-seeming behavior) emerges from
that one repeated mechanical operation at sufficient scale and with the
right training."**

---

### 8. Visual Explanation (described)

**Diagram: "Cosine similarity, geometrically, in 2D (generalizes to
768D)"**
```
        ^ (dimension 2)
        |      * "king" vector
        |    /
        |   /  * "queen" vector (similar direction = similar meaning,
        |  /  /    high cosine similarity)
        | /  /
        |/  /
--------+------------> (dimension 1)
        |
        |         * "bicycle" vector (very different direction =
        |            unrelated meaning, low/near-zero cosine similarity)
```

**Diagram: "Matrix multiplication as transformation"**
```
Input vector x (in "input space")
        |
        v  multiply by weight matrix W
Output vector Wx (in a NEW, transformed space — rotated/scaled/projected)
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **3Blue1Brown's "Essence of Linear Algebra" video series (YouTube)** —
   unanimously regarded as the best visual/geometric intuition-building
   resource for linear algebra that exists; watch this before reading any
   formulas elsewhere — it will make everything else in this module click
   faster.
2. **3Blue1Brown's "But what is a neural network?" video** — previews
   Part 2's next module using exactly the geometric intuition this module
   builds; watch it now as a bridge, even though the full neural network
   module comes next.
3. **"Mathematics for Machine Learning" by Deisenroth, Faisal, Ong (free
   PDF, mml-book.github.io)** — Chapters 2 (linear algebra) and 6
   (probability) specifically, for the more rigorous complement to
   3Blue1Brown's visual intuition.
4. **Jay Alammar's blog, "The Illustrated Word2Vec"** — a superb visual
   bridge from "vectors and dot products" (this module) to "word
   embeddings" (this handbook's next relevant module), worth reading now
   even before the embeddings module for the geometric framing.

**Official documentation:** numpy's documentation for `np.dot`,
`np.linalg.norm`, and matrix multiplication (`@` operator / `np.matmul`).

**GitHub repos:** not especially relevant for pure math content — this
module's value is primarily video/visual + hands-on numpy practice.

---

### 17. Exercises

1. By hand (then verify in numpy), compute the dot product and cosine
   similarity of three pairs of small 2D/3D vectors you construct
   yourself: one pair pointing in nearly the same direction, one pair
   roughly perpendicular, one pair pointing in opposite directions —
   confirm the similarity values match your geometric intuition (high,
   near-zero, negative).
2. Implement cosine similarity from scratch in numpy (without using any
   library's built-in cosine similarity function), and verify it against
   `scipy.spatial.distance.cosine` (noting that scipy's function returns
   `1 - cosine_similarity`, a common gotcha worth confirming explicitly).
3. Implement a simple 2-layer "neural network forward pass" (just the
   matrix multiplications and a nonlinearity — no training yet, that's
   the next module) on a toy 2D input, and print the intermediate
   transformed vectors at each layer to see the transformation happening
   concretely.
4. Given a simple word-frequency-based toy example, compute
   `P(word | previous_word)` for a tiny corpus by hand, connecting
   conditional probability directly to "next-token prediction" in the
   simplest possible concrete case.

### 18. Mini-Project
**Build:** `vector-playground` — a small Python script/notebook that: (a)
generates several small, hand-constructed 2D/3D vectors representing toy
"concepts" (you choose the numbers to deliberately represent, e.g.,
"king," "queen," "bicycle"), (b) computes and prints pairwise cosine
similarities between all of them, (c) uses `matplotlib` to actually plot
the 2D vectors and visually confirm that geometrically similar vectors
(small angle between them) correspond to high computed cosine similarity.

### 19. Production Project
**Build:** `similarity-search-toy` — a minimal, from-scratch (no vector DB
yet — that's Part 4) semantic similarity search tool:
- A small hardcoded set (20-30) of short text snippets, each manually
  assigned a toy vector representation (you'll replace this with real
  embeddings from an actual embedding model in Part 4 — for now, focus
  entirely on the math, not real NLP)
- A `search(query_vector, top_k)` function that computes cosine
  similarity against every stored vector and returns the top-k most
  similar snippets, sorted correctly
- Full test suite verifying the similarity computation and ranking logic
  against hand-computed expected values
- A README explicitly explaining that this project is deliberately
  "real math, toy data" — and that Part 4 will swap in a real embedding
  model and a real vector database, while this exact `search()` logic's
  underlying math stays identical

This project is a direct, hands-on rehearsal for the actual RAG retrieval
mechanism you'll build for real in Part 4 — you're learning the
mathematical core now, with toy data, so Part 4 is "swap in real
embeddings" rather than learning new math under pressure.

---

### 20–21. Practice & Interview Questions

1. Explain what a dot product represents geometrically, and why cosine
   similarity is the natural measure of "how similar are these two
   embeddings."
2. What does matrix multiplication represent geometrically, and how does
   that relate to what a neural network layer does?
3. Explain conditional probability `P(A | B)` in your own words, and
   connect it directly to what a language model computes at each
   generation step.
4. Explain Bayes' theorem's intuition (updating a belief given new
   evidence) with a concrete, non-mathematical example.
5. Why does high-dimensional geometry (768 dimensions, say) still behave
   according to the same mathematical rules as the 2D/3D geometry you can
   visualize, even though you can't picture it directly?

---

### 22. Common Mistakes
- Confusing cosine similarity with a "distance" metric directly — cosine
  *similarity* is high for similar vectors (opposite of a distance, where
  small values mean similar) — watch for this exact sign/direction
  confusion, especially since some libraries (e.g., scipy) return cosine
  *distance* (`1 - similarity`) by a different function name.
- Treating matrix multiplication as pure memorized mechanics without the
  "transforms every vector into a new space" geometric intuition, making
  later neural network content feel arbitrary instead of connected.
- Assuming high-dimensional geometric intuition must be fundamentally
  different from 2D/3D — it isn't; the math is genuinely identical, only
  visualization becomes impossible past 3 dimensions.
- Skipping the visual/geometric resources (3Blue1Brown) in favor of pure
  formula memorization, which tends to produce brittle understanding that
  doesn't transfer well to new situations later in Part 2.

### 23. Debugging Exercise
Given a small semantic-search function returning clearly wrong results
(unrelated items ranked as "most similar"), diagnose that the code is
using raw dot product instead of cosine similarity on vectors of very
different magnitudes (an unnormalized long vector can have a large dot
product with anything, regardless of direction/meaning) — fix by properly
normalizing or switching to true cosine similarity.

---

### 24. Checklist
- [ ] I can explain a dot product and cosine similarity geometrically,
      not just recite the formula
- [ ] I can explain matrix multiplication as "transforming a vector into a
      new space" and connect that directly to what a neural network layer
      does
- [ ] I can explain conditional probability and connect it directly to
      next-token prediction
- [ ] I'm comfortable with the idea that high-dimensional geometry follows
      the same rules as 2D/3D, even without being able to visualize it
- [ ] I've completed `similarity-search-toy` with correct, tested
      similarity ranking logic

### 25. Summary
This module built the mathematical intuition — vectors as points/
directions, dot products/cosine similarity as the measure of "similar
direction = similar meaning," matrix multiplication as learned geometric
transformation, and probability as the literal language a model computes
in — that every subsequent Part 2 module (embeddings, attention, training)
builds directly on top of. `similarity-search-toy`'s cosine-similarity
search logic is the exact mathematical core you'll extend with real
embeddings in Part 4.

### 26. Next Steps
Module 2: **Neural Networks from First Principles** — building directly
on this module's matrix multiplication intuition to explain exactly what
a neural network is, how it represents information, and why stacking
simple transformations produces surprisingly powerful behavior.

---

**Reply "continue" for Module 2, or flag anything to go deeper on.**
