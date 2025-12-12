# Least Privilege Access Control Model

## Overview

This document defines a comprehensive least privilege access control model for the simplified pipeline architecture, ensuring security while enabling efficient environment promotions through Azure Container Registry (ACR) and Azure DevOps.

## Current Access Control Challenges

### Legacy Complex Permissions
Based on the architecture diagram analysis, the current system has:

```yaml
current_access_issues:
  blob_storage:
    - "Overly broad storage account access"
    - "Shared access signatures (SAS) complexity"
    - "Cross-environment storage permissions"
    - "Manual credential rotation"
  
  pipeline_dependencies:
    - "15+ service principals with overlapping permissions"
    - "Shared Azure DevOps service connections"
    - "Broad contributor access across resources"
    - "Difficult permission auditing"
  
  metadata_repos:
    - "Multiple Git repository access controls"
    - "PAT token proliferation"
    - "Cross-organization permissions (PlatformEng vs Cloud)"
    - "Manual access reviews"
```

## Proposed Least Privilege Architecture

### 1. Environment-Scoped Service Principals

#### Development Environment
```yaml
sp_onestream_dev:
  name: "sp-onestream-dev-pipeline"
  purpose: "Development environment builds and deployments"
  permissions:
    acr_dev:
      - scope: "/subscriptions/{sub-id}/resourceGroups/dev-rg/providers/Microsoft.ContainerRegistry/registries/onestreamdev"
      - role: "AcrPush"
      - justification: "Push built images to dev ACR"
    
    acr_dev_pull:
      - scope: "/subscriptions/{sub-id}/resourceGroups/dev-rg/providers/Microsoft.ContainerRegistry/registries/onestreamdev"
      - role: "AcrPull" 
      - justification: "Pull base images and dependencies"
    
    key_vault_dev:
      - scope: "/subscriptions/{sub-id}/resourceGroups/dev-rg/providers/Microsoft.KeyVault/vaults/onestream-dev-kv"
      - role: "Key Vault Secrets User"
      - justification: "Access signing keys and certificates"
  
  restrictions:
    - "Cannot access staging or production ACRs"
    - "Cannot access production key vaults"
    - "Limited to development resource group"
    - "No ARM template deployment permissions"

sp_onestream_dev_iam:
  name: "sp-onestream-dev-iam-service"
  purpose: "IAM service specific development builds"
  permissions:
    acr_dev_iam:
      - scope: "/subscriptions/{sub-id}/resourceGroups/dev-rg/providers/Microsoft.ContainerRegistry/registries/onestreamdev/repositories/iam"
      - role: "AcrPush"
      - justification: "Push IAM service images only"
    
    acr_dev_iam_pull:
      - scope: "/subscriptions/{sub-id}/resourceGroups/dev-rg/providers/Microsoft.ContainerRegistry/registries/onestreamdev/repositories/iam"
      - role: "AcrPull"
      - justification: "Pull IAM service images only"
```

#### Staging Environment
```yaml
sp_onestream_staging:
  name: "sp-onestream-staging-pipeline"
  purpose: "Staging environment deployments only"
  permissions:
    acr_staging:
      - scope: "/subscriptions/{sub-id}/resourceGroups/staging-rg/providers/Microsoft.ContainerRegistry/registries/onestreamstaging"
      - role: "AcrPull"
      - justification: "Pull images for staging deployments"
    
    # NO PUSH PERMISSIONS - Only imports allowed
    
    aks_staging:
      - scope: "/subscriptions/{sub-id}/resourceGroups/staging-rg/providers/Microsoft.ContainerService/managedClusters/onestream-staging-aks"
      - role: "Azure Kubernetes Service Cluster User Role"
      - justification: "Deploy to staging AKS cluster"
  
  restrictions:
    - "Cannot push directly to staging ACR"
    - "Cannot access production resources"
    - "Cannot modify ACR retention policies"
    - "Read-only access to staging Key Vault"
```

#### Production Environment
```yaml
sp_onestream_production:
  name: "sp-onestream-production-pipeline"
  purpose: "Production deployments with highest restrictions"
  permissions:
    acr_production:
      - scope: "/subscriptions/{sub-id}/resourceGroups/prod-rg/providers/Microsoft.ContainerRegistry/registries/onestreamprod"
      - role: "AcrPull"
      - justification: "Pull images for production deployments"
    
    aks_production:
      - scope: "/subscriptions/{sub-id}/resourceGroups/prod-rg/providers/Microsoft.ContainerService/managedClusters/onestream-prod-aks"
      - role: "Azure Kubernetes Service Cluster User Role"
      - justification: "Deploy to production AKS cluster"
  
  restrictions:
    - "NO PUSH permissions to production ACR"
    - "NO import permissions (handled by promotion pipeline)"
    - "Cannot modify any infrastructure"
    - "Cannot access development or staging"
    - "Read-only access to production Key Vault secrets"
```

