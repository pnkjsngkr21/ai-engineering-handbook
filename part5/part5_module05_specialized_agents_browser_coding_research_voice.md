# Part 5, Module 5: Specialized Agents (Browser, Coding, Research, Voice)

> Applies the full Modules 1–4 stack — bounded loop, reflection, memory
> with provenance, orchestration — to four concrete agent categories.
> No new safety mechanism is invented per domain; each domain instead
> forces you to design a correctly-scoped action space and correctly
> calibrate which guards matter most, using tools you already own.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Design a domain-specific action space for browser, coding, research,
   and voice agents, each as an instance of Module 1's
   enumerated-action-space discipline, not a special case requiring new
   theory.
2. Identify which of the four domains most stresses which of your
   existing guards (termination, approval gating, reflection,
   provenance) and explain why the risk profile differs by domain.
3. Explain why a **coding agent's** sandbox is the direct analog of
   Module 1's enumerated action space, taken to its logical extreme —
   and why "the sandbox is the action space" is the correct mental
   model rather than treating sandboxing as a separate concern.
4. Explain why a **browser agent** is the domain where Module 1's
   prompt-injection argument (Section 6.5 of that module) is most
   acute, because the entire observation stream is untrusted,
   attacker-reachable content by default.
5. Explain why a **voice agent** introduces a genuinely new
   constraint — latency budgets tight enough to change which parts of
   the Module 1–4 stack can run synchronously versus must be deferred.
6. Build one working specialized agent (your choice of the four, with
   the other three specified as design exercises) that demonstrably
   inherits `agent-core`'s guards rather than reimplementing them.

## 2. Prerequisites

- Part 5, Modules 1–4, completed — this module builds nothing new at
  the mechanism level; it is entirely about domain-specific action-space
  design and guard calibration.
- Part 0, Module 5 (Docker) and Part 1, Module 9 (Security
  Fundamentals) — directly relevant to the coding agent's sandbox
  design.
- Part 0, Module 4 (Networking, HTTP, REST & JSON) — relevant to the
  browser agent's action space.

## 3. Estimated Study Time

10–13 hours across 2–3 sessions (more if you build more than one
specialized agent end-to-end rather than one plus three design
exercises).

## 4. Difficulty

