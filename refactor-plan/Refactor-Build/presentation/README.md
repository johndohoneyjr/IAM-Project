# OneStream Pipeline Refactoring: Parallel Environment POC Strategy

## Overview

This document outlines the comprehensive approach for setting up a parallel environment to proof-of-concept (POC) the pipeline improvements for OneStream's IAM service, while maintaining existing production workflows during validation.

## 1. Parallel Environment Setup for IAM POC

### 1.1 Environment Architecture

#### POC Resource Structure
```yaml
parallel_environment:
  resource_groups:
    - name: "onestream-poc-dev-rg"
      location: "East US 2"
      purpose: "POC development environment"
    
    - name: "onestream-poc-staging-rg"
      location: "East US 2"
      purpose: "POC staging environment"
    
    - name: "onestream-poc-shared-rg"
      location: "East US 2"
      purpose: "Shared services (Key Vault, monitoring)"
  
  container_registries:
    - name: "onestreampocdev"
      tier: "Premium"
      purpose: "Development container images"
      retention: "30 days"
    
    - name: "onestreampocstaging"
      tier: "Premium"
      purpose: "Staging container images"
      retention: "90 days"
  
  key_vaults:
    - name: "onestream-poc-cosign-kv"
      purpose: "Cosign signing keys"
      access: "Service principals only"
    
    - name: "onestream-poc-secrets-kv"
      purpose: "Application secrets and configuration"
      access: "Environment-specific access"
```

#### Network Isolation
```yaml
network_setup:
  virtual_network:
    name: "onestream-poc-vnet"
    address_space: "10.100.0.0/16"
    
  subnets:
    - name: "poc-aks-subnet"
      address_prefix: "10.100.1.0/24"
      purpose: "AKS cluster nodes"
    
    - name: "poc-pipeline-subnet"
      address_prefix: "10.100.2.0/24"
      purpose: "Azure DevOps agents"
    
    - name: "poc-acr-subnet"
      address_prefix: "10.100.3.0/24"
      purpose: "Container registry access"
  
  network_security:
    - "Isolated from production networks"
    - "Controlled access to production data"
    - "Dedicated DNS resolution"
    - "Separate monitoring and logging"
```

### 1.2 Infrastructure Deployment Scripts

#### POC Environment Creation
```bash
#!/bin/bash
# setup-poc-environment.sh

set -e

# Configuration
SUBSCRIPTION_ID="your-poc-subscription-id"
LOCATION="East US 2"
POC_PREFIX="onestream-poc"
RESOURCE_GROUPS=("${POC_PREFIX}-dev-rg" "${POC_PREFIX}-staging-rg" "${POC_PREFIX}-shared-rg")

echo "🚀 Setting up OneStream POC Environment"

# Login and set subscription
az login
az account set --subscription "$SUBSCRIPTION_ID"

# Create resource groups
for RG in "${RESOURCE_GROUPS[@]}"; do
    echo "Creating resource group: $RG"
    az group create \
        --name "$RG" \
        --location "$LOCATION" \
        --tags Project=OneStreamPOC Environment=POC Purpose=PipelineRefactoring
done

# Create virtual network
echo "Creating POC virtual network"
az network vnet create \
    --resource-group "${POC_PREFIX}-shared-rg" \
    --name "${POC_PREFIX}-vnet" \
    --address-prefix 10.100.0.0/16 \
    --subnet-name "${POC_PREFIX}-aks-subnet" \
    --subnet-prefix 10.100.1.0/24

# Create additional subnets
az network vnet subnet create \
    --resource-group "${POC_PREFIX}-shared-rg" \
    --vnet-name "${POC_PREFIX}-vnet" \
    --name "${POC_PREFIX}-pipeline-subnet" \
    --address-prefix 10.100.2.0/24

az network vnet subnet create \
    --resource-group "${POC_PREFIX}-shared-rg" \
    --vnet-name "${POC_PREFIX}-vnet" \
    --name "${POC_PREFIX}-acr-subnet" \
    --address-prefix 10.100.3.0/24

# Create Container Registries
echo "Creating POC Container Registries"
for ENV in "dev" "staging"; do
    ACR_NAME="${POC_PREFIX}${ENV}"
    RG_NAME="${POC_PREFIX}-${ENV}-rg"
    
    az acr create \
        --resource-group "$RG_NAME" \
        --name "$ACR_NAME" \
        --sku Premium \
        --admin-enabled false \
        --public-network-enabled true \
        --allow-trusted-services true
    
    # Enable content trust and configure retention
    az acr config content-trust update --registry "$ACR_NAME" --status enabled
    az acr config retention update --registry "$ACR_NAME" --status enabled --days 30 --type UntaggedManifests
done

# Create Key Vaults
echo "Creating POC Key Vaults"
az keyvault create \
    --name "${POC_PREFIX}-cosign-kv" \
    --resource-group "${POC_PREFIX}-shared-rg" \
    --location "$LOCATION" \
    --sku premium \
    --enable-rbac-authorization true

az keyvault create \
    --name "${POC_PREFIX}-secrets-kv" \
    --resource-group "${POC_PREFIX}-shared-rg" \
    --location "$LOCATION" \
    --sku standard \
    --enable-rbac-authorization true

# Create AKS clusters for testing
echo "Creating POC AKS clusters"
for ENV in "dev" "staging"; do
    CLUSTER_NAME="${POC_PREFIX}-${ENV}-aks"
    RG_NAME="${POC_PREFIX}-${ENV}-rg"
    
    az aks create \
        --resource-group "$RG_NAME" \
        --name "$CLUSTER_NAME" \
        --node-count 2 \
        --node-vm-size Standard_D2s_v3 \
        --network-plugin azure \
        --vnet-subnet-id "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/${POC_PREFIX}-shared-rg/providers/Microsoft.Network/virtualNetworks/${POC_PREFIX}-vnet/subnets/${POC_PREFIX}-aks-subnet" \
        --enable-managed-identity \
        --attach-acr "${POC_PREFIX}${ENV}"
done

echo "✅ POC Environment setup complete"
echo "📋 Next steps:"
echo "   1. Configure service principals"
echo "   2. Set up Azure DevOps project"
echo "   3. Deploy pipeline templates"
```

