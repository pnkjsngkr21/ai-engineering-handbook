# PART 0 — Prerequisites
## Module 3: Linux, Bash & the Terminal as a Workshop

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Navigate, inspect, and debug a Linux server or container with nothing but
  a shell — no GUI, no IDE.
- Write Bash scripts that are actually safe (quoting, error handling, exit
  codes) instead of scripts that work until they don't.
- Understand processes, signals, file descriptors, and permissions well
  enough to debug a hung or crashing service.
- Use the terminal as a genuine productivity tool (pipes, `grep`/`awk`/`sed`,
  `jq`, `tmux`) rather than just a place to type `cd` and `ls`.
- Read and reason about container logs, `top`/`htop`, `strace`, and `journalctl`
  output — the exact skills you'll lean on constantly in Part 6 (Infrastructure).

### 2. Prerequisites
Modules 1–2. Comfort with a command line in general (you have this from
backend work) — we're deepening, not starting from zero.

### 3. Estimated Study Time
8–10 hours over 4–5 days.

### 4. Difficulty
⭐⭐☆☆☆ (Easy-Medium — mechanics are simple; the value is in habits and
depth of debugging fluency.)

### 5. Why This Matters
Nearly every AI system you build will eventually run on Linux — a Docker
container, a Kubernetes pod, a cloud VM. When a model-serving process hangs,
a GPU runs out of memory, or a container won't start, you diagnose it from
the terminal, often with no GUI available. This is *the* foundational skill
for your platform/infrastructure goal — everything in Part 6 assumes this
fluency without re-teaching it.

---

### 6. Theory

**What is it?**
Linux is a kernel; "a Linux system" is that kernel plus userspace tools
(coreutils, a shell, systemd, etc.). Bash is one shell among several (zsh,
fish) — a program that reads your typed commands, parses them, and executes
other programs. The terminal is just the interface to the shell.

**Why do we need it?**
GUIs don't scale to servers, containers, or automation. Every deployment
pipeline, every Dockerfile, every CI job is fundamentally "a sequence of
shell commands." If you can't operate confidently in a shell, you can't
operate in production AI infrastructure at all — there's no visual dashboard
for "why is this container's memory climbing."

**How does it work? (core mental models)**

**Processes, not "programs":** every running command is a process with a PID,
a parent, an environment, open file descriptors, and a signal handler table.
`kill -9 1234` isn't "closing an app" — it's sending the `SIGKILL` signal to
process 1234, which the kernel handles by terminating it unconditionally
(vs. `SIGTERM`, which asks the process to shut down gracefully — this
distinction matters enormously for how you shut down services safely, and
maps directly to how Kubernetes terminates pods).

**Everything is a file (mostly):** devices, sockets, and pipes are all
represented through the filesystem/file-descriptor abstraction. This is why
`/proc/<pid>/` lets you inspect a running process's memory maps, open files,
and environment just by reading "files."

**Pipes are composition, exactly like function composition:**
```bash
cat access.log | grep "50[0-9]" | awk '{print $7}' | sort | uniq -c | sort -rn
```
Each stage receives the previous stage's stdout as its stdin. This is the
Unix philosophy: small, single-purpose tools, composed via pipes, instead of
one monolithic tool that does everything (compare to how you'd compose
`Stream` operations in Java — same idea, different syntax).

**Bash quoting is the #1 source of subtle bugs:**
```bash
for f in $(ls *.log); do   # BREAKS on filenames with spaces
for f in *.log; do          # correct — let the shell glob, don't parse ls output
```
Unquoted variables undergo word-splitting and globbing — this is why
`rm -rf $DIR/*` with an empty `$DIR` can delete your entire filesystem
(`rm -rf /*`). Always quote: `rm -rf "${DIR:?}"/*`.

**When should Bash be used (vs. Python for scripting)?**
Bash for: quick composition of existing CLI tools, glue in Dockerfiles/CI,
one-liners. Python for: anything with real logic, error handling, or that
will be maintained/tested long-term. A Bash script over ~30–50 lines with
any conditional complexity is usually a sign you should have written it in
Python instead.

---

### 7. Mental Models

**Model 1 — "The shell is a REPL for composing tiny programs."** Learn the
~15 tools that compose well (`grep`, `awk`, `sed`, `sort`, `uniq`, `cut`,
`jq`, `xargs`, `find`) rather than memorizing giant single-tool flag lists.

**Model 2 — "Exit codes are the real return value."** Every command returns
an integer exit code (0 = success, non-zero = failure) — this is what `&&`,
`||`, and `if` in scripts actually check, and it's what CI systems use to
decide pass/fail. `echo $?` right after any command shows you its exit code.

**Model 3 — "Signals are Java's interrupt mechanism, but process-wide and
OS-level."** `SIGTERM` ≈ a graceful shutdown request (like catching
`InterruptedException` and cleaning up); `SIGKILL` ≈ the JVM being killed by
the OS with zero chance to clean up. Kubernetes sends `SIGTERM` then waits a
grace period before `SIGKILL` — understanding this now means Part 6's pod
lifecycle discussion needs no re-explanation.

---

### 8. Visual Explanation (described)

**Diagram: "Anatomy of a piped command"**
```
cat access.log --stdout--> grep "50[0-9]" --stdout--> awk '{print $7}' --stdout--> sort | uniq -c
      (process 1)             (process 2)                 (process 3)              (processes 4,5)
```
Each `|` connects one process's stdout file descriptor directly to the
next process's stdin file descriptor — the kernel handles this via an
actual pipe buffer, and all processes run *concurrently*, streaming data
through, not sequentially waiting for each other to fully finish.

