# Part 7, Module 1: Streaming Interfaces — From Token Stream to Rendered UI

> Part 7 begins here — extending `chat-shell` (Part 0, Module 8) from a
> static skeleton into a real, working streaming interface wired to
> `convo-api`/`llm-client-core`. This module is the frontend-side
> completion of Part 3, Module 9's "streaming reduces perceived latency,
> not total latency" argument: it's one thing to know that claim, and
> another to correctly build the UI that actually delivers on it.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain why rendering a token stream correctly is a genuinely
   different UI problem than rendering a completed response, and name
   the two specific rendering hazards (malformed partial Markdown,
   invalid partial JSON) that a naive implementation hits immediately.
2. Implement incremental Markdown rendering that stays visually stable
   as new tokens arrive, without flickering or rendering broken
   intermediate syntax.
3. Implement safe rendering of **partial structured output** (Part 3,
   Module 2) — showing a tool call or structured response's fields as
   they resolve, without ever rendering invalid JSON or a broken UI
   state.
4. Implement request cancellation correctly — client-initiated stream
   abort that actually stops server-side generation (and therefore
   cost), not just stops updating the UI while the backend keeps
   generating.
5. Connect `chat-shell` to a real backend (`convo-api`, `llm-client-core`,
   optionally routed through `llm-gateway` from Part 6) via Server-Sent
   Events, reusing `streamcat`'s (Part 0, Module 4) SSE-handling logic
   rather than rebuilding it in the frontend.
6. Ship a working, streaming `chat-shell` v1 that correctly renders
   token-by-token text, partial structured output, and supports
   cancellation — the base UI every later Part 7 module builds on.

## 2. Prerequisites

- Part 0, Module 8 (React & Next.js Essentials, `chat-shell`) — this
  module turns that skeleton into a real application.
- Part 0, Module 4 (Networking, HTTP, REST & JSON, `streamcat`) — the
  SSE client logic this module reuses rather than reimplementing.
- Part 3, Module 9 (Latency & Cost Optimization) — the perceived-latency
  argument this module implements the frontend half of.
- Part 3, Module 2 (Structured Outputs) — needed to understand what
  "partial structured output" actually looks like mid-stream.

## 3. Estimated Study Time

9–11 hours across 2 sessions.

## 4. Difficulty

★★★☆☆ (3/5) — No new backend concepts; the difficulty is entirely in
correctly handling the specific edge cases of incremental rendering,
which are easy to get subtly wrong even with solid React fundamentals.

---

## 5. Why This Matters

Part 3, Module 9 established that streaming reduces *perceived*
latency, not total latency — the user sees output start appearing
immediately, even though the full response takes the same total time.
That claim is only true if the frontend renders the stream well. A
frontend that buffers tokens and re-renders the whole response every
few hundred milliseconds, or that flickers and re-flows text as
Markdown syntax completes, or that shows a broken half-rendered JSON
object while a tool call resolves, actively works against the entire
reason streaming exists — the user experiences something that feels
worse than a single clean update, even though technically "more
information arrived sooner." This module is where the backend
optimization work from Part 3 either pays off or gets undermined by the
UI layer, and getting this right is a genuinely underrated skill: most
engineers who've built a chat UI have hit the Markdown-flicker or
broken-JSON problem and shipped a workaround that's fragile rather than
a principled fix.

This also matters because every later Part 7 module — tool-call
visualization, the agent approval-gate UI (finally replacing Part 5,
Module 1's CLI placeholder), RAG citation rendering, a voice interface
— is built on top of the same incremental-rendering foundation this
module establishes. Getting the base streaming UI right once, here,
means every later module extends a solid foundation instead of
patching around the same rendering bugs repeatedly.

---

## 6. Theory

### 6.1 Why rendering a stream is a different problem than rendering a result

A completed LLM response is just a string (or a validated JSON object)
— rendering it is a solved problem (Markdown-to-HTML, or displaying
structured fields). A response *in progress* is neither: it's a
partial string that may contain **incomplete Markdown syntax**
(an opening `**` with no closing `**` yet, an unclosed code fence, a
list item with the next item not yet arrived) or, for structured
output (Part 3, Module 2), **invalid partial JSON** (an object with an
unclosed brace, a string value cut off mid-character). Naively feeding
either of these directly into a Markdown renderer or a `JSON.parse`
call will either throw an error or render visibly broken output —
this is not a hypothetical edge case, it is the default, guaranteed
state of the data for every single token except the last one in the
stream.

### 6.2 Incremental Markdown rendering without flicker or breakage

The correct approach is not "try to parse the partial Markdown as-is"
but **render defensively against incompleteness**: use a Markdown
renderer (or a thin wrapper around one) that's specifically designed to
handle streaming/incomplete input — closing any currently-open syntax
elements (bold, italic, code fences, links) with their matching closer
*for rendering purposes only*, without altering the underlying
accumulated text buffer, so that as soon as the real closing syntax
arrives in a later token, rendering continues seamlessly from the
correct state. This is the same idea as a text editor's "auto-close
brackets" feature, applied to rendering instead of editing: the
renderer's *display* stays syntactically valid at every intermediate
step, even though the underlying accumulated text is, correctly,
incomplete until the stream finishes.

The second, related discipline: **don't re-render the entire response
from scratch on every token.** Append-and-re-render-only-the-new-part
(using React's keyed reconciliation correctly, rather than replacing
the whole content tree on each update) is what actually prevents visual
flicker — a naive `setState` that replaces the full rendered output on
every token can cause the browser to re-layout and potentially lose
scroll position or cause visible flashing, especially for longer
responses. This is a real, measurable UI performance concern, not just
an aesthetic preference, and it's worth profiling (using your browser's
own performance tools) rather than assuming your implementation doesn't
have this problem.