### 2. Promotion-Specific Service Principal

```yaml
sp_onestream_promotion:
  name: "sp-onestream-promotion-engine"
  purpose: "Cross-environment image promotions only"
  permissions:
    acr_dev_pull:
      - scope: "/subscriptions/{sub-id}/resourceGroups/dev-rg/providers/Microsoft.ContainerRegistry/registries/onestreamdev"
      - role: "AcrPull"
      - justification: "Source images for promotion"
    
    acr_staging_import:
      - scope: "/subscriptions/{sub-id}/resourceGroups/staging-rg/providers/Microsoft.ContainerRegistry/registries/onestreamstaging"
      - role: "AcrImportImage"
      - justification: "Import from dev to staging"
    
    acr_staging_pull:
      - scope: "/subscriptions/{sub-id}/resourceGroups/staging-rg/providers/Microsoft.ContainerRegistry/registries/onestreamstaging"
      - role: "AcrPull"
      - justification: "Source images for production promotion"
    
    acr_production_import:
      - scope: "/subscriptions/{sub-id}/resourceGroups/prod-rg/providers/Microsoft.ContainerRegistry/registries/onestreamprod"
      - role: "AcrImportImage"
      - justification: "Import from staging to production"
    
    cosign_verification:
      - scope: "/subscriptions/{sub-id}/resourceGroups/shared-rg/providers/Microsoft.KeyVault/vaults/onestream-cosign-kv"
      - role: "Key Vault Crypto User"
      - justification: "Verify image signatures during promotion"
  
  restrictions:
    - "Cannot push images to any ACR"
    - "Cannot deploy to any environment"
    - "Cannot access application secrets"
    - "Limited to import and verification operations only"
```

### 3. Service-Specific Access Controls

#### IAM Service Team
```yaml
team_iam_developers:
  azure_ad_group: "OneStream-IAM-Developers"
  permissions:
    acr_dev_iam:
      - scope: "/subscriptions/{sub-id}/resourceGroups/dev-rg/providers/Microsoft.ContainerRegistry/registries/onestreamdev/repositories/iam"
      - role: "AcrPull"
      - justification: "Pull IAM images for local development"
    
    devops_project:
      - scope: "/OneStreamPlatformEng/OneStreamPlatform"
      - role: "Build Service Account"
      - justification: "Trigger IAM-specific pipelines"
  
  restrictions:
    - "Cannot access other service repositories"
    - "Cannot push to ACR (only pipelines can push)"
    - "Cannot access staging or production"

team_iam_lead:
  azure_ad_group: "OneStream-IAM-Leads"
  permissions:
    acr_all_iam:
      - scope: "/subscriptions/{sub-id}/resourceGroups/*/providers/Microsoft.ContainerRegistry/registries/*/repositories/iam"
      - role: "AcrPull"
      - justification: "Review IAM images across environments"
    
    devops_project_admin:
      - scope: "/OneStreamPlatformEng/OneStreamPlatform/iam-service-pipeline"
      - role: "Pipeline Administrator"
      - justification: "Manage IAM pipeline configuration"
```

#### Platform Service Team
```yaml
team_platform_developers:
  azure_ad_group: "OneStream-Platform-Developers"
  permissions:
    acr_dev_platform:
      - scope: "/subscriptions/{sub-id}/resourceGroups/dev-rg/providers/Microsoft.ContainerRegistry/registries/onestreamdev/repositories/onestream"
      - role: "AcrPull"
      - justification: "Pull platform images for local development"
    
    acr_dev_platform_push:
      - scope: "/subscriptions/{sub-id}/resourceGroups/dev-rg/providers/Microsoft.ContainerRegistry/registries/onestreamdev/repositories/onestream"  
      - role: "AcrPush"
      - justification: "Manual push for development testing only"
```

### 4. Administrative and Emergency Access

