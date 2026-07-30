# 2Care.ai — Round 2: DevOps & Cloud Technical Interview
### 60 minutes · 15 questions · ~4 minutes per question
### Calibration: 4 years of experience — solid, hands-on fundamentals and honest reasoning under pressure. Not expected to have staff-level breadth across every AWS service, but expected to have *actually operated* the core primitives (Terraform state, Kubernetes scheduling, IAM, networking, CI/CD) rather than only read about them.

---

## Format note for the interviewer

This round is deliberately broader than the take-home review — it tests whether the candidate's Terraform/Kubernetes/AWS fluency is real operational knowledge or vocabulary picked up from documentation. Roughly two-thirds of the questions are general DevOps/cloud fundamentals any strong platform engineer should have opinions on; the rest are lightly anchored to the 2Care.ai context so the candidate can reason concretely rather than in the abstract. Push for **specifics**: exact flag names, exact failure sequences, exact commands — a candidate who stays at the level of "Kubernetes handles that automatically" everywhere is a weaker signal than one who says "I don't fully remember the flag name but here's the mechanism."

Suggested pacing: questions 1–8 (fundamentals) at ~3.5 min each, questions 9–15 (scenario/judgment) at ~4.5 min each, leaving a small buffer.

---

## Question 1 — Terraform state locking

**Ask:** "Two engineers run `terraform apply` on the same workspace within seconds of each other. Walk me through exactly what happens, mechanically, assuming your state is in S3 with a DynamoDB lock table."

