# Kubernetes CI/CD Pipeline with Jenkins, ArgoCD, and Helm
end-to-end Jenkins pipeline for a Java application using SonarQube, Argo CD, Helm, and Kubernetes
This repository contains a complete CI/CD pipeline for automating the deployment of a Spring Boot application to Kubernetes using Jenkins, ArgoCD, and Helm.

## Overview

- **Continuous Integration**: Jenkins builds, tests, and packages the application
- **Continuous Deployment**: ArgoCD deploys the application to Kubernetes using Helm charts
- **GitOps Workflow**: Infrastructure and deployment configurations are version-controlled

## Repository Structure

```
├── Jenkinsfile                  # Jenkins pipeline definition
│── springboot-app/      # Helm chart for the application
│── Chart.yaml
│── values.yaml      # Default values (used for test)
│── values-prod.yaml # Production override values
│── templates/       # Kubernetes resource templates
```

## Setup Instructions

### 1. Jenkins Configuration

Configure credentials in Jenkins:
- GitHub credentials (username + PAT)
- Docker registry credentials
- Kubernetes config
- ArgoCD credentials

### 2. ArgoCD Setup

```bash
# Install ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Create applications
argocd app create springboot-app-test \
  --repo https://github.com/yourname/repo.git \
  --path k8s/helm/springboot-app \
  --dest-namespace test \
  --sync-policy automated
```

### 3. Pipeline Execution

The Jenkins pipeline includes these stages:
1. Checkout source code
2. Build Java application
3. Run unit tests
4. Analyze code quality
5. Package application
6. Deploy to test environment
7. Run acceptance tests
8. Promote to production

## Troubleshooting

- For GitHub authentication issues, use a Personal Access Token
- Ensure correct permissions for Kubernetes configuration files
- Check ArgoCD logs for deployment issues
