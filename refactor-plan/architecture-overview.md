# OneStream Pipeline Architecture Overview

## Executive Summary

This document provides a comprehensive architectural view of the OneStream pipeline refactoring initiative. The transformation moves from a monolithic 600+ line pipeline to service-specific pipelines using immutable container digests and cryptographic signing.

## Current State Architecture

```mermaid
graph TB
    subgraph "Source Control"
        SC[Multiple Service Repos<br/>iam, auth, utility, cls, iac]
    end
    
    subgraph "Build System - Monolithic Pipeline"
        MP[package-build-data.yml<br/>600+ lines]
        MP --> CB[Complex Build Logic<br/>15+ dependencies]
        CB --> PR[Pipeline Resources<br/>Intricate triggers]
        PR --> BS[Bash Scripts<br/>build_metadata_manager.sh<br/>git_commit_datastores.sh]
    end
    
    subgraph "Artifact Storage"
        BL[Blob Storage<br/>Checksum management<br/>Metadata tracking]
        BL --> MD[Manual Download<br/>15-20 min overhead]
    end
    
    subgraph "Container Registry"
        ACR[Azure Container Registry<br/>Tag-based references]
    end
    
    subgraph "Deployment"
        AKS[AKS Clusters<br/>Dev/Staging/Prod]
    end
    
    SC --> MP
    BS --> BL
    MD --> ACR
    ACR --> AKS
    
    style MP fill:#ffcccc,stroke:#ff0000
    style CB fill:#ffcccc,stroke:#ff0000
    style BL fill:#ffcccc,stroke:#ff0000
    style MD fill:#ffcccc,stroke:#ff0000
```

### Current State Problems

| Issue | Impact | Cost |
|-------|--------|------|
| **45-60 min build time** | Slow feedback cycles | Developer productivity loss |
| **60% success rate** | Frequent failures | Time wasted on troubleshooting |
| **Blob storage complexity** | Operational overhead | $70K+/year storage costs |
| **Coupled services** | Single failure affects all | High blast radius |
| **Complex debugging** | Hard to isolate issues | Extended MTTR |

## Target State Architecture

```mermaid
graph TB
    subgraph "Source Control"
        IAM[IAM Service<br/>src/iam/**]
        AUTH[Auth Service<br/>src/auth/**]
        UTIL[Utility Service<br/>src/utility/**]
    end
    
    subgraph "Service-Specific Pipelines"
        IAMP[IAM Pipeline<br/><150 lines]
        AUTHP[Auth Pipeline<br/><150 lines]
        UTILP[Utility Pipeline<br/><150 lines]
    end
    
    subgraph "Build & Security"
        BUILD[Container Build<br/>Capture Digest]
        SIGN[Cosign Sign<br/>Azure Key Vault HSM]
        SCAN[Security Scan<br/>Trivy CVE analysis]
    end
    
    subgraph "Multi-Environment ACR"
        DEV[Dev ACR<br/>onestreamdev]
        STAGE[Staging ACR<br/>onestreamstaging]
        PROD[Production ACR<br/>onestreamprod]
    end
    
    subgraph "Deployment & Verification"
        VERIFY[Signature Verification<br/>Cosign verify]
        DEPLOY[Kubernetes Deploy<br/>Digest-based reference]
    end
    
    IAM --> IAMP
    AUTH --> AUTHP
    UTIL --> UTILP
    
    IAMP --> BUILD
    AUTHP --> BUILD
    UTILP --> BUILD
    
    BUILD --> SIGN
    BUILD --> SCAN
    SIGN --> DEV
    
    DEV -->|Direct ACR Import| STAGE
    STAGE -->|Direct ACR Import| PROD
    
    PROD --> VERIFY
    VERIFY --> DEPLOY
    
    style IAMP fill:#ccffcc,stroke:#00ff00
    style AUTHP fill:#ccffcc,stroke:#00ff00
    style UTILP fill:#ccffcc,stroke:#00ff00
    style SIGN fill:#ffffcc,stroke:#ffaa00
    style VERIFY fill:#ffffcc,stroke:#ffaa00
```

