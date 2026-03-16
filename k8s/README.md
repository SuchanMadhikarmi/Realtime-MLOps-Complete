# Kubernetes Manifests

This directory contains all Kubernetes manifests for deploying the churn prediction model on Kubernetes.

## Files

### `serviceaccount.yaml`

Defines the Kubernetes namespace, service account, and secret for Azure Blob Storage access.

**Components:**
1. **Namespace** - `churn-model`
   - Isolates the ML workload
   - Separate from other applications

2. **Service Account** - `sa-azure-access`
   - Authenticates the KServe InferenceService
   - Grants permissions to read Azure secret

3. **Secret** - `azure-storage-secret`
   - Stores Azure Storage account key
   - Used by KServe to download model from Blob Storage
   - Referenced by InferenceService via serviceAccountName

**Usage:**
```bash
# Apply to cluster
kubectl apply -f serviceaccount.yaml

# Verify
kubectl get ns churn-model
kubectl get sa -n churn-model
kubectl get secret -n churn-model
```

**Important:** Update the secret with your actual Azure Storage key:
```bash
kubectl create secret generic azure-storage-secret \
  --from-literal=AZURE_STORAGE_ACCESS_KEY='YOUR_KEY_HERE' \
  -n churn-model --dry-run=client -o yaml | kubectl apply -f -
```

---

### `inference.yaml`

Defines the KServe InferenceService for model serving.

**Components:**
1. **InferenceService** - `churn-predictor`
   - Serves the scikit-learn model
   - Auto-scales based on traffic
   - Provides REST API endpoint
   - Handles model versioning

**Spec Details:**
```yaml
Predictor:
  serviceAccountName: sa-azure-access      # Uses Azure secret
  sklearn:
    storageUri: az://dvc-store/models/...  # Model location in Azure
    resources:
      limits:
        memory: "1Gi"                       # Max memory
        cpu: "500m"                         # Max CPU
```

**Endpoints:** Once deployed, KServe creates:
- REST endpoint: `/v1/models/churn-predictor:predict`
- gRPC endpoint (optional): `churn-predictor:50009`

**Usage:**
```bash
# Deploy service
kubectl apply -f inference.yaml

# Check status
kubectl get isvc -n churn-model

# Wait for Ready status
kubectl get isvc churn-predictor -n churn-model -w

# Get service URL
kubectl get isvc churn-predictor -n churn-model -o jsonpath='{.status.url}'

# View full spec
kubectl describe isvc churn-predictor -n churn-model
```

---

## Deployment Workflow

### 1. Prerequisites

```bash
# Kubernetes cluster (KIND for local testing)
kind create cluster --name churn-model

# KServe installed
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.11.0/kserve.yaml

# Wait for KServe controller
kubectl wait --for=condition=ready pod \
  -l control-plane=kserve-controller-manager \
  -n kserve --timeout=300s
```

### 2. Apply Manifests

```bash
# Create namespace, service account, secret
kubectl apply -f serviceaccount.yaml

# Create the inference service
kubectl apply -f inference.yaml

# Verify
kubectl get all -n churn-model
```

### 3. Monitor Deployment

```bash
# Watch status
kubectl get isvc churn-predictor -n churn-model -w

# Check logs
kubectl logs -n churn-model -l serving.kserve.io/inferenceservice=churn-predictor --tail=100 -f

# Describe service
kubectl describe isvc churn-predictor -n churn-model
```

### 4. Test Inference

```bash
# Port-forward
kubectl port-forward -n churn-model svc/churn-predictor 8080:80 &

# Make prediction
curl -X POST http://localhost:8080/v1/models/churn-predictor:predict \
  -H "Content-Type: application/json" \
  -d '{"instances": [[45, 24, 79.99, 1920.00, 3]]}'

# Response
{"predictions": [1]}
```

---

## Troubleshooting

### InferenceService Stuck in "Creating"

```bash
# Check events
kubectl describe isvc churn-predictor -n churn-model

# Check predictor pod
kubectl get pods -n churn-model

# View logs
kubectl logs -n churn-model -l serving.kserve.io/inferenceservice=churn-predictor

# Common issues:
# 1. Secret not found
kubectl get secret azure-storage-secret -n churn-model

# 2. Model not accessible
az storage blob list --account-name churnmlops2025 --container-name dvc-store

# 3. Resource quota exceeded
kubectl describe resourcequota -n churn-model
```

### Connection Refused Error

```bash
# Ensure port-forward is running
lsof -i :8080

# Kill old process if needed
pkill -f "kubectl port-forward"

# Start fresh
kubectl port-forward -n churn-model svc/churn-predictor 8080:80
```

### OutOfMemory or CPU Limit Exceeded

Edit `inference.yaml` and increase resources:
```yaml
spec:
  predictor:
    sklearn:
      resources:
        limits:
          memory: "2Gi"      # Increase from 1Gi
          cpu: "1000m"       # Increase from 500m
```

Then reapply:
```bash
kubectl apply -f inference.yaml
```

---

## Advanced Configuration

### Auto-scaling

To enable auto-scaling based on traffic:

```yaml
# Add to inference.yaml under spec
autoscalerClass: keda
autoscalingTarget: 70
autoscalingPolicy:
  maxReplicas: 5
  minReplicas: 1
```

### Model Canary Deployment

```yaml
# Deploy new model version alongside old one
spec:
  predictor:
    canaryTrafficPercent: 10  # Route 10% to new model
    sklearn:
      storageUri: az://dvc-store/models/v2/churn_model.pkl
```

### Custom Resource Limits

```yaml
resources:
  limits:
    memory: "2Gi"
    cpu: "1000m"
  requests:
    memory: "512Mi"
    cpu: "250m"
```

---

## Cleanup

```bash
# Delete inference service
kubectl delete isvc churn-predictor -n churn-model

# Delete all namespace resources
kubectl delete namespace churn-model

# Delete KIND cluster
kind delete cluster --name churn-model
```

---

## More Information

- [KServe Documentation](https://kserve.github.io/)
- [Kubernetes Manifests](https://kubernetes.io/docs/concepts/configuration/overview/)
- [Azure Blob Storage](https://docs.microsoft.com/en-us/azure/storage/blobs/)

