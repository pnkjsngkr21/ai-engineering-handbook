# Part 7, Module 5 — A Voice Interface for the Voice Agent

## 1. Learning Objectives

By the end of this module you will be able to:

1. Explain why voice is not "the chat UI, but audio instead of text" — identify the specific latency, turn-taking, and interruption constraints that don't exist in any text-based surface built so far in Part 7.
2. Apply Part 5, Module 5's key finding (voice's one genuinely new constraint is latency, which changes *when* guards run, not *whether* they run) to the frontend: design a client that lets safety/approval checks run asynchronously after a spoken response without either blocking speech or silently skipping a check.
3. Design a turn-taking state machine (listening → thinking → speaking → interrupted) distinct from, but coexisting with, the tool-call and approval state machines from Module 2.
4. Handle barge-in (a user interrupting the agent mid-speech) as a first-class event that must cleanly cancel in-flight generation and TTS playback, reusing Module 1's verified-cancellation discipline rather than inventing new cancellation logic.
5. Design a voice-appropriate approval gate for `requires_approval` actions, given that the visual approve/reject buttons from Module 2 don't exist in an audio-only context, and that reading out a full action's arguments aloud is often worse UX than a visual confirmation.
6. Decide what, if anything, gets rendered visually alongside a voice interaction (a "visual companion" view), versus what should remain purely audio, as a deliberate accessibility and dual-modality decision rather than a default.
7. Extend `ai-infra-stack`'s and `chat-shell`'s existing streaming/event infrastructure to a real-time audio pipeline without duplicating the reducer/event architecture already proven in Modules 1–4.

## 2. Prerequisites

- Part 7, Modules 1–4 — the SSE/reducer architecture, tool-call lifecycle, and approval-gate pattern are all reused here, adapted to audio's stricter latency requirements.
- Part 5, Module 5 (Specialized Agents) — specifically the voice-agent section: latency as the one new constraint, changing guard *timing*, not guard *existence*.
- Part 6, Module 1 (GPU Fundamentals) — prefill/decode latency profile; voice's round-trip budget makes this concrete rather than abstract.
- Part 3, Module 9 (Latency & Cost Optimization) — streaming for perceived latency, directly relevant to text-to-speech chunking.

## 3. Estimated Study Time

11–14 hours (theory + exercises: ~3 hours; mini-project: ~2.5 hours; production project: ~6–8.5 hours).

## 4. Difficulty

★★★★☆ (4/5) — Comparable to Module 2's complexity, but the real-time audio constraint (sub-second round-trip expectations, barge-in) makes the margin for a sloppy cancellation or state-machine bug much less forgiving than in a text UI, where a half-second of extra latency is invisible and a race condition might just look like a flicker.

## 5. Why This Matters

Part 5, Module 5 made a precise claim about voice agents: there's no new safety *mechanism* needed for voice, only a recalibration of *when* existing guards run, because latency changes what's tolerable before a spoken response starts. That claim was proven at the agent-loop layer. It has never been tested against a real, audio-only frontend, where the visual affordances every other Part 7 module has relied on — a button, a collapsed panel, a hover state — simply don't exist. This module is where that claim either survives contact with a microphone and a speaker, or turns out to have hidden a UI-shaped assumption.

There's a second reason this matters beyond completing Part 7's surface coverage. Voice is the sharpest test yet of a principle that's run through the entire handbook: a correctness guarantee proven at one layer (here, the agent loop) is only as real as the layer that has to actually exercise it under pressure (here, an interface with no time to spare and no idle-click way to double-check a decision). If barge-in doesn't cleanly cancel an in-flight approval-pending action, or if a spoken confirmation is ambiguous enough that a user can't tell what they just approved, you've reintroduced exactly the class of bug Module 2 spent an entire module closing — just in a modality where nobody can go back and re-read the transcript before it's too late.

## 6. Theory

### 6.1 Why voice isn't "chat with audio" — the constraints that don't exist elsewhere

Three properties are unique to voice among every interface Part 7 has built:

