Here's a basic visual preview of your flowchart in a simplified layout:

```text
┌────────────────────────────────────────────────────────────────────────────┐
│              1. Continuous Integration & Artifact Build                    │
└────────────────────────────────────────────────────────────────────────────┘

 Developer Push / PR
         │
         ▼
 GitHub Actions CI
         │
         ▼
 Linting • Unit Tests • Security Scans
         │
         ▼
 DeepEval Clinical Quality Gate
 (100 Synthetic Patient Scenarios)
         │
         ▼
   ┌─────────────────────┐
   │ Accuracy > 98% ?    │
   └───────┬─────────────┘
           │
   ┌───────┴───────────────┐
   │                       │
 No ▼                      ▼ Yes
Fail PR             Build OCI Image
Block Merge                │
                            ▼
                     Push to Amazon ECR
                            │
                            ▼

┌────────────────────────────────────────────────────────────────────────────┐
│       2. Infrastructure & Progressive Deployment (Argo CD)                │
└────────────────────────────────────────────────────────────────────────────┘

                Argo CD Detects Manifest
                         │
                         ▼
                  10% Canary Rollout
                         │
                         ▼
               Provision New Pods (EKS)

                         │
                         ▼

┌────────────────────────────────────────────────────────────────────────────┐
│             3. Zero-Downtime Live Session Handling                         │
└────────────────────────────────────────────────────────────────────────────┘

               Existing Pod Receives SIGTERM
                         │
                         ▼
              Remove from Load Balancer
            (Readiness Probe = Unhealthy)
                         │
                         ▼

          ┌─────────────────────────────────────┐
          │ Graceful Session Draining (15 min)  │
          ├─────────────────────────────────────┤
          │ • Existing calls continue           │
          │ • Context saved to Redis            │
          │ • New calls → Canary Pods           │
          └─────────────────────────────────────┘
                         │
                         ▼

┌────────────────────────────────────────────────────────────────────────────┐
│      4. Observability & Automated Rollback / Promotion                     │
└────────────────────────────────────────────────────────────────────────────┘

          Datadog + Langfuse Telemetry
                    (10 min)
                         │
                         ▼
      ┌────────────────────────────────────┐
      │ Canary Metrics Healthy?            │
      │ 5xx <0.5%                          │
      │ P95 <1500ms                        │
      │ Loop Spike <1%                     │
      └───────────────┬────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        │                           │
        ▼                           ▼
 Automated Rollback         Promote to 100%
 Revert Image Tag            Production
        │                           │
        ▼                           ▼
Terminate Canary Pods      Old Pods Finish
Restore Stable Release     Sessions Gracefully
```

Overall flow:

```text
Developer
    │
    ▼
 CI Pipeline
    │
    ▼
 Build & Push Image
    │
    ▼
 Argo CD
    │
    ▼
10% Canary
    │
    ▼
Session Draining
    │
    ▼
Telemetry Analysis
    │
 ┌──┴──┐
 ▼     ▼
Rollback  Promote
```

This layout closely matches your Mermaid diagram while making the four stages and the decision points easier to visualize in plain text.
