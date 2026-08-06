# PART 0 — Prerequisites
## Module 2: Git & GitHub for Serious Projects

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Use Git's actual data model (not just memorized commands) to reason about
  any situation — merge conflicts, rebases, detached HEAD, lost commits —
  instead of panicking and Googling.
- Run a trunk-based, PR-driven workflow the way production AI/backend teams
  do, including conventional commits and clean history.
- Set up branch protection, required reviews, and status checks on GitHub so
  your portfolio repos look and behave like real engineering-org repos.
- Recover from the git disasters that actually happen in practice (bad
  rebase, force-push mistakes, accidental commits of secrets).
- Structure a GitHub repo (README, CONTRIBUTING, issue templates, PR
  templates) the way employers and open-source maintainers expect.

### 2. Prerequisites
Module 1 (Python) — we'll use the `logsage` project as the repo we practice
on.

### 3. Estimated Study Time
6–8 hours over 3–4 days. You already know git basics from Java work, so this
module is about going from "usable" to "fluent and trustworthy under
pressure."

### 4. Difficulty
⭐⭐☆☆☆ (Easy-Medium — the mental model is the hard part, not the commands.)

### 5. Why This Matters
Every AI-lab and startup engineering team lives inside Git and GitHub all
day: PRs are how code gets reviewed, CI is triggered by pushes, and clean
history is how bugs get bisected months later. For your specific goals —
DoorDash interviews often touch on collaborative workflow expectations
informally, and your freelancing/open-source track (Part 9) is *entirely*
gated on being a comfortable, non-panicky Git user in front of strangers.

---

### 6. Theory

**What is it?**
Git is a content-addressable, directed acyclic graph (DAG) database of
snapshots. That's it. Once that sentence actually clicks, every git command
becomes "a way to move a pointer around the graph" instead of a magic
incantation.

**Why do we need it?**
You need a system that lets many people work on the same codebase
concurrently, track *why* every line changed, and safely experiment (branch)
without risking the stable version (main/trunk). GitHub adds the
collaboration layer on top: code review, CI integration, issue tracking.

**How does it work? (the mental model that fixes 90% of git confusion)**

```
A commit is a snapshot + a pointer to its parent commit(s).
A branch is just a movable label pointing at a commit.
HEAD is a pointer to "whichever branch (or commit) you're currently on."
```

```
main:     A---B---C
                    \
feature:             D---E   <- HEAD -> feature
```

- `git checkout feature` moves HEAD to point at the `feature` label.
- `git commit` creates a new snapshot, then moves the *current branch label*
  to point at it.
- `git merge` creates a new commit with **two parents** (joining two lines of
  history).
- `git rebase` **replays** commits from one branch onto another, creating
  *new* commit objects with new hashes (this is why rebase rewrites history —
  it's not moving the old commits, it's creating fresh ones and abandoning
  the old ones, which is why force-pushing after a rebase is necessary and
  why rebasing shared/pushed branches is dangerous).
- A "detached HEAD" simply means HEAD points directly at a commit instead of
  at a branch label — nothing is broken, you just aren't "on" a named branch,
  so new commits won't be reachable by any branch unless you create one.

**Java/Maven analogy for you:** think of every commit as an immutable JAR
snapshot with a manifest pointing to its "previous version's" JAR. Branches
are like Maven profile pointers that say "the current build for this stream
of work is JAR #47." Merging = combining two JARs' change sets into one new
JAR that depends on both parent JARs.

**How is it implemented (just enough internals)?**
Every commit, tree, and blob is stored as a SHA-1(now moving to SHA-256)
hashed object in `.git/objects`. This is why git is fast at detecting
identical content (same content → same hash → stored once) and why commit
hashes change the instant *any* ancestor changes (rebase, amend) — the hash
is a function of the content plus parent pointers.

**When should each workflow be used?**
- **Trunk-based development (short-lived feature branches, frequent merges
  to `main`):** the modern default at fast-moving AI/startup teams — small
  PRs, fast review cycles, minimal merge conflicts.
- **GitFlow (long-lived develop/release branches):** still seen in some
  enterprise/regulated environments, but heavier than most AI teams need.
  Good to *recognize*, not necessarily adopt.
- **Rebase workflow:** use for your own local, not-yet-pushed feature
  branches to keep history linear and readable before opening a PR. **Never**
  rebase a branch other people are also working on — you'll rewrite history
  out from under them.

**When should it NOT be used this way?**
Don't force-push to `main` or any shared branch. Don't use `git rebase -i`
to rewrite history that's already been reviewed/merged — that destroys the
review record.

---

### 7. Mental Models

**Model 1 — "Git never deletes anything for ~90 days by default."**
Even a "lost" commit after a bad reset is almost always recoverable via
`git reflog`, because git keeps a log of everywhere HEAD has pointed. This
single fact should remove 95% of git panic.

**Model 2 — "A pull request is a diff between two pointers, plus a
conversation."** It's not a special git object — it's a GitHub-level
concept layered on top of comparing two branches.

**Model 3 — "Merge conflicts are git asking you to resolve an ambiguity it
can't resolve algorithmically."** Two lines changed the same region of a
file differently; git can't guess intent, so it hands you both versions.

---

### 8. Visual Explanation (described)

**Diagram: "Trunk-based PR workflow"**
```
main:      A---B-------------F---G   (F, G = merged PRs)
                \           /   \
feature/x:       C1--C2--C3      (squash-merged into F)
                                  \
                              feature/y: D1--D2  (in review, becomes G)
```
Feature branches are short-lived (hours to a couple of days), merged via PR
(often squash-merged so `main` shows one clean commit per feature), then
deleted. This keeps `main`'s history readable as "one line per meaningful
change" rather than a tangle of WIP commits.

