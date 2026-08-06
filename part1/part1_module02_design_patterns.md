# PART 1 — Software Engineering
## Module 2: Design Patterns for AI/Backend Systems

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Apply Strategy, Factory, Adapter, Decorator, and Observer patterns
  fluently in Python, mapping each directly onto Java patterns you already
  know.
- Recognize which pattern fits which AI-system problem (provider swapping,
  prompt-strategy swapping, response post-processing pipelines, event
  notification) without over-engineering a simple problem into an
  unnecessary pattern.
- Understand Python-specific idiom that sometimes makes a "pattern"
  unnecessary (e.g., first-class functions often replace a full Strategy
  class hierarchy) — knowing when *not* to reach for a formal pattern is as
  important as knowing the pattern itself.
- Build a real "pluggable LLM provider" abstraction using these patterns
  together — the concrete deliverable that Part 3 will build directly on
  top of.

### 2. Prerequisites
Module 1 (Clean Architecture/SOLID) — these patterns are concrete
implementations of the SOLID principles from that module.

### 3. Estimated Study Time
8–10 hours over 4–5 days.

### 4. Difficulty
⭐⭐☆☆☆ (Easy — you know all of these from the Gang of Four patterns in
Java; the work here is Python idiom and AI-specific application.)

### 5. Why This Matters
Part 3 (LLM Engineering) requires a provider-agnostic client abstraction
(you'll want to swap between Anthropic, OpenAI, and local models). Part 4
(RAG) requires pluggable chunking/embedding/retrieval strategies. Part 5
(Agents) requires an event/observer model for agent lifecycle hooks. This
module builds the concrete pattern vocabulary and one real reusable
abstraction that all three parts will extend.

---

### 6. Theory

**Strategy Pattern — swappable algorithms behind one interface:**

Java version (familiar): an interface with multiple implementations,
selected/injected at runtime.

Python version — two idiomatic options:
```python
# Option A: Protocol-based (like a Java interface — use when you have
# genuinely different classes with state/setup, e.g., different SDK clients)
class PromptStrategy(Protocol):
    def build_prompt(self, context: dict) -> str: ...

# Option B: just pass a function (Pythonic — use when the "strategy" is
# genuinely stateless logic, no setup/config needed)
def build_concise_prompt(context: dict) -> str: ...
def build_detailed_prompt(context: dict) -> str: ...

def summarize(text: str, prompt_builder: Callable[[dict], str]) -> str:
    prompt = prompt_builder({"text": text})
    ...
```
**AI-specific application:** swapping between prompting strategies
(concise vs. detailed vs. chain-of-thought) for the same underlying task —
exactly what you'll build in Part 3 when comparing prompt-engineering
approaches, and what you'll formalize into an evaluation harness in Part 3
using Module 10's testing patterns.

**Factory Pattern — centralizing object creation logic:**

```python
def create_summarizer(provider: str, config: SummarizerConfig) -> Summarizer:
    match provider:
        case "anthropic": return AnthropicSummarizer(config)
        case "openai": return OpenAISummarizer(config)
        case "local": return LocalModelSummarizer(config)
        case _: raise ValueError(f"Unknown provider: {provider}")
```
**Why this matters specifically for AI systems:** provider selection is
often driven by runtime configuration (an environment variable, a
feature flag, a fallback chain) rather than compile-time knowledge — a
factory centralizes "how do we decide which concrete implementation to
build" in one place, so the rest of the codebase just asks for "a
Summarizer" without knowing or caring which provider backs it (directly
using the `Summarizer` Protocol from Module 1).

**Adapter Pattern — making incompatible interfaces compatible:**

This is *the* pattern for wrapping heterogeneous provider SDKs behind one
common interface — Anthropic's SDK, OpenAI's SDK, and a local model's
inference API all have genuinely different method signatures and response
shapes. An Adapter wraps each one, translating its specific shape into
your app's common `Summarizer`/`Generator` protocol:

```python
class AnthropicAdapter:
    def __init__(self, client: anthropic.Anthropic):
        self._client = client
    async def summarize(self, text: str) -> str:
        response = await self._client.messages.create(...)
        return response.content[0].text   # translating Anthropic's specific
                                            # response shape into a plain string

class OpenAIAdapter:
    def __init__(self, client: openai.AsyncOpenAI):
        self._client = client
    async def summarize(self, text: str) -> str:
        response = await self._client.chat.completions.create(...)
        return response.choices[0].message.content  # different shape, same
                                                        # translated output
```
**This is precisely what you'll build for real in Part 3** — a unified
LLM client interface with Adapters for each provider, so the rest of your
application never touches provider-specific response shapes directly.

**Decorator Pattern — adding behavior without modifying the original
class:**

