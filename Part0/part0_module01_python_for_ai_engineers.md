# PART 0 — Prerequisites
## Module 1: Python for AI Engineers

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Write idiomatic, type-hinted Python that a senior AI engineering team would
  accept in code review.
- Use Python's data model (iterators, context managers, dataclasses,
  protocols) to build clean abstractions instead of copy-pasted scripts.
- Understand exactly where Python's performance ceiling is, why AI workloads
  route around it (C/C++/Rust extensions, vectorization), and how to profile
  before optimizing.
- Structure a Python project the way production AI codebases are structured
  (src layout, pyproject.toml, dependency locking).
- Read someone else's ML/AI Python code (numpy/pandas-heavy, async, or
  decorator-heavy) without getting lost.

### 2. Prerequisites
None formally, but this module assumes general programming fluency (which you
have from Java/Spring). We'll repeatedly compare Python idioms to their Java
equivalents so the transfer is fast.

### 3. Estimated Study Time
10–14 hours over 5–7 days at 2–3 hrs/day.

### 4. Difficulty
⭐⭐☆☆☆ (Easy-Medium for someone with your background — the challenge is
un-learning some Java instincts, not learning to code.)

### 5. Why This Matters
Python is the lingua franca of AI engineering: every major model API SDK
(OpenAI, Anthropic, Google), every vector DB client, every agent framework
(LangChain, LlamaIndex, CrewAI), and nearly all research code targets Python
first. You don't need to become a Python savant — you need enough fluency that
the language gets out of your way while you focus on system design, which is
your actual strength.

---

### 6. Theory

**What is it?**
Python is a dynamically-typed, interpreted, multi-paradigm language. For AI
engineering specifically, three properties matter more than others:
1. **A huge, mature numerical/ML ecosystem** (numpy, pandas, PyTorch, and
   every LLM SDK) — this is *why* Python won the AI race, not because the
   language itself is fast.
2. **A "glue language" design philosophy** — Python is optimized for wiring
   together components (a model call, a database, an API), not for raw
   compute. The compute-heavy work happens in C/C++/CUDA underneath.
3. **Duck typing + protocols** — Python cares whether an object *behaves*
   correctly (has the right methods), not what class it is. This is different
   from Java's nominal typing and takes a moment to adjust to.

**Why do we need it (vs. Java, which you already know)?**
You could technically build AI systems in Java (and companies do, for the
serving/infra layer — more on this in Part 6). But:
- Model provider SDKs, research reproductions, and the fastest-moving OSS
  tooling (LangChain, Hugging Face, vLLM, Ray) ship in Python first, often
  Python-only.
- Python's REPL-driven, notebook-friendly workflow matches how AI systems are
  actually developed: iterate on a prompt or a chunking strategy, look at
  output immediately, adjust.
- For your platform-engineering goal specifically: the infra you'll build
  (Kubernetes operators, Terraform, Go/Rust services) may not be Python at
  all — but you'll always need Python fluency to understand what you're
  serving and to write tooling/automation around it.

**How does it work? (Mental model, not memorization)**

Think of Python's object model as "everything is a dictionary of attributes,"
even things that look primitive:

```
x = 5          # x is a name bound to an int object
x.__class__    # <class 'int'> — even ints are objects
x + 3          # actually calls x.__add__(3) under the hood
```

This is the single idea that explains operator overloading, dunder methods,
duck typing, and why Python lets you do things like `len(obj)` on any object
that defines `__len__`. Compare to Java, where `int` is a primitive and
operator behavior is fixed by the language — in Python, *you* can define what
`+` means for your own class.

**Java → Python translation table (the fast path for you):**

| Java concept                     | Python equivalent                              |
|-----------------------------------|-------------------------------------------------|
| Interface                        | `Protocol` (structural) or ABC (nominal)        |
| `final` fields / immutability    | `@dataclass(frozen=True)`, tuples, `NamedTuple` |
| Checked exceptions                | Exceptions are all unchecked; use type hints + docstrings to signal what can raise |
| Generics `<T>`                   | `TypeVar`, `Generic[T]`, or just duck typing     |
| `try-with-resources`             | context managers (`with open(...) as f:`)        |
| Streams API (`.map().filter()`)  | list/generator comprehensions, `itertools`       |
| `CompletableFuture`              | `asyncio` coroutines + `asyncio.gather`          |
| Maven/Gradle                     | `pyproject.toml` + `uv` or `poetry`              |
| `@Autowired` / DI container      | Explicit constructor injection (Python has no standard DI container — this is a real difference, and a good one to internalize) |
| Strong static typing (compiler-enforced) | Type hints are **optional and unenforced at runtime** — `mypy`/`pyright` catch errors, but nothing stops bad types at runtime unless you validate (e.g., Pydantic) |

