# Part 8, Module 2 — Secrets, Configuration & Credential Rotation for AI Systems

## 1. Learning Objectives

By the end of this module you will be able to:

1. Enumerate the AI-specific secret types (`ai-api-platform` now holds far more than a typical service: multiple LLM provider API keys, vector DB credentials, embedding-service keys, MCP server auth tokens, agent tool credentials) and explain why the *blast radius* of a leaked AI-provider key differs from a typical database credential leak.
2. Apply the secrets-management discipline already established for `auth-gateway` (Part 1, Module 7) to this larger, more heterogeneous credential surface, without rebuilding secret storage from scratch.
3. Design credential rotation for a live, stateful, multi-provider system — specifically, rotate an LLM provider key or an MCP server token without interrupting an in-flight agent run or losing an `approval_pending` action's ability to resume (Part 5, Module 6; Part 7, Module 2).
4. Extend Module 1's per-axis versioning discipline to a fifth axis — credentials/config — and design its rollback path distinctly from a code, model, prompt, or policy rollback.
5. Identify and close the specific new leak surface MCP introduces: a tool call's arguments or a retrieved document can themselves contain secret-shaped strings (an API key pasted into a support ticket, a token embedded in a scraped page) that must never be logged, cached, or echoed back through the trace UI built in Part 7.
6. Apply the principle of least privilege to agent tool credentials specifically — scoping a tool's credential to only what that tool's declared action space (Part 5, Module 1) actually needs, never a broad, reusable key shared across tools.
7. Design an audited, tested secrets-rotation drill that integrates with `ai-deployment-pipeline` (Module 1) rather than existing as a separate, disconnected process.

## 2. Prerequisites

- Part 1, Module 7 (Authentication & Authorization) — JWT/API-key handling and per-identity rate limiting for `auth-gateway`; this module extends that foundation to a much larger credential surface.
- Part 1, Module 9 (Security Fundamentals) — the OWASP checklist and prompt-injection risk assessment preview; directly relevant to §6.5's tool-output secret-leak surface.
- Part 3, Module 3 (Function/Tool Calling & MCP) — the model-requests/app-executes discipline, and MCP's N×M integration model, which is exactly what multiplies the credential surface this module has to manage.
- Part 5, Module 1 (Agent Fundamentals) — the enumerated action space; you'll scope credentials to it directly in §6.6.
- Part 8, Module 1 (Deployment Strategies) — the four-axis versioning discipline this module extends with a fifth axis.

## 3. Estimated Study Time

9–12 hours (theory + exercises: ~2.5 hours; mini-project: ~2 hours; production project: ~5–7.5 hours).

## 4. Difficulty

★★★☆☆ (3/5) — Conceptually the most straightforward module in Part 8 so far; the difficulty is in thoroughness (enumerating every credential and every leak surface correctly) rather than any single hard mechanism.

## 5. Why This Matters

`auth-gateway` (Part 1, Module 7) solved secrets management for a comparatively simple surface: JWTs, API keys, a database credential or two. `ai-api-platform` as it stands after Parts 2–7 holds a materially larger and more heterogeneous set: LLM provider keys (potentially several, per `llm-gateway`'s multi-provider routing from Part 6, Module 7), vector database credentials (`rag-engine`), MCP server tokens (potentially one per connected tool, per Part 3, Module 3's N×M integration model), and now-per-tool agent credentials (Part 5). Treating all of these as "just more secrets, same storage pattern" misses two things that are genuinely new here, not just more of the same: the blast radius of a leaked LLM provider key is different (unauthorized usage billed to your account, potentially at real financial scale, not just unauthorized data access), and the leak *surface itself* is different — a tool call's arguments or a RAG-retrieved document can contain a secret-shaped string that was never a "credential" your system issued, but still needs to never be logged or echoed.

There's also a direct connection back to Module 1's central insight: credentials and configuration are a fifth axis of change, alongside code/model/prompt/policy, and a leaked or rotated credential needs its own rollback story — one that, unlike the other four axes, usually can't tolerate *any* window of exposure, however brief, the way a canary's statistical tolerance for a prompt regression can.

## 6. Theory