## 2. Entra ID Service Principals and Least Privilege Roles

### 2.1 Service Principal Architecture

#### POC-Specific Service Principals
```yaml
service_principals:
  development:
    sp_poc_dev_pipeline:
      name: "sp-onestream-poc-dev-pipeline"
      purpose: "POC development builds and deployments"
      scopes:
        - "/subscriptions/{sub-id}/resourceGroups/onestream-poc-dev-rg"
      roles:
        - "AcrPush" # Push to dev ACR
        - "AcrPull" # Pull from dev ACR
        - "Azure Kubernetes Service Cluster User Role" # Deploy to dev AKS
      key_vault_access:
        - vault: "onestream-poc-cosign-kv"
          permissions: ["keys/sign"]
        - vault: "onestream-poc-secrets-kv"
          permissions: ["secrets/get"]
  
  staging:
    sp_poc_staging_pipeline:
      name: "sp-onestream-poc-staging-pipeline"
      purpose: "POC staging deployments only"
      scopes:
        - "/subscriptions/{sub-id}/resourceGroups/onestream-poc-staging-rg"
      roles:
        - "AcrPull" # Pull from staging ACR only
        - "Azure Kubernetes Service Cluster User Role" # Deploy to staging AKS
      restrictions:
        - "Cannot push to any ACR"
        - "Cannot access development resources"
  
  promotion:
    sp_poc_promotion_engine:
      name: "sp-onestream-poc-promotion-engine"
      purpose: "Cross-environment image promotions"
      scopes:
        - "/subscriptions/{sub-id}/resourceGroups/onestream-poc-dev-rg/providers/Microsoft.ContainerRegistry/registries/onestreampocdex"
        - "/subscriptions/{sub-id}/resourceGroups/onestream-poc-staging-rg/providers/Microsoft.ContainerRegistry/registries/onestreampocstaging"
      roles:
        - "AcrPull" # Pull from source ACRs
        - "AcrImportImage" # Import to target ACRs
      key_vault_access:
        - vault: "onestream-poc-cosign-kv"
          permissions: ["keys/verify"]
```

### 2.2 Service Principal Creation Script

```bash
#!/bin/bash
# create-poc-service-principals.sh

set -e

SUBSCRIPTION_ID="your-poc-subscription-id"
POC_PREFIX="onestream-poc"

echo "🔐 Creating POC Service Principals with Least Privilege Access"

# Development Pipeline SP
echo "Creating development pipeline service principal"
DEV_SP=$(az ad sp create-for-rbac \
    --name "sp-${POC_PREFIX}-dev-pipeline" \
    --skip-assignment \
    --output json)

DEV_SP_ID=$(echo $DEV_SP | jq -r '.appId')
DEV_SP_SECRET=$(echo $DEV_SP | jq -r '.password')

# Assign specific permissions to dev SP
az role assignment create \
    --assignee $DEV_SP_ID \
    --role "AcrPush" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/${POC_PREFIX}-dev-rg/providers/Microsoft.ContainerRegistry/registries/${POC_PREFIX}dev"

az role assignment create \
    --assignee $DEV_SP_ID \
    --role "Azure Kubernetes Service Cluster User Role" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/${POC_PREFIX}-dev-rg/providers/Microsoft.ContainerService/managedClusters/${POC_PREFIX}-dev-aks"

# Staging Pipeline SP
echo "Creating staging pipeline service principal"
STAGING_SP=$(az ad sp create-for-rbac \
    --name "sp-${POC_PREFIX}-staging-pipeline" \
    --skip-assignment \
    --output json)

STAGING_SP_ID=$(echo $STAGING_SP | jq -r '.appId')

# Assign staging permissions (read-only)
az role assignment create \
    --assignee $STAGING_SP_ID \
    --role "AcrPull" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/${POC_PREFIX}-staging-rg/providers/Microsoft.ContainerRegistry/registries/${POC_PREFIX}staging"

az role assignment create \
    --assignee $STAGING_SP_ID \
    --role "Azure Kubernetes Service Cluster User Role" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/${POC_PREFIX}-staging-rg/providers/Microsoft.ContainerService/managedClusters/${POC_PREFIX}-staging-aks"

# Promotion Engine SP
echo "Creating promotion engine service principal"
PROMOTION_SP=$(az ad sp create-for-rbac \
    --name "sp-${POC_PREFIX}-promotion-engine" \
    --skip-assignment \
    --output json)

PROMOTION_SP_ID=$(echo $PROMOTION_SP | jq -r '.appId')

# Cross-environment promotion permissions
az role assignment create \
    --assignee $PROMOTION_SP_ID \
    --role "AcrPull" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/${POC_PREFIX}-dev-rg/providers/Microsoft.ContainerRegistry/registries/${POC_PREFIX}dev"

az role assignment create \
    --assignee $PROMOTION_SP_ID \
    --role "AcrImportImage" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/${POC_PREFIX}-staging-rg/providers/Microsoft.ContainerRegistry/registries/${POC_PREFIX}staging"

# Key Vault permissions
az role assignment create \
    --assignee $DEV_SP_ID \
    --role "Key Vault Crypto User" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/${POC_PREFIX}-shared-rg/providers/Microsoft.KeyVault/vaults/${POC_PREFIX}-cosign-kv"

az role assignment create \
    --assignee $PROMOTION_SP_ID \
    --role "Key Vault Crypto User" \
    --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/${POC_PREFIX}-shared-rg/providers/Microsoft.KeyVault/vaults/${POC_PREFIX}-cosign-kv"

# Store credentials in Key Vault for secure access
az keyvault secret set \
    --vault-name "${POC_PREFIX}-secrets-kv" \
    --name "dev-pipeline-sp-id" \
    --value "$DEV_SP_ID"

az keyvault secret set \
    --vault-name "${POC_PREFIX}-secrets-kv" \
    --name "dev-pipeline-sp-secret" \
    --value "$DEV_SP_SECRET"

echo "✅ Service principals created with least privilege access"
echo "📝 Credentials stored securely in Key Vault"
```

