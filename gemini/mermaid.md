flowchart TD
    %% COLOR DEFINITIONS
    classDef dev fill:#232F3E,stroke:#FF9900,color:#FFF;
    classDef ci fill:#1E293B,stroke:#38BDF8,color:#FFF;
    classDef cd fill:#0F172A,stroke:#10B981,color:#FFF;
    classDef prod fill:#18181B,stroke:#F43F5E,color:#FFF;
    classDef state fill:#312E81,stroke:#818CF8,color:#FFF;

    subgraph STAGE1 ["1. Continuous Integration & Artifact Build"]
        A["Developer Git Push / PR"] :::dev --> B["GitHub Actions CI"] :::ci
        B --> C["Linting, Unit Tests & Security Scans<br/>(TruffleHog, tfsec, Checkov)"] :::ci
        C --> D["DeepEval Clinical Quality Gate<br/>(100 Synthetic Patient Scenarios)"] :::ci
        D --> E{"Passes > 98%<br/>Eval Accuracy?"} :::ci
        E -- No --> F["Fail PR & Block Merge"] :::ci
        E -- Yes --> G["Build OCI Container Image<br/>Tag: v1.4.2-sha-8f3a92"] :::ci
        G --> H["Push to Amazon ECR<br/>(KMS Encrypted & Immutable)"] :::ci
    end

    subgraph STAGE2 ["2. Infrastructure & Argo CD Progressive Deployment"]
        H --> I["Argo CD Detects Git Manifest Update"] :::cd
        I --> J["Argo Rollouts Stage: 10% Canary"] :::cd
        J --> K["Provision New Pods in EKS<br/>(Karpenter Auto-scaling)"] :::cd
    end

    subgraph STAGE3 ["3. Zero-Downtime Live Session Handling"]
        K --> L["Existing Pod Catch SIGTERM Signal"] :::prod
        L --> M["Set Readiness Probe = Unhealthy<br/>(Remove from NLB Routing)"] :::prod
        
        subgraph SESSION_DRAIN ["Graceful Session Draining (15-min Window)"]
            N["Active Voice Calls Stay Connected to Old Pod"] :::state
            O["Turn-by-Turn Context Continually Written to ElastiCache Redis"] :::state
            P["New Inbound Calls Directed strictly to 10% Canary Pods"] :::state
        end
        
        M --> SESSION_DRAIN
    end

    subgraph STAGE4 ["4. Live Observability & Automated Rollback Gate"]
        SESSION_DRAIN --> Q["Datadog & Langfuse Telemetry Collection<br/>(10-min Canary Analysis)"] :::prod
        
        Q --> R{"Canary Metrics Check:<br/>• HTTP/WS 5xx < 0.5%?<br/>• P95 Latency < 1,500ms?<br/>• Repetitive Loop Spike < 1%?"} :::prod
        
        R -- Threshold Breached --> S["AUTOMATED ROLLBACK<br/>(Argo Reverts Image Tag instantly)"] :::prod
        S --> T["Terminate Canary Pods<br/>Route Traffic to Stable Release"] :::prod
        
        R -- All Metrics Healthy --> U["Promote Canary to 100% Production"] :::cd
        U --> V["Old Pods Finish Remaining Sessions & Terminate Gracefully"] :::cd
    end
