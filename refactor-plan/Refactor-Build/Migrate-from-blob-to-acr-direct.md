OneStream's current blob storage approach, can be **completely eliminate the blob storage step** by leveraging immutable digests and direct ACR-to-ACR transfers. Here's how:

## Current vs. Proposed Architecture

### **Current State (from diagram):**
```
Build → package-build-data → Blob Storage → Promote to ACR environments
```

### **Proposed State:**
```
Build → Dev ACR (with digest) → Direct ACR import → Staging/Prod ACR
```

## Elimination Strategy

### **1. Replace Blob Storage with Digest-Based Promotion**

Instead of storing build artifacts in blob storage, capture the digest immediately and use it for direct ACR transfers:

```yaml
# Enhanced build pipeline (replaces package-build-data step)
- task: AzureCLI@2
  displayName: 'Build and Capture Digest'
  name: 'buildCapture'
  inputs:
    scriptType: 'bash'
    inlineScript: |
      # Build directly in dev ACR
      az acr build \
        --registry onestreamdev \
        --image $(SERVICE_NAME):build-$(Build.BuildId) \
        .
      
      # Immediately capture digest (eliminates need for blob storage)
      DIGEST=$(az acr repository show \
        --name onestreamdev \
        --image $(SERVICE_NAME):build-$(Build.BuildId) \
        --query "digest" -o tsv)
      
      # Create promotion manifest (replaces blob artifacts)
      cat > promotion-manifest.json << EOF
      {
        "service": "$(SERVICE_NAME)",
        "buildId": "$(Build.BuildId)",
        "digest": "$DIGEST",
        "sourceImage": "onestreamdev.azurecr.io/$(SERVICE_NAME)@$DIGEST",
        "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
        "commit": "$(Build.SourceVersion)",
        "branch": "$(Build.SourceBranchName)"
      }
      EOF
      
      # Store promotion manifest as pipeline artifact (lightweight metadata)
      echo "##vso[task.setvariable;variable=IMAGE_DIGEST;isOutput=true]$DIGEST"
      echo "##vso[task.setvariable;variable=PROMOTION_MANIFEST;isOutput=true]$(cat promotion-manifest.json)"
      
      # Optional: Add human-readable tags for operational convenience
      az acr import \
        --name onestreamdev \
        --source onestreamdev.azurecr.io/$(SERVICE_NAME)@$DIGEST \
        --image $(SERVICE_NAME):latest \
        --image $(SERVICE_NAME):$(Build.BuildNumber)
```

### **2. Direct ACR-to-ACR Promotion (No Blob Storage)**

```yaml
# promotion-pipeline.yml (replaces blob storage promotion)
parameters:
- name: promotionManifest
  type: string
- name: targetEnvironment
  type: string  # staging, prod

stages:
- stage: DirectACRPromotion
  displayName: 'Direct ACR Promotion to $(targetEnvironment)'
  jobs:
  - job: PromoteByDigest
    steps:
    - task: AzureCLI@2
      displayName: 'Direct ACR Import'
      inputs:
        scriptType: 'bash'
        inlineScript: |
          # Parse promotion manifest
          MANIFEST='${{ parameters.promotionManifest }}'
          SERVICE=$(echo $MANIFEST | jq -r '.service')
          DIGEST=$(echo $MANIFEST | jq -r '.digest')
          SOURCE_IMAGE=$(echo $MANIFEST | jq -r '.sourceImage')
          BUILD_ID=$(echo $MANIFEST | jq -r '.buildId')
          
          TARGET_ACR="onestream${{ parameters.targetEnvironment }}"
          
          echo "Promoting $SERVICE@$DIGEST to $TARGET_ACR"
          
          # Direct ACR-to-ACR transfer (no blob storage involved)
          az acr import \
            --name $TARGET_ACR \
            --source $SOURCE_IMAGE \
            --image $SERVICE@$DIGEST \
            --image $SERVICE:promoted-$(date +%Y%m%d-%H%M%S) \
            --image $SERVICE:from-build-$BUILD_ID
          
          # Verify promotion
          az acr repository show \
            --name $TARGET_ACR \
            --image $SERVICE@$DIGEST \
            --query "digest" -o tsv
          
          echo "✅ Successfully promoted $SERVICE@$DIGEST to $TARGET_ACR"
```

### **3. Multi-Service Batch Promotion**

Since you mentioned eliminating the current "build folders" approach, here's how to handle multiple services:

