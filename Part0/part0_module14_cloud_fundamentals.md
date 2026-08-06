# PART 0 — Prerequisites
## Module 14: Cloud Fundamentals (AWS-first)

---

### 1. Learning Objectives
By the end of this module you will be able to:
- Explain the core AWS primitives (EC2, S3, VPC, IAM, RDS, Lambda) well
  enough to design a deployment architecture, not just click through the
  console.
- Reason correctly about IAM (identity and access management) — the single
  most safety-critical cloud skill, and a common source of real security
  incidents when misunderstood.
- Understand networking fundamentals in the cloud (VPCs, subnets, security
  groups) as an extension of Module 4's networking knowledge into a
  virtualized environment.
- Choose appropriately between compute options (EC2, Lambda, containers on
  Fargate/EKS) based on workload shape — directly relevant to Part 6 and 8.
- Estimate and reason about cloud cost trade-offs, since this is a
  practical daily concern for any AI SaaS builder (Part 9/11) and shows up
  in platform-engineering interviews.

### 2. Prerequisites
Modules 4, 5, 13 (networking, Docker, system design). This module maps
those concepts onto AWS's specific implementation.

### 3. Estimated Study Time
14–18 hours over 7–9 days. This is the final and one of the denser Part 0
modules — it's the direct foundation for Part 6 and Part 8.

### 4. Difficulty
⭐⭐⭐☆☆ (Medium — you likely have some cloud exposure already; the depth
here is in IAM correctness and networking model precision, areas often
under-learned even by engineers who "use AWS daily.")

### 5. Why This Matters
Every deployment in Part 8 and every infrastructure topic in Part 6 (cloud
GPUs, Kubernetes, Terraform) assumes this foundation. For your specific
long-term goal — platform/infrastructure engineering at an AI lab — cloud
fundamentals plus Kubernetes/Terraform (Part 6) is the technical core of
that entire career path. This module is where that path really begins.

---

### 6. Theory

**What is it?**
"The cloud" is, at its core, someone else's data center exposed to you
via APIs — compute (virtual machines, containers, serverless functions),
storage (object storage, block storage, databases), and networking (virtual
networks, load balancers), all provisioned programmatically instead of by
racking physical hardware.

