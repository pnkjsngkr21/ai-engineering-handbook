# Part 2, Module 5: Tokenization — The Real Atomic Unit of Language Models

---

## 1. Learning Objectives

By the end of this module, you will be able to:

- Explain precisely why words are the *wrong* atomic unit for modern language models, and what problem subword tokenization solves.
- Derive and implement Byte-Pair Encoding (BPE) from scratch — the actual algorithm behind GPT-family tokenizers.
- Explain WordPiece and SentencePiece as variations on the same core idea, and know which major model families use which.
- Reason precisely about the practical consequences of tokenization: context window math, cost (tokens = money), multilingual fairness, and bizarre model failures (e.g., why LLMs are historically bad at counting letters in a word).
- Rebuild `embedding-service` (Part 2, Module 4) so its embedding lookup table is indexed by *subword tokens*, not whole words — the correct, production-accurate architecture.

---

## 2. Prerequisites

- Part 2, Module 4 (Embeddings) — you need to understand that an embedding is a lookup table indexed by vocabulary ID; this module explains precisely what goes *into* that vocabulary and why.
- Comfortable with basic string/byte manipulation in Python.
- Comfortable with frequency counting and greedy algorithms (you've done BFS/graph algorithms in your LeetCode prep — BPE is a different flavor of greedy iterative algorithm, closer in spirit to Huffman coding).

---

## 3. Estimated Study Time

**4–6 hours** (1.5–2 hours theory, 2–3 hours implementation, 1 hour exercises).

---

## 4. Difficulty

★★☆☆☆ (2/5) — Conceptually the simplest module in Part 2 so far. The algorithm is genuinely just "count pairs, merge the most frequent, repeat." The value here is less about difficulty and more about how much downstream confusion this one topic prevents.

---

## 5. Why This Matters

You just spent an entire module building word2vec on the assumption that "word" is the natural atomic unit of text. It isn't — and pretending it is would silently break the moment you tried to use your `embedding-service` in a real production system. Consider what actually happens with a whole-word vocabulary:

- **Out-of-vocabulary (OOV) words break everything.** Your training corpus doesn't contain "Bhayandar," "DoorDash," "CompletableFuture," or the misspelling "teh." A whole-word tokenizer either crashes, maps all unknown words to a single generic `<UNK>` token (destroying all information), or needs a vocabulary so enormous it becomes unusable.
- **Vocabulary size explodes with morphology.** "Run," "runs," "running," "runner," "reran" are five *different* entries in a whole-word vocabulary, each needing its own embedding row, each seen fewer times in training (worse learned representations), despite obviously sharing meaning.
- **Every language other than English breaks differently, but breaks.** Agglutinative languages (Turkish, Finnish) can produce a nearly unbounded number of whole-word forms from one root. A whole-word vocabulary either fails on these languages or needs to be language-specific.

Subword tokenization is the industry-standard fix, and once you understand it you unlock a huge amount of otherwise-mysterious behavior you've probably already noticed if you've used ChatGPT or Claude: why API pricing is "per token" and not "per word," why the context window is stated in tokens, why non-English text often costs more tokens for the same sentence, and — famously — why LLMs used to fail spectacularly at questions like "how many letters are in the word 'strawberry'" (the model never sees individual letters; it sees a small number of subword chunks and has to *infer* spelling, which is a genuinely hard task for it).

This module is short by design — it's a single, sharp mental correction (the atomic unit is a *subword token*, not a word) that you need firmly in place before Module 6 (Attention & Transformers), where "sequence of tokens" is the fundamental object every single mechanism operates on.

---

## 6. Theory

### 6.1 The Spectrum: Characters vs. Words vs. Subwords

There are three natural choices for the atomic unit of text, and it's worth seeing why each pure extreme fails before subword tokenization's middle ground makes sense.

**Character-level tokenization** — vocabulary is just the alphabet plus punctuation (maybe ~100–300 tokens total). Pros: no OOV problem, ever — any string can be represented. Cons: sequences become enormous (a 5-word sentence becomes ~25 characters), and the model has to learn "what a word even is" from scratch, burning enormous model capacity and compute on a problem that's trivial for humans. This is analogous to forcing your Spring Boot application to work exclusively with raw byte streams instead of parsed Java objects — technically complete, but you're pushing an enormous amount of avoidable parsing work into every downstream layer.

**Word-level tokenization** — what you implicitly assumed in Module 4. Pros: sequences are short, each token carries a lot of meaning. Cons: unbounded vocabulary (new words, typos, names, morphological variants), the OOV problem described above, and no ability to exploit the fact that "unhappiness" is obviously "un" + "happy" + "ness."

**Subword tokenization** — the actual industry answer. A vocabulary of a few thousand to ~100,000+ "pieces," where common whole words ("the," "is," "cat") get their own single token, and rarer or unseen words get broken into meaningful fragments ("unhappiness" → `["un", "happiness"]` or similar; "CompletableFuture" → `["Complet", "able", "Future"]` or similar, depending on the exact tokenizer). This gives you: bounded, fixed vocabulary size (no OOV — any string can always be decomposed to individual bytes/characters as a fallback), reasonably short sequences, and some exploitation of morphological structure, all at once.

This is a genuinely elegant compression trade-off, and if you want a precise analogy from your own field: it's structurally identical to the reasoning behind variable-length instruction encoding in CPU architectures, or Huffman coding in general — **give short, cheap representations to frequent things, and allow longer, more expensive representations for rare things**, rather than fixing one representation length for everything.

### 6.2 Byte-Pair Encoding (BPE), Precisely

BPE is a compression algorithm from the 1990s, repurposed for tokenizer vocabulary construction in 2015 (Sennrich et al.) and used, in variants, by GPT-2/3/4, LLaMA, and many others. The algorithm:

**Training (done once, offline, to build the vocabulary):**

1. Start with a vocabulary of all individual bytes/characters present in your training corpus (this is your bottom-level fallback — guaranteed no OOV).
2. Represent every word in your corpus as a sequence of these characters, with a special end-of-word marker (so the algorithm can distinguish "est" as a suffix from "est" as its own word).
3. Count the frequency of every *adjacent pair* of symbols across the entire corpus.
4. Find the single most frequent pair (e.g., `("e", "s")` if "es" appears constantly) and **merge it into a new single symbol** — add it to the vocabulary as one unit.
5. Replace every occurrence of that pair in the corpus with the new merged symbol.
6. Repeat steps 3–5 for a fixed number of merges (this merge count is a hyperparameter — GPT-2 used ~50,000 total vocabulary size / merges).

**The result** is a vocabulary that starts at individual bytes and progressively builds up common multi-character chunks, then common sub-words, then — for very frequent whole words like "the" — the entire word as one token, purely as a consequence of frequency-driven greedy merging.

**Inference (used every time you tokenize new text):** apply the learned merge rules, in the order they were learned during training, to any new input text. This is deterministic and fast — you're not recomputing frequencies, just applying a fixed, ordered list of merge rules.

Let's trace a tiny concrete example. Corpus: `"low lower lowest"` (with `</w>` marking end-of-word):

```
Initial (character-level):
l o w </w>   l o w e r </w>   l o w e s t </w>

Pair frequencies: (l,o)=3, (o,w)=3, (w,e)=2, (e,r)=1, (e,s)=1, (s,t)=1, (w,</w>)=1 ...
Most frequent: (l,o) with count 3 → merge into "lo"

After merge 1:
lo w </w>   lo w e r </w>   lo w e s t </w>

Pair frequencies: (lo,w)=3, ...
Most frequent: (lo,w) with count 3 → merge into "low"

After merge 2:
low </w>   low e r </w>   low e s t </w>

Pair frequencies: (e,r)=1, (e,s)=1, (low,e)=2, ...
Most frequent: (low,e) with count 2 → merge into "lowe"

After merge 3:
low </w>   lowe r </w>   lowe s t </w>
```

Notice what's happening: "low" became its own single token after just 2 merges (because it's the common shared root, appearing in all three words), while "er," "est" remain to be merged in later steps as separate suffix pieces. This is exactly the outcome you want — "low," "lower," "lowest" end up sharing the "low" token, meaning the model can transfer what it learns about "low" across all three, while still keeping "-er" and "-est" as reusable suffix tokens applicable to completely different root words later ("small" + "-er", "fast" + "-est").