- **A response has to start speaking before it's fully generated, or the interaction feels broken.** Text streaming (Module 1) tolerates the user watching tokens arrive; a voice interaction has a much tighter perceived-latency budget — silence past roughly half a second to a second starts to feel like the system is stuck, not "thinking," which is precisely the latency-vs-quality tension Part 3, Module 9 named for text and Part 5, Module 5 named for voice specifically.
- **The user can interrupt mid-response (barge-in), and that has to work instantly and cleanly.** No text interface in this handbook has needed to handle a user "talking over" a response — closing a browser tab or clicking cancel (Module 1) is the closest analog, and this module directly reuses that cancellation-verification discipline, but barge-in has to be detected from live audio input, not a discrete click event.
- **There's no idle time for a visual confirmation dialog.** Every approval gate built so far (Module 2) assumes the user can look, read, and click at their own pace. A voice interaction is inherently synchronous and momentary — if the confirmation step takes noticeably longer than the rest of the interaction, or requires reading back a wall of text aloud, it breaks the exact interaction pattern voice exists to provide.

These three properties are why voice gets its own module rather than being folded into Module 1 or 2 as "also supports audio" — the constraints aren't additive, they're structurally different enough to need their own state machine.

### 6.2 Turn-taking as its own state machine, coexisting with tool-call and approval states

Voice interactions have a conversational rhythm that text doesn't need to model explicitly: `listening → thinking → speaking`, with `interrupted` as a transition reachable from `speaking` at any time.

```
 listening ──(end of user utterance detected)──▶ thinking
     ▲                                               │
     │                                      response ready
     │                                               ▼
     └───────(user starts speaking again)──── speaking
                     "barge-in"                      │
                                              response complete
                                                      │
                                                      ▼
                                                 listening
```

This state machine is independent of, and coexists with, the tool-call lifecycle from Module 2 — a single `speaking` turn might have an approval-pending tool call nested inside it, exactly the way a single chat message in Module 2 could contain multiple tool calls at different lifecycle stages. Model them as separate reducers, same discipline as every multi-stream design so far: a `turnReducer` for listening/thinking/speaking/interrupted, and the existing `traceReducer` for tool-call/approval state, both driven by the same underlying event stream but never sharing mutable state directly.

### 6.3 Barge-in as first-class cancellation — reusing, not reinventing, Module 1's discipline