**Why do we need it (vs. your own servers)?**
Elasticity (scale compute up/down on demand instead of over-provisioning
for peak), managed services (a managed Postgres/RDS instance handles
backups/patching/failover you'd otherwise build yourself), and global
reach (deploy near users worldwide without owning data centers there).

**Core primitives, precisely:**

**EC2 (compute):** virtual machines. You choose an instance type (CPU/
memory/GPU ratio matters a lot for AI workloads — Part 6 goes deep on GPU
instance types specifically) and pay per hour/second of usage.

**S3 (object storage):** stores files (objects) as key-value pairs in
"buckets," not a filesystem — no directories in the traditional sense
(though prefixes simulate them). Used for: model artifacts, uploaded
documents (for RAG in Part 4), static frontend assets, logs/backups.
Extremely durable (11 nines), not designed for low-latency random access
the way a local disk or database is.

**VPC (Virtual Private Cloud) — networking, precisely:**
A VPC is your own isolated virtual network within AWS. Inside it:
- **Subnets** — IP address ranges within the VPC, marked "public" (has a
  route to an internet gateway) or "private" (no direct internet route —
  your databases and internal services should live here).
- **Security groups** — stateful, instance-level virtual firewalls (rules
  like "allow inbound TCP 443 from anywhere," "allow inbound 5432 only from
  the app's security group") — this is your primary access-control
  mechanism day to day, and directly extends Module 4's networking model
  into a virtualized, software-defined context.
- **Route tables** — determine where traffic from a subnet is allowed to
  go (to the internet gateway, to a NAT gateway for private-subnet
  outbound access, to a peered VPC, etc.).

**IAM (Identity and Access Management) — the single most important
security concept in this module:**
IAM controls **who** (users, roles, services) can do **what** (actions,
like `s3:GetObject`) on **which resources** (a specific bucket, or `*` for
all). The critical mental model: **IAM policies are deny-by-default, and
an explicit `Deny` always overrides any `Allow`.** The most common,
genuinely dangerous real-world mistake is over-broad permissions
(`"Action": "*", "Resource": "*"`) granted "to make it work" without
narrowing later — this is a recurring root cause of real cloud security
incidents (leaked credentials with overly broad permissions doing far more
damage than they should have). **Principle of least privilege**: grant only
the specific actions on the specific resources actually needed, nothing
more — this is a discipline to build now, not retrofit later.

**IAM Roles vs. Users — a distinction worth being precise about:** a
**user** is a persistent identity (a person, or occasionally a long-lived
service credential). A **role** is a temporary identity assumed by
something (an EC2 instance, a Lambda function, a person via SSO) —
**roles are strongly preferred for anything running inside AWS**, because
they issue short-lived, automatically-rotated credentials instead of
long-lived access keys that can leak and be reused indefinitely if
compromised. This single practice (roles over long-lived access keys) is
one of the highest-leverage cloud security habits you can build.

**Compute choice — matching workload shape to service (relevant directly
to Part 6/8):**

| Workload shape                                    | Best fit                                  |
|------------------------------------------------------|----------------------------------------------|
| Long-running, stateful, full OS control needed        | EC2                                          |
| Short, event-triggered, stateless, infrequent          | Lambda (serverless functions)                 |
| Containerized app, need orchestration but not full k8s | ECS/Fargate                                   |
| Containerized, need full Kubernetes portability/control| EKS (Part 6 covers this in depth)             |
| GPU-heavy model inference/training                     | EC2 GPU instances, or managed services covered in Part 6 (SageMaker, or self-managed vLLM on GPU EC2) |

**When should you NOT default to the most "modern"/serverless option?**
Serverless (Lambda) has cold-start latency and execution time limits —
poor fit for long-running model inference or anything needing persistent
GPU access. For AI workloads specifically, this pushes many teams back
toward container-based (Fargate/EKS) or raw EC2 GPU instances rather than
serverless-everything — a genuinely important, AI-specific deviation from
generic "serverless is always better" advice.

---

### 7. Mental Models

**Model 1 — "IAM is deny-by-default; be as narrow as the task requires,
always."** Treat every policy you write as a candidate security incident
if it's too broad — narrow scope is the default discipline, not an
afterthought.

**Model 2 — "Roles, not long-lived keys, for anything running inside
AWS."** If you find yourself hardcoding an AWS access key into an
application's config, stop and ask whether a role would work instead
(almost always, yes).

**Model 3 — "A VPC's public/private subnet split is Module 4's networking
model, virtualized."** Public subnets face the internet (like a DMZ);
private subnets (databases, internal services) should have no direct
inbound internet route, reachable only via internal traffic or specific
gateways.

---

### 8. Visual Explanation (described)

**Diagram: "A typical 3-tier VPC layout"**
```
Internet
   |
[ Internet Gateway ]
   |
[ Public Subnet: Load Balancer, NAT Gateway ]
   |
[ Private Subnet: App servers / containers ]
   |
[ Private Subnet (isolated): Database (RDS) ]
```
Traffic flows inward from the internet only as far as the load balancer;
app servers reach the database directly (via security group rules) but
have no direct inbound path from the public internet; outbound internet
access for app servers (e.g., calling an LLM provider API) routes through
a NAT gateway in the public subnet, since private subnets have no direct
internet route of their own.

---

### 9–16. Recommended Resources

**Reading order:**
1. **AWS official documentation — "VPC User Guide" and "IAM User Guide"**
   — read these directly; they're thorough and authoritative, if a bit
   dense — worth the investment given how safety-critical IAM correctness
   is.
2. **"AWS Well-Architected Framework" (aws.amazon.com/architecture/well-
   architected)** — specifically the Security and Reliability pillars —
   AWS's own best-practices framework, genuinely useful and current.
3. **"Designing Data-Intensive Applications" (Kleppmann)** — Chapter 8
   (distributed systems trouble) pairs well here, reinforcing Module 13's
   failure-mode thinking in a cloud-specific context.
4. **A Cloud Guru or AWS's own free "AWS Cloud Practitioner Essentials"**
   course — as a structured primer if you want guided video content
   alongside the reading; treat it as supplementary, not primary.

**Official documentation:** docs.aws.amazon.com (primary reference for
everything in this module).

**GitHub repos:** `aws-samples` organization on GitHub has many reference
architectures worth skimming for real IAM policy and VPC configuration
examples (via Terraform, which you'll formalize in Part 6).

---

### 17. Exercises

1. Design (write out, don't necessarily provision) an IAM policy granting
   only `s3:GetObject` and `s3:PutObject` on a single specific bucket
   prefix — no wildcard resources — and explain why this is meaningfully
   safer than a broader policy.
2. Draw (describe in writing) a VPC layout for a simple web app: public
   subnet with a load balancer, private subnet with app servers, isolated
   private subnet with a database — explain the security group rules
   needed at each boundary.
3. Explain, in your own words, why an EC2 instance should assume an IAM
   role rather than have long-lived access keys baked into its
   configuration or code.
4. Compare, for a hypothetical "run a scheduled batch job every night"
   workload, the trade-offs of Lambda vs. a scheduled EC2/Fargate task —
   consider execution time limits, cold starts, and cost.

### 18. Mini-Project
**Build:** Using the AWS free tier (or a well-documented dry-run/plan-only
approach if you'd rather not provision real resources yet — Terraform in
Part 6 will make this properly reproducible), provision a minimal VPC with
public/private subnets, a security group allowing only necessary traffic,
and one EC2 instance in the private subnet reachable only via a bastion
host or Session Manager (not a direct public IP) — document every
decision and its security rationale in a README.

### 19. Production Project
**Build:** `cloud-architecture-doc` — a written cloud architecture
proposal (the kind you'd present in a design review) for deploying
`convo-api` (Module 9) to AWS:
- A described VPC layout (public/private subnets, security groups) with
  explicit justification for each boundary
- An IAM policy set following least privilege for the application's actual
  needs (database access, S3 access for any file uploads, no more)
- A compute choice (EC2 vs. Fargate vs. Lambda) with explicit reasoning
  tied to the workload's actual shape (long-running FastAPI service →
  probably not Lambda; container-friendly → Fargate is a strong
  contender, explicitly discussed against a raw EC2 alternative)
- A cost estimate (rough, back-of-envelope, using AWS's public pricing) for
  a stated traffic assumption, connecting back to Module 13's capacity
  estimation skill
- A section explicitly identifying what you'd change to make the design
  production-grade for a real company (redundancy, monitoring, backups)
  vs. what's acceptable for a portfolio/demo deployment

This document is genuinely interview-relevant (a strong artifact to bring
up when asked to design a deployment) and directly sets up Part 8's actual
deployment work and Part 6's infrastructure deep-dive.

---

### 20–21. Practice & Interview Questions

1. Explain IAM's deny-by-default model and the principle of least
   privilege, with a concrete example of an overly broad policy and how
   you'd narrow it.
2. Why are IAM roles preferred over long-lived access keys for workloads
   running inside AWS?
3. Explain the difference between a public and private subnet, and why a
   database should typically live in a private (often further-isolated)
   subnet.
4. Given a workload description (e.g., "process an occasional file upload
   asynchronously" vs. "run a persistent chat API"), choose an appropriate
   compute service and justify it.
5. What's a NAT gateway for, and why do private-subnet resources need one
   for outbound (but not inbound) internet access?

---

### 22. Common Mistakes
- Overly broad IAM policies (`"Resource": "*"`) granted for convenience and
  never narrowed.
- Long-lived access keys hardcoded into application code or committed to
  git (a direct callback to Module 2's "accidentally committed a secret"
  scenario, now with real cloud-account consequences).
- Placing databases in public subnets with direct internet-facing security
  group rules.
- Defaulting to Lambda for long-running or GPU-heavy AI workloads where
  execution limits and cold starts make it a poor fit.
- Ignoring cost estimation entirely until a surprise bill arrives.

### 23. Debugging Exercise
Given a scenario where an application can't reach the internet from a
private subnet (e.g., failing to call an external LLM API), diagnose that
there's no NAT gateway (or its route isn't configured in the private
subnet's route table), distinguishing this from a security-group-rule
problem — walk through exactly which AWS console/CLI checks would confirm
the root cause.

---

### 24. Checklist
- [ ] I can write a least-privilege IAM policy without defaulting to
      wildcards
- [ ] I understand why roles are preferred over long-lived keys, and use
      that by default
- [ ] I can describe a public/private subnet VPC layout and justify each
      boundary
- [ ] I can match a workload's shape to an appropriate compute service
      (EC2/Lambda/Fargate/EKS) with real reasoning
- [ ] I've completed `cloud-architecture-doc` for `convo-api`

### 25. Summary
Cloud fundamentals extend Module 4's networking model and Module 13's
system-design thinking into AWS's specific primitives — with IAM's
least-privilege discipline as the single highest-stakes concept in this
module. Matching compute service to workload shape (and knowing when
serverless is *not* the right default for AI workloads) is a recurring
judgment call you'll exercise constantly in Parts 6 and 8.

### 26. Next Steps — Part 0 Complete
**Congratulations — Part 0 (Prerequisites) is complete.** You now have the
full engineering foundation this handbook assumes: Python fluency, git
discipline, Linux/Bash comfort, HTTP/networking depth, containerization,
data-layer competence (SQL/Redis), frontend basics (JS/TS/React/Next.js),
a Python backend framework (FastAPI), testing discipline, tooling hygiene,
async programming mastery, system design vocabulary, and cloud
fundamentals.

**Part 1 — Software Engineering** is next: Clean Architecture, SOLID,
design patterns, dependency injection, logging, monitoring/observability,
caching (deepened), background workers/queues, authentication/
authorization, real CI/CD with GitHub Actions, security, performance/
profiling, rate limiting, and API design — synthesizing Part 0's tools into
production-grade software engineering discipline, before Part 2 begins
teaching AI/ML from first principles.

---

**Reply "continue" to begin Part 1, Module 1, or let me know if you'd like
to pause here, revisit anything in Part 0, or reorder given your interview
timeline.**
