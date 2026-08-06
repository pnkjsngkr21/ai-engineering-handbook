# PART 0 — Prerequisites
## Module 11: CLI Tools, Package Management & Virtual Environments

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Explain precisely why Python needs virtual environments (unlike the
  JVM's classpath-per-run-config model) and choose the right tool for
  managing them.
- Use `uv` (the modern, fast standard) confidently for dependency
  management, replacing the older pip+venv+requirements.txt combination.
- Understand lockfiles deeply enough to explain exactly what
  reproducibility guarantee they provide and what they don't.
- Manage Node package management (`npm`/`pnpm`) with the same rigor,
  understanding `package-lock.json`/`pnpm-lock.yaml` the same way.
- Build a personal CLI toolkit (aliases, useful global tools) that makes
  you meaningfully faster day to day.

### 2. Prerequisites
Modules 1, 3, 7 (Python, Bash, JS/TS) — this module formalizes tooling used
implicitly throughout.

### 3. Estimated Study Time
4–6 hours over 2–3 days. Shortest module in Part 0 — mostly consolidation.

### 4. Difficulty
⭐☆☆☆☆ (Easy — mechanical, not conceptually deep, but worth doing properly
once rather than accumulating bad habits.)

### 5. Why This Matters
"Works on my machine" bugs, slow onboarding for new contributors to your
open-source work (Part 9's freelancing goal depends on people being able to
clone and run your projects easily), and CI environment mismatches all trace
back to sloppy dependency management. This is a small module with
outsized payoff for how professional your portfolio repos feel.

---

### 6. Theory

**What is it?**
A virtual environment is an isolated Python installation (its own
`site-packages` directory) so that project A's dependencies don't conflict
with project B's, and neither pollutes your system Python. A lockfile
pins the *exact* resolved version of every dependency (direct and
transitive) so that "install my dependencies" produces byte-for-byte the
same environment on any machine.

**Why do we need this — and how is it different from Java?**
The JVM doesn't have this exact problem in the same way: Maven/Gradle
resolve dependencies **per-project** into a local repository cache
(`~/.m2`), and your classpath is explicitly scoped per build — there's no
global "system Java" whose packages get polluted the way a global `pip
install` pollutes system Python. Python's virtual environments exist
specifically to recreate that per-project isolation, because Python's
"just `pip install` it" felt convenient early on but doesn't scale to
multiple projects with conflicting dependency versions without isolation.

**Why `uv` specifically (the modern standard as of 2025-2026)?**
Historically Python tooling was fragmented (`pip` + `venv` + manually
maintained `requirements.txt`, or `poetry`, or `pipenv` — each with
different trade-offs and none universally adopted). `uv` (from Astral, the
makers of `ruff`) has become the modern default because it's dramatically
faster (written in Rust), handles virtual environments, dependency
resolution, and lockfiles in one coherent tool, and is rapidly becoming the
ecosystem standard. Use it as your default going forward in this handbook.

```bash
uv init myproject          # creates pyproject.toml
uv add fastapi             # adds dependency, updates pyproject.toml + uv.lock
uv sync                    # installs exactly what's in uv.lock
uv run python script.py    # runs inside the project's venv, no manual activation needed
```

**What a lockfile actually guarantees (and doesn't):**
It guarantees that everyone running `uv sync` (or `npm ci`, the lockfile-
respecting equivalent of `npm install`) gets the **exact same dependency
versions**, including transitive dependencies, that were resolved when the
lockfile was generated. It does **not** guarantee the package contents
haven't changed on the registry (rare, but possible for some registries),
and it does **not** protect against OS-level or system-library differences
outside the package manager's control (which is part of why Docker, from
Module 5, matters for true reproducibility).

**Node.js equivalent, briefly:**
`npm install` updates `package-lock.json` based on `package.json`'s
version ranges (which can drift over time as new patch/minor versions are
published). `npm ci` (use this in CI and Docker builds) installs *exactly*
what's in the lockfile, failing if `package.json` and the lockfile are out
of sync — this is your reproducibility guarantee, directly analogous to
`uv sync --frozen`.

---

### 7. Mental Models

**Model 1 — "A virtual environment is a project-scoped classpath that
Python needs explicitly, because Python doesn't have Maven's implicit
per-project isolation built into its basic tooling."**

**Model 2 — "A lockfile is a Maven `pom.xml` plus every transitive
dependency's exact resolved version, frozen in time."** `mvn` and `gradle`
give you strong reproducibility by default in a way Python's ecosystem
historically didn't — lockfiles are how the Python/JS world catches up to
that guarantee.

**Model 3 — "`uv run` is like `mvn exec:java` — it handles environment
activation for you so you never manually `source venv/bin/activate`."**

---

### 8. Visual Explanation (described)

**Diagram: "Dependency resolution vs. lockfile installation"**
```
First time (resolution):
pyproject.toml (version RANGES, e.g. "fastapi>=0.100")
        |
        v  (resolver picks specific compatible versions for everything,
        |   including transitive deps)
   uv.lock (EXACT versions pinned, committed to git)

Every subsequent install (reproducible):
   uv.lock  --uv sync-->  identical environment, every machine, every time
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **`uv` official documentation** (docs.astral.sh/uv) — read the "Getting
   Started" and "Projects" pages; it's concise and well-organized.
2. **Python Packaging User Guide — "Installing packages"** (packaging.
   python.org) — for the conceptual background on virtual environments,
   even if you use `uv` for the mechanics.
3. **npm docs — "package-lock.json"** page — read to understand exactly
   what npm's lockfile guarantees, and the `npm ci` vs. `npm install`
   distinction.

**Official documentation:** docs.astral.sh/uv, docs.npmjs.com.

---

### 17. Exercises

1. Initialize a project with `uv init`, add three dependencies, inspect the
   generated `uv.lock`, and explain what information it contains beyond
   just version numbers (hashes, resolved transitive deps).
2. Delete your virtual environment entirely, run `uv sync`, and confirm
   the environment is rebuilt identically from the lockfile alone.
3. In a Node project, compare the behavior of `npm install` vs. `npm ci`
   after manually editing `package.json` to a slightly different version
   range without updating the lockfile — observe `npm ci` failing loudly
   instead of silently resolving something different.

### 18. Mini-Project
Go back through every Python project built in Modules 1–10 and confirm
each has a `pyproject.toml` + committed `uv.lock`, with `uv sync --frozen`
used in any Dockerfile/CI config instead of a loose `pip install -r
requirements.txt` — retrofit any that were built with older tooling.

### 19. Production Project
**Build:** A personal `dotfiles`/tooling repo (genuinely useful, not
throwaway) containing: your preferred shell aliases (from Module 3 skills),
a documented standard project template (`pyproject.toml` skeleton with
`ruff`, `mypy`, `pytest` pre-configured) you'll reuse for every Python
project going forward, and a short README explaining your standard tooling
choices and why — this becomes a real artifact worth linking from your
portfolio/resume as evidence of engineering maturity around tooling.

---

### 20–21. Practice & Interview Questions

1. Why does Python need virtual environments when Java's Maven/Gradle
   don't have quite the same problem?
2. What exactly does a lockfile guarantee, and what does it not protect
   against?
3. What's the difference between `npm install` and `npm ci`, and why
   should CI/Docker builds always use the latter?
4. Why has the Python ecosystem converged on tools like `uv` recently, and
   what problem were `pip` + manually maintained `requirements.txt` files
   failing to solve well?

---

### 22. Common Mistakes
- Committing a `requirements.txt` with loose version ranges instead of a
  proper lockfile, causing "works on my machine" drift over time.
- Using `pip install` directly into a system/global Python instead of a
  project-scoped virtual environment.
- Using `npm install` in CI/Docker builds instead of `npm ci`, allowing
  silent dependency drift in automated environments.
- Not committing the lockfile to version control at all.

### 23. Debugging Exercise
Given two developers getting different behavior from "the same" codebase,
diagnose that one has a stale/uncommitted lockfile or is using loose
version-range installation instead of the committed lockfile — fix by
standardizing on `uv sync --frozen` / `npm ci` everywhere (local dev
scripts, CI, Dockerfiles).

---

### 24. Checklist
- [ ] Every project from Modules 1–10 has a proper lockfile committed
- [ ] I use `uv sync --frozen` / `npm ci` in all CI/Docker contexts
- [ ] I can explain what a lockfile does and doesn't guarantee
- [ ] I've built a reusable personal tooling/dotfiles repo

### 25. Summary
Virtual environments and lockfiles exist to give Python and Node the same
reproducibility guarantee Maven/Gradle give Java more implicitly —
consistent, exact dependency versions across every machine and
environment. `uv` is the modern, fast, unified standard for the Python
side of this. This was a short, consolidating module — the payoff is in
every project going forward behaving predictably.

### 26. Next Steps
Module 12: **Async Programming (Python & JS)** — a dedicated deep-dive
comparing Python's `asyncio` and JavaScript's event loop side by side,
since async correctness is one of the highest-leverage skills for AI
systems (which are dominated by I/O-bound waiting on model APIs).

---
