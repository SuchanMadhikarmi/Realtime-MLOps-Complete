# ArgoCD - GitOps Deployment

This directory contains the ArgoCD Application manifest for GitOps-based continuous deployment.

## What is GitOps?

GitOps is a deployment methodology where:
1. **Git is the source of truth** - Cluster state is defined in git
2. **Automated reconciliation** - ArgoCD syncs cluster to match git
3. **Declarative configuration** - YAML defines desired state
4. **Continuous monitoring** - ArgoCD watches git and cluster

## File

### `application.yaml`

Defines the ArgoCD Application that manages deployment.

**Key Components:**

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: churn-model
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/SuchanMadhikarmi/Realtime-MLOps-Project-copy
    targetRevision: cicd           # Track 'cicd' branch
    path: k8s                       # Manifests in k8s/ directory
  destination:
    server: https://kubernetes.default.svc
    namespace: churn-model
  syncPolicy:
    automated:
      prune: true                   # Delete resources not in git
      selfHeal: true                # Auto-sync on drift
    syncOptions:
    - CreateNamespace=true          # Auto-create namespace
```

---

## Setup Guide

### 1. Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD components
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for ArgoCD to be ready (takes ~30 seconds)
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=60s
```

### 2. Access ArgoCD UI

```bash
# Port-forward to ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Open browser: https://localhost:8080

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d

# Login with:
# Username: admin
# Password: <from above command>
```

**Note:** You'll get a certificate warning—this is normal for self-signed certs.

### 3. Deploy Application

```bash
# Apply ArgoCD application
kubectl apply -f argocd/application.yaml

# Check status
kubectl get application -n argocd

# Describe application
kubectl describe application churn-model -n argocd
```

### 4. Monitor Sync

```bash
# Watch sync status
kubectl get application churn-model -n argocd -w

# View detailed status
kubectl get application churn-model -n argocd -o yaml

# Check resource sync
kubectl get all -n churn-model
```

---

## Workflow

### Initial Deployment

1. **Developer commits code to `cicd` branch**
   ```bash
   git push origin cicd
   ```

2. **GitHub Actions runs**
   - Trains model
   - Pushes to Azure
   - Updates k8s manifests (optional)
   - Commits changes

3. **ArgoCD detects changes**
   - Polls git repo every 3 minutes (default)
   - Compares git state with cluster state

4. **ArgoCD syncs**
   - Applies new manifests from git
   - Scales deployments
   - Updates service configs
   - Pulls new Docker images

5. **KServe updates**
   - Model serving endpoint gets new version
   - Old version gracefully terminates
   - Load balanced to new pods

### Continuous Reconciliation

```
┌─────────────────┐
│  GitHub (Git)   │  Source of Truth
└────────┬────────┘
         │
         │ (ArgoCD polls)
         ▼
┌─────────────────────┐
│  ArgoCD Controller  │  Reconciliation
│  - Monitors git     │
│  - Detects drift    │
│  - Auto-syncs       │
└────────┬────────────┘
         │
         │ (kubectl apply)
         ▼
┌──────────────────────┐
│  Kubernetes Cluster  │  Actual State
│  - KServe Service    │
│  - Pods              │
│  - Secrets           │
└──────────────────────┘
```

---

## GitOps Principles

### 1. Declarative Configuration

All cluster state is defined in git:
```yaml
# k8s/inference.yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: churn-predictor
spec:
  predictor:
    sklearn:
      storageUri: az://dvc-store/models/churn_model.pkl  # Version pinned
```

### 2. Immutable Artifacts

Models and code are versioned:
- Docker images: `churnmlopsacr.azurecr.io/churn-model:v1.2.3`
- Models: `az://dvc-store/models/churn_model.pkl` (DVC versioning)
- Code: Git commit SHA for traceability

### 3. Automated Recovery

If cluster state drifts from git:
```bash
# Manual drift (e.g., someone runs kubectl edit)
kubectl patch isvc churn-predictor -n churn-model --type merge \
  -p '{"spec":{"predictor":{"sklearn":{"storageUri":"WRONG_PATH"}}}}'

# ArgoCD detects within 3 minutes
# Automatically reverts to git state
```

