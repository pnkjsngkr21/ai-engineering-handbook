# PART 1 — Software Engineering
## Module 6: Background Workers & Queues

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Design and implement a job queue system for long-running AI tasks (batch
  embedding, document processing, multi-step agent workflows) using
  Celery or a lighter-weight async task queue, choosing deliberately
  between them.
- Handle task failure, retries, and dead-letter queues correctly for tasks
  that call flaky external services (LLM providers).
- Design idempotent task handlers (connecting directly back to Part 0
  Module 4's idempotency-key discussion) so at-least-once delivery
  semantics never cause duplicate side effects.
- Decide correctly when a task belongs in a background queue versus a
  synchronous request/response endpoint versus a streaming response — a
  genuine architecture judgment call you'll make repeatedly from Part 4
  onward.
- Monitor and observe background job health (queue depth, task duration,
  failure rate) using Module 4's observability foundation.

### 2. Prerequisites
Modules 1–5, and Part 0 Modules 4/6/12 (idempotency, Redis, async).

### 3. Estimated Study Time
10–12 hours over 5–6 days.

### 4. Difficulty
⭐⭐⭐☆☆ (Medium — you likely have some message-queue exposure from
backend work; the AI-specific task design (long-running, cost-bearing,
partially-failing multi-step jobs) is the newer depth.)

### 5. Why This Matters
Many AI workloads are fundamentally not request/response shaped: embedding
10,000 documents for a RAG index (Part 4), running a multi-step research
agent that takes minutes (Part 5), or batch-processing a large document
upload — all belong in a background queue, not a synchronous HTTP request
that would time out or block. This module builds the queue infrastructure
every capstone in Part 11 that involves "processing" (as opposed to
"instant chat") will depend on.

---

### 6. Theory

**What is it?**
A job queue decouples **task submission** (fast, synchronous — "accept
this work and give me a job ID") from **task execution** (slow,
asynchronous, happens on separate worker processes) via a message broker
(Redis, RabbitMQ, or a cloud queue like SQS) sitting between them.

```
[ API: POST /process-document ] --enqueue--> [ Broker (Redis/RabbitMQ) ]
        |                                              |
        v (returns immediately)                        v
   { "job_id": "abc123" }                    [ Worker process(es) pull
                                                jobs and execute them,
                                                independently, at their
                                                own pace ]
```

**Why do we need this (vs. Part 0 Module 4's synchronous/polling
discussion)?** Some tasks genuinely take minutes (a multi-agent research
workflow, Part 5) or need to run at a controlled rate against a
rate-limited API across thousands of items (batch embedding, Part 4) —
holding an HTTP connection open for minutes is fragile and doesn't scale;
a queue with independent workers processing at a controlled, retry-able
pace is the correct architecture.

**Celery vs. lighter alternatives (Python's actual landscape):**
- **Celery** — the mature, feature-rich standard: supports retries, rate
  limiting, scheduled tasks, multiple broker backends, and a large
  ecosystem. Heavier to configure, genuinely production-proven at scale.
- **Lighter async-native alternatives (e.g., `arq`, or a simple
  Redis-backed queue you build yourself with `asyncio`)** — less
  ceremony, integrates more naturally with an already-async FastAPI
  codebase, sufficient for many portfolio/small-to-medium production
  needs.
**When to choose which:** default to Celery for anything genuinely
production-scale with complex scheduling/retry needs (this is what you'd
likely find at a company like DoorDash for background job infrastructure);
consider a lighter async-native queue for smaller, portfolio-scale
projects in this handbook where Celery's configuration overhead isn't
worth it yet. Both are worth knowing — Celery for the "production at scale"
conversation in interviews, a lighter tool for moving fast on your own
projects.

**At-least-once delivery and idempotency, precisely (the critical
correctness concern):**
Nearly all real message queues/brokers guarantee **at-least-once**
delivery, not exactly-once — a task might be delivered and executed more
than once (e.g., a worker crashes after completing the task's side effect
but before acknowledging the message, causing redelivery). **Your task
handlers must be idempotent** — running the same task twice with the same
input must produce the same net effect as running it once. Concretely:
```python
async def process_document_task(document_id: str):
    # Idempotent: check if already processed BEFORE doing the expensive work
    if await is_already_processed(document_id):
        return  # safe no-op on redelivery
    result = await expensive_llm_processing(document_id)
    await save_result(document_id, result)   # use an upsert, not a blind insert,
                                                # so a duplicate execution doesn't
                                                # create a duplicate row or double-charge
    await mark_processed(document_id)
```
This directly reuses Part 0 Module 4's idempotency-key concept, applied to
background tasks instead of HTTP endpoints.

**Retries, backoff, and dead-letter queues:**
```
Task fails (e.g., LLM provider 500 error)
   |
   v
Retry with exponential backoff + jitter (Module 4's pattern, reused)
   |
   v (after N retries, still failing)
Move to a "dead-letter queue" — NOT silently dropped, NOT retried forever
   |
   v
Alert/log for human investigation — a task that fails N times usually
signals either a genuinely broken input or a systemic provider outage,
either of which deserves human attention, not infinite silent retries
```

**When does a task belong in a queue vs. sync request vs. streaming
response? (the recurring architecture judgment call)**

| Shape                                                        | Right architecture         |
|-----------------------------------------------------------------|---------------------------------|
| User waiting live, response needed in seconds                    | Synchronous request, or streaming (Part 0 Module 4) |
| Task takes minutes, user doesn't need to wait live                | Background queue + job status polling or a webhook/notification |
| Bulk processing many items at a controlled rate against a rate-limited API | Background queue with per-worker concurrency limits (Part 0 Module 12's bounded concurrency, applied at the worker level too) |

---

### 7. Mental Models

**Model 1 — "A queue decouples 'accept the work' from 'do the work,' and
that decoupling is exactly what makes long-running, rate-limited, or
bursty AI workloads tractable."**

**Model 2 — "At-least-once delivery is the default reality of nearly
every real queue — design every task handler assuming it might run twice,
always."** This isn't a hypothetical edge case; it's a routine occurrence
under normal operation (worker restarts, network blips), not a rare
failure mode.

**Model 3 — "A dead-letter queue is a deliberate 'stop and ask a human'
mechanism, not a failure of your retry logic."** Infinite silent retries
on a permanently broken task are worse than a bounded number of retries
followed by a visible, actionable failure state.

---

### 8. Visual Explanation (described)

**Diagram: "Task lifecycle with retries and dead-lettering"**
```
Task enqueued
   |
   v
Worker picks up task, executes
   |
   +-- success --> mark done, notify/store result
   |
   +-- transient failure (e.g., LLM 503) --> retry with backoff+jitter
   |         |
   |         +-- succeeds on retry --> done
   |         +-- fails after N retries --> move to dead-letter queue,
   |                                        alert for human investigation
   |
   +-- permanent failure (e.g., malformed input) --> dead-letter
                                                        immediately,
                                                        no point retrying
```

---

### 9–16. Recommended Resources

**Reading order:**
1. **Celery official documentation** — "First Steps," "Task Basics," and
   specifically the "Retrying" and "Idempotent tasks" sections.
2. **`arq` official documentation** (as the lighter async-native
   alternative) — read its README/docs for a contrasting, simpler
   approach, and form your own judgment on when each tool is the better
   fit.
3. **AWS documentation — "Amazon SQS Dead-Letter Queues"** — even if
   you're not using SQS specifically, this is a clear, canonical
   explanation of the dead-letter-queue pattern applicable to any broker.
4. **"Idempotency Keys" (Stripe's engineering blog)** — Stripe's own
   writeup of exactly this pattern in a genuinely high-stakes production
   context (payments) — an excellent, concrete real-world reference.

**Official documentation:** docs.celeryq.dev, `arq`'s GitHub README.

**GitHub repos:** `celery/celery` itself (for real-world usage patterns
in its own test suite), `samuelcolvin/arq`.

---

### 17. Exercises

1. Set up a minimal Celery (or `arq`) worker processing a simple mock
   "long-running" task, submitted from a FastAPI endpoint that returns
   immediately with a job ID, and a second endpoint to poll job status.
2. Deliberately make a task non-idempotent (e.g., blindly appending to a
   list on every execution instead of upserting), simulate redelivery
   (execute it twice with the same input), and observe the incorrect
   duplicated effect — then fix it to be properly idempotent.
3. Implement retry-with-backoff for a task that simulates transient
   failures, and configure it to move to a dead-letter queue after a fixed
   number of attempts, with a log/alert emitted on dead-lettering.
4. Design (on paper) which of the following belongs in a sync endpoint,
   a streaming response, or a background queue, and justify each: (a)
   answering a single chat message, (b) embedding 50,000 documents for a
   new RAG index, (c) a multi-agent research task expected to take 3
   minutes.

### 18. Mini-Project
Add a background task queue to `convo-api` for a "summarize this long
document" feature: the endpoint enqueues a task and returns a job ID
immediately; a worker processes it (using `llm-client-core` from Module 2)
and stores the result; a second endpoint polls job status/result. Ensure
the task handler is properly idempotent.

### 19. Production Project
**Build:** `job-processor` — a production-quality background processing
system, extending `convo-api`:
- Celery (or `arq`) integration with Redis as the broker
- At least two task types: a single-item task (summarize one document)
  and a batch task (process many documents, using `batch-caller`'s bounded
  concurrency from Part 0 Module 12 *within* a single worker task, since a
  worker executing one "batch" job still needs to respect rate limits
  internally)
- Idempotent task handlers, explicitly tested by simulating redelivery
- Retry with backoff+jitter, and a dead-letter queue with alerting (reuse
  Module 4's structured logging/observability)
- A job-status API (submitted → running → succeeded/failed → dead-lettered)
  with a polling endpoint
- Full test suite, including a test that specifically verifies idempotency
  under simulated duplicate execution
- README with a decision-matrix section explicitly justifying, for this
  project's specific tasks, why each belongs in a background queue rather
  than a synchronous endpoint or streaming response

This becomes the background-processing foundation Part 4 (batch embedding
for RAG indexing) and Part 5 (long-running agent workflows) will extend
directly.

---

### 20–21. Practice & Interview Questions

1. Explain why message queues typically guarantee at-least-once, not
   exactly-once, delivery, and what that means for how you must design
   task handlers.
2. Design an idempotent task handler for "charge a user and send them a
   confirmation email" — what specifically would go wrong with a naive,
   non-idempotent version under redelivery?
3. What's a dead-letter queue for, and why is it better than either
   infinite retries or silently dropping a permanently failing task?
4. Given a workload description, decide whether it belongs in a
   synchronous endpoint, a streaming response, or a background queue —
   walk through your reasoning for an ambiguous case.
5. Compare Celery and a lighter async-native task queue — what factors
   would push you toward one or the other for a given project?

---

### 22. Common Mistakes
- Writing non-idempotent task handlers that assume exactly-once delivery,
  causing duplicated side effects (double charges, duplicate database
  rows) under normal, expected redelivery.
- Retrying a permanently broken task forever instead of dead-lettering it
  after a bounded number of attempts.
- Putting genuinely long-running or bulk work into a synchronous HTTP
  request, causing timeouts and fragile user experience.
- Not monitoring queue depth/task duration at all, missing a growing
  backlog until it becomes a user-visible incident.
- Running batch tasks with unbounded internal concurrency, hitting
  provider rate limits from inside a single worker task (forgetting Part
  0 Module 12's bounded-concurrency lesson applies just as much inside a
  background task as inside a request handler).

### 23. Debugging Exercise
Given a background job system where duplicate database rows are
appearing under normal operation (no unusual errors visible in logs, just
occasional duplicates), diagnose that task redelivery (a normal,
expected occurrence under at-least-once delivery) is triggering a
non-idempotent insert — fix by making the handler check-then-upsert
instead of blindly inserting.

---

### 24. Checklist
- [ ] I can explain at-least-once delivery and why every task handler
      must be idempotent as a result
- [ ] I've built and tested a properly idempotent task handler, including
      a redelivery simulation test
- [ ] I've implemented retry-with-backoff and dead-lettering with
      alerting
- [ ] I can decide, with clear reasoning, whether a workload belongs in a
      sync endpoint, streaming response, or background queue
- [ ] I've completed `job-processor` with full observability integration

### 25. Summary
Background queues decouple task submission from execution, essential for
AI workloads that are long-running, bulk, or bursty against rate-limited
APIs. The single most important correctness discipline is designing every
task handler assuming at-least-once (not exactly-once) delivery —
idempotency isn't optional. `job-processor` is the background-processing
foundation Part 4's batch embedding work and Part 5's long-running agent
workflows will build directly on top of.

### 26. Next Steps
Module 7: **Authentication & Authorization** — securing every API you've
built so far, including AI-specific concerns like per-user rate limiting
and API key management for programmatic/agent access, not just typical
username/password login flows.

---
