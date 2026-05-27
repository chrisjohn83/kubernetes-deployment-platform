# Cluster Setup

> **Prototype Notice:** This platform is currently in prototype status. Features and APIs may change between releases.

---

## Overview

The platform manages one or more Kubernetes clusters provisioned on your cloud provider. Each cluster is segmented by environment (development, staging, production) and isolated using namespaces and RBAC policies.

---

## Cluster Tiers

| Tier | Purpose | Node Size | Autoscaling |
|------|---------|-----------|-------------|
| `dev` | Development & testing | 2–4 nodes, 4 vCPU / 16 GB | Off |
| `staging` | Pre-production validation | 3–8 nodes, 8 vCPU / 32 GB | On |
| `prod` | Live production workloads | 5–20 nodes, 16 vCPU / 64 GB | On |

---

## Requesting a Cluster

Clusters are provisioned by the platform team. Submit a request through the internal portal with:

- Team name and cost centre
- Target cloud region
- Expected workload type (stateless API, stateful database sidecar, batch job, etc.)
- Required tier

Provisioning typically takes 15–30 minutes.

---

## Connecting to a Cluster

Once provisioned, credentials are distributed via the platform CLI:

```bash
kdp-cli cluster list               # list clusters you have access to
kdp-cli cluster use prod-us-east-1 # merge kubeconfig entry
kubectl cluster-info               # confirm connection
```

---

## Namespace Conventions

Teams are allocated one namespace per cluster tier:

```
<team-name>-dev
<team-name>-staging
<team-name>-prod
```

Resource quotas are applied at the namespace level. To view your current quota:

```bash
kubectl describe resourcequota -n my-team-prod
```

---

## Node Pools

The platform supports multiple node pools optimised for different workload types. Specify the pool via a node selector in your deployment manifest:

```yaml
spec:
  nodeSelector:
    kdp.io/pool: compute   # options: general, compute, memory, gpu
```

---

## RBAC and Access Control

Access is role-based. Default roles are:

| Role | Permissions |
|------|-------------|
| **viewer** | Read-only access to namespace resources |
| **developer** | Deploy, scale, and read logs within the namespace |
| **admin** | Full namespace control, including secrets |

Request role changes through the internal access management portal. The platform team does not grant cluster-level admin to individual users in production.

---

*Documentation version: 0.1.0 — May 2026*
*Maintained by the Platform Engineering team.*