### Target State Benefits

| Benefit | Improvement | Quantified Impact |
|---------|-------------|-------------------|
| **Build time** | <25 minutes | 40%+ faster |
| **Success rate** | >95% | 35% improvement |
| **Storage costs** | Eliminated blob storage | $70K+/year savings |
| **Blast radius** | Service-specific failures | Reduced by 83% (6 services to 1) |
| **Debugging** | Isolated service logs | <30 min MTTR (vs 2-4 hours) |

## Core Architectural Patterns

### 1. Immutable Digest Pattern

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Build as Build Stage
    participant DevACR as Dev ACR
    participant StageACR as Staging ACR
    participant ProdACR as Production ACR
    participant Deploy as Deployment
    
    Dev->>Build: Push code
    Build->>Build: Build container
    Build->>DevACR: Push image
    DevACR-->>Build: Return sha256:abc123...
    
    Note over Build,DevACR: Digest captured:<br/>sha256:abc123...
    
    Build->>DevACR: Sign with Cosign
    
    DevACR->>StageACR: Import by digest<br/>sha256:abc123...
    
    Note over StageACR: Same digest,<br/>same binary
    
    StageACR->>ProdACR: Import by digest<br/>sha256:abc123...
    
    Note over ProdACR: Exact same<br/>binary as Dev
    
    ProdACR->>Deploy: Deploy by digest
    Deploy->>Deploy: Verify signature
    Deploy->>Deploy: Deploy container
```

**Key Principle**: Build once, promote the exact same binary through all environments.

### 2. Cryptographic Trust Chain

```mermaid
graph TB
    subgraph "Build Phase"
        A[Source Code] --> B[Container Build]
        B --> C[Capture Digest<br/>sha256:abc123...]
    end
    
    subgraph "Signing Phase"
        C --> D[Cosign Sign]
        D --> E[Azure Key Vault HSM<br/>FIPS 140-2 Level 3]
        E --> F[Signature Attached<br/>to Image]
    end
    
    subgraph "Verification Phase"
        F --> G[Pre-Deployment Check]
        G --> H{Verify Signature}
        H -->|Valid| I[Deploy]
        H -->|Invalid| J[Reject & Alert]
    end
    
    subgraph "Audit Trail"
        I --> K[Log Deployment]
        J --> L[Log Rejection]
        K --> M[Complete Traceability]
        L --> M
    end
    
    style E fill:#FFD700,stroke:#FF8C00
    style H fill:#FFD700,stroke:#FF8C00
    style M fill:#90EE90,stroke:#006400
```

**Key Principle**: Every deployment must verify cryptographic signatures before proceeding.

### 3. Template Hierarchy Pattern

```mermaid
graph TB
    subgraph "Governance Layer - Extends Template"
        GOV[secure-build-template.yml]
        GOV --> REQ1[Security Scan Required]
        GOV --> REQ2[Signing Required]
        GOV --> REQ3[Digest Tracking Required]
        GOV --> REQ4[Manual Approval for Prod]
    end
    
    subgraph "Service Pipelines - Consume Template"
        IAM[iam-service-pipeline.yml<br/>extends: secure-build-template.yml]
        AUTH[auth-service-pipeline.yml<br/>extends: secure-build-template.yml]
    end
    
    subgraph "Reusable Steps - Include Template"
        BUILD[build-container.yml]
        SIGN[cosign-sign.yml]
        IMPORT[acr-direct-import.yml]
        DEPLOY[deploy-with-verification.yml]
    end
    
    GOV --> IAM
    GOV --> AUTH
    
    IAM --> BUILD
    IAM --> SIGN
    IAM --> IMPORT
    IAM --> DEPLOY
    
    AUTH --> BUILD
    AUTH --> SIGN
    AUTH --> IMPORT
    AUTH --> DEPLOY
    
    style GOV fill:#FFB6C1,stroke:#8B0000
    style BUILD fill:#87CEEB,stroke:#00008B
    style SIGN fill:#87CEEB,stroke:#00008B
    style IMPORT fill:#87CEEB,stroke:#00008B
    style DEPLOY fill:#87CEEB,stroke:#00008B