★★★☆☆ (3/5) — Mechanically light; the difficulty is calibration
judgment (deciding what's actually risky in each domain) rather than new
implementation complexity.

---

## 5. Why This Matters

By this point in Part 5, you have a general-purpose agent runtime. It
would be easy — and wrong — to treat "build a browser agent" or "build a
coding agent" as a fresh design problem requiring new research. The
actual skill this module tests is the opposite: can you look at a new
domain and correctly map it onto the guards you already have, rather
than either (a) under-guarding it because it "feels" like a simple
domain, or (b) over-building bespoke new safety machinery that
duplicates what `agent-core` already does. Interviewers evaluating
agent-systems experience are specifically listening for whether a
candidate reaches for "let me add a new safety layer" or "let me map
this onto the enumerated-action-space and approval-gate pattern I
already trust" — the second answer is the one that scales across
domains you haven't seen yet.

It also matters because each domain genuinely does stress a different
part of the stack the hardest, and knowing which one is the actual skill
being taught here: a coding agent's real danger is in what its sandbox
permits, not in its planning quality. A browser agent's real danger is
that literally every observation it reads is potentially adversarial
content, at a scale single-document RAG (Part 4) never had to consider.
A voice agent's real danger isn't safety at all — it's an engineering
constraint (latency) that forces you to restructure *when* your existing
guards run, not whether they run.

---

## 6. Theory

### 6.1 Browser agents: the observation stream is the attack surface

A browser agent's action space (Module 1) is comparatively simple to
enumerate: navigate, click, type, extract text, wait. The hard part is
not the action space — it's that **every single observation is
attacker-reachable by default.** A research agent retrieving from
`rag-engine` (Part 4) reads from a corpus you control and have already
access-controlled (Part 4, Module 5). A browser agent reads whatever is
on whatever page it navigates to — pages you do not control, that may
have been specifically crafted to contain text targeting exactly this
agent's planning prompt (the scenario from Module 1's debugging
exercise, except now the "malicious document" isn't a hypothetical edge
case in your own corpus, it's the default assumption for every page).

This means the input-guardrail layer (Part 3, Module 6; wired into
`agent-core` in Module 1) that filters observations before they re-enter
the scratch-pad is not an optional hardening step for a browser agent —
it is the single most load-bearing piece of the entire stack, and it
must run on every page's extracted text before that text influences
`think()`'s next decision, with no exceptions for "trusted-looking"
domains (a compromised or spoofed trusted-looking domain is precisely
what an attacker would use). The second load-bearing piece: every
action with real-world effect (submitting a form, making a purchase,
sending a message) is `requires_approval: True` by default for a
browser agent, and the burden of proof is on the developer to justify
setting it `False` for a specific, narrow, reversible action — the
inverse default from, say, `rag-engine`'s read-only retrieval action in
Module 1's Production Project.

### 6.2 Coding agents: the sandbox *is* the action space

It's tempting to give a coding agent one broad `run_shell_command`
action, reasoning that code execution is inherently general-purpose and
enumerating narrower actions would be limiting. This is exactly the
mistake Module 1, Section 6.5 warned against, and coding agents are
where getting it wrong is most catastrophic, because the action's blast
radius is "arbitrary code execution" by construction.

The correct framing: **the sandbox's permissions, not the agent's
prompt, define the action space.** A coding agent's `execute_code`
action is safe exactly to the extent that the sandbox it runs in has no
network access it doesn't need, no filesystem access outside a scoped
working directory, no credentials, and a hard resource/time limit
independent of Module 1's iteration cap (a single sandboxed execution
could itself hang or fork-bomb, which the loop-level iteration cap does
nothing to prevent). This reuses Part 0, Module 5's Docker fundamentals
directly: the sandbox is a container with an explicitly minimal
capability set, and "what can this agent do" is answered by reading the
container's configuration, not the agent's system prompt. Reflection
(Module 2) is high-value here specifically for one narrow question —
"did the code actually run and produce the expected kind of output, or
did it error/hang/produce nonsense" — because a coding agent's
observations (stdout, stderr, exit codes) are exactly the kind of
structured, low-ambiguity signal Module 2, Section 6.4 said needs
*less* reflection, except for the one case that matters most: silent
wrong-output success (code that runs cleanly but computes the wrong
thing), which is unambiguous to a human reviewer and easy for reflection
to miss unless it's specifically prompted to check output *plausibility*
against the sub-goal, not just execution success.

### 6.3 Research agents: mostly a direct instance of Modules 1–4, correctly composed

A research agent (retrieve → synthesize → cite, potentially across many
sources, potentially over a long horizon) is the domain closest to
"nothing new" — it is `agent-core` plus `rag-engine` (Module 1's
Production Project) plus, for genuinely long research tasks, Module 3's
resumable task-state mechanism. The one design decision worth naming
explicitly: a research agent's `synthesize`/`cite` step should reuse
Part 4, Module 8's `FaithfulnessScorer` as its reflection mechanism
(Module 2) rather than a generic critique prompt — faithfulness
checking against retrieved context *is* the correctly-calibrated
reflection for this specific action type, and building a separate,
weaker critique mechanism here would be reinventing something you
already validated.

### 6.4 Voice agents: latency forces synchronous/asynchronous restructuring

Voice is the one domain in this module that introduces a genuinely new
constraint, and it isn't a safety constraint — it's latency. A voice
agent's round-trip budget (user speaks, agent must begin responding) is
measured in a small number of hundreds of milliseconds if the
interaction is to feel conversational, per Part 3, Module 9's streaming
discipline (perceived latency, not total latency, is what matters) —
applied here under a much tighter budget than a chat UI's "streaming
tokens while the user reads" tolerance.

