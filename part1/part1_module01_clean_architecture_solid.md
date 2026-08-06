# PART 1 — Software Engineering
## Module 1: Clean Architecture & SOLID Principles

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Apply Clean Architecture's dependency rule to structure Python/FastAPI
  projects the way you already structure Spring Boot applications.
- Recognize and fix SOLID violations in real code, including in AI-specific
  contexts (swappable LLM providers, swappable vector stores) where these
  principles pay off especially clearly.
- Explain *why* Clean Architecture matters more, not less, in AI systems —
  because model providers, vector DBs, and frameworks change fast, and
  your business logic shouldn't be coupled to any of them.
- Judge when strict layering is worth the ceremony and when it's
  over-engineering for a project's actual scale — an explicit design
  judgment, not a rule to apply blindly everywhere.

### 2. Prerequisites
Part 0 in full, especially Module 9 (FastAPI) and Module 1 (Python).

### 3. Estimated Study Time
8–10 hours over 4–5 days. This module should move fast for you — it's
mostly re-expressing concepts you already apply daily in Java, in Python's
idiom.

### 4. Difficulty
⭐⭐☆☆☆ (Easy for you — you already think this way; the work here is
translation and AI-specific application, not new concepts.)

### 5. Why This Matters
Every AI system in this handbook will need to swap providers (Anthropic ↔
OpenAI ↔ local models), swap vector databases, and evolve rapidly as the
ecosystem changes. Clean Architecture's dependency inversion is *the*
mechanism that makes those swaps cheap instead of a rewrite — arguably even
more valuable here than in typical backend work, since the AI tooling
landscape genuinely does change this fast.

---

### 6. Theory

**What is it?**
Clean Architecture (Robert C. Martin) organizes code into concentric
layers, with a single rule: **dependencies point inward only.** Inner
layers (business logic/entities, use cases) know nothing about outer
layers (frameworks, databases, external APIs). Outer layers depend on
inner layers, never the reverse.

```
[ Frameworks & Drivers (FastAPI routes, DB drivers, LLM SDK clients) ]
        depends on
[ Interface Adapters (repositories implementing interfaces, DTOs) ]
        depends on
[ Use Cases (application-specific business logic) ]
        depends on
[ Entities (core business objects/rules, framework-agnostic) ]
```

**Why do we need it (AI-specific framing)?**
Imagine your "summarize this document" use case directly imports
`anthropic.Anthropic()` and calls it inline. Six months later you want to
add OpenAI as a fallback, or swap to a self-hosted model (Part 6). If your
business logic is directly coupled to one SDK, this is a rewrite. If your
use case depends on an **abstraction** (`class Summarizer(Protocol): def
summarize(text: str) -> str`), and a separate adapter implements that
abstraction using whichever SDK, swapping providers means writing one new
adapter class — your use case code doesn't change at all.

**SOLID, translated directly (you already know these — this is precise
Python framing, with AI-specific examples):**

**S — Single Responsibility Principle.** A class/module should have one
reason to change. AI-specific example: a class that both calls an LLM
*and* parses/validates its output *and* writes to a database has three
reasons to change (SDK version bump, output format change, schema change)
— split it into three collaborating pieces.

**O — Open/Closed Principle.** Open for extension, closed for
modification. AI-specific example: adding a new supported LLM provider
should mean adding a new class implementing your `Summarizer`/`Generator`
protocol, not editing a giant `if provider == "anthropic": ... elif
provider == "openai": ...` conditional block scattered through your
codebase.

**L — Liskov Substitution Principle.** Any implementation of an interface
should be substitutable without breaking callers. AI-specific example: if
your `VectorStore` protocol's `search()` method is documented to return
results sorted by relevance descending, *every* implementation (Qdrant,
pgvector, Pinecone) must actually honor that, or callers relying on sort
order will silently break when you swap implementations.

**I — Interface Segregation Principle.** Prefer several small, specific
interfaces over one large one. AI-specific example: don't force every
`VectorStore` implementation to implement `delete_by_metadata_filter()` if
only some backends support it — split into a smaller core `VectorStore`
protocol plus an optional `SupportsMetadataDeletion` protocol that only
capable implementations provide.

**D — Dependency Inversion Principle.** High-level modules (use cases)
should depend on abstractions, not low-level modules (concrete SDK
clients) — and those abstractions should be owned by the high-level
module, not the low-level one. This is the mechanism underlying everything
above, and it's exactly what FastAPI's `Depends()` (Module 9) is a
*mechanical tool* for — but DIP itself is a design principle you apply
regardless of framework.