### 6.3 WordPiece and SentencePiece — Same Idea, Different Merge Criterion / Preprocessing

**WordPiece** (used by BERT) is nearly identical to BPE with one key difference: instead of merging the *most frequent* pair, it merges the pair that most increases the **likelihood of the training corpus** under a simple language model — a slightly more principled but computationally similar criterion. In practice, this produces broadly similar-looking vocabularies to BPE.

**SentencePiece** (used by T5, LLaMA, and many multilingual models) isn't a different merge algorithm at all — it's a different *preprocessing* wrapper. The key insight: BPE as originally described assumes you can already split text into words using whitespace, which is a Western-language assumption that fails for languages without whitespace-delimited words (Japanese, Chinese, Thai). SentencePiece treats the *raw input string, including whitespace itself*, as just another character sequence to be tokenized — whitespace becomes a regular symbol (commonly rendered as `▁`), not special punctuation. This makes the entire pipeline language-agnostic and perfectly reversible (you can always reconstruct the exact original string, including spacing, from the token sequence — genuinely important for both correctness and for languages where whitespace carries information).

You don't need to memorize which model uses which by rote — the important thing for interviews and practical work is understanding that **all three solve the identical problem (bounded vocabulary, no OOV, exploit shared substructure) with minor variations in the merge criterion and text preprocessing.**