This means several pieces of the Modules 1–4 stack that were fine to run
synchronously, in-loop, for a text agent must be restructured for a
voice agent:

- **Reflection (Module 2)**, if run synchronously before every spoken
  response, can blow the latency budget on its own, since it's a full
  extra LLM call. For voice, prefer confidence-only reflection that
  runs *asynchronously, after* the response has already been spoken,
  feeding into episodic memory (Module 3) for future turns rather than
  gating the current one — a genuine change to Module 2's "plan-altering
  vs. confidence-only" policy choice per action type, driven purely by
  the latency constraint, not by any change in how much the step
  actually needs scrutiny.
- **Human approval gates (Module 1)** for `requires_approval: True`
  actions cannot use the same UI pattern as a text or browser agent —
  "pause and wait for a click" doesn't map onto a live voice
  conversation. The action either needs a voice-native confirmation
  turn ("I'll go ahead and do X — say stop if you don't want that")
  with an explicit, narrow confirmation window, or the action needs to
  be deferred out of the live conversation entirely and confirmed
  through a different channel afterward — a design decision that must
  be made per action type, not solved generically.
- **Memory retrieval (Module 3)**, which for a text agent can afford a
  `rag-engine` call mid-`think()`, needs its latency profile measured
  specifically for the voice case — this is Part 3, Module 9's
  model/tool-routing discipline again: a voice agent may need a
  smaller, faster retrieval path for in-conversation lookups, with
  fuller retrieval deferred to between-turn processing.

The mental model worth taking away: voice doesn't change *what* guards
you need, it changes *when* they're allowed to run relative to the
user-facing response — the same distinction Part 3, Module 9 drew
between "must happen before the response" and "can happen after,"
now applied to an entire agent stack instead of a single generation
call.

---

## 7. Mental Models

1. **"The sandbox's configuration is the coding agent's real action
   space — read that, not the prompt, to know what it can do."**
2. **"For a browser agent, assume every observation is adversarial
   content by default, because it usually can be."**
3. **"A research agent is mostly `agent-core` plus `rag-engine`,
   correctly composed — resist inventing anything new here."**
4. **"Voice doesn't remove any guard — it forces you to decide which
   guards run before the user hears a response and which run after."**

---

## 8. Visual Explanation

```
   Same underlying stack (Modules 1-4) for all four domains:

        [ bounded loop | reflection | memory+provenance |
                  orchestration (if multi-agent) ]

   What changes per domain is WHERE the risk concentrates:

   Browser  ──► observation-stream trust (6.1)
              every page = potentially adversarial input
              action defaults flip to requires_approval=True

   Coding   ──► the sandbox boundary (6.2)
              action space = container capability set,
              not the action's name in the prompt

   Research ──► composition, not novelty (6.3)
              agent-core + rag-engine + FaithfulnessScorer
              as reflection — nothing new to invent

   Voice    ──► timing of guard execution (6.4)
              same guards, but latency forces some
              (reflection, approval) to move from
              "before response" to "after response,
              async" for specific action types
```

---

## 9. Recommended Resources

1. **OWASP — "LLM08: Excessive Agency" (LLM Top 10)** — directly
   addresses over-broad action spaces; read specifically against
   Section 6.2's sandbox argument.
2. **Docker security documentation — capability restrictions and
   `--network none`** — the concrete mechanism behind Section 6.2's
   "sandbox is the action space" claim; implement, don't just read.
3. **Anthropic — "Building Effective Agents"** (revisit) — the section
   on autonomous coding/browsing agents specifically; compare its
   guardrail recommendations against your own Module 1 mechanisms.
4. **Part 3, Module 9 (your own material)** — re-read the
   streaming/perceived-latency section specifically before designing
   the voice agent's guard-timing restructuring in Section 6.4.

---

## 10. Exercises

1. For each of the four domains, write the enumerated action list (5–8
   actions) and mark `requires_approval` for each, using Section 6's
   domain-specific defaults (browser defaults toward `True`; research's
   `retrieve` defaults toward `False`, mirroring Module 1's Production
   Project).
