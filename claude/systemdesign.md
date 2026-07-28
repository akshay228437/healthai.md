# Platform Design: Agent Runtimes for Healthcare Conversations
### 2Care.ai Take-Home — Platform Engineer

---

## 0. Assumptions & questions I'd ask first

Since scale and architecture were left open, I designed against these assumptions. I'd confirm all of them in a real kickoff.

**Scale & shape**
- Tens to low-hundreds of enterprise healthcare customers (hospital systems, clinic networks), not one mega-tenant. Low hundreds of thousands of conversations/month today, growing ~3x/year.
- Both voice (inbound/outbound patient calls) and chat (web widget, SMS). Voice is latency-sensitive and dominates infra complexity; chat dominates raw volume.
- Conversations are multi-turn and can be long-running (minutes for voice, hours-spanning for async chat) — session state must survive longer than a single request/Lambda invocation.
- Outcomes are written back into clinical systems (EHR) — I assume via FHIR APIs (Epic/Cerner-style), not direct DB writes, and that at least early on, writes are staged for human review before landing, graduating to direct write per-customer as trust is established.

**Questions I'd actually ask the team**
1. Is a write-back to the EHR ever synchronous/blocking on the conversation, or always async/staged? Changes the failure-mode story a lot.
2. Do any customers contractually require a dedicated AWS account (not just a namespace) per tenant? This is the single biggest cost/complexity swing in the whole design.
3. Is there an existing model relationship (Bedrock vs. direct Anthropic/OpenAI API) or is that open too?
4. What's the current incident/on-call story — is there already a security/compliance function I'd be gating against, or am I building that from scratch?

I'm treating "no dedicated account required, Bedrock is fine, we're building the compliance gating from scratch" as the working assumption below.

---

## 1. The stack

### Agent layer

| Concern | Choice | Rejected | Why |
|---|---|---|---|
| Orchestration framework | **LangGraph** | Hand-rolled ReAct loop; CrewAI | Need an explicit state machine, not a free-form loop — clinical conversations require deterministic checkpoints (e.g., "before this branch, tenant-boundary check must have passed"). LangGraph gives first-class graph checkpointing (persist state externally, resume after a crash or deploy) and mature human-in-the-loop interrupts. CrewAI's multi-agent-crew model is overkill for a single-session patient conversation and its state-persistence story is weaker. |
| Voice transport | **Amazon Connect** (PSTN/SIP ingress) + audio streamed via Chime SDK media pipeline | Twilio | Staying inside AWS's compliance boundary matters more than Twilio's nicer DX — Connect's BAA and audit posture are simpler to reason about for an auditor than a second vendor relationship. |
| ASR | **Deepgram** (medical-tuned model) | Amazon Transcribe Medical | Transcribe Medical is the "safe" AWS-native choice, but Deepgram's streaming latency and accuracy on non-clinical-vocabulary conversational speech (which is most of what patients actually say) was materially better in practice at teams I've seen use both. Deepgram does have a BAA available — this is a real trade to revisit if that changes. |
| TTS | **Amazon Polly** | ElevenLabs | Polly stays inside the AWS boundary (no extra BAA), "good enough" naturalness for a clinical assistant, and per-request cost is far lower at volume than ElevenLabs. Not the highest-quality voice on the market — acceptable trade for cost + compliance simplicity. |
| Chat transport | **ECS Fargate service behind an ALB, WebSocket** | API Gateway WebSocket + Lambda | Chat sessions can be long-lived; Lambda's execution-time ceiling and cold starts are a bad fit for a persistent bidirectional connection holding conversation state in memory between turns. |
| Model routing | **Bedrock** (Claude Sonnet for generation, Claude Haiku for cheap sub-tasks: intent classification, PII/PHI pre-screen, EHR-field extraction) | Direct Anthropic API, OpenAI | Bedrock keeps patient conversation content inside the existing AWS BAA boundary — no second BAA to negotiate and audit. A lightweight router (a Haiku call) tags each turn's risk/complexity tier before deciding which model handles generation. |
| Session state | **DynamoDB** for hot checkpoints (partition key `tenant_id#session_id`), **Aurora Postgres** (row-level security) for durable conversation history/metadata | Redis for checkpoints | DynamoDB's single-digit-ms latency and native per-tenant partitioning fit the hot path; Redis would need us to build our own persistence/backup story for something PHI-adjacent — not worth it. |
| Eval | **Braintrust** (offline eval sets, regression gating in CI) + **self-hosted Langfuse** (in-VPC trace capture) | A hosted SaaS observability tool (e.g., hosted Langfuse/Helicone cloud) | Self-hosting the tracing layer inside our VPC is the whole point — we do not want raw patient transcripts leaving our account into a third-party SaaS without our own BAA and audit control over it. |