```

**Key Principle**: Centralized governance, distributed execution, DRY reusable components.

## Service Principal Architecture

```mermaid
graph TB
    subgraph "Development Environment"
        DEVSP[sp-onestream-dev-pipeline]
        DEVSP --> DEVACR[Dev ACR: Push/Pull]
        DEVSP --> DEVAKS[Dev AKS: Deploy]
        DEVSP --> DEVKV[Key Vault: Sign]
    end
    
    subgraph "Staging Environment"
        STAGESP[sp-onestream-staging-pipeline]
        STAGESP --> STAGEACR[Staging ACR: Pull Only]
        STAGESP --> STAGEAKS[Staging AKS: Deploy]
    end
    
    subgraph "Production Environment"
        PRODSP[sp-onestream-prod-pipeline]
        PRODSP --> PRODACR[Prod ACR: Pull Only]
        PRODSP --> PRODAKS[Prod AKS: Deploy]
    end
    
    subgraph "Promotion Engine"
        PROMSP[sp-onestream-promotion-engine]
        PROMSP --> DEVACR2[Dev ACR: Pull]
        PROMSP --> STAGEACR2[Staging ACR: Import]
        PROMSP --> PRODACR2[Prod ACR: Import]
        PROMSP --> KV[Key Vault: Verify Signatures]
    end
    
    style DEVSP fill:#90EE90
    style STAGESP fill:#FFD700
    style PRODSP fill:#FF6347
    style PROMSP fill:#9370DB
```

### Least Privilege RBAC Matrix

| Service Principal | Dev ACR | Staging ACR | Prod ACR | Dev AKS | Staging AKS | Prod AKS | Key Vault |
|-------------------|---------|-------------|----------|---------|-------------|----------|-----------|
| **dev-pipeline** | Push/Pull | ❌ | ❌ | Deploy | ❌ | ❌ | Sign |
| **staging-pipeline** | ❌ | Pull | ❌ | ❌ | Deploy | ❌ | ❌ |
| **prod-pipeline** | ❌ | ❌ | Pull | ❌ | ❌ | Deploy | ❌ |
| **promotion-engine** | Pull | Import | Import | ❌ | ❌ | ❌ | Verify |

## Multi-Environment Flow

```mermaid
graph TB
    subgraph "Development Environment"
        DEV1[Code Commit to 'develop']
        DEV2[Auto Build & Test]
        DEV3[Sign with Dev Key]
        DEV4[Deploy to Dev AKS]
        DEV5[Health Check]
        
        DEV1 --> DEV2
        DEV2 --> DEV3
        DEV3 --> DEV4
        DEV4 --> DEV5
    end
    
    subgraph "Staging Environment"
        STAGE1[Code Merge to 'main']
        STAGE2[Auto Promotion to Staging ACR]
        STAGE3[QA Approval Gate]
        STAGE4[Deploy to Staging AKS]
        STAGE5[Integration Tests]
        
        DEV5 -->|Merge to main| STAGE1
        STAGE1 --> STAGE2
        STAGE2 --> STAGE3
        STAGE3 --> STAGE4
        STAGE4 --> STAGE5
    end
    
    subgraph "Production Environment"
        PROD1[Manual Approval]
        PROD2[Sign with Prod Key]
        PROD3[Promotion to Prod ACR]
        PROD4[Canary Deployment 10%]
        PROD5[Health Monitor 15 min]
        PROD6{Health Check}
        PROD7[Full Rollout]
        PROD8[Automatic Rollback]
        
        STAGE5 --> PROD1
        PROD1 --> PROD2
        PROD2 --> PROD3
        PROD3 --> PROD4
        PROD4 --> PROD5
        PROD5 --> PROD6
        PROD6 -->|Healthy| PROD7
        PROD6 -->|Unhealthy| PROD8
    end
    
    style DEV5 fill:#90EE90
    style STAGE5 fill:#FFD700
    style PROD7 fill:#87CEEB
    style PROD8 fill:#FF6347