### 6.4 The Concrete, Practical Consequences You'll Hit Immediately

**Tokens, not words, are the unit of everything.** Context window sizes ("128K context," "1M context") are stated in tokens. API pricing is per-token. As a rough rule of thumb for English text, **1 token ≈ 4 characters ≈ 0.75 words** — but this is only a rough average; it varies significantly by tokenizer and language.

**Non-English languages often cost more tokens for the same meaning.** Because BPE vocabularies are built from training corpora that are disproportionately English, common English words often get single-token representations, while the equivalent word in, say, Hindi or Japanese may fragment into several tokens for the same semantic content — a real, measurable fairness and cost issue in multilingual deployment that's an active area of tokenizer research.

**Numbers tokenize weirdly and inconsistently**, which is part of why LLMs have historically struggled with arithmetic — "1234" might tokenize as one piece in one model and as `["12", "34"]` in another, and there's no guarantee digits are chunked in a way that respects place value. Some newer tokenizers deliberately force digit-by-digit tokenization specifically to improve arithmetic performance — a direct example of tokenizer *design decisions* propagating into model *capability* differences.

**"Letters in a word" tasks are hard because the model literally doesn't see letters.** If "strawberry" tokenizes as a single opaque token (or two or three subword chunks), the model has no direct access to the individual character sequence — it has to have implicitly memorized the spelling from training data, the same way you might struggle to instantly count the syllables in a word you've heard your whole life but never had to explicitly decompose. This is a tokenization-caused blind spot, not a "the model is bad at counting" problem in general.

### 6.5 The Embedding Table, Revisited

Recall from Module 4: the embedding matrix `W_in` has shape `(vocab_size, d)`. Now you know precisely what fills each row: not a whole word, but a *subword token* from the BPE/WordPiece/SentencePiece vocabulary. A real production tokenizer's vocabulary size is typically 30,000–100,000+ tokens (GPT-2: ~50,000; GPT-4 family and LLaMA 3: ~100,000-128,000+). Every one of those vocabulary entries gets its own row in the embedding table, learned via exactly the mechanism you built in Module 4 — the only thing that's changed is *what counts as a "word" being embedded.*

This is why `embedding-service`'s production upgrade in this module matters: your Module 4 version, using `sentence-transformers`, was already correctly using a subword tokenizer under the hood (every real embedding model does) — but you hadn't yet made that explicit or inspectable. This module makes it visible and lets you verify it directly.

---

## 7. Mental Models

1. **"Tokens are the currency; words are just a human convenience."** Everything downstream — cost, context limits, model capability — is denominated in tokens, not words.
2. **"BPE is greedy compression: merge what's frequent, keep what's rare intact."** The exact same instinct as Huffman coding — short codes for common things.
3. **"No such thing as out-of-vocabulary, only 'falls back to smaller pieces.'"** Subword tokenization's core guarantee: any string, however novel, can always be represented — worst case, byte by byte.
4. **"If the model can't see the letters, don't expect it to reason about the letters."** The single most useful debugging mental model for a whole category of "why did the LLM get this obviously simple thing wrong" bugs.

---

## 8. Visual Explanation

**Diagram 1 — Three Tokenization Granularities, Same Sentence**

