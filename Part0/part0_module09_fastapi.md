# PART 0 — Prerequisites
## Module 9: FastAPI for Python Backends

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Build production-structured FastAPI services, mapping FastAPI's concepts
  directly onto Spring Boot equivalents you already know deeply.
- Use Pydantic (which you met in Module 1) as FastAPI's request/response
  validation layer, understanding exactly how it replaces Bean Validation.
- Implement dependency injection FastAPI-style (`Depends`), and understand
  precisely how it differs from Spring's container-managed DI.
- Build async endpoints correctly, knowing when `async def` actually buys
  you concurrency and when it doesn't (a common source of subtle bugs).
- Structure a FastAPI project (routers, services, repositories) the way a
  senior engineer would organize a Spring Boot project, adapted to Python
  idiom.

### 2. Prerequisites
Modules 1 (Python), 4 (HTTP/REST), 6 (SQL/Postgres/Redis) — this module
assembles those into a real backend framework.

### 3. Estimated Study Time
10–14 hours over 5–7 days.

### 4. Difficulty
⭐⭐☆☆☆ (Easy-Medium — you already know REST API design deeply from Spring
Boot; this is mostly a "same concepts, different syntax and philosophy"
module.)

### 5. Why This Matters
Nearly every backend service you build from Part 3 onward — LLM
orchestration APIs, RAG endpoints, agent servers — will be FastAPI, because
it's the dominant Python web framework in the AI ecosystem (native async,
Pydantic-based validation, automatic OpenAPI docs). Since you already know
Spring Boot deeply, this module is fast and mostly about **precise mapping**
rather than new concepts — with a few genuine differences (no DI container,
different async semantics) worth understanding carefully so you don't
carry over wrong assumptions.

---

### 6. Theory

**What is it?**
FastAPI is a Python web framework built on **Starlette** (ASGI, the async
equivalent of WSGI) for routing/HTTP, and **Pydantic** for request/response
validation and serialization. It auto-generates OpenAPI/Swagger docs from
your type hints and Pydantic models — a direct analog to
Springdoc/Swagger annotations, except derived automatically instead of via
separate annotations.

**Direct Spring Boot → FastAPI mapping:**

| Spring Boot                         | FastAPI                                             |
|--------------------------------------|------------------------------------------------------|
| `@RestController` + `@RequestMapping`| A Python module with an `APIRouter`                  |
| `@GetMapping("/users/{id}")`         | `@router.get("/users/{id}")`                          |
| `@RequestBody` + a DTO class + Bean Validation | A Pydantic `BaseModel` as a function parameter (validation happens automatically) |
| `@Autowired` / Spring's IoC container| `Depends(...)` — **but there is no container**; see below |
| `@ExceptionHandler`                  | `@app.exception_handler(SomeException)`               |
| `application.yml` / `@ConfigurationProperties` | Pydantic `BaseSettings` reading from env vars |
| Servlet filters / interceptors       | Starlette **middleware**                              |
| `@Transactional`                     | Explicit `async with session.begin():` (no annotation magic — you write the transaction boundary yourself) |
| Springdoc/Swagger annotations        | Automatic, derived from type hints and Pydantic models |

**The most important genuine difference: `Depends()` is not a DI
container.** Spring's `@Autowired` resolves dependencies from a
long-lived, framework-managed container with defined bean scopes
(singleton, prototype, request). FastAPI's `Depends()` is **just a
function that FastAPI calls for you before your endpoint runs, injecting
its return value as a parameter** — there's no container, no bean
lifecycle management, no automatic singleton caching unless you build it
yourself (e.g., via a module-level cached instance or `lru_cache`). This
is a real philosophical difference: FastAPI is deliberately more explicit
and "mechanical" — you can trace *exactly* what gets called and when by
reading the code, with no hidden container magic. Some engineers coming
from Spring find this refreshing; others miss the container's automatic
scope/lifecycle management. Know both trade-offs.

```python
# Dependency: just a function
async def get_db_session():
    async with SessionLocal() as session:
        yield session   # yield-based deps support setup/teardown, like a
                         # context manager — this IS FastAPI's closest
                         # analog to @Transactional-style resource scoping

@router.get("/users/{id}")
async def get_user(id: int, db: AsyncSession = Depends(get_db_session)):
    ...
```

