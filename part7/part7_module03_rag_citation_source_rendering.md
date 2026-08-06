# Part 7, Module 3 — RAG Citation & Source Rendering

## 1. Learning Objectives

By the end of this module you will be able to:

1. Design a citation contract that maps generated text spans to the retrieved chunks (`rag-engine`, Part 4) that supposedly support them, without ever letting the frontend invent an attribution the backend didn't confirm.
2. Distinguish three separate claims a citation can make — "this text is *grounded in* this source," "this source was *retrieved*," and "this source was *used* in the final answer" — and render each correctly instead of collapsing them into one undifferentiated "sources" list.
3. Render citations as a third independent live stream alongside the token stream (Module 1) and the trace-event stream (Module 2), reusing the `type`-discriminated SSE multiplexing pattern established last module instead of inventing new transport.
4. Build an inline citation marker that stays correctly anchored to its text span even while that span is still streaming and potentially being revised by Module 1's auto-close/repair logic.
5. Surface `rag-engine`'s faithfulness check (Part 4, Module 8) as a visible, honest signal — including what a UI should do when a claim fails faithfulness verification *after* it has already been shown to the user.
6. Handle the access-control seam from Part 4's capstone (the auth-before-scoring RRF rank-leak) at the UI layer: never render a citation preview for a source the current user isn't authorized to see, even transiently.
7. Extend `chat-shell` v2 into v3, adding a citation panel that a user can open per-message without breaking the trace panel from Module 2.

## 2. Prerequisites

- Part 7, Modules 1–2 (Streaming Interfaces; Visualizing Agent Reasoning) — this module reuses both the partial-structured-output renderer and the multi-stream SSE architecture directly.
- Part 4, Module 8 (RAG Evaluation) — context precision/recall, faithfulness, and answer relevance as distinct signals; you're about to render one of these live for the first time.
- Part 4, Module 9 (RAG capstone) — `RAGEngine`'s single public entry point, and the audited access-control seam (filtering after fusion can leak information about documents a user can't see).
- Part 4, Module 5 (Metadata Filtering) — access control as a security boundary enforced in code, never in a prompt or, as this module adds, never in a UI assumption either.

## 3. Estimated Study Time

9–12 hours (theory + exercises: ~2.5 hours; mini-project: ~2 hours; production project: ~5–7 hours).

## 4. Difficulty

★★★☆☆ (3.5/5) — Less simultaneous state than Module 2 (no human-decision gate here), but the span-anchoring problem under streaming text is genuinely fiddly, and the access-control requirement raises the security stakes higher than a typical rendering task.

## 5. Why This Matters

`rag-engine` shipped in Part 4 with a hard rule: internal components aren't importable, so the security boundary can't be bypassed by application code. That discipline is worthless if the *frontend* takes the final answer text and a loose bag of "sources used" and free-associates which source supports which sentence — which is exactly what happens if you build a citation UI by just showing a numbered list at the bottom of the response, Wikipedia-footnote style, without a real span-to-source mapping from the backend. Users trust citations more than they trust unattributed text, often *more* than the confidence is warranted — which makes a sloppy or invented mapping actively worse than no citations at all, not just cosmetically wrong.

There's also a live continuation of Module 2's central lesson: a UI must never render information the backend hasn't structurally confirmed. Last module that meant tool-call states; this module it means citation-to-span mappings, and — sharper — it means never rendering even a *preview* of a source the current user isn't authorized to read, which is a genuine security requirement, not a UX one. Part 4's capstone found that rank leakage through result *ordering* alone can expose the existence of restricted documents. A citation UI that shows an unauthorized document's title in a hover preview is the same leak through a different channel, and it's the kind of thing that's easy to miss because it feels like "just UI."

## 6. Theory

### 6.1 Three different claims, one careless "Sources" list

A typical naive citation UI collapses three genuinely different facts into one undifferentiated list at the bottom of a response:

