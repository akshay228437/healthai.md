# Platform Design: 2Care.ai Agent Runtime & Infrastructure

---

## 0. Framing & Initial Assumptions

### Clarifying Questions I’d Ask Before Building

If this were an internal kickoff with product & clinical ops, I would ask:

1. **PSTN / Telephony Provider:** Are we using **Twilio**, **SignalWire**, or **Amazon Chime SDK / PSTN Audio**? (Assumed: **Twilio** via WebSocket media streams).
2. **Clinical Integration Interfaces:** Are we integrating directly via **FHIR/HL7 (e.g., Epic/Cerner)**, or calling an internal backend API that handles EHR mutation? (Assumed: Microservice layer with audit logging sitting between agent runtime and EHR).
3. **Model Hosting:** Do we rely purely on managed APIs (**Bedrock / Anthropic / OpenAI**), or do we host custom fine-tuned models on **SageMaker** or **vLLM/EKS**? (Assumed: Primary managed LLMs with Bedrock/Anthropic + fallbacks; optional open-weights for specialized NLP).

### Scale & Architecture Assumptions

* **Concurrency:** ~1,500 peak simultaneous voice sessions (~500 hrs/day voice) + ~5,000 active chat sessions.
* **Latency SLO:** Voice round-trip time (TTFT/speech-to-speech) $< 1.2$ seconds total latency end-to-end to maintain conversational flow.
* **Tenancy:** Multi-tenant infrastructure with strict logical isolation (cell/namespace per tenant) for healthcare organizations (HIPAA BAA compliance required across all services).

---

## 1. The Stack

### Agent Layer

* **Agent Framework:** **LiveKit Agents** (or custom **Python/asyncio** harness over **LangChain/LlamaIndex**).
* *Why:* LiveKit handles real-time WebRTC/PSTN media streaming, VAD (Voice Activity Detection), and turn-taking out-of-the-box with low-latency C++ bindings, avoiding Python GIL bottlenecks during audio frame processing.


* **Transport & Media Handling:** **LiveKit Server (WebRTC)** for voice streaming; **WebSocket / HTTP/2** via **AWS API Gateway** or **Envoy** for chat.
* **Model Routing & Gateway:** **LiteLLM Proxy** or **AWS Bedrock Guardrails**.
* *Why:* Provides standard OpenAI-compatible interfaces, unified rate limiting, cost tracking, dynamic fallback (e.g., Anthropic Claude 3.5 Sonnet $\rightarrow$ Bedrock Claude 3.5 Sonnet $\rightarrow$ OpenAI GPT-4o), and automatic PHI sanitization before requests hit upstream APIs.


* **Session State:** **Redis Cluster (AWS ElastiCache for Redis)** with in-memory persistence for active state, flushed asynchronously to **Amazon DynamoDB** (encrypted with customer-managed KMS keys) upon session termination.
* **Evaluation Framework:** **DeepEval / Ragas** integrated with **Langfuse** (self-hosted on EKS) for tracing, evaluation scoring, and LLM-as-a-judge quality monitoring.

```
       [ Telephony / Web client ]
                   │
                   ▼
       [ AWS Network Load Balancer ]
                   │
       ┌───────────┴───────────┐
       ▼                       ▼
 [ LiveKit Server ]     [ API Gateway (Chat) ]
       │                       │
       └───────────┬───────────┘
                   ▼
         [ Agent Runtime (EKS) ]
                   │
        ┌──────────┴──────────┐
        ▼                     ▼
 [ LiteLLM Gateway ]    [ Clinical API ]
        │                     │
  (Bedrock/OpenAI)       (DynamoDB/EHR)

```

### Infrastructure Layer

* **Compute & Orchestration:** **Amazon EKS (Kubernetes 1.30+)** with **Karpenter** for dynamic auto-scaling.
* *Why EKS over ECS/Lambda:* Voice runtime requires persistent, stateful, low-latency WebSockets with bidirectional streaming. Lambda execution limits and cold starts render it unsuitable for real-time turn-taking voice; ECS lacks Karpenter's granular node-level bin-packing and fast scale-out.


* **Networking:** **AWS VPC** with isolated private subnets, **AWS Transit Gateway**, **VPC Endpoints (AWS PrivateLink)** for all AWS services (DynamoDB, S3, KMS) to prevent traffic from touching the public internet.
* **Data Stores:**
* **AWS ElastiCache Redis** (Active session state & turn cache).
* **Amazon DynamoDB** (Conversation metadata, audit logs, transcript index).
* **Amazon S3** (Encrypted raw audio recordings & transcript archives, lifecycle policy to Glacier).


