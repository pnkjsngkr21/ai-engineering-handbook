# Part 2, Module 10: Vision, Speech & Multimodal Models (Part 2 Capstone)

---

## 1. Learning Objectives

By the end of this module, you will be able to:

- Explain how a Vision Transformer (ViT) converts an image into a sequence of "tokens" a transformer can process, using patch embeddings — the direct visual analogue of Module 5's text tokenization.
- Explain how speech/audio is converted into a token-like sequence for transformer processing (spectrograms, Whisper-style architectures).
- Explain the two dominant architectural strategies for combining vision (or audio) and text into a single multimodal model, and why each exists.
- Correctly reason about what multimodal models can and cannot do well, grounded in how their training data and architecture actually work, rather than treating "multimodal" as an undifferentiated capability upgrade.
- Connect every major mechanism from Modules 4 through 9 into one unified mental model of "how a modern foundation model works, end to end" — the explicit goal of this capstone.
- Build a small multimodal RAG-style project that retrieves and reasons over both text and images, directly previewing Part 4.

---

## 2. Prerequisites

- Part 2, Module 6 (Attention & the Transformer Architecture) — this module assumes you can comfortably map the "sequence of vectors → attention → contextual output" pipeline onto a new kind of input.
- Part 2, Module 4 (Embeddings) and Module 5 (Tokenization) — the core idea of "convert raw input into a sequence of dense vectors" is the same idea, applied to a new modality.
- All of Part 2, Modules 1–9 — this module is explicitly cumulative, and the closing review section assumes you've genuinely completed the earlier modules, not skimmed them.

---

## 3. Estimated Study Time

**7–9 hours** (2–3 hours theory, 3–4 hours implementation, 1–2 hours on the cumulative review and exercises).

---

## 4. Difficulty

★★★☆☆ (3/5) — The core insight of this module (patches/spectrograms are just another kind of tokenization) is a satisfying, comparatively easy generalization once Modules 4-6 are solid. Most of the "difficulty" here is really the weight of the cumulative review, not new mechanism.

---

## 5. Why This Matters

Every model you've built understanding around so far — word2vec, the transformer, reasoning models — has assumed the input is text. But the production systems you'll build from Part 3 onward will routinely need to handle images (a user uploads a screenshot of an error, a product photo, a scanned document), and increasingly, audio (a voice agent, a meeting transcription tool). The genuinely good news, and the reason this module can be comparatively short relative to Module 6: **you do not need a new theory of "how a model understands an image."** The entire trick — and it really is one core trick, applied consistently — is: **figure out a principled way to chop the non-text input into a sequence of tokens, embed those tokens, and feed the resulting sequence into the exact same transformer architecture from Module 6.** Once you see this, "multimodal AI" stops being a separate, mysterious field and becomes a specific, well-understood extension of everything you already know.

This module closes out Part 2 for a deliberate reason: it's both the natural extension of "tokenization + embeddings + attention" to new modalities, and the natural moment to step back and consolidate. You've built, from scratch, a working understanding of the entire modern foundation-model pipeline: how raw text becomes tokens (Module 5), how tokens become meaningful vectors (Module 4), how those vectors become contextual through attention (Module 6), how the resulting architecture gets trained at scale and adapted (Module 7), how to rigorously know if any of this actually worked (Module 8), and how a specific training regime unlocks genuinely new reasoning capability (Module 9). Before Part 3 shifts you from "how models work" to "how to build systems with them," this module's closing review is designed to make sure that whole picture is genuinely unified in your head, not seven separate, loosely-connected topics.

---

## 6. Theory

### 6.1 Vision Transformers: Images as Sequences of Patches

Recall the entire point of Module 5's tokenization: a transformer needs its input as a *sequence of discrete units*, each of which gets mapped to a dense embedding vector (Module 4), before attention (Module 6) can do anything with it. An image, as a raw grid of pixels, isn't naturally sequential at all — it's a 2D (or, with color channels, 3D) array of numbers.

**The Vision Transformer (ViT) solution, precisely:** cut the image into a grid of fixed-size square patches (a common choice: 16x16 pixels per patch). For a 224x224 pixel image, this produces a 14x14 grid of patches — 196 patches total. Each patch (a small block of raw pixel values) is flattened into a single long vector and passed through a learned linear projection (a single matrix multiply — genuinely nothing more exotic than that) to produce a fixed-size embedding vector for that patch, exactly analogous to Module 4's word embedding lookup, except here the "vocabulary" isn't a fixed set of discrete tokens — it's a continuous linear projection applied to raw pixel patches.

