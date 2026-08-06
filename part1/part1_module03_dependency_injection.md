# PART 1 — Software Engineering
## Module 3: Dependency Injection & Application Structure

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Explain precisely how FastAPI's `Depends()` resolves dependency graphs,
  including nested dependencies, and how its lifecycle differs from
  Spring's IoC container in every important way.
- Design a "composition root" — the single place where concrete
  implementations get wired to abstractions — for a FastAPI application,
  keeping that wiring cleanly separated from business logic.
- Manage dependency scope correctly (per-request vs. singleton-like
  application lifetime) without a framework container doing it for you
  automatically.
- Structure a growing FastAPI application's folders/modules in a way that
  stays navigable as it scales past a toy project.
- Test code that uses dependency injection cleanly, using FastAPI's
  dependency override mechanism for tests.

### 2. Prerequisites
Modules 1–2 (Clean Architecture, Design Patterns) and Part 0 Module 9
(FastAPI).

### 3. Estimated Study Time
6–8 hours over 3–4 days.

### 4. Difficulty
⭐⭐☆☆☆ (Easy — mostly precise application of concepts already covered;
this module is about wiring discipline, not new ideas.)

### 5. Why This Matters
Every project from Part 3 onward wires together multiple adapters,
decorators, repositories, and use cases (Module 1/2's outputs) into a real
running application — this module is where you build a repeatable,
clean way to do that wiring so it doesn't turn into scattered,
hard-to-trace object construction spread across the codebase.

---

### 6. Theory

**What is it?**
Dependency Injection is providing an object's dependencies from the
outside rather than having it construct them itself — you already know
this deeply from Spring. FastAPI's `Depends()` mechanism (introduced in
Part 0 Module 9) is the specific tool; this module goes deeper into how it
actually resolves dependency *graphs* (dependencies of dependencies), and
— critically — how to manage things Spring's container would normally
handle for you automatically (singleton caching, scope).

**How `Depends()` resolves a graph, precisely:**
```python
def get_settings() -> Settings:
    return Settings()   # reads env vars, etc.

def get_llm_client(settings: Settings = Depends(get_settings)) -> Generator:
    return create_generator(settings.llm_provider, settings)  # Factory from Module 2

def get_summarize_use_case(
    llm_client: Generator = Depends(get_llm_client),
    repo: ConversationRepository = Depends(get_repository),
) -> SummarizeDocumentUseCase:
    return SummarizeDocumentUseCase(llm_client, repo)

@router.post("/summarize")
async def summarize_endpoint(
    request: SummarizeRequest,
    use_case: SummarizeDocumentUseCase = Depends(get_summarize_use_case),
):
    return await use_case.execute(request.text)
```
FastAPI walks this dependency tree **per request**, calling each
`Depends()` function in order, resolving nested dependencies first —
`get_settings()` runs, then its result feeds `get_llm_client()`, then that
feeds `get_summarize_use_case()`, and finally your endpoint receives the
fully-constructed use case. This is structurally similar to how Spring
resolves a constructor-injected bean graph — the crucial difference (from
Module 9, worth restating precisely here) is **there is no automatic
singleton caching** — by default, every one of these functions **re-runs
on every single request**, unlike a Spring `@Bean` with default singleton
scope.

**Managing "singleton-like" dependencies without a container:**
For something genuinely expensive to construct per-request (a database
connection pool, an HTTP client with persistent connections — Module 0.4's
connection reuse lesson applies directly here), you don't want it rebuilt
on every request. Options:
```python
# Option A: module-level singleton, constructed once at import/startup time
_llm_client: Generator | None = None
def get_llm_client() -> Generator:
    global _llm_client
    if _llm_client is None:
        _llm_client = create_generator(...)
    return _llm_client

# Option B: FastAPI's `lifespan` context manager (recommended, more
# explicit and testable) — construct expensive resources once at app
# startup, store on `app.state`, and have a Depends() function just read
# from there
@asynccontextmanager
async def lifespan(app: FastAPI):
    app.state.llm_client = create_generator(...)   # built once
    app.state.db_pool = await create_pool(...)      # built once
    yield
    await app.state.db_pool.close()                 # cleanup on shutdown

def get_llm_client(request: Request) -> Generator:
    return request.app.state.llm_client   # cheap, reused every request
```
**Option B (the `lifespan` pattern) is the production-standard approach**
— it's explicit about what's constructed once vs. per-request, and
handles startup/shutdown cleanup properly (closing connection pools
gracefully — tying back to Part 0 Module 3's graceful-shutdown/signal
discussion).