### 2.3 Role-Based Access Control Matrix

```yaml
rbac_matrix:
  resources:
    dev_acr:
      sp_poc_dev_pipeline: ["AcrPush", "AcrPull"]
      sp_poc_promotion_engine: ["AcrPull"]
      sp_poc_staging_pipeline: [] # No access
    
    staging_acr:
      sp_poc_dev_pipeline: [] # No access
      sp_poc_promotion_engine: ["AcrImportImage", "AcrPull"]
      sp_poc_staging_pipeline: ["AcrPull"]
    
    dev_aks:
      sp_poc_dev_pipeline: ["Azure Kubernetes Service Cluster User Role"]
      sp_poc_promotion_engine: [] # No access
      sp_poc_staging_pipeline: [] # No access
    
    staging_aks:
      sp_poc_dev_pipeline: [] # No access
      sp_poc_promotion_engine: [] # No access
      sp_poc_staging_pipeline: ["Azure Kubernetes Service Cluster User Role"]
    
    cosign_key_vault:
      sp_poc_dev_pipeline: ["Key Vault Crypto User"]
      sp_poc_promotion_engine: ["Key Vault Crypto User"]
      sp_poc_staging_pipeline: [] # No access
```

## 3. Simplified Azure DevOps POC Project

### 3.1 Project Structure

#### POC Project Setup
```yaml
azdo_poc_project:
  name: "OneStream-Pipeline-POC"
  description: "Proof of concept for pipeline refactoring"
  visibility: "Private"
  
  repositories:
    - name: "iam-service-poc"
      purpose: "IAM service source code for testing"
      branch_policies:
        - "Require pull request reviews"
        - "Require build validation"
    
    - name: "poc-pipeline-templates"
      purpose: "Reusable pipeline templates"
      structure:
        - "templates/build/"
        - "templates/deploy/"
        - "templates/promote/"
  
  pipelines:
    - name: "iam-poc-build-pipeline"
      purpose: "Build and test IAM service"
      trigger: "CI on main and feature branches"
    
    - name: "iam-poc-promotion-pipeline"
      purpose: "Promote images between environments"
      trigger: "Manual with approvals"
    
    - name: "iam-poc-deploy-pipeline"
      purpose: "Deploy to target environments"
      trigger: "After successful promotion"
```

### 3.2 Service Connection Configuration

```yaml
service_connections:
  poc_dev_connection:
    name: "sc-onestream-poc-dev"
    type: "Azure Resource Manager"
    service_principal: "sp-onestream-poc-dev-pipeline"
    scope: "onestream-poc-dev-rg"
    restrictions:
      - "Limited to POC development pipelines"
      - "Automatic credential rotation every 90 days"
  
  poc_staging_connection:
    name: "sc-onestream-poc-staging"
    type: "Azure Resource Manager"
    service_principal: "sp-onestream-poc-staging-pipeline"
    scope: "onestream-poc-staging-rg"
    restrictions:
      - "Limited to POC staging pipelines"
      - "Require approval for pipeline access"
  
  poc_promotion_connection:
    name: "sc-onestream-poc-promotion"
    type: "Azure Resource Manager"
    service_principal: "sp-onestream-poc-promotion-engine"
    scope: "Cross-environment promotion only"
    restrictions:
      - "Limited to promotion pipelines only"
      - "Require security team approval"
```

### 3.3 POC Pipeline Templates

