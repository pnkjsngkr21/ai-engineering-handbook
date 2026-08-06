# Part 6, Module 5: Load Balancing & Horizontal Scaling for Model Serving

> Takes a single quantized, vLLM-served model instance (Modules 1–4)
> and scales it across multiple GPU instances to handle real production
> traffic volume — reusing Part 1, Module 13's fan-out/circuit-breaker
> patterns (`resilient-gateway`) at the model-serving layer instead of
> the backend-service layer.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain why a single GPU instance, however well-tuned (Modules 2–4),
   has a hard ceiling on throughput, and why horizontal scaling —
   adding more instances — is the only way past that ceiling, not
   further per-instance tuning.
2. Distinguish load-balancing strategies generically applicable to any
   backend service (round robin, least connections) from the
   GPU-serving-specific strategy that actually matters most here:
   **routing by current KV cache/queue occupancy**, not just raw
   request count.
3. Explain why naive round-robin load balancing is a worse fit for
   GPU-backed inference than it is for typical stateless backend
   services, connecting the reason directly to Module 1's KV cache
   variability argument.
4. Reuse `resilient-gateway`'s (Part 1, Module 13) circuit-breaker and
   fan-out patterns for a GPU-serving fleet, and identify what's
   different about failure modes here versus typical backend service
   failures (a GPU running out of memory mid-request is a different
   failure shape than a typical service timeout).
5. Design and load-test a multi-instance vLLM deployment behind a
   GPU-aware load balancer, measuring real throughput scaling (not
   assuming linear scaling with instance count) as instance count
   increases.
6. Extend `ai-api-platform` and `gpu-capacity-planner` with real
   horizontal-scaling capability, feeding directly into Module 6's
   Kubernetes-based autoscaling.

## 2. Prerequisites

- Part 6, Modules 1–4 — this module scales what those modules built and
  tuned; without them, there's nothing correctly-tuned to scale
  horizontally in the first place.
- Part 1, Module 13 (System Design Fundamentals, `resilient-gateway`) —
  this module's core artifact directly extends that fan-out/
  circuit-breaker pattern rather than reinventing resilience patterns.
- Part 1, Module 11 (`ai-api-platform`) — the platform being extended.

## 3. Estimated Study Time

10–12 hours across 2–3 sessions.

## 4. Difficulty

★★★★☆ (4/5) — The load-balancing concepts are approachable given your
backend background; correctly reasoning about GPU-specific occupancy
signals, and actually load-testing to measure real (sub-linear) scaling
rather than assuming ideal scaling, is where the genuine difficulty is.

---

## 5. Why This Matters

You already know, from years of backend work, that a single service
instance has a throughput ceiling and that horizontal scaling behind a
load balancer is the standard answer. What this module teaches is
specifically what's *different* about applying that standard answer to
GPU-backed model serving — and the difference matters enough that
naively reusing a generic load balancer configuration (round robin
across instances, as you might for a stateless REST API) produces
measurably worse results than a GPU-aware one, for reasons directly
traceable to Module 1's KV cache argument.

This matters for the interview-readiness thread specifically because
this is exactly the kind of question that separates "has deployed a
model behind an API" from "understands model-serving infrastructure":
an interviewer asking "how would you scale this to handle 10x traffic"
is listening for whether you reach for "add more instances behind a
generic load balancer" (a correct but incomplete answer) or "add more
instances behind a load balancer that routes based on each instance's
current KV cache occupancy, because request cost isn't uniform the way
it is for a typical stateless API call" (the answer that shows you
understand what's actually different about this domain).

---

## 6. Theory

### 6.1 The hard ceiling on a single instance

Modules 2–4 optimized a single vLLM instance's throughput as far as
scheduling (continuous batching), memory management (PagedAttention),
and precision (quantization) can take it. But a single GPU has a fixed
amount of VRAM and a fixed memory bandwidth — there is a real ceiling
on how many concurrent requests one instance can serve, no matter how
well-tuned, and `gpu-capacity-planner` (Module 1) already computes
roughly where that ceiling sits for a given model/traffic profile.
Beyond that ceiling, the only lever left is **horizontal scaling**:
running multiple instances and distributing traffic across them. This
is the same principle as any backend capacity ceiling (Part 0, Module
13/14) — the new content in this module is what's specific to
distributing traffic across GPU-backed instances rather than
stateless backend services.

### 6.2 Why request cost is not uniform — and why that breaks naive round robin

