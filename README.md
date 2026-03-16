# Churn Model MLOps Demo

A simple demonstration of MLOps practices for a customer churn prediction model.

## What Does This Model Do?

**Real-World Example:**

Imagine you run a telecom company with thousands of customers. Some customers are happy and stay for years, while others leave (churn) after a few months. This model predicts which customers are likely to leave.

**Example Customer:**
- **Sarah** is 45 years old
- Been a customer for 24 months
- Pays $79.99/month
- Total spent: $1,920
- Called customer support 3 times this month

**Model Prediction:**
```json
{
  "churn": 1,
  "churn_probability": 0.73
}
```

**Translation:** Sarah has a **73% chance of canceling her subscription**. Why? She's calling support frequently (unhappy) and paying relatively high fees. Your business can now:
- Offer her a discount
- Reach out with personalized support
- Prevent losing her before she leaves

**The model looks at patterns** like:
- High monthly charges → More likely to churn
- More support calls → Customer is frustrated
- Low tenure → Haven't built loyalty yet

This helps businesses **save customers proactively** instead of reacting after they've already left!

## Project Structure

```
churn-model/
├── generate_data.py          # Generate synthetic churn dataset
├── train.py                   # Train the model
├── api.py                     # FastAPI inference server
├── requirements.txt           # Python dependencies
├── Dockerfile                 # Container image
├── .dvc/config               # DVC configuration
├── models/
│   └── churn_model.pkl.dvc   # DVC metadata for model
├── k8s/
│   ├── deployment.yaml       # Kubernetes deployment
│   └── inference.yaml        # KServe inference service
├── .github/workflows/
│   └── mlops-pipeline.yaml   # GitHub Actions CI/CD
└── argocd/
    └── application.yaml      # ArgoCD application
```

## MLOps Pipeline Steps

### 1. Initial Setup

```bash
# Install dependencies
pip install -r requirements.txt

# Generate dataset
python generate_data.py

# Train model
python train.py

# Test API locally
python api.py
# Visit http://localhost:8000/docs
```

### 2. DVC Setup (Data Version Control)

```bash
# Initialize DVC
dvc init

# Configure Azure Blob Storage remote
dvc remote add -d myremote az://dvc-store/churn-model

# Track model with DVC
dvc add models/churn_model.pkl

# Push to Azure Blob Storage
dvc push

# Commit DVC metadata
git add models/churn_model.pkl.dvc .dvc/ .gitignore
git commit -m "Track model with DVC"
```

### 3. Push Model to Azure Blob Storage

After training the model and setting up DVC:

```bash
# Configure Azure credentials (if not already done)
export AZURE_STORAGE_ACCOUNT=churnmlops2025
export AZURE_STORAGE_CONTAINER_NAME=dvc-store
export AZURE_STORAGE_KEY=<your-storage-account-key>

# Create container (if needed)
az storage container create --name dvc-store --account-name churnmlops2025

# Push model to Azure Blob Storage using DVC
dvc push

# Verify model is in Azure
az storage blob list --account-name churnmlops2025 --container-name dvc-store
```

The model will be stored in Azure Blob Storage at: `az://dvc-store/churn-model/models/churn_model.pkl`

### 4. Azure Blob Storage Configuration

Models are automatically pushed to Azure Blob Storage by the GitHub Actions pipeline. No manual configuration needed.

### 5. Kubernetes with KIND

```bash
# Create KIND cluster
kind create cluster --name churn-model
```

### 6. KServe Setup

```bash
# Install KServe
kubectl apply -f https://github.com/kserve/kserve/releases/download/v0.11.0/kserve.yaml

# Create namespace, ServiceAccount and Azure storage secret for KServe
# Update k8s/serviceaccount.yaml with your Azure storage key first
kubectl apply -f k8s/serviceaccount.yaml

# Deploy inference service
kubectl apply -f k8s/inference.yaml

# Check inference service
kubectl get inferenceservice -n churn-model

# Wait for it to be ready
kubectl get inferenceservice churn-predictor -n churn-model -w
```