```

## Deployment Strategy

### Canary Deployment with Automated Rollback

```mermaid
stateDiagram-v2
    [*] --> Deploying: Initiate Deployment
    Deploying --> CanaryActive: Deploy 10% Traffic
    CanaryActive --> Monitoring: Monitor Health
    
    Monitoring --> HealthCheck: After 15 minutes
    
    HealthCheck --> FullRollout: Health OK<br/>(Error rate < 5%<br/>CPU < 90%<br/>Memory < 85%)
    HealthCheck --> AutoRollback: Health Degraded
    
    FullRollout --> Production: 100% Traffic
    Production --> [*]
    
    AutoRollback --> PreviousDigest: Restore Previous Version
    PreviousDigest --> HealthVerify: Verify Rollback
    HealthVerify --> Production: Rollback Complete
    Production --> [*]
```

### Rollback Architecture

```mermaid
graph TB
    subgraph "Deployment History"
        H1[Version N-2<br/>sha256:old...]
        H2[Version N-1<br/>sha256:previous...]
        H3[Version N<br/>sha256:current...]
    end
    
    subgraph "Health Monitoring"
        M1[Error Rate Monitor]
        M2[CPU/Memory Monitor]
        M3[Endpoint Health Check]
        M4[User Feedback]
    end
    
    subgraph "Rollback Decision"
        D1{Health Status}
        D1 -->|Degraded| R1[Trigger Rollback]
        D1 -->|Healthy| K1[Keep Current]
    end
    
    subgraph "Rollback Execution"
        R1 --> R2[Identify Last Known Good<br/>sha256:previous...]
        R2 --> R3[Verify Signature]
        R3 --> R4[Update Deployment]
        R4 --> R5[Monitor Recovery]
        R5 --> R6[Alert Team]
    end
    
    M1 --> D1
    M2 --> D1
    M3 --> D1
    M4 --> D1
    
    style R1 fill:#FF6347
    style R3 fill:#FFD700
    style R5 fill:#90EE90
```

**Rollback Targets**:
- Detection Time: <2 minutes
- Decision Time: <30 seconds
- Execution Time: <3 minutes
- Total RTO: <5 minutes

## Security Architecture

### Supply Chain Security

```mermaid
graph TB
    subgraph "Code Security"
        CS1[Branch Protection]
        CS2[PR Reviews Required]
        CS3[Status Checks]
    end
    
    subgraph "Build Security"
        BS1[Isolated Build Agent]
        BS2[No Secrets in Logs]
        BS3[Trivy CVE Scan]
        BS4[Policy Compliance]
    end
    
    subgraph "Artifact Security"
        AS1[Cosign Signature]
        AS2[Azure Key Vault HSM<br/>FIPS 140-2 Level 3]
        AS3[Immutable Digest Reference]
        AS4[SLSA Provenance]
    end
    
    subgraph "Deployment Security"
        DS1[Signature Verification]
        DS2[Namespace Isolation]
        DS3[Network Policies]
        DS4[RBAC Enforcement]
    end
    
    subgraph "Runtime Security"
        RS1[Container Image Scanning]
        RS2[Security Context]
        RS3[Pod Security Standards]
        RS4[Audit Logging]
    end
    
    CS1 --> BS1
    CS2 --> BS1
    CS3 --> BS1
    
    BS1 --> AS1
    BS3 --> AS1
    BS4 --> AS1
    
    AS1 --> DS1
    AS3 --> DS1
    AS4 --> DS1
    
    DS1 --> RS1
    DS2 --> RS1
    DS3 --> RS1
    DS4 --> RS1
    
    style AS2 fill:#FFD700
    style DS1 fill:#FFD700
    style RS4 fill:#90EE90