### Infrastructure layer

- **Compute:** EKS for the agent runtime (long-running, stateful, needs persistent bidirectional connections and fine-grained network policy per tenant). Lambda only for event-driven glue (Terraform CI checks, EHR write-back triggers, guardrail webhook fan-out) — **not** the runtime itself.
  - *Rejected:* ECS Fargate as primary compute. Fargate is operationally simpler, but at dozens of healthcare tenants I want Kubernetes-native `NetworkPolicy` + OPA Gatekeeper/Kyverno enforcement per-namespace, and Karpenter for fast, inference-aware node autoscaling if we self-host any ASR/TTS models later. Fargate's isolation primitives are thinner at this tenant count.
- **Orchestration:** Argo CD (GitOps) + **Argo Rollouts** for canary/blue-green agent releases. Deliberately separate from Terraform: Terraform owns infra (VPC, cluster, IAM, data stores); Argo/Helm own workload versions. An agent version bump is never a Terraform apply — this keeps the blast radius of an infra pipeline change separate from an agent code rollback, and rollback of a bad agent version happens in seconds via Argo Rollouts, not a `terraform apply`.
- **Networking:** Private subnets only for tenant workloads, VPC endpoints/PrivateLink for Bedrock/S3/DynamoDB (no public internet path for AWS-native calls), service mesh (mTLS) for pod-to-pod traffic, Transit Gateway for customers needing dedicated connectivity to their own EHR endpoint.
- **Data stores:** DynamoDB (session state), Aurora Postgres w/ RLS (history, metadata), S3 with **per-tenant KMS keys** for raw audio/transcripts (so a data-deletion request becomes crypto-shredding one key, not a hunt-and-delete job), OpenSearch/pgvector if RAG over clinical protocol docs is needed.
- **Terraform organization:** account-per-environment (dev/staging/prod) via AWS Organizations, remote state in S3 + DynamoDB lock (or Terraform Cloud for policy-as-code via Sentinel if budget allows). Modules: `network`, `eks-cluster`, `data-stores`, and a `tenant-onboarding` module invoked per new customer — provisioning namespace, IAM role (IRSA), KMS key, and network policy from a config file, so onboarding a customer is a PR, not a set of manual console clicks.
  - *Rejected:* separate AWS account per customer from day one. Best isolation, but operationally heavy at this scale — I'd reserve dedicated accounts for the largest/most sensitive customers who require it contractually, not as the default.

---

## 2. The runtime and its harness

- **Versioning:** agent code, prompts, and guardrail configs are versioned together in one monorepo, built into a semver-tagged container image via CI. Prompts are never hot-edited in a dashboard disconnected from code review — too much clinical risk in an unreviewed prompt change.
- **Release:** Argo Rollouts canary. A new version gets 1% of **new** sessions only (never injected into a session already in progress), monitored for a bake period against automated gates (error rate delta, guardrail-trigger-rate delta, eval-judge score on shadow traffic, latency p95) before progressing 1% → 10% → 50% → 100%.
- **Live sessions during a deploy or scale-in:** never kill a pod mid-turn. On SIGTERM, the pod stops accepting new sessions, finishes in-flight turns, checkpoints state to DynamoDB, then terminates — `terminationGracePeriodSeconds` tuned to the expected worst-case turn latency. A session in progress finishes on the version it started on unless a critical safety patch forces a hard cutover, which is a documented, human-approved exception path, not a default behavior.
- **Catching a bad version before it reaches customers:** the canary gates above, plus a mandatory offline eval-gate in CI (regression suite must pass before a canary can even open).
- **Undoing it after it's live:** Argo Rollouts aborts and reverts to last-known-good automatically if a gate fails. Because session state is externalized to DynamoDB rather than held in-pod, a session that was on the bad version can resume under the previous version's code without losing conversation context — this is the main reason state externalization was non-negotiable in the design.
- **Terraform gating for prod:** PR-based plan output (Atlantis or TF Cloud), Sentinel/OPA policy checks (no public S3, no `0.0.0.0/0` ingress to data stores, mandatory tenant/data-classification tags), and a second, security-tagged approver required for anything touching IAM or data stores.

