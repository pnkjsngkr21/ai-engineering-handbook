# Part 6, Module 6: Kubernetes & Terraform for AI Infrastructure

> Automates what Module 5 did by hand: provisioning GPU instances,
> deploying vLLM behind `gpu-fleet-gateway`, and — using Module 5's
> occupancy metrics directly — automatically scaling instance count up
> and down as real traffic demands, with infrastructure defined as
> versioned, reviewable code rather than manual setup.

---

## 1. Learning Objectives

By the end of this module, you will be able to:

1. Explain why GPU-aware autoscaling needs a **custom metric**
   (occupancy from Module 5), not Kubernetes' default CPU/memory-
   utilization-based autoscaling, and implement that custom-metric
   autoscaling concretely.
2. Provision GPU-backed Kubernetes node pools using Terraform,
   extending Part 0, Module 14's cloud-infrastructure-as-code
   foundation to GPU-specific resource types and constraints.
3. Explain why GPU node provisioning has a meaningfully longer and less
   predictable "warm-up" time than typical CPU-backed node scaling
   (driver installation, model weight loading), and design autoscaling
   policy that accounts for this lag rather than assuming instant
   elasticity.
4. Design a Kubernetes deployment for `gpu-fleet-gateway`'s worker
   instances that correctly handles GPU scheduling constraints (one
   pod per GPU, resource requests/limits for GPU resources
   specifically) — different from typical CPU/memory pod scheduling.
5. Distinguish which parts of this Part 6 stack genuinely benefit from
   Kubernetes/Terraform automation at your current project's scale, and
   which would be over-engineering for now — the same "measure
   before automating" discipline this handbook has applied throughout.
6. Extend `ai-api-platform`'s CI/CD (Part 1, Module 8) with real
   infrastructure-as-code deployment for the GPU serving fleet.

## 2. Prerequisites

- Part 6, Module 5 (`gpu-fleet-gateway`) — this module automates its
  deployment and scaling; without Module 5's occupancy metric, there's
  nothing correct to autoscale on.
- Part 0, Module 14 (Cloud Fundamentals) — this module extends that
  infrastructure-as-code foundation to GPU-specific resources.
- Part 1, Module 8 (CI/CD & GitHub Actions) — this module's deployment
  automation extends that existing pipeline rather than building a
  separate one.

## 3. Estimated Study Time

12–15 hours across 3 sessions.

## 4. Difficulty

★★★★☆ (4/5) — Kubernetes and Terraform each have real learning curves
independent of AI-specific content; this module assumes you can follow
official Kubernetes/Terraform onboarding material for the general
mechanics and focuses specifically on what's different for GPU
workloads.

---

## 5. Why This Matters

