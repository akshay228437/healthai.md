# 2Care.ai — Platform Design: Agent Runtime & Infrastructure
### Model Answer / Answer Key

---

## Clarifying Questions (would ask before finalizing assumptions)

The brief rewards stating these even if unanswered — they materially change the design:

1. Is single-tenant physical isolation (dedicated VPC/account per customer) a **current** enterprise requirement, or a future one we should design an escape hatch for?
2. Does any customer require model outputs to never touch a non-Bedrock model (some EU/health systems mandate a single named vendor for the BAA chain)?
3. Is human escalation routed to 2Care's own clinical staff, or to the customer's? This changes who owns the "stop and ask" boundary in Section 6/7.
4. What's the actual current scale — is 500 concurrent voice calls today's number or a 12-month target? (Materially changes whether EKS+Karpenter is justified now, per the closing paragraph.)
5. Are EHR writes idempotent on the customer's side, or do we need dedup/outbox logic on ours?

---

## Assumptions (stated explicitly, with reasoning)

| Assumption | Value | Why |
|---|---|---|
| Voice concurrency | 500 concurrent calls at peak | Realistic mid-market healthcare voice AI scale; drives the EKS-vs-Fargate and NLB sizing decisions |
| Chat throughput | 5,000 active sessions/min, bursty | Reminder blasts are a known healthcare-ops pattern; drives queue-based smoothing decisions |
| EHR write volume | ~100 writes/sec | Sets the bar for the EHR integration's retry/idempotency design |
| Compliance perimeter | HIPAA BAA-eligible AWS services only | Non-negotiable given PHI on every interaction |
| Isolation model | Logical (shared EKS, tenant-scoped IAM + per-tenant KMS CMK), not physical clusters | Balances cost against compliance; **explicitly the thing to defend to an auditor — see Section 3** |
| Release cadence | 5–10 changes/week | Sets the bar for how heavy the eval gate can be before it becomes a bottleneck |
| Deploy constraint | Zero dropped calls, session state pod-independent | The single hardest constraint in the whole system; shapes Section 2 entirely |

---

## 1. The Stack

**Agent layer**
- **Framework — LangGraph.** Chosen over a bare LLM-calls-in-a-loop approach or CrewAI/AutoGen because clinical workflows are stateful, branching, and need deterministic checkpoints (e.g., "confirm medication name before writing to EHR"). LangGraph's explicit graph state and checkpointing map directly onto that. *Rejected: raw orchestration in application code* — reinventing state machines badly is a known failure mode in regulated agent systems.
- **Chat transport — WebSocket**, for bidirectional low-latency streaming. *Rejected: long-polling/SSE-only* — doesn't support true bidirectional barge-in for chat-with-voice-handoff scenarios.
- **Voice — Chime SDK + Transcribe Medical + Polly.** Transcribe Medical specifically because generic ASR mis-transcribes drug names and clinical terms at a rate that matters for safety. *Rejected: Twilio + generic Whisper* — cheaper, but no medical vocabulary and no BAA-native integration with the rest of the AWS PHI perimeter.
- **Model routing — Bedrock**, with a lightweight in-house router in front of it (small model for intent/FAQ triage, frontier model for clinical reasoning). *Rejected: raw provider APIs (OpenAI/Anthropic direct)* — because PHI would then leave the VPC boundary over the public internet per-request, multiplying the BAA surface across vendors. This is the single highest-leverage decision in the stack for compliance, and it should be named as such.
- **State — DynamoDB (session metadata) + ElastiCache Redis (hot context) + S3 (transcripts/archives).** Three-tier because a single store forces a bad tradeoff between write-latency (Redis needs) and durability/audit (S3 needs).
- **Eval/observability — LangSmith + OpenTelemetry**, feeding Datadog/Langfuse (see Section 4). LangSmith for dev-time tracing; OTel for standardized prod metrics so we aren't locked to one vendor's instrumentation format.

**Infrastructure layer**
- **Compute/orchestration — EKS + Karpenter.** *Rejected: ECS Fargate* — Fargate's per-task scaling lag (30–90s cold starts) is incompatible with voice concurrency bursts where a call must acquire compute in single-digit seconds. Karpenter's just-in-time node provisioning is the actual reason EKS wins here, not "EKS is more standard."
- **Terraform organization — single repo, `modules/` + `environments/`, Terragrunt on top.** Terragrunt specifically to DRY environment configs (dev/staging/prod) without copy-pasted `.tf` files that drift. State in S3 (encrypted, versioned) + DynamoDB lock table — standard, but worth naming because the brief fixes "Terraform" as a constraint and this is where that constraint gets operationalized.
- **CI/CD — GitHub Actions → ECR → Argo CD (GitOps) → Argo Rollouts (canary).** GitOps specifically so that "what's running in prod" is always derivable from a git SHA — critical for an auditor asking "prove what code handled this patient's data on date X."