### 6.1 The expanded credential surface, and why blast radius differs by type

Enumerate what `ai-api-platform` now actually holds, because a secrets audit that misses a category is worse than no audit at all — it creates false confidence:

- **LLM provider API keys** — Anthropic, OpenAI, and any other provider `llm-gateway` (Part 6, Module 7) routes to. Blast radius: a leaked key can be used for unauthorized, real-money-billed generation at whatever rate limit the key allows, potentially far exceeding a typical data-access leak's cost, and — worse — usage from a leaked key is indistinguishable from your own legitimate traffic in the provider's eyes, complicating detection.
- **Vector database / embedding service credentials** (`rag-engine`, Part 4) — blast radius closer to a traditional database leak: unauthorized read access to potentially sensitive indexed content, subject to whatever access-control scoping (Part 4, Module 5) the credential itself carries.
- **MCP server tokens** (Part 3, Module 3) — one per connected external tool/service, potentially numbering in the dozens for a mature agent deployment. Blast radius varies per tool — a read-only search MCP server leaking its token is lower-stakes than a leaked token for an MCP server that can send email or modify records.
- **Agent tool credentials** (Part 5) — the sharpest new category: credentials an agent's tools use to act on a user's or organization's behalf. Blast radius here is the agent's actual action space (Part 5, Module 1) — if scoped correctly (§6.6), a leaked tool credential can do only what that specific tool's declared actions allow; if scoped broadly (a single shared "agent service account" key), a leak's blast radius becomes everything every tool can do, which is a design failure independent of the leak itself.

The practical consequence: secrets management for `ai-api-platform` needs per-category rotation cadence and per-category incident response, not one blanket policy — an LLM provider key leak needs immediate revocation and provider-side usage audit; an over-broadly-scoped agent tool credential leak needs the scoping fixed as much as the leak itself remediated.

### 6.2 Reusing `auth-gateway`'s storage pattern — what's actually new here

The underlying secret *storage* mechanism (a secrets manager, encrypted at rest, accessed via short-lived tokens rather than baked into config files or environment variables checked into source) doesn't need reinventing — this is exactly what Part 1, Module 7 already built for `auth-gateway`'s JWT signing keys and database credentials. What's new is entirely about *scope and rotation orchestration* across a much larger, heterogeneous set, and about the specific new leak surface in §6.5. Don't rebuild secret storage; extend its *inventory and rotation policy*.

### 6.3 Rotating a credential without interrupting a live, stateful agent run

