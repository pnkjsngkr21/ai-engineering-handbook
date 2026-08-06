# Part 7, Module 2 — Visualizing Agent Reasoning & Tool Calls

## 1. Learning Objectives

By the end of this module you will be able to:

1. Design an information hierarchy for an agent trace — decide what a user needs to *see*, what should be *available on demand*, and what should stay *server-side only* — and defend that decision with a reason, not a preference.
2. Model a tool call's execution as an explicit client-side state machine (`proposed → validated → approval_pending? → executing → succeeded|failed`) and render each state without inventing information the backend hasn't confirmed yet.
3. Build the handbook's first real human-in-the-loop UI: a live, in-context approve/reject control wired to `agent-core`'s `requires_approval` gate (Part 5, Module 1), replacing the CLI `input()` placeholder for good.
4. Render reflection critiques (Part 5, Module 2) with progressive disclosure — confidence-only reflection collapsed to a badge, plan-altering reflection expandable to the actual critique text — so the trace stays scannable under load.
5. Decide, with an explicit trust model, which of the four provenance tags (`human_verified` / `externally_sourced` / `agent_derived` / `peer_agent_derived`, Part 5 Modules 3–4) are shown to an end user versus an operator, and why those audiences need different things.
6. Extend `chat-shell` from a single-stream chat UI (Part 7, Module 1) into a UI that renders two structurally different kinds of streams at once — token stream and event stream — without one blocking or corrupting the other.
7. Verify, end-to-end, that an approval decision made in the UI actually reaches `agent-core`'s execution layer and gates the real tool call, not just a client-side visual state.

## 2. Prerequisites

- Part 7, Module 1 (Streaming Interfaces) — the partial-structured-output renderer and the SSE client you'll reuse here were built there.
- Part 5, Module 1 (Agent Fundamentals) — the agent loop, the `Action` type, and `requires_approval` as a structural execution-layer gate, not a prompt instruction.
- Part 5, Module 2 (Reflection) — the distinction between confidence-only and plan-altering reflection.
- Part 5, Modules 3–4 (Agent Memory, Multi-Agent Systems) — the four provenance values and why they exist.
- Comfortable with React state (`useReducer` is used heavily this module) and TypeScript discriminated unions.

## 3. Estimated Study Time

10–13 hours (theory + exercises: ~3 hours; mini-project: ~2 hours; production project: ~6–8 hours).

## 4. Difficulty

★★★★☆ (4/5) — Not conceptually harder than Module 1, but this module has more *simultaneous* moving state than anything else in Part 7 so far: a token stream, an event stream, a state machine per tool call, and a pending human decision, all live at once.

## 5. Why This Matters

Every agent you've built since Part 5 has been observable only through logs and `print()` statements. That's fine for development. It is not fine for production, and it's the reason "no real approval-gate frontend" was named as an explicit, honest limitation at the end of Part 5's capstone. A `requires_approval` action that only a CLI can approve is not actually deployable to a real user — it's a demo.

There's a sharper reason this matters beyond shipping a nicer UI. Right now, `requires_approval` is enforced structurally in `agent-core`'s execution layer (Part 5, Module 1) — the model cannot talk its way around it, because the gate lives in code, not in the prompt. That guarantee is worthless if the *frontend* silently auto-approves, or shows a stale action, or lets a double-click fire two approvals. The backend guard is only as trustworthy as the UI that exercises it. This module is where that guarantee either survives contact with a real interface or quietly breaks.

There's also a DoorDash-relevant transfer here, if less obvious than earlier modules: this is fundamentally an *event-sourced UI over an async backend process* problem — rendering a live-updating view of a long-running, multi-step server-side operation, with a user decision gating one step of it. That's the same shape as a delivery-tracking UI (order placed → assigned → picked up → in transit → delivered, with a customer able to intervene mid-flow), just recast in agent vocabulary. If this pattern makes sense to you by the end of the module, you've generalized a piece of system design, not just learned an AI UI trick.

## 6. Theory

### 6.1 What's actually worth surfacing — an information-design problem before it's a rendering problem