That last row is the biggest mental shift: **Python's type hints are a
documentation and tooling layer, not a runtime guarantee.** This is why
Pydantic (validates data at runtime) is everywhere in AI Python code — it's
compensating for what Java's type system gives you for free.

---

### 7. Mental Models

**Model 1 — "Python is slow, but your AI code usually isn't Python."**
When you call `model.generate()` or do a matrix multiply, you're calling into
C/CUDA. Python is the orchestration layer, not the hot loop. The exception:
if you write a manual `for` loop over millions of rows instead of using
vectorized numpy/pandas operations, *then* you feel Python's real interpreter
overhead. Rule of thumb: if you're iterating over numeric data row-by-row in
pure Python, you're doing it wrong.

**Model 2 — "Duck typing is Java's Liskov Substitution Principle, informally
enforced by convention instead of the compiler."**
You already understand LSP from SOLID. Python just doesn't check it for you.

**Model 3 — "A Python project's dependency file is its build.gradle, but
much more fragile without a lockfile."**
This is why modern Python tooling (`uv`, `poetry`) obsesses over lockfiles —
Python historically had weaker reproducibility guarantees than Maven/Gradle,
and the ecosystem is actively fixing this.

---

### 8. Visual Explanation (described)

**Diagram: "Where Python sits in an AI system's latency budget"**
```
[ Your Python orchestration code ]
        |
        | (network call, ~100-2000ms)
        v
[ Model Provider API (Anthropic/OpenAI) ] --- runs in optimized C++/CUDA
        |
        | (network call, ~5-50ms)
        v
[ Vector DB / Postgres / Redis ]  --- also C/C++ under the hood
```
The takeaway: your Python code's own execution time (microseconds to low
milliseconds) is almost never the bottleneck in an AI application. Network
I/O and model inference dominate. This should reframe how you think about
"performance" in this domain — it's about concurrency and caching (things you
already know from backend engineering), not micro-optimizing Python loops.

---

### 9–16. Recommended Resources (with reasoning)

**Reading order:**
1. **Official Python Tutorial** (docs.python.org/3/tutorial) — skim, don't
   read cover to cover. You know programming; you need Python's specific
   syntax and idioms, not "what is a variable."
2. **"Python's Class Development Toolkit" — Raymond Hettinger (PyCon talk)** —
   the best explanation of the object model mental shift above.
3. **Real Python's "Python Type Checking" guide** — because type hints +
   Pydantic are load-bearing in every AI codebase you'll read.
4. **Fluent Python by Luciano Ramalho (book)** — the closest thing to
   "Effective Java" for Python. Read chapters on data model, iterators, and
   decorators; skip the parts on older Python idioms superseded by newer
   syntax.

**Why these and not generic "Learn Python" courses:** you don't need
beginner pacing. You need idiom-transfer from a language you already know,
plus the specific patterns (async, type hints, dataclasses/Pydantic) that
show up constantly in AI codebases.

**Official documentation:** docs.python.org (primary reference, always
check version — target Python 3.12+).

**GitHub repositories to read (not just docs) for idiom exposure:**
- `pydantic/pydantic` — clean, modern Python, heavily used in every LLM SDK.
- `anthropics/anthropic-sdk-python` — read this to see how a production AI
  SDK is structured (you'll be using this SDK constantly later).
- `psf/requests` — the canonical example of clean, Pythonic API design.

**Blog posts:**
- Sebastian Raschka's Python/PyTorch posts (particularly relevant once you
  hit Part 2) — clear, first-principles explanations, no hand-waving.

---

### 17. Exercises (do these before the mini-project)

1. Rewrite this Java-style code Pythonically:
   ```python
   def get_user(user_id):
       if user_id is None:
           raise ValueError("user_id cannot be null")
       result = None
       for u in users:
           if u["id"] == user_id:
               result = u
       return result
   ```
   Target: use a generator expression + `next()` with a default, add type
   hints, and use idiomatic `None` handling.

2. Implement a `@dataclass(frozen=True)` representing an LLM API request
   (model, messages, temperature, max_tokens) and explain in a comment why
   immutability matters here (hint: think about retry logic and caching keys).

3. Write a context manager (`__enter__`/`__exit__`, or `@contextmanager`)
   that times a block of code and logs the duration — this exact pattern
   will reappear in Part 3 when we measure LLM call latency.

4. Explain, in your own words, why `except Exception:` (bare except) is
   dangerous in production Python — connect this to how you handle exceptions
   in Spring Boot `@ExceptionHandler`s.