A typical stateless backend service handles requests whose cost is
usually roughly comparable to each other (a REST endpoint doing a
similar amount of work per call), which is exactly the assumption
round-robin load balancing (send request N+1 to the next instance in
sequence, regardless of what each instance is currently doing) is built
on. LLM inference requests violate this assumption badly: two requests
sent to different instances at the same moment can have wildly
different cost, because — per Module 1's KV cache argument — a
request's actual VRAM/compute footprint depends on its context length
and expected generation length, not just "one request equals one unit
of work." Round-robin can easily send a long-context, long-generation
request to an instance that's already near its KV cache ceiling from
serving other long requests, while a nearly-idle instance sits
underutilized simply because it wasn't "next in the rotation."

**The fix: route by current occupancy, not by rotation.** A GPU-aware
load balancer should track each instance's current queue depth and/or
estimated KV cache utilization (vLLM instances can expose this as a
real-time metric) and route new requests to the least-loaded instance
by that measure — the model-serving analog of "least connections"
load balancing rather than round robin, but with a GPU-specific
occupancy signal instead of a generic connection count, because
connection count alone still doesn't capture the KV-cache-driven cost
variability that round robin misses.

### 6.3 Failure modes are shaped differently than typical backend failures

`resilient-gateway` (Part 1, Module 13) already handles the general
shape of downstream failure: timeouts, error responses, and circuit-
breaking to stop cascading failure when a downstream dependency is
unhealthy. GPU-backed instances fail in a way that's the same *pattern*
but a different *specific cause* worth naming: an instance running out
of VRAM mid-request (because concurrent KV cache usage exceeded what
was provisioned for, per Module 1's capacity-planning arithmetic) is a
distinct failure mode from a typical backend service's slow database
query or network timeout, and it has a specific operational
consequence — an out-of-memory event can affect *all* currently
in-flight requests on that instance simultaneously, not just the one
that triggered it, because the KV cache is shared infrastructure across
all requests currently being served by that instance. This means a
circuit breaker for a GPU-serving instance should trip on
memory-pressure signals specifically (not just latency/error-rate
signals as `resilient-gateway` originally tracked), and a tripped
instance's in-flight requests need a defined recovery behavior (retry
against a different, healthy instance) rather than simply failing them
outright — reusing `resilient-gateway`'s retry/fallback machinery, with
a new, GPU-specific trigger condition added to it.

### 6.4 Scaling is sub-linear, and you must measure by how much

It's tempting to assume that doubling instance count roughly doubles
throughput. In practice, this is close to true only up to a point —
shared infrastructure (the load balancer itself, any shared model-
weight-loading storage, network bandwidth between the load balancer and
instances) introduces overhead that grows with instance count, meaning
real scaling is sub-linear, with the exact shape depending on your
specific deployment's shared-infrastructure bottlenecks. This is a
direct extension of `batching-benchmark`'s discipline (Module 2):
**measure your own scaling curve empirically** — run your load test at
1, 2, 4, and 8 instances and plot actual throughput, rather than
assuming a linear projection from a single-instance number. Knowing
where your own deployment's scaling curve starts to flatten is exactly
the kind of concrete, evidence-based capacity-planning fact that
distinguishes real production experience from a theoretical estimate.

---

## 7. Mental Models

1. **"A single GPU's ceiling is real and fixed — past it, more
   instances is the only lever left, not further per-instance tuning."**
2. **"Round robin assumes uniform request cost; LLM requests aren't
   uniform — route by occupancy, not by rotation."**
3. **"A GPU instance running out of memory takes down every in-flight
   request on it at once — that's a different failure shape than a
   single slow database call."**
4. **"Scaling is sub-linear past some point — measure your own curve,
   don't project linearly from one instance."**

---

## 8. Visual Explanation

```
   NAIVE ROUND ROBIN (assumes uniform request cost):

   req 1 (short) -> instance A
   req 2 (long, big KV footprint) -> instance B
   req 3 (short) -> instance A
   req 4 (long, big KV footprint) -> instance B  <- B is now near its
                                                     KV cache ceiling;
                                                     A is comparatively
                                                     idle, but round
                                                     robin doesn't know

   OCCUPANCY-AWARE ROUTING (Section 6.2):

   req 1 (short)  -> checks occupancy -> instance A (least loaded)
   req 2 (long)   -> checks occupancy -> instance A (still least loaded)
   req 3 (short)  -> checks occupancy -> instance B (now least loaded)
   req 4 (long)   -> checks occupancy -> instance C (least loaded of all)
                     -- load distributed by ACTUAL current cost, not turn order

   ──────────────────────────────────────────────────

   SCALING CURVE -- measure, don't assume linear:

   throughput
        │                       ┌── real curve, sub-linear
        │                  ┌────┘   past some instance count
        │             ┌────┘
        │        ┌────┘
        │   ┌────┘
        │ ┌─┘  ┌ ─ ─ ─ ─ ─ ─ ─ ─  <- naive linear projection
        │┌┘  ─
        └──────────────────────────► instance count
         1    2    4    8
```