`agent-core`'s trace (Part 5, Module 1) already logs a `thought` → `action` → `observation` cycle per iteration, plus reflection critiques (Module 2) and provenance tags (Modules 3–4). It is tempting to render *all of it*, in order, as a scrolling log. Resist that. A raw trace is optimized for a developer debugging a failure after the fact. A user watching an agent work in real time needs something else: enough to trust that progress is happening and to intervene where it matters, without needing to read prose to find the one thing they're allowed to act on.

Think of this as three tiers, by default visibility:

- **Always visible:** the current high-level activity ("Searching knowledge base…", "Drafting summary…") and any `approval_pending` action. This is the minimum needed to answer "what is it doing right now, and does it need me."
- **Available on demand (collapsed by default):** the full thought/action/observation trace, reflection critiques, provenance tags. A user who wants to audit the reasoning can expand it; a user who just wants the answer never has to look at it.
- **Never sent to the client:** raw tool arguments containing secrets, internal retry/backoff detail, full memory-store contents. This is a security boundary, not a taste decision — it's the same principle as `RAGEngine`'s single public entry point (Part 4, Module 9): the client only ever sees what the public trace contract exposes, never the internals.

This maps directly onto how you'd design a REST response shape at DoorDash: you wouldn't return your entire order-processing state machine's internal fields to a mobile client; you'd return a curated `OrderStatus` DTO and keep the rest server-side. Same discipline, applied to an agent trace instead of an order.

The practical consequence: `agent-core` needs a **trace event contract** — a stable, versioned, deliberately reduced event shape emitted over the wire — that is *not* the same as its internal trace representation. Conflating the two means every internal refactor becomes a frontend breaking change. Keep them separate from the start.

```python
# agent-core: internal trace record (rich, for logs/eval, NEVER sent as-is)
@dataclass
class TraceStep:
    iteration: int
    thought: str
    action: Action | None
    observation: Observation | None
    reflection: ReflectionResult | None
    provenance_writes: list[ProvenanceRecord]
    latency_ms: int
    raw_tool_args: dict  # may contain secrets — never leaves the server

# agent-core: public trace EVENT contract (thin, versioned, sent over the wire)
class AgentEvent(TypedDict):
    v: Literal[1]
    type: Literal[
        "activity",            # human-readable one-liner for the "always visible" tier
        "tool_call_state",     # lifecycle transition for a specific tool call
        "reflection",          # confidence-only or plan-altering, pre-classified
        "provenance",          # a memory write's provenance tag, for trust display
        "final_answer",
    ]
    payload: dict
```

The `activity` string is generated server-side from the `thought`/`action`, deliberately — the model's raw chain-of-thought is not always something you want to show a user verbatim (it can be verbose, uncertain-sounding in ways that erode trust unnecessarily, or in rarer cases reveal information you don't want surfaced). Summarizing it into a short activity label at the point of emission keeps that decision server-side and auditable, rather than leaving the frontend to decide what to hide.

### 6.2 The tool call lifecycle as an explicit state machine

A tool call is not a single event — it's a sequence, and a UI that only renders "tool called" and "tool result" loses the states that actually matter to a watching user, especially the pending-approval one. Model it explicitly:

```
proposed → validated → [approval_pending → approved|rejected] → executing → succeeded|failed
```