#### Build Pipeline Template
```yaml
# poc-templates/iam-build-template.yml
parameters:
- name: serviceName
  type: string
  default: 'iam'
- name: environment
  type: string
  default: 'dev'

variables:
- group: onestream-poc-dev-variables

stages:
- stage: Build
  displayName: 'Build IAM POC Service'
  jobs:
  - job: BuildAndTest
    displayName: 'Build, Test, and Sign'
    pool:
      vmImage: 'ubuntu-latest'
    
    steps:
    # Version generation
    - task: Bash@3
      displayName: 'Generate Version'
      name: version_info
      inputs:
        targetType: 'inline'
        script: |
          # Generate POC version
          POC_VERSION="poc-$(Build.BuildId)-$(echo $(Build.SourceVersion) | cut -c1-7)"
          echo "##vso[task.setvariable variable=pocVersion;isOutput=true]$POC_VERSION"
          echo "POC Version: $POC_VERSION"
    
    # Build container
    - task: AzureCLI@2
      displayName: 'Build Container with Digest'
      name: build_container
      inputs:
        azureSubscription: 'sc-onestream-poc-dev'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          # Build with POC tag
          CONTAINER_TAG="${{ parameters.serviceName }}:$(version_info.pocVersion)"
          
          echo "Building POC container: $CONTAINER_TAG"
          
          # Build and capture digest
          BUILD_OUTPUT=$(az acr build \
            --registry onestreampocdex \
            --image "$CONTAINER_TAG" \
            --file Dockerfile \
            . \
            --output json)
          
          # Extract digest
          IMAGE_DIGEST=$(echo "$BUILD_OUTPUT" | jq -r '.outputImages[0].digest')
          
          if [[ -z "$IMAGE_DIGEST" || "$IMAGE_DIGEST" == "null" ]]; then
            echo "ERROR: Failed to capture image digest"
            exit 1
          fi
          
          # Output for subsequent stages
          echo "##vso[task.setvariable variable=imageDigest;isOutput=true]$IMAGE_DIGEST"
          echo "##vso[task.setvariable variable=digestReference;isOutput=true]onestreampocdex.azurecr.io/${{ parameters.serviceName }}@$IMAGE_DIGEST"
          
          echo "✅ Built POC image with digest: $IMAGE_DIGEST"
    
    # Sign with Cosign
    - task: AzureCLI@2
      displayName: 'Sign Container with Cosign'
      inputs:
        azureSubscription: 'sc-onestream-poc-dev'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          # Install cosign
          curl -O -L "https://github.com/sigstore/cosign/releases/latest/download/cosign-linux-amd64"
          sudo mv cosign-linux-amd64 /usr/local/bin/cosign
          sudo chmod +x /usr/local/bin/cosign
          
          # Login to ACR
          az acr login --name onestreampocdex
          
          # Sign image with Azure Key Vault
          IMAGE_REF="$(build_container.digestReference)"
          
          cosign sign \
            --azure-kv-name onestream-poc-cosign-kv \
            --azure-kv-key cosign-signing-key \
            "$IMAGE_REF"
          
          echo "✅ Signed POC image: $IMAGE_REF"
    
    # Security scan
    - task: AzureCLI@2
      displayName: 'Security Scan with Trivy'
      inputs:
        azureSubscription: 'sc-onestream-poc-dev'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          # Install trivy
          sudo apt-get update
          sudo apt-get install wget apt-transport-https gnupg lsb-release
          wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
          echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | sudo tee -a /etc/apt/sources.list.d/trivy.list
          sudo apt-get update
          sudo apt-get install trivy
          
          # Scan image
          IMAGE_REF="$(build_container.digestReference)"
          
          echo "Scanning POC image: $IMAGE_REF"
          trivy image --exit-code 1 --severity HIGH,CRITICAL "$IMAGE_REF"
          
          echo "✅ Security scan passed for POC image"

- stage: DeployDev
  displayName: 'Deploy to POC Development'
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  variables:
    imageDigest: $[ stageDependencies.Build.BuildAndTest.outputs['build_container.imageDigest'] ]
    digestReference: $[ stageDependencies.Build.BuildAndTest.outputs['build_container.digestReference'] ]
  
  jobs:
  - deployment: DeployToPOCDev
    displayName: 'Deploy IAM to POC Development'
    environment: 'onestream-poc-development'
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: Kubernetes@1
            displayName: 'Deploy to POC AKS'
            inputs:
              connectionType: 'Azure Resource Manager'
              azureSubscriptionEndpoint: 'sc-onestream-poc-dev'
              azureResourceGroup: 'onestream-poc-dev-rg'
              kubernetesCluster: 'onestream-poc-dev-aks'
              command: 'apply'
              arguments: '-f manifests/poc/'
          
          - task: KubernetesManifest@0
            displayName: 'Update Image Reference'
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'sc-onestream-poc-dev'
              namespace: 'iam-poc'
              manifests: 'manifests/poc/deployment.yaml'
              containers: 'iam:$(digestReference)'
          
          - task: Bash@3
            displayName: 'Verify POC Deployment'
            inputs:
              targetType: 'inline'
              script: |
                # Wait for deployment
                kubectl wait --for=condition=available deployment/iam-poc \
                  --namespace=iam-poc \
                  --timeout=300s
                
                # Verify correct image deployed
                DEPLOYED_IMAGE=$(kubectl get deployment iam-poc \
                  --namespace=iam-poc \
                  -o jsonpath='{.spec.template.spec.containers[0].image}')
                
                echo "✅ POC deployment successful"
                echo "📌 Deployed image: $DEPLOYED_IMAGE"
```

## 4. IAM Deployment Version Management Throughout Approval Stages

### 4.1 Version Lifecycle Flow

```yaml
iam_version_lifecycle:
  development:
    version_format: "poc-{build-id}-{git-sha}"
    example: "poc-12345-a1b2c3d"
    digest: "sha256:abc123..."
    approval_gates: ["Automated tests", "Security scan"]
    retention: "30 days"
  
  staging:
    version_reference: "Same digest as development"
    digest: "sha256:abc123..."  # Immutable reference
    approval_gates: ["QA team approval", "Integration tests"]
    deployment_method: "ACR import from dev"
    retention: "90 days"
  
  production:
    version_reference: "Same digest as staging"
    digest: "sha256:abc123..."  # Immutable reference
    approval_gates: ["Business approval", "Security final review", "Change advisory board"]
    deployment_method: "ACR import from staging"
    retention: "1 year"
```

