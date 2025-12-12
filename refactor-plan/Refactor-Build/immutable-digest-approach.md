An **immutable digest** is a cryptographic hash (typically SHA-256) that uniquely identifies the exact content of a container image or artifact. Unlike tags, which can be moved to point to different images, digests are permanent and unchangeable.

## How Immutable Digests Work

### **Digest Structure**
```
sha256:abc123def456789...
```
- **Algorithm**: Usually SHA-256
- **Hash**: 64-character hexadecimal string
- **Immutable**: Cannot be changed once created
- **Content-Based**: Generated from the actual image layers and metadata

### **Tag vs Digest Comparison**

```bash
# MUTABLE TAG (can change)
myacr.azurecr.io/myapp:v1.2.3
myacr.azurecr.io/myapp:latest

# IMMUTABLE DIGEST (never changes)
myacr.azurecr.io/myapp@sha256:abc123def456789...
```

## Why Digests Solve OneStream's Tagging Issues

Based on your meeting notes mentioning "Issue with Tarball is the tagging," here's how digests address those problems:

### **1. Tag Overwriting Problem**
```bash
# Problem: Same tag, different content
docker push myacr.azurecr.io/app:v1.0  # First push
# Later someone accidentally pushes different code with same tag
docker push myacr.azurecr.io/app:v1.0  # Overwrites previous image!

# Solution: Use digest reference
docker pull myacr.azurecr.io/app@sha256:abc123...  # Always gets exact same image
```

### **2. Cross-Tenant ACR Sync Issues**
```yaml
# Problem: Tags can conflict during ACR sync
Source ACR: myapp:v1.0 (content A)
Target ACR: myapp:v1.0 (content B) # Conflict!

# Solution: Digest-based sync
Source ACR: myapp@sha256:contentA123...
Target ACR: myapp@sha256:contentA123...  # No conflict, exact content
```

## How Digests Are Generated

### **Image Build Process**
```bash
# When you build an image
docker build -t myapp:latest .

# Docker generates layers with digests
Layer 1: sha256:layer1hash...
Layer 2: sha256:layer2hash...
Manifest: sha256:manifesthash...  # This becomes the image digest
```

### **Digest Calculation**
```json
{
  "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
  "schemaVersion": 2,
  "config": {
    "digest": "sha256:configdigest...",
    "mediaType": "application/vnd.docker.container.image.v1+json",
    "size": 1234
  },
  "layers": [
    {
      "digest": "sha256:layer1digest...",
      "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
      "size": 5678
    }
  ]
}
```

The digest is calculated from this entire manifest structure.

## Practical Examples for OneStream

### **1. FIPS-Compliant Azure Linux Images**
```dockerfile
# Instead of using potentially mutable tag
FROM mcr.microsoft.com/azurelinux/base/core:3.0

# Use immutable digest for FIPS compliance
FROM mcr.microsoft.com/azurelinux/base/core@sha256:fips-validated-digest-here
```

### **2. Helm Chart with Digest References**
```yaml
# values.yaml
image:
  repository: myacr.azurecr.io/onestream-app
  digest: sha256:abc123def456...  # Immutable reference
  # tag: "v1.2.3"  # Remove mutable tag

# deployment.yaml
containers:
- name: app
  image: "{{ .Values.image.repository }}@{{ .Values.image.digest }}"
```

### **3. Pipeline with Digest Management**
```yaml
# Addressing your build-release.yml concerns
- task: AzureCLI@2
  displayName: 'Get Build Digest'
  name: 'getDigest'
  inputs:
    scriptType: 'bash'
    inlineScript: |
      # After building image
      DIGEST=$(az acr repository show \
        --name $(ACR_NAME) \
        --image $(IMAGE_NAME):$(Build.BuildId) \
        --query "digest" -o tsv)
      
      echo "##vso[task.setvariable;variable=IMAGE_DIGEST;isOutput=true]$DIGEST"
      echo "Immutable digest: $DIGEST"

# Later stage uses digest instead of tag
- task: HelmDeploy@0
  inputs:
    arguments: '--set image.digest=$(getDigest.IMAGE_DIGEST)'
```

## Benefits for OneStream's Use Cases

### **1. Eliminates Tag Collision Issues**
- **Problem**: Different teams pushing same tag
- **Solution**: Each build gets unique digest
- **Result**: No accidental overwrites

### **2. Ensures Reproducible Deployments**
```bash
# Deploy exact same image to all environments
kubectl set image deployment/app \
  container=myacr.azurecr.io/app@sha256:abc123...
```

### **3. Supports Multi-Tenant Architecture**
```yaml
# Different tenants, same validated image
tenant-a:
  image: myacr.azurecr.io/app@sha256:validated-digest

tenant-b:
  image: myacr.azurecr.io/app@sha256:validated-digest  # Exact same content
```

### **4. Enables Secure Cross-ACR Sync**
```bash
# Sync preserves content integrity
az acr import \
  --name prod-acr \
  --source dev-acr.azurecr.io/app@sha256:tested-digest \
  --image app@sha256:tested-digest
```

## Security Implications

### **Supply Chain Security**
- **Verification**: Can verify image hasn't been tampered with
- **Attestation**: Can sign digests for supply chain validation
- **Compliance**: FIPS/FedRAMP requirements met with validated digests

### **Example with Cosign (for signature verification)**
```bash
# Sign the digest
cosign sign myacr.azurecr.io/app@sha256:abc123...

# Verify before deployment
cosign verify myacr.azurecr.io/app@sha256:abc123...
```

## Addressing Your Pipeline Concerns

Based on your notes about "Central as possible - top level yaml for each service":

```yaml
# Generic pipeline template using digests
parameters:
- name: services
  type: object

stages:
- stage: BuildAll
  jobs:
  - ${{ each service in parameters.services }}:
    - job: Build_${{ service.name }}
      steps:
      - template: build-with-digest.yml
        parameters:
          serviceName: ${{ service.name }}
          
- stage: DeployAll
  dependsOn: BuildAll
  jobs:
  - ${{ each service in parameters.services }}:
    - job: Deploy_${{ service.name }}
      variables:
        serviceDigest: $[ stageDependencies.BuildAll.Build_${{ service.name }}.outputs['build.IMAGE_DIGEST'] ]
      steps:
      - template: deploy-with-digest.yml
        parameters:
          serviceName: ${{ service.name }}
          imageDigest: $(serviceDigest)
```

This approach eliminates the tagging issues you mentioned while supporting the centralized pipeline architecture you're working toward.