**The "composition root" concept:**
A composition root is the one place in your application where concrete
implementations get chosen and wired to abstractions — typically your
`lifespan` function plus your `Depends()` provider functions, grouped in a
dedicated module (e.g., `app/dependencies.py` or `app/composition.py`).
Business logic (use cases) never constructs its own dependencies or knows
which concrete adapter it's receiving — it only depends on the Protocol
(Module 1). This keeps "what implementation are we actually using" a
decision made in exactly one place, trivially swappable.

**Testing with dependency overrides:**
```python
app.dependency_overrides[get_llm_client] = lambda: FakeGenerator()
# now any test hitting an endpoint that depends on get_llm_client
# receives FakeGenerator() instead — no real network calls, fast and
# deterministic, directly enabling Module 10's testing pyramid at the
# integration-test layer
```
This is FastAPI's direct equivalent of Spring's `@MockBean` — swap real
dependencies for test doubles at the DI layer, without touching the
endpoint or use-case code at all.

---

### 7. Mental Models

**Model 1 — "`Depends()` re-runs every request unless you explicitly cache
it — the opposite default from Spring's singleton-scoped beans."** Always
ask, for each dependency: "is this cheap enough to rebuild every request,
or does it need to be constructed once via `lifespan`?"

**Model 2 — "The composition root is the only place in the codebase that
should know which concrete class implements which abstraction."** If you
find a concrete class name (e.g., `AnthropicAdapter`) imported inside a use
case or route handler instead of only inside the composition root, that's
a Dependency Inversion violation creeping back in.