### 4.2 Approval Gate Implementation

#### Development to Staging Promotion
```yaml
# promotion-dev-to-staging.yml
trigger: none

parameters:
- name: imageDigest
  displayName: 'Image Digest to Promote'
  type: string
- name: buildId
  displayName: 'Source Build ID'
  type: string

variables:
- group: onestream-poc-promotion-variables

stages:
- stage: ValidatePromotion
  displayName: 'Validate Development Readiness'
  jobs:
  - job: ValidationChecks
    displayName: 'Pre-Promotion Validation'
    pool:
      vmImage: 'ubuntu-latest'
    
    steps:
    - task: AzureCLI@2
      displayName: 'Verify Image Signature'
      inputs:
        azureSubscription: 'sc-onestream-poc-promotion'
        scriptType: bash
        scriptLocation: inlineScript
        inlineScript: |
          # Verify image exists and is signed
          IMAGE_REF="onestreampocdex.azurecr.io/iam@${{ parameters.imageDigest }}"
          
          # Check image exists
          az acr repository show \
            --name onestreampocdex \
            --image "iam@${{ parameters.imageDigest }}"
          
          # Verify signature
          cosign verify \
            --azure-kv-name onestream-poc-cosign-kv \
            --azure-kv-key cosign-signing-key \
            "$IMAGE_REF"
          
          echo "✅ Image validation passed for promotion"
    
    - task: Bash@3
      displayName: 'Check Development Deployment Health'
      inputs:
        targetType: 'inline'
        script: |
          # Verify current dev deployment is healthy
          kubectl config use-context onestream-poc-dev-aks
          
          DEPLOYMENT_STATUS=$(kubectl get deployment iam-poc \
            --namespace=iam-poc \
            -o jsonpath='{.status.conditions[?(@.type=="Available")].status}')
          
          if [[ "$DEPLOYMENT_STATUS" != "True" ]]; then
            echo "❌ Development deployment not healthy"
            exit 1
          fi
          
          echo "✅ Development deployment healthy"

- stage: PromoteToStaging
  displayName: 'Promote to Staging'
  dependsOn: ValidatePromotion
  condition: succeeded()
  
  jobs:
  - deployment: StagingPromotion
    displayName: 'Promote IAM to Staging'
    environment: 'onestream-poc-staging-promotion'
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: AzureCLI@2
            displayName: 'Import Image to Staging ACR'
            name: promote_image
            inputs:
              azureSubscription: 'sc-onestream-poc-promotion'
              scriptType: bash
              scriptLocation: inlineScript
              inlineScript: |
                # Import preserving digest
                SOURCE_REF="onestreampocdex.azurecr.io/iam@${{ parameters.imageDigest }}"
                TARGET_REF="iam@${{ parameters.imageDigest }}"
                
                echo "Promoting from: $SOURCE_REF"
                echo "Promoting to: onestreampocstaging.azurecr.io/$TARGET_REF"
                
                az acr import \
                  --name onestreampocstaging \
                  --source "$SOURCE_REF" \
                  --image "$TARGET_REF" \
                  --force
                
                # Verify digest preservation
                IMPORTED_DIGEST=$(az acr repository show \
                  --name onestreampocstaging \
                  --image "$TARGET_REF" \
                  --query "digest" -o tsv)
                
                if [[ "$IMPORTED_DIGEST" != "${{ parameters.imageDigest }}" ]]; then
                  echo "ERROR: Digest mismatch after import"
                  exit 1
                fi
                
                echo "##vso[task.setvariable variable=stagingReference;isOutput=true]onestreampocstaging.azurecr.io/$TARGET_REF"
                echo "✅ Successfully promoted to staging ACR"
          
          - task: TriggerPipeline@1
            displayName: 'Trigger Staging Deployment'
            inputs:
              serviceConnection: 'azure-devops-service-connection'
              project: '$(System.TeamProject)'
              Pipeline: 'iam-poc-deploy-staging'
              Parameters: |
                {
                  "imageReference": "$(promote_image.stagingReference)",
                  "sourceDigest": "${{ parameters.imageDigest }}",
                  "sourceBuildId": "${{ parameters.buildId }}"
                }
```

### 4.3 Version Tracking and Metadata

```yaml
version_metadata:
  storage_method: "Azure DevOps pipeline variables + ACR tags"
  
  metadata_structure:
    image_digest: "sha256:abc123..."
    source_build_id: "12345"
    source_commit: "a1b2c3d"
    build_timestamp: "2025-10-07T10:30:00Z"
    approval_history:
      - stage: "development"
        approved_by: "automated"
        timestamp: "2025-10-07T10:35:00Z"
      - stage: "staging"
        approved_by: "qa-team@onestream.com"
        timestamp: "2025-10-07T14:20:00Z"
      - stage: "production"
        approved_by: "change-board@onestream.com"
        timestamp: "2025-10-08T09:15:00Z"
    
    deployment_history:
      - environment: "poc-development"
        deployed_at: "2025-10-07T10:40:00Z"
        status: "healthy"
      - environment: "poc-staging"
        deployed_at: "2025-10-07T14:25:00Z"
        status: "healthy"
```