Everything built in Modules 1–5 works, but it was deployed and scaled
by hand — you ran vLLM instances, configured routing, and manually
observed the scaling curve. That's the right way to *learn* the
underlying behavior (per this handbook's whole "measure it yourself
before automating" philosophy), but it's not how production systems are
actually operated: a real deployment needs infrastructure defined as
reviewable, versioned code, and scaling decisions made automatically in
response to real traffic, not manually triggered by an engineer
watching a dashboard.

This matters specifically for the platform-engineering track you're
pursuing, because "can configure Kubernetes autoscaling" is a generic
skill many candidates have; "understands why GPU autoscaling needs a
different metric and a different warm-up assumption than typical
CPU-based autoscaling" is the specific, differentiated skill that
connects your general infrastructure competence to actual AI-systems
experience — and it's a direct, natural extension of the ops
background you already came into this bootcamp with, applied to a
domain (GPU scheduling) that's newer to you than Kubernetes itself.

---

## 6. Theory

### 6.1 Why default Kubernetes autoscaling doesn't work for GPU serving

Kubernetes' Horizontal Pod Autoscaler (HPA), by default, scales based
on CPU or memory utilization. This is the wrong signal for GPU-backed
model serving for the same reason Module 5 rejected round-robin load
balancing: request cost isn't uniform, and CPU/memory utilization on
the pod running a vLLM instance doesn't capture what actually
determines whether that instance is near its real capacity ceiling —
KV cache occupancy does (Module 1, Section 6.3; Module 5, Section 6.2).
A vLLM pod can show modest CPU utilization while being genuinely near
its KV cache ceiling (GPU-side memory, not CPU/host memory, is the
actual constraint), which means CPU-based HPA would fail to scale up
exactly when scaling up is needed.

**The fix:** Kubernetes supports custom-metrics-based autoscaling
(via the custom metrics API, commonly backed by Prometheus — directly
reusing Part 1, Module 4's `observability-stack`). Wire
`gpu-fleet-gateway`'s occupancy metric (Module 5) into
`observability-stack`, expose it as a queryable metric, and configure
the HPA to scale on *that* metric's threshold instead of CPU/memory —
the autoscaler now makes the same decision a human watching Module 5's
occupancy dashboard would make, automatically.

### 6.2 GPU node warm-up: elasticity is not instant

A typical CPU-backed autoscaling event — spin up a new pod on an
existing node, or even provision a new node — is usually fast enough
(seconds to low minutes) that autoscaling policy can treat it as
close to instantaneous relative to traffic changes. A GPU node is
meaningfully slower to become useful: provisioning a new GPU-backed
instance may require installing/verifying GPU drivers, and — this is
the part specific to model serving — the vLLM instance running on that
new node must load the full model's weights into VRAM before it can
serve any request, which for a large model is a real, non-trivial
amount of time (directly related to Module 1's VRAM arithmetic: more
parameters means more bytes to load from storage into VRAM before
serving the first request).

This has a concrete design consequence: **autoscaling policy for GPU
fleets must scale up proactively, ahead of predicted demand, or accept
a real latency/availability gap during the warm-up window** — reactive
scaling (wait until occupancy crosses a threshold, then start
provisioning) that works fine for fast-warming CPU services leaves a
real gap here. Two practical mitigations, both worth implementing
rather than choosing between only in theory: maintain a small buffer of
already-warm, already-model-loaded instances above your measured
baseline demand (a direct cost/availability trade-off, calculable using
`gpu-capacity-planner`), and/or scale proactively based on
leading-indicator signals (e.g., queue depth trend, not just current
occupancy) rather than waiting for the threshold to be fully crossed.

### 6.3 GPU-specific Kubernetes scheduling

Scheduling a vLLM pod onto a GPU node has constraints that don't apply
to typical CPU/memory-scheduled workloads: a GPU is generally treated
as a resource that can't be fractionally shared across pods the way
CPU cores can be (barring more advanced GPU-sharing/MIG configurations,
which are an advanced topic beyond this module's scope) — meaning your
pod spec's resource requests/limits for GPU resources typically mean
"one full GPU, dedicated to this pod," and the Kubernetes scheduler
must be configured (via device plugins) to be aware of GPU resources as
a distinct schedulable resource type, not just an extension of CPU/
memory. Getting this wrong (e.g., failing to set explicit GPU resource
limits) risks the scheduler co-locating GPU pods incorrectly or leaving
GPU capacity unaccounted for in scheduling decisions — a subtle,
production-relevant misconfiguration that's easy to miss coming from a
purely CPU/memory-scheduling background.

### 6.4 Terraform for GPU infrastructure: same discipline, GPU-specific resources

Provisioning GPU-backed Kubernetes node pools via Terraform is, at the
level of *discipline*, identical to Part 0, Module 14's general
infrastructure-as-code approach: declare the desired state (node pool
type, GPU instance type, count, autoscaling bounds) in version-
controlled configuration, apply it through a reviewed, auditable
pipeline, rather than manually clicking through a cloud console. What's
GPU-specific is the resource types and constraints themselves (GPU
instance types are a distinct, often more limited and more
expensive/quota-constrained resource category than general-purpose
compute instances, and cloud providers frequently have separate quota
limits specifically for GPU capacity that must be requested/verified
in advance, unlike general compute capacity which is typically far more
readily available on demand).

### 6.5 When this automation is (and isn't) worth it yet

Consistent with this module's "measure before automating" thread: full
Kubernetes/Terraform automation, with custom-metric autoscaling and
proactive warm-up buffering, is real operational complexity, and it is
worth exactly as much investment as your actual traffic and
reliability requirements justify — a personal project or an early-stage
prototype almost certainly does not need this module's full machinery,
and manually managing a small, fixed `gpu-fleet-gateway` deployment
(Module 5) is the correctly-scoped choice at that stage, mirroring
Module 3's Ollama-vs-vLLM scoping argument one level up the stack. The
threshold for justifying this module's automation is the same kind of
explicit, measured decision as everywhere else in this Part: real,
variable production traffic, a genuine reliability/SLA requirement, and
team size large enough that manual scaling operations become a real
operational burden rather than an occasional convenience.

---

## 7. Mental Models

1. **"CPU/memory utilization doesn't tell you a GPU-backed pod's real
   headroom — KV cache occupancy does; autoscale on that, not the
   default."**
2. **"A GPU node isn't 'ready' the moment it boots — it's ready once
   the model's weights are loaded into VRAM; plan for that lag."**
3. **"A GPU is a whole-unit schedulable resource, not a fractional one
   like CPU cores — the scheduler needs to know that explicitly."**
4. **"Automate this once your traffic and reliability needs justify the
   complexity — not because Kubernetes is the 'serious' way to do
   things."**

---

## 8. Visual Explanation

```
   DEFAULT HPA (wrong signal for GPU serving):

   pod CPU utilization: ▁▂▂▁▂  <- looks fine, low
   pod GPU KV cache occ: ████████████████████  <- actually near ceiling!
                          HPA never fires -- scaling on the wrong metric

   CUSTOM-METRIC HPA (Section 6.1):

   observability-stack (Part 1 M4) <- gpu-fleet-gateway occupancy metric
              │
              ▼
   Kubernetes custom metrics API
              │
              ▼
   HPA scales on OCCUPANCY threshold, not CPU/memory
              │
              ▼
   new GPU node provisioned (Terraform-managed pool)
              │
   ⏳ WARM-UP LAG (Section 6.2): driver check + full model
      weight load into VRAM -- NOT instant, unlike a typical
      CPU pod spin-up
              │
              ▼
   instance joins gpu-fleet-gateway's routing pool,
   now genuinely ready to serve

   mitigation: keep a small pre-warmed buffer above
   measured baseline demand, sized via gpu-capacity-planner
```

---

## 9. Recommended Resources

1. **Kubernetes official documentation — Horizontal Pod Autoscaler
   with custom metrics, and GPU device plugins** — the authoritative,
   current reference for both mechanisms this module relies on.
2. **NVIDIA — Kubernetes device plugin documentation** — the specific,
   practical reference for GPU-aware scheduling (Section 6.3).
3. **Terraform — cloud-provider GPU instance/node-pool resource
   documentation** (for your chosen cloud provider) — current,
   authoritative reference for the specific resource blocks you'll
   write; provider APIs and instance-type names change, so use the
   live docs over a cached tutorial.
4. **Your own Part 0, Module 14 and Part 1, Module 4/8 material** —
   this module is a direct, GPU-specific extension of your existing
   cloud infrastructure-as-code and observability/CI-CD foundations;
   re-read before starting rather than treating this as unfamiliar
   territory.

---

## 10. Exercises

1. Write a Terraform configuration provisioning a GPU-backed node pool
   (even a minimal, single-node one for learning purposes), including
   explicit GPU resource type and count.
2. Wire `gpu-fleet-gateway`'s occupancy metric into `observability-stack`
   (Part 1, Module 4) and confirm it's queryable via the metrics
   pipeline you already have.
3. Configure an HPA using that custom metric instead of CPU/memory, and
   trigger a scale-up event by driving occupancy above your configured
   threshold in a test load.
4. Measure, with real timestamps, how long it actually takes from
   "autoscaler decides to scale up" to "new instance is genuinely ready
   to serve requests" (through model weight loading) for your specific
   model size, and compare that lag against your traffic's actual
   burstiness to judge whether reactive-only scaling would leave a real
   gap.
5. Using Section 6.5's criteria, write a one-paragraph justification
   for whether your own current project's traffic and reliability needs
   actually justify this module's full automation yet, or whether
   Module 5's manual `gpu-fleet-gateway` management is still the
   correctly-scoped choice.

---

## 11. Mini-Project

**`warmup-lag-report`**: using Exercise 4's measured data, a short
report stating your specific model's real cold-start-to-ready latency,
and a concrete recommendation (buffer size via `gpu-capacity-planner`,
or a proactive-scaling trigger point) for how your autoscaling policy
should account for that lag given your traffic's actual burst pattern.

---

## 12. Production Project: `gpu-infra` (Terraform + Kubernetes automation)

### Scope

Build `gpu-infra`, extending `ai-api-platform`'s CI/CD (Part 1, Module
8):

- Terraform configuration provisioning GPU-backed Kubernetes node
  pools, with autoscaling bounds (min/max instance count) as
  version-controlled, reviewable parameters — not manually adjusted
  infrastructure.
- Kubernetes deployment manifests for `gpu-fleet-gateway`'s vLLM
  instances, with correct GPU resource requests/limits (Section 6.3)
  and device-plugin configuration.
- Custom-metric HPA configuration, scaling on `gpu-fleet-gateway`'s
  occupancy metric (Module 5) via `observability-stack` (Part 1,
  Module 4) — not default CPU/memory-based autoscaling.
- A pre-warmed instance buffer, sized using `warmup-lag-report`
  (Mini-Project) and `gpu-capacity-planner` (Module 1), to absorb
  Section 6.2's warm-up lag for your measured traffic burstiness.
- Integrated into `ai-api-platform`'s existing CI/CD pipeline (Part 1,
  Module 8), so infrastructure changes go through the same
  tiered/reviewed pipeline as application code changes — no separate,
  undocumented deployment process for infrastructure.

### Explicit extension point

This is Part 6's infrastructure-automation capstone artifact; **Part
6's remaining modules** (LiteLLM as a multi-provider gateway, high
availability/disaster recovery) will run on top of `gpu-infra`'s
provisioned fleet, and **Part 8 (Production AI)** will extend this
same CI/CD-integrated infrastructure-as-code discipline to the full
application stack, not just the GPU-serving layer.

