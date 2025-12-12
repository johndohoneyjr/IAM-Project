To verify hash and sign images with Cosign in OneSream's AKS deployment, you need to deploy several components. Here's what gets deployed:

## Core Cosign Verification Components

### **1. Cosign Binary in Init Containers**

```yaml
# deployment-with-cosign-verification.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: onestream-iam-service
spec:
  template:
    spec:
      # Init container to verify image signature before main container starts
      initContainers:
      - name: cosign-verify
        image: gcr.io/projectsigstore/cosign:v2.2.0
        command: ["/ko-app/cosign"]
        args:
        - "verify"
        - "--key"
        - "/etc/cosign/public-key"
        - "onestreamdev.azurecr.io/iam/service@sha256:abc123def456..."
        volumeMounts:
        - name: cosign-public-key
          mountPath: /etc/cosign
          readOnly: true
        env:
        - name: COSIGN_EXPERIMENTAL
          value: "1"  # For keyless verification
      
      containers:
      - name: iam-service
        # Only starts if cosign verification succeeds
        image: "onestreamdev.azurecr.io/iam/service@sha256:abc123def456..."
        
      volumes:
      - name: cosign-public-key
        secret:
          secretName: cosign-public-key
```

### **2. Admission Controllers**

#### **Cosign Policy Controller (Recommended)**
```yaml
# Install Cosign Policy Controller
apiVersion: v1
kind: Namespace
metadata:
  name: cosign-system
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cosign-policy-controller
  namespace: cosign-system
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cosign-policy-controller
  template:
    metadata:
      labels:
        app: cosign-policy-controller
    spec:
      containers:
      - name: cosign-policy-controller
        image: gcr.io/projectsigstore/cosign-policy-controller:latest
        env:
        - name: SYSTEM_NAMESPACE
          value: cosign-system
        - name: CONFIG_LOGGING_NAME
          value: config-logging
        - name: CONFIG_OBSERVABILITY_NAME
          value: config-observability
        - name: METRICS_DOMAIN
          value: sigstore.dev/cosign
        ports:
        - containerPort: 8443
          name: webhook
        - containerPort: 9090
          name: metrics
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          capabilities:
            drop:
            - ALL
```

#### **Validation Webhook**
```yaml
# cosign-validation-webhook.yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionWebhook
metadata:
  name: cosign-validation-webhook
webhooks:
- name: cosign.sigstore.dev
  clientConfig:
    service:
      name: cosign-policy-controller
      namespace: cosign-system
      path: /validate
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  - operations: ["CREATE", "UPDATE"]
    apiGroups: ["apps"]
    apiVersions: ["v1"]
    resources: ["deployments", "daemonsets", "statefulsets"]
  admissionReviewVersions: ["v1", "v1beta1"]
  sideEffects: None
  failurePolicy: Fail  # Reject unsigned images
```

### **3. Policy Configuration**

#### **ClusterImagePolicy for OneStream**
```yaml
# onestream-cosign-policy.yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: onestream-image-policy
spec:
  images:
  - glob: "onestreamdev.azurecr.io/**"
  - glob: "onestreamstaging.azurecr.io/**" 
  - glob: "onestreamprod.azurecr.io/**"
  authorities:
  - name: onestream-authority
    keyless:
      url: https://fulcio.sigstore.dev
      identities:
      - issuer: https://login.microsoftonline.com/YOUR-TENANT-ID/v2.0
        subject: ".*@onestream\\.com"  # Only OneStream developers
    # OR use Azure Key Vault keys
    key:
      kms: azurekms://onestream-kv.vault.azure.net/cosign-signing-key
  policy:
    matchRepoDigest: true
    type: cue
    data: |
      predicateType: "https://slsa.dev/provenance/v0.2"
      predicate: {
        buildType: "azure-devops"
        builder: {
          id: "https://dev.azure.com/onestream"
        }
      }
```

### **4. Secrets and ConfigMaps**

#### **Cosign Public Key Secret**
```yaml
# cosign-public-key-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: cosign-public-key
  namespace: default
type: Opaque
data:
  public-key: |
    LS0tLS1CRUdJTiBQVUJMSUMgS0VZLS0tLS0K...  # Base64 encoded public key
```

#### **Azure Key Vault Integration**
```yaml
# azure-kv-secret-provider.yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: cosign-azure-kv
spec:
  provider: azure
  parameters:
    usePodIdentity: "false"
    useVMManagedIdentity: "true"
    userAssignedIdentityID: "YOUR-MANAGED-IDENTITY-CLIENT-ID"
    keyvaultName: "onestream-cosign-kv"
    objects: |
      array:
        - |
          objectName: cosign-signing-key
          objectType: key
          objectVersion: ""
  secretObjects:
  - secretName: cosign-signing-key
    type: Opaque
    data:
    - objectName: cosign-signing-key
      key: key
```

## Enhanced Deployment Pipeline Integration

