# Contributing to Realtime MLOps Project

Thank you for your interest in contributing! This document provides guidelines for contributing to the project.

## Code of Conduct

- Be respectful and inclusive
- Focus on the code, not the person
- Provide constructive feedback
- Help others learn and grow

## How to Contribute

### Reporting Bugs

Create an issue with:
- **Title**: Clear description of the bug
- **Description**: What happened and what you expected
- **Steps to reproduce**: Detailed steps to recreate
- **Environment**: OS, Python version, dependencies
- **Logs**: Error messages or stack traces

Example:
```
Title: KServe InferenceService fails to load model from Azure

Description:
When deploying KServe InferenceService, the service gets stuck in "Creating" state
and never becomes ready.

Steps to reproduce:
1. Create KIND cluster
2. Install KServe
3. Create Azure storage secret
4. Apply inference.yaml
5. Wait 5 minutes

Expected: Service becomes Ready
Actual: Service stays in Creating state forever
```

### Requesting Features

Create an issue with:
- **Title**: Feature description
- **Motivation**: Why this feature is needed
- **Proposed solution**: How you'd implement it
- **Alternatives**: Other approaches considered

### Submitting Pull Requests

1. **Fork the repository**
   ```bash
   gh repo fork https://github.com/SuchanMadhikarmi/Realtime-MLOps-Project-copy
   cd Realtime-MLOps-Project-copy
   ```

2. **Create a feature branch**
   ```bash
   git checkout -b feature/amazing-feature
   ```

3. **Make your changes**
   - Write clean, readable code
   - Add docstrings to functions
   - Follow PEP 8 style guide
   - Comment complex logic

4. **Test your changes**
   ```bash
   # Run locally
   python train.py
   python api.py
   
   # Test with Kubernetes if applicable
   kubectl apply -f k8s/inference.yaml
   ```

5. **Commit with clear messages**
   ```bash
   git add .
   git commit -m "feat: add model explainability with SHAP values"
   ```
   
   Use conventional commits:
   - `feat:` New feature
   - `fix:` Bug fix
   - `docs:` Documentation
   - `style:` Code style
   - `refactor:` Code refactoring
   - `test:` Tests
   - `chore:` Build, dependencies

6. **Push to your fork**
   ```bash
   git push origin feature/amazing-feature
   ```

7. **Create Pull Request**
   - Describe what you changed
   - Link related issues
   - Request reviewers
   - Wait for feedback

8. **Address review feedback**
   - Make requested changes
   - Push to the same branch
   - GitHub will update the PR automatically

## Development Setup

```bash
# Clone
git clone https://github.com/YOUR_USERNAME/Realtime-MLOps-Project-copy.git
cd Realtime-MLOps-Project-copy

# Create virtual environment
python3 -m venv venv
source venv/bin/activate

# Install dependencies + dev dependencies
pip install -r requirements.txt
pip install pytest black flake8

# Format code
black .

# Lint code
flake8 .

# Run tests (if added)
pytest
```

## Code Standards

### Python Style

```python
# ✅ Good
def predict_churn(age, tenure, monthly_charges):
    """
    Predict customer churn probability.
    
    Args:
        age: Customer age
        tenure: Months as customer
        monthly_charges: Monthly fee
        
    Returns:
        dict: {'prediction': 0/1, 'probability': float}
    """
    features = [[age, tenure, monthly_charges]]
    prediction = model.predict(features)[0]
    probability = model.predict_proba(features)[0][1]
    return {'prediction': int(prediction), 'probability': float(probability)}

# ❌ Bad
def predict(a, t, m):
    return model.predict([[a, t, m]])
```

### Kubernetes YAML

```yaml
# ✅ Good - use 2-space indent, clear structure
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: churn-predictor
  namespace: churn-model
spec:
  predictor:
    serviceAccountName: sa-azure-access
    sklearn:
      storageUri: az://dvc-store/models/churn_model.pkl

# ❌ Bad - inconsistent indent, unclear
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
    name: churn-predictor
spec:
    predictor:
       serviceAccountName: sa-azure-access
       sklearn:
         storageUri: az://dvc-store/models/churn_model.pkl
```

## What to Work On

Good first issues:
- [ ] Add input validation to API endpoints
- [ ] Write unit tests for model training
- [ ] Add logging to training script
- [ ] Improve error messages
- [ ] Update documentation
- [ ] Add model performance metrics

Intermediate issues:
- [ ] Add model explainability (SHAP)
- [ ] Implement A/B testing for model versions
- [ ] Add model monitoring/drift detection
- [ ] Create Prometheus metrics
- [ ] Add API rate limiting

## Pull Request Checklist

- [ ] Changes follow code standards
- [ ] Code is formatted (`black .`)
- [ ] Code passes linting (`flake8`)
- [ ] Tests pass (if applicable)
- [ ] Documentation is updated
- [ ] Commit messages are clear
- [ ] No unrelated changes included

## Review Process

1. **Automated checks** - GitHub Actions runs tests
2. **Code review** - Maintainer reviews changes
3. **Feedback** - Suggestions for improvement
4. **Approval** - Ready to merge
5. **Merge** - Changes go to main branch

## Release Process

Maintainers follow semantic versioning (MAJOR.MINOR.PATCH):
- `MAJOR`: Breaking changes
- `MINOR`: New features
- `PATCH`: Bug fixes

## Questions?

- Check existing issues/PRs
- Read the [README](README.md)
- Review [AZURE_DEPLOYMENT_GUIDE.md](AZURE_DEPLOYMENT_GUIDE.md)
- Create a discussion

## License

By contributing, you agree your changes are licensed under MIT License.

---

Thank you for contributing! 🙏