* **Terraform Organization:**
* Directory layout structured around **Terragrunt** or modular Terraform roots separated by account and environment:
* `infrastructure/modules/` (Re-usable modules: `eks-cluster`, `vpc`, `tenant-isolation`).
* `infrastructure/live/`
* `management/` (CI/CD, ECR, Datadog/Langfuse observability).
* `dev/`
* `prod-us-east-1/`
* `prod-us-west-2/`




* State files strictly locked via **DynamoDB** and stored in **S3** with Object Locking and KMS encryption.



### Critical Tech Choices & Rejections

1. **EKS + Karpenter vs. AWS Fargate / ECS:** *Rejected Fargate/ECS.* Voice audio processing requires deterministic CPU allocation for WebRTC encoding/decoding and UDP socket buffers. Karpenter allows pinning audio agents to dedicated EC2 compute (`c6i` instances) with host-path network performance.
2. **LiveKit vs. Custom Twilio Media Streams to Python:** *Rejected purely custom WebSocket handlers.* Building custom Jitter buffers, VAD, and turn-taking state machines in raw Python introduces audio artifacts and high tail latency. LiveKit offloads real-time transport handling.
3. **LiteLLM Gateway vs. Direct SDK calls:** *Rejected direct SDK imports.* Direct SDKs lack centralized cost controls, cross-provider automatic retries, unified rate-limiting, and standard audit logging across multiple teams.

---

## 2. The Runtime and its Harness

```
   [ Git Push / PR ] ──> [ CI: Test & Lint ] ──> [ Build OCI Image ]
                                                         │
                                                         ▼
   [ Progressive Rollout ] <── [ Argo CD Deployment ] <── [ Pre-deploy Gate: Eval Harness ]
             │
             ├─> 10% Canary (5-min voice/chat session monitor)
             └─> 100% Prod (Auto-rollback on error/latency spike)

```

### Build, Version, and Release Cycle

* **Artifacts:** Agent runtimes packaged as OCI-compliant container images built via **GitHub Actions**, tagged with Git SHA and semantic versioning (`v1.4.2-sha-8f3a92`).
* **Deployment System:** **Argo CD** implementing GitOps patterns. Infrastructure and app configurations live declaratively in Git repositories.

### Zero-Downtime Session Management

* **Active Voice/Chat Session Handling:** Agent containers use **Graceful Termination Signals (`SIGTERM`)**.
* When a deployment or scale-in occurs:
1. Kubernetes marks the pod as `Terminating` and removes it from the Service Endpoints / Load Balancer.
2. The agent process catches `SIGTERM` and stops accepting *new* incoming calls or chat connections.
3. The active session is allowed to finish natively (up to a configurable `terminationGracePeriodSeconds` of 900 seconds).
4. Session state is continuously checkpointed to Redis after every turn, ensuring that if a node crashes unexpectedly, another agent pod can resume the conversation state using the `session_id`.





### Release Guardrails & Automated Rollback

* **Pre-deployment Evals (CI/CD Gate):** Before deploying to staging/prod, an automated eval pipeline runs **DeepEval** against 100 test scenarios (clinical accuracy, safety/guardrail evasion, edge cases). If overall accuracy drops below 98%, the pipeline halts.
* **Progressive Rollouts (Argo Rollouts):**
* Deployments use a **Canary** strategy: 10% traffic for 10 minutes.
* **Automated Rollback Criteria:** Argo Rollouts monitors Datadog / Prometheus metrics during the canary phase:
1. HTTP/WebSocket 5xx error rate $> 0.5\%$.
2. Latency p95 $> 1,500\text{ ms}$.
3. Guardrail/Fallback triggers increase by $> 2\%$.


* If any threshold breaches, Argo CD instantly reverts the Kubernetes Deployment to the prior Git SHA without human intervention.



### Terraform Production Gates

* All IaC changes require a PR with mandatory `terraform plan` output generated automatically via **Atlantis** or **GitHub Actions**.
* Mandatory review approvals from 2 Senior Engineers/SecOps.
* Security scanners (**Checkov** and **tfsec**) run on PRs to catch exposed S3 buckets, permissive Security Groups, or missing encryption configurations before merging.

---

## 3. Safety in a Regulated Environment (HIPAA & Healthcare Compliance)

### Multi-Tenant Isolation