**Async in FastAPI — where the real subtlety lives:**
`async def` endpoints run on the event loop and **do not block other
requests while awaiting I/O** (an LLM API call, a DB query) — this is
genuinely similar in spirit to how non-blocking I/O works in reactive Spring
(WebFlux), and very different from traditional blocking Spring MVC
(request-per-thread). But: **if you call a blocking, synchronous library
function inside an `async def` endpoint (e.g., a non-async DB driver, or
CPU-bound work), you block the entire event loop for every concurrent
request**, not just the one thread handling that request. This is the
single most damaging FastAPI mistake and has no clean Java MVC analog —
in blocking Spring MVC, one slow request blocks one thread; in FastAPI, one
blocking call inside `async def` can stall *the whole process*.

**When to use `def` vs `async def` for an endpoint:**
FastAPI actually runs plain `def` endpoints in a separate thread pool
automatically — so if you're calling a synchronous, blocking library (many
older DB drivers, some SDKs), a plain `def` endpoint is *correct*, not
a mistake, because FastAPI handles the thread-offloading for you. Use
`async def` when everything inside is genuinely async-native (async DB
drivers, `httpx.AsyncClient`, async LLM SDK calls).

---

### 7. Mental Models

**Model 1 — "`Depends()` is a function call FastAPI makes for you, not a
container resolving a bean."** Read any FastAPI dependency by asking "what
function is this, and what does calling it return?" — there's no hidden
lifecycle to reason about beyond that.

**Model 2 — "One blocking call inside `async def` stalls the whole
process, not just one request."** This is the single most important thing
to internalize differently from Spring MVC's thread-per-request model.

**Model 3 — "Pydantic models are your DTOs + Bean Validation, fused into
one class."** The same class defines the shape, the validation rules, and
(via `.model_dump()`) the serialization — less ceremony than
DTO+Validator+Mapper in typical Spring Boot code, at the cost of Python's
looser structure overall.

---

### 8. Visual Explanation (described)

