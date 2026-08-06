# PART 0 — Prerequisites
## Module 10: Testing & Debugging Discipline

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Design a deliberate testing strategy (unit / integration / contract /
  end-to-end) for a service, choosing the right mix instead of defaulting
  to one type everywhere.
- Write effective pytest tests (fixtures, parametrization, mocking) and
  React Testing Library tests, at a level equivalent to your JUnit/Mockito
  fluency.
- Understand why testing **non-deterministic AI outputs** (LLM responses)
  requires a different strategy than testing deterministic code — and know
  the actual techniques (golden datasets, LLM-as-judge, snapshot testing
  with tolerance) used in production AI teams, previewed here ahead of Part
  3's deeper evaluation content.
- Debug systematically using a repeatable method (reproduce → isolate →
  hypothesize → verify) instead of ad-hoc print-statement flailing.
- Set up test coverage reporting and understand what coverage percentage
  actually tells you (and doesn't).

### 2. Prerequisites
Modules 1–9. This module formalizes testing habits already required in
every prior project.

### 3. Estimated Study Time
8–10 hours over 4–5 days.

### 4. Difficulty
⭐⭐☆☆☆ (Easy-Medium — you already understand testing deeply from
JUnit/Mockito; the new territory is testing non-deterministic AI outputs.)

### 5. Why This Matters
Testing discipline is what separates a portfolio project from a demo. It's
also directly interview-relevant — "how would you test this" is a common
follow-up in senior backend interviews, and for AI systems specifically,
"how do you test something that doesn't return the same output twice" is a
question that trips up engineers without a deliberate framework for it.
Part 3 builds a full LLM evaluation pipeline — this module lays the
conceptual groundwork.

---

### 6. Theory

**What is it?**
The testing pyramid (unit → integration → end-to-end, in increasing
cost/decreasing quantity) is the same shape in Python/JS as in Java — the
principles transfer directly. What's genuinely new for AI systems is a
**fourth category**: evaluation of non-deterministic, quality-graded
outputs, which doesn't fit neatly into pass/fail unit tests.

**Why do we need different testing strategies for different layers?**
Unit tests are fast and precise but test units in isolation (mocking
dependencies) — great for business logic, useless for "does this SQL query
actually work against real Postgres." Integration tests exercise real
dependencies (a real test database, a real Redis instance, often via
Testcontainers-equivalent tooling) — slower, but catch real integration
bugs unit tests can't. End-to-end tests exercise the whole system as a
user would — slowest, most brittle, reserved for critical user journeys.

**Java → Python/JS testing mapping:**

| Java/JUnit                          | Python (pytest)                    | JS/TS (Vitest/Jest + RTL)          |
|---------------------------------------|--------------------------------------|---------------------------------------|
| `@Test`                              | a function named `test_*`            | `test(...)` / `it(...)`               |
| `@BeforeEach`/`@AfterEach`           | pytest `fixture`s                    | `beforeEach`/`afterEach`               |
| Mockito `@Mock`/`when(...).thenReturn`| `unittest.mock` / `pytest-mock`      | `vi.fn()` / `jest.fn()`                |
| `@ParameterizedTest`                 | `@pytest.mark.parametrize`           | `test.each([...])`                     |
| Testcontainers                        | `testcontainers-python` (same idea)  | (less common; often just Docker Compose in CI) |
| JaCoCo coverage                       | `pytest-cov` / `coverage.py`         | `vitest --coverage` / Istanbul         |

**Why does testing LLM output need a different approach?**
A traditional unit test asserts exact equality (`assertEquals(expected,
actual)`). An LLM asked "summarize this paragraph" will produce a
*different but equally valid* summary every time — exact-match assertions
are the wrong tool entirely. Production AI teams instead use:
- **Golden datasets** — a curated set of (input, expected-properties) pairs,
  where "expected" is a set of *properties* the output must have (contains
  key facts, correct length range, correct format), not an exact string.
- **LLM-as-judge** — using a second LLM call to *grade* the first model's
  output against a rubric (e.g., "does this summary contain the three key
  facts from the source? yes/no"), turning a fuzzy quality judgment into a
  more test-like pass/fail signal. (You'll build this for real in Part 3.)
- **Structural/schema assertions** — if the LLM is producing structured
  output (JSON via function calling), you *can* assert exact schema
  validity even when content varies — validate the shape, not the exact
  content.
- **Regression/snapshot testing with human review** — track output changes
  over time (e.g., after a prompt change or model upgrade) and flag them
  for human review rather than auto-failing, since "different" isn't
  necessarily "wrong."

This module previews these ideas conceptually; Part 3 builds a full
evaluation harness using them for real.

**Debugging discipline — a repeatable method (language-agnostic, but worth
formalizing now):**
1. **Reproduce reliably** — if you can't reproduce it consistently, you
   don't understand it yet; find the minimal reproduction.
2. **Isolate** — bisect the problem space (which layer? which input?
   which recent change?) using `git bisect`, logging, or a debugger, not
   guessing.
3. **Hypothesize** — form a specific, falsifiable hypothesis about the
   cause before changing code ("I think X is null because Y").
4. **Verify** — test the hypothesis directly (a targeted log line, a
   debugger breakpoint, a minimal test) before writing a fix; don't fix
   symptoms you haven't confirmed the cause of.

---

### 7. Mental Models

**Model 1 — "The testing pyramid is about return-on-investment, not
dogma."** More unit tests (cheap, fast, precise) than integration tests
(slower, broader) than E2E tests (slowest, most valuable per-test but most
expensive to maintain) — the shape, not exact ratios, is what matters.

**Model 2 — "Non-deterministic output needs property-based or
graded assertions, not exact-match assertions."** The moment you're
tempted to write `assert response == "exact expected string"` against an
LLM call, stop — you're using the wrong assertion type for a
non-deterministic system.

**Model 3 — "Debugging without a hypothesis is just editing code and
hoping."** If you can't state, in one sentence, what you think is wrong
before you start changing code, you're not debugging yet — you're
guessing.

---

### 8. Visual Explanation (described)

**Diagram: "The testing pyramid, adapted for AI systems"**
```
                /  E2E tests (few, slow, real user journeys)          \
               /--------------------------------------------------------\
              /  Integration tests (real DB/Redis, moderate count)       \
             /------------------------------------------------------------\
            /  Unit tests (many, fast, isolated, mocked dependencies)      \
           /--------------------------------------------------------------- \
          [  NEW: Eval/golden-dataset layer — non-deterministic AI outputs,  ]
          [  graded via LLM-as-judge or property assertions, run separately  ]
          [  from the pass/fail pyramid above (often on a schedule, not      ]
          [  every commit, since it's slower and costs money per LLM call)   ]
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **pytest official docs — "Fixtures" and "Parametrizing tests"** —
   read directly; pytest's docs are clear and example-driven.
2. **React Testing Library docs — "Guiding Principles"** — specifically
   the philosophy of testing behavior, not implementation details (a
   principle that transfers directly from good JUnit/Mockito practice).
3. **"Testing of Machine Learning Systems" — Google's ML Test Score paper
   (research.google)** — an excellent, practical framework for what "well
   tested" means for ML/AI systems specifically, worth reading even at this
   early stage as a preview of Part 3.
4. **Martin Fowler's "Test Pyramid" article** (martinfowler.com) — the
   canonical explanation of pyramid shape and trade-offs, framework-
   agnostic.

**Official documentation:** docs.pytest.org, testing-library.com/docs.

**GitHub repos:** look at test suites in `pydantic/pydantic` or
`tiangolo/fastapi` itself for examples of well-parametrized, well-organized
test files.

---

### 17. Exercises

1. Write a parametrized pytest test covering 5+ input/expected-output pairs
   for a pure function, comparing the syntax directly to
   `@ParameterizedTest` in JUnit.
2. Mock an external HTTP call in a pytest test using `pytest-mock` or
   `unittest.mock.patch`, and explain the equivalent Mockito pattern you'd
   use for the same scenario.
3. Write a React Testing Library test for the `chat-shell` project (Module
   8) that types into the input, clicks send, and asserts the message
   appears — testing behavior, not internal component state.
4. Design (on paper) a golden dataset of 5 (prompt, expected-properties)
   pairs for a hypothetical "extract the sentiment of this review" LLM
   feature — write what properties you'd check (not exact strings).

### 18. Mini-Project
**Build:** Bring `convo-api` (Module 9) to a real, deliberate test
structure: unit tests for the service layer (mocking the repository),
integration tests against a real test Postgres/Redis instance (via
Testcontainers or your Module 5 Compose stack spun up for CI), and a
coverage report via `pytest-cov`, with a documented rationale for why each
test is at the layer it's at.

### 19. Production Project
**Build:** `eval-harness-preview` — a small standalone tool that
implements the "LLM-as-judge" pattern conceptually, using a **rule-based
mock judge** for now (since we haven't built real LLM integration yet —
Part 3 will replace the mock with a real judge call):
- A golden dataset file (JSON) of (input, expected-properties) pairs for a
  toy task (e.g., "does this generated text mention all N required
  keywords, and stay under M words")
- A runner that evaluates a (mocked, for now) "generation function" against
  every golden example, scoring pass/fail per property, and produces a
  summary report (pass rate, failing examples with reasons)
- Structured so that swapping the mock generation function for a real LLM
  call, and the mock judge for a real LLM-as-judge call, in Part 3 requires
  changing only one function each — architected for that extension
  deliberately

This previews the evaluation harness you'll build for real once you have
actual LLM integration in Part 3 — you're building the scaffolding now, so
Part 3 is "plug in the real thing" rather than starting from zero.

---

### 20–21. Practice & Interview Questions

1. Explain the testing pyramid and how you'd decide, for a given piece of
   logic, whether it needs a unit test, an integration test, or both.
2. Why can't you test an LLM's output with exact-match assertions, and
   what are two alternative strategies that work for non-deterministic
   output?
3. Walk through your systematic debugging method (reproduce → isolate →
   hypothesize → verify) using a real bug you've encountered.
4. What does test coverage percentage actually measure, and what does a
   high coverage percentage *not* guarantee? (Expected: it measures lines
   executed, not correctness of assertions — you can have 100% coverage
   with tests that assert nothing meaningful.)
5. How would you structure a CI pipeline differently for fast unit tests
   (run on every push) vs. a slower, cost-incurring LLM evaluation suite
   (perhaps run nightly or on a label, not every commit)?

---

### 22. Common Mistakes
- Writing exact-match assertions against non-deterministic LLM output.
- Treating coverage percentage as a proxy for test quality rather than a
  (weak) proxy for what code paths were merely executed.
- Debugging by randomly changing code until symptoms disappear, without
  ever forming or verifying a hypothesis about root cause.
- Over-relying on E2E tests for coverage that unit tests could provide
  much faster and more reliably.
- Mocking too much in integration tests, defeating their purpose (an
  "integration test" that mocks the database isn't actually testing
  integration).

### 23. Debugging Exercise
Given a flaky test that passes locally but fails intermittently in CI,
apply the reproduce → isolate → hypothesize → verify method to discover
it's a test-ordering/shared-state issue (e.g., tests relying on database
state left over from a previous test, or a timing assumption that doesn't
hold under CI's slower/shared hardware) — fix by properly isolating test
state (transactional rollback per test, or fresh fixtures).

---

### 24. Checklist
- [ ] I can explain the testing pyramid and apply it deliberately, not by
      habit
- [ ] I understand why exact-match assertions fail for LLM output and know
      at least two alternative strategies
- [ ] I follow a repeatable debugging method rather than ad-hoc guessing
- [ ] I've built `eval-harness-preview` with a clean seam for swapping in
      real LLM calls later
- [ ] I understand what coverage percentage does and doesn't tell me

### 25. Summary
Testing principles transfer directly from your JUnit/Mockito experience —
the pyramid shape, mocking, parametrization are the same ideas in new
syntax. The genuinely new territory is testing non-deterministic AI
output, which needs golden datasets, LLM-as-judge grading, or structural
assertions instead of exact-match comparisons. `eval-harness-preview`
gives you the scaffolding for Part 3's real evaluation pipeline.

### 26. Next Steps
Module 11: **CLI Tools, Package Management & Virtual Environments** — a
shorter, practical module tightening up the tooling hygiene (dependency
locking, environment isolation) that's been used implicitly throughout
Part 0 so far, making it deliberate and well-understood.

---