**Python idiom for these principles — `Protocol` vs. ABC:**
```python
from typing import Protocol

class Summarizer(Protocol):
    async def summarize(self, text: str) -> str: ...

class AnthropicSummarizer:
    def __init__(self, client): self._client = client
    async def summarize(self, text: str) -> str:
        # calls self._client...
        ...

class OpenAISummarizer:
    def __init__(self, client): self._client = client
    async def summarize(self, text: str) -> str:
        # calls self._client differently...
        ...

# Use case depends on the Protocol, not either concrete class:
class SummarizeDocumentUseCase:
    def __init__(self, summarizer: Summarizer):
        self._summarizer = summarizer
    async def execute(self, text: str) -> str:
        return await self._summarizer.summarize(text)
```
`Protocol` gives you structural typing (duck typing, checked by static
analysis) — a class doesn't need to explicitly inherit from `Summarizer` to
satisfy it, unlike Java's explicit `implements`. This is a genuine, useful
Python-idiom difference: you can retrofit an existing class to satisfy a
protocol with zero changes to that class, as long as its method
signatures match.

**When is strict layering overkill?**
For a small script or a genuinely single-purpose CLI tool, four formal
layers is ceremony without payoff. The judgment call: **apply Clean
Architecture's dependency-inversion discipline (depend on abstractions at
integration points that are likely to change or need swapping — LLM
providers, vector stores, payment providers) without necessarily building
the full four-layer folder structure for a small project.** The principle
(depend on abstractions where volatility is expected) is what matters;
the ceremony (exact folder layout) is negotiable based on project size.

---

### 7. Mental Models

**Model 1 — "Depend on what's stable, isolate what's volatile."** Model
providers, specific vector DBs, and specific frameworks are volatile (they
change, get swapped, get upgraded). Your core business rules (how a
conversation should be summarized, what "relevant" means for your RAG
system) are comparatively stable — structure dependencies so volatile
things depend on stable things, never the reverse.

**Model 2 — "A Protocol is an interface Java would make you declare
`implements` for — Python just checks the shape instead of the
declaration."** This is a real productivity gain once internalized: you
can adapt existing code to fit an abstraction after the fact.

**Model 3 — "SOLID isn't a checklist to satisfy everywhere uniformly —
it's a set of pressure-release valves you reach for exactly where change
is likely."** Apply DIP hardest at your genuine integration boundaries
(LLM providers, vector stores, databases); don't over-abstract stable,
unlikely-to-change internal logic just for the sake of "following SOLID."

---

### 8. Visual Explanation (described)

**Diagram: "Dependency direction with an LLM provider swap"**
```
[ FastAPI route ]
       |  depends on
       v
[ SummarizeDocumentUseCase ]   <- core business logic, framework-agnostic
       |  depends on (abstraction)
       v
[ Summarizer Protocol ]
       ^  implemented by
       |
[ AnthropicSummarizer ]  or  [ OpenAISummarizer ]  or  [ LocalModelSummarizer ]
  (Part 3)                     (Part 3)                  (Part 6)
```
The use case never imports `anthropic` or `openai` directly — swapping
providers means writing a new box on the bottom row and wiring it in via
`Depends()`, with zero changes to the use case or route.

---

### 9–16. Recommended Resources

**Reading order:**
1. **"Clean Architecture" by Robert C. Martin** — the source text; read the
   chapters on the dependency rule and the specific SOLID chapters — you
   likely already grasp most of this from Java experience, so skim
   confidently rather than reading cover to cover.
2. **Python `typing.Protocol` official documentation** — the specific
   mechanism you'll use to express these interfaces in Python.
3. **"Architecture Patterns with Python" by Percival & Gregory (free
   online, "cosmicpython.com")** — an excellent, Python-specific treatment
   of Clean Architecture / hexagonal architecture ideas, directly
   applicable to FastAPI projects.

**Official documentation:** docs.python.org's `typing` module docs for
`Protocol`.

**GitHub repos:** `tiangolo/full-stack-fastapi-template` (revisit from
Module 9, now looking specifically at how it separates concerns), and any
well-regarded "hexagonal architecture Python" example repo for a concrete
reference layout.

---

### 17. Exercises

1. Take a piece of code that directly calls a specific SDK inline inside
   business logic, refactor it to depend on a `Protocol` instead, and
   write two interchangeable implementations satisfying that protocol.
2. Identify a Single Responsibility violation in one of your Part 0
   projects (e.g., `convo-api`) and split it into properly separated
   classes/modules.