### **1. Pre-Deployment Verification**
```yaml
# enhanced-deployment-pipeline.yml
- task: AzureCLI@2
  displayName: 'Verify Image Signature Before Deploy'
  inputs:
    scriptType: 'bash'
    inlineScript: |
      # Verify signature exists and is valid
      cosign verify \
        --key azurekms://onestream-kv.vault.azure.net/cosign-signing-key \
        $(TARGET_ACR).azurecr.io/$(SERVICE_NAME)@$(IMAGE_DIGEST)
      
      if [ $? -eq 0 ]; then
        echo "✅ Image signature verified"
      else
        echo "❌ Image signature verification failed"
        exit 1
      fi

- task: HelmDeploy@0
  displayName: 'Deploy Verified Image'
  inputs:
    command: 'upgrade'
    chartType: 'FilePath'
    chartPath: './charts/$(SERVICE_NAME)'
    arguments: '--set image.digest=$(IMAGE_DIGEST) --set image.verificationEnabled=true'
```

### **2. Runtime Verification Sidecar**
```yaml
# runtime-verification-sidecar.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: iam-service
        image: "onestreamprod.azurecr.io/iam/service@sha256:verified-digest"
        
      # Sidecar for continuous verification
      - name: cosign-verifier
        image: gcr.io/projectsigstore/cosign:v2.2.0
        command: ["/bin/sh"]
        args:
        - -c
        - |
          while true; do
            echo "Performing periodic signature verification..."
            cosign verify \
              --key /etc/cosign/public-key \
              onestreamprod.azurecr.io/iam/service@sha256:verified-digest
            
            if [ $? -ne 0 ]; then
              echo "❌ Signature verification failed - alerting..."
              # Send alert to monitoring system
              curl -X POST http://monitoring-service/alert \
                -d '{"alert": "signature_verification_failed", "service": "iam-service"}'
            fi
            
            sleep 300  # Check every 5 minutes
          done
        volumeMounts:
        - name: cosign-public-key
          mountPath: /etc/cosign
          readOnly: true
```

## Monitoring and Alerting Components

### **1. Cosign Verification Metrics**
```yaml
# cosign-metrics-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: cosign-metrics-config
data:
  config.yaml: |
    metrics:
      enabled: true
      port: 9090
      path: /metrics
    verification:
      log_failures: true
      alert_on_failure: true
      webhook_url: "http://alertmanager:9093/api/v1/alerts"
```

### **2. ServiceMonitor for Prometheus**
```yaml
# cosign-servicemonitor.yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: cosign-policy-controller
spec:
  selector:
    matchLabels:
      app: cosign-policy-controller
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

## OneStream-Specific Verification Workflow

### **1. Multi-Environment Verification**
```yaml
# Environment-specific verification policies
# dev-environment-policy.yaml
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: onestream-dev-policy
spec:
  images:
  - glob: "onestreamdev.azurecr.io/**"
  authorities:
  - keyless:
      url: https://fulcio.sigstore.dev
      identities:
      - issuer: https://login.microsoftonline.com/DEV-TENANT-ID/v2.0
        subject: ".*@onestream\\.com"

---
# prod-environment-policy.yaml  
apiVersion: policy.sigstore.dev/v1beta1
kind: ClusterImagePolicy
metadata:
  name: onestream-prod-policy
spec:
  images:
  - glob: "onestreamprod.azurecr.io/**"
  authorities:
  - key:
      kms: azurekms://onestream-prod-kv.vault.azure.net/prod-signing-key
  policy:
    matchRepoDigest: true
    type: cue
    data: |
      predicateType: "https://onestream.com/fips-compliance/v1"
      predicate: {
        fipsCompliant: true
        securityScanPassed: true
        buildPipeline: "onestream-prod-pipeline"
      }
```

### **2. FIPS Compliance Verification**
```yaml
# fips-compliance-verification.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      initContainers:
      - name: fips-compliance-check
        image: gcr.io/projectsigstore/cosign:v2.2.0
        command: ["/bin/sh"]
        args:
        - -c
        - |
          echo "Verifying FIPS compliance attestation..."
          cosign verify-attestation \
            --key azurekms://onestream-prod-kv.vault.azure.net/fips-key \
            --type https://onestream.com/fips-compliance/v1 \
            onestreamprod.azurecr.io/$(SERVICE_NAME)@$(IMAGE_DIGEST)
          
          if [ $? -eq 0 ]; then
            echo "✅ FIPS compliance verified"
          else
            echo "❌ FIPS compliance verification failed"
            exit 1
          fi
```

## Summary of Deployed Components

| Component | Purpose | Deployment Location |
|-----------|---------|-------------------|
| **Cosign Policy Controller** | Admission control | `cosign-system` namespace |
| **ValidatingAdmissionWebhook** | Policy enforcement | Cluster-wide |
| **ClusterImagePolicy** | Verification rules | Cluster-wide |
| **Init Containers** | Pre-deployment verification | Per deployment |
| **Verification Sidecars** | Runtime monitoring | Per deployment (optional) |
| **Secrets/ConfigMaps** | Keys and configuration | Per namespace |
| **ServiceMonitor** | Metrics collection | Monitoring namespace |

This setup ensures that **only signed, verified images with valid digests** can be deployed to OneStream's AKS clusters, providing the security and compliance OneStream needs.