#### DevOps Team
```yaml
team_devops_engineers:
  azure_ad_group: "OneStream-DevOps-Engineers"
  permissions:
    acr_admin_dev:
      - scope: "/subscriptions/{sub-id}/resourceGroups/dev-rg/providers/Microsoft.ContainerRegistry/registries/onestreamdev"
      - role: "Contributor"
      - justification: "Full development ACR management"
    
    acr_reader_all:
      - scope: "/subscriptions/{sub-id}/resourceGroups/*/providers/Microsoft.ContainerRegistry/registries/*"
      - role: "Reader"
      - justification: "Monitor all ACR resources"
    
    devops_admin:
      - scope: "/OneStreamPlatformEng/OneStreamPlatform"
      - role: "Project Administrator"
      - justification: "Manage all pipelines and service connections"
  
  restrictions:
    - "Cannot push directly to staging or production ACRs"
    - "Cannot bypass promotion pipeline"
    - "Must use emergency access process for production"
```

#### Emergency Access (Break-Glass)
```yaml
emergency_access:
  group: "OneStream-Emergency-Access"
  activation: "Azure AD PIM (Just-In-Time)"
  duration: "4 hours maximum"
  approval_required: true
  approvers: ["CTO", "Head of Security", "DevOps Manager"]
  
  permissions:
    emergency_acr_admin:
      - scope: "/subscriptions/{sub-id}/resourceGroups/prod-rg/providers/Microsoft.ContainerRegistry/registries/onestreamprod"
      - role: "Contributor"
      - justification: "Emergency production ACR access"
      - conditions:
        - "Critical incident declared"
        - "Manager approval obtained"
        - "Incident ticket created"
  
  monitoring:
    - "All emergency access logged to SIEM"
    - "Automatic notification to security team"
    - "Post-incident review required"
    - "Automatic deactivation after time limit"
```

## Azure DevOps Service Connections

### Environment-Specific Service Connections

#### Development Service Connection
```yaml
sc_onestream_dev:
  name: "sc-onestream-dev-acr"
  type: "Azure Resource Manager"
  service_principal: "sp-onestream-dev-pipeline"
  subscription: "OneStream Development"
  resource_group: "dev-rg"
  
  security_settings:
    - "Restrict to specific pipelines: iam-service-pipeline"
    - "Require approval for new pipelines"
    - "Enable audit logging"
    - "Rotate credentials every 90 days"
  
  accessible_pipelines:
    - "iam-service-pipeline"
    - "utility-service-pipeline"
    - "dev-integration-tests"
```

#### Staging Service Connection
```yaml
sc_onestream_staging:
  name: "sc-onestream-staging-deploy"
  type: "Azure Resource Manager" 
  service_principal: "sp-onestream-staging-pipeline"
  subscription: "OneStream Staging"
  resource_group: "staging-rg"
  
  security_settings:
    - "Restrict to deployment pipelines only"
    - "Require manager approval"
    - "Enable detailed audit logging"
    - "Rotate credentials every 60 days"
  
  approval_settings:
    required_approvers: ["Staging Environment Owner"]
    timeout: "24 hours"
    auto_approve_on_timeout: false
```

#### Production Service Connection
```yaml
sc_onestream_production:
  name: "sc-onestream-production-deploy"
  type: "Azure Resource Manager"
  service_principal: "sp-onestream-production-pipeline"
  subscription: "OneStream Production"
  resource_group: "prod-rg"
  
  security_settings:
    - "Restrict to production deployment pipelines only"
    - "Require multiple approvals"
    - "Enable comprehensive audit logging"
    - "Rotate credentials every 30 days"
    - "Enable conditional access policies"
  
  approval_settings:
    required_approvers: 
      - "Production Environment Owner"
      - "Security Team Representative"
      - "Change Advisory Board"
    timeout: "48 hours"
    auto_approve_on_timeout: false
    notification_channels: ["security-alerts", "prod-deploy-alerts"]
```

#### Promotion Service Connection
```yaml
sc_onestream_promotion:
  name: "sc-onestream-cross-env-promotion"
  type: "Azure Resource Manager"
  service_principal: "sp-onestream-promotion-engine"
  subscription: "OneStream Shared Services"
  
  security_settings:
    - "Restrict to promotion pipeline only"
    - "Require security team approval"
    - "Enable comprehensive audit logging"
    - "Rotate credentials every 45 days"
  
  accessible_pipelines:
    - "promotion-dev-to-staging"
    - "promotion-staging-to-production"
    - "emergency-rollback-pipeline"
```

## Key Vault Access Control