## 5. Rollback Strategy for Canary Deployment Failures

### 5.1 Canary Deployment with Rollback

#### Canary Deployment Pipeline
```yaml
# canary-deploy-with-rollback.yml
parameters:
- name: imageDigest
  displayName: 'New Image Digest'
  type: string
- name: canaryPercentage
  displayName: 'Canary Traffic Percentage'
  type: number
  default: 10
- name: healthCheckDuration
  displayName: 'Health Check Duration (minutes)'
  type: number
  default: 15

variables:
- group: onestream-poc-staging-variables
- name: serviceName
  value: 'iam'

stages:
- stage: PrepareCanary
  displayName: 'Prepare Canary Deployment'
  jobs:
  - job: CanaryPrep
    displayName: 'Prepare Canary Environment'
    pool:
      vmImage: 'ubuntu-latest'
    
    steps:
    - task: Bash@3
      displayName: 'Capture Current Production State'
      name: capture_state
      inputs:
        targetType: 'inline'
        script: |
          # Get current production image
          CURRENT_IMAGE=$(kubectl get deployment $(serviceName) \
            --namespace=$(serviceName)-poc \
            -o jsonpath='{.spec.template.spec.containers[0].image}')
          
          # Extract current digest
          CURRENT_DIGEST=$(echo "$CURRENT_IMAGE" | grep -o 'sha256:[a-f0-9]*')
          
          # Get current replica count
          CURRENT_REPLICAS=$(kubectl get deployment $(serviceName) \
            --namespace=$(serviceName)-poc \
            -o jsonpath='{.spec.replicas}')
          
          # Calculate canary and stable replica counts
          CANARY_REPLICAS=$(( $CURRENT_REPLICAS * ${{ parameters.canaryPercentage }} / 100 ))
          STABLE_REPLICAS=$(( $CURRENT_REPLICAS - $CANARY_REPLICAS ))
          
          # Ensure at least 1 replica for each
          if [[ $CANARY_REPLICAS -eq 0 ]]; then
            CANARY_REPLICAS=1
            STABLE_REPLICAS=$(( $CURRENT_REPLICAS - 1 ))
          fi
          
          # Output variables for rollback
          echo "##vso[task.setvariable variable=currentImage;isOutput=true]$CURRENT_IMAGE"
          echo "##vso[task.setvariable variable=currentDigest;isOutput=true]$CURRENT_DIGEST"
          echo "##vso[task.setvariable variable=stableReplicas;isOutput=true]$STABLE_REPLICAS"
          echo "##vso[task.setvariable variable=canaryReplicas;isOutput=true]$CANARY_REPLICAS"
          
          echo "📊 Current state captured:"
          echo "   Current image: $CURRENT_IMAGE"
          echo "   Stable replicas: $STABLE_REPLICAS"
          echo "   Canary replicas: $CANARY_REPLICAS"

- stage: DeployCanary
  displayName: 'Deploy Canary Version'
  dependsOn: PrepareCanary
  variables:
    currentImage: $[ stageDependencies.PrepareCanary.CanaryPrep.outputs['capture_state.currentImage'] ]
    currentDigest: $[ stageDependencies.PrepareCanary.CanaryPrep.outputs['capture_state.currentDigest'] ]
    stableReplicas: $[ stageDependencies.PrepareCanary.CanaryPrep.outputs['capture_state.stableReplicas'] ]
    canaryReplicas: $[ stageDependencies.PrepareCanary.CanaryPrep.outputs['capture_state.canaryReplicas'] ]
  
  jobs:
  - deployment: CanaryDeploy
    displayName: 'Deploy Canary'
    environment: 'onestream-poc-staging-canary'
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            displayName: 'Deploy Stable Version'
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'sc-onestream-poc-staging'
              namespace: '$(serviceName)-poc'
              manifests: |
                apiVersion: apps/v1
                kind: Deployment
                metadata:
                  name: $(serviceName)-stable
                  namespace: $(serviceName)-poc
                spec:
                  replicas: $(stableReplicas)
                  selector:
                    matchLabels:
                      app: $(serviceName)
                      version: stable
                  template:
                    metadata:
                      labels:
                        app: $(serviceName)
                        version: stable
                    spec:
                      containers:
                      - name: $(serviceName)
                        image: $(currentImage)
                        ports:
                        - containerPort: 8080
          
          - task: KubernetesManifest@0
            displayName: 'Deploy Canary Version'
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'sc-onestream-poc-staging'
              namespace: '$(serviceName)-poc'
              manifests: |
                apiVersion: apps/v1
                kind: Deployment
                metadata:
                  name: $(serviceName)-canary
                  namespace: $(serviceName)-poc
                spec:
                  replicas: $(canaryReplicas)
                  selector:
                    matchLabels:
                      app: $(serviceName)
                      version: canary
                  template:
                    metadata:
                      labels:
                        app: $(serviceName)
                        version: canary
                    spec:
                      containers:
                      - name: $(serviceName)
                        image: onestreampocstaging.azurecr.io/$(serviceName)@${{ parameters.imageDigest }}
                        ports:
                        - containerPort: 8080
          
          - task: Bash@3
            displayName: 'Configure Traffic Splitting'
            inputs:
              targetType: 'inline'
              script: |
                # Update service to include both stable and canary
                kubectl apply -f - <<EOF
                apiVersion: v1
                kind: Service
                metadata:
                  name: $(serviceName)-service
                  namespace: $(serviceName)-poc
                spec:
                  selector:
                    app: $(serviceName)
                  ports:
                  - port: 80
                    targetPort: 8080
                ---
                apiVersion: networking.istio.io/v1beta1
                kind: VirtualService
                metadata:
                  name: $(serviceName)-vs
                  namespace: $(serviceName)-poc
                spec:
                  http:
                  - match:
                    - headers:
                        x-canary:
                          exact: "true"
                    route:
                    - destination:
                        host: $(serviceName)-service
                        subset: canary
                  - route:
                    - destination:
                        host: $(serviceName)-service
                        subset: stable
                      weight: $(( 100 - ${{ parameters.canaryPercentage }} ))
                    - destination:
                        host: $(serviceName)-service
                        subset: canary
                      weight: ${{ parameters.canaryPercentage }}
                ---
                apiVersion: networking.istio.io/v1beta1
                kind: DestinationRule
                metadata:
                  name: $(serviceName)-dr
                  namespace: $(serviceName)-poc
                spec:
                  host: $(serviceName)-service
                  subsets:
                  - name: stable
                    labels:
                      version: stable
                  - name: canary
                    labels:
                      version: canary
                EOF
                
                echo "✅ Canary deployment configured with ${{ parameters.canaryPercentage }}% traffic"

- stage: MonitorCanary
  displayName: 'Monitor Canary Health'
  dependsOn: DeployCanary
  jobs:
  - job: HealthMonitoring
    displayName: 'Monitor Canary Health'
    pool:
      vmImage: 'ubuntu-latest'
    timeoutInMinutes: ${{ parameters.healthCheckDuration }}
    
    steps:
    - task: Bash@3
      displayName: 'Continuous Health Monitoring'
      name: health_check
      inputs:
        targetType: 'inline'
        script: |
          echo "🔍 Starting ${{ parameters.healthCheckDuration }}-minute health monitoring"
          
          MONITOR_DURATION=${{ parameters.healthCheckDuration }}
          END_TIME=$(($(date +%s) + $MONITOR_DURATION * 60))
          
          HEALTH_CHECKS_PASSED=0
          HEALTH_CHECKS_FAILED=0
          
          while [[ $(date +%s) -lt $END_TIME ]]; do
            # Check canary pod health
            CANARY_READY=$(kubectl get pods -n $(serviceName)-poc \
              -l app=$(serviceName),version=canary \
              -o jsonpath='{.items[*].status.conditions[?(@.type=="Ready")].status}' | grep -c "True" || echo "0")
            
            CANARY_TOTAL=$(kubectl get pods -n $(serviceName)-poc \
              -l app=$(serviceName),version=canary \
              --no-headers | wc -l)
            
            # Check error rates
            ERROR_RATE=$(kubectl logs -n $(serviceName)-poc \
              -l app=$(serviceName),version=canary \
              --since=1m | grep -i error | wc -l || echo "0")
            
            # Health check endpoint
            HEALTH_RESPONSE=$(kubectl exec -n $(serviceName)-poc \
              $(kubectl get pods -n $(serviceName)-poc -l app=$(serviceName),version=canary -o jsonpath='{.items[0].metadata.name}') \
              -- curl -s -o /dev/null -w "%{http_code}" http://localhost:8080/health || echo "000")
            
            echo "Health check $(date): Ready=$CANARY_READY/$CANARY_TOTAL, Errors=$ERROR_RATE, Health=$HEALTH_RESPONSE"
            
            # Evaluate health
            if [[ $CANARY_READY -eq $CANARY_TOTAL && $ERROR_RATE -lt 5 && $HEALTH_RESPONSE -eq 200 ]]; then
              HEALTH_CHECKS_PASSED=$((HEALTH_CHECKS_PASSED + 1))
            else
              HEALTH_CHECKS_FAILED=$((HEALTH_CHECKS_FAILED + 1))
            fi
            
            # Fail fast if too many failures
            if [[ $HEALTH_CHECKS_FAILED -gt 3 ]]; then
              echo "❌ Canary health check failed - triggering rollback"
              echo "##vso[task.setvariable variable=canaryHealthy;isOutput=true]false"
              exit 1
            fi
            
            sleep 30
          done
          
          # Final health evaluation
          SUCCESS_RATE=$((HEALTH_CHECKS_PASSED * 100 / (HEALTH_CHECKS_PASSED + HEALTH_CHECKS_FAILED)))
          
          if [[ $SUCCESS_RATE -ge 95 ]]; then
            echo "✅ Canary health monitoring passed ($SUCCESS_RATE% success rate)"
            echo "##vso[task.setvariable variable=canaryHealthy;isOutput=true]true"
          else
            echo "❌ Canary health monitoring failed ($SUCCESS_RATE% success rate)"
            echo "##vso[task.setvariable variable=canaryHealthy;isOutput=true]false"
            exit 1
          fi

- stage: DecisionPoint
  displayName: 'Promote or Rollback Decision'
  dependsOn: 
  - PrepareCanary
  - MonitorCanary
  condition: always()
  variables:
    canaryHealthy: $[ stageDependencies.MonitorCanary.HealthMonitoring.outputs['health_check.canaryHealthy'] ]
    currentImage: $[ stageDependencies.PrepareCanary.CanaryPrep.outputs['capture_state.currentImage'] ]
  
  jobs:
  - job: AutomatedDecision
    displayName: 'Automated Promote/Rollback Decision'
    pool:
      vmImage: 'ubuntu-latest'
    
    steps:
    - task: Bash@3
      displayName: 'Evaluate Canary Results'
      inputs:
        targetType: 'inline'
        script: |
          echo "📊 Canary evaluation results:"
          echo "   Health status: $(canaryHealthy)"
          echo "   Previous image: $(currentImage)"
          echo "   New digest: ${{ parameters.imageDigest }}"
          
          if [[ "$(canaryHealthy)" == "true" ]]; then
            echo "✅ Canary successful - proceeding to full promotion"
            echo "##vso[task.setvariable variable=shouldPromote]true"
          else
            echo "❌ Canary failed - initiating rollback"
            echo "##vso[task.setvariable variable=shouldPromote]false"
          fi

- stage: PromoteOrRollback
  displayName: 'Execute Promotion or Rollback'
  dependsOn: 
  - PrepareCanary
  - DecisionPoint
  condition: always()
  variables:
    shouldPromote: $[ stageDependencies.DecisionPoint.AutomatedDecision.outputs['shouldPromote'] ]
    currentImage: $[ stageDependencies.PrepareCanary.CanaryPrep.outputs['capture_state.currentImage'] ]
  
  jobs:
  - deployment: FinalDeployment
    displayName: 'Final Deployment Action'
    environment: 'onestream-poc-staging-final'
    condition: always()
    
    strategy:
      runOnce:
        deploy:
          steps:
          - task: Bash@3
            displayName: 'Execute Promotion or Rollback'
            inputs:
              targetType: 'inline'
              script: |
                if [[ "$(shouldPromote)" == "true" ]]; then
                  echo "🚀 Executing full promotion"
                  
                  # Update main deployment to new version
                  kubectl set image deployment/$(serviceName) \
                    $(serviceName)=onestreampocstaging.azurecr.io/$(serviceName)@${{ parameters.imageDigest }} \
                    --namespace=$(serviceName)-poc
                  
                  # Wait for rollout
                  kubectl rollout status deployment/$(serviceName) \
                    --namespace=$(serviceName)-poc \
                    --timeout=300s
                  
                  # Clean up canary resources
                  kubectl delete deployment $(serviceName)-stable --namespace=$(serviceName)-poc || true
                  kubectl delete deployment $(serviceName)-canary --namespace=$(serviceName)-poc || true
                  
                  echo "✅ Promotion completed successfully"
                  
                else
                  echo "🔄 Executing rollback to previous version"
                  
                  # Rollback to previous image
                  kubectl set image deployment/$(serviceName) \
                    $(serviceName)=$(currentImage) \
                    --namespace=$(serviceName)-poc
                  
                  # Wait for rollback
                  kubectl rollout status deployment/$(serviceName) \
                    --namespace=$(serviceName)-poc \
                    --timeout=300s
                  
                  # Clean up canary resources
                  kubectl delete deployment $(serviceName)-stable --namespace=$(serviceName)-poc || true
                  kubectl delete deployment $(serviceName)-canary --namespace=$(serviceName)-poc || true
                  
                  echo "✅ Rollback completed successfully"
                  echo "📋 Previous version restored: $(currentImage)"
                fi
          
          - task: Bash@3
            displayName: 'Verify Final State'
            inputs:
              targetType: 'inline'
              script: |
                # Verify deployment health
                kubectl wait --for=condition=available deployment/$(serviceName) \
                  --namespace=$(serviceName)-poc \
                  --timeout=300s
                
                # Get final image
                FINAL_IMAGE=$(kubectl get deployment $(serviceName) \
                  --namespace=$(serviceName)-poc \
                  -o jsonpath='{.spec.template.spec.containers[0].image}')
                
                echo "📋 Final deployment verification:"
                echo "   Final image: $FINAL_IMAGE"
                echo "   Deployment status: Healthy"
                
                if [[ "$(shouldPromote)" == "true" ]]; then
                  echo "✅ Canary promotion completed successfully"
                else
                  echo "🔄 Rollback completed successfully"
                fi
```