```python
class CachingSummarizerDecorator:
    def __init__(self, wrapped: Summarizer, cache: Redis):
        self._wrapped = wrapped
        self._cache = cache
    async def summarize(self, text: str) -> str:
        key = hash_key(text)
        if cached := await self._cache.get(key):
            return cached
        result = await self._wrapped.summarize(text)
        await self._cache.set(key, result)
        return result
```
**AI-specific application:** this is literally how you'd add caching
(Part 0 Module 6), retry/backoff (Part 0 Module 4), or logging/tracing
(this module, later section) to *any* `Summarizer` implementation without
modifying it — stack decorators (`LoggingDecorator(CachingDecorator(
AnthropicAdapter(...)))`) to compose cross-cutting concerns cleanly,
directly mirroring how you'd stack Spring AOP-style concerns or Servlet
filters in Java, just done explicitly here rather than via
annotations/framework magic.

**Observer Pattern — event notification without tight coupling:**

```python
class AgentEventBus:
    def __init__(self):
        self._listeners: list[Callable[[AgentEvent], None]] = []
    def subscribe(self, listener): self._listeners.append(listener)
    def publish(self, event: AgentEvent):
        for listener in self._listeners:
            listener(event)
```
**AI-specific application:** Part 5 (Agents) needs to notify multiple
independent concerns (logging, a UI update via streaming, a cost tracker,
an evaluation harness) whenever an agent takes an action or a tool call
completes — Observer decouples "the agent doing things" from "everything
that needs to react to those things," so adding a new listener (e.g., a
new cost-tracking dashboard) never requires touching the agent's core
loop.

**When should you NOT reach for a formal pattern?**
Python's first-class functions, closures, and duck typing often replace
what would be a multi-class pattern hierarchy in Java. If a "Strategy" is
genuinely just a stateless function, don't build a class hierarchy for it
— pass the function directly (Option B in the Strategy section above).
Reaching for heavyweight OOP patterns where a plain function suffices is a
common over-engineering mistake when translating Java instincts into
Python.

---

### 7. Mental Models

**Model 1 — "Adapter unifies incompatible shapes; Decorator adds behavior
to a compatible shape."** Don't confuse them: Adapter is about translation
(different SDK response formats → one common interface); Decorator is
about composition (stacking cross-cutting concerns onto an already-
compatible interface).

**Model 2 — "In Python, ask whether you need a class at all before
reaching for a Java-style pattern class hierarchy."** A function is often
a complete, sufficient Strategy.

**Model 3 — "Observer is how you keep an agent's core loop simple while
still supporting many independent side-concerns (logging, cost tracking,
UI updates) — without the core loop needing to know any of them exist."**

---

### 8. Visual Explanation (described)

**Diagram: "Decorator stacking for a Summarizer"**
```
LoggingDecorator
   wraps
CachingDecorator
   wraps
RetryWithBackoffDecorator
   wraps
AnthropicAdapter (the real provider call)
```
A call to `.summarize(text)` flows through each decorator layer in turn —
logging records the call, caching short-circuits on a hit, retry/backoff
handles transient failures, and only the innermost layer actually talks to
the network — each concern fully separated, composable, and testable in
isolation.

---

### 9–16. Recommended Resources

**Reading order:**
1. **"Design Patterns: Elements of Reusable Object-Oriented Software" (Gang
   of Four)** — you likely already know this book's content from Java; use
   it as a reference for precise pattern definitions, not a full re-read.
2. **"Python Design Patterns" — Brandon Rhodes (python-patterns.guide)** —
   specifically valuable for its honest discussion of which classic
   patterns are Python-idiomatic and which are unnecessary in a language
   with first-class functions and duck typing.
3. **Refactoring.Guru's pattern catalog** — clear, concise, illustrated
   summaries with both Java and Python code examples side by side, good
   for quick cross-referencing.

**Official documentation:** Python's `typing` docs for `Protocol` (Module
1) and `Callable` type hints, used throughout this module's examples.

**GitHub repos:** the Vercel AI SDK and LangChain's Python SDK both use
Adapter-pattern-style provider abstractions extensively — worth skimming
their provider-integration code once you've completed this module, to see
these exact patterns in a widely-used real codebase.

---

### 17. Exercises

1. Implement the Strategy pattern both ways (Protocol-based class
   hierarchy, and plain function-based) for a "text formatting strategy"
   problem, and write a short comparison of when you'd choose each.
2. Build two Adapter classes wrapping two different (mocked) "provider
   SDKs" with genuinely different method signatures/response shapes,
   behind one common `Protocol`.
3. Stack three Decorators (logging, caching, retry) around a mock
   `Summarizer` implementation, and write a test confirming each layer's
   behavior independently (e.g., confirm caching actually short-circuits
   the wrapped call on a hit).