---

### 9–16. Recommended Resources

**Reading order:**
1. **Pro Git book, Chapters 1–3** (free, git-scm.com/book) — the single best
   explanation of the data model. Chapter 3 (branching) is the most
   important chapter in this entire module.
2. **"Git from the Bottom Up" by John Wiegley** — short, internals-focused,
   confirms the mental model above with actual implementation detail.
3. **GitHub's own docs on branch protection rules and required reviews** —
   because you'll configure these on every portfolio repo going forward.
4. **Conventional Commits spec** (conventionalcommits.org) — the commit
   message convention (`feat:`, `fix:`, `chore:`) used widely enough that
   it's worth adopting as a default habit; it also enables automated
   changelog generation, which we'll wire into CI in Part 1.

**Why these:** Pro Git is free, official, and explains internals rather than
just commands — which is exactly the gap for someone who already "knows git"
at a surface level. The other three are the specific conventions you'll be
expected to already know at any team using modern GitHub workflows.

**GitHub repos to study (for workflow, not code):**
- Any well-run mid-size OSS repo's `CONTRIBUTING.md` + recent PR history —
  e.g., `pydantic/pydantic` or `astral-sh/uv` — look at how they structure
  PR descriptions, review comments, and commit squashing.

**Videos:** "Git Internals" — search current top-rated conference talks on
git internals (e.g., from Git Merge conference archives) rather than
generic "Git tutorial for beginners" content, which stays at the command
level you've already surpassed.

---

### 17. Exercises

1. Create a repo, make 4 commits on `main`, then create a branch, make 2
   commits, and use `git log --graph --oneline --all` to actually *see* the
   DAG structure described above. Confirm your mental model matches reality.
2. Intentionally create a merge conflict (edit the same line on two
   branches), resolve it, and explain in your own words what git could and
   couldn't figure out on its own.
3. Make a commit, then `git reset --hard HEAD~1` to "lose" it, then recover
   it using `git reflog` + `git cherry-pick` or `git reset`. This exercise
   alone will permanently remove git-panic from your toolkit.
4. Write 5 commit messages in Conventional Commits format for a hypothetical
   set of changes (a bug fix, a new feature, a refactor, a docs update, a
   dependency bump).

### 18. Mini-Project
Take the `logsage` repo from Module 1 and:
- Set up branch protection on `main` (require PR, require passing checks,
  no direct pushes, no force-push).
- Add a `CONTRIBUTING.md`, PR template, and issue templates (bug report,
  feature request).
- Practice the full loop: branch → commit (conventional commits) → PR →
  (self-)review → squash-merge → delete branch.

### 19. Production Project
**Build:** A documented "Git workflow standard" for your own portfolio,
`git-workflow.md`, that you'll reuse across every project in this handbook —
covering branch naming convention, commit message format, PR template, and a
short "how to recover from common mistakes" runbook (bad rebase, accidental
secret commit, wrong branch). Treat this like an internal engineering
standards doc — this is genuinely the kind of artifact that impresses in a
portfolio review, because it signals you think about process, not just code.

---

### 20–21. Practice & Interview Questions

1. Explain what `git rebase` actually does at the object level, and why it
   changes commit hashes.
2. What's the difference between `git merge` and `git rebase`, and when
   would you choose one over the other on a team?
3. A teammate accidentally committed an API key. Walk through exactly what
   you'd do (this tests whether you know that simply deleting the file in a
   new commit does *not* remove it from history — you need `git filter-repo`
   or BFG, plus key rotation, since the key is compromised the moment it's
   pushed).
4. What is a detached HEAD state and how do you get out of it safely?
5. Explain squash-merge vs. regular merge vs. rebase-and-merge as GitHub PR
   merge strategies — trade-offs of each for history readability vs. traceability.

---

### 22. Common Mistakes
- Force-pushing to a shared branch (rewrites history teammates rely on).
- Committing secrets and thinking "I'll just delete it in the next commit"
  (it's still in history — this is a real, recurring production incident
  type).
- Giant, unreviewable PRs (a code-review anti-pattern, not just a git one —
  but git enables it if you don't discipline yourself to branch/commit small).
- Rebasing public/shared branches.
- Vague commit messages ("fix stuff", "wip") — useless for `git bisect` or
  future archaeology.

### 23. Debugging Exercise
You're mid-rebase and it's conflicted three commits deep, and you're not
sure what state you're in. Practice: `git status` (tells you you're
rebasing and which files conflict), `git rebase --abort` (bail out safely to
where you started), then redo it more carefully one commit at a time with
`git rebase -i` and resolving one conflict at a time. Goal: build the
reflex of "abort and retry calmly" instead of panicking mid-rebase.

---

### 24. Checklist
- [ ] I can explain git's DAG/object model without looking anything up
- [ ] I've recovered a "lost" commit via reflog at least once
- [ ] I have a repo with branch protection + PR template + issue templates
- [ ] I default to Conventional Commits without thinking about it
- [ ] I know exactly what to do if I accidentally commit a secret
- [ ] I understand rebase vs. merge well enough to choose deliberately, not
      by habit

### 25. Summary
Git is a graph of immutable snapshots, and branches are just movable labels
on that graph — once that clicks, every command (and every recovery
situation) becomes reasoning instead of memorization. GitHub layers
collaboration (PRs, reviews, CI triggers, branch protection) on top of that
graph. Your goal this module was fluency and calm under pressure, not new
vocabulary — you'll rely on this constantly across every remaining module
and project.

### 26. Next Steps
Module 3: **Linux, Bash & the Terminal as a Workshop** — moving from "I can
use a terminal" to treating it as your primary environment for running
containers, inspecting logs, and debugging production AI services, which
becomes essential starting in Part 6 (Infrastructure).

---
