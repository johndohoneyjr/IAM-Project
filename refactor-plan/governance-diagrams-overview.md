# OneStream Pipeline Refactoring - Governance Architecture Diagrams

## Executive Summary

This document provides three comprehensive SVG diagrams for OneStream's pipeline refactoring governance validation. These diagrams demonstrate the technical architecture, security model, and implementation strategy for transforming from a monolithic 600+ line pipeline to service-specific pipelines with immutable container digests and cryptographic signing.

## Architecture Overview

The OneStream pipeline refactoring initiative transitions from:
- **Current State**: Monolithic build pipeline (45-60 min runtime, ~60% success rate)
- **Target State**: Service-specific pipelines (<25 min runtime, >95% success rate)

### Key Benefits
- **60% performance improvement** in build times
- **Immutable container digests** eliminate blob storage dependency
- **Cryptographic signing** with Azure Key Vault HSM (FIPS 140-2 Level 3)
- **Zero-downtime deployments** with automated rollback capabilities
- **Cost optimization** eliminating $70K+/year in blob storage costs

## Governance Diagrams

### 1. Template Design Architecture
**File**: `template-design-architecture.svg`
**Purpose**: Shows the central template framework and 12 service-specific pipeline architecture

**Key Components**:
- Central Template Framework with governance controls
- 12 Service-Specific Pipelines (IAM pilot complete, others staged)
- Reusable Template Steps (build, test, sign, deploy)
- Template Variables for environment-specific configuration
- Security scans, Cosign signing, and compliance gates

**Governance Value**:
- Demonstrates consistent template-based approach for all services
- Shows reusable components reducing maintenance overhead
- Illustrates governance controls embedded in template framework
- Provides clear migration path for remaining 11 services

### 2. Infrastructure Architecture
**File**: `infrastructure-architecture.svg`
**Purpose**: End-to-end Azure infrastructure showing Dev/Test/Prod environments and platform integration

**Key Components**:
- **Azure DevOps**: Pipeline orchestration, repos, service connections
- **Development Environment**: onestreamdev ACR, AKS cluster, Key Vault, VNet
- **Test/Staging Environment**: onestreamstaging ACR, AKS cluster, monitoring, load testing
- **Production Environment**: onestreamprod ACR, AKS cluster, security, backup/DR
- **Shared Services**: Azure AD, DNS, Front Door, Policy, Sentinel
- **Security Framework**: Cryptographic signing, vulnerability management, network security, access control, audit/monitoring, compliance
- **Performance Metrics**: DORA metrics alignment (deployment frequency, lead time, change failure rate, MTTR)

**Governance Value**:
- Complete view of Azure infrastructure across all environments
- Security controls and compliance framework clearly defined
- Performance targets aligned with DORA metrics for elite performers
- Microsoft Well-Architected Framework compliance demonstrated

### 3. RBAC Security Architecture
**File**: `rbac-security-diagram.svg`
**Purpose**: Detailed Azure Entra ID role-based access control and least privilege security model

**Key Components**:
- **Azure Entra ID**: Service principals, built-in RBAC roles, conditional access, PIM
- **Development Security**: sp-onestream-poc-dev-pipeline with AcrPush/Pull, cluster user, crypto user
- **Staging Security**: sp-onestream-poc-staging-pipeline with read-only access, promotion engine
- **Production Security**: sp-onestream-poc-prod-pipeline with strict read-only, approval gates
- **Cross-Environment Framework**: Zero trust, least privilege, defense in depth
- **Compliance Validation**: Identity, network, cryptographic, deployment, audit, governance
- **Security Metrics**: Zero incidents, 100% signature verification, <5min response, 100% compliance

**Governance Value**:
- Demonstrates least privilege access model across all environments
- Shows complete separation of environment permissions
- Illustrates cryptographic signing and verification controls
- Provides security metrics and compliance validation framework

## Technical Implementation Details

### Immutable Digest Approach
- **Container Images**: Referenced by sha256 digest, not mutable tags
- **Cross-Environment Promotion**: Direct ACR-to-ACR imports preserving exact binary content
- **Cryptographic Verification**: Cosign signatures with Azure Key Vault HSM
- **Audit Trail**: Complete traceability from source code to production deployment