2. Design a coding agent's sandbox configuration (network access,
   filesystem scope, resource limits, time limits) independent of any
   agent framework — purely as a container spec — and justify each
   restriction.
3. Construct a web page containing an injected instruction targeting a
   browser agent's `requires_approval: True` submit action. Confirm
   your input-guardrail layer (reused from Module 1) flags it before it
   reaches `think()`.
4. Redesign Module 2's reflection policy for one action type under a
   200ms latency budget: decide what moves to async, what's dropped
   entirely, and what stays synchronous, and justify each choice against
   Section 6.4.
5. Take your research agent design and confirm its `synthesize` action's
   reflection is literally calling `FaithfulnessScorer` (Part 4, Module
   8), not a separately-written critique prompt — if it isn't, refactor
   it to be.

---

## 11. Mini-Project

**`domain-risk-map`**: a short document, one section per domain, stating
in one or two sentences each: (a) which existing guard (termination,
approval, reflection, provenance) is most load-bearing for this domain
and why, and (b) one concrete attack or failure scenario specific to the
domain, and which guard catches it. This is meant to be a fast
calibration exercise, not a build — the point is to prove you can map a
new domain onto existing mechanisms quickly, before committing to
building any one of them in full.

---

## 12. Production Project: one specialized agent, built in full

### Scope