* **Compute:** Kubernetes **Namespaces** with strict **NetworkPolicies** prohibiting cross-tenant pod communication. High-compliance tenants receive dedicated node groups using Kubernetes `nodeSelector` / `tolerations`.
* **Data:** DynamoDB tables and S3 buckets use tenant-prefixed partition keys with AWS IAM Condition Keys (`aws:PrincipalTag/TenantId`) restricting access at the storage level.

```
       ┌─────────────────────────────────────────────────────────┐
       │                   AWS VPC (Private)                     │
       │                                                         │
       │   Tenant A Namespace               Tenant B Namespace   │
       │   ┌───────────────────┐            ┌──────────────────┐ │
       │   │  Agent Pods       │            │  Agent Pods      │ │
       │   └─────────┬─────────┘            └────────┬─────────┘ │
       │             │ NetworkPolicy Isolation       │           │
       └─────────────┼───────────────────────────────┼───────────┘
                     │                               │
                     ▼                               ▼
       ┌────────────────────────┐      ┌────────────────────────┐
       │ KMS Key A (Tenant A)   │      │ KMS Key B (Tenant B)   │
       └────────────────────────┘      └────────────────────────┘

```

### Data Security & Sensitive Data Handling (PHI/PII)

* **Encryption:**
* **In-Transit:** TLS 1.3 enforced for all internal/external HTTP/WebSocket traffic; mTLS between microservices via **Linkerd / Istio** service mesh.
* **At-Rest:** AES-256 via AWS KMS with Customer Managed Keys (CMKs) rotated annually. Separate KMS keys per tenant.


* **PHI Redaction Layer:** The LiteLLM / Agent harness runs an inline redaction pipe (**Microsoft Presidio** or **Amazon Comprehend Medical**) *before* passing raw text to external/third-party LLM APIs. Audio recordings on S3 are encrypted at rest with strict IAM lifecycle expiration policies.

### Auditability & Controls

* **AWS Native Controls:**
* **AWS CloudTrail:** Captures all API interactions across AWS accounts.
* **AWS Config:** Continuously monitors resource configurations for compliance drift (e.g., unencrypted S3 buckets, public security groups).
* **AWS GuardDuty:** Threat detection for EKS audit logs and VPC flow logs.


* **Where AWS Controls Fall Short (And Custom Solutions Required):**
* *AWS CloudTrail doesn't audit LLM prompts/responses or internal conversational decisions.*
* **Solution:** Implement a dedicated **Immutable Clinical Audit Log** microservice. Every agent decision, tool execution (e.g., write to EHR), model input/output (redacted PHI), and confidence score is signed and written to **Amazon QLDB** (Quantum Ledger Database) or DynamoDB with `DeletionProtectionEnabled` and strict IAM write-only access.



---

## 4. Observability: Knowing It Works

```
   Level 1: Infrastructure        Level 2: Conversational          Level 3: Clinical / Hidden
   ┌───────────────────────┐      ┌────────────────────────┐       ┌───────────────────────┐
   │ • CPU / Memory        │      │ • Turn-taking latency   │       │ • Repetitive loop %   │
   │ • Pod restarts        │      │ • STT / TTS latency    │       │ • Silent fallback rate│
   │ • HTTP / WS 5xx rates │      │ • Interruption counts  │       │ • EHR mutation failure│
   └───────────────────────┘      └────────────────────────┘       └───────────────────────┘

```

### Measuring System & Agent Health

To catch degrading or quietly failing agents, observability must span three distinct levels:

| Level | Focus Area | Metrics Measured | Tooling |
| --- | --- | --- | --- |
| **1. Infrastructure** | Compute & Network | Pod CPU/RAM, WebRTC packet loss, WebSocket disconnect rates, API 5xx errors. | Datadog / Prometheus + Grafana |
| **2. Conversational** | Agent Responsiveness | Speech-to-Text (STT) latency, Time to First Token (TTFT), Text-to-Speech (TTS) synth time, barge-in/interruption counts. | OpenTelemetry + Langfuse |
| **3. Clinical / Quality** | Task Completion & Safety | Semantic drift, repetitive loops, silent fallbacks to human operator, tool call success/failure rate, hallucination score. | Langfuse + Custom Async Eval Workers |

### Identifying Quiet/Hidden Failures

An agent might be "healthy" on HTTP status codes while frustrating patients or outputting inaccurate information.