```
Sentence: "The unhappiness was overwhelming."

CHARACTER-LEVEL (huge sequence, tiny vocab):
[T,h,e, ,u,n,h,a,p,p,i,n,e,s,s, ,w,a,s, ,o,v,e,r,w,h,e,l,m,i,n,g,.]
  33 tokens, vocab size ~100

WORD-LEVEL (short sequence, unbounded vocab, OOV risk):
[The, unhappiness, was, overwhelming, .]
  5 tokens, vocab size unbounded — "unhappiness" needs its OWN vocab entry

SUBWORD-LEVEL (BPE, the real answer — short-ish sequence, bounded vocab):
[The, un, happi, ness, was, overwhelm, ing, .]
  8 tokens, vocab size ~50,000 — shares "un-", "-ness", "-ing" with
  thousands of other words it's never even seen combined this way
```

**Diagram 2 — BPE Merge Process (Huffman-coding-style intuition)**

```
Frequency-driven greedy merging, round by round:

Round 1:  find most frequent adjacent PAIR of symbols  →  merge it
Round 2:  find most frequent adjacent PAIR (now including new merged
          symbols as candidates)                         →  merge it
Round 3:  repeat...
   ...
Round N:  stop after a fixed vocabulary-size budget is reached

Result: common short chunks emerge early (single letters → common
bigrams → common syllables → very common whole words), rare
combinations are never merged and stay as several separate tokens.
```

**Diagram 3 — Where Tokenization Sits in the Pipeline**

```
Raw text
   │
   ▼
[TOKENIZER]  ── string → list of token IDs (this module)
   │
   ▼
Token IDs: [464, 2159, 373, ...]
   │
   ▼
[EMBEDDING LOOKUP]  ── token ID → dense vector, via W_in[id]  (Module 4)
   │
   ▼
Sequence of embedding vectors
   │
   ▼
[TRANSFORMER / ATTENTION]  ── Module 6, next
```

---

## 9. Recommended Resources

