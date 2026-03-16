# 🚀 Real-Time MLOps Project - Customer Churn Prediction

A production-ready MLOps pipeline for predicting customer churn using scikit-learn, FastAPI, DVC, KServe, ArgoCD, and GitHub Actions on **Microsoft Azure**.

---

## 📋 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Azure Resources](#azure-resources)
- [Project Structure](#project-structure)
- [Getting Started](#getting-started)
- [CI/CD Pipeline](#cicd-pipeline)
- [Kubernetes & KServe Deployment](#kubernetes--kserve-deployment)
- [GitOps with ArgoCD](#gitops-with-argocd)
- [Model Inference](#model-inference)
- [Monitoring & Troubleshooting](#monitoring--troubleshooting)
- [Contributing](#contributing)

---

## 🎯 Overview

This project demonstrates **production-grade MLOps practices** using a customer churn prediction model:

### What the Model Does

The model predicts which customers are likely to cancel their subscriptions based on:
- **Age** - Customer age
- **Tenure** - Months as a customer
- **Monthly Charges** - Monthly subscription fee
- **Total Charges** - Lifetime spending
- **Support Calls** - Number of support interactions

**Example Prediction:**
```json
{
  "customer_age": 45,
  "tenure_months": 24,
  "monthly_charges": 79.99,
  "total_charges": 1920.00,
  "support_calls": 3,
  "churn_prediction": 1,
  "probability": 0.73
}
```
👉 73% chance of cancellation → Enable proactive retention strategies

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        GitHub (Source)                       │
│  - Push to 'cicd' branch triggers automation               │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                  GitHub Actions (CI/CD)                      │
│  1. Checkout code                                            │
│  2. Generate synthetic dataset                              │
│  3. Train model (scikit-learn)                              │
│  4. Push model to Azure Blob Storage                        │
│  5. Build Docker image                                      │
│  6. Push to Azure Container Registry                        │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│              Azure Services                                  │
│  - Blob Storage: Models & Data (DVC)                        │
│  - Container Registry: Docker Images                        │
│  - Service Principal: Authentication                        │
└────────────────────┬────────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                Kubernetes Cluster (KIND/AKS)                │
│                                                              │
│  Namespace: churn-model                                    │
│  ┌────────────────────────────────────────────────────┐   │
│  │  KServe InferenceService (churn-predictor)        │   │
│  │  - Serves sklearn model                            │   │
│  │  - Scales automatically                            │   │
│  │  - Monitors predictions                            │   │
│  └────────────────────────────────────────────────────┘   │
│                                                              │
│  ┌────────────────────────────────────────────────────┐   │
│  │  ArgoCD Application                                │   │
│  │  - Watches git repo for changes                    │   │
│  │  - Auto-syncs k8s manifests                        │   │
│  │  - GitOps continuous deployment                    │   │
│  └────────────────────────────────────────────────────┘   │
└───────────────────────────────────────────────────────────┘
```

---

## ☁️ Azure Resources

All Azure resources are provisioned for this project:

| Resource | Name | Purpose |
|----------|------|---------|
| **Storage Account** | `churnmlops2025` | Store models & training data (DVC) |
| **Blob Container** | `dvc-store` | Version control for ML artifacts |
| **Container Registry** | `churnmlopsacr.azurecr.io` | Store Docker images |
| **Service Principal** | `sp-churn-mlops` | Authentication for GitHub Actions & Kubernetes |
| **Resource Group** | `rg-churn-mlops` | Organize all resources |

### Service Principal Credentials (Store Safely)
```
Client ID:       38e01995-0739-4d30-a8fb-c1d8ea47b209
Tenant ID:       cdf5fc8b-d50a-446f-a05d-028e22d1461a
Subscription ID: 1cc54060-e727-4cd8-afdd-e1bcb64cb07a
```

⚠️ **Client Secret** is NEVER stored in git—only in GitHub Secrets!

---

## 📁 Project Structure

```
realtime-mlops-project/
├── README.md                          # Project documentation
├── AZURE_DEPLOYMENT_GUIDE.md         # Azure setup guide
├── requirements.txt                   # Python dependencies
├── Dockerfile                         # Container image definition
│
├── generate_data.py                   # Create synthetic dataset
├── train.py                           # Train sklearn model
├── api.py                             # FastAPI inference server
│
├── models/
│   └── churn_model.pkl               # Trained model (generated)
│
├── data/
│   └── churn_data.csv                # Training data (generated)
│
├── k8s/                              # Kubernetes manifests
│   ├── serviceaccount.yaml           # Namespace, SA, Azure secret
│   └── inference.yaml                # KServe InferenceService
│
├── argocd/
│   └── application.yaml              # ArgoCD application (GitOps)
│
├── .github/workflows/
│   └── mlops-pipeline.yaml           # CI/CD automation
│
└── .dvc/                             # DVC configuration (local)
    └── config                        # Remote storage settings
```

---

## 🚀 Getting Started

### Prerequisites

- Python 3.11+
- Docker (for local testing)
- `kubectl` (for Kubernetes)
- `kind` (for local K8s cluster)
- Azure CLI
- Git

### Quick Start (Local)

```bash
# Clone repository
git clone https://github.com/SuchanMadhikarmi/Realtime-MLOps-Project-copy.git
cd Realtime-MLOps-Project-copy

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Generate dataset
python generate_data.py

# Train model
python train.py

# Test API locally
python api.py
# Visit: http://localhost:8000/docs
```

### API Endpoints

**FastAPI Docs:** `http://localhost:8000/docs`

**Predict Endpoint:**
```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "age": 45,
    "tenure_months": 24,
    "monthly_charges": 79.99,
    "total_charges": 1920.00,
    "num_support_calls": 3
  }'
```

**Response:**
```json
{
  "churn": 1,
  "churn_probability": 0.73
}
```

---

## 🔄 CI/CD Pipeline

### How It Works

1. **Developer pushes to `cicd` branch** 
2. **GitHub Actions workflow triggers:**
   - Installs dependencies
   - Generates synthetic dataset
   - Trains model (logs metrics to stdout)
   - Authenticates to Azure (via Service Principal)
   - Pushes model to Azure Blob Storage
   - Builds Docker image
   - Pushes image to Azure Container Registry
   - Updates Kubernetes manifests (optional)

3. **Model is ready for deployment**

### Pipeline Status

Check workflow runs: `Settings → Actions → All workflows`

### Required GitHub Secrets

⚠️ **All 8 secrets must be configured:**

```
AZURE_CLIENT_ID              = 38e01995-0739-4d30-a8fb-c1d8ea47b209
AZURE_CLIENT_SECRET          = <from Azure Portal>
AZURE_TENANT_ID              = cdf5fc8b-d50a-446f-a05d-028e22d1461a
AZURE_SUBSCRIPTION_ID        = 1cc54060-e727-4cd8-afdd-e1bcb64cb07a
AZURE_STORAGE_CONNECTION_STRING = <from Storage Account>
AZURE_STORAGE_KEY            = <from Storage Account>
ACR_LOGIN_SERVER             = churnmlopsacr.azurecr.io
ACR_USERNAME                 = churnmlopsacr
ACR_PASSWORD                 = <from Azure Portal>
```

**How to add secrets:**
1. Go to `Settings → Secrets and variables → Actions`
2. Click "New repository secret"
3. Add each secret name and value

### DVC Integration

Models are automatically versioned in Azure Blob Storage:
- Path: `az://dvc-store/models/churn_model.pkl`
- Tracked via DVC metadata files
- Reproducible training pipeline

---

## 🎮 Kubernetes & KServe Deployment

### Prerequisites
- KIND cluster or AKS cluster
- `kubectl` configured
- Model in Azure Blob Storage ✅

### Step 1: Create KIND Cluster (Local Testing)

```bash
kind create cluster --name churn-model
```

### Step 2: Install KServe

```bash
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.11.0/kserve.yaml

# Wait for KServe controller to be ready
kubectl wait --for=condition=ready pod \
  -l control-plane=kserve-controller-manager \
  -n kserve --timeout=300s
```

### Step 3: Create Kubernetes Namespace & Secret

```bash
# Apply namespace and service account
kubectl apply -f k8s/serviceaccount.yaml

# Create Azure storage secret
# ⚠️ Use your ACTUAL storage key from Azure Portal
kubectl create secret generic azure-storage-secret \
  --from-literal=AZURE_STORAGE_ACCESS_KEY='YOUR_STORAGE_KEY_HERE' \
  -n churn-model \
  --dry-run=client -o yaml | kubectl apply -f -

# Verify
kubectl get secret azure-storage-secret -n churn-model -o yaml
```

### Step 4: Deploy KServe InferenceService

```bash
kubectl apply -f k8s/inference.yaml

# Monitor deployment
kubectl get isvc -n churn-model -w

# Wait until STATUS = Ready (takes 2-5 minutes)
kubectl get isvc churn-predictor -n churn-model
```

### Step 5: Setup ArgoCD (GitOps Continuous Deployment)

**What is GitOps?**
GitOps is a deployment methodology where Git is the single source of truth. ArgoCD automatically syncs your Kubernetes cluster to match your git repository.

**Installation:**

```bash
# 1. Create ArgoCD namespace
kubectl create namespace argocd

# 2. Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# 3. Wait for ArgoCD to be ready
kubectl wait --for=condition=ready pod \
  -l app.kubernetes.io/name=argocd-server \
  -n argocd --timeout=60s
```

**Access ArgoCD Dashboard:**

```bash
# Port-forward to ArgoCD server
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Open browser: https://localhost:8080
# Login with username: admin and password from above
```

**Deploy Application (GitOps):**

```bash
# Apply ArgoCD Application manifest
kubectl apply -f argocd/application.yaml

# Check sync status
kubectl get application churn-model -n argocd

# Watch for ready status
kubectl get application churn-model -n argocd -w

# Describe application
kubectl describe application churn-model -n argocd
```

**How It Works:**

1. **Git as Source of Truth** - `argocd/application.yaml` points to your git repo (`cicd` branch)
2. **Automatic Sync** - ArgoCD monitors git and automatically applies changes
3. **Self-Healing** - If cluster state drifts, ArgoCD restores it
4. **Audit Trail** - All deployments tracked in git commit history

**Rolling Back a Deployment:**

```bash
# See git history
git log --oneline k8s/inference.yaml

# Revert to previous version
git revert <commit-hash>
git push origin cicd

# ArgoCD automatically syncs (within 3 minutes) or manually:
# kubectl port-forward svc/argocd-server -n argocd 8080:443
# Then click SYNC in ArgoCD UI
```

**For More Details:** See [argocd/README.md](argocd/README.md) for comprehensive GitOps guide

---

## � GitOps with ArgoCD

ArgoCD provides **continuous deployment** by automatically syncing your Kubernetes cluster with your git repository. This section covers the complete GitOps workflow.

### Key Benefits

✅ **Git as Single Source of Truth** - Cluster state defined in git  
✅ **Automatic Deployment** - Changes push automatically  
✅ **Self-Healing** - Detects and corrects drift  
✅ **Rollback in Seconds** - Revert via git commit  
✅ **Full Audit Trail** - Complete deployment history  
✅ **No Manual kubectl** - Everything declarative  

### Complete GitOps Workflow

```
Your Code → GitHub (cicd branch) 
           ↓
       GitHub Actions
       - Train model
       - Push to Azure
       - Update manifests
           ↓
    Git Commit Change
           ↓
    ArgoCD Detects Change
           ↓
   Kubernetes Cluster Updated
    - New model deployed
    - KServe synced
    - Service ready
```

### Monitoring ArgoCD

```bash
# Check application sync status
kubectl get application churn-model -n argocd -o wide

# View detailed status
kubectl get application churn-model -n argocd -o jsonpath='{.status}'

# Check if cluster matches git
argocd app diff churn-model

# View deployment history
git log --oneline -- k8s/

# See which resources are synced
kubectl get all -n churn-model
```

### Real-World Example: Deploy New Model Version

**Scenario:** You've trained a new model version and want to deploy it.

```bash
# Step 1: New model pushed to Azure (GitHub Actions does this automatically)
# New model: az://dvc-store/models/churn_model_v2.pkl

# Step 2: Update git manifest
vim k8s/inference.yaml
# Change: storageUri: az://dvc-store/models/churn_model_v2.pkl

# Step 3: Commit and push
git add k8s/inference.yaml
git commit -m "Deploy churn model v2"
git push origin cicd

# Step 4: ArgoCD automatically syncs within 3 minutes
# ✅ Old model pods terminate
# ✅ New model pods start
# ✅ Service stays live during transition

# Step 5: Verify new version
kubectl get isvc churn-predictor -n churn-model -o jsonpath='{.spec.predictor.sklearn.storageUri}'
```

### Rollback Example

```bash
# See what changed
git log --oneline k8s/inference.yaml

# Rollback to previous version
git revert <commit-hash>
git push origin cicd

# ArgoCD auto-syncs back to old model
# Old version becomes active within 3 minutes
```

For detailed ArgoCD setup, troubleshooting, and advanced configurations, see [argocd/README.md](argocd/README.md).

---

## �📊 Model Inference

### Via KServe (REST API)

```bash
# Port-forward to service
kubectl port-forward -n churn-model svc/churn-predictor-predictor-default 8080:80 &

# Make prediction
curl -X POST http://localhost:8080/v1/models/churn-predictor:predict \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [
      [45, 24, 79.99, 1920.00, 3]
    ]
  }'
```

**Response:**
```json
{
  "predictions": [1]
}
```

### Via FastAPI (Local)

```bash
# Start API server
python api.py

# Make prediction
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "age": 45,
    "tenure_months": 24,
    "monthly_charges": 79.99,
    "total_charges": 1920.00,
    "num_support_calls": 3
  }'
```

---

## 🔍 Monitoring & Troubleshooting

### Check Model Status

```bash
# Describe inference service
kubectl describe isvc churn-predictor -n churn-model

# Check predictor pod logs
kubectl logs -n churn-model -l serving.kserve.io/inferenceservice=churn-predictor

# Check if model is ready
kubectl get isvc churn-predictor -n churn-model -o jsonpath='{.status.conditions[*].message}'
```

### Common Issues

**Issue: InferenceService stuck in "Creating" state**

```bash
# Check events
kubectl describe isvc churn-predictor -n churn-model

# Check storage secret exists
kubectl get secret azure-storage-secret -n churn-model

# Verify Azure connectivity
kubectl run -it --rm debug --image=mcr.microsoft.com/azure-cli:latest -- bash
az storage blob list --account-name churnmlops2025 --container-name dvc-store
```

**Issue: "Cannot download model"**

```bash
# Verify Azure storage secret
kubectl get secret azure-storage-secret -n churn-model -o yaml

# Check blob storage path
az storage blob list --account-name churnmlops2025 --container-name dvc-store --output table
```

**Issue: Port-forward not working**

```bash
# Get correct service name
kubectl get svc -n churn-model

# Check if pods are running
kubectl get pods -n churn-model

# Port-forward with correct service
kubectl port-forward -n churn-model svc/churn-predictor 8080:80
```

---

## 🔐 Security Best Practices

### ✅ What We Do Right

- ✅ Azure Service Principal instead of root credentials
- ✅ Secrets stored in GitHub Secrets (encrypted)
- ✅ Empty placeholder in serviceaccount.yaml (no hardcoded keys)
- ✅ Kubernetes secrets for Azure storage access
- ✅ RBAC on Azure resources (Service Principal has limited roles)

### ⚠️ Important Security Notes

1. **Rotate Azure Storage Key Regularly**
   - Azure Portal → Storage Accounts → Access Keys → Rotate key1
   - Update `AZURE_STORAGE_KEY` secret in GitHub

2. **Never Commit Secrets**
   - GitHub push protection blocks credential commits
   - Always use GitHub Secrets for sensitive data

3. **Limit Service Principal Permissions**
   - Current: Contributor on resource group
   - Production: Use minimal scoped roles

4. **Network Security**
   - Enable VNet endpoints for Blob Storage
   - Use Private Link for secure connectivity

---

## 📚 Key Technologies

| Component | Purpose | Version |
|-----------|---------|---------|
| **scikit-learn** | ML model training | 1.8.0+ |
| **FastAPI** | REST API server | 0.135.0+ |
| **DVC** | Data version control | 3.67.0+ |
| **KServe** | ML model serving | 0.11.0+ |
| **ArgoCD** | GitOps deployment | Latest |
| **GitHub Actions** | CI/CD automation | Built-in |
| **Azure Blob Storage** | Model storage | Standard |
| **Azure Container Registry** | Docker images | Standard |
| **Kubernetes** | Orchestration | 1.24+ |

---

## 🛠️ Development Workflow

### Adding New Features

1. Create feature branch
2. Make changes
3. Test locally (`python train.py`, `python api.py`)
4. Push to GitHub
5. Create Pull Request
6. Merge to `cicd` branch
7. GitHub Actions auto-deploys

### Training a New Model

```bash
# Modify generate_data.py or train.py
# Then push changes:
git add .
git commit -m "Update model training"
git push origin cicd

# GitHub Actions automatically:
# - Trains new model
# - Pushes to Azure Blob Storage
# - Updates KServe deployment (via ArgoCD)
```

---

## 📖 Additional Resources

- [Azure Deployment Guide](AZURE_DEPLOYMENT_GUIDE.md)
- [KServe Documentation](https://kserve.github.io/)
- [GitHub Actions Guide](https://docs.github.com/en/actions)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [DVC Documentation](https://dvc.org/)

---

## 🤝 Contributing

### Issue Reporting

Found a bug? Create an issue with:
- Description
- Steps to reproduce
- Expected vs actual behavior
- Environment details

### Pull Requests

1. Fork the repository
2. Create feature branch (`git checkout -b feature/amazing-feature`)
3. Commit changes (`git commit -m 'Add amazing feature'`)
4. Push to branch (`git push origin feature/amazing-feature`)
5. Open Pull Request

### Code Standards

- Python: PEP 8 compliance
- YAML: Consistent formatting (2-space indent)
- Commits: Clear, descriptive messages
- Tests: Add tests for new features

---

## 📝 License

This project is open source and available under the MIT License.

---

## 👨‍💼 Project Maintainer

**Suchan Madhikarmi**  
[GitHub](https://github.com/SuchanMadhikarmi) | [LinkedIn](https://linkedin.com/in/suchanmadhikarmi)

---

## 📞 Support

For questions or issues:
1. Check [Troubleshooting](#monitoring--troubleshooting) section
2. Review [AZURE_DEPLOYMENT_GUIDE.md](AZURE_DEPLOYMENT_GUIDE.md)
3. Create GitHub issue
4. Contact maintainer

---

**Last Updated:** March 2026  
**Status:** ✅ Production Ready
