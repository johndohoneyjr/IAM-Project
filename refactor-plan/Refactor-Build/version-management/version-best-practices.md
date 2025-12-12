# Version Management Best Practices

## Overview

This document outlines Microsoft best practices for container image version management in the context of OneStream's simplified pipeline architecture. It addresses semantic versioning, immutable references, and environment promotion strategies that eliminate complexity while maintaining enterprise-grade controls.

## Microsoft Container Versioning Best Practices

### 1. Immutable Image References

Following [Microsoft's container tagging recommendations](https://learn.microsoft.com/en-us/azure/container-registry/container-registry-image-tag-version), we implement immutable digest-based references:

```yaml
# Microsoft Best Practice: Always use digests for production deployments
deployment_references:
  development:
    mutable_tags: "Allowed for rapid iteration"
    example: "onestreamdev.azurecr.io/iam:latest"
    use_case: "Development and testing"
  
  staging_production:
    immutable_digests: "Required for reproducible deployments"
    example: "onestreamprod.azurecr.io/iam@sha256:a1b2c3d4e5f6..."
    use_case: "Staging and production deployments"
```

### 2. Semantic Versioning Strategy

#### Application Version Schema
```yaml
semantic_versioning:
  format: "{major}.{minor}.{patch}-{build}.{commit}"
  examples:
    stable_release: "1.2.3-12345.a1b2c3d"
    pre_release: "1.3.0-beta.12346.b2c3d4e"
    hotfix: "1.2.4-hotfix.12347.c3d4e5f"
  
  components:
    major: "Breaking changes"
    minor: "New features, backward compatible"
    patch: "Bug fixes, backward compatible"
    build: "Azure DevOps build ID"
    commit: "Git commit short SHA"
```

#### Container Image Tagging
```yaml
container_tagging:
  unique_tags:
    format: "{service}-{version}-{environment}"
    examples:
      development: "iam-1.2.3-12345.a1b2c3d-dev"
      staging: "iam-1.2.3-12345.a1b2c3d-staging"
      production: "iam-1.2.3-12345.a1b2c3d-prod"
  
  immutable_digests:
    format: "{service}@sha256:{digest}"
    examples:
      all_environments: "iam@sha256:a1b2c3d4e5f6789..."
  
  benefits:
    - "Exact artifact traceability across environments"
    - "Guaranteed reproducible deployments"
    - "Supply chain security validation"
    - "Simplified rollback procedures"
```

### 3. Environment-Specific Versioning

#### Development Environment
```yaml
development_versioning:
  strategy: "Rapid iteration with mutable tags"
  
  tagging_approach:
    feature_branches:
      format: "{service}-{branch}-{build}"
      example: "iam-feature-auth-12345"
      retention: "7 days"
    
    main_branch:
      format: "{service}-main-{build}"
      example: "iam-main-12345"
      retention: "30 days"
    
    latest_tag:
      format: "{service}:latest"
      example: "iam:latest"
      purpose: "Always points to most recent successful build"
  
  promotion_criteria:
    - "All unit tests pass"
    - "Security scan clean"
    - "Build artifacts signed"
    - "Integration tests successful"
```

#### Staging Environment
```yaml
staging_versioning:
  strategy: "Stable versions with immutable references"
  
  tagging_approach:
    promoted_versions:
      format: "{service}-{version}-staging"
      example: "iam-1.2.3-12345.a1b2c3d-staging"
      source: "Promoted from development"
    
    digest_references:
      format: "{service}@{digest}"
      example: "iam@sha256:a1b2c3d4e5f6..."
      purpose: "Deployment references"
  
  promotion_criteria:
    - "Development validation complete"
    - "Security approval obtained"
    - "Change management approval"
    - "Staging-specific tests pass"
  
  retention_policy:
    successful_deployments: "90 days"
    failed_deployments: "30 days"
    auto_cleanup: "Enabled"
```

#### Production Environment
```yaml
production_versioning:
  strategy: "Immutable references with comprehensive validation"
  
  tagging_approach:
    released_versions:
      format: "{service}-{version}-prod"
      example: "iam-1.2.3-12345.a1b2c3d-prod"
      source: "Promoted from staging"
    
    digest_references:
      format: "{service}@{digest}"
      example: "iam@sha256:a1b2c3d4e5f6..."
      purpose: "All deployment references must use digests"
    
    locked_images:
      write_enabled: false
      purpose: "Prevent accidental deletion"
      unlock_process: "Emergency approval required"
  
  promotion_criteria:
    - "Staging validation complete"
    - "Business approval obtained"
    - "Security final approval"
    - "Change advisory board approval"
    - "Production readiness checklist complete"
  
  retention_policy:
    current_production: "Permanent"
    previous_versions: "1 year"
    rollback_candidates: "6 months"
```

## Version Lifecycle Management

### 1. Build-Time Version Generation

```yaml
# Pipeline template for version generation
version_generation:
  parameters:
    - name: serviceName
      type: string
    - name: buildReason
      type: string
  
  steps:
  - task: Bash@3
    displayName: 'Generate Version Information'
    name: version_info
    inputs:
      targetType: 'inline'
      script: |
        # Extract version from source
        APP_VERSION=$(cat version.txt || echo "0.1.0")
        
        # Build metadata
        BUILD_ID="$(Build.BuildId)"
        GIT_SHA=$(git rev-parse --short HEAD)
        BUILD_REASON="${{ parameters.buildReason }}"
        
        # Generate semantic version
        if [[ "$BUILD_REASON" == "PullRequest" ]]; then
          FULL_VERSION="$APP_VERSION-pr-$BUILD_ID.$GIT_SHA"
        elif [[ "$(Build.SourceBranch)" == "refs/heads/main" ]]; then
          FULL_VERSION="$APP_VERSION-$BUILD_ID.$GIT_SHA"
        else
          BRANCH_NAME=$(echo "$(Build.SourceBranch)" | sed 's/refs\/heads\///g' | sed 's/[^a-zA-Z0-9]/-/g')
          FULL_VERSION="$APP_VERSION-$BRANCH_NAME-$BUILD_ID.$GIT_SHA"
        fi
        
        # Container tag
        CONTAINER_TAG="${{ parameters.serviceName }}-$FULL_VERSION"
        
        # Output variables
        echo "##vso[task.setvariable variable=appVersion;isOutput=true]$APP_VERSION"
        echo "##vso[task.setvariable variable=fullVersion;isOutput=true]$FULL_VERSION"
        echo "##vso[task.setvariable variable=containerTag;isOutput=true]$CONTAINER_TAG"
        echo "##vso[task.setvariable variable=gitSha;isOutput=true]$GIT_SHA"
        
        echo "Generated version: $FULL_VERSION"
        echo "Container tag: $CONTAINER_TAG"
```

### 2. Digest Capture and Tracking

```yaml
# Template for capturing immutable digests
digest_capture:
  steps:
  - task: AzureCLI@2
    displayName: 'Build and Capture Digest'
    name: build_image
    inputs:
      scriptType: bash
      scriptLocation: inlineScript
      inlineScript: |
        # Build container with ACR
        BUILD_OUTPUT=$(az acr build \
          --registry $(acrName) \
          --image $(containerTag) \
          --file Dockerfile \
          . \
          --output json)
        
        # Extract digest from build output
        IMAGE_DIGEST=$(echo "$BUILD_OUTPUT" | jq -r '.outputImages[0].digest')
        
        if [[ -z "$IMAGE_DIGEST" || "$IMAGE_DIGEST" == "null" ]]; then
          echo "ERROR: Failed to capture image digest"
          exit 1
        fi
        
        # Output digest for subsequent stages
        echo "##vso[task.setvariable variable=imageDigest;isOutput=true]$IMAGE_DIGEST"
        echo "##vso[task.setvariable variable=digestReference;isOutput=true]$(acrName).azurecr.io/$(serviceName)@$IMAGE_DIGEST"
        
        echo "✅ Built image with digest: $IMAGE_DIGEST"
        echo "📌 Digest reference: $(acrName).azurecr.io/$(serviceName)@$IMAGE_DIGEST"
```

### 3. Cross-Environment Promotion

```yaml
# Template for promoting versions across environments
promotion_template:
  parameters:
    - name: sourceEnvironment
      type: string
    - name: targetEnvironment
      type: string
    - name: imageDigest
      type: string
    - name: serviceName
      type: string
  
  steps:
  - task: AzureCLI@2
    displayName: 'Verify Source Image'
    inputs:
      scriptType: bash
      scriptLocation: inlineScript
      inlineScript: |
        # Verify source image exists and is signed
        SOURCE_REF="${{ parameters.sourceEnvironment }}.azurecr.io/${{ parameters.serviceName }}@${{ parameters.imageDigest }}"
        
        # Check image exists
        az acr repository show \
          --name ${{ parameters.sourceEnvironment }} \
          --image ${{ parameters.serviceName }}@${{ parameters.imageDigest }}
        
        # Verify signature (if Cosign is enabled)
        if command -v cosign &> /dev/null; then
          cosign verify --key cosign.pub "$SOURCE_REF"
          echo "✅ Image signature verified"
        fi
  
  - task: AzureCLI@2
    displayName: 'Promote to Target Environment'
    name: promote_image
    inputs:
      scriptType: bash
      scriptLocation: inlineScript
      inlineScript: |
        # Import image preserving digest
        SOURCE_REF="${{ parameters.sourceEnvironment }}.azurecr.io/${{ parameters.serviceName }}@${{ parameters.imageDigest }}"
        TARGET_REF="${{ parameters.serviceName }}@${{ parameters.imageDigest }}"
        
        az acr import \
          --name ${{ parameters.targetEnvironment }} \
          --source "$SOURCE_REF" \
          --image "$TARGET_REF" \
          --force
        
        # Verify import preserved digest
        IMPORTED_DIGEST=$(az acr repository show \
          --name ${{ parameters.targetEnvironment }} \
          --image "$TARGET_REF" \
          --query "digest" -o tsv)
        
        if [[ "$IMPORTED_DIGEST" != "${{ parameters.imageDigest }}" ]]; then
          echo "ERROR: Digest mismatch after import"
          echo "Expected: ${{ parameters.imageDigest }}"
          echo "Actual: $IMPORTED_DIGEST"
          exit 1
        fi
        
        # Output target reference
        TARGET_FULL_REF="${{ parameters.targetEnvironment }}.azurecr.io/${{ parameters.serviceName }}@${{ parameters.imageDigest }}"
        echo "##vso[task.setvariable variable=targetReference;isOutput=true]$TARGET_FULL_REF"
        
        echo "✅ Successfully promoted image"
        echo "📌 Target reference: $TARGET_FULL_REF"
```

## Version Governance and Compliance

### 1. Release Approval Gates

```yaml
# Version-based approval process
approval_gates:
  development_to_staging:
    requirements:
      - "All automated tests pass"
      - "Security scan results acceptable"
      - "Code review completed"
      - "Integration tests successful"
    
    approval_process:
      auto_approve: true
      conditions:
        - "No critical vulnerabilities"
        - "All tests green"
        - "Build artifacts signed"
  
  staging_to_production:
    requirements:
      - "Staging validation complete"
      - "Business stakeholder approval"
      - "Security team approval"
      - "Change management process"
    
    approval_process:
      auto_approve: false
      required_approvers:
        - "Product Owner"
        - "Security Representative"
        - "Operations Manager"
      timeout: "48 hours"
      escalation: "Auto-reject if timeout"
```

### 2. Version Auditing and Compliance

```yaml
# Audit trail requirements
audit_requirements:
  version_tracking:
    - "All version changes logged"
    - "Promotion path documented"
    - "Approval decisions recorded"
    - "Deployment outcomes tracked"
  
  compliance_reporting:
    - "Monthly version compliance report"
    - "Security patch tracking"
    - "Rollback event documentation"
    - "Access audit for version management"
  
  retention_policies:
    audit_logs: "7 years"
    version_metadata: "3 years"
    promotion_history: "2 years"
    approval_records: "7 years"
```

### 3. Automated Version Cleanup

```yaml
# ACR lifecycle management
acr_lifecycle_policies:
  development:
    untagged_retention: "7 days"
    feature_branch_retention: "14 days"
    main_branch_retention: "30 days"
    auto_cleanup: true
  
  staging:
    untagged_retention: "30 days"
    failed_deployment_retention: "30 days"
    successful_deployment_retention: "90 days"
    auto_cleanup: true
  
  production:
    untagged_retention: "90 days"
    previous_version_retention: "1 year"
    current_version_retention: "permanent"
    auto_cleanup: false  # Manual approval required
    
    cleanup_approval:
      required_approvers: ["Operations Manager", "Security Team"]
      notification_period: "30 days"
      escalation_process: "Change Advisory Board"
```

## Rollback and Recovery Procedures

### 1. Version-Based Rollback

```yaml
# Rollback procedure using immutable versions
rollback_process:
  identification:
    - "Identify target rollback version"
    - "Verify version availability in target environment"
    - "Confirm version compatibility"
    - "Check rollback authorization"
  
  execution:
    - "Update deployment manifests with previous digest"
    - "Apply configuration changes"
    - "Verify service health"
    - "Validate rollback success"
  
  example_commands:
    identify_previous: |
      # List recent deployments
      kubectl get deployments iam-service -o yaml | grep image:
      
      # Find previous working version
      az acr repository show-tags \
        --name onestreamprod \
        --repository iam \
        --orderby time_desc \
        --top 10
    
    execute_rollback: |
      # Update deployment with previous digest
      kubectl set image deployment/iam-service \
        iam-container=onestreamprod.azurecr.io/iam@sha256:previous-digest
      
      # Monitor rollout
      kubectl rollout status deployment/iam-service
```

### 2. Emergency Version Procedures

```yaml
# Emergency version management
emergency_procedures:
  hotfix_deployment:
    triggers:
      - "Critical security vulnerability"
      - "Production service outage"
      - "Data integrity issue"
    
    process:
      - "Declare incident"
      - "Activate emergency access"
      - "Fast-track version approval"
      - "Deploy with reduced validation"
      - "Monitor and validate"
      - "Conduct post-incident review"
  
  emergency_rollback:
    triggers:
      - "Failed deployment"
      - "Service degradation"
      - "Security incident"
    
    process:
      - "Immediate rollback authorization"
      - "Bypass normal approval gates"
      - "Execute automated rollback"
      - "Verify service restoration"
      - "Document incident details"
```

## Implementation Guidelines

### 1. Migration from Current System

```yaml
migration_strategy:
  phase_1: "Parallel implementation"
    - "Implement new versioning alongside existing"
    - "Validate version parity"
    - "Train teams on new processes"
  
  phase_2: "Gradual cutover"
    - "Migrate development environment first"
    - "Move staging after validation"
    - "Final production migration"
  
  phase_3: "Legacy cleanup"
    - "Remove old versioning systems"
    - "Update documentation"
    - "Finalize team training"
```

### 2. Team Training Requirements

```yaml
training_requirements:
  developers:
    - "Semantic versioning principles"
    - "Container digest concepts"
    - "Development workflow changes"
  
  devops_engineers:
    - "ACR lifecycle management"
    - "Promotion pipeline configuration"
    - "Emergency procedures"
  
  operations_team:
    - "Version-based deployment procedures"
    - "Rollback processes"
    - "Monitoring and alerting"
```

## Benefits of New Version Management

### Security Benefits
- **Immutable References**: Prevent tampering with deployed artifacts
- **Supply Chain Security**: Complete provenance tracking from build to deployment
- **Signature Verification**: Cryptographic validation at every promotion step
- **Audit Trail**: Complete version history for compliance

### Operational Benefits
- **Simplified Rollbacks**: Exact version identification and restoration
- **Faster Promotions**: Direct ACR imports vs complex blob operations
- **Better Debugging**: Clear version correlation between environments
- **Reduced Complexity**: Fewer moving parts in version management

### Compliance Benefits
- **Regulatory Compliance**: Clear audit trail for all version changes
- **Change Management**: Structured approval process for version promotions
- **Risk Reduction**: Immutable references reduce deployment risks
- **Documentation**: Automated version tracking and reporting