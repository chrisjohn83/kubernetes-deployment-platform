# Developer Quick Start Guide

> **Prototype Notice:** This platform is currently in prototype status. Features and APIs may change between releases. Please report issues to your platform team.

Welcome to the Kubernetes Deployment Platform. This guide gets you from zero to a running microservice in under 30 minutes.

---

## Prerequisites

Before you begin, make sure you have the following installed on your local machine:

- **kubectl** v1.28+ — [Install guide](https://kubernetes.io/docs/tasks/tools/)
- **Helm** v3.12+ — [Install guide](https://helm.sh/docs/intro/install/)
- **Docker** v24+ (for building images)
- Platform CLI (`kdp-cli`) — install via `brew install kdp-cli` or download from the internal portal
- Access credentials to the container registry (request from your platform admin)

---

## Step 1 — Authenticate

Log in to the platform and configure your local kubeconfig:

```bash
kdp-cli login
kdp-cli cluster use <your-team-cluster>
kubectl get nodes   # verify connectivity
```

---

## Step 2 — Create Your Namespace

Each team works in an isolated namespace. Create one for your project:

```bash
kubectl create namespace my-service
kubectl label namespace my-service team=my-team env=dev
```

---

## Step 3 — Build and Push Your Container Image

```bash
docker build -t registry.internal.example.com/my-team/my-service:0.1.0 .
docker push registry.internal.example.com/my-team/my-service:0.1.0
```

---

## Step 4 — Deploy with Helm

Use the platform's base Helm chart to deploy your service:

```bash
helm repo add kdp https://charts.internal.example.com
helm repo update

helm install my-service kdp/microservice \
  --namespace my-service \
  --set image.repository=registry.internal.example.com/my-team/my-service \
  --set image.tag=0.1.0 \
  --set service.port=8080
```

---

## Step 5 — Verify the Deployment

```bash
kubectl get pods -n my-service
kubectl get svc -n my-service
kubectl logs -n my-service deploy/my-service --tail=50
```

Your service is live once all pods show `Running` status.

---

## Next Steps

| Topic | File |
|-------|------|
| Cluster Setup | `02-cluster-setup.md` |
| Helm Charts | `03-helm-charts.md` |
| CI/CD Pipelines | `04-cicd-pipelines.md` |
| Autoscaling | `05-autoscaling.md` |
| Monitoring | `06-monitoring.md` |
| Rollbacks | `07-rollbacks.md` |
| Secrets Management | `08-secrets-management.md` |

---

*Documentation version: 0.1.0 — May 2026*
*Maintained by the Platform Engineering team.*