A typical service's credential rotation (rotate a database password, restart connections) is usually safe because requests are short-lived. `agent-core`'s runs are not — an agent run can be sitting in `approval_pending` for an arbitrary duration (Part 5, Module 6; Part 7, Module 2's resumption requirement), and rotating the underlying LLM provider key or an MCP server token mid-run must not break that run's ability to resume once the human decision comes back.

The fix: **credential rotation must be dual-key during the transition window**, exactly the same pattern Part 1, Module 7 would have used for `auth-gateway`'s signing keys — old and new credentials both valid simultaneously for a bounded overlap period, with in-flight runs continuing to authenticate with whichever credential they started with until they complete or naturally reconnect, rather than a hard cutover that could strand a paused run mid-approval with no valid credential to resume execution once approved.

```python
# Credential rotation state — dual-valid during transition, not a hard cutover
class CredentialVersion:
    version_id: str
    status: Literal["active", "deprecated", "revoked"]
    valid_until: datetime | None  # None while "active"; set when moved to "deprecated"

# A paused agent run resuming after approval re-authenticates using whichever
# credential version was active when the run started, as long as it's not yet
# "revoked" — giving the deprecated version a bounded grace window
# (long enough to cover realistic approval-pending durations) before hard revocation.
```

This directly extends Module 1's per-axis rollback philosophy: credentials are the fifth axis, and their "rollback" (in this case, rotation) needs its own transition discipline, distinct from a feature-flag flip, because the failure mode of getting it wrong is a stranded, unresumable paused run rather than a quiet accuracy regression.

### 6.4 Least privilege for agent tool credentials — scoping to the declared action space

Part 5, Module 1 established the enumerated action space as a security boundary: an agent can only take actions explicitly defined as available to it, not anything the model might imagine. Credential scoping needs to mirror that boundary exactly — a tool's credential should carry only the permissions that tool's declared actions require, never a broader, reusable identity shared across multiple tools.

Concretely: if `agent-core` has a `send_email` tool and a `read_calendar` tool, they should authenticate with two separate, narrowly-scoped credentials (an email-send-only token, a calendar-read-only token), not one shared "agent service account" with broad access to an email/calendar platform. This is a direct extension of the least-privilege principle Part 1, Module 7 already applied to `auth-gateway`'s API-key scoping, now applied per-tool rather than per-service, because a tool's action space (Part 5, Module 1) is a finer-grained boundary than a whole service's API surface.

The test for correct scoping: **if this specific tool's credential leaked, could an attacker do anything beyond what that tool's declared actions already allow the agent to do?** If yes, the credential is scoped too broadly — the leak's blast radius should never exceed what a well-behaved instance of the agent could already do through that tool, given how the tool is designed to be used.

### 6.5 The new leak surface: secrets appearing inside tool outputs and retrieved content

This is the module's sharpest genuinely new problem, and it doesn't exist in any prior part's threat model. A tool call's *arguments*, a tool's *result*, or a RAG-retrieved *document* can contain a secret-shaped string that your system never issued and doesn't control — a user pastes an API key into a support-ticket body that gets ingested into `rag-engine`'s corpus (Part 4); a scraped web page an agent's browser tool (Part 5, Module 5) visits contains a leaked credential in its HTML; a tool's JSON response includes an internal token as part of some unrelated field.

None of Part 7's trace-rendering work (Module 2's tool-call lifecycle, Module 3's citation rendering) was designed with this threat in mind — a tool result or a cited document's content flows straight into the UI's rendering pipeline by default. The fix is a **secret-pattern scanning pass**, applied to tool arguments, tool results, and retrieved-document content *before* they're either logged, cached (Part 3, Module 8's caching layers), or rendered in any trace/citation UI (Part 7) — using the same class of pattern-matching (regex for common key formats, entropy-based detection for high-randomness strings) a code-scanning secret-detection tool would use, applied at a different point in the pipeline. A detected pattern gets redacted before it ever reaches a log line, a cache key, or a rendered `<ToolCallCard>`/`<CitationText>` component — treated the same way `agent-core`'s internal `TraceStep.raw_tool_args` was already deliberately excluded from the public `AgentEvent` contract (Part 7, Module 2, §6.1), now generalized to catch secrets that weren't anticipated as arguments at all, but simply showed up in content flowing through the system.

```python
def scrub_secrets(content: str) -> str:
    # Applied to tool args, tool results, and retrieved document content
    # BEFORE logging, caching, or rendering — never after.
    for pattern in KNOWN_SECRET_PATTERNS:  # provider key formats, common token shapes
        content = pattern.sub("[REDACTED]", content)
    if shannon_entropy(content) > HIGH_ENTROPY_THRESHOLD:
        # Flag for review rather than auto-redacting a high-entropy string that
        # might just be a hash or an ID — avoid over-redacting legitimate content
        flag_for_review(content)
    return content
```

### 6.6 Configuration as the fifth deployment axis

Module 1 named four axes of AI-specific change (model, prompt, corpus, policy). Credentials and configuration (feature-flag states, rate-limit thresholds, provider routing weights) form a fifth, and it needs the same discipline: versioned independently, rollback-able independently, and — specific to this axis — rotated on a schedule regardless of whether a leak is suspected, not just reactively. Unlike the other four axes, a credential rotation's "canary" isn't a statistical evaluation of output quality; it's a much simpler binary check (did the new credential authenticate successfully, did any in-flight run get stranded), but it still deserves the same rigor Module 1 insisted on: a pre-declared rollback path (the dual-key transition window, §6.3), tested via a drill before it's ever needed in a real incident.

## 7. Mental Models

- **"A leaked LLM provider key isn't a data-access leak — it's a blank check billed to your account, and it looks like your own traffic."**
- **"Scope a tool's credential to its declared action space — if a leak could do more than the tool's design already allows, it's scoped too broadly."**
- **"Secrets don't only leak from your vault — they leak from a scraped page, a pasted ticket, a tool's own output. Scan content, not just config."**
- **"Rotation is a dual-key transition, not a cutover — a paused agent run shouldn't get stranded because a key rotated while it was waiting for a human."**

## 8. Visual Explanation

**Credential categories and their distinct blast radii:**

```
LLM provider keys        → unauthorized billed usage, hard to distinguish from real traffic
Vector DB credentials    → unauthorized read access, scoped by existing access control (Part 4)
MCP server tokens        → varies per connected tool's own capability
Agent tool credentials   → should equal exactly the tool's declared action space, no more
```

**Dual-key rotation window, protecting a paused approval-pending run:**

```
 time ──▶
 credential v1:  [active]────────[deprecated, still valid]────[revoked]
 credential v2:                  [active]───────────────────────────────▶
                                       ▲
                          agent run pauses here (approval_pending)
                          resumes later, still within v1's grace window —
                          authenticates successfully, not stranded
```

**Secret-scrubbing pass, positioned before every downstream sink:**

```
 tool result / retrieved document
            │
            ▼
     scrub_secrets()
      │         │
   clean      flagged/redacted
      │         │
      ▼         ▼
  logging · caching · Part 7 trace/citation rendering
  (only ever receives scrubbed content)
```

## 9. Recommended Resources

1. **Part 1, Module 7 (this handbook)** — re-read directly before starting; this module explicitly extends, rather than replaces, that module's secrets-storage foundation.
2. **HashiCorp Vault documentation — "Dynamic Secrets" and "Lease/Renewal"** — the canonical reference for dual-key/short-lived-credential rotation patterns, directly applicable to §6.3's design regardless of which specific secrets manager `ai-api-platform` uses.
3. **OWASP — "Secrets Management Cheat Sheet"** — a practical checklist covering credential storage, rotation, and least-privilege scoping; cross-check §6.4's tool-credential scoping design against it.
4. **GitHub's secret-scanning documentation (docs.github.com)** — a well-documented reference implementation of pattern-based and entropy-based secret detection, directly relevant to §6.5's `scrub_secrets` design, even though you're applying the same technique to a different pipeline stage (runtime content, not source code).
5. **Part 3, Module 3 (Function/Tool Calling & MCP) and Part 1, Module 9 (Security Fundamentals)** — re-read specifically for the model-requests/app-executes discipline and the prompt-injection risk framing; both are directly relevant background for why tool outputs and retrieved content are an adversarial-content surface, not a trusted one.

## 10. Exercises

1. Enumerate every credential category in `ai-api-platform` as it stands after Part 7 (use §6.1 as a starting checklist, but be exhaustive for your own mental model) and assign each a rotation cadence and an incident-response severity tier.
2. Design the dual-key transition window's grace period (§6.3): what real-world duration should it be, given that an `approval_pending` run (Part 5, Module 6) can legitimately sit paused for an arbitrary amount of time? What's the trade-off in choosing too short a window versus too long one?
3. Take `agent-core`'s `send_email` and `read_calendar` tools (referenced throughout Part 5/7) and design their actual credential scopes concretely — what specific permissions should each carry, and what's the test (per §6.4) that confirms neither is scoped too broadly?
4. A RAG-ingested support ticket (Part 4) contains what looks like an API key in its body. Walk through what `scrub_secrets` should do with it at ingestion time versus at retrieval/citation-render time (Part 7, Module 3) — should it be redacted at ingestion, at render, or both, and why?
5. Design the rotation drill for an LLM provider key: what does the drill need to prove, concretely, beyond "the new key works" — tie your answer back to §6.3's stranded-run failure mode specifically.

## 11. Mini-Project

Build a standalone `scrub_secrets` function (per §6.5) with a small test suite covering: a common API-key format (e.g., a recognizable provider key prefix), a JWT-shaped string, a high-entropy string that should be flagged rather than auto-redacted, and ordinary low-entropy content that should pass through untouched. This isolates the detection logic from the harder job of wiring it into every downstream sink in the Production Project.

## 12. Production Project: `ai-secrets-governance` — Extending `ai-deployment-pipeline`

Extend `ai-api-platform`'s secrets management (building on Part 1, Module 7's foundation) and `ai-deployment-pipeline` (Module 1) with AI-specific governance.

**Scope:**

1. **Full credential inventory** (Exercise 1) documented with rotation cadence and incident-response severity per category, stored as a living document alongside `ai-deployment-pipeline`'s existing ADR log (Part 7, Module 6's pattern, now applied to Part 8).
2. **Dual-key rotation mechanism** (§6.3) implemented for at least the LLM-provider-key and MCP-token categories, with the grace-period design from Exercise 2 justified and documented.
3. **Least-privilege credential scoping audit** (Exercise 3) for every existing `agent-core` tool, confirming each tool's credential matches its declared action space with no excess permission, using the "could a leak do more than the tool already allows" test from §6.4 as the explicit pass/fail criterion.
4. **`scrub_secrets`** from the Mini-Project, wired into every downstream sink identified in §6.5: logging, caching (Part 3, Module 8's cache layers), and Part 7's trace/citation rendering pipeline (extending Module 2's `AgentEvent` emission and Module 3's citation content pipeline to scrub before serialization, not after).
5. **`credential-rotation-drill`**: extends Module 1's `deployment-regression-drill` philosophy to credentials specifically — forces a rotation mid-run against a deliberately-paused `approval_pending` agent action, and confirms the run resumes successfully within the grace window and fails safely (with a clear, actionable error, not a silent hang) if resumed after the window closes.
6. **Fifth-axis integration into `ai-deployment-pipeline`**: credential/config versioning added alongside Module 1's four existing axes, with its own rollback path (the dual-key mechanism) distinct from a feature-flag flip.

**Documentation**: an ADR justifying the dual-key rotation design and the specific grace-period duration chosen; and an incident-response runbook per credential category, distinguishing "revoke immediately, accept some stranded runs" (appropriate for a confirmed active leak) from "rotate on the normal dual-key schedule" (appropriate for routine, non-incident rotation) as two genuinely different response paths.

**Hands off to:** Part 8's disaster-recovery module, which extends this module's credential-availability concerns (what happens if the secrets manager itself is unavailable during a DR event) into the full application-stack DR story; and the compliance/data-privacy module, which will need this module's secret-scanning infrastructure as a building block for handling other categories of sensitive data flowing through tool outputs and retrieved content.

## 13. Practice & Interview Questions

1. Why does a leaked LLM provider API key have a different, and often worse, blast radius than a leaked database credential, even though both are "just a secret"?
2. Explain why credential rotation for a stateful agent system needs a dual-key transition window rather than a hard cutover, and what specific failure mode the hard-cutover approach would create.
3. What's the concrete test for whether an agent tool's credential is scoped correctly, and why is "give the agent one shared service account" the wrong default even if it's operationally simpler?
4. Describe the new secret-leak surface introduced by RAG and tool-calling that wouldn't exist in a traditional web application's threat model, and where in the pipeline it needs to be caught.
5. In an interview: you're asked how you'd manage API keys for a system that calls out to five different LLM providers and a dozen MCP-connected tools. Walk through storage, rotation, and least-privilege scoping, and explain why you wouldn't use one shared credential across all of them.
6. Why does an entropy-based secret detector need a "flag for review" path distinct from auto-redaction, rather than redacting every high-entropy string automatically?

## 14. Common Mistakes

- **Treating all AI-system secrets as one undifferentiated category with one rotation policy.** Different credential types have meaningfully different blast radii and need different cadences and incident-response severity.
- **Hard-cutover credential rotation on a system with long-lived paused states.** Strands `approval_pending` agent runs mid-flow, exactly the resumption guarantee Part 5, Module 6 and Part 7, Module 2 already fought to establish.
- **A single shared "agent service account" credential across multiple tools.** Makes a leak's blast radius equal to everything every tool can do, rather than being bounded by any single tool's declared action space.
- **Assuming tool outputs and retrieved documents are trusted content that can't contain secrets.** They're adversarial or at least unpredictable content, per Part 1, Module 9 and Part 3, Module 3's threat model, and need the same scrutiny as any other untrusted input.
- **Scrubbing secrets only at the logging layer and forgetting caching or the Part 7 rendering pipeline.** A secret redacted from logs but still cached or rendered in a citation preview has only closed one of several leak paths.
- **Auto-redacting every high-entropy string without a review path.** Creates false confidence while potentially destroying legitimate content (hashes, IDs) that merely happens to look random.

## 15. Debugging Exercise

An LLM provider key rotation is performed during a routine maintenance window. Within an hour, several agent runs that were sitting in `approval_pending` before the rotation fail permanently when a human finally approves them — even though the rotation drill (built per this module) passed cleanly in staging the week before.

Form a hypothesis before reading further:

<details>
<summary>Hint 1</summary>
The drill passed in staging. What's different about the *timing* of runs in staging (where the drill deliberately forces a resume within a controlled window) versus real production runs, whose `approval_pending` duration is whatever a real human's response time happens to be?
</details>

<details>
<summary>Hint 2</summary>
Re-read §6.3 and Exercise 2. Was the grace-period duration chosen based on a realistic distribution of real approval-wait times, or based on whatever duration the drill itself happened to test?
</details>

<details>
<summary>Likely root cause</summary>
The dual-key grace period was set to comfortably cover the drill's own test scenario (say, a resume forced within minutes) but was never validated against the actual distribution of real-world `approval_pending` durations, some of which — a human genuinely taking a few hours to review a consequential action — legitimately exceed that window. The drill "passing" only proved the mechanism works within the window it happened to test, not that the window itself was sized correctly against production reality, which is a subtly different claim. This is the credential-rotation instance of a mistake this handbook has now named in several forms: proving a mechanism works is not the same as proving its *parameters* were chosen correctly, the same gap between "the gate exists" and "the gate's thresholds were validated" that Module 1's canary work and Part 7's capstone both had to correct for their own respective mechanisms. The fix is to derive the grace period from actual historical `approval_pending` duration data (a distribution, with a wide safety margin on its tail) rather than an arbitrarily convenient testing duration, and to add a monitoring alert specifically for "a run is approaching the grace window's expiry while still pending" so an operator (via `ops-console`, Part 7, Module 4) gets a chance to intervene before a stranding failure occurs, not just after.
</details>

## 16. Checklist

- [ ] Full credential inventory documented, with rotation cadence and incident-response severity assigned per category
- [ ] Dual-key rotation mechanism implemented for LLM-provider and MCP-token credentials
- [ ] Grace-period duration derived from real `approval_pending` duration data, not an arbitrary or drill-convenient value
- [ ] Every agent tool credential scoped to its declared action space, verified against the "could a leak do more than the tool allows" test
- [ ] `scrub_secrets` wired into logging, caching, and Part 7's trace/citation rendering pipeline — all three, not just one
- [ ] High-entropy content flagged for review rather than blindly auto-redacted
- [ ] `credential-rotation-drill` proves both successful resumption within the grace window and safe (non-silent) failure outside it
- [ ] Credential/config axis integrated into `ai-deployment-pipeline` alongside Module 1's four existing axes
- [ ] Separate incident-response runbooks exist for "confirmed leak" versus "routine scheduled rotation"
- [ ] Monitoring alert exists for a run approaching grace-window expiry while still `approval_pending`

## 17. Summary

`ai-api-platform`'s secrets surface grew, silently, across every part of this handbook since Part 1 — a provider key here, an MCP token there, an agent tool credential somewhere else — without ever being audited as a whole system. This module's job was that audit, plus two genuinely new problems Part 1's original secrets work never had to solve: rotating a credential without stranding a long-paused, stateful agent run, and recognizing that secrets can leak into the system from content it never issued — a pasted ticket, a scraped page, a tool's own output — not just from a config file. Both problems trace back to the same root cause as everything else in Part 8 so far: this system has state and behavior (paused runs, adversarial content flowing through tool calls) that a traditional service's operational playbooks were never designed to anticipate, and closing that gap is what "production" actually means at this layer.

## 18. Next Steps

Reply "continue" for Module 3, or flag anything to go deeper on.