**Diagram: "What happens on a blocking call inside async def"**
```
Correct (async-native I/O):
Request A: await llm_call()  --- yields control, event loop free ---
Request B: await db_query()  --- also running concurrently ---
Request C: await cache_get() --- also running concurrently ---
(all three make progress "at once" on one thread, via cooperative yielding)

Incorrect (blocking call inside async def):
Request A: sync_blocking_db_call()  --- BLOCKS THE ENTIRE EVENT LOOP ---
Request B: (frozen, waiting for A to finish, even though B doesn't depend on A)
Request C: (also frozen)
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **FastAPI official docs (fastapi.tiangolo.com)** — genuinely one of the
   best-written framework docs available; read "Tutorial - User Guide" in
   full, it's dense but efficient, and written by someone (tiangolo) who
   clearly thought hard about pedagogy.
2. **FastAPI docs — "Concurrency and async/await"** section specifically —
   this page directly addresses the blocking-call trap above; read it
   carefully, don't skim.
3. **SQLAlchemy 2.0 async docs** — for the async DB session pattern you'll
   use with the `Depends(get_db_session)` pattern shown above.
4. **Pydantic V2 docs — "Settings Management"** — for `BaseSettings`, your
   `application.yml`/`@ConfigurationProperties` equivalent.

**Official documentation:** fastapi.tiangolo.com (primary reference,
unusually good), docs.pydantic.dev.

**GitHub repos:** `tiangolo/full-stack-fastapi-template` (the official
reference architecture — study its project structure directly), and
`zhanymkanov/fastapi-best-practices` (a widely-referenced community
best-practices repo).

---

### 17. Exercises

1. Build a FastAPI endpoint with a Pydantic request body model that has a
   nested object and a field validator (`@field_validator`) — compare this
   mentally to a Spring `@Valid` DTO with a custom `@Constraint`.
2. Write a `Depends()`-based dependency for a database session using the
   `yield` pattern, and explain exactly when the code after `yield` runs
   (it's the teardown, run after the endpoint returns — directly
   analogous to a `finally` block or a Java try-with-resources close).
3. Deliberately call a blocking `time.sleep(2)` (simulating a blocking I/O
   call) inside an `async def` endpoint, hit it with two concurrent
   requests, and observe that the second request waits for the first to
   fully finish — then fix it by using `await asyncio.sleep(2)` instead
   (or offloading via `def` + FastAPI's automatic thread pool) and observe
   both requests now proceed concurrently.
4. Set up global exception handling for a custom exception type, mapping
   it to a specific HTTP status code and JSON error body — compare to
   `@ExceptionHandler` + `@ControllerAdvice` in Spring.

### 18. Mini-Project
**Build:** A small FastAPI service exposing CRUD endpoints for the
`conversations`/`messages` schema you designed in Module 6, using async
SQLAlchemy, Pydantic request/response models, and a `Depends()`-based
DB session dependency. Include automatic OpenAPI docs (you get this for
free — verify it at `/docs`) and at least one custom exception handler.

### 19. Production Project
**Build:** `convo-api` — a properly structured FastAPI backend for
conversation management, portfolio-quality:
- Layered structure: routers → services → repositories (mirroring
  Controller → Service → Repository from Spring Boot, adapted to Python
  modules/classes)
- Async endpoints backed by async SQLAlchemy against Postgres (from
  Module 6), plus a Redis-backed cache layer (reusing Module 6's caching
  work) for a "get recent conversations" endpoint
- Proper Pydantic settings management (`BaseSettings`) for configuration,
  no hardcoded values
- Global exception handling with structured JSON error responses
- Full test suite (pytest + `httpx.AsyncClient` for endpoint tests),
  including a test that specifically verifies concurrent requests aren't
  blocking each other (timing-based assertion)
- Dockerized (reusing Module 5 skills) with a `docker-compose.yml`
  wiring it to Postgres and Redis
- README documenting the layered architecture and explicitly noting where
  design decisions differ from a typical Spring Boot equivalent, and why

This is the backend service you'll extend with real LLM endpoints starting
in Part 3 — treat its architecture seriously, since you'll be building on
top of it for a long time.

---

### 20–21. Practice & Interview Questions

1. Explain exactly what `Depends()` does and how it differs from Spring's
   `@Autowired`/IoC container — specifically around lifecycle/scope
   management.
2. What happens if you put a blocking, synchronous call inside an
   `async def` endpoint, and why is this worse than the equivalent mistake
   in a traditional blocking Spring MVC controller?
3. When would you write a FastAPI endpoint as plain `def` instead of
   `async def`, and why does that not hurt concurrency the way you might
   expect?
4. How does FastAPI derive its OpenAPI documentation automatically, and
   what's providing the "source of truth" for that schema (Pydantic
   models + type hints)?
5. Design the layered architecture (routers/services/repositories) for a
   FastAPI service exposing a "summarize this document" endpoint that
   calls an external LLM API and caches results — map each layer's
   responsibility explicitly.

---

### 22. Common Mistakes
- Calling blocking/synchronous code inside `async def`, stalling the
  entire event loop for all concurrent requests.
- Assuming `Depends()` gives you Spring-style singleton caching by default
  (it doesn't — a dependency function runs on every request unless you
  explicitly cache/memoize it).
- Putting business logic directly in route handler functions instead of a
  service layer, making the code hard to test and reuse.
- Not using Pydantic `BaseSettings` for configuration, leading to scattered
  `os.environ.get(...)` calls with no validation or central documentation
  of required config.
- Forgetting to close/manage async resources properly outside the
  `yield`-based dependency pattern, leaking DB connections under load.

### 23. Debugging Exercise
Given a FastAPI service that becomes unresponsive under concurrent load
despite low CPU usage, diagnose that a synchronous, blocking library call
(e.g., a non-async HTTP client or DB driver) is running inside an
`async def` endpoint, stalling the event loop — fix by either switching to
an async-native client or converting the endpoint to plain `def` so
FastAPI offloads it to its thread pool.

---

### 24. Checklist
- [ ] I can map every core Spring Boot concept to its FastAPI equivalent
      without hesitating
- [ ] I understand precisely why `Depends()` isn't a DI container, and what
      that means practically
- [ ] I've deliberately caused and fixed the "blocking call inside
      async def" bug
- [ ] I've built `convo-api` with a proper layered architecture, tests, and
      Docker packaging
- [ ] I'm comfortable choosing `def` vs. `async def` deliberately, not by
      habit

### 25. Summary
FastAPI maps closely onto Spring Boot concepts you already know deeply —
routing, DTOs/validation, exception handling, configuration — with two
genuine differences worth real attention: `Depends()` is mechanical
function-calling, not a managed IoC container, and blocking calls inside
`async def` stall the *entire* event loop, not just one thread. `convo-api`
is the real backend service you'll extend with LLM endpoints starting in
Part 3.

### 26. Next Steps
Module 10: **Testing & Debugging Discipline** — formalizing the testing
habits you've been asked to include in every project so far into a real,
deliberate testing strategy (unit, integration, contract tests) across both
your Python and TypeScript code.

---

**Reply "continue" for Module 10, or flag anything to go deeper on.**