### 4. Audit Trail

All changes tracked in git:
```bash
# See deployment history
git log --oneline k8s/inference.yaml

# See who changed what and when
git show <commit>

# Rollback to previous version
git revert <commit>
git push origin cicd
# ArgoCD auto-syncs
```

---

## Common Tasks

### Deploy New Model Version

1. **GitHub Actions trains new model**
   ```bash
   git push origin cicd
   # (Workflow runs, trains, pushes to Azure)
   ```

2. **Update k8s/inference.yaml**
   ```yaml
   storageUri: az://dvc-store/models/churn_model_v2.pkl  # New version
   ```

3. **Commit and push**
   ```bash
   git add k8s/inference.yaml
   git commit -m "Deploy new model version"
   git push origin cicd
   ```

4. **ArgoCD auto-syncs**
   - Detects git change
   - Updates KServe service
   - Pulls new model
   - Ready within 5 minutes

### Rollback to Previous Version

```bash
# See history
git log --oneline k8s/inference.yaml

# Revert to previous commit
git revert <commit-hash>
git push origin cicd

# ArgoCD automatically rolls back in cluster
# old model version becomes active
```

### Manual Sync (if needed)

```bash
# Via CLI
argocd app sync churn-model

# Via UI
# ArgoCD dashboard → Applications → churn-model → SYNC button
```

### Disable Auto-sync

```bash
# Edit application
kubectl edit application churn-model -n argocd

# Change:
# - synchPolicy.automated: true → false

# Then manually sync when ready
argocd app sync churn-model
```

---

## Monitoring & Troubleshooting

### Check Sync Status

```bash
# Quick status
kubectl get application churn-model -n argocd

# Detailed status
kubectl get application churn-model -n argocd -o jsonpath='{.status}'

# Watch for changes
watch kubectl get application churn-model -n argocd
```

### View Sync Logs

```bash
# ArgoCD controller logs
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller -f

# View specific application status
argocd app get churn-model

# See sync details
argocd app info churn-model
```

### Troubleshoot Out-of-Sync

```bash
# Check what's different
argocd app diff churn-model

# If git is correct but cluster wrong:
argocd app sync churn-model --force

# Delete and re-sync
kubectl delete application churn-model -n argocd
kubectl apply -f argocd/application.yaml
```

### Repository Connection Issues

```bash
# Check repo credentials
argocd repo list

# Test connection
argocd repo list -o table

# View application repository status
kubectl get application churn-model -n argocd -o yaml | grep -A 10 "repositoryConnectionState"
```

---

## Advanced Configuration

### Sync Wave - Ordered Deployments

```yaml
# Apply secret first, then service
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "0"   # Secret (applied first)
---
metadata:
  annotations:
    argocd.argoproj.io/sync-wave: "1"   # Service (applied second)
```

### Sync Options

```yaml
syncPolicy:
  syncOptions:
  - CreateNamespace=true        # Auto-create namespace
  - RespectIgnoreDifferences=true  # Ignore certain fields
  - PruneLast=true              # Delete old resources last
```

### Resource Tracking

```yaml
# Track by name instead of auto-generated names
trackingMethod: annotation
```

---

## Best Practices

✅ **Do:**
- Keep git as source of truth
- Use semantic versioning for images
- Tag releases in git
- Document change reasons in commits
- Review PRs before merging
- Test changes in dev cluster first
- Monitor application health

❌ **Don't:**
- Manually edit resources in cluster
- Hardcode host/IP addresses
- Store secrets in git (use Sealed Secrets)
- Skip testing
- Deploy directly without ArgoCD
- Ignore sync failures

---

## Cleanup

```bash
# Delete application
kubectl delete application churn-model -n argocd

# Delete ArgoCD (optional)
kubectl delete namespace argocd

# This keeps the cluster resources (KServe, namespace, etc.)
# To also delete cluster resources:
kubectl delete namespace churn-model
```

---

## More Information

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [GitOps Guide](https://www.gitops.tech/)
- [Kubernetes GitOps](https://kubernetes.io/blog/2021/04/07/gitops-with-argo-cd/)