**Strong answer:** The first `apply` acquires a lock by writing an item to the DynamoDB lock table (a conditional write keyed on the state file's lock ID, so it fails if an item already exists). The second engineer's `apply` attempts the same conditional write, fails because the item exists, and Terraform surfaces a `Error acquiring the state lock` message showing who holds it (identifying info like operation ID, who, and when, sourced from the lock metadata). The second apply does **not** proceed, corrupt the state, or silently queue — it simply errors out immediately. Once the first apply completes (or is interrupted and the lock is force-unlocked), the second engineer can retry. A strong candidate also mentions that `terraform force-unlock` exists for when a lock is stuck (e.g., a crashed CI runner left it dangling) and that using it carelessly is how state actually gets corrupted — the lock isn't the risk, humans bypassing it under pressure is.

**What differentiates strong vs weak:** A weak answer says "Terraform prevents concurrent applies" without describing the actual mechanism (conditional DynamoDB write) or the failure UX. A strong answer can also discuss what happens if the S3 backend itself has eventual consistency issues (largely solved since 2020, but worth knowing it used to be a real gotcha) or how Terragrunt's `remote_state` block wires this up per-environment without duplicating backend config.

---

## Question 2 — Kubernetes pod networking

**Ask:** "A pod in your EKS cluster needs to call an external HTTPS API — say, Twilio, not through Bedrock. Trace the path a packet takes from the pod to the public internet and back, naming every component involved."

**Strong answer:** The pod has an ENI-backed IP address (assuming the AWS VPC CNI, which is EKS's default) drawn from the VPC's subnet CIDR — so pod IPs are real VPC IPs, not an overlay network, which is the specific thing that differentiates AWS's CNI from something like Calico's default overlay mode. The packet leaves the pod's network namespace, hits the node's route table, and — since the destination is outside the VPC — gets routed to a NAT Gateway sitting in a public subnate (assuming the pod itself is in a private subnet, which it should be for a PHI-handling workload). The NAT Gateway performs source-NAT translation to its own public IP and sends the packet out through an Internet Gateway attached to the VPC. The response follows the reverse path, with the NAT Gateway performing the reverse translation to route it back to the originating pod IP. A strong candidate also flags the operational implication: NAT Gateway costs money per GB processed and can become a real, easily-missed cost/bottleneck at scale, and that a VPC endpoint (for AWS services) or PrivateLink avoids this entirely by keeping traffic off the NAT path — which is exactly why Bedrock traffic in their own design doesn't take this path at all.

**Follow-up:** "How would that path differ for a call to Bedrock instead of Twilio?" (Expected: no NAT Gateway or Internet Gateway at all — traffic goes through a VPC interface endpoint directly to Bedrock's service, staying entirely inside AWS's network.)

**What differentiates strong vs weak:** Weak candidates say "it goes through the load balancer" (wrong direction — that's inbound) or can't distinguish outbound egress path from inbound ingress path. Strong candidates name NAT Gateway, IGW, and VPC endpoints correctly and unprompted.

---

## Question 3 — IAM least privilege in practice

**Ask:** "Give me a real IAM policy snippet — doesn't need to be perfectly syntactically correct, just structurally right — for a Lambda function that should only be able to read (not write) from one specific DynamoDB table, and explain what's wrong with a common mistake engineers make here."

**Strong answer:** Should sketch something like an identity-based policy attached to the Lambda's execution role, with `Effect: Allow`, `Action` limited to `dynamodb:GetItem` and `dynamodb:Query` (explicitly not `PutItem`/`DeleteItem`/`Scan` if scan isn't needed), and `Resource` scoped to the exact table ARN — not `"*"` and not just the table name without the account/region-qualified ARN. The common mistake to name: using a wildcard resource (`"Resource": "*"`) "to get it working" during development and never tightening it before shipping to production — which is exactly the kind of finding a Checkov/tfsec scan should catch in CI. A strong candidate also mentions that IAM evaluates an explicit `Deny` before any `Allow`, so a permissions boundary or SCP at the account level can enforce a hard ceiling even if an individual role's policy is too permissive — a second line of defense worth having for anything touching PHI.

**What differentiates strong vs weak:** Weak candidates describe IAM in only the vaguest terms ("least privilege means giving minimum access"). Strong candidates can actually write the shape of the policy and name the specific wildcard mistake without being prompted.

---

## Question 4 — Karpenter vs. Cluster Autoscaler

**Ask:** "Your design uses Karpenter over the standard Kubernetes Cluster Autoscaler. What's the actual mechanical difference, not just 'Karpenter is faster'?"

**Strong answer:** Cluster Autoscaler works by scaling existing, pre-defined node groups (ASGs) up or down to match pending pod demand — it's constrained to the instance types and sizes you've pre-configured in those node groups, and provisioning a new node means the ASG launching an EC2 instance from a fixed template. Karpenter instead observes unschedulable pods directly and provisions right-sized nodes on demand from a much broader set of instance types, without needing pre-defined node groups at all — it picks the instance type/size that best fits the pending workload's actual resource requests rather than rounding up to a fixed group's shape. This matters concretely for voice-concurrency bursts: Cluster Autoscaler's node-group scaling still goes through ASG launch mechanics with meaningfully more latency at the margin, while Karpenter's direct EC2 fleet API calls provision faster and more precisely sized, which reduces both cold-start latency and wasted capacity from over-provisioned node-group instance types.

**Follow-up:** "What's a downside of Karpenter's flexibility?" (Expected: less predictable instance-type mix can complicate capacity/cost forecasting and Savings Plan/RI coverage planning compared to a fixed node-group shape.)

**What differentiates strong vs weak:** Weak candidates just repeat "Karpenter is faster and better." Strong candidates can explain *why* mechanically and can name a real tradeoff, not just a one-sided endorsement.

---

## Question 5 — Blue-green vs. canary deployments

**Ask:** "You chose canary (via Argo Rollouts) over blue-green for the agent runtime. What would blue-green have gotten you, and why isn't it the right fit here specifically?"

**Strong answer:** Blue-green flips 100% of traffic from the old environment to the new one at once (after the new environment is fully warmed and validated), giving an instant, clean rollback (flip traffic back) and avoiding any period where two versions serve traffic simultaneously and might behave inconsistently for the same user. But it requires running two full-scale environments concurrently during the cutover, which roughly doubles compute cost during every release, and — the more important reason for this specific system — it exposes 100% of live patients to a new, unvalidated version the instant it flips, with no gradual, metrics-gated exposure window. Canary's core advantage here is that a single new build is exposed to a small slice of real traffic first (10%), with automated metrics-based rollback before the bad build ever reaches the remaining 90% of patients — which directly serves the "zero dropped calls, no bad version reaches most customers" constraint the runtime was designed around.

**What differentiates strong vs weak:** Weak candidates can name both strategies but can't connect the choice to *this system's specific constraint* (patient safety exposure gating). Strong candidates connect the deployment strategy choice directly back to the domain risk.

---

## Question 6 — VPC connectivity options

**Ask:** "Name three different ways to connect a service in your VPC to an AWS-managed service like Bedrock or DynamoDB without traffic touching the public internet, and say when you'd use each."

**Strong answer:** (1) **Gateway VPC Endpoints** — free, route-table-based, but only support S3 and DynamoDB; good default for those two services with zero ongoing cost. (2) **Interface VPC Endpoints (PrivateLink)** — an ENI in your subnet backed by AWS PrivateLink, supports most other AWS services including Bedrock; costs per-hour plus per-GB, but keeps traffic entirely off the public internet and inside AWS's backbone. (3) **VPC Peering** or **Transit Gateway** — for connecting your own VPCs to each other (e.g., a shared-services VPC to per-environment VPCs), not for reaching AWS-managed services directly; Transit Gateway is the better choice over pairwise peering once you have more than a handful of VPCs to connect, since peering doesn't transit (no hub-and-spoke) and becomes an O(n²) mesh problem past a few VPCs.

**What differentiates strong vs weak:** Weak candidates conflate VPC Peering with VPC Endpoints (a genuinely common confusion). Strong candidates cleanly separate "connecting to an AWS-managed service" from "connecting your own VPCs to each other."

---

## Question 7 — Kubernetes probes and cascading failure

**Ask:** "A misconfigured liveness probe is one of the more common ways engineers accidentally cause a self-inflicted outage in Kubernetes. Describe a specific way that happens, and how you'd configure probes correctly for the agent pods in this system."

**Strong answer:** A common failure: liveness probe timeout is set too aggressively short (e.g., 2 seconds) for a pod that's briefly under real load or doing legitimate slow work (e.g., a long LLM inference call blocking the health endpoint's thread pool) — kubelet decides the pod is dead, kills and restarts it, which drops any active session on that pod and, if many pods hit load simultaneously, can trigger a cascading restart storm across the whole deployment right when the system is already under stress. Correct configuration: readiness and liveness probes should be separate and serve different purposes — readiness reflects "should this pod receive new traffic right now" (this is exactly the mechanism used during the graceful-drain sequence, flipping to unhealthy on SIGTERM to stop new sessions without killing the pod), while liveness should only reflect "is this process fundamentally hung/deadlocked," with a generous timeout and failure threshold (multiple consecutive failures required, not one) so a single slow request doesn't trigger a kill. For this system specifically, liveness should almost never fire during normal operation — a genuinely deadlocked agent process should be rare, and the bar for killing a pod mid-conversation should be very high given the cost of dropping a live patient session.

**What differentiates strong vs weak:** Weak candidates conflate readiness and liveness or can't describe a real cascading-failure scenario. Strong candidates connect the answer back to the specific graceful-drain mechanism already in the design.

---

## Question 8 — Secrets management

**Ask:** "AWS Secrets Manager vs. Systems Manager Parameter Store vs. HashiCorp Vault — when would you actually reach for each, and which did you choose here?"

**Strong answer:** Parameter Store (Standard tier) is free and fine for non-sensitive config values and even some secrets, but lacks automatic rotation and has a smaller payload size limit; Secrets Manager costs per secret per month but adds native automatic rotation (e.g., rotating an RDS credential on a schedule via a Lambda rotation function) and tighter IAM-based per-secret access control, which matters more for anything touching PHI-adjacent credentials (DB passwords, third-party API keys like Transcribe/Chime service credentials). Vault is the right choice when you need a cloud-agnostic secrets layer (multi-cloud, on-prem plus cloud) or dynamic short-lived credentials issued per-request rather than static secrets — but it's meaningfully more operational overhead to run and secure yourself, which usually isn't worth it for a single-cloud, all-AWS shop like this one. Given the stack is AWS-native end to end, Secrets Manager is the right default specifically because of automatic rotation and its native integration with the KMS CMK-per-tenant model already in place, rather than for any other differentiator.

**What differentiates strong vs weak:** Weak candidates name the tools without a real decision criterion. Strong candidates lead with rotation as the actual differentiator, not "Secrets Manager is more secure" (an assertion, not a mechanism).

---

## Question 9 — CI/CD supply chain security

**Ask:** "Your pipeline builds and pushes a container image to ECR. Beyond scanning the Terraform, what would you add to make sure the image running in production is provably the exact one that passed CI — not a tampered or substituted one?"

**Strong answer:** Image signing (e.g., Sigstore/cosign) at the end of the CI build, with an admission controller in the cluster (e.g., Kyverno or the AWS-native signature verification via ECR) that refuses to run any image without a valid signature from the CI pipeline's signing identity — this closes the gap where someone could push an unscanned or hand-built image directly to ECR and have it deployed. Pairing this with an SBOM (Software Bill of Materials) generated at build time gives a queryable record of every dependency in the shipped image, which matters both for a CVE response ("which of our running images contain the vulnerable library") and for an auditor asking what's actually running in a PHI-handling pod. A strong candidate also connects this to the git-SHA image tagging already shown in their own CI/CD diagram — signing plus SBOM turns that tag from "a label" into "a provable, tamper-evident chain from git commit to running container."

**What differentiates strong vs weak:** Weak candidates say "we scan images for vulnerabilities" (already covered elsewhere) without addressing the actual question — provenance/tampering, not vulnerability scanning. Strong candidates immediately distinguish scanning from signing/provenance.

---

## Question 10 — Multi-AZ vs. multi-region

**Ask:** "Your design is multi-AZ within a region. Walk me through what actually breaks if the entire AWS region — say, us-east-1 — has an outage, and whether you think multi-region is warranted here."

**Strong answer:** A full regional outage takes down every AZ in that region simultaneously — EKS control plane, all worker nodes, RDS/DynamoDB regional endpoints (DynamoDB Global Tables aside), Bedrock in that region, all of it — so multi-AZ alone provides zero protection against this specific failure. Multi-region would require: duplicating the EKS/data-store infrastructure in a second region, a cross-region data replication strategy for DynamoDB (Global Tables) and S3 (Cross-Region Replication), a global traffic-routing layer (Route 53 health-check-based failover or a global accelerator) to redirect voice/chat traffic to the healthy region, and — critically for this system — a plan for what happens to in-flight sessions during the failover, since session state doesn't survive a region cutover at all cleanly. Whether it's warranted: for a healthcare system handling active patient calls, a full regional outage really would mean total service unavailability with sessions dropped, which is a real business risk — but multi-region roughly doubles infrastructure cost and operational complexity, and AWS regional outages affecting all AZs simultaneously are rare in practice. A defensible answer is "not day one, but it should be an explicit, costed roadmap item revisited as scale and customer contractual requirements grow" rather than either dismissing it or over-building for it prematurely.

**What differentiates strong vs weak:** Weak candidates either say "multi-AZ already covers that" (factually wrong — conflates AZ and region failure) or reflexively say "yes we need multi-region" without weighing the real cost. Strong candidates give a reasoned, costed position either way.

---

## Question 11 — DynamoDB hot partitions

**Ask:** "Your session metadata lives in DynamoDB, presumably keyed by something like session_id or tenant_id. Describe a realistic way this table develops a 'hot partition,' and how you'd detect and fix it."

**Strong answer:** If the partition key is tenant_id rather than something higher-cardinality, one very large or very active tenant (e.g., a hospital system running thousands of simultaneous calls) concentrates a disproportionate share of read/write traffic onto that tenant's specific partition, which DynamoDB's underlying storage nodes can throttle even while the table's overall provisioned/on-demand capacity looks fine in aggregate — showing up as `ProvisionedThroughputExceededException` or elevated latency for just that tenant, not visible in a table-wide average. Detection: CloudWatch metrics broken down would show it, but the more direct signal is per-partition consumed capacity, which isn't natively surfaced by default and often requires either DynamoDB Contributor Insights (a built-in feature for exactly this) or your own tenant-level metric tagging. Fix: change the partition key design to something with naturally higher cardinality — e.g., a composite key of tenant_id + session_id, or a write-sharding suffix appended to the hot tenant's key — so a single tenant's traffic spreads across multiple physical partitions rather than concentrating on one.

**What differentiates strong vs weak:** Weak candidates haven't encountered this in practice and can only describe it abstractly ("uneven load"). Strong candidates name the actual symptom (throttling exception, per-partition not per-table), the actual detection tool (Contributor Insights), and a concrete fix (composite key / write sharding).

---

## Question 12 — Cost optimization: commitment models

**Ask:** "Beyond Spot instances for eval/analytics workloads, what commitment-based savings would you apply to the on-demand customer-facing compute, and what's the risk of over-committing?"

**Strong answer:** Compute Savings Plans (rather than Reserved Instances specifically) are the better fit for EKS worker nodes managed by Karpenter, because Savings Plans apply automatically across changing instance families/sizes/regions within a commitment, whereas RIs are tied to a specific instance type/AZ and would fight against Karpenter's whole point of picking the best-fit instance type per workload. The risk of over-committing: locking into a 1- or 3-year Savings Plan based on today's traffic level, then having actual usage drop (fewer customers than projected, a customer churns) leaves you paying for committed capacity you're not using — the classic accidental-waste failure mode of reserved capacity models. A sound approach is to commit conservatively to a baseline you're highly confident about (e.g., the steady-state 24/7 floor of traffic) and leave burst capacity on-demand or Spot, re-evaluating the commitment level quarterly against actual usage data rather than committing to a large multi-year number upfront based on a growth projection.

**What differentiates strong vs weak:** Weak candidates only know "Reserved Instances save money" without the RI-vs-Savings-Plan distinction or the Karpenter-compatibility angle. Strong candidates name the real risk (locked-in waste) unprompted.

---

## Question 13 — Incident response and postmortems

**Ask:** "Walk me through, step by step, what you personally do in the first 15 minutes after a PagerDuty alert fires for 'EHR write failure rate > 2% over 5 minutes.'"

**Strong answer:** Acknowledge the page immediately (stops the escalation chain from waking a second person unnecessarily), then check the aggregated dashboard (Datadog/Grafana) for the blast radius first — is this one tenant or all tenants, is it correlated with a recent deploy (checking Argo CD's recent rollout history is the very first thing to rule in/out, since a bad deploy is the most common and most reversible cause), and is the failure isolated to EHR writes specifically or symptomatic of a broader outage (Bedrock down, EKS node issues). If it correlates with a recent canary/rollout, the fastest safe action is manually triggering rollback rather than waiting for automated detection to catch up, since a human noticing early can act faster than a metrics window closing. If it's not deploy-related, next check the EHR integration's own error logs for the actual failure mode (auth token expiry, downstream EHR vendor outage, a schema change on their side breaking the write payload) to determine whether this is even something 2Care's side can fix versus needing to notify the customer of a third-party dependency issue. Throughout, the priority order is: stop the bleeding (rollback or failover) before root-causing — a full RCA happens after mitigation, not instead of it.

**Follow-up:** "What goes in the postmortem, and who reads it?" (Expected: timeline, root cause, what worked/didn't in detection and response, concrete action items with owners and deadlines — not just a narrative — circulated at minimum to platform eng and probably to the customer if their patient data flow was affected, per whatever the incident-communication policy says.)

**What differentiates strong vs weak:** Weak candidates jump straight to "check the logs" without a triage framework (blast radius → correlate with recent change → mitigate → root-cause). Strong candidates have an ordered, practiced-sounding process.

---

## Question 14 — Terraform module design and drift

**Ask:** "Someone manually changes a security group rule directly in the AWS console to unblock themselves during an incident, bypassing Terraform entirely. What happens the next time `terraform plan` runs against that resource, and how do you handle it?"

**Strong answer:** `terraform plan` will detect drift — it refreshes state against real AWS resource values, sees the security group's actual rules no longer match what's recorded in state/config, and will show a plan to revert the manual change back to the Terraform-defined configuration (since Terraform is declarative and treats its config as the source of truth, not the live resource). If that plan is applied without anyone noticing the drift first, the manual incident fix gets silently un-done, which is exactly the dangerous scenario — the incident-time fix disappears the next time CI runs an unrelated, unrelated apply. Handling it properly: the manual change should be immediately followed up with a corresponding Terraform PR that codifies the same change properly (ideally within the same incident's remediation window), so the "temporary" console fix becomes the permanent, reviewed, versioned one before the next apply cycle reverts it. A strong candidate also mentions `terraform plan` in CI running on a schedule (not just on PR) specifically to catch this kind of drift proactively, rather than discovering it only when the next real change happens to touch that resource.

**What differentiates strong vs weak:** Weak candidates don't know Terraform will actively try to revert an out-of-band change (a genuinely common misunderstanding — some assume Terraform "leaves it alone"). Strong candidates know this is a real operational risk and describe the process discipline to prevent it, not just the mechanical behavior.

---

## Question 15 — Scaling a WebSocket-heavy service

**Ask:** "5,000 chat sessions per minute with high bursting, over long-lived WebSocket connections. What's the actual bottleneck you'd expect to hit first as this scales — not CPU, something more specific to long-lived connections — and how would you address it?"

**Strong answer:** The first bottleneck is usually **file descriptor / connection limits** per node — each open WebSocket holds a file descriptor and some memory for its connection state, and the default OS/container ulimits are often far lower than what a burst of thousands of concurrent long-lived connections needs, meaning a node can hit "too many open files" well before CPU or memory pressure becomes visible on a standard dashboard. Addressing it: raising `ulimit -n` and the corresponding Kubernetes pod/container resource and network settings explicitly (this is not automatic — it needs deliberate configuration), and more importantly, sizing pods for **connection count, not just CPU/RAM** when configuring the Horizontal Pod Autoscaler — scaling on CPU alone will under-provision for a WebSocket-heavy workload where a pod can be CPU-idle but still holding thousands of open, mostly-quiet connections. A second-order bottleneck worth naming: the Network Load Balancer itself has per-target and per-AZ connection limits, and sticky/long-lived connections interact differently with NLB target deregistration during a deploy than short HTTP requests do — which is exactly why the graceful-drain sequence (readiness-probe flip, not just removing the pod) matters so much more here than it would for a stateless REST API.

**What differentiates strong vs weak:** Weak candidates default to "add more pods" or "scale up CPU" without identifying that connection count, not compute, is the actual constraint for this specific workload shape. Strong candidates name file descriptors/connection-count-based autoscaling unprompted — this is the single clearest signal of real production experience with long-lived-connection systems versus stateless-API experience only.

---

## Scoring guide for the panel

| Band | Description |
|---|---|
| **Strong hire signal** | Answers 11+ of 15 with the mechanism-level detail shown above, unprompted, and volunteers at least one real tradeoff/downside per answer rather than one-sided endorsements. |
| **Hire, with a specific gap to mentor** | Answers 8–10 well; gaps cluster in one area (e.g., strong on Kubernetes/Terraform but weak on cost/FinOps, or vice versa) rather than being scattered — a coherent gap is coachable, a scattered one is a broader fundamentals concern. |
| **Borderline** | Answers 5–7 well; relies on "it just handles that automatically" more than twice; struggles with the quantitative questions (2, 10, 11) specifically, which tend to separate real operational experience from documentation-level familiarity. |
| **Lean reject** | Fewer than 5 answered with real mechanism-level detail; consistently one-sided (never names a downside/tradeoff); cannot do the back-of-envelope math in Q2/Q11 at all. |
