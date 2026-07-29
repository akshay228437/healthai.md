flowchart TD

%% Color Definitions
classDef dev fill:#232F3E,stroke:#FF9900,color:#FFFFFF
classDef ci fill:#1E293B,stroke:#38BDF8,color:#FFFFFF
classDef cd fill:#0F172A,stroke:#10B981,color:#FFFFFF
classDef prod fill:#18181B,stroke:#F43F5E,color:#FFFFFF
classDef state fill:#312E81,stroke:#818CF8,color:#FFFFFF

subgraph STAGE1["1. Continuous Integration & Artifact Build"]
    A["Developer Git Push / PR"] --> B["GitHub Actions CI"]
    B --> C["Linting, Unit Tests & Security Scans\n(TruffleHog, tfsec, Checkov)"]
    C --> D["DeepEval Clinical Quality Gate\n(100 Synthetic Patient Scenarios)"]
    D --> E{"Passes > 98%\nEval Accuracy?"}
    E -- No --> F["Fail PR & Block Merge"]
    E -- Yes --> G["Build OCI Container Image\nTag: v1.4.2-sha-8f3a92"]
    G --> H["Push to Amazon ECR\n(KMS Encrypted & Immutable)"]
end

subgraph STAGE2["2. Infrastructure & Argo CD Progressive Deployment"]
    H --> I["Argo CD Detects Git Manifest Update"]
    I --> J["Argo Rollouts Stage: 10% Canary"]
    J --> K["Provision New Pods in EKS\n(Karpenter Auto-scaling)"]
end

subgraph STAGE3["3. Zero-Downtime Live Session Handling"]
    K --> L["Existing Pod Catch SIGTERM Signal"]
    L --> M["Set Readiness Probe = Unhealthy\n(Remove from NLB Routing)"]

    subgraph SESSION_DRAIN["Graceful Session Draining (15-min Window)"]
        N["Active Voice Calls Stay Connected to Old Pod"]
        O["Turn-by-Turn Context Written to ElastiCache Redis"]
        P["New Inbound Calls Routed to 10% Canary Pods"]
    end

    M --> N
    N --> O
    O --> P
end

subgraph STAGE4["4. Live Observability & Automated Rollback Gate"]
    P --> Q["Datadog & Langfuse Telemetry Collection\n(10-min Canary Analysis)"]

    Q --> R{"Canary Metrics Check\nHTTP/WS 5xx < 0.5%\nP95 Latency < 1500ms\nLoop Spike < 1%?"}

    R -- Threshold Breached --> S["AUTOMATED ROLLBACK\nArgo Reverts Image Tag"]
    S --> T["Terminate Canary Pods\nRoute Traffic to Stable Release"]

    R -- Healthy --> U["Promote Canary to 100% Production"]
    U --> V["Old Pods Finish Sessions\nTerminate Gracefully"]
end

class A dev
class B,C,D,E,F,G,H ci
class I,J,K,U,V cd
class L,M,Q,R,S,T prod
class N,O,P state