**From this point forward, the architecture is, remarkably, almost identical to the text transformer you already built in Module 6:**
- Each patch embedding gets a positional encoding added (section 6.5 of Module 6, applied unchanged — the model needs to know patch #1 is top-left and patch #196 is bottom-right, exactly as it needed to know token order in text).
- A special learnable "class token" (analogous in spirit, though not identical in purpose, to Module 5's end-of-word marker) is often prepended to the sequence, whose final-layer output vector is used as the overall image representation for classification tasks.
- The resulting sequence of patch embeddings is fed through ordinary multi-head self-attention and transformer blocks (Module 6, sections 6.2–6.6), completely unchanged from the text case — attention computed between image patches learns which regions of the image are relevant to which other regions, exactly as text attention learns which words are relevant to which other words.

**The genuinely striking empirical finding (Dosovitskiy et al., 2020, "An Image is Worth 16x16 Words"):** this remarkably simple adaptation — chop into patches, linearly embed, add position, run through an unmodified transformer — matches or exceeds the previous state-of-the-art (convolutional neural networks, which had dominated computer vision for a decade) when trained on sufficiently large image datasets. This is a genuinely important, somewhat humbling lesson about the generality of the attention mechanism: **the same architecture that learned to model language turns out to be a remarkably general sequence-processing engine, largely indifferent to what the "tokens" in the sequence actually represent**, provided you solve the comparatively modest problem of getting your input into token-sequence-plus-position form in the first place.

### 6.2 Speech and Audio: The Same Trick, a Different Front-End

Raw audio is a very high-sample-rate 1D waveform (commonly 16,000-44,100 samples per second) — far too fine-grained to feed directly as "one token per sample" into a transformer (the sequence would be absurdly long, and individual raw samples carry almost no meaningful information in isolation, unlike a word or an image patch). The standard front-end transforms raw audio into a more tractable representation before tokenization:

**Spectrograms**, specifically the **mel spectrogram** (a representation of audio's frequency content over time, using a frequency scale — the mel scale — chosen to roughly match human auditory perception) convert the 1D waveform into a 2D representation (time on one axis, frequency on the other) — which, notice, is now structurally very similar to an image, and can be patch-tokenized in a directly analogous way to section 6.1's ViT approach.

**OpenAI's Whisper**, a widely used and well-documented real-world example, uses roughly this pipeline: raw audio → mel spectrogram → a sequence of embedded "patches" of the spectrogram (processed by an encoder, using the same bidirectional/encoder-style attention discussed in Module 6, section 6.7, since the model has the entire audio clip available at once, similar to how BERT-style encoders process complete text) → a decoder (causal, autoregressive, exactly like Module 6's decoder-only text generation) that generates the output transcription one text token at a time, conditioned on the encoded audio via cross-attention (Module 6, section 6.7's encoder-decoder cross-attention mechanism, here connecting an audio encoder to a text decoder instead of connecting two text sequences).

**The pattern to notice and hold onto:** speech recognition, at the architectural level, is genuinely just encoder-decoder machine translation (Module 6, section 6.7) where the "source language" happens to be a sequence of audio-spectrogram-patch embeddings instead of foreign-language text tokens, and the "target language" is ordinary text. You are not learning a new architecture here — you are recognizing the same encoder-decoder transformer pattern from Module 6, with a different, task-appropriate front-end for converting the input into an embeddable sequence.

### 6.3 Combining Modalities: Two Dominant Strategies

Once you can convert both text and images (or audio) into sequences of embedding vectors, "multimodal" simply means: **feed embeddings from more than one modality into the same model, so attention can relate them to each other.** There are two dominant architectural strategies in production use today, and the distinction matters for understanding real systems' capabilities and costs:

**Strategy 1 — Early fusion / unified sequence (used by most modern multimodal chat models, e.g., GPT-4V-style and Claude's vision capability):** the image is patch-tokenized and embedded (section 6.1), the text is tokenized and embedded (Module 5/4), and the two resulting sequences of embeddings are simply **concatenated into one combined sequence**, fed through a single, shared transformer. Attention then operates freely across the entire combined sequence — a text token's query can attend to image-patch keys, and vice versa, exactly the same multi-head self-attention mechanism from Module 6 with zero architectural modification, just a longer, mixed-modality input sequence. This is architecturally elegant (no new attention mechanism needed at all) and is the dominant approach in current frontier multimodal models, precisely because it lets the single, powerful, already-well-understood transformer/attention mechanism handle cross-modal relationships with no special-casing.

**Strategy 2 — Late fusion / separate encoders with a connecting module (used by some earlier and specialized architectures, e.g., CLIP-style dual-encoder setups and models like LLaVA that connect a separately pretrained vision encoder to a separately pretrained language model):** a vision encoder (e.g., a ViT, per section 6.1, often itself already pretrained separately, sometimes using a contrastive objective similar in spirit to Module 4's embedding-training logic — pulling matching image/caption pairs' embeddings close together and mismatched pairs apart) produces image embeddings; a separate, smaller "connector" module (often just a small learned linear or MLP projection) maps these embeddings into the same vector space/dimensionality the language model's own token embeddings live in; the projected image embeddings are then inserted into the language model's input sequence, much like Strategy 1's concatenation, but critically, the vision encoder and language model can each be developed, pretrained, and even swapped somewhat independently. This is a common, more modular, often cheaper-to-train approach (you can reuse a strong, already-pretrained vision encoder and a strong, already-pretrained language model, training only the comparatively small connector module to align them) — directly analogous to a familiar backend pattern: composing two independently-developed, well-tested services behind a lightweight adapter/translation layer, rather than building one monolithic system from scratch.

**The practical, honest takeaway:** regardless of which strategy a specific model uses under the hood, the core capability being unlocked is the same: **letting attention relate information across modalities**, using representations that were built, in both cases, by the same fundamental "convert to an embeddable sequence" logic you've now seen applied to text (Module 5/4), images (section 6.1), and audio (section 6.2).

### 6.4 What Multimodal Models Genuinely Can and Cannot Do Well

This section exists because "multimodal" is frequently oversold or misunderstood as an undifferentiated capability upgrade, and a precise, grounded understanding will make you a much better judge of when to actually reach for a multimodal model in production, versus when a simpler, cheaper, more reliable pipeline is the better engineering choice.

**Genuinely strong, well-evidenced capabilities:** describing the general content and layout of an image, reading clearly-rendered printed text within an image (a real, useful, well-documented OCR-adjacent capability), answering straightforward questions about depicted objects/scenes, and — for audio — transcribing clear speech in well-represented languages with high accuracy.

**Genuine, well-documented limitations worth knowing precisely, not just vaguely:** precise counting of many small or overlapping objects in an image remains unreliable in a way directly analogous to Module 5's "letters in a word" limitation — the model's internal representation, built from patch embeddings rather than a precise pixel-level symbolic count, doesn't naturally support exact enumeration the way a purpose-built counting algorithm would. Precise spatial reasoning (exact pixel coordinates, fine-grained geometric measurements) is often approximate rather than exact, again because patch embeddings compress and blur fine spatial detail by design (recall the entire point of embeddings, per Module 4, is compression — trading exact reconstruction for a compact, semantically useful representation, and that trade-off has real costs for tasks that specifically need exactness). Handwriting recognition, unusual fonts, and low-quality/rotated/heavily degraded images perform meaningfully worse than clean printed text, tracking the training data distribution these models actually saw (an audio/vision-specific instance of the same "the model is only as good as its training distribution" caution you should already be applying everywhere in this handbook).

**The practical engineering lesson:** exactly as Module 8 taught you to rigorously evaluate any specific claim about model quality rather than trusting a general reputation, "this model is multimodal" tells you almost nothing about whether it's reliably good at *your specific* image/audio task — you must evaluate it, on your own representative data, with the same rigor (`eval-framework`) you'd apply to any other model capability claim.

### 6.5 Cumulative Review: Connecting Modules 4 Through 10

This is the moment to step back and see Part 2 as one coherent story, not seven separate topics. Read through this slowly — the goal is genuine synthesis, not a quick skim.

**It starts with a single, recurring meta-pattern, first named explicitly in Module 4 and now confirmed across every subsequent module: a simple, automatically-available training objective, applied at sufficient scale, forces meaningful structure to emerge — without anyone hand-designing that structure directly.**

- **Module 4 (Embeddings):** the objective was "predict a word's context." The emergent structure was a geometry where semantic similarity corresponds to spatial closeness — nobody hand-labeled "cat" and "dog" as similar; it fell out of shared context statistics.
- **Module 5 (Tokenization):** not itself a "the objective forces structure" story, but the necessary preprocessing step that made a bounded, meaning-preserving vocabulary possible in the first place — without it, Module 4's objective couldn't even be well-posed at internet scale, across every language and every possible word.
- **Module 6 (Attention & Transformers):** the objective — now "predict masked or next tokens using a learned, content-based weighting over the whole sequence" — forced the emergence of contextual embeddings (finally resolving Module 4's static-embedding limitation) and, remarkably, forced individual attention heads to specialize in different relational patterns, again with no explicit design.
- **Module 7 (Training vs. Inference, Fine-tuning):** the single objective of next-token prediction, applied to internet-scale, maximally diverse text, was shown to force the emergence of broad world knowledge and reasoning patterns as a side effect — the same meta-pattern, now at the scale of an entire pretraining run rather than a single mechanism.
- **Module 8 (Evaluating Models):** the necessary discipline for actually verifying whether any of these emergent capabilities are real, robust, and relevant to your task — without it, every claim in every other module would be an untested assertion.
- **Module 9 (Reasoning Models):** a specifically engineered instance of the same meta-pattern taken one step further — using RL against a *verifiable* (rather than merely statistical) reward signal to force the emergence of genuine multi-step, self-correcting reasoning behavior, and revealing test-time compute as a second, independent scaling axis alongside Module 7's training-time scaling.
- **Module 10 (this module):** the discovery that the entire mechanism — tokenize into a sequence, embed, apply attention — is not specific to language at all; it is a remarkably general sequence-processing recipe that applies, with only front-end adaptations, to images, audio, and their combination with text.

**The second unifying thread, running underneath all of this: everything is a matrix, and every "learned" thing is a matrix (or a small number of matrices) whose values were found via gradient descent (Module 3) against some well-chosen loss function.** `W_in`/`W_out` in word2vec, `W_Q`/`W_K`/`W_V`/`W_O` in attention, the LoRA `A`/`B` adapter matrices, the linear patch-embedding projection in a ViT, the connector module in a LLaVA-style multimodal model — these are all, mechanically, the exact same kind of object: a matrix of numbers, initialized randomly, nudged by backpropagation toward values that make some chosen prediction task work well. The dazzling variety of modern AI capability sits on top of a genuinely small, recombination-friendly set of underlying mechanisms, applied again and again, at increasing scale and sophistication, to increasingly well-chosen objectives.

**Hold both of these threads together, and "how does a modern foundation model work" stops being an opaque, sprawling topic and becomes a specific, traceable story** — one you built, piece by piece, from a toy word2vec model all the way to a mechanistically honest understanding of how reasoning models and multimodal chat assistants actually work. This is the exact foundation Part 3 (LLM Engineering) now builds on: everything from here forward assumes you understand *why* the model behaves the way it does, not just *how to call its API.*

---

## 7. Mental Models

1. **"If you can turn it into a sequence of embeddable tokens, attention doesn't care what modality it came from."** The single biggest lesson of this module — patches, spectrograms, and word-pieces all feed the same underlying machine.
2. **"Vision and audio front-ends solve one problem: getting non-text input into token-sequence-plus-position form. Everything downstream is Module 6, unmodified."**
3. **"'Multimodal' is not a capability level, it's an input format — evaluate the specific task, always."** The same Module 8 discipline applies with zero exceptions, regardless of how impressive the general reputation of a multimodal model is.
4. **"A simple, scalable objective plus enormous scale is the recurring engine of this entire field."** Word2vec's context prediction, GPT's next-token prediction, and reasoning models' verifiable-reward RL are three instances of exactly the same underlying principle.
5. **"It's matrices, gradient descent, and a well-chosen loss function, recombined at increasing scale — all the way down."** The unifying, demystifying thread beneath everything in Part 2.

---

## 8. Visual Explanation

**Diagram 1 — ViT: An Image Becomes a Token Sequence**

```
Original image (224 x 224 pixels)
   │
   │  cut into a grid of 16x16 patches
   ▼
14 x 14 grid = 196 patches
   │
   │  flatten each patch, apply ONE learned linear projection
   ▼
196 patch embedding vectors  (+ 1 optional "class" token)
   │
   │  add positional encoding (Module 6, section 6.5 — unchanged)
   ▼
Sequence of 197 embedded "tokens"
   │
   ▼
Ordinary multi-head self-attention + transformer blocks
(EXACTLY Module 6's architecture — no modification)
```

**Diagram 2 — Whisper-Style Speech Recognition: Encoder-Decoder, Reapplied**

```
Raw audio waveform
   │
   │  compute mel spectrogram (time x frequency 2D representation)
   ▼
Spectrogram, patch-tokenized (same trick as Diagram 1)
   │
   ▼
ENCODER (bidirectional attention — sees the whole clip at once,
 same as BERT-style encoding, Module 6 section 6.7)
   │
   │  cross-attention (Module 6, section 6.7)
   ▼
DECODER (causal, autoregressive — generates transcription text
 one token at a time, same mechanism as GPT-style generation)
   │
   ▼
"The quick brown fox jumps over the lazy dog"
```

**Diagram 3 — Two Multimodal Fusion Strategies**

```
STRATEGY 1 — Early fusion / unified sequence:

  [image patch emb 1] [image patch emb 2] ... [text tok emb 1] [text tok emb 2] ...
   └──────────────────────────┬──────────────────────────────┘
                    ONE combined sequence, fed through
                    ONE shared transformer — attention
                    freely relates image and text tokens,
                    no architectural change needed.

STRATEGY 2 — Separate encoders + connector:

  Image ──> [Vision Encoder, e.g. ViT] ──> image embeddings
                                                  │
                                          [Connector module]
                                          (small learned projection,
                                           aligns image embedding
                                           space with text embedding
                                           space)
                                                  │
                                                  ▼
  Text ──> [Text tokenizer + embeddings] ──> concatenated with
                                              projected image
                                              embeddings ──> fed
                                              into Language Model
```

**Diagram 4 — Part 2's Unified Story, End to End**

```
  Module 5: Raw text ──> subword tokens (bounded vocab, no OOV)
                              │
  Module 4: tokens ──> dense embedding vectors (meaning as geometry)
                              │
  Module 6: embeddings ──> CONTEXTUAL embeddings, via attention
                            (+ positional encoding, residuals, layer norm)
                              │
  Module 7: architecture ──> TRAINED at scale (pretrain → SFT → RLHF),
                              a base model becomes a helpful assistant
                              │
  Module 8: trained model ──> RIGOROUSLY EVALUATED (benchmarks + your
                               own golden data + statistical honesty)
                              │
  Module 9: model ──> further trained (RL on verifiable rewards) to
                       REASON via test-time compute, for the right tasks
                              │
  Module 10: SAME architecture ──> generalized to images (ViT patches)
                                    and audio (spectrogram patches),
                                    fused with text via attention
                              │
                              ▼
                 A modern, multimodal, reasoning-capable
                 foundation model — every piece traceable
                 back to a mechanism you built yourself.
```

---

## 9. Recommended Resources

1. **[Dosovitskiy et al. — "An Image is Worth 16x16 Words: Transformers for Image Recognition at Scale" (2020)](https://arxiv.org/abs/2010.11929)** — The original Vision Transformer paper; short, and directly the primary source for section 6.1.
2. **[OpenAI — "Robust Speech Recognition via Large-Scale Weak Supervision" (Whisper paper, 2022)](https://arxiv.org/abs/2212.04356)** — The Whisper paper; read this for a precise, well-documented real-world example of the encoder-decoder audio-to-text pipeline in section 6.2.
3. **[Radford et al. — "Learning Transferable Visual Models From Natural Language Supervision" (CLIP paper, 2021)](https://arxiv.org/abs/2103.00020)** — The paper behind the contrastive image-text embedding approach referenced in section 6.3; directly connects back to Module 4's embedding-training logic, now applied across two modalities at once.
4. **[Liu et al. — "Visual Instruction Tuning" (LLaVA paper, 2023)](https://arxiv.org/abs/2304.08485)** — A clear, concrete, and unusually accessible example of the "separate encoders + small connector module" fusion strategy from section 6.3; a good complement to the early-fusion approach used by most frontier chat models.
5. **[Jay Alammar — "The Illustrated Stable Diffusion" and related visual-model explainers](https://jalammar.github.io/)** — While this handbook doesn't cover image *generation* (diffusion models) in depth, Alammar's visual explainer style continues to be an excellent companion for building intuition on the vision side of this module.
6. **[Anthropic / OpenAI — official vision capability documentation](https://docs.claude.com)** — Search current official docs for practical guidance on image input handling, limitations, and best practices for the specific multimodal APIs you'll actually call in Part 3 — this is a fast-evolving product surface, so prefer live documentation over any static description (including this module's) for exact current capabilities.

---

## 10. Exercises

1. **Patch math.** For a 384x384 pixel image and a patch size of 16x16, calculate the total number of patches. If the model's embedding dimension is 768, what is the shape of the full patch-embedding matrix (before any class token or positional encoding is added)?
2. **Spectrogram inspection.** Using any available Python audio library (e.g., `librosa`), load a short audio clip and compute and plot its mel spectrogram. Describe, in your own words, why this 2D representation is more directly tokenizable than the raw 1D waveform.
3. **Fusion strategy identification.** For each of the following (hypothetically described) systems, identify whether it more closely matches early-fusion (unified sequence) or late-fusion (separate encoders + connector), and justify briefly: (a) a system built by connecting an existing, independently pretrained image classifier to an existing, independently pretrained chat model via a small trained adapter, (b) a system trained from the start on a single combined sequence of interleaved image-patch and text tokens.
4. **Known-limitation stress test.** If you have access to a multimodal model, test its counting ability directly: give it an image with a moderately large number of small, similar objects (e.g., a photo of a pile of coins, or a hand-drawn grid of dots) and ask it to count them precisely. Compare its answer to the actual count, and connect any discrepancy back to section 6.4's explanation.
5. **Cumulative review, written.** Without looking back at section 6.5, write your own 200-300 word summary connecting Modules 4 through 9 into one story, in your own words. Then compare your summary against section 6.5 and note anything you missed or would phrase differently.

---

## 11. Mini-Project: `vit-patch-demo`

A small, focused implementation project making the patch-embedding mechanism from section 6.1 concrete, rather than only conceptual.

**Requirements:**
- Load a small image dataset (e.g., a handful of images from a public, freely licensed source, or a standard small toy dataset).
- Implement the patch-extraction and flattening step from scratch (no need to implement a full ViT training loop — the goal is to make the *tokenization* step tangible): split an image into a grid of fixed-size patches, flatten each into a vector, and apply a single learned (or even randomly initialized, for this demo) linear projection to produce patch embeddings.
- Visualize the patch grid overlaid on the original image, and print the resulting embedding matrix's shape, confirming it matches your hand-calculated expectation from Exercise 1.
- As a bonus extension: load a small pretrained ViT (via Hugging Face `transformers`) and use its `output_attentions` option (directly analogous to Module 6's attention inspection) to visualize which image patches a specific attention head attends to most strongly for a chosen input image — a genuinely satisfying, direct visual confirmation that image attention works exactly like the text attention you already understand deeply.

---

## 12. Production Project: `multimodal-rag-preview`

This is the Part 2 capstone production project — a small, focused, but genuinely functional multimodal retrieval system, deliberately designed as a direct, concrete preview of Part 4 (full RAG), extending `contextual-embedding-service` (Part 2, Module 6) with real multimodal capability.

**What to build:**

1. **Extend `contextual-embedding-service`** with a new endpoint that accepts either text or images and returns embeddings in a **shared embedding space** — use a CLIP-style model (via `sentence-transformers`'s multimodal support, or the `open_clip` library) specifically chosen so that a text query and a relevant image end up with high cosine similarity (directly reusing Module 4's core similarity intuition, now across modalities).
2. **Build a small multimodal document store**: index a modest collection (20-40 items) of mixed text snippets and images (e.g., product descriptions and their corresponding product photos, or recipe text and food photos) using the shared embedding space from step 1.
3. **Implement cross-modal retrieval**: given a text query (e.g., "a red running shoe"), retrieve the most relevant *images* by embedding the query text and finding nearest-neighbor image embeddings (reusing the cosine-similarity nearest-neighbor logic from `similarity-search-toy`, Part 2 Module 1, now operating across modalities) — and, symmetrically, given an image query, retrieve the most relevant *text* snippets.
4. **Add a generation step**: pass a retrieved image, along with the original text query, to a multimodal chat model, and have it generate a grounded natural-language answer referencing the actual retrieved image content — a small, complete, working multimodal RAG loop (retrieve relevant content across modalities, then generate a grounded answer), directly previewing the full RAG architecture Part 4 will build out in depth.
5. **Evaluate it properly**: build a small golden dataset of (query, expected-relevant-item) pairs and score retrieval quality using precision@k (directly reusing the IR-metric scorer you built into `eval-framework` in Module 8) — do not skip this step; it's a direct, deliberate exercise of Module 8's discipline applied to a genuinely new task type.
6. **Document known limitations explicitly**: based on section 6.4's theory and your own testing, write a short README section on what this specific system is and isn't reliable at (e.g., "reliably retrieves images matching described visual style/content; not reliable for precise counting or fine-grained spatial queries").

**Explicitly designed for extension:** `multimodal-rag-preview` is the direct architectural seed for Part 4's full RAG system — the same shared-embedding-space retrieval pattern, the same nearest-neighbor search logic, and the same "retrieve then generate, grounded in retrieved content" loop, will be extended in Part 4 with production-grade chunking, hybrid search, reranking, and a real vector database, for text-primary RAG at scale, informed directly by what you've now personally verified works (and doesn't) across modalities here.

---

## 13. Practice & Interview Questions

1. Explain, precisely, how a Vision Transformer converts an image into a sequence a transformer can process. What plays the role of Module 5's "token" in this pipeline?
2. Why is a mel spectrogram, rather than the raw audio waveform, the typical input representation for a speech-to-text transformer?
3. Explain the architectural difference between early-fusion (unified sequence) and late-fusion (separate encoders + connector) multimodal models, and give a genuine trade-off of each approach.
4. Why does a Vision Transformer need positional encoding, given that Module 6 established attention alone is permutation-invariant? What would go wrong for an image specifically if positional encoding were omitted?
5. Name two genuine, well-documented limitations of current multimodal models, and explain, mechanistically (not just "it's not perfect"), why each limitation follows from how these models represent and process visual/audio input.
6. Explain the recurring "simple objective + massive scale forces emergent structure" pattern, and give one example each from Module 4, Module 7, and Module 9 illustrating it.
7. A colleague says "since GPT-4 can see images now, it must have some fundamentally different architecture than a text-only LLM." Evaluate this claim precisely, using this module's material.
8. Why would you still want a rigorous, task-specific evaluation (per Module 8) before trusting a multimodal model's performance on your specific production image/audio task, even if it has a strong general reputation?

---

## 14. Common Mistakes

- **Assuming vision/audio processing requires an entirely different architecture from the text transformer** — the genuinely correct mental model is "different front-end, same attention-based backbone."
- **Forgetting that image patch embeddings still need positional encoding**, since attention's permutation-invariance (Module 6) applies identically to any modality, not just text.
- **Treating "multimodal" as a single undifferentiated capability tier**, rather than evaluating the *specific* modality-task combination you actually need rigorously, per Module 8.
- **Expecting exact counting or precise spatial/geometric reasoning from a patch-embedding-based model**, when this is a well-documented, architecturally-grounded limitation rather than an occasional random error.
- **Confusing a model that merely accepts image input with one that was genuinely well-trained on your specific image domain** (e.g., medical scans, satellite imagery, or specialized diagrams) — general multimodal training data is broad but not unlimited, and out-of-distribution image domains should be evaluated with extra skepticism.
- **Skipping proper retrieval evaluation (precision@k, etc.) on a multimodal RAG-style system** just because it "looks like it's finding relevant stuff" in casual testing — exactly the Module 8 discipline this handbook has been building toward, and it applies with zero exceptions here.

---

## 15. Debugging Exercise

You've built `multimodal-rag-preview` and it appears to work well in casual testing — you type a few text queries, and the retrieved images look plausible. But when you run your proper precision@k evaluation (per this module's production project, step 5), the measured retrieval precision is noticeably worse than your casual impression suggested, particularly for queries involving colors and counts (e.g., "three blue chairs" retrieves images with the wrong color or wrong number of chairs, even when a genuinely correct match exists in your small document store).

**Your task:** Using this module's theory (particularly section 6.4's discussion of known multimodal limitations, and the patch-embedding compression argument), explain at least two distinct, plausible mechanistic reasons why color-and-count-specific queries might show measurably worse retrieval precision than more general content queries in a CLIP-style shared embedding space. Then, using Module 8's evaluation discipline, propose a concrete diagnostic step: how would you determine whether this is a genuine, inherent limitation of the embedding model's representation (little you can do about it directly, other than choosing a different embedding model or adding a complementary non-embedding-based check), versus a fixable problem with your specific golden dataset or document store (e.g., ambiguous or mislabeled test queries)?

---

## 16. Checklist

- [ ] I can explain the Vision Transformer patch-embedding pipeline end to end, and connect it explicitly to Module 5's tokenization and Module 4's embedding concepts.
- [ ] I can explain how spectrograms make audio tokenizable, and can describe the Whisper-style encoder-decoder pipeline using Module 6's own vocabulary (bidirectional encoder, causal decoder, cross-attention).
- [ ] I can explain both major multimodal fusion strategies (early fusion, late fusion + connector) and give a genuine trade-off of each.
- [ ] I can name specific, mechanistically-grounded limitations of current multimodal models, not just a vague "it's not perfect."
- [ ] I've completed `vit-patch-demo` and directly visualized patch tokenization (and, if attempted, patch-level attention).
- [ ] I've built `multimodal-rag-preview`, including a proper precision@k evaluation via `eval-framework`, and documented its real, tested limitations.
- [ ] I can write, from memory and in my own words, a coherent summary connecting Modules 4 through 10 into one unified story (section 6.5).
- [ ] I can answer all 8 interview questions without notes.

---

## 17. Summary

This capstone module closes Part 2 by showing that everything you've learned about text — tokenization, embeddings, attention — generalizes to images and audio through nothing more exotic than a different front-end: images are chopped into patches and linearly embedded (Vision Transformers), audio is converted to a spectrogram and patch-embedded in much the same way (Whisper-style models), and both are fed into the exact same attention-based transformer architecture from Module 6, combined with text via either a unified early-fusion sequence or a separate-encoder-plus-connector late-fusion strategy. Multimodal capability is genuinely powerful but not undifferentiated — real, architecturally-grounded limitations exist (precise counting, fine spatial reasoning), and Module 8's evaluation discipline applies to multimodal claims exactly as rigorously as to any other model capability claim. Zooming out across all of Part 2: a small, recombination-friendly set of mechanisms — matrices trained by gradient descent against a simple, scalable, often self-supervised objective — repeatedly and reliably forces meaningful emergent structure, from word2vec's semantic geometry, through attention's contextual embeddings and specialized heads, to pretraining's world knowledge, to reasoning models' verifiable-reward-trained multi-step deliberation, to a single architecture's generalization across every modality covered in this module. You've built a working, mechanistic understanding of the entire modern foundation-model pipeline, from first principles, and validated every major claim through hands-on implementation — `toy-word2vec`, `toy-bpe`, `toy-transformer`, `lora-finetune-toolkit`, `eval-framework`, `reasoning-vs-standard-lab`, and now `multimodal-rag-preview`.

---

## 18. Next Steps

**Part 2 is now complete.** You move next into **Part 3 — LLM Engineering**, which shifts the handbook's center of gravity from "how models work internally" to "how to build reliable, production-grade systems on top of models you now deeply understand." Part 3 covers prompting and prompt engineering (grounded, per this module's closing review, in a genuine understanding of *why* certain prompts work rather than folklore), structured outputs, function/tool calling and MCP, memory, evaluation of full LLM-powered pipelines (directly extending `eval-framework`), guardrails, context engineering, caching, and latency/cost optimization. This is also where `llm-client-core` (Part 1, Module 2) finally gets real, production provider adapters (Anthropic, OpenAI) plugged in, turning the pluggable-client abstraction you designed early in the handbook into a genuinely functioning, production-usable component.

---

Congratulations on completing Part 2 — Reply "continue" to begin Part 3, Module 1, or flag anything to go deeper on first.