* **Repetition Detection:** Track unique n-gram similarity over $N$ turns. If an agent repeats the same phrase $> 2$ times, trigger an automatic escalation.
* **Sentiment / Frustration Signals:** Analyze STT acoustic features or run lightweight real-time sentiment scoring on patient inputs. A spike in user interruptions indicates degrading interaction quality.
* **Tool Call Mismatch:** Monitor the ratio of conversation turns to successful EHR mutations. If an agent completes a 5-minute call without invoking expected clinical tool calls, flag for audit.

### 2 AM PagerDuty Trigger Criteria

* **Immediate Page:**
* System-wide WebSockets / voice media server crash ($> 1\%$ connection failures).
* EHR write failure rate $> 2\%$ over a 5-minute window.
* Spike in LLM Guardrail violations ($> 5\%$ of active sessions within 10 minutes).
* Security/Isolation alerts (unauthorized KMS access or cross-tenant API attempt).


* **Non-paging Alert (Slack/Email):** Gradual drift in semantic quality scores, individual non-critical fallback triggers, minor latency rises below SLO thresholds.

---

## 5. Cost Strategy

### Cost Drivers at Scale

At scale, cost distribution typically breaks down as:

1. **Model API Fees (LLMs, STT, TTS):** $\sim 60\text{--}70\%$ of total bill.
2. **Telephony & Media Transport (Twilio + WebSockets):** $\sim 15\text{--}20\%$.
3. **AWS Compute & Data Stores (EKS, ElastiCache, DynamoDB):** $\sim 10\text{--}15\%$.

```
                       ┌──────────────────────────────┐
                       │  Total Operating Costs       │
                       └──────────────┬───────────────┘
                                      │
         ┌────────────────────────────┼────────────────────────────┐
         ▼                            ▼                            ▼
  [ Model APIs: 60-70% ]      [ Telephony: 15-20% ]      [ AWS Compute: 10-15% ]
  • LLM Tokens                • Twilio / PSTN            • EKS Nodes (Karpenter)
  • Deepgram STT / ElevenLabs • Bandwidth / Media         • ElastiCache / DynamoDB

```

### Cost Optimization Tactics

* **Model Tiering & Routing:**
* Route simple intent extraction, greeting, and conversational filler to lighter, cheaper models (e.g., **Claude 3.5 Haiku** or **GPT-4o-mini**).
* Reserve primary reasoning / clinical summaries for **Claude 3.5 Sonnet**.
* Use local cached embeddings (e.g., **FastEmbed**) for RAG retrievals rather than remote API calls.


* **STT / TTS Optimizations:**
* Run open-source STT (**Whisper v3** on GPU compute) or high-efficiency providers (**Deepgram Nova-2**) over expensive proprietary pipelines.
* Implement active VAD (Voice Activity Detection) at the edge/client side to stop streaming audio tokens to STT during silence.


* **Infrastructure Optimization:**
* Utilize **AWS Karpenter** to leverage **Spot Instances** for stateless worker pods (evals, non-real-time audio processing) and Graviton (`a1`/`c7g`) instances for EKS nodes, reducing compute spend by $30\text{--}40\%$.



---

## 6. Making the Team Faster: Automation Priorities

### Internal Workflows to Automate

1. **Automated Clinical Golden-Dataset Evals on Pull Requests.**
2. **One-Click Tenant Provisioning (Terraform + Helm pipeline).**
3. **Automated Incident Context Aggregator (Logs, Langfuse Traces, EHR Tool Logs).**

### Priority Automation: Synthetic Patient Testing Pipeline

#### Trigger

* Triggered automatically whenever a PR modifies `agent/prompts/`, `agent/tools/`, or core agent dependencies.

#### Autonomous Actions Allowed

* Spins up a ephemeral EKS namespace.
* Executes 50+ concurrent synthetic patient interactions using simulated LLM personas (e.g., confused patient, angry patient, non-English speaker).
* Computes performance metrics (Task completion %, PHI leak check, safety compliance, latency).
* Posts a detailed evaluation report directly to the PR.

#### Guardrails Against Sensitive Data Exposure

* Pipeline uses **100% synthetic, dummy patient data** generated specifically for testing.
* Pre-push git hooks and automated scanners (**TruffleHog / GitGuardian**) verify no real PHI/PII or production credentials exist in the branch.

#### Detecting Degradation/Regressions

* The PR pipeline compares test scores against the baseline branch. If task completion score drops by $> 1\%$ or guardrail compliance drops below $100\%$, the CI/CD pipeline **fails automatically** and blocks merging.

---

## 7. Continuous Improvement (Week 1 to Week 12)