```

### Compliance & Audit Trail

```mermaid
sequenceDiagram
    participant Dev as Developer
    participant Git as Git Commit
    participant Pipeline as Pipeline
    participant ACR as Container Registry
    participant KV as Key Vault
    participant Audit as Azure Monitor
    participant Deploy as Deployment
    
    Dev->>Git: Commit code
    Git->>Audit: Log: Code commit
    
    Git->>Pipeline: Trigger build
    Pipeline->>Audit: Log: Pipeline started
    
    Pipeline->>ACR: Build & push image
    ACR->>Audit: Log: Image pushed with digest
    
    Pipeline->>KV: Request signing
    KV->>Audit: Log: Signature operation
    
    Pipeline->>ACR: Attach signature
    ACR->>Audit: Log: Signature attached
    
    ACR->>Deploy: Promote to prod
    Deploy->>KV: Verify signature
    KV->>Audit: Log: Signature verified
    
    Deploy->>Deploy: Deploy container
    Deploy->>Audit: Log: Deployment complete
    
    Note over Audit: Complete traceable<br/>audit trail from<br/>commit to production
```

## Performance Architecture

### Build Performance Optimization

```mermaid
graph LR
    subgraph "Build Parallelization"
        B1[Code Analysis]
        B2[Unit Tests]
        B3[Container Build]
        B4[Security Scan]
    end
    
    subgraph "Caching Strategy"
        C1[Docker Layer Cache]
        C2[Package Cache<br/>npm/nuget/pip]
        C3[Build Artifact Cache]
    end
    
    subgraph "Optimization Techniques"
        O1[Multi-stage Dockerfile]
        O2[Minimal Base Image]
        O3[Parallel Test Execution]
        O4[Incremental Builds]
    end
    
    B1 & B2 & B3 & B4 --> C1
    C1 --> C2
    C2 --> C3
    
    C3 --> O1
    O1 --> O2
    O2 --> O3
    O3 --> O4
    
    O4 --> R[Build Complete<br/><10 minutes]
    
    style R fill:#90EE90
```

### Network Performance Optimization

```mermaid
graph TB
    subgraph "Direct ACR Import"
        A1[Dev ACR<br/>East US 2]
        A2[Staging ACR<br/>East US 2]
        A3[Prod ACR<br/>East US 2]
    end
    
    subgraph "Benefits"
        B1[Same Region Transfer]
        B2[Azure Backbone Network]
        B3[No Public Internet]
        B4[Digest Preservation]
    end
    
    A1 -->|Direct Import| A2
    A2 -->|Direct Import| A3
    
    A1 --> B1
    A2 --> B2
    A3 --> B3
    
    B1 & B2 & B3 --> B4
    
    B4 --> R[<5 min promotion<br/>vs 15-20 min blob storage]
    
    style R fill:#90EE90
```

## Monitoring & Observability

```mermaid
graph TB
    subgraph "Pipeline Metrics"
        PM1[Build Duration]
        PM2[Success Rate]
        PM3[Queue Time]
        PM4[Resource Usage]
    end
    
    subgraph "Application Metrics"
        AM1[Health Endpoints]
        AM2[Error Rates]
        AM3[Latency]
        AM4[Throughput]
    end
    
    subgraph "Security Metrics"
        SM1[Signature Verification Rate]
        SM2[CVE Count]
        SM3[Failed Deployments]
        SM4[Rollback Frequency]
    end
    
    subgraph "Azure Monitor"
        MON[Log Analytics Workspace]
    end
    
    subgraph "Visualization"
        DASH[Grafana Dashboards]
        ALERT[Alert Rules]
    end
    
    PM1 & PM2 & PM3 & PM4 --> MON
    AM1 & AM2 & AM3 & AM4 --> MON
    SM1 & SM2 & SM3 & SM4 --> MON
    
    MON --> DASH
    MON --> ALERT
    
    ALERT --> TEAM[Team Notifications]
    
    style MON fill:#87CEEB
    style DASH fill:#90EE90