1. **[Andrej Karpathy — "Let's build the GPT Tokenizer" (video + code)](https://www.youtube.com/watch?v=zduSFxRajkE)** — The single best resource for this topic anywhere; Karpathy builds a BPE tokenizer completely from scratch on video, in Python, and it's the direct inspiration for this module's mini-project. Watch this after attempting the mini-project yourself, then again to check your understanding.
2. **[Sennrich, Haddow, Birch — "Neural Machine Translation of Rare Words with Subword Units" (2015)](https://arxiv.org/abs/1508.07909)** — The original paper that introduced BPE for NLP vocabulary construction; short and directly readable.
3. **[Hugging Face — "Summary of the Tokenizers" (official docs)](https://huggingface.co/docs/transformers/tokenizer_summary)** — The clearest official comparison of BPE, WordPiece, and SentencePiece side by side, with the exact algorithmic differences spelled out.
4. **[OpenAI — `tiktoken` GitHub repository](https://github.com/openai/tiktoken)** — The actual production BPE tokenizer used by GPT models; use its interactive "Tokenizer" tool (linked from OpenAI's platform docs) to paste in real sentences and see exactly how they get chunked — invaluable for building fast, correct intuition.
5. **[Google — SentencePiece GitHub repository and paper](https://github.com/google/sentencepiece)** — Read the README for the precise motivation behind whitespace-as-symbol preprocessing, especially the multilingual justification.

---

## 10. Exercises

1. **Trace BPE by hand.** Given the tiny corpus `"aaabdaaabac"` (a classic didactic BPE example, no word boundaries), manually perform the first 3 merge steps of byte-pair encoding, showing pair-frequency counts at each step.
2. **Predict, then verify.** Before running any code, predict how the string `"tokenization"` would likely be split by a BPE tokenizer trained on general English text (which prefixes/suffixes are common enough to be single tokens?). Then use OpenAI's `tiktoken` (or the interactive web tool) to check your prediction against a real GPT tokenizer.
3. **Cost estimation.** Using `tiktoken`, compute the token count for the same paragraph of text written in English and (if you can source or write a rough equivalent) in Hindi or another non-Latin-script language you're comfortable with. Compare token counts and connect the difference back to Theory section 6.4.
4. **OOV stress test.** Feed a BPE tokenizer a deliberately invented, never-before-seen "word" (e.g., a random string like `"zqvexblatt"`) and observe how it gets decomposed. Confirm it never crashes or returns an `<UNK>` — trace exactly which fallback pieces it uses.
5. **Digit tokenization audit.** Using `tiktoken`, check how the numbers `"1234567"`, `"100"`, and `"3.14159"` are tokenized. Connect what you observe back to Theory section 6.4's explanation of LLM arithmetic weaknesses.
6. **Reversibility check.** Tokenize a sentence with unusual spacing (e.g., double spaces, a leading tab) using a SentencePiece-based tokenizer if you have one available, and confirm you can perfectly reconstruct the original string — including whitespace — from the token sequence alone.

---

## 11. Mini-Project: `toy-bpe`

Implement Byte-Pair Encoding completely from scratch in Python (no libraries beyond `collections` for counting) — directly following the algorithm in Theory section 6.2.

**Requirements:**
- A `train(corpus: str, num_merges: int) -> list[tuple[str, str]]` function that returns the ordered list of learned merge rules, following the counting → merge → repeat loop.
- A `tokenize(text: str, merges: list[tuple[str, str]]) -> list[str]` function that applies the learned merges, *in the order they were learned*, to new text.
- Train on a few paragraphs of real English text (a few hundred words is enough to see interesting structure — try Wikipedia's opening paragraph on any topic you like) for ~50–100 merges.
- Print the vocabulary after training and manually inspect: do you see recognizable English morphemes emerging (e.g., "-ing," "-tion," "the")?
- Tokenize a *held-out* sentence (not in the training corpus) and confirm it decomposes sensibly, with no crashes on any word, including ones never seen during training.

---

## 12. Production Project: `embedding-service` v2 — Tokenizer-Aware Upgrade

This directly extends `embedding-service` from Part 2, Module 4 (do not start a new service — this is a targeted upgrade).

**What changes:**

1. **Expose the tokenizer explicitly as a first-class part of the API**, rather than leaving it hidden inside the embedding model:
   ```
   POST /tokenize
   { "text": "CompletableFuture is great for fan-out patterns" }
   →
   { "tokens": ["Complet", "able", "Future", " is", " great", ...],
     "token_ids": [7085, 481, 24029, ...],
     "token_count": 9 }
   ```
2. **Add a `/estimate-cost` endpoint** that, given a batch of texts and a target model name, returns the token count for each — directly useful for the RAG chunking work coming in Part 4, where chunk size needs to be planned in *tokens*, not characters or words.
3. **Add tokenizer-awareness to your caching layer** (from Module 4's Redis integration): cache keys should now explicitly include the tokenizer/model identifier, since the same raw text produces different token sequences under different tokenizers — a subtle correctness bug if ignored (two different models' embeddings could otherwise collide in cache under a naive text-only key).
4. **Add a golden test suite**: a fixed set of ~15 tricky strings (empty string, emoji, code snippets with `CompletableFuture`-style CamelCase, non-English text, numbers, repeated whitespace) with their expected token counts pinned as test fixtures, so future changes to the underlying model/tokenizer are caught by CI (reuse your tiered CI pipeline from Part 1, Module 8) rather than silently changing chunking/cost behavior in production.
5. **Document, in the service's README, the exact token-to-character/word ratio you measured empirically** for your test corpus — this becomes a concrete, reusable planning number for Part 4's chunking strategy design.

**Explicitly designed for extension:** the `/estimate-cost` and `/tokenize` endpoints become directly load-bearing in Part 4 when you design chunk sizes for RAG (chunks are sized in tokens to respect context window budgets) and in Part 3 when you reason about prompt construction costs and context window management for `llm-client-core`.

---

## 13. Practice & Interview Questions

1. Why do modern LLMs use subword tokenization instead of whole-word tokenization? Name at least two distinct failure modes of whole-word tokenization that subword tokenization fixes.
2. Walk through the BPE training algorithm from memory, in your own words, using the "low/lower/lowest" style example.
3. What is the practical difference between BPE and WordPiece? Between BPE and SentencePiece?
4. Why can a well-designed subword tokenizer never produce an "out-of-vocabulary" error, even on text with typos, code, or made-up words?
5. Explain, mechanistically (not just "it's weird"), why LLMs have historically underperformed on character-counting tasks like "how many r's are in strawberry."
6. A PM asks why the API bills "per token" instead of "per word" or "per character" — give a precise, technically grounded answer connecting this to the tokenizer.
7. Why might the same sentence, translated into two different languages, cost meaningfully different numbers of tokens to process through the same model? What's the practical/ethical concern this raises?
8. If you were designing a chunking strategy for a RAG system (Part 4 preview), why would you compute chunk boundaries in tokens rather than characters or words?

---

## 14. Common Mistakes

- **Assuming "1 token ≈ 1 word."** It's a rough average at best, and wildly wrong for non-English text, code, and technical vocabulary — always measure with the actual tokenizer, never assume.
- **Estimating context window usage or API cost by counting words or characters instead of actually tokenizing.** This routinely leads to bugs where a prompt that "should fit" overflows the model's context window in production.
- **Forgetting that different models use different tokenizers.** A token count computed with one model's tokenizer (e.g., GPT-4's) does not transfer to another model (e.g., LLaMA's) — always tokenize with the tokenizer matching the specific model you're calling.
- **Building a cache keyed only on raw text, ignoring the tokenizer/model identifier** — a real bug class that silently serves wrong-model embeddings or token counts from cache.
- **Expecting an LLM to reliably do character-level tasks (spelling, counting letters, reversing a string) without accounting for the fact that it doesn't directly observe individual characters** — set expectations (and design workarounds, like asking the model to space out letters) accordingly.

---

## 15. Debugging Exercise

Your team's RAG prototype (a preview of Part 4) is failing a subtle but important test: chunks that are supposed to be "under 500 tokens" to fit a downstream model's limits are intermittently exceeding that budget in production, causing occasional API errors from the downstream LLM about context length.

Here's the chunking code in question:

```python
def chunk_text(text: str, max_tokens: int = 500) -> list[str]:
    words = text.split()
    chunks = []
    current_chunk = []
    for word in words:
        current_chunk.append(word)
        if len(current_chunk) >= max_tokens:  # <-- bug is here
            chunks.append(" ".join(current_chunk))
            current_chunk = []
    if current_chunk:
        chunks.append(" ".join(current_chunk))
    return chunks
```

**Your task:** Identify precisely why this function can silently produce chunks that exceed the actual token budget, even though it appears to enforce a `500` limit. Rewrite it to correctly enforce a *true* token-based limit, using the tokenizer-aware `embedding-service` endpoints you built in this module's production project (or a real tokenizer library directly) rather than counting whitespace-separated words. Explain, in a sentence or two, why this exact class of bug is easy to miss in code review — the function *looks* correct and even has a variable named `max_tokens`.

---

## 16. Checklist

- [ ] I can explain why subword tokenization exists, using the OOV and morphology problems as concrete motivation.
- [ ] I can trace the BPE training algorithm by hand on a small example and explain each step's purpose.
- [ ] I've implemented `toy-bpe` from scratch and confirmed it produces sensible subword splits on real text.
- [ ] I understand the practical differences between BPE, WordPiece, and SentencePiece at a level I could explain in an interview.
- [ ] I can explain, mechanistically, why token count ≠ word count ≠ character count, and why this matters for cost and context windows.
- [ ] I've upgraded `embedding-service` with explicit tokenizer endpoints, cost estimation, and tokenizer-aware caching, with a golden test suite in place.
- [ ] I can answer all 8 interview questions without notes.

---

## 17. Summary

Words are a human convenience, not the atomic unit language models actually operate on. Subword tokenization — via BPE, WordPiece, or SentencePiece, all variations on the same greedy, frequency-driven merging idea — solves the out-of-vocabulary problem completely, keeps vocabularies bounded, and exploits shared morphological structure across related words, at the cost of needing to explicitly reconstruct "words" from pieces when necessary. You implemented BPE from scratch in `toy-bpe`, traced exactly how common English morphemes emerge purely from merge frequency, and understood the very real practical consequences: token-based (not word-based) pricing and context limits, multilingual token-cost disparities, and specific, mechanistically explainable LLM weaknesses like poor character counting. You then upgraded `embedding-service` to expose its tokenizer explicitly, added cost estimation, and made your caching layer tokenizer-aware — directly setting up the token-budget-driven chunking strategy you'll need for RAG in Part 4.

---

## 18. Next Steps

**Next module: Part 2, Module 6 — "Attention & the Transformer Architecture."** You now have both halves of the input pipeline solid: text → subword tokens (this module) → dense embedding vectors (Module 4). Module 6 tackles the mechanism that made modern LLMs possible: how a sequence of embedding vectors gets transformed, via the attention mechanism, into *contextual* representations — finally delivering on the "static vs. contextual embeddings" promise from Module 4, and giving you the architecture underlying every model you'll call via API for the rest of this handbook.

---

Reply "continue" for Module N, or flag anything to go deeper on.