- **Retrieved**: this chunk came back from `rag-engine`'s retrieval/re-rank pipeline (Part 4, Modules 2–4) for this query. Says nothing about whether the model used it.
- **Used**: the model's context actually included this chunk when generating the response (post-compression, Part 4, Module 7 — a retrieved chunk can be dropped or compressed away before generation and never influence the answer at all).
- **Grounded-in**: a *specific span* of the generated answer is faithfulness-verified (Part 4, Module 8) against this specific chunk — the strongest and narrowest claim, and the only one that should produce an inline citation marker on actual text.

These need three different renderings, because they carry three different amounts of trust. A "Sources" list showing everything *retrieved* implies grounding it hasn't earned. `rag-engine`'s public trace already distinguishes these (they're separate stages in Part 4's `RAGTraceScorer`); this module's job is to not throw that distinction away at render time.

```typescript
type CitationClaim = "retrieved" | "used" | "grounded";

interface Citation {
  chunk_id: string;
  source_title: string;
  claim: CitationClaim;
  // Only present when claim === "grounded":
  span_start?: number;   // character offset into the final answer text
  span_end?: number;
  faithfulness_score?: number;  // from Part 4 Module 8's verification pass
}
```

Only `grounded` citations get an inline marker on the text. `used`-but-not-`grounded` chunks (context that influenced the answer generally but backs no specific claim precisely enough to verify) go in an expandable "Also referenced" section, visually subordinate. `retrieved`-only chunks aren't shown to the end user at all by default — they're debugging/eval information, same tier-3 treatment as Module 2's "never sent to the client" internal-noise category, though unlike that category these *are* safe to send (no secrets), just not useful to show by default. Reserve them for a future operator/debug view.

### 6.2 Span anchoring under a still-streaming answer

An inline citation marker (a small superscript, `[1]`) has to sit at a specific character offset in the answer text. But the answer is streaming (Module 1), and Module 1's auto-close logic means the *displayed* text can differ from the final text at any given moment — a markdown list item might be auto-closed for display and then continue differently once more tokens arrive.

The fix is the same discipline Module 1 already established for structured fields: **a citation marker is only rendered once its span is confirmed complete**, which in practice means the span arrives as its own event, keyed to the *final* answer text's offsets (which `rag-engine` knows because the faithfulness check runs after generation, not during), not to whatever the display buffer looks like mid-stream. Concretely: citation events arrive on the event stream *after* the corresponding `final_answer` event (§6.3 below), never interleaved with still-in-flight tokens. This sidesteps the anchoring problem entirely rather than solving it live — a deliberately simpler design than trying to re-anchor markers against a moving display buffer, and the right trade-off, because citations answering "what supports this claim" are naturally a post-hoc concern, not something a user needs mid-generation.

```
token stream:      "Revenue grew 12% in Q3, driven mainly by..."  (still arriving)
                                        │
                                        ▼  generation completes
event stream:       final_answer  ──▶  citation(span: 0-24, chunk: doc_44, claim: grounded)
                                    ──▶ citation(span: 40-71, chunk: doc_51, claim: grounded)
```

This is a meaningful simplification versus Module 2's tool-call states, which genuinely do need live mid-stream rendering because a user needs to know *right now* whether an action needs their approval. Citations don't carry that urgency — get the mapping right rather than getting it fast.

### 6.3 Reusing the multiplexed stream, not inventing a fourth transport

Module 2 established a `type`-discriminated `AgentEvent` schema riding the same SSE connection as the token stream. Citations extend that same schema with two more event types rather than opening a new connection or a new polling endpoint:

```typescript
type StreamEvent =
  | { type: "token"; text: string }                    // Module 1
  | { type: "tool_call_state"; ... }                    // Module 2
  | { type: "reflection"; ... }                         // Module 2
  | { type: "provenance"; ... }                         // Module 2
  | { type: "final_answer"; text: string }               // new: marks generation complete
  | { type: "citation"; payload: Citation };             // new: this module
```

The `citationReducer` is a third independent reducer alongside `tokenReducer` and `traceReducer` — same architecture as Module 2's §12.2, same reason: independently testable, no shared mutable state, no risk of one stream's bug corrupting another's render.