4. Build a minimal `EventBus` (Observer pattern) and demonstrate two
   independent listeners (e.g., a logger and a counter) both reacting to
   the same published event stream without knowing about each other.

### 18. Mini-Project
Refactor the `Summarizer` protocol and its (mocked, so far) implementations
from Module 1 to use the Factory pattern for instantiation based on a
config value, and wrap the chosen implementation with Caching and
Logging Decorators before it's injected into your use case via
`Depends()`.

### 19. Production Project
**Build:** `llm-client-core` — a standalone, reusable Python package (not
just a module inside one project — structure it as its own installable
package, reusing Module 1/11 skills) implementing:
- A core `Generator` Protocol (`async def generate(prompt: str, **kwargs)
  -> GenerationResult`)
- Adapter implementations for at least two mock "providers" with
  deliberately different internal response shapes (you'll swap these for
  real Anthropic/OpenAI adapters in Part 3 with minimal changes)
- A Factory function selecting the adapter based on config
- Caching, retry-with-backoff, and logging Decorators, composable in any
  order
- An `EventBus` that Decorators/Adapters can publish events to (call
  started, call succeeded, call failed, cache hit) — with at least one
  real listener (e.g., a simple in-memory cost/call counter)
- Full test suite covering each pattern in isolation and the composed
  whole
- A README explicitly diagramming the pattern usage and explaining the
  design decisions

This package is the **literal foundation for Part 3's real LLM
integration** — in Part 3 you'll write real Anthropic/OpenAI adapters
implementing this exact `Generator` protocol, and everything else (caching,
retry, logging, cost tracking) carries over unchanged.

---

### 20–21. Practice & Interview Questions

1. Explain the difference between Adapter and Decorator, using an
   AI-provider example for each.
2. When would you use a Python function instead of a full Strategy class
   hierarchy, and what signals tell you a function is genuinely
   sufficient?
3. Why does a Factory pattern matter more in a system where the concrete
   implementation is chosen by runtime configuration (e.g., an environment
   variable selecting the LLM provider) rather than compile-time code?
4. Design a decorator stack (order matters — explain why) for a
   `Generator` that needs caching, retry-with-backoff, and rate-limiting —
   what order would you stack them in and why?
5. Explain the Observer pattern's value for an agent system that needs
   logging, cost tracking, and UI streaming updates, all reacting to the
   same underlying events, without the agent's core loop knowing about any
   of them.

---

### 22. Common Mistakes
- Building a full class hierarchy for a Strategy that's genuinely just a
  stateless function — unnecessary ceremony in Python.
- Confusing Adapter (translating incompatible shapes) with Decorator
  (adding behavior to a compatible shape), leading to muddled
  responsibilities in a single class.
- Hardcoding provider selection logic (`if/elif` chains) scattered through
  the codebase instead of centralizing it in one Factory.
- Stacking decorators in an order that doesn't make sense (e.g., caching
  *outside* a retry decorator would cache a value the retry logic hasn't
  actually stabilized on yet, if not designed carefully — think through
  decorator order deliberately, don't assume any order works).
- Building an Observer/EventBus with tight coupling anyway (listeners that
  reach back into the publisher's internals instead of only reacting to
  the published event data).

### 23. Debugging Exercise
Given a decorator stack where caching returns stale error responses
(because a failed call's error was cached as if it were a valid result),
diagnose the bug (the caching decorator's placement relative to the retry
decorator, or its lack of error-case handling) and fix the ordering/logic
so only successful results are cached.

---

### 24. Checklist
- [ ] I can implement Strategy both ways (class-based and function-based)
      and choose correctly between them
- [ ] I've built real Adapter classes unifying incompatible interfaces
- [ ] I understand Decorator composition and can reason about stacking
      order
- [ ] I've built a working Observer/EventBus with independent listeners
- [ ] I've completed `llm-client-core`, ready to extend with real provider
      adapters in Part 3

### 25. Summary
Strategy, Factory, Adapter, Decorator, and Observer are patterns you
already know from Java — this module's value was precise Python idiom
(knowing when a function replaces a class hierarchy), and building one
real, reusable artifact, `llm-client-core`, that directly implements the
"pluggable, swappable AI provider" architecture that Module 1's Clean
Architecture principles called for. Part 3 extends this package with real
provider integrations rather than starting from scratch.

### 26. Next Steps
Module 3: **Dependency Injection & Application Structure** — going deeper
on FastAPI's `Depends()` mechanics from Part 0 Module 9, composition roots,
and how to wire together everything built so far (repositories, adapters,
decorators, use cases) into a coherent, testable application structure.

---