---

## 9. Recommended Resources

1. **vLLM documentation — production deployment / distributed serving
   guidance** — the current, official reference for multi-instance
   deployment patterns and available occupancy metrics.
2. **Your own Part 1, Module 13 (`resilient-gateway`) code** — this
   module's core artifact is a direct extension of it; re-read your own
   circuit-breaker implementation before adding the memory-pressure
   trigger condition from Section 6.3.
3. **NGINX/Envoy load-balancing documentation — least-connections and
   custom load-balancing strategies** — practical reference for
   implementing occupancy-aware routing beyond round robin, if you're
   building on an existing proxy rather than a fully custom balancer.
4. **Your own Part 0, Module 12/13 material (async programming, system
   design fundamentals)** — the queuing-theory intuition from those
   modules transfers directly to reasoning about occupancy-based
   routing here.

---

## 10. Exercises

1. Deploy two or more vLLM instances (can be smaller/cheaper instances
   for this exercise, since the point is the routing logic, not raw
   scale). Confirm both serve requests correctly in isolation before
   adding a load balancer.
2. Implement naive round-robin routing across your instances and run a
   load test with a heterogeneous (short and long) request-length mix.
   Measure per-instance occupancy over time and confirm you observe
   Section 6.2's imbalance.
3. Implement occupancy-aware routing (using vLLM's exposed queue/KV
   cache metrics) and re-run the same load test. Measure whether
   per-instance occupancy is now meaningfully more balanced, and
   whether aggregate throughput or tail latency improved.
4. Extend `resilient-gateway`'s circuit breaker with a memory-pressure
   trigger condition specific to GPU instances. Simulate an instance
   approaching its KV cache ceiling and confirm the breaker trips and
   in-flight-affected requests are retried against a healthy instance.
5. Run your load test at 1, 2, 4, and (if feasible) 8 instances, and
   plot the actual throughput scaling curve. Identify, from your own
   data, roughly where it starts to flatten, and hypothesize why, given
   your specific deployment's shared infrastructure.

---

## 11. Mini-Project

**`scaling-curve-report`**: a short report containing your Exercise 5
throughput-vs-instance-count plot, alongside a hypothesis for what
shared-infrastructure bottleneck causes the sub-linearity you observed,
and a recommendation for the instance count at which further scaling
stops being cost-effective for your specific deployment and traffic
profile (using `gpu-capacity-planner`'s cost model, Module 1, applied
per-instance).

---

## 12. Production Project: `gpu-fleet-gateway`

### Scope

Extend `resilient-gateway` (Part 1, Module 13) into `gpu-fleet-gateway`:

- Implements occupancy-aware routing (Section 6.2) across multiple
  vLLM instances, using each instance's real-time queue depth/KV cache
  utilization as the routing signal, not round robin or plain
  least-connections.
- Extends the existing circuit breaker with a memory-pressure trigger
  condition (Section 6.3) specific to GPU instances, alongside the
  original latency/error-rate triggers, with defined retry-against-a-
  healthy-instance behavior for requests affected by a tripped
  instance.
- Ships `scaling-curve-report` (Mini-Project) as a standing,
  re-runnable load test, so any future change to instance count,
  quantization level (Module 4), or model choice can be re-benchmarked
  for its actual (not assumed) scaling behavior.
- Feeds real occupancy and scaling data back into `gpu-capacity-planner`
  (Module 1), so its recommendations account for actual measured
  multi-instance behavior, not just single-instance theoretical
  capacity.

### Explicit extension point

**Part 6, Module 6 (Kubernetes-based Scaling & Deployment)** will take
`gpu-fleet-gateway`'s occupancy metrics as the direct input to
autoscaling policy — deciding *when* to add or remove GPU instances
automatically, using exactly the same occupancy signal this module
established as the correct one to route on.

---

## 13. Practice & Interview Questions