---

### 9–16. Recommended Resources

**Reading order:**
1. **"The Linux Command Line" by William Shotts (free online)** — Part 1
   and Part 3 (scripting) specifically; skip the beginner GUI-comparison
   chapters.
2. **`man bash` — specifically the "QUOTING" and "PARAMETER EXPANSION"
   sections** — yes, actually read the man page for these two sections;
   it's the definitive fix for the quoting bugs described above.
3. **"Shell scripting with shellcheck"** — install `shellcheck` and run it
   on every script you write from now on; it catches the exact class of
   quoting/globbing bugs above automatically.
4. **`man 7 signal`** — the process signal model, directly relevant to
   graceful shutdown patterns you'll implement in every service later.

**Official documentation:** GNU Coreutils manual, `man` pages generally —
Linux's own docs are unusually good and often better than third-party
tutorials.

**Tools worth deliberately learning (not just "knowing exist"):**
`tmux` (persistent terminal sessions — critical once you're SSH'd into a
remote box in Part 6/8), `jq` (JSON on the command line — you'll use this
constantly once you're inspecting LLM API responses/logs), `htop`,
`journalctl`, `strace`.

**GitHub repos:** `koalaman/shellcheck` (read the wiki of common issues it
catches — each one is a mini lesson in shell pitfalls).

---

### 17. Exercises

1. Write a one-liner that finds the 10 most common HTTP status codes in a
   log file, using `awk`/`grep`/`sort`/`uniq -c`.
2. Write a Bash script with `set -euo pipefail` at the top; explain what
   each of the three flags does and why production scripts should always
   include this line.
3. Start a long-running process (`sleep 300 &`), find its PID, send it
   `SIGTERM`, confirm it exits; repeat with a script that traps `SIGTERM`
   and logs "shutting down gracefully" before exiting — this is the exact
   pattern a well-behaved containerized service uses.
4. Use `jq` to extract a nested field from a sample JSON API response (use
   any public API's JSON output) without writing any Python.

### 18. Mini-Project
**Build:** `logtail-alert.sh` — a Bash script that tails a log file,
greps for ERROR-level lines, and prints an alert (with timestamp) when
errors exceed N per minute. Must include `set -euo pipefail`, pass
shellcheck cleanly, and handle the log file not existing yet (graceful
wait/retry, not a crash).

### 19. Production Project
**Build:** A "Linux debugging runbook," `debugging-runbook.md`, documenting
your own step-by-step process for diagnosing a service that's:
(a) using too much memory, (b) hanging/unresponsive, (c) crash-looping. For
each scenario, list the exact commands you'd run in order (`ps`, `top`,
`/proc/<pid>/status`, `journalctl -u <service>`, `dmesg`, `strace -p <pid>`)
and what each command's output would tell you. This becomes a real artifact
you'll actually use once you're debugging Docker/Kubernetes services in
Part 6 and 8 — and it's the kind of document that stands out in a platform
engineering portfolio or interview.

---

### 20–21. Practice & Interview Questions

1. What's the difference between `SIGTERM` and `SIGKILL`, and why does it
   matter for how you design a service's shutdown handler?
2. Explain what happens, step by step, when you run `cat file | grep foo`.
3. Why is `rm -rf $DIR/*` dangerous if `$DIR` is unset, and how do you guard
   against it?
4. What does `set -euo pipefail` do, and why would you add it to every
   production Bash script?
5. A process shows high memory in `top` but you suspect a memory leak vs.
   just heavy legitimate usage — how would you start investigating from the
   command line?

---

### 22. Common Mistakes
- Parsing `ls` output in scripts (breaks on spaces/special characters) —
  use globs instead.
- Unquoted variable expansions leading to word-splitting bugs.
- Not checking exit codes / not using `set -e`, so scripts silently
  continue after a failed command.
- Treating `SIGKILL` as an acceptable default shutdown method (skips
  cleanup, can corrupt state, drops in-flight requests).
- Writing complex logic in Bash instead of switching to Python once
  conditionals/data structures get involved.

### 23. Debugging Exercise
Given a script that occasionally fails silently in CI but works locally,
find that it's missing `set -e` (so an early failing command doesn't stop
the script) and that a variable is unquoted, causing a globbing issue only
present in the CI environment's directory structure. Fix both.

---

### 24. Checklist
- [ ] I'm comfortable composing 4+ commands in a pipeline without looking
      anything up
- [ ] Every script I write starts with `set -euo pipefail` and passes
      shellcheck
- [ ] I can explain signals well enough to design a graceful shutdown handler
- [ ] I've used `jq`, `tmux`, and `journalctl`/`strace` at least once, for real
- [ ] I've written the debugging runbook production project

### 25. Summary
The terminal is a workshop of small, composable tools connected by pipes —
fluency here isn't about memorizing flags, it's about internalizing the
Unix philosophy of composition, understanding processes/signals well enough
to debug real production issues, and writing Bash that's safe by default
(`set -euo pipefail`, proper quoting). This is foundational, not optional,
for everything in Part 6 and 8.

### 26. Next Steps
Module 4: **Networking, HTTP, REST & JSON** — the request/response
mechanics underneath every API call you'll make to a model provider, vector
DB, or your own services, taught with an eye toward the parts (timeouts,
retries, streaming) that matter specifically for LLM API integration later.

---