**What I'd reject and why (the 3–4 decisions that matter most):**
1. Bedrock over direct provider APIs — compliance boundary, not preference.
2. EKS+Karpenter over Fargate — burst-latency requirement, not familiarity.
3. Logical multi-tenancy over per-customer clusters — cost/velocity tradeoff, explicitly revisited in the closing paragraph and Section 3.
4. LangGraph over a custom orchestrator — state-checkpointing requirement for clinical safety, not framework fashion.

---

## 2. The Runtime and Its Harness

**Pre-deployment gate.** Every PR touching prompts/agent code/runtime params triggers an automated eval harness (DeepEval/Ragas) running ~100 simulated patient interactions against a temp EKS worker pool, covering hallucination, edge cases, and safety-evasion attempts. **Hard gate:** overall accuracy < 98% OR any single PHI-leak vector detected → the pipeline fails, the image is never promoted. This number should be treated as a living SLO, re-derived from real incident data every quarter, not a permanent constant.

**Progressive canary.** Argo Rollouts shifts NLB target-group weight: 10% → new release, 90% → stable, for a fixed 10-minute window, then a defined stepped promotion (e.g., 25% → 50% → 100%) gated on live metrics, not a timer alone.

**Automated rollback criteria** (any breach = auto-abort, redirect to stable, no human needed):
- Infra: WebSocket drops / HTTP 5xx > 0.5%
- Conversational: P95 voice turn latency > 1,500ms
- Clinical: repetitive-loop rate > 1% or silent-fallback rate > 2%

**Zero-downtime session handling — the crux of the "zero dropped calls" constraint:**
1. Argo CD sends SIGTERM to the outgoing pod.
2. The pod immediately flips its k8s `readinessProbe` to unhealthy — pulled from the NLB pool, receives zero *new* sessions, but keeps serving its *active* ones.
3. `terminationGracePeriodSeconds: 900` (15 min) — long enough to cover the stated 5–15 min voice session length; the active call finishes naturally on the old pod.
4. Every dialogue turn checkpoints session context to Redis. If a pod hard-crashes (not a graceful SIGTERM — an EC2/node failure), the client's reconnect picks the `session_id` back up from Redis on a fresh pod, mid-conversation, with no perceptible loss beyond the reconnect gap itself.

**What gates a Terraform change to prod:** plan output reviewed in PR, `terraform validate`/`tflint`/`checkov` (policy-as-code for encryption/tagging/IAM-least-privilege) run in CI, apply only via Atlantis/Terragrunt run in the pipeline (never local `apply`), and — for anything touching the KMS/IAM tenant-isolation boundary specifically — a second required reviewer from the security/compliance side, not just platform eng.

---

## 3. Safety in a Regulated Environment

