A good way to approach this assignment is to design **from the inside out**. The core of the platform is the **AI Agent Runtime**, because everything else (deployment, infrastructure, security, monitoring, cost) exists to support it.

---

# Step 1 — Design the Agent Layer

## Objective

Design a reusable runtime capable of serving thousands of concurrent **voice** and **chat** healthcare conversations while remaining secure, observable, and easy to update.

---

# Functional Requirements

The agent should be able to:

* Handle voice and chat sessions
* Maintain conversation state
* Call clinical APIs/EHR systems
* Use multiple LLMs
* Execute tools/functions
* Produce structured outputs
* Support streaming responses
* Log safely (without exposing PHI)
* Be versioned independently from infrastructure

---

# High-Level Architecture

```
                Customer

      Voice                  Chat
        │                     │
        ▼                     ▼
Telephony/WebRTC         WebSocket/API
        │                     │
        └────────────┬────────┘
                     ▼
              API Gateway / ALB
                     │
                     ▼
            Agent Runtime Service
                     │
      ┌──────────────┼───────────────┐
      ▼              ▼               ▼
 Session Store   Model Router    Tool Executor
      │              │               │
      ▼              ▼               ▼
 DynamoDB      OpenAI/Claude     EHR APIs
                     │
                     ▼
              Response Streaming
```

---

# What Framework Would I Choose?

## LangGraph

I would build the runtime using **LangGraph**.

### Why?

Healthcare conversations are long-running.

Unlike a chatbot, patient conversations involve:

* authentication
* collecting symptoms
* insurance lookup
* appointment booking
* medication verification
* follow-up questions

This is naturally a **state machine**, not a simple prompt-response loop.

LangGraph provides:

* explicit graph execution
* checkpoints
* retries
* branching logic
* human-in-the-loop support
* durable execution

This fits healthcare workflows extremely well.

---

## Why not LangChain?

LangChain is excellent for experimentation.

However,

production healthcare systems require

* deterministic execution
* explicit workflow transitions
* recoverability

LangGraph is purpose-built for production agent orchestration.

---

## Why not AutoGen?

AutoGen shines for

* multi-agent collaboration
* research
* autonomous planning

But healthcare agents generally require

* predictable execution
* compliance
* limited autonomy

AutoGen introduces unnecessary complexity.

---

# Transport Layer

Because the assignment includes **voice and chat**, transport matters.

## Chat

Use

```
WebSocket
```

Why?

* bidirectional
* low latency
* streaming responses
* typing indicators
* session continuity

REST is insufficient because responses stream token-by-token.

---

## Voice

Voice stack:

```
SIP/WebRTC

↓

Amazon Chime SDK

↓

Streaming Audio

↓

Speech-to-Text

↓

LLM

↓

Text-to-Speech
```

Amazon Chime SDK integrates well with AWS and supports low-latency audio streams.

---

# Speech Services

## Speech-to-Text

I would choose

Amazon Transcribe Medical

Why?

* HIPAA eligible
* medical vocabulary
* speaker separation
* AWS native
* compliant

Alternative:

Deepgram

Pros:

* excellent latency

Cons:

* another vendor
* PHI leaves AWS boundary

Since healthcare prioritizes compliance over small latency gains, Transcribe Medical is a better fit.

---

## Text-to-Speech

Amazon Polly

Why?

* AWS native
* secure
* scalable
* inexpensive

Alternative

ElevenLabs

Pros

* more natural voices

Cons

* external vendor
* additional compliance reviews

---

# Model Routing

Never hardcode one model.

Create a Model Router.

```
            Request

                │
                ▼

         Model Router

      ┌────────┼────────┐

 GPT-4.1   Claude 4   Llama
```

Routing depends on

* customer
* task
* latency
* cost
* fallback availability

---

## Why?

Some tasks need

reasoning

Others need

cheap summarization

Others need

tool calling

Routing avoids vendor lock-in.

---

# Session State

Session memory should NOT live inside containers.

Instead:

Short-term memory

→ DynamoDB

Conversation history

→ S3

Redis

→ active session cache

Why?

Containers can restart anytime.

External state allows seamless recovery.

---

# Tool Calling

The LLM should never directly access databases.

Instead

```
LLM

↓

Tool Executor

↓

Appointment API

↓

Insurance API

↓

Medication API

↓

FHIR Server
```

Benefits

* permission control
* auditing
* retries
* validation
* timeout handling

---

# Structured Outputs

Every conversation should end with JSON like

```json
{
  "patient_id":"123",
  "intent":"appointment_booking",
  "appointment_date":"2026-08-15",
  "confidence":0.97
}
```

Why?

Clinical systems require structured data—not free-form text.

---

# Evaluation Framework

Offline

* regression datasets
* prompt testing
* conversation replay

Online

* latency
* hallucination rate
* task completion
* escalation rate
* user satisfaction

I would use:

* LangSmith for tracing and evaluation
* OpenTelemetry for distributed tracing
* MLflow (optional) for experiment tracking if prompts/models evolve significantly

---

# Versioning

Each agent should have:

```
Agent Runtime v2.3.1

Prompt Pack v15

Workflow Graph v9

Tool Set v4

Model Config v12
```

These versions are independent.

This allows:

* updating prompts without rebuilding containers
* rolling back workflow logic
* A/B testing different prompt versions
* customer-specific configurations

---

# Why This Stack?

| Component       | Choice                    | Rejected              | Reason                                                      |
| --------------- | ------------------------- | --------------------- | ----------------------------------------------------------- |
| Agent Framework | LangGraph                 | LangChain             | Better stateful orchestration                               |
| Chat Transport  | WebSocket                 | REST                  | Supports streaming and real-time interaction                |
| Voice           | Amazon Chime SDK          | Twilio                | Native AWS integration and simplified compliance            |
| STT             | Amazon Transcribe Medical | Deepgram              | HIPAA-eligible and medical vocabulary                       |
| TTS             | Amazon Polly              | ElevenLabs            | Keeps sensitive audio within AWS                            |
| State Store     | DynamoDB + Redis + S3     | PostgreSQL            | Better scalability for session state and transcript storage |
| Evaluation      | LangSmith + OpenTelemetry | Custom solution       | Faster implementation with strong tracing support           |
| Model Routing   | Custom Router             | Single-model approach | Avoids vendor lock-in and optimizes cost/latency            |

---

This establishes the **agent runtime**, which is the foundation for the remaining sections. The natural next step is **Step 2: Infrastructure Layer (AWS architecture, compute, networking, data stores, orchestration, and Terraform organization)**, where this runtime is deployed and operated at scale.