### Service-Specific Pipeline Benefits
- **Independent Deployments**: Services can deploy without affecting others
- **Reduced Blast Radius**: Failures isolated to single service
- **Faster Feedback**: <25 minute build times enable rapid iteration
- **Template Consistency**: Governance controls embedded in reusable templates

### Security Model
- **Zero Trust Architecture**: Never trust, always verify approach
- **Least Privilege**: Minimum required permissions with environment isolation
- **Multi-Factor Authentication**: Required for all human access
- **Continuous Monitoring**: Real-time security event detection and response

## Compliance & Governance

### Standards Alignment
- **SOC 2 Type II**: Control framework implementation
- **NIST Cybersecurity Framework**: Risk management and security controls
- **FIPS 140-2 Level 3**: Cryptographic key management compliance
- **Azure Well-Architected Framework**: Reliability, security, cost optimization, operational excellence, performance efficiency

### Approval Gates
- **Development**: Automatic deployment on `develop` branch
- **Staging**: Manual approval required for promotion
- **Production**: Change advisory board approval + business owner sign-off

### Audit Capabilities
- **Complete Audit Trail**: From source commit to production deployment
- **Access Logging**: All service principal activities tracked
- **Change Tracking**: Immutable digest references enable precise rollback
- **Compliance Reporting**: Automated SOC 2 and NIST framework validation

## Performance & Reliability Targets

### DORA Metrics Alignment
| Metric | Current (Legacy) | Target (Refactored) | DORA Elite |
|--------|------------------|---------------------|------------|
| Deployment Frequency | Weekly | Multiple per day | Multiple per day |
| Lead Time for Changes | Days | <1 hour | <1 hour |
| Change Failure Rate | 40% | 0-15% | 0-15% |
| Mean Time to Recovery | Hours | <1 hour | <1 hour |

### Operational Metrics
- **Build Performance**: 45-60 min → <25 min (60% improvement)
- **Success Rate**: 60% → >95% (35 percentage point improvement)
- **Cost Optimization**: Eliminate $70K+/year blob storage costs
- **Availability**: 99.9% uptime SLA with zero-downtime deployments

## Migration Strategy

### Phase 1: Foundation (Complete)
- Template framework development
- IAM service pilot implementation
- Security model establishment
- Infrastructure provisioning

### Phase 2: Service Migration (In Progress)
- Remaining 11 services migrated to template-based approach
- Parallel testing with legacy pipeline
- Performance validation and optimization

### Phase 3: Production Rollout (Planned)
- Legacy pipeline decommissioning
- Full production workload migration
- Cost optimization realization

## Risk Mitigation

### Technical Risks
- **Parallel Testing**: Run both pipelines during transition
- **Automated Rollback**: Digest-based references enable instant rollback
- **Incremental Migration**: One service at a time reduces risk

### Security Risks
- **Defense in Depth**: Multiple security layers with continuous monitoring
- **Least Privilege**: Environment isolation prevents lateral movement
- **Cryptographic Signing**: Supply chain integrity guaranteed

### Operational Risks
- **Monitoring & Alerting**: Real-time visibility into pipeline health
- **Documentation**: Comprehensive runbooks and procedures
- **Training**: Team education on new processes and tools

## Conclusion

These three governance diagrams provide comprehensive technical validation for OneStream's pipeline refactoring initiative. The architecture demonstrates:

1. **Technical Excellence**: Template-based approach with reusable components
2. **Security Leadership**: Zero trust model with cryptographic verification
3. **Operational Excellence**: DORA metrics alignment with elite performance targets
4. **Cost Optimization**: Significant infrastructure cost reduction
5. **Compliance Readiness**: SOC 2 and NIST framework alignment

The refactoring initiative positions OneStream for accelerated software delivery with enhanced security, reliability, and cost efficiency.

---

**Document Version**: 1.0  
**Last Updated**: October 22, 2025  
**Review Status**: Ready for Executive Governance Review  
**Prepared By**: OneStream DevOps Engineering Team