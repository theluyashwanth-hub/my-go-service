[README.md](https://github.com/user-attachments/files/26488442/README.md)
# 🚀 Go Service — Helm Chart on Private S3 Repository

> A production-grade DevOps pipeline: Go microservice → Docker → Kubernetes (K3s) → Helm → AWS S3 + ECR → fully automated CI/CD via GitHub Actions.

[![CI/CD Pipeline](https://github.com/theluyashwanth-hub/my-go-service/actions/workflows/release.yaml/badge.svg)](https://github.com/theluyashwanth-hub/my-go-service/actions/workflows/release.yaml)
![Go Version](https://img.shields.io/badge/Go-1.21-blue)
![Helm](https://img.shields.io/badge/Helm-3.x-informational)
![K3s](https://img.shields.io/badge/Kubernetes-K3s-orange)
![AWS](https://img.shields.io/badge/AWS-ECR%20%7C%20S3-yellow)
![License](https://img.shields.io/badge/license-MIT-green)

---

## 📌 Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Tech Stack](#tech-stack)
- [Project Structure](#project-structure)
- [Prerequisites](#prerequisites)
- [Local Setup](#local-setup)
- [AWS Infrastructure Setup](#aws-infrastructure-setup)
- [Kubernetes Setup (K3s)](#kubernetes-setup-k3s)
- [Helm Chart](#helm-chart)
- [CI/CD Pipeline](#cicd-pipeline)
- [Deploying the Service](#deploying-the-service)
- [Environment Configuration](#environment-configuration)
- [Verify the Deployment](#verify-the-deployment)
- [Key Commands Reference](#key-commands-reference)
- [Why Each Tool Was Chosen](#why-each-tool-was-chosen)
- [Author](#author)

---

## Overview

This project solves the real-world problem of **manual, error-prone deployments** by implementing a fully automated DevOps pipeline.

**The problem:** Traditional deployments require developers to manually build images, push to registries, update configuration files, and restart services — every single release. One missed step breaks production.

**The solution:** A single `git tag` push triggers an automated pipeline that:
1. Builds a Docker image and pushes it to **AWS ECR**
2. Packages the Helm chart and publishes it to a **private AWS S3** chart museum
3. Makes the new version available for immediate deployment to **Kubernetes (K3s)**

**Zero manual steps. Zero human error.**

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Developer Laptop                          │
│   Write code → git tag v1.0.0 → git push origin v1.0.0         │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                     GitHub Repository                            │
│   Source code + Helm chart + CI/CD workflow stored here         │
└──────────────────────────┬──────────────────────────────────────┘
                           │  tag push triggers workflow
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                   GitHub Actions Pipeline                        │
│                                                                  │
│  Step 1: docker build + docker push  ──────► AWS ECR            │
│  Step 2: helm package + helm s3 push ──────► AWS S3             │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              AWS EC2 — K3s Kubernetes Cluster                    │
│                                                                  │
│  ┌──────────────────────┐  ┌───────────────────────────────┐   │
│  │  namespace: dev       │  │  namespace: production         │   │
│  │  - 1 replica          │  │  - 2 replicas                  │   │
│  │  - LOG_LEVEL=debug    │  │  - LOG_LEVEL=info              │   │
│  │  - image: latest      │  │  - image: v1.0.0               │   │
│  └──────────────────────┘  └───────────────────────────────┘   │
│                                                                  │
│  Pods pull image ◄── AWS ECR                                     │
│  Helm pulls chart ◄── AWS S3                                     │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow Summary

```
Code Change → Git Tag → GitHub Actions → ECR (image) + S3 (chart)
                                       → helm upgrade → K3s → Live App
```

---

## Tech Stack

| Layer | Technology | Purpose |
|---|---|---|
| **Application** | Go 1.21 | HTTP microservice, compiles to single binary |
| **Containerization** | Docker (multi-stage) | Package app into a ~12MB container image |
| **Orchestration** | K3s (Kubernetes) | Run, scale, and manage containers |
| **Package Management** | Helm 3 | Template and deploy Kubernetes manifests |
| **Chart Storage** | AWS S3 + helm-s3 | Private versioned Helm chart museum |
| **Image Registry** | AWS ECR | Private Docker image registry |
| **CI/CD** | GitHub Actions | Automate build, push, package on tag push |
| **Cloud** | AWS EC2 | Host the Kubernetes cluster |
| **Access Control** | AWS IAM | Least-privilege credentials for pipeline |

---

## Project Structure

```
my-go-service/
│
├── main.go                          # Go HTTP microservice
├── go.mod                           # Go module definition
├── Dockerfile                       # Multi-stage container build
├── .gitignore                       # Files excluded from git
├── README.md                        # This file
│
├── helm/
│   └── go-service/
│       ├── Chart.yaml               # Chart metadata (name, version)
│       ├── values.yaml              # Default configuration values
│       ├── values-dev.yaml          # Dev environment overrides
│       ├── values-prod.yaml         # Production environment overrides
│       └── templates/
│           ├── deployment.yaml      # Kubernetes Deployment template
│           ├── service.yaml         # Kubernetes Service template
│           └── configmap.yaml       # Kubernetes ConfigMap template
│
└── .github/
    └── workflows/
        └── release.yaml             # GitHub Actions CI/CD pipeline
```

---

## Prerequisites

### Local Machine
- [Git](https://git-scm.com/)
- [Go 1.21+](https://go.dev/dl/)
- [Docker Desktop](https://docs.docker.com/get-docker/)
- [Helm 3](https://helm.sh/docs/intro/install/)
- [AWS CLI v2](https://aws.amazon.com/cli/)
- [kubectl](https://kubernetes.io/docs/tasks/tools/)

### AWS Account
- IAM user with `AmazonS3FullAccess` + `AmazonEC2ContainerRegistryFullAccess`
- S3 bucket created (e.g. `my-helm-charts-yourname`)
- ECR private repository created (e.g. `go-service`)

### GitHub
- Repository with the following secrets configured:

| Secret | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | IAM user access key |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key |
| `AWS_REGION` | e.g. `ap-south-1` |
| `ECR_REGISTRY` | e.g. `123456789.dkr.ecr.ap-south-1.amazonaws.com` |
| `S3_BUCKET` | e.g. `my-helm-charts-yourname` |

---

## Local Setup

```bash
# 1. Clone the repository
git clone https://github.com/theluyashwanth-hub/my-go-service.git
cd my-go-service

# 2. Install Go dependencies
go mod download

# 3. Run the app locally
go run main.go

# 4. Test the health endpoint
curl http://localhost:8080/health
# Expected: OK - service is healthy
```

---

## AWS Infrastructure Setup

### 1. Create S3 Bucket

```bash
aws s3 mb s3://my-helm-charts-yourname --region ap-south-1
```

### 2. Create ECR Repository

```bash
aws ecr create-repository \
  --repository-name go-service \
  --region ap-south-1
```

### 3. Install helm-s3 Plugin

```bash
helm plugin install https://github.com/hypnoglow/helm-s3.git
```

### 4. Initialize S3 as Helm Repository

```bash
helm s3 init s3://my-helm-charts-yourname/charts
helm repo add my-repo s3://my-helm-charts-yourname/charts
helm repo list
```

---

## Kubernetes Setup (K3s)

### Install K3s on EC2

```bash
# Install K3s (single command)
curl -sfL https://get.k3s.io | sh -

# Configure kubectl
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown ubuntu:ubuntu ~/.kube/config
export KUBECONFIG=~/.kube/config

# Verify cluster is running
kubectl get nodes
```

### Create Namespaces

```bash
kubectl create namespace dev
kubectl create namespace production
kubectl get namespaces
```

### Create ECR Image Pull Secret

```bash
TOKEN=$(aws ecr get-login-password --region ap-south-1)
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

for NS in dev production; do
  kubectl create secret docker-registry ecr-secret \
    --docker-server=$ACCOUNT.dkr.ecr.ap-south-1.amazonaws.com \
    --docker-username=AWS \
    --docker-password=$TOKEN \
    --namespace=$NS
done
```

> **Note:** ECR tokens expire after 12 hours. Re-run the above if pods show `ImagePullBackOff`.

---

## Helm Chart

### Chart Structure

The Helm chart uses **values override pattern** — `values.yaml` holds defaults, environment-specific files only override what changes:

```
values.yaml          ← base defaults
values-dev.yaml      ← dev overrides  (merged on top of values.yaml)
values-prod.yaml     ← prod overrides (merged on top of values.yaml)
```

### Validate the Chart

```bash
# Lint for errors
helm lint helm/go-service/

# Preview generated YAML for production
helm template go-service helm/go-service/ \
  -f helm/go-service/values-prod.yaml

# Package manually
helm package helm/go-service/

# Push to S3
helm s3 push go-service-0.1.0.tgz my-repo
```

---

## CI/CD Pipeline

### Trigger

The pipeline runs **only on version tag pushes**. Ordinary commits to `main` do NOT trigger it.

```bash
git tag v1.0.0
git push origin v1.0.0
```

### Pipeline Steps

```
1. Checkout code
2. Configure AWS credentials (from GitHub Secrets)
3. Login to Amazon ECR
4. docker build → docker push to ECR (tagged with git tag)
5. Install Helm + helm-s3 plugin
6. Update Chart.yaml appVersion to match git tag
7. helm package → helm s3 push to S3
```

### View Pipeline Runs

```
https://github.com/theluyashwanth-hub/my-go-service/actions
```

---

## Deploying the Service

### Deploy to Dev

```bash
helm repo update

helm upgrade --install go-service my-repo/go-service \
  -f helm/go-service/values-dev.yaml \
  --namespace dev \
  --set image.tag=v1.0.0
```

### Deploy to Production

```bash
helm upgrade --install go-service my-repo/go-service \
  -f helm/go-service/values-prod.yaml \
  --namespace production \
  --set image.tag=v1.0.0
```

### Rollback to Previous Version

```bash
# View release history
helm history go-service -n production

# Rollback to revision 1
helm rollback go-service 1 -n production
```

---

## Environment Configuration

| Setting | Dev | Production |
|---|---|---|
| `replicaCount` | 1 | 2 |
| `image.tag` | `latest` | `v1.0.0` |
| `LOG_LEVEL` | `debug` | `info` |
| `cpu limit` | `100m` | `200m` |
| `memory limit` | `128Mi` | `256Mi` |

---

## Verify the Deployment

```bash
# Check pods are running
kubectl get pods -n production
kubectl get pods -n dev

# Check services
kubectl get svc -n production

# View deployment details
kubectl describe deployment go-service -n production

# Check pod logs
kubectl logs -n production -l app=go-service

# Test the live endpoint
kubectl port-forward svc/go-service 8080:8080 -n production &
sleep 2
curl http://localhost:8080/health
# Expected: OK - service is healthy
```

---

## Key Commands Reference

```bash
# Cluster status
kubectl get nodes
kubectl get pods -A

# Helm operations
helm list -n production
helm history go-service -n production
helm rollback go-service 1 -n production

# Force pod restart (pick up new image)
kubectl rollout restart deployment/go-service -n production
kubectl rollout status deployment/go-service -n production

# Check ECR images
aws ecr list-images --repository-name go-service --region ap-south-1

# Check S3 charts
aws s3 ls s3://my-helm-charts-yourname/charts/

# Refresh ECR pull secret (every 12 hours)
TOKEN=$(aws ecr get-login-password --region ap-south-1)
ACCOUNT=$(aws sts get-caller-identity --query Account --output text)
kubectl delete secret ecr-secret -n production --ignore-not-found
kubectl create secret docker-registry ecr-secret \
  --docker-server=$ACCOUNT.dkr.ecr.ap-south-1.amazonaws.com \
  --docker-username=AWS --docker-password=$TOKEN -n production
```

---

## Why Each Tool Was Chosen

**Go** — Compiles to a single static binary. No runtime dependencies. Docker image is ~12MB vs ~300MB for Java. The standard language for cloud-native infrastructure (Kubernetes, Docker, Terraform are all written in Go).

**Docker multi-stage build** — Stage 1 compiles. Stage 2 runs only the compiled binary. Reduces attack surface, speeds up image pulls, lowers ECR storage cost.

**K3s** — CNCF-certified Kubernetes that runs in 512MB RAM. Every `kubectl` command and Helm chart works identically to full Kubernetes. Perfect for single-node EC2 deployments without sacrificing compatibility.

**Helm** — Eliminates duplicate Kubernetes YAML across environments. One chart, multiple value files. Provides versioning, rollback, and upgrade capabilities out of the box.

**AWS S3 + helm-s3** — Private chart museum with 99.999999999% durability at near-zero cost. No chart server to maintain. IAM-controlled access. Industry-standard pattern used by engineering teams at scale.

**AWS ECR** — Co-located with EC2 in the same AWS region — fast image pulls. Integrated with IAM — no separate credentials. Automatic vulnerability scanning on push.

**GitHub Actions** — Pipeline lives in the same repo as code — version controlled together. No separate CI server to maintain. Free for public repositories.

---

## Author

**Yashwanth** — [GitHub](https://github.com/theluyashwanth-hub)

> *"Built to understand DevOps from scratch — not just use it."*

---

*Project Category: Kubernetes / DevOps | Difficulty: Medium | Stack: Go · Docker · K3s · Helm · AWS*