### 18. Mini-Project
**Build:** A small CLI tool, `wordfreq`, that reads a text file and prints
the top-N most frequent words, with:
- Type hints throughout
- A `pyproject.toml` (using `uv init`)
- Argument parsing via `argparse` or `typer`
- One unit test using `pytest`
- A `README.md` explaining usage

This is intentionally simple — the goal is tooling fluency (project
structure, dependency management, testing), not algorithmic difficulty.

### 19. Production Project (portfolio-quality)
**Build:** `logsage` — a log file analyzer CLI, closer to something you'd
build at DoorDash-scale:
- Reads structured (JSON-lines) logs from a directory
- Filters by time range, log level, service name
- Outputs summary statistics (error rate over time, slowest endpoints)
- Fully typed, tested (>80% coverage), with `ruff` linting and `mypy` clean
- Packaged properly (`pyproject.toml`, installable via `pip install -e .`)
- Includes a GitHub Actions workflow that runs lint + tests on every push
  (we'll build this for real in Part 1; for now, a placeholder workflow file
  is fine — we'll wire it up properly later)
- README with architecture explanation, usage examples, and a "design
  decisions" section explaining *why* you structured it this way

This project deliberately mirrors backend tooling you already know how to
build in Java — the point is proving Python fluency on familiar ground before
we move into AI-specific territory in Part 2.

---

### 20–21. Practice & Interview Questions

1. Explain Python's GIL (Global Interpreter Lock) — what it actually
   restricts, and why it matters less for AI workloads that are I/O-bound
   (waiting on model APIs) than CPU-bound.
2. When would you reach for `multiprocessing` vs `asyncio` vs `threading` in
   a Python AI service? (Expected answer shape: asyncio for I/O-bound
   concurrent API calls to LLM providers; multiprocessing for CPU-bound work
   like local embedding generation; threading rarely wins in pure Python due
   to the GIL, except for I/O-bound work where asyncio is usually cleaner
   anyway.)
3. What's the difference between a Python generator and a list, and why does
   this matter when streaming tokens from an LLM response?
4. Why do production AI codebases use Pydantic so heavily? What problem does
   it solve that plain type hints don't?
5. Live-coding style: given a messy dict of API response data, write a
   Pydantic model that validates and parses it, handling an optional field
   and a nested object.

---

### 22. Common Mistakes
- Treating Python type hints as a runtime guarantee (they're not — you need
  Pydantic or explicit validation).
- Writing row-by-row loops over numeric/tabular data instead of vectorized
  operations (this becomes very costly once you hit Part 2/4 with embeddings
  and pandas).
- Using bare `except:` clauses, swallowing errors silently — this bites hard
  in AI pipelines where a silent failure (e.g., an embedding call failing)
  can corrupt a whole RAG index without any visible error.
- Mutable default arguments (`def f(x=[])`) — a classic Python gotcha with no
  Java equivalent, causes very confusing bugs.
- Ignoring dependency locking, leading to "works on my machine" — use `uv` or
  `poetry` with a committed lockfile from day one.

### 23. Debugging Exercise
You're given a script where a list passed as a default argument accumulates
values across unrelated function calls (the classic mutable-default-argument
bug). Find it, explain *why* it happens (tie back to the "everything is an
object, defaults are evaluated once" model), and fix it.

---

### 24. Checklist
- [ ] I can explain Python's object model (`x.__add__`) mental model
- [ ] I can write and use `@dataclass` and Pydantic models confidently
- [ ] I understand the GIL well enough to choose asyncio vs multiprocessing
- [ ] I have a working `uv`/`poetry`-managed project with a lockfile
- [ ] I've completed `wordfreq` and `logsage` end to end
- [ ] I'm comfortable reading unfamiliar Python in an OSS repo (tested by
      reading `anthropic-sdk-python` source for 20 minutes without getting lost)

### 25. Summary
Python's role in AI engineering is orchestration, not raw computation — the
heavy lifting happens in C/CUDA underneath. Your job is to be fluent enough
in Python's idioms (data model, type hints + Pydantic, async) that the
language never slows you down while you focus on system design, which is
where your existing backend strength transfers directly.

### 26. Next Steps
Module 2: **Git & GitHub for Serious Projects** — moving past "I know git
basics" into the workflows (trunk-based dev, conventional commits, PR
hygiene) that production AI teams actually use, since every project from here
on will be built and versioned properly.

---

**Reply "continue" when you're ready for Module 2, or tell me if you want to
go deeper/faster on anything in Module 1 first.**
