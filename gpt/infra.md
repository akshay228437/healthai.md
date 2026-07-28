# Step 2 — Infrastructure Layer (AWS + Terraform)

Now that we've designed the **Agent Runtime**, the next step is designing the infrastructure that runs it reliably, securely, and at scale.

Since the assignment explicitly states **AWS** and **Terraform** are fixed, the focus is on selecting the right AWS services and explaining the tradeoffs.

---

# Design Goals

The infrastructure should:

* Scale to thousands of concurrent voice/chat sessions.
* Handle zero-downtime deployments.
* Isolate healthcare customers (tenants).
* Recover automatically from failures.
* Be fully reproducible using Terraform.
* Meet HIPAA/SOC2-style compliance requirements.
* Support rapid feature delivery.

---

# High-Level AWS Architecture

```text
                        Internet
                            │
                  Route53 + AWS WAF
                            │
                    Application Load Balancer
                            │
        ┌───────────────────┴───────────────────┐
        │                                       │
        ▼                                       ▼
   Chat API Service                      Voice Gateway
        │                                       │
        └───────────────┬───────────────────────┘
                        ▼
               Agent Runtime (ECS Fargate)
                        │
        ┌───────────────┼───────────────────────┐
        ▼               ▼                       ▼
    DynamoDB         ElastiCache            S3
(Session State)       (Redis)         (Transcripts/Logs)
        │               │                       │
        └───────────────┼───────────────────────┘
                        ▼
                Amazon Bedrock / LLM APIs
                        │
                        ▼
                 External Healthcare APIs
```

---

# Compute Choice

## Chosen: Amazon ECS on Fargate

Each agent runtime runs inside a Docker container on **Amazon ECS Fargate**.

### Why ECS Fargate?

Advantages:

* No EC2 management.
* No Kubernetes control plane.
* Automatic scaling.
* Simple deployments.
* Lower operational overhead.
* Native AWS integrations.
* Good fit for stateless services.

Since the agent state is stored externally (Redis/DynamoDB), containers can be replaced without losing conversations.

---

## Why Not EKS?

EKS is powerful but introduces additional operational complexity:

* Kubernetes upgrades
* Cluster autoscaler
* Node groups
* Networking (CNI)
* RBAC management
* Ingress controllers
* Control plane costs

Unless the engineering team already has deep Kubernetes expertise or needs Kubernetes-specific capabilities (e.g., service mesh, CRDs), ECS Fargate provides similar scalability with much lower maintenance.

---

## Why Not Lambda?

Lambda is not a good fit because:

* Voice sessions are long-lived.
* WebSockets require persistent connections.
* Streaming responses don't align well with Lambda's execution model.
* Cold starts can affect latency.

Lambda is better suited for asynchronous background tasks.

---

# Container Registry

Use:

**Amazon ECR**

Why?

* Native IAM integration.
* Image scanning.
* Lifecycle policies.
* Private registry.
* Terraform support.

Each runtime image is immutable and tagged using semantic versions.

Example:

```text
agent-runtime:v2.4.0
agent-runtime:v2.5.0
```

---

# Networking

Deploy everything inside a dedicated VPC.

```text
                 VPC

     Public Subnets
          │
         ALB

──────── Private Boundary ────────

 Private Subnets

 ECS Tasks
 Redis
 DynamoDB Endpoint
 Secrets Manager Endpoint
```

### Principles

* No public IPs for workloads.
* Private subnets for all application services.
* VPC Endpoints for AWS services to avoid internet traffic.
* NAT Gateway only where necessary.

---

# Storage Choices

## 1. DynamoDB

Purpose:

* Session metadata
* Conversation state
* Agent checkpoints

Why?

* High throughput
* Low latency
* Automatic scaling
* No operational overhead

---

## 2. Amazon S3

Purpose:

* Conversation transcripts
* Evaluation datasets
* Audio recordings
* Prompt versions
* Audit logs

Benefits:

* Durable
* Cost-effective
* Lifecycle policies
* Versioning

---

## 3. ElastiCache Redis

Purpose:

* Active session cache
* Streaming state
* Temporary conversation context

Why?

Redis offers sub-millisecond access, reducing repeated reads from DynamoDB during active sessions.

---

# Secrets Management

Use:

AWS Secrets Manager

Store:

* API keys
* Database credentials
* OAuth tokens
* LLM provider credentials