```yaml
# batch-promotion.yml (replaces blob storage build folders)
parameters:
- name: servicesManifest
  type: object  # Array of promotion manifests
- name: targetEnvironment
  type: string

stages:
- stage: BatchPromotion
  displayName: 'Batch Promote All Services'
  jobs:
  - job: PromoteAllServices
    steps:
    - task: AzureCLI@2
      displayName: 'Batch ACR Import'
      inputs:
        scriptType: 'bash'
        inlineScript: |
          TARGET_ACR="onestream${{ parameters.targetEnvironment }}"
          
          # Process each service manifest
          echo '${{ convertToJson(parameters.servicesManifest) }}' | jq -c '.[]' | while read -r manifest; do
            SERVICE=$(echo $manifest | jq -r '.service')
            DIGEST=$(echo $manifest | jq -r '.digest')
            SOURCE_IMAGE=$(echo $manifest | jq -r '.sourceImage')
            
            echo "Promoting $SERVICE@$DIGEST..."
            
            # Direct import (no blob storage)
            az acr import \
              --name $TARGET_ACR \
              --source $SOURCE_IMAGE \
              --image $SERVICE@$DIGEST \
              --image $SERVICE:batch-promoted-$(date +%Y%m%d) &
            
            # Run imports in parallel for efficiency
          done
          
          # Wait for all imports to complete
          wait
          
          echo "✅ Batch promotion complete to $TARGET_ACR"
```

## Benefits of Eliminating Blob Storage

### **1. Simplified Architecture**
- **Remove**: Blob storage accounts, storage management, lifecycle policies
- **Reduce**: Infrastructure overhead, storage costs, complexity
- **Simplify**: Only ACRs to manage, standard Azure container registry operations

### **2. Faster Promotion Process**
```bash
# Current: Build → Blob → Download → Push to ACR
# Time: ~10-15 minutes per service

# New: Build → Direct ACR import
# Time: ~2-3 minutes per service (native Azure network transfer)
```

### **3. Improved Security and Compliance**
```yaml
# No intermediate storage security concerns
# Direct ACR-to-ACR transfers:
- Encrypted in transit (TLS)
- Audit logs in Azure Activity Logs
- RBAC controls on ACR resources
- No blob storage access keys to manage
```

### **4. Cost Optimization**
```yaml
# Eliminated costs:
- Blob storage accounts
- Storage transactions
- Egress from blob to ACR
- Storage management overhead

# Reduced costs:
- Faster promotion = less pipeline time
- No duplicate storage (blob + ACR)
```

## Implementation Plan

### **Phase 1: Parallel Implementation**
```yaml
# Keep current blob storage, add digest capture
- stage: Build
  jobs:
  - job: BuildWithBoth
    steps:
    - template: current-package-build-data.yml  # Keep existing
    - template: new-digest-capture.yml          # Add digest approach
```

### **Phase 2: Direct ACR Validation**
```yaml
# Test direct ACR promotion alongside blob storage
- stage: ValidationPromotion
  jobs:
  - job: TestDirectACR
    steps:
    - template: direct-acr-promotion.yml
    - template: validate-against-blob-promotion.yml
```

### **Phase 3: Cutover**
```yaml
# Remove blob storage steps entirely
- stage: Build
  jobs:
  - job: DigestOnlyBuild
    steps:
    - template: digest-capture-only.yml
    
- stage: Promote
  jobs:
  - job: DirectACROnly
    steps:
    - template: direct-acr-promotion.yml
```

## Migration Script Example

```bash
#!/bin/bash
# migrate-from-blob-to-acr.sh

# For each service currently in blob storage
for service in iam-service auth-service; do
  echo "Migrating $service from blob storage to ACR-direct..."
  
  # Find latest build in blob storage
  LATEST_BUILD=$(az storage blob list \
    --container-name builds \
    --prefix $service/ \
    --query "sort_by([], &properties.lastModified)[-1].name" -o tsv)
  
  if [[ -n "$LATEST_BUILD" ]]; then
    # Download and re-push to dev ACR to get digest
    az storage blob download \
      --container-name builds \
      --name $LATEST_BUILD \
      --file /tmp/$service.tar
    
    # Import to dev ACR
    az acr import \
      --name onestreamdev \
      --source /tmp/$service.tar \
      --image $service:migrated-from-blob
    
    # Get digest for future promotions
    DIGEST=$(az acr repository show \
      --name onestreamdev \
      --image $service:migrated-from-blob \
      --query "digest" -o tsv)
    
    echo "✅ $service migrated with digest: $DIGEST"
    
    # Clean up temp file
    rm /tmp/$service.tar
  fi
done
```

## Summary

By eliminating blob storage and using immutable digests with direct ACR-to-ACR transfers, OneStream achieves:

1. **Simplified architecture** - Remove blob storage layer entirely
2. **Faster promotions** - Direct ACR imports vs. blob download/upload
3. **Lower costs** - No blob storage, faster pipelines
4. **Better security** - Fewer storage endpoints to secure
5. **Easier operations** - Standard ACR operations only
6. **Immutable deployments** - Digest-based references ensure content integrity

The key insight is that **ACR itself becomes your artifact store** - you don't need blob storage when you have immutable digests for exact content identification and ACR import for efficient cross-registry transfers.