### Signing Key Access
```yaml
kv_onestream_cosign:
  name: "onestream-cosign-keys"
  location: "shared-rg"
  
  access_policies:
    sp_promotion_engine:
      permissions:
        certificates: ["get", "list"]
        keys: ["get", "list", "sign", "verify"]
        secrets: []
      justification: "Verify signatures during promotion"
    
    sp_dev_pipeline:
      permissions:
        certificates: ["get"]
        keys: ["sign"]
        secrets: []
      justification: "Sign images during development builds"
    
    emergency_access_group:
      permissions:
        certificates: ["get", "list", "create", "update"]
        keys: ["get", "list", "create", "update", "sign", "verify"]
        secrets: []
      justification: "Emergency key management"
      activation: "PIM required"
```

### Application Secrets Access
```yaml
kv_onestream_app_secrets:
  environments:
    development:
      name: "onestream-dev-secrets"
      access_policies:
        sp_dev_pipeline:
          permissions:
            secrets: ["get", "list"]
          justification: "Access dev configuration secrets"
    
    staging:
      name: "onestream-staging-secrets"
      access_policies:
        sp_staging_pipeline:
          permissions:
            secrets: ["get"]
          justification: "Access staging configuration secrets"
    
    production:
      name: "onestream-prod-secrets"
      access_policies:
        sp_production_pipeline:
          permissions:
            secrets: ["get"]
          justification: "Access production configuration secrets"
        
        emergency_access_group:
          permissions:
            secrets: ["get", "list"]
          justification: "Emergency secret access"
          activation: "PIM required"
```

## Monitoring and Audit Requirements

### Access Monitoring
```yaml
monitoring_requirements:
  acr_access:
    - "Log all push/pull operations"
    - "Monitor cross-environment access patterns"
    - "Alert on unusual access times"
    - "Track image signature verification"
  
  service_principal_usage:
    - "Monitor all SP authentication events"
    - "Track permission escalation attempts"
    - "Alert on failed authentication"
    - "Audit credential rotation compliance"
  
  pipeline_execution:
    - "Log all pipeline service connection usage"
    - "Monitor deployment approvals"
    - "Track promotion pipeline execution"
    - "Alert on policy violations"
```

### Compliance Auditing
```yaml
audit_requirements:
  quarterly_reviews:
    - "Service principal permission audit"
    - "Azure AD group membership review"
    - "Service connection access review"
    - "Key Vault access policy review"
  
  automated_compliance:
    - "Daily service principal usage reports"
    - "Weekly access pattern analysis"
    - "Monthly permission drift detection"
    - "Continuous policy compliance monitoring"
  
  incident_response:
    - "Immediate access revocation capability"
    - "Emergency access activation logging"
    - "Breach response procedures"
    - "Post-incident access review"
```

## Implementation Checklist

### Phase 1: Service Principal Creation
- [ ] Create environment-specific service principals
- [ ] Assign minimal required permissions
- [ ] Configure credential rotation policies
- [ ] Implement monitoring and alerting

### Phase 2: Azure DevOps Configuration
- [ ] Create scoped service connections
- [ ] Configure approval workflows
- [ ] Implement pipeline restrictions
- [ ] Enable audit logging

### Phase 3: Key Vault Setup
- [ ] Create environment-specific key vaults
- [ ] Configure access policies
- [ ] Implement secret rotation
- [ ] Enable monitoring

### Phase 4: Monitoring Implementation
- [ ] Deploy access monitoring solution
- [ ] Configure compliance dashboards
- [ ] Implement automated alerting
- [ ] Create audit reporting

### Phase 5: Team Training
- [ ] Train teams on new access model
- [ ] Document emergency procedures
- [ ] Conduct access review training
- [ ] Implement change management process

## Benefits of Least Privilege Model

### Security Improvements
- **Reduced Attack Surface**: Limited blast radius for compromised credentials
- **Environment Isolation**: No cross-environment access without explicit promotion
- **Audit Clarity**: Clear permission boundaries and access patterns
- **Incident Containment**: Faster response and isolation capabilities

### Operational Benefits
- **Simplified Troubleshooting**: Clear permission boundaries aid debugging
- **Automated Compliance**: Built-in compliance monitoring and reporting
- **Scalable Access**: Easy to add new services with appropriate permissions
- **Reduced Manual Overhead**: Automated permission management and rotation

### Compliance Advantages
- **Clear Segregation**: Explicit environment and service boundaries
- **Comprehensive Auditing**: Complete access trail for compliance reporting
- **Risk Reduction**: Minimized privileged access exposure
- **Policy Enforcement**: Automated policy compliance monitoring