# PART 2 — AI Foundations
## Module 2: Neural Networks from First Principles

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Explain exactly what a neuron, a layer, and a full network compute,
  mechanically, with no hand-waving — building directly on Module 1's
  matrix multiplication intuition.
- Explain why nonlinear activation functions are mathematically necessary
  for a network to represent anything beyond a single linear
  transformation — a precise, not just intuitive, explanation.
- Build a tiny neural network from scratch in raw numpy (forward pass
  only — training/backpropagation is the next module), so the mechanics
  are never a black box.
- Read and understand a PyTorch model definition, connecting every line
  back to the raw-numpy mechanics you built yourself.
- Explain, at a conceptual level, why depth (many layers) and width (many
  neurons per layer) each contribute to a network's representational
  capacity.

### 2. Prerequisites
Module 1 (linear algebra/probability intuition) — this module is a direct
continuation.

### 3. Estimated Study Time
10–14 hours over 5–7 days.

### 4. Difficulty
⭐⭐⭐☆☆ (Medium — the mechanics are genuinely simple once Module 1's
intuition is solid; building it from scratch once, in raw numpy, is what
cements real understanding versus surface familiarity.)

### 5. Why This Matters
Every model you'll ever call via an API (Anthropic, OpenAI) or run
locally (Part 6) is, at its computational core, built from the exact same
few operations this module teaches — matrix multiplications and
nonlinear activation functions, stacked and scaled up enormously. You will
likely never write a transformer from scratch in practice, but
understanding *this* module deeply enough to build a tiny network from
raw numpy is what turns every subsequent, more complex AI concept
(attention, training, fine-tuning) into "a specific, learnable extension
of something I already understand" rather than an opaque black box.

---

### 6. Theory

**What is a neuron? (mechanically, precisely, no metaphor needed)**

A single "neuron" computes exactly this:
```python
output = activation_function(dot(weights, inputs) + bias)
```
That's the entire mechanical definition. `weights` is a vector (one
number per input), `dot(weights, inputs)` is exactly the dot product from
Module 1, `bias` is a single learnable offset number, and
`activation_function` is a simple nonlinear function applied to the
result. **A "neuron" is not a biological metaphor you need to reason
about — it's precisely: one weighted sum plus a nonlinearity.**