### 6.4 Faithfulness as a visible, honest signal — including failure

Part 4, Module 8 built faithfulness verification as an evaluation tool. This module is the first time it becomes something an end user sees directly, and that raises a question the eval context never had to answer: **what does the UI do when a claim fails faithfulness verification after the answer has already been shown?**

The honest options, in order of how much they cost trust if handled badly:

- **Best**: run the faithfulness check *before* streaming the final answer to the user at all, so a failing claim is caught and either regenerated or hedged before display. This is possible when latency budget allows a verification pass before the user-facing stream starts (Part 3, Module 10's verification-pass mechanism, reused here exactly as it was reused for RAG faithfulness in Part 4).
- **Acceptable, if the above isn't affordable**: stream the answer immediately for responsiveness, run faithfulness verification concurrently, and if a span fails, visually flag it in place (a distinct marker, not silently removed) rather than either hiding the failure or leaving an unverified claim looking identical to a verified one. Silently downgrading a failed claim to look like ordinary unmarked text is the one option that's never acceptable — it's strictly worse than not having citations, because it actively launders an unverified claim as unremarkable.
- **Never**: retroactively edit or delete the already-shown answer text to cover for a failed check. A user may have already read it; editing text out from under them without explanation is a worse trust violation than the original ungrounded claim.

`chat-shell` v3 implements the second option by default (concurrent verification with an in-place flag), with a documented note that the first option is a latency/quality trade-off worth revisiting once real production latency budgets are known — consistent with how Part 6 treated latency-vs-quality trade-offs throughout.

### 6.5 Access control at render time — closing the leak channel, not just the query channel

Part 4's capstone fixed rank leakage in `rag-engine`'s retrieval pipeline: filtering happens before fusion, not after, so a user's result ordering never reveals ranks of documents they can't see. That fix lives entirely server-side, inside `RAGEngine`'s one public entry point. It's tempting to assume that's the whole story and that anything `RAGEngine` returns to the client is therefore automatically safe to render however the UI wants.

It isn't, for one specific reason worth internalizing: **a citation preview is a second query surface**, even though it doesn't look like one. If a citation's `source_title` or a hover-preview snippet is rendered from data the client fetched *separately* from the original authorized query — for instance, a "click to see full source" feature that fetches a document by `chunk_id` on demand — that fetch needs the *exact same* authorization check the original retrieval had, every single time, not an assumption that "if it was cited, it must already be visible." A `chunk_id` leaking into client-side state and being fetchable independently, without re-checking access, is a real path around `RAGEngine`'s access-control boundary — it just doesn't look like a retrieval-time bug because it happens later, in a different component, well after the original authorized query returned.

The rule this module adds to the running access-control discipline: **every citation-triggered fetch re-enters through `RAGEngine`'s public interface and re-checks authorization, with no client-side or UI-layer exception**, even for data that was technically already "seen" once during the original response. No caching a document body client-side across users or sessions; no trusting a `chunk_id` as if it were itself an access token.

## 7. Mental Models

- **"Retrieved, used, and grounded are three different claims — don't let a UI collapse them into one 'Sources' list."**
- **"A citation is a post-hoc concern, not a live one — anchor it to the final text, not the moving stream."**
- **"An unverified claim shown as ordinary text is worse than no citation at all — flag it, don't launder it."**
- **"A `chunk_id` is not an access token — every citation fetch re-enters the authorization boundary."**

## 8. Visual Explanation

**Citation claim tiers, by default visibility:**

```
┌─ Answer text ────────────────────────────────────────────┐
│ Revenue grew 12% in Q3¹, driven mainly by enterprise      │
│ expansion in EMEA².                                       │  ← grounded: inline markers
│                                                             │
│ ▸ Also referenced (2 sources)                              │  ← used, not grounded: collapsed
│                                                             │
│ [retrieved-only chunks: not shown — debug/eval only]       │  ← never rendered to end user
└─────────────────────────────────────────────────────────────┘
   ¹ → Q3 Earnings Report, p.4   (faithfulness: ✓ verified)
   ² → Regional Sales Summary    (faithfulness: ⚠ flagged — tap for detail)
```

**Event ordering — citations arrive after generation, not interleaved with it:**

```
 token   token   token  ... token   final_answer   citation   citation
  │       │       │           │          │              │          │
  ▼       ▼       ▼           ▼          ▼              ▼          ▼
[tokenReducer accumulates]  [marks done]  [citationReducer places markers]
```

**Citation-triggered fetch re-entering the access boundary:**

```
 user clicks citation marker
          │
          ▼
 GET /rag/chunk/{chunk_id}?run_id=...
          │
          ▼
   RAGEngine.get_chunk(chunk_id, user_context)   ← same auth check as
          │                                          original retrieval,
          ▼                                          every time, no exception
   200 (authorized) | 403 (never happened, per user)
```

## 9. Recommended Resources

1. **Anthropic — "Citations" API documentation** (docs.anthropic.com) — the closest official reference for how a model-native citation feature separates "cited span" from "source document" at the API level; useful ground truth to compare your `Citation` contract against, even though `rag-engine`'s pipeline is retrieval-based rather than using the native citations feature directly.
2. **Part 4, Module 8 (RAG Evaluation)** — re-read specifically the section distinguishing faithfulness from answer relevance; this module is the direct UI payoff of that distinction and it's worth having it fresh.
3. **Nielsen Norman Group — "Trust and Credibility in Web Design"** — older but foundational on how source attribution affects perceived trust; relevant to §6.4's argument that a mishandled failed-faithfulness case actively costs more trust than no citation.
4. **W3C — "Using ARIA to enhance accessible names for footnotes/citations"** (WAI-ARIA practices) — inline citation markers need programmatic association with their footnote content for screen readers; don't ship a superscript number with no accessible relationship to what it references.
5. **OWASP — "Insecure Direct Object Reference" (IDOR) entry, Top 10 documentation** — directly relevant to §6.5; the citation-triggered-fetch problem is a textbook IDOR shape (an ID leaking into client reach without a re-checked authorization), worth reading in that specific framing rather than only as an AI-specific concern.

## 10. Exercises

1. Write the `citationReducer`'s action types as a TypeScript discriminated union, handling `final_answer` and `citation` events, and enforcing (in the reducer itself, not just by convention) that no `citation` event is processed before its `final_answer` has been received for that message.
2. A `used`-but-not-`grounded` chunk and a `retrieved`-only chunk both technically came from the same `RAGEngine.retrieve()` call. Design the exact backend-side filter that decides which of the three claim tiers a given chunk ends up in before it's ever serialized into an `AgentEvent`.
3. Design the in-place flag UI for a faithfulness failure (§6.4's middle option). What should it look like, and what should tapping it show the user — the failing text, the source it was checked against, or both?
4. Walk through §6.5's citation-triggered-fetch flow and identify: what specifically would go wrong if `chunk_id` were cached client-side across a logout/login as a different user on the same device? Design the fix.
5. `rag-engine`'s capstone found that filtering-after-fusion leaks rank information. Construct an analogous leak at the citation-rendering layer — a way a UI could reveal the *existence* of a restricted document without ever displaying its content. (Hint: think about what a "3 more sources" count, or a citation-count badge, might reveal even with zero content shown.)

## 11. Mini-Project

Build a standalone `<CitationText>` component that takes a fixed final answer string and a hardcoded array of `Citation` objects (mixing all three claim tiers) and renders: inline markers only for `grounded` claims, a collapsed "Also referenced" section for `used` claims, and nothing for `retrieved`-only claims. Include one deliberately-failing faithfulness score and implement the in-place flag from Exercise 3. No streaming yet, no backend — this isolates the rendering/tiering logic from the harder multi-stream wiring the Production Project handles.

## 12. Production Project: `chat-shell` v3 — Citation-Aware Trace UI

Extend `chat-shell` v2 (Module 2) into v3, adding real citation rendering wired to `rag-engine`.

**Scope:**

1. **Citation event types** added to the `AgentEvent`/`StreamEvent` contract (§6.3), emitted by `rag-engine` after generation completes, classified into the three claim tiers server-side (Exercise 2) before serialization — the tiering logic lives in `rag-engine`, never inferred client-side.
2. **`citationReducer`**, structurally independent of `tokenReducer` and `traceReducer`, verified via an extension of `agent-trace-render-stress-test` (Module 2) covering out-of-order and duplicate citation events.
3. **`<CitationText>`** from the Mini-Project, now wired to the live reducer, replacing the plain answer-text renderer from Module 1 for RAG-backed responses specifically (non-RAG chat responses from Module 1/2 render unchanged — this is additive, not a rewrite of the base renderer).
4. **In-place faithfulness-failure flag**, implemented per §6.4's concurrent-verification default, with the note documented as an ADR that the pre-verification option remains open for a future latency-budget revisit.
5. **Citation-triggered fetch endpoint**, `GET /rag/chunk/{chunk_id}`, implemented to re-enter `RAGEngine`'s public interface and re-check authorization on every call, with no caching layer that could let a previously-authorized fetch serve a different, unauthorized user or session.
6. **`citation-access-control-test`**: an explicit security test, extending Part 4, Module 5's `access-control-security-bench` philosophy into the frontend/API-boundary layer — attempts a citation fetch as an unauthorized user against a `chunk_id` legitimately obtained by an authorized one, and asserts a 403, not a cache hit. This is a CI-blocking test, same tier as `access-control-security-bench` itself.
7. **Existence-leak audit** (Exercise 5): review every UI element that shows a *count* of sources (e.g., "3 more sources") and confirm the count itself is computed only from documents the current user is authorized to see — not from an unfiltered retrieval count.

**Documentation**: an ADR for the claim-tiering design (§6.1) and its rationale, and an explicit note in `chat-shell`'s architecture doc that citation counts and previews are governed by the same access-control boundary as retrieval itself — closing the loop opened by Part 4's capstone limitation list.

**Hands off to:** the operator-facing SLO dashboard (later in Part 7), which will be the first place raw `retrieved`-only chunks and full provenance tags (deferred in Module 2) are surfaced, now to an authorized-operator audience rather than an end user; and the Part 7 capstone, which assembles Modules 1–3's three independent reducers plus the eventual approval-history and citation-history views into one polished application shell.

## 13. Practice & Interview Questions

1. Why does a citation UI need three distinct claim tiers (retrieved/used/grounded) instead of a single "sources" list, and what specifically goes wrong with user trust if they're collapsed?
2. Explain why citation events can safely arrive strictly after the `final_answer` event rather than needing to be anchored live during token streaming, and why that's a legitimate simplification rather than a shortcut that loses something important.
3. A `chunk_id` referenced in a citation leaks into client-visible state by necessity (the UI needs it to fetch/preview the source). Why is that not the same thing as the `chunk_id` being safe to treat as an access token, and what specific test would catch a regression here?
4. Design what "in-place flagging" of a failed faithfulness check should look like, and justify why silently removing or downgrading the claim to look like normal text is worse than showing it flagged.
5. How does the citation-triggered-fetch access-control requirement in §6.5 relate to Part 4's rank-leak seam — same underlying category of bug, or a genuinely different one? Defend your answer.
6. In an interview: you're asked to design source attribution for an AI answer product. Walk through why you'd want the claim-mapping to originate server-side (in the retrieval/generation service) rather than being inferred or fuzzy-matched client-side from the raw answer text and a source list.

## 14. Common Mistakes

- **Collapsing retrieved/used/grounded into one "sources" list.** Implies a grounding claim the system hasn't actually verified for most of what's listed.
- **Anchoring citation markers to the live streaming buffer instead of the final text.** Creates markers that visibly jump or misalign as auto-close/repair logic (Module 1) revises the display mid-stream.
- **Silently hiding or softening a failed faithfulness check.** The single worst option in §6.4 — it launders an unverified claim as ordinary, unremarkable text.
- **Treating a `chunk_id` as implicitly safe once it's appeared in a citation.** It must re-enter the same authorization check on every subsequent fetch, with zero exceptions, including for the same user re-fetching their own earlier citation.
- **Caching fetched source content client-side across sessions or users.** Turns a correct per-request authorization check into a leak the moment a device or session is shared or reused.
- **Showing a raw source count without checking it's computed post-authorization-filter.** An "N more sources" badge can leak the existence of restricted documents even with zero content shown.

## 15. Debugging Exercise

QA reports: a user occasionally sees a citation marker in the answer text, but tapping it shows "Source unavailable" instead of a preview — even though the answer text clearly claims something was grounded in it. It's inconsistent and hard to reproduce.

Form a hypothesis before reading further:

<details>
<summary>Hint 1</summary>
Re-read §6.5. The citation event tells the client a `chunk_id` exists and was used for grounding. The preview is a *separate* fetch, re-checking authorization every time, by design. What happens if those two things — "this chunk grounded a claim" and "this user is currently authorized to fetch this chunk" — can legitimately disagree at different moments?
</details>

<details>
<summary>Hint 2</summary>
Think about what could change a user's authorization *between* the original retrieval (which already filtered correctly) and the moment they click the citation marker, possibly minutes later.
</details>

<details>
<summary>Likely root cause</summary>
This is very likely not a bug at all, but the access-control design working exactly as intended under a legitimate scenario: the user's permissions changed between when the answer was generated (and correctly grounded in a document they *were* authorized to see at that moment) and when they clicked the citation (by which point, e.g., a document's sharing settings changed, or their session's permissions were revoked/altered). §6.5's rule — every citation fetch re-checks authorization fresh, every time, no exception — means this is exactly the expected behavior when authorization state legitimately changes between generation and click. The actual fix isn't in the access-control logic (which is correct); it's in the UI/UX: "Source unavailable" is a confusing message for this specific case. It should distinguish "no longer accessible to you" (a real, informative state) from a generic error, so the inconsistency reads as expected behavior instead of a bug. This is also worth logging distinctly server-side, since a sudden spike in this specific denial reason is a useful signal that permissions data may be propagating inconsistently.
</details>

## 16. Checklist

- [ ] Citation contract distinguishes retrieved / used / grounded, computed and classified server-side before serialization
- [ ] Only `grounded` claims render inline markers; `used` renders collapsed; `retrieved`-only never renders to end users
- [ ] Citation events arrive strictly after `final_answer`, never interleaved with in-flight tokens
- [ ] `citationReducer` structurally independent of `tokenReducer` and `traceReducer`
- [ ] Failed faithfulness checks are flagged in place, never silently hidden or downgraded
- [ ] Every citation-triggered fetch re-enters `RAGEngine`'s authorization check, with no client-side caching exception
- [ ] `citation-access-control-test` passing as a CI-blocking security test
- [ ] Source-count displays audited to confirm they're computed post-authorization-filter
- [ ] Accessible (ARIA-associated) citation markers, not bare superscript numbers with no programmatic link to their content
- [ ] ADR written for claim-tiering design and the access-control boundary extension into the citation layer

## 17. Summary

Citations look like a display problem and are actually a trust-and-security problem wearing a display problem's clothes. The core discipline is the same one this handbook keeps returning to at every new layer: never let a client render a claim the backend hasn't structurally confirmed, and never let a security boundary that's correctly enforced in one component get silently reopened in another. This module split one undifferentiated "sources" list into three honestly distinct claims, gave a failed faithfulness check a visible and non-laundering treatment, and closed a real IDOR-shaped gap in the citation-fetch path by re-checking authorization on every single reference to a `chunk_id`, treating it as data, never as an implicit access token. `chat-shell` v3 now renders three independent live streams — tokens, agent trace, and citations — on one shared architecture, setting up the Part 7 capstone to assemble them, plus whatever comes next, into one coherent application.

## 18. Next Steps

Reply "continue" for Module 4, or flag anything to go deeper on.