---

## 3. Safety in a regulated environment

- **Tenant isolation:** namespace-per-tenant (account-per-tenant for the largest customers), enforced `NetworkPolicy`/Cilium so no pod-to-pod traffic crosses tenant namespaces except through an explicit shared model-gateway service; IAM scoped per tenant via IRSA (no shared over-broad role); per-tenant KMS key for data at rest.
- **What's allowed near sensitive data:** only the agent runtime pods and the EHR-writeback service can touch PHI tables directly. Everything else — analytics, the observability pipeline — gets de-identified/tokenized data. A redaction step (built on AWS Comprehend Medical, with a regex/allowlist fallback) strips PHI before anything reaches Langfuse or CloudWatch.
- **Proving it to an auditor:** infra is entirely Terraform — git history and plan/apply logs are the change log. AWS Config continuously checks for drift; each HIPAA Security Rule requirement is mapped to a specific Terraform-enforced control plus a CloudTrail/Config evidence artifact, aggregated in Security Hub.
- **AWS controls leaned on:** IAM/IRSA (least privilege), per-tenant KMS CMKs, VPC + PrivateLink (PHI never crosses the public internet), GuardDuty, Macie (specifically to continuously scan S3 for PHI landing in the wrong bucket), CloudTrail with Object Lock (immutability), Config, Security Hub.
- **Where AWS controls aren't enough alone:** AWS's IAM boundary won't tell you if a *prompt* leaked one patient's context into another patient's window, or if a RAG retrieval crossed a tenant boundary at the application layer — that's not an infra failure, it's an app-level guardrail gap. So we add a **context firewall**: every retrieval/tool call is tagged with `tenant_id`, and a runtime assertion — not just IAM — checks that every document/tool result entering the prompt matches the session's tenant before it's allowed into context. Also: the AWS BAA covers AWS services, not Deepgram or any third party in the path — anything touching raw PHI needs its own BAA or must never see raw content, which is part of why Bedrock (in-boundary) is preferred over an external model API for anything handling live conversation content.

---

## 4. Knowing it works, after it ships

Infra metrics (latency, error rate, pod restarts) catch "broken." They don't catch "wrong." The signals that actually distinguish healthy / degrading / silently failing:

- **Guardrail trigger rate over time** — a rising trend is drift, even if nothing else moved.
- **Eval-judge score trend** — a rolling sample of de-identified live transcripts scored asynchronously (Claude-as-judge against a clinical rubric), not just the offline golden set.
- **Task completion rate** — did the conversation reach a resolved state, or get abandoned/escalated to a human?
- **The hardest case — a confident, wrong answer that trips no guardrail** — mitigated by human clinical review of a small, *risk-weighted* daily transcript sample, plus reconciling what actually landed in the EHR against expected structured fields.

**Tooling:** OpenTelemetry traces end-to-end (ASR → router → LLM → tool calls → TTS/EHR write), self-hosted Langfuse for the LLM-specific view, Grafana/Prometheus/CloudWatch for infra, PagerDuty for alerting.

**What pages a human at 2am, no threshold needed:** a single PHI/tenant-boundary guardrail trip; any guardrail categorized "clinical safety" (e.g., agent giving dosage/medical guidance outside its allowed scope) — that one pages on the *first* occurrence, not after a rate threshold; a canary abort mid-deploy; EHR write-validation failures above a small threshold.

---

## 5. Cost

**What dominates at scale:** for voice-heavy customers, ASR/TTS per-minute charges and telephony minutes often outweigh raw LLM token cost. For chat-heavy customers, it's model tokens — especially if full conversation history is re-sent every turn. Compute (over-provisioned EKS nodes for peak) and data egress are secondary but real.

**Controlling it without degrading experience:**
- Summarize older turns instead of re-sending full history each call (cuts tokens; validate no quality loss via the eval harness before shipping).
- Route cheap sub-tasks (intent classification, urgency triage) to Haiku, reserve Sonnet for generation that actually needs it.
- Bedrock prompt caching for repeated system prompts/tool schemas — meaningful savings at high volume of repeated structure.
- Karpenter-driven right-sizing, scaling low-traffic tenant namespaces toward zero off-hours where contractually acceptable.
- Reserved/Savings Plans for baseline compute, spot only for shadow-eval/batch work — never spot for live customer-facing pods.
- Committed-use pricing with Bedrock/Deepgram once volume is established.