- **proposed**: the model requested this call; arguments may still be streaming in (this is where Module 1's partial-structured-output renderer gets reused — a tool call's arguments are themselves a structured JSON object streamed token by token, and must obey the same rule: never render a field as complete until it structurally is).
- **validated**: arguments passed schema validation (Part 3, Module 2's parse→validate→repair pipeline). A call that fails validation goes to `failed` here, before ever reaching execution — this state existing at all is what lets a user see "the model tried something malformed" distinctly from "the tool itself failed."
- **approval_pending**: only reachable if `Action.requires_approval` is `True` (Part 5, Module 1). Execution is *blocked* server-side at this state — not just "waiting for the UI to look nice," but actually paused.
- **approved / rejected**: terminal for the decision, not for the call — approval leads into `executing`; rejection leads straight to `failed` with a distinct reason code (`user_rejected`), which matters for eval later (a rejection is not a system failure and must not be scored as one).
- **executing → succeeded/failed**: the actual tool call, same as always.

Each transition is a distinct `AgentEvent` of type `tool_call_state`, keyed by a stable `call_id`, so the client can always find the right in-flight card to update rather than guessing from ordering:

```typescript
type ToolCallState =
  | "proposed" | "validated"
  | "approval_pending" | "approved" | "rejected"
  | "executing" | "succeeded" | "failed";

interface ToolCallEvent {
  call_id: string;         // stable across the whole lifecycle
  tool_name: string;
  state: ToolCallState;
  args_partial?: string;   // raw partial JSON, only present during "proposed"
  args?: Record<string, unknown>; // only present once "validated"
  result?: unknown;        // only present at "succeeded"
  error?: { code: string; message: string }; // only at "failed"
}
```

Notice what's *not* here: no field is ever populated before its state guarantees it's real. This is the exact same discipline Module 1 established for partial JSON rendering, now applied one level up, to an entire multi-step process instead of a single object.

### 6.3 The approval gate — the module's real new interaction pattern

Everything up to here is "render state that changes over time," which Module 1 already established the pattern for. The approval gate is genuinely new: it's the first place in the handbook where the *user's action, taken mid-stream, changes what the backend does next*. Get the contract wrong here and you either create a security hole (UI decision doesn't actually gate anything) or a UX trap (user approves, nothing visibly happens, they approve again, and now you've double-executed a side-effecting action).

Three requirements, non-negotiable:

1. **The decision is a request, not a client-side state flip.** The "Approve" button does not set local state to "approved" and move on — it sends a request to `agent-core` (`POST /agent/{run_id}/approve/{call_id}`), and the UI stays in `approval_pending` until it receives back the server-confirmed `approved` event over the same event stream every other state transition comes through. This closes the same class of bug Module 1's cancellation-verification exercise caught: a UI can *look* correct while the backend disagrees.
2. **The control must be idempotent and single-use.** Once a decision is sent, disable the control immediately (optimistic UI is fine for *disabling*, never for *confirming*) and ignore any further clicks for that `call_id`. `agent-core`'s endpoint should independently reject a second decision on an already-decided `call_id` — defense in depth, not just a frontend nicety, because a flaky network can cause a legitimate retry from the client.
3. **A pause must survive a page refresh.** An agent run can sit in `approval_pending` for an arbitrary amount of real time — a user might close the tab and come back in ten minutes. On reconnect, the client must be able to ask "what's the current state of run X" and get back the *current* state (still `approval_pending`, or already resolved while they were away), not silently lose the pending decision. This is a direct callback to Part 5, Module 6's audited seam #5: every `requires_approval` pause must be a tested, valid resumption point. That was proven at the `agent-core` layer; this module proves it survives the UI layer too.

```typescript
async function submitApprovalDecision(
  runId: string,
  callId: string,
  decision: "approved" | "rejected"
) {
  // Disable immediately — but this is UX polish, not the source of truth
  dispatch({ type: "DECISION_SENT", callId });

  const res = await fetch(`/agent/${runId}/approve/${callId}`, {
    method: "POST",
    body: JSON.stringify({ decision }),
  });

  if (!res.ok) {
    // Re-enable — the request didn't land, the pending state is still real
    dispatch({ type: "DECISION_FAILED", callId });
    return;
  }
  // Do NOT dispatch "approved" here. Wait for the tool_call_state event
  // on the SSE stream — that's the only source of truth.
}
```

### 6.4 Rendering reflection — informative without being a wall of text

Part 5, Module 2 established two reflection modes: confidence-only (a scalar-ish signal, "I'm fairly sure this is right") and plan-altering (a critique that actually changes the next action). These need visually distinct treatment, because they carry different amounts of information and a user's attention is the scarce resource here.

- Confidence-only reflection: a small inline badge on the relevant step (e.g., a subtle dot or check), expandable to one line of text on hover/tap. It confirms reflection *happened* without demanding attention.
- Plan-altering reflection: a visible, collapsed-by-default card ("Reconsidered approach — tap to see why") directly after the step it affected, expandable to the actual critique text. This one earns screen space because it explains a visible change in the agent's behavior that would otherwise look erratic — the agent tried something, then visibly did something different, and a user deserves to know why without digging through logs.

The classification (`confidence_only` vs `plan_altering`) is already a first-class field on `ReflectionResult` from Part 5, Module 2 — this module doesn't invent new backend logic, it's the first place that classification actually earns its keep for an end user rather than just for `reflection-agreement-eval`.

### 6.5 Provenance as a trust signal — audience-dependent, not universal

The four provenance tags exist to protect `agent-core`'s memory system from poisoning (Part 5, Module 3) and correlated-failure false corroboration (Module 4). That's an internal integrity mechanism. Whether to show it to an end user is a separate design question, and the answer depends on audience:

- **End user, in the trace UI:** show a *reduced* signal, not the raw tag. `human_verified` and `externally_sourced` facts get a small "verified" indicator; `agent_derived` and `peer_agent_derived` get no special marking (they're the default, unflagged state) — because a raw tag like `peer_agent_derived` is internal jargon that means nothing to a user and, worse, could be misread as some kind of endorsement. The design principle: provenance should let a user trust *more*, in specific, earned cases — it should never look like a system disclaimer shifting blame for an unverified claim, because that erodes trust in everything, including the parts that *are* verified.
- **Operator, in a debugging/eval view (out of scope for this module, flagged for Part 7's later operator dashboard):** show the raw tag, always, on every memory read/write, because an operator's job is exactly to audit provenance integrity.

This is the same audience-split discipline as Part 4, Module 5's access-control work: the same underlying fact (a provenance tag) is a *different* kind of information depending on who's looking at it, and the UI layer is where that gets enforced, not assumed.

## 7. Mental Models

- **"The trace is a debugging tool wearing a user-facing costume — don't let it forget which one it is."** Default to the reduced view; the full trace is one click away, not the landing state.
- **"A pending approval is a paused server, not a paused UI."** The button disabling is cosmetic; the state that matters lives behind the fetch call and is only confirmed by the event stream.
- **"Never render a field before its state guarantees it's real"** — Module 1's rule, now applied one layer up: to tool call lifecycles and approval decisions, not just to streamed JSON fields.
- **"Provenance protects the agent internally; trust signals protect the user's attention externally"** — same tag, two different jobs, two different renderings.

## 8. Visual Explanation

**Tool call lifecycle, as rendered state:**

```
 [proposed]──validate──▶[validated]
                              │
              requires_approval?
                 │yes              │no
                 ▼                 │
        [approval_pending]         │
           │approve   │reject      │
           ▼          ▼            │
      [executing]  [failed:        │
           │        user_rejected] │
     succeed│fail        ◀─────────┘
           ▼    ▼
     [succeeded][failed]
```

**Trace panel information tiers (default collapsed state):**

```
┌─ Chat message (always visible) ─────────────────────────┐
│  "Searching your knowledge base…"                        │  ← activity, tier 1
│                                                            │
│  ⚠ Needs your approval: Send email to finance@acme.com   │  ← approval_pending, tier 1
│    [ Approve ]  [ Reject ]                                │
│                                                            │
│  ▸ Show reasoning (4 steps, 1 reconsideration)            │  ← tier 2, collapsed
└────────────────────────────────────────────────────────────┘
                          │ tap "Show reasoning"
                          ▼
┌─ Expanded trace ───────────────────────────────────────────┐
│  1. Thought: need last quarter's revenue figures           │
│     → tool: rag_search   ✓ succeeded                       │
│  2. Thought: figures found, drafting summary                │
│     ⟳ Reconsidered approach — tap to see why                │
│  3. Thought: summary ready, drafting email                  │
│     → tool: send_email   ⏸ awaiting your approval           │
└────────────────────────────────────────────────────────────┘
```

**Two independent live streams, one message:**

```
   token stream (Module 1's renderer)     event stream (this module)
        │                                        │
        ▼                                        ▼
   "Based on the Q3..."               tool_call_state: rag_search → executing
        │                                        │
        ▼                                        ▼
  useReducer(tokenReducer)            useReducer(traceReducer)
        │                                        │
        └──────────────┬─────────────────────────┘
                        ▼
              single <ChatMessage> render,
              two independent slices of state
```

## 9. Recommended Resources

1. **Anthropic — "Building effective agents"** (engineering blog) — revisit specifically for how Anthropic frames human-in-the-loop checkpoints; useful cross-check against the approval-gate design in §6.3. You've read this before in Part 5; it's worth a second pass now that you're building the interaction it describes.
2. **React docs — `useReducer`** (react.dev) — this module leans on reducers more than any prior one, specifically for keeping the token-stream reducer and trace-event reducer provably independent. Read the "extracting state logic into a reducer" guide if you haven't used `useReducer` for genuinely concurrent state before.
3. **W3C WAI-ARIA Authoring Practices — "Alert and Message Dialogs"** — the approval gate is functionally an inline confirmation dialog; this is the authoritative reference for making it keyboard-accessible and screen-reader-announced correctly, which matters more here than in a typical chat UI because the user is being asked to *authorize a real action*, not just read text.
4. **Kleppmann, *Designing Data-Intensive Applications*, Ch. 11 (Stream Processing)** — you've used this book since Part 1; the "state machine over an event log" framing in this chapter is precisely the model this module's `tool_call_state` events implement. Re-read the section on idempotent consumers before building §6.3's approval endpoint.
5. **MDN — `EventSource` and reconnection behavior** — needed for the resumption requirement in §6.3.3; specifically the `Last-Event-ID` header, which you'll use to let a reconnecting client ask "what did I miss" rather than replaying the whole trace from scratch.

## 10. Exercises

1. Design the `AgentEvent` contract's version-negotiation strategy: if `agent-core` starts emitting `v: 2` events with a new `type`, what should a `chat-shell` client still on `v: 1` rendering logic do? Write the fallback rule.
2. Take the tool call lifecycle diagram in §8 and add one more terminal state: `timed_out` (approval never received within some window). Where does it fit in the state machine, and what should `agent-core` do with the underlying action when it happens?
3. Write the reducer action types (as a TypeScript discriminated union) for the trace-event reducer, handling all five `AgentEvent` types from §6.1.
4. For the provenance-display rule in §6.5, write out the exact mapping from the four raw tags to the two end-user-facing states (`verified` / unmarked). Justify why `peer_agent_derived` is unmarked rather than getting its own visual treatment.
5. A user rejects a `send_email` tool call. Sketch (as an `AgentEvent` sequence) what `agent-core` should do next in the loop — does it just stop, or does it need to feed the rejection back to the model as an observation so it can adapt? Justify your answer against Part 5, Module 1's loop design.

## 11. Mini-Project

Build a standalone React component, `<ToolCallCard>`, that takes a stream of mock `ToolCallEvent` objects (hardcoded array, played back on a timer — no real backend yet) and renders the full lifecycle from §6.2 and §8, including a working (locally-simulated) approve/reject control for calls with `requires_approval: true`. This isolates the state machine and the approval UX from the harder problem of wiring it to a real dual-stream backend, which the Production Project tackles next.

Acceptance bar: feed it a rejected-call sequence and a succeeded-call sequence back to back; both must render correctly with no stale state bleeding from one card to the next.

## 12. Production Project: `chat-shell` v2 — Agent Trace UI

Extend `chat-shell` (Part 0, Module 8 → real v1 in Part 7, Module 1) into `chat-shell` v2, wired to `agent-core` (Part 5, capstone, v2.0) instead of the plain `convo-api`/`llm-client-core` chat endpoint.

**Scope:**

1. **`AgentEvent` contract**, implemented server-side in `agent-core` as described in §6.1 — a genuinely new, versioned, reduced event shape, not a repackaging of the internal `TraceStep`. Emitted over the same SSE connection Module 1 already established (extend `streamcat`'s client, don't replace it), multiplexed with the token stream using a `type` discriminator on each SSE message.
2. **Dual reducer architecture**: a `tokenReducer` (from Module 1, unchanged) and a new `traceReducer`, both fed by the same SSE `onmessage` handler, dispatching to whichever reducer matches the message's `type`. Proves the two streams are independently correct — build the stress test from Exercise verification below to confirm neither reducer can corrupt the other's state under interleaved, adversarially-ordered messages.
3. **`<ToolCallCard>`** from the Mini-Project, now wired to real events and a real approval endpoint per §6.3, including the disable-on-submit, ignore-duplicate-decisions, and reconnect-resumption behavior.
4. **`<ReflectionBadge>` / `<ReflectionCard>`** per §6.4's two-tier treatment.
5. **Provenance display** per §6.5's reduced, end-user-facing mapping only (no raw-tag operator view yet — that's explicitly future work).
6. **Approval endpoint in `agent-core`**: `POST /agent/{run_id}/approve/{call_id}`, idempotent, rejecting a second decision on an already-decided call, and a `GET /agent/{run_id}/state` endpoint for reconnect resumption.
7. **`agent-trace-render-stress-test`**: extends Module 1's `streaming-render-stress-test` philosophy — feeds deliberately adversarial event orderings (a `tool_call_state: succeeded` arriving before its `validated`, duplicate events, an approval decision response racing the SSE `approved` event) and asserts the UI never shows an impossible or stale state. This becomes a standing regression tool the same way every prior part's eval/bench tool has.
8. **End-to-end approval verification test**: proves a `reject` decision sent from the UI actually prevents `agent-core` from executing the underlying tool call — not just that the UI stops showing a spinner. This is the module's single most important test, directly closing the "no real approval-gate frontend" limitation named in Part 5's capstone.

**Documentation** (extending `chat-shell`'s existing docs from Module 1): an ADR for the decision to keep `AgentEvent` as a separate contract from `agent-core`'s internal `TraceStep`, and a short "what a user sees vs. what an operator would need" note explicitly flagging the operator dashboard as future work — this hands off directly to the operator-facing SLO dashboard named as a Part 6, Module 8 limitation and slated later in Part 7.

**Hands off to:** RAG citation/source rendering (next in Part 7), which will reuse this module's `type`-discriminated SSE multiplexing pattern for a third stream (citation events) rather than inventing a new transport; and the future operator dashboard, which will reuse `AgentEvent`'s `provenance` events but render them with the raw-tag mapping this module explicitly deferred.

## 13. Practice & Interview Questions

1. Why does the tool call lifecycle need a `validated` state distinct from `proposed`, given that both happen before execution and neither has run the tool yet?
2. Explain why the approval "Approve" button in §6.3 should not set local state to `approved` directly. What specific bug does waiting for the server-confirmed event prevent?
3. A trace UI needs to survive a page refresh mid-approval-pending. Design the minimal API surface needed for that, and explain why `Last-Event-ID`-based SSE reconnection alone isn't sufficient — what does it not tell you that a fresh state fetch would?
4. Why is it a security concern, not just a UX concern, for `agent-core`'s internal `TraceStep` (with `raw_tool_args`) to be serialized directly to the client instead of going through a reduced `AgentEvent` contract?
5. Design a rate-limit/idempotency story for the approval endpoint that protects against both a slow double-click and a genuine client retry after a dropped network response, without letting a legitimate retry get silently swallowed.
6. In a system design interview: you're asked to design a live UI for a long-running, multi-step backend process where a user can intervene mid-flow (this module's exact shape, recast generically — think order tracking, or a CI pipeline UI with a manual approval gate). Walk through the event contract, the state machine, and the reconnect story, in that order, and explain why that order matters.
7. Why does provenance need two different renderings for two different audiences, rather than one raw-tag display shown to everyone?

## 14. Common Mistakes

- **Streaming the internal trace representation directly to the client.** Couples the UI to internal refactors and risks leaking `raw_tool_args`. Always go through a versioned, reduced event contract.
- **Treating "Approve" as a client-side state transition.** Creates a class of bug where the UI shows approved but the backend never received or acted on the decision — exactly the failure mode Module 1's cancellation exercise was built to catch, recurring here in a more consequential form (this time gating a real side-effecting action, not just a display).
- **No idempotency on the approval endpoint.** A double-click, a retried request after a timeout, or a reconnect-and-resubmit can trigger the gated action twice if the endpoint isn't defensively idempotent.
- **Rendering the full raw trace by default.** Buries the one thing a user actually needs to act on (a pending approval) under prose they didn't ask to read. Default collapsed; earn the expansion.
- **Showing raw provenance tags to end users.** `agent_derived` reads as a disclaimer and erodes trust in the whole trace, including the genuinely verified parts — collapse to a two-state signal instead.
- **Losing pending-approval state on reconnect.** If a refresh mid-`approval_pending` can't recover the current state, you've silently broken the exact resumption guarantee Part 5, Module 6 already proved at the backend layer — don't let the frontend regress it.
- **One reducer trying to handle both token and trace events.** Conflating the two makes it much easier for a race between them to corrupt shared state; keep them structurally independent, as in §12.2.

## 15. Debugging Exercise

You wire up `<ToolCallCard>` against the real `agent-core` backend. In manual testing, most of the time it works. But occasionally, for a `requires_approval` tool call, the "Approve" button flashes briefly enabled again *after* being clicked, and — much more rarely — the underlying tool call appears to execute twice in the backend logs for a single approval.

Given everything in §6.3, form a hypothesis before reading further:

<details>
<summary>Hint 1</summary>
Look at the sequence of events again: `DECISION_SENT` is dispatched immediately (optimistic disabling), then the fetch resolves, then separately the SSE stream delivers the real `approved` event. What happens if the fetch response comes back successfully but the SSE `approved` event is delayed or arrives out of order relative to something else?
</details>

<details>
<summary>Hint 2</summary>
Check what `DECISION_SENT` actually disables — is it disabling based on `call_id`, or based on some other key that might get reset if the component re-renders for an unrelated reason (e.g., a new token arriving on the *token* reducer)?
</details>

<details>
<summary>Likely root cause</summary>
The button's disabled state was derived from local component state (a `useState` boolean scoped to the `<ToolCallCard>` instance) rather than from the `call_id`'s state in the shared `traceReducer`. When the parent chat message re-renders for an unrelated reason (a new streamed token elsewhere in the same message), React can remount or reset that local `useState`, un-disabling the button while the server-side decision is genuinely still in flight — and a second click sends a second `POST /agent/{run_id}/approve/{call_id}`. Because the approval endpoint wasn't independently idempotent per §12.6/Common Mistake #3, the second request executed the tool again. Fix: disabled state must be derived from `traceReducer`'s state for that `call_id` (single source of truth, survives unrelated re-renders), *and* the backend endpoint must independently reject a second decision on an already-decided `call_id` regardless of what the frontend does — defense in depth, exactly as specified in §6.3.
</details>

## 16. Checklist

- [ ] `AgentEvent` contract defined, versioned, and kept structurally separate from `agent-core`'s internal `TraceStep`
- [ ] Tool call lifecycle rendered as the full explicit state machine from §6.2/§8, no field shown before its state guarantees it
- [ ] Approval control sends a real request and waits for server-confirmed SSE event before showing "approved" — never flips state optimistically
- [ ] Approval endpoint is idempotent against duplicate decisions on the same `call_id`
- [ ] Reconnect/refresh mid-`approval_pending` recovers correct current state via a dedicated state-fetch endpoint
- [ ] Reflection rendered with two-tier treatment (confidence-only badge vs. plan-altering expandable card)
- [ ] Provenance shown to end users only as the reduced two-state signal, never as raw tags
- [ ] Token reducer and trace reducer verified structurally independent under adversarial interleaving
- [ ] `agent-trace-render-stress-test` passing against adversarial event orderings
- [ ] End-to-end test proves a UI rejection actually prevents backend execution of the gated tool call
- [ ] ADR written for the `AgentEvent`/`TraceStep` separation decision

## 17. Summary

An agent trace UI is not "the chat UI, but with more logs shown." It's a three-tier information-design problem (always visible / on-demand / never sent) layered on top of two genuinely new engineering problems: a tool call is a multi-state lifecycle that must never render a field before its state guarantees it's real, and — for the first time in the handbook — a UI action can change what the backend does next, which means an approval decision must be treated as a real request confirmed by the server, never a local state flip. Reflection and provenance both already existed as backend concepts (Part 5); this module's contribution was deciding, deliberately, what of each is worth a user's attention and what would just erode trust or bury the one thing they need to act on. `chat-shell` v2 closes the "no real approval-gate frontend" limitation named at the end of Part 5's capstone, and does it with the same discipline as every capstone before it: a stable versioned contract, an explicit state machine, and a stress test that tries to break it before a real user does.

## 18. Next Steps

Reply "continue" for Module 3, or flag anything to go deeper on.