When a user interrupts the agent mid-speech, three things must happen essentially simultaneously: audio playback stops, the underlying generation request is actually cancelled server-side (not just muted client-side — the exact distinction Module 1's cancellation-verification exercise was built around), and the turn state transitions to `listening` so the new utterance is captured. Get the middle one wrong and you reproduce Module 1's original bug in a much less forgiving setting: the agent keeps generating (and being billed for, and potentially executing tool calls from) a response the user already interrupted and doesn't want, purely because the UI stopped *playing* audio without actually cancelling the underlying call.

```typescript
function handleBargeIn(runId: string) {
  audioPlayer.stop();                    // immediate, client-side, for perceived responsiveness
  dispatch({ type: "TURN_INTERRUPTED" }); // turnReducer: speaking → listening
  cancelGeneration(runId);               // MUST reach the actual provider call —
                                          // same verified-cancellation requirement as Module 1,
                                          // now on a much tighter, real-time budget
}
```

The verification bar is identical to Module 1's: confirm cancellation reaches the `llm-client-core` adapter call, not just that the client stopped rendering/playing. The stakes are higher here because a voice agent that's mid-`executing` on a `requires_approval`-gated action when barge-in happens needs a defined answer — does interruption also cancel a pending tool execution, or only the spoken response? (Answered explicitly in §6.4.)

### 6.4 The approval gate, redesigned for zero idle time

Module 2's visual approve/reject buttons don't exist here. Reading a full action's arguments aloud before asking for confirmation is often worse than a glance at a screen — verbose, easy to mishear, and it defeats the point of a fast, spoken interaction. Three deliberate design choices, all following Part 5, Module 5's core finding that latency changes *when* guards run rather than whether:

- **A short, deliberately narrow confirmation phrase**, not a full argument readout: "I'll send that email to finance — say 'yes' to confirm." This is a genuine trade-off (less detail than Module 2's visual card shows) accepted because the alternative (reading the full email body aloud) breaks the interaction's pace far more than it protects the user — and it's a decision that should be revisited per-action-type, not applied uniformly: a `send_email` action might warrant this brief phrasing, while a much higher-stakes action might legitimately need to punt to a visual confirmation instead (see the dual-modality decision in §6.5).
- **An explicit, narrow confirmation grammar**, not open-ended natural-language parsing of the response. Accept only a small, well-defined set of confirms/denials ("yes," "confirm," "go ahead" vs. "no," "cancel," "don't") and treat anything ambiguous as a *denial by default*, never as an assumed confirmation — the same fail-closed principle as any security gate, applied to intent recognition instead of authentication.
- **Barge-in during a confirmation phrase is itself the denial signal**, not merely an interruption to be recovered from. If a user starts talking over "I'll send that email—", the safest interpretation is that they want to stop or reconsider, not that they're about to say "yes" faster. Treat any barge-in during a pending confirmation as an automatic reject, log it distinctly (`reason: interrupted_during_confirmation`) so it isn't conflated with an explicit spoken "no" in eval data, and let the user re-initiate if they did mean to approve.

### 6.5 The visual companion — a deliberate, not default, dual-modality decision

It's tempting to assume a voice interface should always have a matching visual view (a screen showing the transcript, trace, and citations from Modules 1–3 running alongside the audio). Resist defaulting to this either way — it's a real design decision with a real trade-off, not a free addition:

- **Argument for a visual companion**: high-stakes approval decisions (§6.4) genuinely benefit from a fallback to Module 2's visual card when an action is too consequential to compress into a short spoken phrase, and accessibility for users who benefit from a visual transcript alongside or instead of audio.
- **Argument against defaulting to one always-on**: a voice interface's entire value proposition for many use cases (hands-free, eyes-elsewhere use) disappears the moment a user has to look at a screen to actually understand what's happening — at which point you've quietly rebuilt Module 2/3's UI with an audio track bolted on, not built a genuine voice interface.

The resolution: the visual companion is **conditional on stakes**, not a universal always-shown layer. Low-stakes turns (a spoken answer, a `read`-only tool call) stay audio-only. A `requires_approval` action above some configurable stakes threshold explicitly escalates to the visual approval card from Module 2 — reusing that component directly, not rebuilding it — with the voice interface pausing and stating aloud that a visual confirmation is needed. This is the module's version of Module 2's approval-gate design principle: don't let a UI shortcut quietly weaken a guarantee that exists precisely because an action was consequential.

## 7. Mental Models

- **"Silence past a second doesn't feel like patience — it feels like broken."** Voice's perceived-latency budget is stricter than text's, structurally, not just by preference.
- **"Muting audio isn't cancelling the request — the same rule as clicking 'stop' in Module 1, now with real stakes if you get it wrong."**
- **"Ambiguous speech is a denial, not an assumed yes — fail closed on intent recognition exactly like you'd fail closed on auth."**
- **"A visual companion is a stakes-triggered escalation, not a default second screen — otherwise you didn't build a voice interface, you built a chat UI with sound."**

## 8. Visual Explanation

**Turn-taking state machine coexisting with tool-call state:**

```
 turnReducer:       listening ──▶ thinking ──▶ speaking ──▶ listening
                                                   │
 traceReducer                                      │ (nested, independent)
 (from Module 2):                          proposed → validated →
                                            approval_pending (voice-native
                                            confirmation OR escalate to
                                            visual card, per §6.5) →
                                            executing → succeeded/failed
```

**Barge-in during a pending voice confirmation:**

```
 agent: "I'll send that email to finance—"
                │
        user starts talking  ◀── barge-in detected mid-confirmation
                │
                ▼
       treated as: reason = "interrupted_during_confirmation"
       tool call → failed (NOT executed)
       turn state → listening (captures the new utterance)
```

**Stakes-based escalation to visual companion:**

```
 requires_approval action proposed
              │
       stakes classifier
        │            │
     low/medium     high
        │            │
        ▼            ▼
  spoken narrow   pause voice, show Module 2's
  confirmation     <ToolCallCard> visual approve/reject,
  (§6.4)           state aloud: "please confirm on screen"
```

## 9. Recommended Resources

1. **Part 5, Module 5 (this handbook)** — re-read the voice-agent section immediately before building; this module is close to a direct frontend translation of its central claim and should be checked against it.
2. **Google's Web Speech API / MediaStream docs (developer.mozilla.org)** — for the practical mechanics of capturing microphone input and detecting end-of-utterance/barge-in in a browser context, if building the browser-native version rather than a native app.
3. **Nuance/Amazon Alexa "Voice Design Guide" or similar published voice-UX guidelines** — useful for calibrating confirmation phrasing (§6.4) against real, tested conventions rather than inventing phrasing from scratch; treat as design precedent, not gospel.
4. **Part 3, Module 9 (Latency & Cost Optimization)** — re-read the streaming-for-perceived-latency section specifically; TTS chunking strategy is the audio-output analog of the token-streaming-for-perceived-latency argument made there.
5. **W3C — "Web Speech API" recommendation** — the formal spec for speech synthesis/recognition interruption handling, relevant to implementing barge-in detection correctly rather than approximately.

## 10. Exercises

1. Design the stakes classifier from §6.5: what specific properties of a `requires_approval` action (irreversibility? cost? recipient count? data sensitivity?) should push it above the visual-escalation threshold, and where would `send_email` versus, say, `delete_all_records` land?
2. Write the exact confirmation grammar (accepted phrases for confirm/deny) you'd use for a narrow spoken approval, and justify your out-of-grammar default (§6.4's fail-closed rule).
3. A user barges in *after* saying "yes" but *before* the tool call transitions from `approved` to `executing`. Should the barge-in still be treated as a denial per §6.4's rule, or has the decision already been made? Justify your answer against the tool-call lifecycle from Module 2.
4. Sketch the `turnReducer`'s action types as a TypeScript discriminated union, and show how a barge-in event correctly resets `turnReducer` state to `listening` without needing to know anything about whatever `traceReducer` state a nested tool call happened to be in.
5. Voice's stricter latency budget makes Part 6's prefill/decode split (Module 1's Theory) more operationally urgent than it was for text. Explain concretely why a decode-phase latency regression that was tolerable for a text chat UI might be unacceptable for a voice interface, tying your answer back to §6.1's "silence feels broken" argument.

## 11. Mini-Project

Build a standalone `<VoiceTurnIndicator>` component driven by a mocked, timer-based sequence of turn-state events (`listening → thinking → speaking`, including one barge-in mid-`speaking`). It should visually indicate current turn state (even though the real interface is audio-first, a minimal visual indicator is a reasonable accessibility affordance) and correctly reset to `listening` on the mocked barge-in event, with no lingering `speaking`-state artifacts. No real audio yet — this isolates the turn-taking state machine from the harder audio-pipeline wiring in the Production Project.

## 12. Production Project: `voice-shell` v1

Build `voice-shell` as a new, audio-first client extending `chat-shell`'s architecture (Modules 1–4) rather than rebuilding it — same reducer discipline, same `AgentEvent` contract, new turn-taking layer and audio pipeline on top.

**Scope:**

1. **`turnReducer`** (§6.2), independent of `traceReducer`, `tokenReducer`, and `citationReducer`, verified via an extension of `agent-trace-render-stress-test` covering interleaved turn and tool-call events.
2. **Barge-in handling** (§6.3), with an end-to-end test proving a barge-in event actually cancels the underlying `llm-client-core` provider call — the voice-specific version of Module 1's cancellation-verification test, now against a tighter real-time budget.
3. **Voice-native confirmation grammar** (§6.4), with the fail-closed ambiguous-input rule and the distinct `interrupted_during_confirmation` denial-reason logging.
4. **Stakes classifier and visual-escalation path** (§6.5, Exercise 1), reusing `<ToolCallCard>` from Module 2 directly for high-stakes escalations rather than rebuilding a voice-specific confirmation UI for those cases.
5. **TTS chunking strategy**, applying Part 3, Module 9's streaming-for-perceived-latency argument to audio output — begin speaking as soon as a safely-speakable chunk of the response is available, not only once generation fully completes, while remaining compatible with barge-in cancelling mid-chunk cleanly.
6. **`voice-interaction-stress-test`**: adversarial scenarios — barge-in during a confirmation phrase, barge-in during `executing` (Exercise 3's question, resolved and tested), rapid back-to-back utterances, and ambiguous confirmation phrases — asserting the system never silently executes an action the user didn't clearly approve.
7. **Accessibility note and `<VoiceTurnIndicator>` integration** as a genuine dual-modality affordance (not a full visual companion by default, per §6.5), documented as the accessibility-driven exception to the "audio-only by default" rule.

**Documentation**: an ADR for the stakes-classifier thresholds (Exercise 1) and the decision to escalate to Module 2's existing visual card rather than building a separate high-stakes voice confirmation flow; and an explicit note tying the barge-in cancellation guarantee back to Module 1's original cancellation-verification requirement, showing it as the same guarantee re-proven under a stricter constraint rather than a new one invented from scratch.

**Hands off to:** the Part 7 capstone, which will need to decide how `voice-shell`, `chat-shell`, and `ops-console` relate as one coherent product surface (three separate applications sharing a component/event architecture, per the pattern Module 4 already established for `ops-console`) rather than three unrelated builds.

## 13. Practice & Interview Questions

1. Why does voice need its own turn-taking state machine rather than reusing the tool-call lifecycle state machine from Module 2 directly?
2. Explain why muting client-side audio playback on barge-in is not sufficient, and what specifically must also happen for the interruption to be safe rather than just quiet.
3. Why should an ambiguous spoken confirmation be treated as a denial by default, rather than the system asking a clarifying follow-up question? What does asking a follow-up cost in this specific modality that it wouldn't cost in a text chat?
4. Design the stakes classifier for escalating a voice approval decision to a visual confirmation card. What signals would you use, and why shouldn't every `requires_approval` action escalate by default?
5. In an interview: you're asked why a voice assistant feels "laggy" even though its underlying model is the same one powering a text chat product that feels fine. Walk through the latency-budget argument from §6.1 and connect it to Part 6's prefill/decode distinction.
6. Why is defaulting to an always-on visual companion screen for a voice interface a design failure, even if it technically provides more information to the user?

## 14. Common Mistakes

- **Treating voice as "chat plus audio."** Misses the structurally different latency budget, barge-in requirement, and the absence of idle time for visual confirmation — leads to an interface that technically works but feels broken in ways a text UI never would.
- **Stopping audio playback on barge-in without cancelling the underlying generation call.** The same bug Module 1's cancellation exercise caught, now with real cost and correctness consequences (a cancelled-sounding response can still complete and execute a tool call server-side).
- **Treating an ambiguous confirmation as an assumed "yes."** Violates the fail-closed principle that every other approval-gated action in the handbook has followed since Module 2.
- **Reading full action arguments aloud before every confirmation.** Defeats the pace advantage voice exists to provide; reserve full detail for the stakes-triggered visual escalation instead.
- **Defaulting to an always-visible companion screen.** Quietly turns the voice interface into Module 2/3's UI with an audio track, losing the hands-free/eyes-elsewhere value proposition voice is meant to deliver.
- **Building a separate, from-scratch high-stakes confirmation UI for voice instead of escalating to Module 2's existing visual card.** Duplicates a component and its already-verified guarantees for no reason.

## 15. Debugging Exercise

During testing, a tester barges in while the agent is mid-sentence confirming a `send_email` action ("I'll send that email to—"). The system correctly stops speaking and correctly logs the denial. But thirty seconds later, the email arrives in the recipient's inbox anyway.

Form a hypothesis before reading further:

<details>
<summary>Hint 1</summary>
The denial was logged correctly and the turn state transitioned correctly. That means the *client-side* and *event-logging* paths worked. What's left that could still have gone wrong, given §6.3's explicit distinction between "stopped playing" and "actually cancelled"?
</details>

<details>
<summary>Hint 2</summary>
Check whether the barge-in handler's `cancelGeneration(runId)` call in §6.3's code sample was actually wired to cancel the *tool call* specifically, or only the *text/speech generation* — these can be two separate server-side operations if the tool call had already been dispatched for execution before the confirmation phrase even started playing.
</details>

<details>
<summary>Likely root cause</summary>
This is a sequencing bug, not a cancellation-plumbing bug: the implementation had already dispatched the tool call for execution optimistically — perhaps to reduce perceived latency by starting the send while the confirmation phrase was still playing, intending to cancel it only if denied — rather than correctly holding execution at the `approval_pending` state until confirmation completes, as the state machine in Module 2 and this module's §6.4 both require. The barge-in correctly cancelled the *voice generation and turn state*, but there was no execution to cancel, because it had already started, which is precisely the failure mode `requires_approval` exists to prevent (Part 5, Module 1's structural gate). The fix is to re-verify, with an explicit test, that no `executing` state is ever entered before an `approved` event — not `approval_pending`, not "confirmation phrase started playing" — regardless of any latency-optimization temptation to start early. This is the voice-specific instance of a mistake the handbook has flagged before in different clothes: an optimization that quietly bypasses a structural safety gate is never worth the latency it saves.
</details>

## 16. Checklist

- [ ] `turnReducer` implemented independently of `traceReducer`/`tokenReducer`/`citationReducer`
- [ ] Barge-in verified to cancel the actual underlying provider call, not just client-side audio playback
- [ ] Ambiguous spoken confirmations default to denial, never to an assumed approval
- [ ] Barge-in during an active confirmation phrase is logged and treated as a distinct denial reason
- [ ] Stakes classifier implemented and documented, escalating high-stakes actions to Module 2's existing visual `<ToolCallCard>` rather than a new component
- [ ] No `executing` state is ever entered before a confirmed `approved` event, verified by an explicit test against the exact bug in §15
- [ ] TTS chunking begins speech before full generation completes, without breaking clean mid-chunk cancellation
- [ ] Visual companion is stakes-triggered, not an always-on default
- [ ] `voice-interaction-stress-test` covers barge-in-during-confirmation, barge-in-during-execution, and ambiguous-phrase scenarios
- [ ] ADR written for stakes-classifier thresholds and the decision to reuse Module 2's visual card for escalation

## 17. Summary

Voice looks like a modality swap and is actually a constraints swap: a stricter latency budget that changes what "responsive" even means, a barge-in requirement no text interface in this handbook has needed, and the disappearance of idle time for a leisurely visual confirmation. None of that required a new safety mechanism — Part 5, Module 5's claim held — but it required re-proving every existing guarantee (verified cancellation, fail-closed approval, a structural `approval_pending` gate that latency-optimization pressure can't be allowed to route around) under a much less forgiving margin for error. The debugging exercise in this module is really the whole module's thesis in miniature: a guard that's structurally correct on paper is only as real as the interface that has no idle moment left in which to accidentally skip it. `voice-shell` closes the last of Part 5, Module 5's UI-shaped open questions, and sets up the Part 7 capstone's real remaining problem — not building anything new, but reconciling `chat-shell`, `ops-console`, and `voice-shell` into one coherent product.

## 18. Next Steps

Reply "continue" for the Part 7 capstone module, or flag anything to go deeper on.