### 5.2 Rollback Automation and Monitoring

```yaml
rollback_automation:
  triggers:
    - "Health check failures"
    - "Error rate threshold exceeded"
    - "Manual intervention required"
    - "Performance degradation detected"
  
  rollback_speed:
    - "Automated detection: < 2 minutes"
    - "Rollback initiation: < 30 seconds"
    - "Full rollback completion: < 5 minutes"
    - "Service restoration: < 10 minutes total"
  
  verification_steps:
    - "Verify previous version availability"
    - "Update deployment manifests"
    - "Monitor rollback progress"
    - "Validate service health"
    - "Confirm traffic restoration"
    - "Alert stakeholders of status"
```

## Isolation from Production

### Production Safety Measures
```yaml
isolation_strategy:
  network_isolation:
    - "Separate VNet for POC environments"
    - "No connectivity to production networks"
    - "Isolated DNS and service discovery"
  
  data_isolation:
    - "Synthetic test data only"
    - "No production data access"
    - "Separate monitoring and logging"
  
  access_isolation:
    - "POC-specific service principals"
    - "No production resource access"
    - "Separate Azure AD groups"
  
  operational_isolation:
    - "Independent deployment cycles"
    - "Separate incident response"
    - "Isolated monitoring dashboards"
```

This comprehensive POC strategy ensures that OneStream can validate the new pipeline approach with complete isolation from production systems while demonstrating all the key benefits of the refactored architecture.