```
   [ Week 1 Live Session ] ──> [ Langfuse Tracing / Logs ]
                                         │
                                         ▼
   [ CI PR / Automated Eval ] <── [ Prompt/RAG Tuning ] <── [ Anonymized Low-Score Curations ]
             │
             ▼
   [ Argo CD Canary Deploy ] ──> [ Week 12 Improved Agent ]

```

### Feedback Loop Architecture

1. **Data Ingestion:** Every live call log, transcript, and trace is stored in Langfuse/S3. Low-scoring or user-escalated sessions are flagged automatically.
2. **Anonymization & Curation:** Flagged transcripts pass through an offline anonymization pipeline (removing PHI/PII) and are converted into structured test cases added to the "Golden Evaluation Dataset."
3. **Prompt & Knowledge Tuning:** Prompt engineers and clinical teams update RAG vector stores or adjust prompt directives in Git to address newly discovered edge cases.
4. **CI/CD Validation:** The updated agent runtimes are tested against the expanded Golden Dataset via automated evals before deployment.

### Preventing Regressions

* **Regression Test Suite:** The eval suite grows monotonically. A bug discovered in Week 2 becomes a permanent test case in Week 3.
* **Shadow Deployments:** Before routing live traffic to a new agent version, shadow traffic (mirroring real user inputs to the candidate agent without returning its output to the patient) is evaluated offline to verify performance against live distributions.

---

## 8. When Is This Design Wrong?

**This design is the wrong one if 2Care.ai shifts from real-time interactive voice agents to asynchronous, batch-oriented clinical operations (or pure low-concurrency chat widgets).**

If our core workload consists of generating post-visit clinical summaries from pre-recorded audio or handling slow, asynchronous SMS/email communication, the operational overhead of EKS, WebRTC (LiveKit), Karpenter, and persistent WebSockets is unnecessary over-engineering. In that scenario, a fully serverless, event-driven architecture using **AWS Lambda**, **Amazon Bedrock**, **AWS Step Functions**, and **DynamoDB** would be dramatically simpler to maintain, significantly cheaper at lower or unpredictable volumes, and require far less platform management.

---

## 9. Working with AI

### (a) AI Tool Usage & Reflection

* **How AI Was Used:** Used LLM assistants (Claude 3.5 Sonnet & Cursor) to rapidly draft the infrastructure topology, structure the Markdown document, generate Terragrunt directory trees, and refine LaTeX formulas/tables for readability.
* **Where It Got Things Wrong & How It Was Caught:**
* *Wrong Suggestion:* The AI initially suggested using **AWS Lambda + API Gateway WebSockets** for handling real-time voice streaming with LiveKit.
* *Correction:* I caught this because AWS Lambda has a 15-minute execution limit, cold-start latency spikes that disrupt real-time WebRTC audio frames, and lacks native bidirectional UDP streaming support required for WebRTC. I corrected the proposal to run **EKS with dedicated node groups managed by Karpenter**.



### (b) Deep Dive: Synthetic Patient Testing Automation

```
  [ GitHub Pull Request ] ──> [ GitHub Actions Runner ]
                                    │
                                    ▼
                         [ Ephemeral EKS Namespace ]
                                    │
                  ┌─────────────────┴─────────────────┐
                  ▼                                   ▼
          [ Candidate Agent ] <───────────> [ Synthetic Patient LLM ]
                  │                                   │
                  └─────────────────┬─────────────────┘
                                    ▼
                         [ Langfuse & DeepEval ]
                                    │
                                    ▼
                         [ PR Status Check / Gate ]

```

* **Trigger:** A pull request pushed to `main` containing changes to `src/agent/`, `prompts/`, or `config/`.
* **Autonomous Actions:**
1. Spins up an ephemeral Kubernetes namespace (`test-pr-123`) on EKS.
2. Deploys the PR container build along with a **Synthetic Patient Simulator** (an LLM agent prompted with patient profiles, symptoms, and communication styles).
3. Executes 50 test scenarios concurrently over WebSockets.
4. Collects performance metrics, posts a benchmark comparison table to the PR, and tears down the ephemeral namespace.


* **PHI Guardrail:** The testing framework is restricted to reading mock datasets stored in `tests/fixtures/synthetic_patients.json`. It has **no network routes (VPC NetworkPolicies)** to production DynamoDB databases or production KMS keys.
* **Detecting Failure/Degradation:**
* If the automated eval returns a safety compliance score $< 100\%$ or task completion score lower than the current `main` branch, the GitHub Action status check turns **RED**, blocking the "Merge" button automatically.
