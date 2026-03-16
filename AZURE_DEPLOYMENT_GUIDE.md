# Azure MLOps Migration - Deployment Guide

## Status Summary ✅

### Completed Tasks
- ✅ Migrated from AWS S3 → Azure Blob Storage (`churnmlops2025`)
- ✅ Migrated from AWS ECR → Azure Container Registry (`churnmlopsacr.azurecr.io`)
- ✅ Service Principal created for GitHub Actions CI/CD (`sp-churn-mlops`)
- ✅ GitHub Actions workflow configured (`.github/workflows/mlops-pipeline.yaml`)
- ✅ Kubernetes manifests updated (`k8s/inference.yaml`, `k8s/serviceaccount.yaml`)
- ✅ ArgoCD application manifest updated (`argocd/application.yaml`)
- ✅ Removed exposed storage key from repository (security fix)

### GitHub Secrets Configured ✅
All 8 required secrets are set in Settings → Secrets → Actions:
- `AZURE_CLIENT_ID` - Service Principal Client ID
- `AZURE_CLIENT_SECRET` - Service Principal Client Secret
- `AZURE_TENANT_ID` - Azure Tenant ID
- `AZURE_SUBSCRIPTION_ID` - Azure Subscription ID
- `AZURE_STORAGE_CONNECTION_STRING` - Storage Account connection string
- `ACR_LOGIN_SERVER` - `churnmlopsacr.azurecr.io`
- `ACR_USERNAME` - `churnmlopsacr`
- `ACR_PASSWORD` - ACR access credential

---

## Next Steps - Azure Kubernetes Secret Setup

### Option A: Manual Setup (Recommended for Development)

Once your Kubernetes cluster is running and you've deployed the `serviceaccount.yaml`:

```bash
# 1. Apply the namespace and service account
kubectl apply -f k8s/serviceaccount.yaml

# 2. Create the secret with the actual storage key
kubectl create secret generic azure-storage-secret \
  --from-literal=AZURE_STORAGE_ACCESS_KEY='<your-storage-account-key>' \
  -n churn-model \
  --dry-run=client -o yaml | kubectl apply -f -

# 3. Verify the secret was created
kubectl get secret azure-storage-secret -n churn-model
kubectl describe secret azure-storage-secret -n churn-model
```

### Option B: Automated Setup via GitHub Actions (Production-Ready)

To automatically create/update the Kubernetes secret when the workflow runs:

1. **Add `AZURE_STORAGE_KEY` to GitHub Secrets**
   - Go to repo Settings → Secrets and Variables → Actions
   - Add secret: `AZURE_STORAGE_KEY` = `<primary-key-from-azure-portal>`
   - ⚠️ **WARNING**: This key can read/write all blobs. Rotate key1 first before adding!

2. **Extend the GitHub Actions workflow** to create the Kubernetes secret:

Add this step after "Push model to Azure Blob Storage" in `.github/workflows/mlops-pipeline.yaml`:

```yaml
    - name: Authenticate to Kubernetes
      run: |
        # Configure kubeconfig for your cluster
        # Example for local KIND cluster (adjust as needed)
        echo "${{ secrets.KUBECONFIG_B64 }}" | base64 -d > $HOME/.kube/config
        chmod 600 $HOME/.kube/config

    - name: Create/Update Azure Storage Secret in Kubernetes
      run: |
        kubectl create namespace churn-model --dry-run=client -o yaml | kubectl apply -f -
        kubectl create secret generic azure-storage-secret \
          --from-literal=AZURE_STORAGE_ACCESS_KEY='${{ secrets.AZURE_STORAGE_KEY }}' \
          -n churn-model \
          --dry-run=client -o yaml | kubectl apply -f -
        echo "✅ Azure storage secret created in Kubernetes"
```

For this to work, you'll need to add `KUBECONFIG_B64` as a GitHub Secret:
```bash
cat ~/.kube/config | base64 -w 0 | xclip -selection clipboard
```

---

## Verification Checklist

### ✅ Before Running CI/CD:
- [ ] All 8 GitHub Secrets configured
- [ ] `AZURE_STORAGE_KEY` added to secrets (after rotating key1)
- [ ] Kubernetes cluster is running (KIND or live cluster)
- [ ] `kubectl` can access the cluster

