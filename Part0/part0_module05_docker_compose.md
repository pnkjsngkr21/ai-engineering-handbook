# PART 0 — Prerequisites
## Module 5: Docker & Docker Compose

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Explain what a container actually is (namespaces + cgroups + a layered
  filesystem) rather than treating Docker as a black box.
- Write production-quality, minimal, multi-stage Dockerfiles for both
  Python and Node/Java services.
- Use Docker Compose to run multi-service local environments (app +
  Postgres + Redis + vector DB) the way you'll need to for nearly every
  project from Part 4 onward.
- Debug a container from the outside (logs, exec, inspect) the way you
  debugged bare processes in Module 3 — same mental models, containerized.
- Understand image layering, caching, and size optimization well enough to
  keep build times and image sizes production-reasonable.

### 2. Prerequisites
Module 3 (Linux/processes) is essential — containers are Linux processes
with restricted views of the system, not a separate "VM-like" thing, and
that mental model only clicks if Module 3 landed.

### 3. Estimated Study Time
10–12 hours over 5–6 days.

### 4. Difficulty
⭐⭐⭐☆☆ (Medium — you likely have some Docker exposure already from backend
work; this module pushes into multi-stage builds, layer caching, and
Compose orchestration depth that many backend engineers skip.)

### 5. Why This Matters
Every project from Part 4 onward runs multiple services locally (your app,
a vector DB, Redis, maybe a local model server) — Compose is how you spin
all of that up with one command. Every production deployment in Part 8
starts from a Docker image. And Part 6 (Kubernetes) is fundamentally "how do
we run containers, but at scale, across many machines" — nothing there
makes sense without solid container fundamentals first.

---

### 6. Theory

**What is it?**
A container is **not a lightweight VM**. It's a regular Linux process that
the kernel restricts using:
- **Namespaces** — give the process its own isolated view of PIDs, network
  interfaces, mounts, hostname, etc. (so `ps` inside a container only shows
  that container's processes, even though the host kernel sees everything).
- **cgroups (control groups)** — limit and account for resource usage (CPU,
  memory) so one container can't starve others.
- **A layered filesystem (overlayfs)** — a container's filesystem is a
  stack of read-only image layers plus one writable layer on top.

This is *why* containers start in milliseconds (no OS boot, just process
creation with restricted namespaces) while VMs take seconds-to-minutes (a
whole kernel boots).

**Why do we need it?**
"Works on my machine" is a dependency/environment-parity problem. A
container packages your app *and* its exact runtime dependencies
(interpreter version, system libraries, environment) into one artifact that
runs identically on your laptop, in CI, and in production. For AI
workloads specifically, this matters even more than typical backend work,
because dependency trees (specific CUDA versions, specific PyTorch builds)
are notoriously fragile — pinning them inside a container is often the only
sane way to get reproducibility.

**How is a Dockerfile actually built? (layer caching mental model)**

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen                 # <- this layer is cached...
COPY . .                             # ...as long as this line's inputs don't change
CMD ["python", "-m", "myapp"]
```

Each instruction creates a new image layer. Docker caches layers by content
hash of their inputs — if `pyproject.toml`/`uv.lock` haven't changed, the
`RUN uv sync` layer is reused from cache, skipping a potentially slow
dependency install. This is *why* the order of Dockerfile instructions
matters enormously: **copy dependency manifests and install dependencies
before copying application code**, so code changes (which happen constantly)
don't invalidate the expensive dependency-install layer.

**Multi-stage builds (the production-quality pattern):**
```dockerfile
FROM python:3.12 AS builder
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

FROM python:3.12-slim AS runtime
WORKDIR /app
COPY --from=builder /app/.venv /app/.venv
COPY . .
ENV PATH="/app/.venv/bin:$PATH"
CMD ["python", "-m", "myapp"]
```
The builder stage has full build tooling (compilers, dev headers); the
runtime stage copies only the *result* (the installed venv) into a slim
final image. This keeps production images small (faster deploys, smaller
attack surface) while still allowing a full build toolchain during the
build step.

**When should you use Docker Compose vs. a single Dockerfile?**
Single container/Dockerfile: one service, one process. Compose: whenever
your **local development** environment needs multiple services talking to
each other (your app + Postgres + Redis + a vector DB) — Compose defines
them all in one YAML file, on a shared network, startable with `docker
compose up`.

**When should containers NOT be used?**
For very low-level GPU/driver work or when you need bare-metal performance
tuning with no isolation overhead at all — rare in typical AI-application
engineering, but worth knowing as an edge case (some GPU-serving setups run
closer to bare metal for this reason, discussed more in Part 6).

---

### 7. Mental Models

**Model 1 — "A container is a process wearing a costume, not a mini
computer."** Everything from Module 3 (signals, PIDs, `/proc`) still
applies *inside* the container — it's the same kernel, restricted view.

**Model 2 — "Docker layers are like Git commits — a stack of diffs, cached
by content hash."** Reordering Dockerfile instructions to put
rarely-changing steps first is directly analogous to structuring commits
so unrelated changes don't invalidate each other's cache/review.

**Model 3 — "Compose is a docker-run flag list turned into readable YAML,
plus a shared virtual network for free."** Every `docker compose` service
gets DNS-resolvable by its service name automatically (`redis://redis:6379`
just works) — this is the single most useful Compose feature and removes
the need to hardcode IPs.