---

## 13. Practice & Interview Questions

1. Why is default Kubernetes HPA (CPU/memory-based) the wrong choice
   for autoscaling GPU-backed model-serving pods?
2. What custom metric should drive GPU autoscaling instead, and how
   would you wire it into an existing observability pipeline?
3. Why does GPU node autoscaling need to account for a real warm-up
   lag that typical CPU-based autoscaling can mostly ignore?
4. What's specifically different about scheduling a GPU-backed pod in
   Kubernetes compared to a typical CPU/memory-scheduled pod?
5. How would you decide whether a given project's traffic actually
   justifies this module's full Terraform/Kubernetes automation, versus
   a simpler, manually-managed deployment?

---

## 14. Common Mistakes

- **Using default CPU/memory-based HPA for GPU-serving workloads** —
  scales on the wrong signal, missing exactly the cases where KV cache
  occupancy is high but CPU utilization looks fine.
- **Assuming GPU nodes scale up as fast as CPU nodes** — leaving a real
  availability/latency gap during the model-weight-loading warm-up
  window if autoscaling policy is purely reactive.
- **Treating GPUs as fractionally schedulable like CPU cores** without
  explicit resource requests/limits and device-plugin configuration —
  risking scheduler misallocation.
- **Building this full automation for a low-traffic personal or
  prototype project** — real operational complexity paid for with no
  corresponding benefit, the same over-engineering mistake Module 3
  warned against for vLLM itself.