1. Why does round-robin load balancing, which works reasonably well for
   typical stateless backend services, perform worse for LLM-inference
   serving specifically?
2. What signal should a GPU-aware load balancer route on instead of
   simple round robin or connection count, and why?
3. How does a GPU instance running out of memory differ, in failure
   shape, from a typical backend service timeout, and what does that
   difference imply for circuit-breaker design?
4. Why shouldn't you assume linear throughput scaling as you add more
   GPU instances? What would you actually measure to find your real
   scaling curve?
5. How does this module's `gpu-fleet-gateway` reuse `resilient-gateway`
   from Part 1, and what's genuinely new versus what's inherited?

---

## 14. Common Mistakes

- **Using plain round-robin or generic least-connections load
  balancing for GPU-backed inference** — misses the KV-cache-driven
  cost variability that makes LLM requests fundamentally non-uniform,
  unlike typical stateless service requests.
- **Treating a GPU out-of-memory event like a generic service timeout**
  — missing that it can affect every in-flight request on that instance
  simultaneously, which changes the correct recovery behavior.
- **Assuming linear scaling with instance count** — leads to
  under-provisioning or over-provisioning based on a projection instead
  of your own measured curve.
- **Building a parallel resilience system instead of extending
  `resilient-gateway`** — duplicating circuit-breaker/retry logic
  instead of adding a new trigger condition to infrastructure you
  already trust and have tested.
- **Scaling instance count without re-running `quantization-eval-bench`
  or `batching-benchmark`** — a fleet-level configuration change (more
  instances, different routing) can interact with per-instance tuning
  decisions in ways worth re-verifying, not assuming still hold.

---

## 15. Debugging Exercise

Your multi-instance vLLM deployment, load-balanced with plain
round-robin, shows good aggregate throughput on average, but tail
latency (p99) is much worse than a single well-tuned instance's tail
latency was in Module 2's testing — even though average per-instance
utilization looks healthy across the fleet.

Using Section 6.2, walk through: (a) why "average utilization looks
healthy" can coexist with badly skewed tail latency under round-robin
routing (hint: think about what happens to the small subset of
requests that land on an instance currently serving several other
long-context requests, even while the fleet average looks fine), (b)
the specific metric (per-instance occupancy over time, not just
fleet-average utilization) that would reveal the actual imbalance, and
(c) the concrete fix — switching the routing strategy, not adding more
instances, since more instances under the same flawed routing logic
would not fix the underlying skew.

---

## 16. Checklist

- [ ] I can explain why round-robin load balancing is a worse fit for
      LLM inference than for typical stateless services, tracing the
      reason to KV cache cost variability.
- [ ] My load balancer routes based on real occupancy (queue depth/KV
      cache utilization), not rotation or plain connection count.
- [ ] My circuit breaker has a GPU-specific memory-pressure trigger,
      distinct from and in addition to latency/error-rate triggers.
- [ ] I've measured my own multi-instance scaling curve (1, 2, 4+
      instances) rather than assuming linear scaling.
- [ ] `scaling-curve-report` documents a real, measured recommendation
      for cost-effective instance count for my specific deployment.
- [ ] `gpu-fleet-gateway` is built as an extension of `resilient-gateway`,
      not a parallel system.

---

## 17. Summary

A single GPU instance has a real, fixed throughput ceiling; past it,
horizontal scaling — more instances behind a load balancer — is the
only lever left, and this is exactly your existing backend-scaling
instinct applied correctly. What's genuinely new here is that LLM
inference requests are not uniform in cost the way typical stateless
requests are, which makes round-robin a measurably worse choice than
occupancy-aware routing, and that a GPU instance's out-of-memory
failure has a different shape (affecting all in-flight requests at
once) than a typical service timeout, requiring a dedicated
circuit-breaker trigger. Scaling itself is sub-linear past some point,
and the correct instance-count decision comes from measuring your own
deployment's real scaling curve, not from projecting linearly — the
same "measure, don't assume" discipline running through this entire
Part, now applied to fleet-level capacity.

---

## 18. Next Steps

Next: **Part 6, Module 6 — Kubernetes & Terraform for AI Infrastructure**,
which automates what this module did by hand: provisioning GPU
instances, deploying vLLM behind `gpu-fleet-gateway`, and — using this
module's occupancy metrics directly — automatically scaling instance
count up and down as real traffic demands, with infrastructure defined
as versioned, reviewable code rather than manual setup.

Reply "continue" for Module 6, or flag anything to go deeper on.