**A layer is many neurons computed in parallel — which is exactly a
matrix multiplication (Module 1's intuition, directly applied):**
```python
import numpy as np

def layer_forward(inputs, W, b, activation):
    # inputs: shape (n_input_features,)
    # W: shape (n_neurons, n_input_features) — one row of weights per neuron
    # b: shape (n_neurons,) — one bias per neuron
    z = W @ inputs + b        # matrix-vector multiply (Module 1) + bias
    return activation(z)       # apply nonlinearity elementwise

def relu(z):
    return np.maximum(0, z)    # the most common activation function: zero
                                 # out negative values, pass positive values
                                 # through unchanged
```
**A full network is just multiple layers chained together**, the output
of one layer becoming the input to the next:
```python
def network_forward(x, W1, b1, W2, b2):
    h1 = layer_forward(x, W1, b1, relu)         # hidden layer
    output = layer_forward(h1, W2, b2, lambda z: z)  # output layer (often
                                                        # no activation, or
                                                        # a different one,
                                                        # depending on the task)
    return output
```
**This is, mechanically, the entire "neural network" concept.** Everything
else — how many layers, how wide each layer is, what specific activation
functions, how the weights get learned (next module) — is elaboration on
top of this exact skeleton.

**Why is a nonlinear activation function mathematically necessary? (the
precise reason, not just "because it works better"):**

If every layer were purely linear (`z = Wx + b`, no nonlinearity), then
stacking layers would be mathematically equivalent to just **one** linear
transformation, no matter how many layers you stack — because the
composition of linear functions is itself just another linear function
(`W2(W1x + b1) + b2` algebraically reduces to `W'x + b'` for some single
combined `W'` and `b'`). **A network of purely linear layers, no matter
how deep, can only ever represent a linear function** — which is a
severely limited class of functions (real-world relationships like "is
this email spam" or "what's the next word" are not linear functions of
their inputs). The nonlinear activation function, applied between layers,
is precisely what prevents this collapse and lets depth actually add
representational power.

**Common activation functions, briefly, and why choice matters:**
- **ReLU** (`max(0, z)`) — simple, fast to compute, the modern default for
  hidden layers in most architectures, including transformers (with
  variants like GELU used in many actual LLM implementations).
- **Sigmoid** (`1 / (1 + e^-z)`) — squashes output to (0, 1), historically
  common but rarely used in hidden layers of modern deep networks (it
  suffers from a specific training problem — "vanishing gradients" —
  covered precisely in the next module on backpropagation).
- **Softmax** — not applied per-neuron but across a whole output layer at
  once, converting a vector of raw numbers into a valid **probability
  distribution** (all values between 0 and 1, summing to 1) — this is
  **exactly** what sits at the very end of a language model's forward
  pass, turning raw output scores into "probability of each possible next
  token" (Module 1's probability content, now mechanically connected).

**Why does depth/width matter? (conceptual, precise intuition)**
Each layer learns to represent its input in terms of increasingly
abstract, useful features — an early layer might respond to simple
patterns, later layers combine those into more complex, task-relevant
representations. Width (more neurons per layer) increases how many
distinct patterns a single layer can represent simultaneously; depth (more
layers) increases how many times those patterns get recombined into
increasingly abstract representations. Modern LLMs are both extremely
deep and extremely wide — this module's tiny 2-layer numpy network is a
toy illustration of the exact same mechanical principle operating at a
vastly larger scale.

---

### 7. Mental Models

**Model 1 — "A neuron is one weighted sum plus a nonlinearity. A layer is
many neurons computed as one matrix multiplication. A network is many
layers chained together. There is no additional mystery beyond this."**

**Model 2 — "Without a nonlinearity, any number of stacked layers
collapses mathematically into just one linear transformation — the
nonlinearity is what makes depth actually matter, not just an
implementation detail."**

**Model 3 — "Softmax is the mechanical bridge between 'raw network
output numbers' and 'a genuine probability distribution' — this is
literally the last step in producing next-token probabilities in a
language model."**

---

### 8. Visual Explanation (described)

**Diagram: "A tiny 2-layer network's forward pass"**
```
Input x (3 numbers)
    |
    v  W1 @ x + b1  (matrix multiply — Module 1)
Hidden layer pre-activation z1 (4 numbers)
    |
    v  ReLU(z1)  (nonlinearity — zeroes out negatives)
Hidden layer output h1 (4 numbers)
    |
    v  W2 @ h1 + b2
Output pre-activation z2 (2 numbers)
    |
    v  softmax(z2)
Final output: a valid probability distribution over 2 classes,
summing to 1.0
```

**Diagram: "Why linear layers alone collapse"**
```
Layer 1: z1 = W1x + b1          (linear)
Layer 2 (no nonlinearity): z2 = W2(W1x + b1) + b2
                              = (W2W1)x + (W2b1 + b2)
                              = W'x + b'   <- just ONE linear transform,
                                              no matter how many layers
                                              you stack, without a
                                              nonlinearity in between
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **3Blue1Brown's "Neural Networks" series (YouTube, 4 videos)** —
   the best available visual explanation of exactly this module's content,
   building the intuition with genuinely beautiful animations; watch this
   in full before or alongside the reading here.
2. **Michael Nielsen's "Neural Networks and Deep Learning" (free online
   book, neuralnetworksanddeeplearning.com), Chapter 1** — a precise,
   from-scratch mathematical treatment that pairs excellently with
   3Blue1Brown's visual intuition.
3. **Andrej Karpathy's "Neural Networks: Zero to Hero" video series
   (YouTube)**, specifically the earliest videos — Karpathy builds neural
   networks from raw Python/numpy in real time, which is precisely the
   skill this module's exercises ask you to practice.
4. **PyTorch official "60 Minute Blitz" tutorial** — once you've built
   things by hand in numpy, this shows the equivalent PyTorch code, so you
   can map your from-scratch understanding directly onto the framework
   you'll actually use going forward.

**Official documentation:** pytorch.org/tutorials (the "60 Minute Blitz"
specifically).

**GitHub repos:** `karpathy/micrograd` — a genuinely tiny (under 100
lines), complete, from-scratch neural network + autodiff implementation in
pure Python — reading this entire repo is one of the highest-value single
activities in this whole module.

---

### 17. Exercises

1. Implement the tiny 2-layer network's forward pass (as shown in the
   theory section) from scratch in raw numpy, with hardcoded, small
   weight matrices you choose yourself, and print the intermediate values
   at every step.
2. Empirically demonstrate the "linear layers collapse" claim: build a
   2-layer network with NO nonlinearity between layers, compute its
   output for several inputs, then compute the single equivalent combined
   linear transformation (`W' = W2 @ W1`, `b' = W2 @ b1 + b2`) and confirm
   the outputs match exactly.
3. Implement `softmax` from scratch, apply it to a small raw output
   vector, and confirm the result sums to 1.0 and that a larger raw value
   correctly maps to a larger probability.
4. Read `karpathy/micrograd`'s full source code (it's short) and write a
   short explanation, in your own words, of what each core file/class
   does.

### 18. Mini-Project
**Build:** `neural-net-from-scratch` — a from-scratch numpy implementation
of a small feedforward network (2-3 layers, configurable widths) with a
forward pass only (no training yet), including ReLU and softmax
activations, structured as a clean, tested Python module (reusing Part 0
Module 1's project conventions) rather than a throwaway script.

### 19. Production Project
**Build:** `activation-function-explorer` — a small, genuinely useful
educational/reference tool (and a nice portfolio artifact demonstrating
real conceptual understanding, not just library usage):
- Implements ReLU, sigmoid, tanh, and softmax from scratch in numpy
- Includes a visualization (matplotlib, or reuse the Visualizer-style
  charting skills from earlier if building this as a shareable artifact)
  plotting each activation function's shape and its derivative's shape
  (previewing why some activations cause training problems, ahead of the
  next module on backpropagation)
- Full test suite verifying each implementation against known
  mathematical properties (e.g., sigmoid's output is always strictly
  between 0 and 1; softmax's output always sums to 1)
- A README explaining, in your own words, when you'd choose each
  activation function and why — written as a genuine reference document
  you'd actually want to consult later, not just an exercise writeup

This project builds real fluency with the exact mathematical objects
(activation functions and their derivatives) that the next module's
backpropagation content directly manipulates.

---

### 20–21. Practice & Interview Questions

1. Explain, mechanically and precisely, what a single neuron computes.
2. Why is a nonlinear activation function mathematically necessary for a
   multi-layer network to represent more than a linear function? Give the
   precise algebraic reason, not just "it works better empirically."
3. What does softmax compute, and why is it exactly the right function to
   sit at the end of a language model's forward pass?
4. Explain the difference between depth (more layers) and width (more
   neurons per layer), and what each conceptually contributes to a
   network's representational capacity.
5. Walk through, step by step, the forward pass of a small 2-layer network
   given specific example weights and an input vector.

---

### 22. Common Mistakes
- Treating "neural network" as a mysterious black box instead of
  recognizing it as repeated matrix multiplication plus simple
  nonlinearities — this mistake compounds into confusion at every later
  Part 2 module if not corrected here.
- Forgetting why nonlinearities matter and being unable to explain the
  "linear layers collapse into one linear transform" argument precisely
  when asked.
- Confusing softmax (used across a whole output layer, producing a
  probability distribution) with per-neuron activation functions like
  ReLU (applied independently to each value).
- Skipping the from-scratch numpy implementation in favor of jumping
  straight to PyTorch, missing the mechanical understanding that makes
  PyTorch code legible rather than magical.

### 23. Debugging Exercise
Given a from-scratch numpy network implementation whose output values
don't sum to 1 despite an intended softmax output layer, diagnose (by
inspecting the code) that the softmax was implemented incorrectly (e.g.,
missing the normalization step, dividing by the wrong axis's sum in a
batched implementation) and fix it, verifying against the mathematical
property (`sum == 1.0`) directly in a test.

---

### 24. Checklist
- [ ] I can explain a neuron, a layer, and a full network mechanically,
      with no metaphor needed
- [ ] I can prove (algebraically, in my own words) why linear layers
      without nonlinearities collapse into one linear transform
- [ ] I've built a small feedforward network from scratch in raw numpy,
      forward pass only
- [ ] I understand softmax's role precisely, including its direct
      connection to next-token probability
- [ ] I've read and understood `karpathy/micrograd`'s full source code

### 25. Summary
A neural network, mechanically, is nothing more than repeated matrix
multiplication (Module 1's core operation) interspersed with simple
nonlinear activation functions — the nonlinearity is what prevents
depth from mathematically collapsing into a single linear transform.
Softmax converts raw output numbers into a genuine probability
distribution, directly connecting back to Module 1's probability content
and directly foreshadowing how a language model produces next-token
probabilities. `neural-net-from-scratch` and `activation-function-
explorer` are genuine, tested, from-scratch artifacts proving this
understanding isn't just passive reading.

### 26. Next Steps
Module 3: **Backpropagation & Training Neural Networks** — how a network's
weights actually get learned from data, building directly on this
module's forward-pass mechanics to explain gradient descent and the
backpropagation algorithm precisely, from scratch.

---

**Reply "continue" for Module 3, or flag anything to go deeper on.**