```

## Technology Stack

### Core Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Pipeline** | Azure DevOps YAML | CI/CD automation |
| **Container Registry** | Azure Container Registry (Premium) | Image storage with digests |
| **Signing** | Cosign + Azure Key Vault HSM | Cryptographic signatures |
| **Security Scanning** | Trivy | CVE vulnerability detection |
| **Orchestration** | Azure Kubernetes Service | Container orchestration |
| **Monitoring** | Azure Monitor + Grafana | Observability and alerting |
| **Secrets** | Azure Key Vault | Secure credential storage |
| **Networking** | Azure Virtual Network | Network isolation |

### Version Requirements

- Azure DevOps: Latest
- Cosign: v2.2.0+
- Trivy: v0.48.0+
- Kubernetes: 1.28+
- Azure Linux: 3.0 (FIPS-compliant)
- Helm: 3.13+

## Migration Strategy

```mermaid
graph TB
    START[Current: Monolithic Pipeline] --> PHASE1[Phase 1: Foundation<br/>Templates & Infrastructure]
    PHASE1 --> PHASE2[Phase 2: IAM Pilot<br/>First Service Migration]
    PHASE2 --> PHASE3[Phase 3: Parallel Testing<br/>Legacy vs New Comparison]
    PHASE3 --> DECISION{Results<br/>Acceptable?}
    DECISION -->|No| ITERATE[Iterate & Fix]
    ITERATE --> PHASE3
    DECISION -->|Yes| PHASE4[Phase 4: Service Expansion<br/>Migrate Remaining Services]
    PHASE4 --> PHASE5[Phase 5: Production Rollout<br/>Switch Primary Pipeline]
    PHASE5 --> PHASE6[Phase 6: Legacy Decommission<br/>Remove Old Pipeline]
    PHASE6 --> END[Target: Service-Specific Pipelines]
    
    style START fill:#ffcccc
    style END fill:#90EE90
    style DECISION fill:#FFD700
```

## Disaster Recovery

### Backup Strategy

```mermaid
graph TB
    subgraph "Critical Assets"
        A1[Pipeline YAML Files<br/>Git Repository]
        A2[Container Images<br/>ACR with Geo-Replication]
        A3[Signing Keys<br/>Key Vault with Backup]
        A4[Deployment Configs<br/>Git Repository]
    end
    
    subgraph "Backup Locations"
        B1[Primary Region<br/>East US 2]
        B2[DR Region<br/>West US 2]
    end
    
    subgraph "Recovery Procedures"
        R1[RTO: 4 hours]
        R2[RPO: <1 hour]
        R3[Automated Failover]
    end
    
    A1 --> B1
    A2 --> B1
    A3 --> B1
    A4 --> B1
    
    B1 -.Replicate.-> B2
    
    B2 --> R1
    B2 --> R2
    B2 --> R3
    
    style R1 fill:#FFD700
    style R2 fill:#FFD700
    style R3 fill:#90EE90
```

## Conclusion

This architecture provides:

✅ **Performance**: <25 minute builds (vs 45-60 minutes)  
✅ **Reliability**: >95% success rate (vs 60%)  
✅ **Security**: 100% cryptographically signed containers  
✅ **Cost**: $70K+/year savings (blob storage elimination)  
✅ **Maintainability**: <150 lines per service (vs 600+ monolithic)  
✅ **Scalability**: Easy addition of new services  
✅ **Auditability**: Complete traceability from commit to production  

---

**Next Steps**: Proceed to [Stage 1: Foundation & Templates](./stage-1-foundation-templates.md)