**Important:** Before deploying, ensure the Azure storage secret exists:
```bash
kubectl create secret generic azure-storage-secret \
  --from-literal=AZURE_STORAGE_ACCESS_KEY='<your-storage-key>' \
  -n churn-model --dry-run=client -o yaml | kubectl apply -f -
```

### 7. Test KServe Inference

```bash
# Get the inference service URL
INGRESS_HOST=$(kubectl get inferenceservice churn-predictor -n churn-model -o jsonpath='{.status.url}' | cut -d/ -f3)
SERVICE_HOSTNAME=$(kubectl get inferenceservice churn-predictor -n churn-model -o jsonpath='{.status.url}' | cut -d/ -f3)

# For local KIND cluster, port-forward
kubectl port-forward -n churn-model service/churn-predictor-predictor-default 8080:80

# Test prediction with curl
# Note: sklearn models expect data as arrays, not named features
# Order: age, tenure_months, monthly_charges, total_charges, num_support_calls
curl -X POST http://localhost:8080/v1/models/churn-predictor:predict \
  -H "Content-Type: application/json" \
  -d '{
    "instances": [
      [45, 24, 79.99, 1920.00, 3]
    ]
  }'
```

Expected response:
```json
{
  "predictions": [1]
}
```

### 8. GitHub Actions

**Required Secrets (Already Configured):**
- `AZURE_CLIENT_ID`
- `AZURE_CLIENT_SECRET`
- `AZURE_TENANT_ID`
- `AZURE_SUBSCRIPTION_ID`
- `AZURE_STORAGE_CONNECTION_STRING`
- `ACR_LOGIN_SERVER` (churnmlopsacr.azurecr.io)
- `ACR_USERNAME`
- `ACR_PASSWORD`

**Pipeline Flow:**
1. Checkout code
2. Generate dataset
3. Train model
4. Push model to Azure Blob Storage via DVC
5. Build Docker image
6. Push image to Azure Container Registry (ACR)
7. Update `inference.yaml` with new image tag (optional)
8. Commit changes (triggers ArgoCD)

### 9. ArgoCD (GitOps)

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Deploy application
kubectl apply -f argocd/application.yaml

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Complete MLOps Workflow

1. **Developer pushes code** → GitHub
2. **GitHub Actions triggered:**
   - Trains model
   - Pushes model to Azure Blob Storage (DVC)
   - Builds Docker image
   - Pushes to Azure Container Registry (ACR)
   - Updates `inference.yaml` (optional)
3. **ArgoCD detects change** in `inference.yaml`
4. **ArgoCD syncs** → Deploys to Kubernetes
5. **KServe serves** the new model version

## API Usage

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

Response:
```json
{
  "churn": 1,
  "churn_probability": 0.73
}
```

## Key Components

- **DVC**: Version control for data and models in Azure Blob Storage
- **Azure Blob Storage**: Remote storage for models and data
- **Azure Container Registry (ACR)**: Docker image registry (churnmlopsacr.azurecr.io)
- **Azure Service Principal**: Authentication for GitHub Actions and Kubernetes
- **KServe**: Serverless ML inference on Kubernetes
- **KIND**: Local Kubernetes for testing
- **GitHub Actions**: CI/CD automation
- **ArgoCD**: GitOps continuous deployment

## Notes

- Azure resources configured for this project:
  - Storage Account: `churnmlops2025`
  - Container Registry: `churnmlopsacr.azurecr.io`
  - Service Principal: `sp-churn-mlops`
  - Resource Group: `rg-churn-mlops`
- All GitHub Actions secrets are already configured
- For detailed deployment instructions, see [AZURE_DEPLOYMENT_GUIDE.md](AZURE_DEPLOYMENT_GUIDE.md)
- This is a minimal demo - production setups require monitoring, logging, and security hardening