*(This is the section to make concrete — a bullet list of "applies safety policies" doesn't survive an audit. Every claim below should have a mechanism attached.)*

**Tenant isolation — concretely, not just "IAM roles":**
- Namespace-per-tenant within the shared EKS cluster, with Kubernetes NetworkPolicies denying all cross-namespace traffic by default.
- Per-tenant KMS CMK — so a tenant's data is cryptographically, not just logically, unrecoverable by another tenant even if a control-plane bug leaked object paths.
- Tenant ID is threaded through every DynamoDB partition key, S3 prefix, and Redis key namespace, and enforced at the IAM policy layer (not just application logic) via tenant-scoped STS role assumption per request — so a bug in *application code* can't cross the boundary, only a bug in the IAM policy itself could, which is a much smaller and more auditable surface.
- **What this buys an auditor:** you can demonstrate the isolation boundary exists in three independent layers (network, crypto, IAM) rather than asserting "the code checks a tenant_id field."

**Where AWS controls are *not* enough on their own** (the brief explicitly asks this — answer it directly):
- Bedrock's zero-retention guarantee covers *model training*, not runtime logging — a misconfigured LangSmith/Langfuse trace could still capture PHI in a prompt/response pair and ship it to a third-party SaaS dashboard. This requires an explicit PII/PHI redaction pass *before* traces leave the VPC, which is our responsibility, not AWS's.
- IAM least-privilege policies prevent unauthorized *access*, but don't prevent an authorized agent process from being prompt-injected into requesting data outside its intended scope — that requires an application-layer guardrail (see below), because AWS has no concept of "this LLM call is scoped to this patient's chart only."
- AWS's BAA covers infrastructure; it does not cover clinical accuracy. A HIPAA-compliant pipe can still confidently transmit a wrong dosage — that's a Section 4/clinical-quality problem, not an AWS-control problem, and treating it as "covered by the BAA" would be a real gap.

**AI safety / clinical guardrails, concretely:**
- Input: prompt-injection detection (regex + classifier) before the request reaches LangGraph; requests routed through Bedrock Guardrails for policy-violating content.
- Output: a second, cheaper model call scores the response for clinical-safety-relevant claims (dosage, diagnosis language) before it's spoken/sent; anything flagged routes to human-in-the-loop escalation rather than being delivered.
- PHI redaction: a dedicated middleware step scrubs PHI fields from anything destined for non-BAA-covered logging/observability tools, with an allowlist (not blocklist) of what's permitted to leave the perimeter.

**Auditability, concretely:** immutable audit log (CloudTrail + application-level event log to a write-once S3 bucket with Object Lock) capturing who/what/when for every PHI access and every AI-generated clinical action, retained per HIPAA's 6-year minimum, queryable per-tenant for SOC 2 evidence collection without needing to grep raw logs by hand.

---

## 4. Observability

Three tiers, because a green Kubernetes dashboard tells you nothing about whether the agent is clinically safe:

| Level | Focus | Metrics | Tooling |
|---|---|---|---|
| Infrastructure | Compute/network | Pod CPU/RAM, WebRTC packet loss, WS disconnects, 5xx rate | Datadog / Prometheus+Grafana |
| Conversational | Responsiveness | STT latency, TTFT, TTS synth time, barge-in counts | OpenTelemetry + Langfuse |
| Clinical/quality | Task completion & safety | Semantic drift, repetitive loops, silent fallback rate, tool success rate, hallucination score | Langfuse + custom async eval workers |

**2 AM PagerDuty criteria (only things requiring immediate human action):**
- Platform: large-scale WS/WebRTC failure, agent runtime fully unavailable, Bedrock outage, multi-customer blast radius.
- Clinical: EHR write failure rate > 2% over 5 min, tool execution failures hitting live patient workflows, safety-guardrail violation spike.

Everything else (a single tenant's elevated latency, a slow drift in hallucination score) is a **business-hours** ticket, explicitly, so the on-call rotation doesn't get desensitized by noise — a distinction worth stating outright, since an unstated paging policy is how healthcare on-call teams burn out.

---

## 5. Cost

**What actually dominates the bill, in order:**
1. **LLM inference (Bedrock).** Drivers: conversation count/duration, token volume, model choice, tool-call retries.
   - Model routing: cheap model for intent/FAQ, frontier model reserved for clinical reasoning — this alone is typically the largest lever available.
   - Context management: Redis for hot session state, DynamoDB for summarized history, S3 for full archive — avoids resending full conversation history on every turn, which is the single most common silent cost leak in agent systems.
2. **Voice processing (Transcribe Medical + Polly + media streaming).** Voice Activity Detection to avoid transcribing silence; bounding max session audio duration; capping retry attempts on transient STT failures rather than retrying indefinitely.
3. **Compute (EKS).** On-demand for customer-facing agent pods (can't tolerate interruption mid-call); Spot for eval/analytics workloads where interruption is fine. Karpenter provisions/deprovisions to actual demand rather than static reserved capacity.

**Monitoring/alerting:** Cost Explorer + cost allocation tags per customer/environment for chargeback and trend visibility; Langfuse for per-conversation token/cost attribution (so a single misbehaving tenant or prompt version is traceable to a dollar figure, not just an aggregate spike). Alert immediately on: unexpected Bedrock cost spikes, infinite-loop token burn, abnormal voice volume — these are the patterns that turn a $2k/day bill into a $50k/day bill overnight if unmonitored.

**The goal is stated explicitly, not implied:** cost-efficient AI that preserves clinical quality and safety — not the lowest possible number. A model-routing decision that saves 20% on inference but measurably degrades clinical accuracy is the wrong trade, and should be called out as such rather than left as an unstated tension.

---

## 6. Making the Team Faster

**Priorities, in order:**
1. Automated clinical golden-dataset evals on every PR (already the CI gate in Section 2 — the point is making this *fast enough* not to be routinely skipped under the 5–10/week cadence).
2. One-click tenant provisioning (Terraform + Helm pipeline) — every day spent hand-provisioning a new customer is a day not spent on the product, and manual provisioning is also where tenant-isolation misconfigurations creep in.
3. Automated incident-context aggregator — pulling logs, Langfuse traces, and EHR tool logs into one place automatically on alert fire, so an on-call engineer isn't manually correlating three dashboards at 2am.

**Ship first: Synthetic Patient Testing Pipeline** (chosen because it directly de-risks the release cadence that everything else in Section 2 depends on) — detailed one level deeper below.

**Trigger:** any PR touching prompts, agent graph code, or tool definitions.

**Allowed to do autonomously:**
- Spin up an ephemeral, namespaced EKS test environment.
- Run 50+ concurrent synthetic patient personas (confused patient, angry patient, non-English speaker, adversarial/safety-evasion attempts) against the candidate build.
- Score task completion %, PHI-leak indicators, guardrail compliance, latency.
- Post the report as a PR comment.
- **Auto-fail the PR** (block merge) if task-completion score drops >1 point vs. baseline, or guardrail compliance falls below 100%.

**Must stop and ask a human, never autonomous:**
- It never touches real tenant data, real credentials, or the production EHR sandbox — only 100% synthetic personas and dummy EHR endpoints, enforced structurally (the ephemeral namespace has no IAM path to any production secret or real-tenant KMS key, not just an application-level check).
- It cannot merge its own PR, promote its own build, or modify the baseline it's comparing against — those require a human approval regardless of score.
- If TruffleHog/GitGuardian pre-push scanning flags anything resembling real PHI/PII or a leaked credential in the branch, the pipeline halts entirely and pages a human rather than "sanitizing and continuing" — a false positive costing 10 minutes of human review is far cheaper than a false negative shipping real PHI into a test log.

**How you'd know it's making things worse (not just catching bad PRs, but failing itself):** track the synthetic pipeline's own false-negative rate against real production incidents retroactively — if a class of failure reaches production that the synthetic personas should have caught, that's a signal the persona set is stale and needs expanding, reviewed monthly, not something assumed to stay accurate indefinitely.

---

## 7. Getting Better After Go-Live

**Week 1–2 — baseline.** Monitor conversation success rate, human escalation rate, latency, tool failures, patient feedback, AI quality metrics via Langfuse/CloudWatch/customer feedback channels. No changes shipped yet — this window exists purely to know what "normal" looks like for *this* customer's patient population, which varies enough between customers that a global baseline is insufficient.

**Week 3–6 — improve agent behavior.** Target prompts, LangGraph workflow structure, EHR integration edge cases, model selection, based on what week 1–2 baseline actually surfaced (not a generic backlog) — e.g., reducing repeated responses, improving tool-call accuracy, handling scenarios the synthetic personas didn't anticipate.

**Week 6–12 — safe release.** Every change: automated eval (accuracy/safety/latency/cost) → canary (5% → 20% → 50% → 100%, same automated rollback criteria as Section 2) → production. If quality regresses at any step, automatic rollback, no manual gate required to catch it — the same harness that gates initial release gates every subsequent improvement, so "getting better" never bypasses the safety mechanism that made go-live safe in the first place.

---

## Working With AI

**(a) How AI was used, and where it was wrong:**
Used for first-draft structuring (turning the 7 open questions into a scaffold) and for surfacing standard AWS service names/tradeoffs (Bedrock vs. direct APIs, EKS vs. Fargate) faster than manually re-deriving them. Where it got things wrong: it initially suggested a fully generic safety section ("apply guardrails, encrypt data, monitor compliance") without concrete mechanisms — caught by re-reading the brief's explicit sub-question ("where AWS controls are not enough on their own") and noticing the draft never actually answered it; had to go back and force every safety claim to name a specific mechanism (network policy, KMS CMK, redaction middleware) rather than a category. It also initially proposed vendor tools (Datadog, LangSmith, Langfuse, Prometheus) without delineating which tool owns which responsibility — caught by cross-checking each tool's claimed metrics against the three-tier observability table and removing overlap.

**(b) One level deeper on the first automation shipped (Synthetic Patient Testing Pipeline):** see Section 6 above — trigger, autonomous scope, guardrails, and degradation-detection are specified there rather than repeated here.

---

## Closing: When This Design Is Wrong

This design is the wrong choice below roughly 50 concurrent voice/chat streams with a platform team under 3 engineers — the operational overhead of EKS, Karpenter, Argo CD, and Argo Rollouts isn't justified by the traffic volume, and the same safety/canary guarantees can be achieved more cheaply with ECS Fargate + Step Functions + managed Bedrock Agents while the team is small. It's also the wrong design if an enterprise customer contractually requires physical single-tenant isolation (separate AWS account per customer) rather than the logical multi-tenancy assumed here — that's a different architecture, not a tuning of this one, and someone should say so explicitly rather than trying to bolt physical isolation onto a shared-cluster design after the fact.