**Model 3 — "Dependency overrides in tests are how you keep integration
tests fast and deterministic without abandoning the real wiring for
everything else."** Override only the specific external dependency (the
LLM client, in most of your tests) — let the rest of the real wiring run,
so you're still testing real integration where it's cheap to do so
(Module 10's testing pyramid, applied concretely).

---

### 8. Visual Explanation (described)

**Diagram: "Composition root wiring a request"**
```
app startup (lifespan):
  build real DB pool, real LLM client adapter (Module 2's Factory) once
        |
        v
per-request (Depends() graph):
  get_settings() -> get_llm_client() [reads app.state, cheap] -> get_summarize_use_case()
        |
        v
route handler receives a fully-wired SummarizeDocumentUseCase,
with zero knowledge of which concrete adapter backs it
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **FastAPI official docs — "Dependencies" section in full**, especially
   "Classes as Dependencies," "Sub-dependencies," and "Dependencies with
   yield" — read carefully, this is the authoritative reference for
   exactly the mechanics above.
2. **FastAPI official docs — "Lifespan Events"** — the modern
   startup/shutdown pattern (superseding older `@app.on_event("startup")`
   decorators, which you may see in older tutorials/code — prefer
   `lifespan`).
3. **FastAPI official docs — "Testing Dependencies with Overrides"** —
   directly relevant to your testing strategy going forward.

**Official documentation:** fastapi.tiangolo.com (this module leans almost
entirely on the official docs, which are unusually thorough on this exact
topic).

**GitHub repos:** revisit `tiangolo/full-stack-fastapi-template` — study
specifically its `dependencies.py` (or equivalent) and `lifespan`
implementation as a concrete reference composition root.

---

### 17. Exercises

1. Build a small dependency graph (3+ levels deep) and add print/logging
   statements to each provider function to confirm, empirically, that they
   all re-run on every request by default.
2. Convert an expensive-to-construct dependency (simulate with a sleep on
   construction) from being rebuilt per-request to being built once via
   `lifespan`, and measure the latency difference under repeated requests.
3. Write a test using `app.dependency_overrides` to replace a real
   dependency with a fake, confirming the endpoint behaves correctly
   without touching any real external resource.
4. Identify a place in `convo-api` where a concrete class name leaks
   outside the composition root (into a use case or route handler
   directly), and fix it.

### 18. Mini-Project
Refactor `convo-api`'s dependency wiring to use the `lifespan` pattern for
genuinely expensive resources (DB pool, LLM client/HTTP client — reusing
`llm-client-core` from Module 2), with a dedicated `dependencies.py`
module serving as the composition root, and confirm via logging that
expensive resources are built exactly once at startup while
request-scoped dependencies (like the use case itself) are still built
fresh per request.

### 19. Production Project
**Build:** Finalize `convo-api`'s application structure into a clean,
documented, production-grade layout:
```
app/
  entities/          # Module 1: framework-agnostic core objects
  use_cases/          # Module 1: application logic, depends on protocols
  adapters/           # Module 1/2: concrete implementations (repos, LLM adapters)
  api/                # FastAPI routers
  dependencies.py     # composition root: all Depends() providers
  lifespan.py         # startup/shutdown, expensive resource construction
  main.py             # app assembly only — no business logic here
tests/
  unit/               # fast, dependency-overridden or fully mocked
  integration/         # real DB/Redis via Module 5's Compose stack
```
- Full test suite demonstrating both unit tests (dependency-overridden,
  fast) and integration tests (real dependencies via Docker Compose)
- A README with an explicit "how to add a new endpoint / new adapter"
  walkthrough, written as if onboarding a new contributor — a genuinely
  useful open-source-readiness exercise for Part 9's freelancing goals
- Verify (and document, with timing evidence) that expensive resources are
  constructed exactly once per application lifetime, not per request

---

### 20–21. Practice & Interview Questions

1. Explain exactly how FastAPI resolves a multi-level `Depends()` graph
   for a single incoming request.
2. Why doesn't `Depends()` cache/singleton its results by default, and
   what are your two main options for handling expensive dependencies that
   should only be constructed once?
3. What is a composition root, and why should concrete class names
   (specific adapters, specific repository implementations) never appear
   outside it?
4. How do you use `app.dependency_overrides` for testing, and why is this
   preferable to constructing the whole app with real dependencies in
   every test?
5. Compare FastAPI's `Depends()` philosophy (explicit, mechanical, no
   hidden container) to Spring's IoC container (implicit, magic,
   annotation-driven) — what are the genuine trade-offs of each?

---

### 22. Common Mistakes
- Assuming `Depends()` behaves like a Spring singleton bean by default,
  leading to unexpectedly expensive per-request reconstruction of costly
  resources.
- Constructing dependencies inline inside route handlers instead of
  through the composition root, scattering wiring decisions throughout the
  codebase.
- Leaking concrete class references outside the composition root, quietly
  reintroducing the coupling Module 1's Clean Architecture was meant to
  prevent.
- Not using `lifespan` for resource cleanup, leaking connections on
  shutdown.
- Writing tests that spin up the entire real application (real DB, real
  LLM calls) when a dependency override would make the same test faster
  and more deterministic.

### 23. Debugging Exercise
Given a FastAPI app that's noticeably slower under load than expected,
diagnose (via logging added to provider functions, per Exercise 1) that an
expensive resource (e.g., an HTTP client) is being reconstructed on every
single request instead of reused, and fix it by moving construction into
`lifespan`.

---

### 24. Checklist
- [ ] I can explain FastAPI's per-request dependency resolution precisely,
      including why it doesn't cache by default
- [ ] I use `lifespan` correctly for genuinely expensive, once-per-app
      resources
- [ ] I have a clean, documented composition root with no leaked concrete
      references elsewhere
- [ ] I can write tests using `dependency_overrides` confidently
- [ ] `convo-api`'s final structure is clean, tested, and documented for a
      new contributor

### 25. Summary
FastAPI's `Depends()` resolves a per-request dependency graph mechanically
and explicitly, with no hidden container and no automatic singleton
caching — the `lifespan` pattern is how you handle genuinely
once-per-application resources. A disciplined composition root keeps
concrete implementation choices in exactly one place, and dependency
overrides make testing fast and deterministic without abandoning real
integration testing elsewhere. `convo-api` is now a fully, cleanly wired
application ready to be extended with real AI functionality in Part 3.

### 26. Next Steps
Module 4: **Logging, Monitoring & Observability** — instrumenting
everything built so far with structured logging, metrics, and distributed
tracing, since debugging AI systems in production depends heavily on
visibility into non-deterministic model behavior and multi-step
request/agent flows.

---