---

### 8. Visual Explanation (described)

**Diagram: "Image layers vs. running container"**
```
Image (read-only layers, shared/cached across containers):
  [ python:3.12-slim base ]
  [ + system deps layer ]
  [ + dependency install layer ]
  [ + app code layer ]

Running container = image layers (read-only) + one writable layer on top
(anything the process writes at runtime lives only in that writable layer,
and disappears when the container is removed unless you use a volume)
```
This is why containers are meant to be **stateless/ephemeral** — anything
that must persist (a database's data) needs an explicit volume mounted
outside the container's writable layer.

---

### 9–16. Recommended Resources

**Reading order:**
1. **Docker's official "Get Started" + "Build" docs** (docs.docker.com) —
   specifically the multi-stage build and build-cache pages.
2. **"Docker Deep Dive" by Nigel Poulton (book)** — the best single
   resource for the namespaces/cgroups/overlayfs internals mental model.
3. **Docker Compose official docs, "Compose file reference"** — you'll
   reference this repeatedly, no need to memorize it.
4. **`docs.docker.com/develop/dev-best-practices`** — official Docker
   guidance on layer ordering, `.dockerignore`, and image size reduction.

**Official documentation:** docs.docker.com is genuinely excellent and
should be your primary reference over third-party tutorials.

**GitHub repos to study:** look at the Dockerfiles in `pydantic/pydantic`
or any well-maintained FastAPI starter template for real multi-stage build
examples; also worth glancing at `docker-library` (official images) repos
to see how base images themselves are constructed.

---

### 17. Exercises

1. Containerize a simple Python script with a naive single-stage
   Dockerfile, note the image size (`docker images`), then rewrite it as a
   multi-stage build and compare sizes.
2. Deliberately reorder a Dockerfile so `COPY . .` happens *before*
   dependency installation, rebuild after a trivial code change, and observe
   (via `docker build` output) that the dependency-install layer had to
   re-run — then fix the ordering and confirm caching kicks in.
3. Write a `docker-compose.yml` with an app service and a Postgres service,
   confirm the app can reach Postgres via its service name (not `localhost`,
   not a hardcoded IP).
4. Use `docker exec -it <container> sh` to get a shell inside a running
   container and use Module 3 skills (`ps`, checking env vars, `cat
   /proc/1/status`) to inspect it from the inside.

### 18. Mini-Project
**Build:** Containerize the `logsage` CLI (from Module 1) with a proper
multi-stage Dockerfile, a `.dockerignore`, and a `docker-compose.yml` that
mounts a local logs directory as a volume so the containerized tool can
analyze logs from the host without rebuilding the image.

### 19. Production Project
**Build:** `local-ai-stack` — a `docker-compose.yml` that stands up a
realistic local development environment you'll actually reuse starting in
Part 4: a FastAPI app container, Postgres, Redis, and a vector database
(e.g., Qdrant or pgvector-enabled Postgres — we'll pick the specific one in
Part 4, so a placeholder service is fine for now). Requirements:
- Named volumes for all stateful services (Postgres data, Redis
  persistence) so data survives `docker compose down`/`up` cycles
- Healthchecks defined for each service, with the app service configured
  to wait for its dependencies' healthchecks before starting
  (`depends_on: condition: service_healthy`)
- Environment variables via a `.env` file (not hardcoded), with a
  `.env.example` committed and the real `.env` gitignored
- A README documenting the architecture and exactly how to bring the stack
  up/down/reset

This exact compose file becomes the local-dev foundation for essentially
every RAG/agent project starting in Part 4 — worth building it carefully
now.

---

### 20–21. Practice & Interview Questions

1. What's the actual difference between a container and a VM, at the
   kernel-mechanism level (namespaces/cgroups vs. a hypervisor virtualizing
   hardware)?
2. Why does instruction order matter in a Dockerfile, and how does that
   relate to build cache invalidation?
3. What problem do multi-stage builds solve, and what would an image look
   like without them (larger size, build tools present in production image,
   larger attack surface)?
4. How does Docker Compose's default network let services reach each other
   by name, and why shouldn't you hardcode container IPs?
5. Where does a container's writable data go, and why do you need volumes
   for anything that must survive a container restart/removal?

---

### 22. Common Mistakes
- Not using a `.dockerignore`, accidentally baking `.git`, `node_modules`,
  or local venvs into the image (bloats size, sometimes leaks secrets from
  local env files).
- `COPY . .` before installing dependencies, destroying build cache on
  every code change.
- Running processes as root inside containers unnecessarily (a real
  security smell — production images should run as a non-root user).
- Treating a container's filesystem as durable storage without a volume,
  then being surprised data vanished after a restart.
- Not setting resource limits (memory/CPU) in Compose/production, letting
  one container starve others on the same host.

### 23. Debugging Exercise
Given a Compose stack where the app container crash-loops on startup
because it starts before Postgres is actually ready to accept connections
(not just "container started," but "Postgres is done initializing"),
diagnose using `docker compose logs` and fix it with a proper healthcheck +
`depends_on: condition: service_healthy` instead of a naive `depends_on`
(which only waits for container start, not readiness) or a fixed `sleep`.

---

### 24. Checklist
- [ ] I can explain namespaces/cgroups/overlayfs well enough to explain why
      containers start fast
- [ ] I write multi-stage Dockerfiles by default for anything going to
      production
- [ ] I order Dockerfile instructions deliberately for cache efficiency
- [ ] I've built a Compose stack with healthchecks and named volumes
- [ ] I've completed the `local-ai-stack` production project

### 25. Summary
Containers are restricted Linux processes, not mini-VMs — namespaces give
process isolation, cgroups enforce resource limits, and a layered
filesystem gives you fast, cacheable builds. Docker Compose extends this to
multi-service local environments with free service-name DNS resolution.
This module's `local-ai-stack` project is the literal foundation you'll
reuse for nearly every hands-on project starting in Part 4.

### 26. Next Steps
Module 6: **SQL, PostgreSQL & Redis** — the data layer every AI application
still needs (conversation history, user data, caching, and — later — vector
search via pgvector), taught assuming your existing relational-DB
competence and focused on the AI-specific access patterns.

---

**Reply "continue" for Module 6, or flag anything to go deeper on.**