- **Deploying infrastructure changes outside the existing CI/CD
  pipeline** — creating an undocumented, unreviewed parallel process
  for infrastructure that application code doesn't get to skip.

---

## 15. Debugging Exercise

Your GPU fleet is configured with custom-metric autoscaling that
correctly triggers on occupancy, but during a real traffic spike, users
still experience a period of elevated latency and some failed requests
before the fleet "catches up."

Using Section 6.2 and `warmup-lag-report`, walk through: (a) why a
correctly-triggering autoscaler can still leave a real gap if scaling
is purely reactive, (b) the specific measurement (cold-start-to-ready
latency, compared against how fast the traffic spike actually grew)
that would explain the gap, and (c) the concrete fix — is it a bigger
pre-warmed buffer, an earlier (more proactive) scaling trigger point, or
both, and how would you decide between them using `gpu-capacity-planner`'s
cost model?

---

## 16. Checklist

- [ ] I can explain why default CPU/memory-based HPA is the wrong
      signal for GPU-serving autoscaling.
- [ ] My autoscaler scales on `gpu-fleet-gateway`'s real occupancy
      metric, wired through `observability-stack`, not CPU/memory.
- [ ] I've measured my own model's real cold-start-to-ready latency,
      not assumed it's negligible.
- [ ] My Kubernetes deployment correctly requests/limits GPU resources
      explicitly, with device-plugin configuration.
- [ ] My Terraform configuration provisions GPU node pools as
      version-controlled, reviewable infrastructure.
- [ ] I have an explicit, measured justification (Section 6.5) for
      whether this module's automation is actually warranted for my
      current project's scale.
- [ ] `gpu-infra` deployment changes go through the same CI/CD pipeline
      as application code, not a separate manual process.

---

## 17. Summary

Automating Modules 1–5's manually-run deployment means confronting two
genuinely GPU-specific facts that default Kubernetes tooling doesn't
handle out of the box: autoscaling needs a custom occupancy metric,
because CPU/memory utilization doesn't reflect real GPU-serving
headroom, and GPU nodes have a real, measurable warm-up lag (driver
setup plus model weight loading) that purely reactive scaling doesn't
account for, requiring a proactive buffer or leading-indicator trigger
instead. Terraform provisioning and Kubernetes scheduling for GPUs
follow the same infrastructure-as-code discipline as any other cloud
resource (Part 0, Module 14), with GPU-specific resource types and
quota constraints as the concrete new content — and, as throughout this
Part, the automation itself is only worth building once real traffic
and reliability requirements justify its genuine operational
complexity.

---

## 18. Next Steps

Next: **Part 6, Module 7 — LiteLLM & Multi-Provider Gateways**, unifying
hosted APIs, self-hosted vLLM, and Ollama behind one routing layer with
consistent fallback and cost-tracking, extending `llm-client-core`'s
adapter roster (Modules 2–3) with a proxy-layer alternative and
comparing the two approaches directly.

Reply "continue" for Module 7, or flag anything to go deeper on.