3. Write a small example demonstrating a Liskov Substitution violation
   (an implementation that doesn't honor its interface's implicit
   contract) and explain concretely what breaks for callers.
4. Design (on paper) the abstraction boundary you'd introduce for
   supporting multiple vector database backends in Part 4, before you've
   built any of them — decide what belongs in the core `VectorStore`
   protocol vs. an optional capability protocol.

### 18. Mini-Project
Refactor `convo-api` (from Part 0, Module 9) to depend on a
`ConversationRepository` protocol instead of directly importing SQLAlchemy
session logic into your service layer — provide one real
(SQLAlchemy-backed) implementation and one in-memory fake implementation
(useful for fast unit tests without a real database, tying back to Module
10's testing pyramid).

### 19. Production Project
**Build:** Formalize `convo-api`'s architecture into clean, documented
layers:
- `entities/` — core domain objects, no framework imports at all
- `use_cases/` — application logic, depends only on protocols defined
  alongside or above it
- `adapters/` — concrete implementations (SQLAlchemy repository, Redis
  cache adapter) implementing the protocols
- `api/` — FastAPI routes, wiring concrete adapters to use cases via
  `Depends()`
- An architecture decision record (`docs/adr/0001-clean-architecture.md`)
  explaining *why* you introduced this structure now, what problem it
  solves for this specific project, and explicitly noting where you
  deliberately did **not** over-abstract (per Model 3 above) — this kind
  of documented judgment call is exactly what distinguishes a senior-level
  portfolio project from a rule-following one.

---

### 20–21. Practice & Interview Questions

1. Explain Clean Architecture's dependency rule and why it matters more,
   not less, in a system that integrates with fast-changing AI provider
   SDKs.
2. Give a concrete example of each SOLID principle being violated and
   fixed, ideally with an AI-system-relevant example (provider swapping,
   vector store swapping).
3. What's the difference between Python's `Protocol` (structural typing)
   and explicit interface inheritance (nominal typing, like Java's
   `implements`)? When would you prefer one over the other in Python?
4. When is strict four-layer Clean Architecture overkill, and how do you
   decide where to apply DIP versus where it's unnecessary ceremony?
5. Walk through, step by step, how you'd add support for a second LLM
   provider to a properly layered codebase versus a codebase with SDK
   calls scattered through business logic.

---

### 22. Common Mistakes
- Coupling business logic directly to a specific SDK/vector DB client,
  making future swaps expensive.
- Over-applying Clean Architecture ceremony to small scripts/projects
  where it adds friction without payoff.
- Violating Liskov Substitution by writing an interface implementation
  that doesn't honor the interface's implicit behavioral contract (e.g., a
  different sort order, different error behavior).
- Building one giant interface (violating Interface Segregation) instead
  of several smaller, purpose-specific ones.
- Confusing "using `Depends()`" (a FastAPI mechanism) with "having
  properly inverted dependencies" (a design principle) — you can use
  `Depends()` while still being tightly coupled if you inject a concrete
  class instead of an abstraction.

### 23. Debugging Exercise
Given a codebase where switching from one LLM provider to another required
changes in 6 different files across business logic, given that logic was
directly importing and calling the SDK inline, refactor it toward a
protocol-based abstraction and confirm the same swap now requires changing
only the dependency-injection wiring (one line) and adding one new adapter
class.

---

### 24. Checklist
- [ ] I can state Clean Architecture's dependency rule precisely and
      explain its AI-specific value
- [ ] I can identify and fix violations of each SOLID principle in real
      code
- [ ] I understand `Protocol`'s structural typing and when to use it over
      nominal inheritance
- [ ] I've refactored `convo-api` into a properly layered structure with a
      documented ADR
- [ ] I can judge when strict layering is worth it vs. overkill, and can
      articulate why

### 25. Summary
Clean Architecture and SOLID are concepts you already apply daily in
Java — this module was about precise Python idiom (`Protocol` for
structural interfaces) and, more importantly, recognizing why dependency
inversion pays off especially strongly in AI systems, where model
providers, vector databases, and frameworks change fast enough that
coupling your business logic to any one of them is a real, recurring
liability.

### 26. Next Steps
Module 2: **Design Patterns for AI/Backend Systems** — the concrete
patterns (Strategy, Factory, Adapter, Decorator, Observer) that implement
the SOLID principles from this module, with specific AI-system
applications for each (Strategy for swappable prompting strategies,
Factory for provider instantiation, Adapter for wrapping heterogeneous
provider SDKs behind one interface).

---