---

## 6. Making the team faster

Candidates for automation, roughly in order of toil removed:
1. **Tenant onboarding** — Terraform module + CI pipeline standing up namespace/IAM/KMS/network policy from a config file instead of manual applies.
2. **Canary promotion** — auto-progressing a rollout through gated stages instead of a human clicking "next."
3. **Guardrail-trigger triage** — auto-classify and route triggers to the right on-call/clinical reviewer instead of a human reading every one.
4. **EHR write-back validation** — deterministic schema/field checks before anything reaches the clinical system.

**The one I'd ship first: canary promotion.** It's the automation with the richest guardrail story — it touches production, live patient sessions, and the sensitive-data path — so getting its autonomy boundary right first sets the pattern for everything after it. (Full detail on triggers/guardrails in the "Working with AI" section below, per the prompt's request.)

---

## 7. Getting better after go-live

Week 1 → week 12 improvement loop: de-identified transcripts, guardrail triggers, and customer-reported issues feed a **per-tenant improvement backlog**. Flagged transcripts, after clinical-reviewer sign-off, join that tenant's golden eval set — sign-off matters so the eval set doesn't just inherit whatever bias the current agent already has.

I'd default to **shared-base-agent + per-tenant prompt/config/retrieval-corpus customization**, not per-customer fine-tuning. Fine-tuning per customer is expensive, and worse, it multiplies the number of model artifacts we have to keep an eval/version/rollback discipline over — it breaks the "one bad version, one rollback" story from section 2.

Shipping a per-tenant change uses the same canary machinery as section 2, but scoped: a config change for Customer A canaries only against Customer A's traffic and golden set, never touching Customer B. This is why tenant-level config has to be a first-class versioned artifact alongside the shared agent version, not a side file.

---

## Where this design is wrong

This assumes a multi-tenant SaaS shape — tens to low-hundreds of customers sharing infrastructure with namespace-level isolation. If the actual shape is closer to one or two enormous customers with a contractual or regulatory requirement for full account-level physical isolation per hospital system — not just namespace/IAM boundaries — then the shared-cluster, namespace-per-tenant model is the wrong call from the start, and it should be overruled in favor of account-per-tenant from day one, even though that costs more operationally. The tell would be a customer who won't sign off on shared compute at any isolation level short of a dedicated account.

---

## Working with AI

**(a) How I used AI on this**

*[This section needs your own honest account — I drafted the architecture with you, but "where it got things wrong and how you caught it" has to be your real experience, not mine. Fill in specifics like: which tradeoff Claude initially got wrong or gave a generic answer for, what made you push back, what you changed. A thin or fabricated answer here is exactly what they say they're grading against — the prompt is explicit that pretending you didn't use AI (or implicitly, papering over the real experience) is the only thing that hurts you.]*

**(b) One level deeper: canary promotion automation**

- **Triggers:** a new agent version image passes CI (build + offline eval-gate against the golden set above threshold) → the automation opens a canary at 1% of new sessions automatically.
- **What it may do autonomously:** progress 1% → 10% → 25% → 50% *only if all of* — error-rate delta, guardrail-trigger-rate delta, eval-judge rolling score, and latency p95 stay within pre-set bounds, evaluated over a minimum session-count floor per stage. Critically, it only ever reads **pre-aggregated, de-identified counters** — never raw transcript content — for its own decisions.
- **The guardrail that keeps it from doing something unsafe:** two things. First, structurally: because its inputs are aggregate counters, not raw PHI, there's no path for the automation itself to leak sensitive data even if manipulated. Second, a hard circuit breaker — a single "clinical safety" category guardrail trip during canary halts and reverts immediately, regardless of how good the aggregate stats look; one trip is never averaged away. The final stage, 50% → 100%, always requires a human sign-off no matter how clean the metrics are — that's the point where a subtle failure would hit the full population.
- **How you'd know it started making things worse:** every auto-promotion decision is logged against the metrics it acted on (fully auditable), alerts fire if it stalls or reverts, and a weekly review compares its decisions against a shadow "would a human have promoted here" log — to catch it drifting too aggressive or too conservative over time.
