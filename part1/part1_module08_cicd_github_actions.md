# PART 1 — Software Engineering
## Module 8: CI/CD & GitHub Actions

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Design a complete CI pipeline (lint, type-check, test, build) using
  GitHub Actions, running automatically on every push/PR.
- Structure a CD pipeline that deploys automatically on merge to main,
  with proper environment separation (staging vs. production) — the
  direct prerequisite for Part 8's real deployment work.
- Handle secrets correctly in CI/CD (GitHub Secrets, never hardcoded or
  logged), including secrets needed for LLM provider API calls in
  automated tests.
- Design a CI strategy specifically for AI projects: separating fast,
  free unit tests (run every push) from slower, cost-incurring LLM
  evaluation suites (run on a schedule or a specific label, per Part 0
  Module 10's eval-harness preview).
- Set up branch protection rules requiring CI to pass before merge,
  connecting back to Part 0 Module 2's git workflow standard.

### 2. Prerequisites
Modules 1–7, Part 0 Modules 2 (Git), 5 (Docker), 10 (testing).

### 3. Estimated Study Time
8–10 hours over 4–5 days.

### 4. Difficulty
⭐⭐☆☆☆ (Easy-Medium — you likely have real CI/CD exposure from backend
work; the AI-specific cost/speed tiering for eval suites is the newer
consideration.)

### 5. Why This Matters
A portfolio project with a real, working CI/CD pipeline (visible via
GitHub's checks and Actions tab) is immediately more credible to both
interviewers and freelance clients than one without — it's tangible proof
of production discipline. It's also simply necessary: every project going
forward benefits from automated testing/deployment, and Part 8's
production deployment work assumes this pipeline already exists.

---

### 6. Theory

**What is it?**
CI (Continuous Integration) automatically runs checks (lint, type-check,
test, build) on every code change, catching problems before merge. CD
(Continuous Deployment/Delivery) automatically deploys code that passes CI
to an environment (staging, then production) — "Delivery" if a human
approves the final production push, "Deployment" if it's fully automatic.

**GitHub Actions structure, precisely:**
```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: astral-sh/setup-uv@v3        # Module 11's uv, in CI
      - run: uv sync --frozen               # exact lockfile install, Module 11's lesson
      - run: uv run ruff check .            # lint
      - run: uv run mypy .                  # type-check
      - run: uv run pytest --cov            # test + coverage (Module 10)
```
Each `job` runs in a fresh, isolated environment (a "runner"); `steps`
execute in sequence within a job. Jobs can run in parallel by default
unless you specify dependencies (`needs:`) — a good default for, e.g.,
running lint and tests as parallel jobs rather than one long sequential
job, cutting total CI wall-clock time.

**Handling secrets correctly:**
```yaml
      - run: uv run pytest
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```
Secrets live in GitHub's encrypted repository/organization secrets store
(Settings → Secrets and variables → Actions), never committed to the
repo, never printed in logs (GitHub Actions automatically masks secret
values that appear in log output, but design tests to avoid needing to
print them at all). For tests that would otherwise make real, cost-
incurring LLM API calls, use `dependency_overrides` (Module 3) or mocked
clients in your standard fast CI suite — reserve real API-key-consuming
test runs for a deliberately separate, slower pipeline (see below).

**Tiered CI strategy for AI projects — the genuinely AI-specific design
decision:**
```
On every push/PR (fast, free, seconds-to-minutes):
  - lint, type-check
  - unit tests (mocked LLM calls via dependency_overrides, Module 3)
  - integration tests against real Postgres/Redis (Docker Compose in CI)

On a schedule (nightly) OR a specific PR label (e.g., "run-eval"):
  - the real LLM evaluation suite (Part 0 Module 10's eval-harness,
    extended with real LLM calls in Part 3) — genuinely costs money and
    takes longer, so it shouldn't gate every single push
```
This tiering directly mirrors Module 4's alert/dashboard distinction
applied to CI: not everything deserves to block every single commit —
design deliberately for what must gate merges versus what runs on a
different cadence.

**CD environment separation:**
```
main branch merge
   |
   v
Deploy to STAGING automatically (a real environment, but not user-facing)
   |
   v (manual approval gate, or automatic after staging smoke tests pass)
Deploy to PRODUCTION
```
GitHub Actions' `environment:` key supports required reviewers for
production deployments — a genuine safety mechanism for anything beyond a
solo portfolio project, and good practice to build the habit of even for
solo work.

**Build/deploy Docker images in CI (tying back to Part 0 Module 5):**
```yaml
      - run: docker build -t myapp:${{ github.sha }} .
      - run: docker push myregistry/myapp:${{ github.sha }}
```
Tagging images by commit SHA (not just `latest`) gives you traceable,
reproducible deployments — you can always identify exactly which commit
produced a running production image, essential for debugging production
incidents (Module 4's observability philosophy extended to deployment
itself).

---

### 7. Mental Models

**Model 1 — "CI is a fast feedback loop that should run on every push;
anything cost-incurring or slow (real LLM eval calls) belongs on a
different, deliberate cadence, not gating every commit."**

**Model 2 — "Secrets in CI follow the exact same discipline as Part 0
Module 2's 'never commit a secret' lesson, extended to logs — never let a
secret value reach a log line, even in an automated pipeline."**

**Model 3 — "Tag deployed artifacts by commit SHA, not just `latest`, so
every running instance is traceable back to an exact, reviewable
commit."**

---

### 8. Visual Explanation (described)

**Diagram: "Tiered CI/CD pipeline for an AI project"**
```
Push/PR
   |
   +--> [Fast CI: lint, typecheck, unit tests (mocked LLM), integration
   |     tests] --- runs in ~2-5 min, gates merge ---
   |
   v (on merge to main)
[Build + tag Docker image by commit SHA] --> [Deploy to staging]
   |
   v
[Staging smoke tests] --> (manual approval, or auto) --> [Deploy to production]

Separately, on a schedule (nightly) or a PR label:
[Slow eval suite: real LLM calls, cost-incurring] --- does NOT gate every push ---
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **GitHub Actions official documentation** — "Understanding GitHub
   Actions" and "Workflow syntax" pages — the authoritative reference,
   thorough and current.
2. **GitHub Actions docs — "Using secrets in GitHub Actions"** and
   "Using environments for deployment" — directly relevant to the secrets
   and environment-separation content above.
3. **Martin Fowler's "Continuous Delivery" article** (martinfowler.com) —
   the canonical explanation of CI/CD philosophy and the
   delivery-vs-deployment distinction.
4. **Google's SRE Workbook, chapter on release engineering** (free
   online, sre.google) — practical guidance on staged rollouts and
   deployment safety, relevant ahead of Part 8.

**Official documentation:** docs.github.com/actions (primary reference for
this entire module).

**GitHub repos:** look at the `.github/workflows/` directory of any
well-maintained popular open-source Python project (`fastapi/fastapi`,
`pydantic/pydantic`) for real, production-grade CI configuration examples.

---

### 17. Exercises

1. Set up a GitHub Actions workflow for one of your Part 0/1 projects
   running lint, type-check, and tests on every push and PR, using `uv
   sync --frozen` for reproducible installs (Module 11).
2. Split lint/typecheck/test into separate parallel jobs and measure the
   wall-clock time difference versus running them sequentially in one job.
3. Add a GitHub Actions secret and use it in a workflow step, confirming
   (by checking the Actions log) that the value is masked if it happens to
   appear in output.
4. Design (write out, don't necessarily fully implement) a two-tier CI
   config: one workflow triggered on every push (fast, mocked), one
   triggered on a schedule or PR label (slow, real LLM calls) — explain
   your tiering decisions.

### 18. Mini-Project
Add a complete CI workflow to `auth-gateway` (Module 7): lint, type-check,
unit tests (with LLM calls mocked via dependency overrides), and
integration tests against real Postgres/Redis spun up as GitHub Actions
service containers (or via your Docker Compose stack) — all gating PR
merges via branch protection (Part 0 Module 2).

### 19. Production Project
**Build:** A complete CI/CD pipeline for your most complete project so far
(`auth-gateway` extended, or a project of your choice from Parts 0-1):
- Fast CI workflow: lint, type-check, unit + integration tests, running on
  every push/PR, required to pass before merge (branch protection)
- Docker image build and push, tagged by commit SHA, triggered on merge to
  main
- A staging deployment step (can target a free-tier service like Fly.io/
  Railway for now — full depth comes in Part 8) triggered automatically
  after a successful build
- A separate, tiered workflow (nightly schedule or label-triggered)
  running a slower evaluation suite that would involve real LLM calls
  (stub this with your `eval-harness-preview` from Part 0 Module 10 for
  now, since real LLM integration comes in Part 3)
- A README badge showing CI status, and a documented section explaining
  the tiering rationale (what runs on every push vs. on a schedule, and
  why)

This pipeline becomes the template you copy/adapt for every subsequent
capstone project in Part 11 — invest real care here.

---

### 20–21. Practice & Interview Questions

1. Explain the difference between Continuous Delivery and Continuous
   Deployment.
2. Why should an AI project's CI pipeline separate fast, mocked unit tests
   from slower, cost-incurring LLM evaluation suites, and how would you
   structure that separation?
3. How does GitHub Actions handle secrets, and what precautions should you
   still take even with built-in masking?
4. Why is tagging deployed Docker images by commit SHA (rather than just
   `latest`) important for production debugging and rollback?
5. Design a CI/CD pipeline for a FastAPI + Postgres + Redis project from
   scratch, explaining each stage's purpose and what would block a merge
   versus what wouldn't.

---

### 22. Common Mistakes
- Running slow, cost-incurring LLM evaluation calls on every single push,
  wasting money and slowing down the fast feedback loop CI is meant to
  provide.
- Hardcoding secrets in workflow files instead of using GitHub's secrets
  store.
- Deploying directly to production with no staging step or smoke tests.
- Tagging Docker images only as `latest`, losing the ability to trace a
  running instance back to a specific commit.
- Not requiring CI to pass before merge (no branch protection), making the
  pipeline advisory rather than enforced.

### 23. Debugging Exercise
Given a CI pipeline where tests pass locally but fail in GitHub Actions
specifically, diagnose (using Module 3's testing skills combined with
Module 11's lockfile discipline) that the CI environment installed
different dependency versions than local (a missing `--frozen`/`ci` flag
allowing drift) — fix by enforcing exact lockfile installs in CI.

---

### 24. Checklist
- [ ] I've built a real, working CI workflow gating PR merges via branch
      protection
- [ ] I keep all secrets in GitHub's secrets store, never hardcoded
- [ ] I've designed (and ideally implemented) a tiered CI strategy
      separating fast tests from slow/costly eval runs
- [ ] I tag Docker images by commit SHA in my build pipeline
- [ ] I've completed a full CI/CD pipeline for at least one project, with
      a documented tiering rationale

### 25. Summary
A well-designed CI/CD pipeline is both a genuine engineering necessity and
a visible credibility signal in your portfolio. The AI-specific design
decision that doesn't exist in typical backend CI is tiering: fast, free,
mocked tests gate every push, while slower, cost-incurring LLM evaluation
suites run on a separate, deliberate cadence. This pipeline template
carries forward into every capstone project in Part 11.

### 26. Next Steps
Module 9: **Security Fundamentals** — a focused module on the security
concerns specific to AI systems (prompt injection previewed, secrets
management, dependency vulnerabilities) beyond what's already been covered
incidentally in auth (Module 7) and cloud (Part 0 Module 14).

---

**Reply "continue" for Module 9, or flag anything to go deeper on.**