Never bake secrets into:

* Docker images
* Terraform variables
* Git repositories

---

# Identity and Permissions

Every ECS task gets its own IAM role.

Example:

```text
Appointment Agent

↓

Can call:

Appointment API

Cannot call:

Billing API
```

This follows the principle of least privilege.

---

# Messaging

Some operations should be asynchronous.

Use Amazon SQS for:

* Transcript processing
* Analytics
* Notifications
* Batch evaluations

Benefits:

* Decouples services
* Improves resilience
* Handles traffic spikes

---

# Event Processing

Use Amazon EventBridge for domain events.

Example events:

```text
Conversation Started

Conversation Completed

Agent Escalated

Deployment Finished

Evaluation Failed
```

This enables event-driven workflows without tight coupling.

---

# API Layer

Use:

* Application Load Balancer (ALB) for HTTP/WebSocket traffic.
* API Gateway if exposing public APIs with features like authentication, rate limiting, and API keys.

Internal service-to-service communication remains within the VPC.

---

# Terraform Organization

A modular structure keeps infrastructure reusable and maintainable.

```text
terraform/
│
├── modules/
│   ├── vpc/
│   ├── ecs/
│   ├── ecr/
│   ├── alb/
│   ├── dynamodb/
│   ├── redis/
│   ├── iam/
│   ├── cloudwatch/
│   └── security/
│
├── environments/
│   ├── dev/
│   ├── staging/
│   └── production/
│
└── shared/
```

---

# Environment Strategy

Maintain separate AWS accounts for:

```text
Organization

├── Shared Services
├── Development
├── Staging
└── Production
```

Benefits:

* Strong isolation
* Separate IAM boundaries
* Independent billing
* Reduced blast radius

---

# Terraform State

Use a remote backend:

* S3 bucket for state storage
* DynamoDB table for state locking

```text
Terraform

↓

S3 Backend

↓

DynamoDB Lock Table
```

This prevents concurrent state modifications and supports team collaboration.

---

# Auto Scaling

Scale ECS services based on metrics such as:

* CPU utilization
* Memory utilization
* Request count
* Active session count (custom CloudWatch metric)

Scaling based on active sessions is particularly relevant for conversational AI workloads.

---

# High Availability

Deploy resources across multiple Availability Zones:

* ALB
* ECS tasks
* Redis (Multi-AZ)
* NAT Gateways (where used)

S3 and DynamoDB are inherently multi-AZ.

---

# CI/CD Pipeline (High Level)

```text
Developer
     │
 GitHub
     │
 GitHub Actions
     │
 Build Docker Image
     │
 Push to ECR
     │
 Deploy to ECS
```

Terraform changes follow a separate pipeline with plan, review, and apply stages.

---

# Major Infrastructure Decisions

| Component          | Chosen            | Rejected                  | Reason                                                       |
| ------------------ | ----------------- | ------------------------- | ------------------------------------------------------------ |
| Compute            | ECS Fargate       | EKS                       | Lower operational overhead while meeting scalability needs   |
| Container Registry | Amazon ECR        | Docker Hub                | Better IAM integration and security                          |
| State Store        | DynamoDB          | PostgreSQL                | Scales well for session state with minimal maintenance       |
| Cache              | ElastiCache Redis | In-memory container cache | Shared, resilient, and survives container restarts           |
| Storage            | Amazon S3         | EFS                       | Better suited for transcripts and immutable artifacts        |
| Messaging          | Amazon SQS        | Kafka                     | Simpler managed queue for asynchronous processing            |
| Events             | EventBridge       | Custom event bus          | Native event routing and integrations                        |
| IaC                | Terraform Modules | Monolithic Terraform      | Improved reuse, maintainability, and environment consistency |

---

## Why This Infrastructure?

This architecture prioritizes:

* **Operational simplicity** by choosing managed AWS services where possible.
* **Resilience** through stateless compute, externalized session state, and multi-AZ deployments.
* **Security** with private networking, least-privilege IAM roles, and centralized secrets management.
* **Scalability** using ECS Fargate, DynamoDB, Redis, and event-driven components.
* **Maintainability** through modular Terraform and isolated AWS accounts.

This provides a strong foundation for the next section: **Step 3 – Runtime Lifecycle & Deployment Harness**, which covers how agent runtimes are built, versioned, deployed, rolled back, and how live patient sessions are protected during releases.