### 6.3 Partial structured output: never render invalid state

Rendering a tool call or structured response's fields as they resolve
(e.g., showing "searching for: [query being typed out]..." as a
`search` tool call's arguments stream in) needs a stricter discipline
than Markdown, because the failure mode is worse: attempting
`JSON.parse` on a partial JSON string will throw, and building UI logic
that reacts to a thrown exception on every intermediate token is both
fragile and wasteful. The correct approach: use an incremental/
streaming JSON parser (one designed specifically to accept partial
input and report which fields are confirmed-complete versus still
in-progress, rather than an all-or-nothing parser) and render only the
fields the parser reports as complete, with a clear, deliberate loading
indicator for fields still in progress — never attempting to display a
guessed or truncated value for an incomplete field, which risks showing
the user information that will change before the stream finishes (e.g.,
showing a partial dollar amount that then changes as more digits
stream in, which is actively misleading, not just visually rough).

This is the direct frontend instance of Part 3, Module 2's core
lesson — structured output needs validation before you trust it —
applied at the token-stream granularity instead of the whole-response
granularity: validate what you have so far, and only render the parts
that are actually valid, discarding nothing, but never presenting
unvalidated partial state as if it were reliable.

### 6.4 Cancellation must actually stop generation, not just stop rendering

A "stop" button that only stops the frontend from processing further
incoming stream events, while the backend keeps generating tokens
(and the calling application keeps paying for them, per Part 3, Module
9's cost discipline), is a UI-only fix for what's actually a
full-stack problem. Correct cancellation requires the frontend's abort
signal (e.g., aborting the underlying `fetch`/`EventSource` connection)
to propagate all the way back through `convo-api`/`llm-client-core` to
the actual generation call, so the upstream provider (hosted API,
`VLLMAdapter`, or `llm-gateway`) genuinely stops generating and stops
being billed for tokens the user will never see. Verify this
end-to-end, not just at the UI layer — an abort button that "looks
like it worked" because the UI stopped updating, while server logs show
generation continued to completion, is a real, costly bug this module
exists to catch before it ships.

---

## 7. Mental Models

1. **"A response in progress is guaranteed to be structurally incomplete
   at every step but the last — render defensively against that, don't
   treat it as an edge case."**
2. **"Close open syntax for display purposes only; never touch the
   underlying accumulated buffer — that's what keeps rendering seamless
   when the real closer arrives."**
3. **"Only ever render a structured field once a streaming parser
   confirms it's complete — a guessed partial value is actively
   misleading, not just unfinished."**
4. **"Cancellation that doesn't reach the actual generation call is a
   UI illusion, not a fix — verify the abort reaches the provider."**

---

## 8. Visual Explanation

```
   TOKEN STREAM ARRIVING:  "Here's **bold text tha..."
                                          │
                                          ▼
   NAIVE RENDER (broken):  Here's <b>bold text tha...
                            (unclosed <b> leaks into
                             surrounding text/DOM)

   DEFENSIVE RENDER (correct):
   display buffer:  "Here's **bold text tha"
   for RENDERING only, auto-close: "Here's **bold text tha**"
                                    └── closer added for display,
                                        underlying buffer unchanged
   next token arrives: "...t keeps going**"
   underlying buffer: "Here's **bold text that keeps going**"
   render: seamlessly continues, no flicker, no broken state

   ──────────────────────────────────────────────────

   PARTIAL STRUCTURED OUTPUT (tool call arguments streaming):

   raw partial JSON: {"query": "weather in Sa
                       └── INVALID JSON, JSON.parse() throws

   streaming parser reports:
     query: IN PROGRESS -> show loading indicator, NOT a guessed value
   ...next chunk...
   raw partial JSON: {"query": "weather in San Francisco", "units":
     query: COMPLETE -> "weather in San Francisco"  (render now, safe)
     units: IN PROGRESS -> loading indicator

   ──────────────────────────────────────────────────

   CANCELLATION -- must reach the actual generation call:

   [Stop button] → abort signal → fetch/EventSource aborted
        │
        ▼ MUST propagate, not stop at the UI layer
   convo-api → llm-client-core adapter → actual provider call
        │
        ▼
   provider genuinely stops generating (verified in provider logs/cost)
```

---

## 9. Recommended Resources

1. **MDN — Server-Sent Events / `EventSource` documentation** — the
   authoritative reference for the streaming transport this module
   builds on, reused directly from Part 0, Module 4's `streamcat`.
2. **react-markdown / streaming-markdown libraries' documentation** —
   practical, current reference for incremental Markdown rendering
   approaches; evaluate at least one purpose-built streaming renderer
   rather than hand-rolling Markdown-completion logic from scratch.
3. **A streaming/incremental JSON parser library's documentation**
   (e.g., a partial-JSON-parsing package for JavaScript) — the
   practical tool behind Section 6.3's discipline; don't hand-roll a
   partial JSON parser unless you have a specific reason to.
4. **Your own Part 0, Module 4 (`streamcat`) code** — re-read before
   wiring `chat-shell` to a real backend; the SSE client-handling logic
   is meant to be reused, not rebuilt.

---

## 10. Exercises

1. Wire `chat-shell` to a real, running `convo-api`/`llm-client-core`
   backend via SSE, and confirm you can see raw tokens arriving in the
   browser's network tab before building any rendering logic on top.
2. Implement naive Markdown rendering (re-render the whole buffer on
   every token, feed it directly to a standard, non-streaming-aware
   Markdown renderer) and deliberately observe the flicker/breakage
   this module warns about — seeing the failure firsthand makes the
   fix's necessity concrete.
3. Replace it with defensive, streaming-aware Markdown rendering
   (Section 6.2) and confirm the flicker/breakage is gone.
4. Implement partial structured-output rendering (Section 6.3) for a
   simple tool call, using a streaming JSON parser, and confirm no
   `JSON.parse` exception is ever thrown during rendering, even with
   deliberately chunked, slow-arriving partial JSON.
5. Implement the Stop button, then verify cancellation end-to-end: add
   temporary logging at the actual provider-call layer (`llm-client-core`)
   and confirm that clicking Stop causes that log to show the call was
   genuinely aborted, not just that the UI stopped updating.

---

## 11. Mini-Project

**`streaming-render-stress-test`**: a small test harness that feeds
your rendering components a recorded, deliberately-chunked token
stream (varying chunk sizes, including single-character chunks) for
both a Markdown-heavy response and a structured tool-call response, and
asserts that the rendered output never shows broken syntax, never
throws a parsing exception, and matches the correct final rendered
state once the stream completes — regardless of how the stream was
chunked. This is meant to catch rendering bugs that only appear under
specific, unlucky chunk-boundary conditions that manual testing is
unlikely to hit reliably.

---

## 12. Production Project: `chat-shell` v1 (streaming, production-connected)

### Scope

Upgrade `chat-shell` (Part 0, Module 8) from a UI skeleton into a real,
working application:

- Connected to `convo-api`/`llm-client-core` via SSE, reusing
  `streamcat`'s client-side stream-handling logic.
- Defensive, streaming-aware Markdown rendering (Section 6.2), with no
  full-buffer re-render on each token — verified via browser
  performance profiling, not just visual inspection.
- Partial structured-output rendering (Section 6.3) for at least one
  real tool call, using a streaming JSON parser, never displaying
  unvalidated partial field values.
- End-to-end-verified cancellation (Section 6.4), with the verification
  test from Exercise 5 kept as a permanent regression check, not a
  one-time manual confirmation.
- `streaming-render-stress-test` (Mini-Project) included as a standing
  test suite, run against real recorded streams from your own backend,
  not synthetic best-case data.

### Explicit extension point

**Part 7, Module 2** will build tool-call and agent-reasoning
visualization directly on top of this module's partial-structured-
output rendering foundation. **Part 7's later approval-gate UI module**
replaces Part 5, Module 1's CLI-based human-approval placeholder with a
real interface, built on this module's streaming and cancellation
infrastructure (an approval prompt is, itself, effectively a paused
stream awaiting a decision). **Part 6, Module 8's** phase-specific SLO
dashboards (flagged as a limitation there) will be surfaced through
this same `chat-shell` foundation in a later operator-facing module.

---

## 13. Practice & Interview Questions

1. Why is rendering a token stream a genuinely different problem than
   rendering a completed response, not just a smaller version of the
   same problem?
2. Describe the "close open syntax for display only" technique for
   streaming Markdown rendering, and explain why it doesn't corrupt the
   underlying accumulated text.
3. Why is attempting `JSON.parse` directly on partial streaming JSON
   the wrong approach, and what should you use instead?
4. Why might a chat UI's "Stop" button fail to actually save cost, even
   though the UI itself stops visibly updating when clicked?
5. How would you test that your streaming UI renders correctly
   regardless of how the underlying stream happens to be chunked?

---

## 14. Common Mistakes

- **Feeding partial Markdown or partial JSON directly into a
  non-streaming-aware parser/renderer** — guaranteed to throw errors or
  render visibly broken output, since incompleteness is the normal
  state, not an edge case.
- **Re-rendering the entire response buffer on every token** — causes
  visible flicker and real, measurable UI performance cost, especially
  for longer responses.
- **Rendering a guessed or truncated value for an in-progress
  structured field** — actively misleading, since the value can and
  will change before the stream completes.
- **Building a "Stop" button that only stops frontend rendering** —
  looks correct in the UI while the backend keeps generating and
  billing, a costly bug that's easy to miss without explicit end-to-end
  verification.
- **Testing streaming UI only against convenient, evenly-chunked mock
  data** — missing bugs that only appear at specific, unlucky
  chunk-boundary conditions real network streaming can produce.

---

## 15. Debugging Exercise

QA reports that your chat UI occasionally shows a brief flash of raw,
unrendered Markdown syntax (literal asterisks or backticks visible
in the UI) partway through a response, which then "fixes itself" a
moment later as more tokens arrive.

Using Section 6.2, walk through: (a) the most likely cause (hint: this
is exactly the signature of failing to auto-close open syntax for
display purposes — the renderer is briefly showing the raw, unclosed
markup because nothing closed it for rendering), (b) the specific test
from `streaming-render-stress-test`, using an unlucky chunk boundary
that splits syntax markers across chunks, that should have caught this
before it reached QA, and (c) the fix, stated precisely as a change to
the display-only auto-closing logic, not as a general "improve the
Markdown renderer" note.

---

## 16. Checklist

- [ ] I can explain why partial Markdown and partial JSON are the
      *normal* state of streaming data, not an edge case to handle
      separately.
- [ ] My Markdown rendering auto-closes open syntax for display only,
      without altering the underlying accumulated buffer.
- [ ] My rendering doesn't re-render the full buffer from scratch on
      every token, verified via profiling, not just visual inspection.
- [ ] My structured-output rendering uses a streaming JSON parser and
      never displays a field before it's confirmed complete.
- [ ] I've verified cancellation end-to-end, confirming the abort
      actually reaches the provider call, not just the UI layer.
- [ ] `streaming-render-stress-test` passes against real, variably-
      chunked recorded streams, not just synthetic even-chunked data.
- [ ] `chat-shell` v1 is genuinely connected to a real backend, not
      running against mocked responses.

---

## 17. Summary

Rendering a stream correctly is a distinct problem from rendering a
completed response, because incompleteness is the guaranteed state of
the data at every step but the last. The fix for Markdown is
display-only auto-closing of open syntax, leaving the underlying
buffer untouched; the fix for structured output is a streaming JSON
parser that only surfaces fields once they're confirmed complete, never
a guessed partial value. Cancellation only counts if it's verified to
reach the actual generation call, not just the UI. Getting all of this
right in `chat-shell` v1 is what actually delivers on Part 3, Module
9's perceived-latency argument at the UI layer, and it's the
foundation every later Part 7 module — tool-call visualization, the
real approval-gate UI, RAG citations, voice — builds on directly.

---

## 18. Next Steps

Next: **Part 7, Module 2 — Visualizing Agent Reasoning & Tool Calls**,
building a real UI for `agent-core`'s scratch-pad, reflection critiques,
and tool-call execution (Part 5) on top of this module's partial-
structured-output rendering foundation — turning an agent's internal
loop from something visible only in logs into something a user can
actually watch and understand as it happens.

Reply "continue" for Module 2, or flag anything to go deeper on.