Choose **one** of the four domains and build it completely, extending
`agent-core`/`agent-orchestrator` (Modules 1–4) with only the
domain-specific action space and guard calibration described in this
module — no new safety mechanism invented from scratch. Specify the
other three at the design level (action space + guard calibration,
matching Exercise 1's format) without full implementation.

Whichever domain you choose, the deliverable must include:

- A fully enumerated, schema-typed action space with `requires_approval`
  and `reflection_policy` set explicitly per action (Modules 1–2's
  discipline, no silent defaults).
- For the browser agent specifically, if chosen: a passing test using
  Exercise 3's injected-page scenario, proving the observation
  input-guardrail actually catches it.
- For the coding agent specifically, if chosen: an actual container
  sandbox spec (not a description) with network/filesystem/resource
  restrictions, and a test proving an out-of-scope action (e.g., network
  access) is genuinely blocked at the sandbox layer, not just discouraged
  in the prompt.
- For the research agent specifically, if chosen: `synthesize`'s
  reflection step calling `FaithfulnessScorer` directly, with a passing
  regression test reusing Part 4, Module 8's golden dataset categories.
- For the voice agent specifically, if chosen: measured (not estimated)
  latency for each guard under the restructured sync/async policy from
  Section 6.4, with a stated latency budget and evidence the budget is
  met.
- The domain's regression tests added to the standing Part 5 suite
  (`loop-stress-test`, `reflection-agreement-eval`, `poisoning-drill`,
  `correlated-failure-drill`), so the specialized agent is proven to
  inherit — not bypass — the general guarantees.

### Explicit extension point

Whichever domain you build becomes a direct seed for **Part 11's
capstone projects** (Browser Agent, AI Code Reviewer, Research Agent,
Voice Assistant are named capstones matching these four domains
exactly), and the domain-risk-map produced here becomes the starting
design document for the other three when you eventually build them as
separate capstones.

---

## 13. Practice & Interview Questions

1. Why is "the sandbox's configuration" the correct way to think about
   a coding agent's action space, rather than the set of action names
   in its prompt?
2. Why does a browser agent need to treat every observation as
   potentially adversarial by default, in a way a research agent
   reading from `rag-engine` does not?
3. What's the one genuinely new constraint voice agents introduce, and
   how does it change *when* guards run rather than *whether* they run?
4. Why should a research agent's reflection step reuse
   `FaithfulnessScorer` rather than a fresh critique prompt?
5. Give an example of a coding-agent failure that a sandbox boundary
   catches but that loop-level guards (iteration cap, cost budget) do
   not, and explain why.
6. How would you decide, for a specific voice-agent action, whether its
   `requires_approval` gate should be a live voice confirmation turn or
   deferred to a different channel entirely?

---

## 14. Common Mistakes

- **Giving a coding agent one broad `run_shell_command` action** —
  reproduces Module 1, Section 6.5's exact warning, with the worst
  possible blast radius.
- **Trusting sandbox restrictions described in a prompt instead of
  enforced by the container** — "the agent was told not to access the
  network" is not a security boundary; the container's network
  configuration is.
- **Treating browser-agent observations like RAG observations** —
  assuming input-guardrail filtering that was "enough" for a controlled
  document corpus (Part 4) is enough for arbitrary web content, when the
  threat model is meaningfully worse.
- **Building a bespoke research-agent critique mechanism** — reinventing
  `FaithfulnessScorer` instead of reusing it, usually producing a weaker
  check with less validated calibration.
- **Solving voice latency by cutting guards entirely** — dropping
  reflection or approval gates outright to hit a latency budget, instead
  of restructuring them into an async, after-the-response pattern that
  preserves the guarantee while meeting the timing constraint.
- **Skipping the regression-suite integration** — building a
  specialized agent that passes its own new tests but was never run
  against `poisoning-drill`/`correlated-failure-drill`/etc., so you
  don't actually know whether it inherited the general guarantees or
  quietly bypassed them.

---

## 15. Debugging Exercise

Your coding agent, sandboxed with no network access, is somehow
exfiltrating data — a code execution result includes content that
appears to have been fetched from an external URL. Using Section 6.2,
walk through: (a) the most likely gap (hint: "no network access" is a
claim about the sandbox's *runtime* network policy — check whether the
build/dependency-installation step for the agent's code environment runs
under the same restriction, since package installation is a common
side-channel), (b) the specific sandbox configuration check that would
have caught this before deployment, and (c) why this is a sandbox-layer
fix, not a prompt-layer or reflection-layer fix.

---

## 16. Checklist

- [ ] I have a fully enumerated action space for my chosen domain, with
      explicit `requires_approval`/`reflection_policy` per action.
- [ ] I have a design-level action space + guard calibration for the
      other three domains (`domain-risk-map`), even without full builds.
- [ ] My browser agent (if built) passes the injected-page test,
      proving observation filtering catches it.
- [ ] My coding agent (if built) has a real, tested sandbox boundary —
      verified by an actual blocked out-of-scope action, not just a
      documented restriction.
- [ ] My research agent (if built) uses `FaithfulnessScorer` directly
      as its reflection mechanism, not a bespoke critique prompt.
- [ ] My voice agent (if built) has measured, not estimated, per-guard
      latency, against a stated budget.
- [ ] My specialized agent's tests run alongside the full Part 5
      regression suite, not in isolation.

---

## 17. Summary

None of the four domains — browser, coding, research, voice — require a
new safety mechanism. Each requires correctly mapping the domain's real
risk onto the guards already built in Modules 1–4: a coding agent's
danger lives in its sandbox's configuration, not its prompt; a browser
agent's danger lives in treating every observation as potentially
adversarial by default; a research agent is mostly correct composition
of what you already have, down to reusing `FaithfulnessScorer` directly
as its reflection step; and a voice agent's real constraint is timing,
which changes when guards run, not whether they run. The skill this
module tests — correctly calibrating existing mechanisms to a new
domain's actual risk profile, rather than inventing bespoke new
machinery per domain — is the one that scales to domains not covered
here.

---

## 18. Next Steps

Next: **Part 5, Module 6 — Agent Evaluation & Production Readiness**,
the Part 5 capstone: assembling `loop-stress-test`,
`reflection-agreement-eval`, `poisoning-drill`,
`correlated-failure-drill`, and this module's domain-specific
regression tests into one coordinated CI gate for agentic systems —
mirroring exactly what Part 4, Module 9 did for RAG — plus the honest
architecture review naming what Part 5 still doesn't cover (self-hosted
model serving for lower-latency agents, a real frontend, and
production deployment maturity) and which future Part addresses each.

Reply "continue" for Module 6, or flag anything to go deeper on.