### ✅ After First Workflow Run:
- [ ] GitHub Actions workflow completes successfully
- [ ] Model artifact appears in Azure Blob Storage (`az://dvc-store/models/churn_model.pkl`)
- [ ] KServe can access the model (InferenceService should be `Ready`)
- [ ] Kubernetes secret exists: `kubectl get secret azure-storage-secret -n churn-model`

### ✅ Manual Testing:
```bash
# Test Azure authentication
az login --service-principal \
  -u $AZURE_CLIENT_ID \
  -p $AZURE_CLIENT_SECRET \
  --tenant $AZURE_TENANT_ID

# Test blob access
az storage blob list --account-name churnmlops2025 --container-name dvc-store

# Test Kubernetes access
kubectl get serviceaccount sa-azure-access -n churn-model
kubectl get secret azure-storage-secret -n churn-model
```

---

## Security Notes ⚠️

1. **Storage Account Key Rotation**
   - A storage account key was exposed in the repository
   - **ACTION REQUIRED**: Rotate key1 in Azure Portal immediately
   - In Azure Portal: Storage Account → Access Keys → Rotate key1
   - Update `AZURE_STORAGE_CONNECTION_STRING` and `AZURE_STORAGE_KEY` secrets with new credentials

2. **Service Principal Credentials**
   - Don't commit the client secret to the repository
   - Rotate periodically (e.g., every 90 days)
   - Use Azure Managed Identity for pod-to-pod authentication when possible

3. **KServe Storage Configuration**
   - KServe uses the `serviceAccountName: sa-azure-access` to mount the secret
   - The secret must exist in the same namespace before deploying the InferenceService

---

## Troubleshooting

### Issue: GitHub Actions workflow fails at "Create/Update Azure Storage Secret"
**Solution**: Ensure `KUBECONFIG_B64` secret is configured and base64-encoded correctly

### Issue: KServe InferenceService shows `ModelLoadingFailed`
**Solution**: 
- Verify secret exists: `kubectl get secret azure-storage-secret -n churn-model`
- Check KServe pod logs: `kubectl logs -f <pod-name> -n churn-model`
- Ensure storage URI format is correct: `az://dvc-store/models/churn_model.pkl`

### Issue: "Cannot access Azure Blob Storage"
**Solution**:
- Verify Service Principal has `Contributor` role on resource group
- Verify Service Principal has `AcrPush` role on ACR
- Test connectivity: `az storage blob list --account-name churnmlops2025 --container-name dvc-store`

---

## Files Updated on `cicd` Branch

| File | Status | Notes |
|------|--------|-------|
| `.github/workflows/mlops-pipeline.yaml` | ✅ Ready | Trains model, pushes to ACR + Blob, updates inference.yaml |
| `k8s/serviceaccount.yaml` | ✅ Secure | Empty placeholder for storage key (populated at runtime) |
| `k8s/inference.yaml` | ✅ Updated | Azure Blob Storage path configured |
| `argocd/application.yaml` | ✅ Updated | Correct repo URL and cicd branch |
| `requirements.txt` | ✅ Updated | Added dvc-azure, azure-storage-blob |

---

## Quick Reference - Required Secrets

```yaml
# These must be added to GitHub Settings → Secrets → Actions
AZURE_CLIENT_ID: "38e01995-0739-4d30-a8fb-c1d8ea47b209"
AZURE_CLIENT_SECRET: <get from Azure Portal>
AZURE_TENANT_ID: "cdf5fc8b-d50a-446f-a05d-028e22d1461a"
AZURE_SUBSCRIPTION_ID: "1cc54060-e727-4cd8-afdd-e1bcb64cb07a"
AZURE_STORAGE_CONNECTION_STRING: <retrieved from Azure Portal>
AZURE_STORAGE_KEY: <storage account primary key>
ACR_LOGIN_SERVER: "churnmlopsacr.azurecr.io"
ACR_USERNAME: "churnmlopsacr"
ACR_PASSWORD: <retrieved from Azure Portal>
KUBECONFIG_B64: <optional - base64 kubeconfig for K8s automation>
```

**Note**: Never commit actual credentials to the repository. All sensitive values should be stored in GitHub Secrets or Azure Key Vault.
