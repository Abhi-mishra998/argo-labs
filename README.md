# 🚀 Argo Labs - Complete CI/CD Pipeline on Kubernetes

[![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![ArgoCD](https://img.shields.io/badge/ArgoCD-EF7B4D?style=for-the-badge&logo=argo&logoColor=white)](https://argoproj.github.io/cd/)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)](https://opensource.org/licenses/MIT)

> A production-ready CI/CD pipeline implementation using the complete Argo suite: ArgoCD, Argo Rollouts, Argo Workflows, Argo Events, and Argo Image Updater with GitOps principles.

## 📋 Table of Contents

- [Overview](#-overview)
- [Architecture](#-architecture)
- [Features](#-features)
- [Prerequisites](#-prerequisites)
- [Installation](#-installation)
- [Project Structure](#-project-structure)
- [Deployment Strategies](#-deployment-strategies)
- [Monitoring](#-monitoring)
- [Commands Reference](#-commands-reference)
- [Troubleshooting](#-troubleshooting)
- [Contributing](#-contributing)

## 🎯 Overview

This project demonstrates a complete **CI/CD pipeline** for microservices deployment using Kubernetes and the Argo ecosystem. It implements **GitOps principles** for declarative continuous delivery, featuring advanced deployment strategies including **Blue/Green** and **Canary releases** with automated rollbacks based on Prometheus metrics.

### What Makes This Special?

- ✅ **Complete GitOps Implementation** - Declarative infrastructure and application management
- ✅ **Progressive Delivery** - Canary and Blue/Green deployment strategies
- ✅ **Event-Driven Automation** - Automated CI/CD triggers using Argo Events
- ✅ **Automated Analysis** - Integration with Prometheus for deployment validation
- ✅ **Multi-Environment Setup** - Staging and Production environments with Kustomize overlays

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         Developer                            │
└────────────────────┬────────────────────────────────────────┘
                     │ Git Push
                     ▼
┌─────────────────────────────────────────────────────────────┐
│                      Git Repository                          │
└────────────────────┬────────────────────────────────────────┘
                     │
        ┌────────────┴────────────┬──────────────────┐
        ▼                         ▼                   ▼
┌──────────────┐         ┌──────────────┐    ┌──────────────┐
│ Argo Events  │         │ Argo Workflow│    │   ArgoCD     │
│  (Webhook)   │────────▶│   (CI Jobs)  │    │  (GitOps)    │
└──────────────┘         └──────┬───────┘    └──────┬───────┘
                                │                    │
                                │ Update Image       │ Sync
                                ▼                    ▼
                         ┌──────────────────────────────┐
                         │    Kubernetes Cluster        │
                         │  ┌────────────────────────┐  │
                         │  │   Argo Rollouts        │  │
                         │  │ (Blue/Green, Canary)   │  │
                         │  └────────┬───────────────┘  │
                         │           │                  │
                         │           ▼                  │
                         │  ┌────────────────────────┐  │
                         │  │  Prometheus/Grafana    │  │
                         │  │   (Monitoring)         │  │
                         │  └────────────────────────┘  │
                         └──────────────────────────────┘
```

## ✨ Features

### Core Capabilities

- **GitOps with ArgoCD**: Automated synchronization of desired state from Git to Kubernetes
- **Progressive Delivery**: 
  - Blue/Green deployments for zero-downtime releases
  - Canary deployments with traffic shifting via Nginx Ingress
- **CI Pipeline**: Automated builds and tests using Argo Workflows
- **Event-Driven Automation**: Trigger pipelines on Git events using Argo Events
- **Automated Image Updates**: Argo Image Updater for continuous deployment
- **Metrics & Analysis**: Prometheus integration for automated rollout decisions
- **Multi-Environment**: Separate staging and production environments

## 📦 Prerequisites

### Hardware Requirements
- **RAM**: Minimum 8GB (16GB recommended)
- **CPU**: 4+ cores
- **Storage**: 10GB available disk space

### Software Requirements
- Docker Desktop or Docker Engine
- kubectl CLI
- Kind (Kubernetes in Docker)
- Kustomize

## 🛠️ Installation

### 1. Setup Local Kubernetes Cluster

```bash
# Install Docker (Ubuntu/Debian)
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg lsb-release -y
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo systemctl enable docker && sudo systemctl start docker
```

```bash
# Create Kind cluster
kind create cluster --config kind-config.yaml

# Verify cluster
kubectl cluster-info
kubectl get nodes
```

### 2. Install Kustomize

```bash
# Download and install Kustomize
curl -s "https://raw.githubusercontent.com/kubernetes-sigs/kustomize/master/hack/install_kustomize.sh" | bash
sudo mv kustomize /usr/local/bin/
```

### 3. Install ArgoCD

```bash
# Create namespace
kubectl create namespace argocd

# Install ArgoCD
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Wait for pods to be ready
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### 4. Configure ArgoCD Access

```bash
# Install password generator utility
sudo apt-get install apache2-utils -y

# Generate bcrypt hash for your password (replace 'mypassword' with your password)
htpasswd -bnBC 10 "" mypassword | tr -d ':\n'

# Delete old secret and create new one
kubectl -n argocd delete secret argocd-secret

# Create new secret with password "admin" (replace with your generated hash)
kubectl -n argocd create secret generic argocd-secret \
  --from-literal=admin.password=$(htpasswd -bnBC 10 "" admin | tr -d ':\n') \
  --from-literal=admin.passwordMtime=$(date -u +%FT%T%Z)

# Restart ArgoCD server
kubectl -n argocd rollout restart deployment argocd-server
kubectl -n argocd rollout status deployment argocd-server
```

### 5. Expose ArgoCD UI

```bash
# Patch service to NodePort
kubectl patch svc argocd-server -n argocd --patch \
  '{"spec": { "type": "NodePort", "ports": [ { "nodePort": 32100, "port": 443, "protocol": "TCP", "targetPort": 8080 } ] } }'

# Get node IP
kubectl get nodes -o wide

# Access ArgoCD at: http://<NODE_IP>:32100
# Username: admin
# Password: admin (or your custom password)
```

### 6. Install Argo Rollouts

```bash
# Install Argo Rollouts controller
kubectl create namespace argo-rollouts
kubectl apply -n argo-rollouts -f https://github.com/argoproj/argo-rollouts/releases/latest/download/install.yaml

# Install Argo Rollouts Dashboard (optional)
kubectl argo rollouts dashboard -p 3100
```

### 7. Install Argo Workflows

```bash
# Install Argo Workflows
kubectl create namespace argo
kubectl apply -n argo -f https://github.com/argoproj/argo-workflows/releases/latest/download/install.yaml
```

### 8. Deploy Applications

```bash
# Apply staging environment
kubectl apply -k staging/

# Apply production environment
kubectl apply -k prod/

# Verify deployments
kubectl get applications -n argocd
```

## 📁 Project Structure

```
argo-labs/
├── base/                      # Base Kubernetes manifests
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── ingress.yaml
│   └── kustomization.yaml
├── staging/                   # Staging overlay
│   ├── kustomization.yaml
│   ├── rollout.yaml
│   └── namespace.yaml
├── prod/                      # Production overlay
│   ├── kustomization.yaml
│   ├── rollout-canary.yaml
│   └── namespace.yaml
├── workflows/                 # Argo Workflows
│   └── vote-ci-workflow.yaml
├── events/                    # Argo Events
│   └── git-webhook.yaml
└── monitoring/               # Prometheus/Grafana configs
    ├── servicemonitor.yaml
    └── grafana-dashboard.yaml
```

## 🎯 Deployment Strategies

### Blue/Green Deployment (Staging)

Blue/Green deployment allows instant rollback by switching traffic between two identical environments.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    blueGreen:
      activeService: vote-app-active
      previewService: vote-app-preview
      autoPromotionEnabled: false
```

**Deploy to staging:**
```bash
kubectl argo rollouts promote vote-app -n staging
```

### Canary Deployment (Production)

Canary deployment gradually shifts traffic to the new version with automated analysis.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
spec:
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause: {duration: 30s}
      - setWeight: 50
      - pause: {duration: 30s}
      - setWeight: 100
```

**Monitor canary deployment:**
```bash
kubectl argo rollouts get rollout vote-app -n prod --watch
```

## 📊 Monitoring

### Prometheus Integration

The project includes automated analysis using Prometheus metrics:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: success-rate
spec:
  metrics:
  - name: success-rate
    successCondition: result >= 0.95
    provider:
      prometheus:
        address: http://prometheus:9090
        query: |
          sum(rate(http_requests_total{status="200"}[5m])) /
          sum(rate(http_requests_total[5m]))
```

**View metrics:**
```bash
# Port-forward Grafana
kubectl port-forward -n monitoring svc/grafana 3000:3000

# Access at http://localhost:3000
```

## 🔧 Commands Reference

### Essential Commands

```bash
# Check cluster status
kubectl cluster-info
kubectl get nodes

# View all pods across namespaces
kubectl get pods -A

# Get ArgoCD applications
kubectl get applications -n argocd

# Describe specific application
kubectl describe application vote-staging -n argocd

# View application YAML
kubectl get application vote-staging -n argocd -o yaml

# Switch namespace context
kubectl config set-context --current --namespace=staging

# View API resources
kubectl api-resources

# Check CRDs
kubectl get crds

# Build Kustomize manifests
kustomize build base/

# Get service details
kubectl get svc -n argocd

# View Argo Rollouts
kubectl argo rollouts list -n prod

# Watch rollout progress
kubectl argo rollouts get rollout vote-app -n prod --watch

# Promote rollout manually
kubectl argo rollouts promote vote-app -n prod

# Abort rollout
kubectl argo rollouts abort vote-app -n prod

# Submit Argo Workflow
argo submit -n argo --watch vote-ci-workflow.yaml

# List workflows
argo list -n argo

# Get workflow logs
argo logs -n argo <workflow-name>

# Get Docker container IP
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' kind-control-plane
```

### Debugging Commands

```bash
# Check pod logs
kubectl logs -f <pod-name> -n <namespace>

# Describe pod for events
kubectl describe pod <pod-name> -n <namespace>

# Execute into pod
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# Check rollout status
kubectl argo rollouts status vote-app -n prod

# View rollout history
kubectl argo rollouts history vote-app -n prod

# Check ArgoCD sync status
argocd app get vote-staging

# Force sync application
argocd app sync vote-staging --force
```

## 🐛 Troubleshooting

### ArgoCD Login Issues

```bash
# Verify secret exists
kubectl get secret argocd-secret -n argocd

# Check password hash
kubectl -n argocd get secret argocd-secret -o jsonpath='{.data.admin\.password}' | base64 --decode

# Reset to default password "password"
kubectl -n argocd patch secret argocd-secret \
  -p '{"stringData": {
    "admin.password": "$2a$10$rRyBsGSHK6.uc8fntPwVIuLVHgsAhAX7TcdrqW/RADU0uh7CaChLa",
    "admin.passwordMtime": "'$(date +%FT%T%Z)'"
  }}'
```

### Rollout Stuck in Progressing

```bash
# Check analysis runs
kubectl get analysisruns -n prod

# View analysis details
kubectl describe analysisrun <analysis-run-name> -n prod

# Manually promote if needed
kubectl argo rollouts promote vote-app -n prod
```

### Image Pull Errors

```bash
# Check image pull secrets
kubectl get secrets -n staging

# Verify image exists
docker pull <image-name>

# Check pod events
kubectl describe pod <pod-name> -n staging
```

## 🎓 Learning Resources

This project is based on the **Ultimate Argo Bootcamp** course covering:

- ✅ Building CI/CD pipelines with Argo tools
- ✅ Implementing Blue/Green and Canary deployments
- ✅ GitOps principles with ArgoCD
- ✅ Workflow orchestration with Argo Workflows
- ✅ Event-driven automation with Argo Events
- ✅ Monitoring with Prometheus and Grafana

**Course Details:**
- Rating: 4.2/5 ⭐
- Students: 9,076+
- Duration: 8 hours
- Level: Intermediate

## 🤝 Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/AmazingFeature`)
3. Commit your changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

## 📝 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- [Argo Project](https://argoproj.github.io/) for the amazing suite of tools
- [Kubernetes](https://kubernetes.io/) community
- Original repository: [sfd226/argo-labs](https://github.com/sfd226/argo-labs)

## 📧 Contact

**Abhi Mishra** - [@Abhi-mishra998](https://github.com/Abhi-mishra998)

Project Link: [https://github.com/Abhi-mishra998/argo-labs](https://github.com/Abhi-mishra998/argo-labs)

---

⭐ **If you found this project helpful, please consider giving it a star!** ⭐

---

**Built with ❤️ using Kubernetes and the Argo Suite**
