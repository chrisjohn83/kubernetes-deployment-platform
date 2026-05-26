# Autoscaling

> **Prototype Notice:** This platform is currently in prototype status. Features and APIs may change between releases.

---

## Overview

The platform supports two autoscaling mechanisms:

- **Horizontal Pod Autoscaler (HPA)** — scales the number of pod replicas
- **Vertical Pod Autoscaler (VPA)** — right-sizes resource requests per pod

Cluster-level node autoscaling is managed by the platform team and is enabled by default on staging and production clusters.

---

## Horizontal Pod Autoscaler (HPA)

HPA scales the number of pod replicas based on CPU utilisation, memory utilisation, or custom metrics.

### Enable via Helm Chart

```yaml
# values.yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

### Apply Directly with kubectl

```bash
kubectl autoscale deployment my-service \
  --namespace my-team-prod \
  --min=2 --max=10 \
  --cpu-percent=70
```

### Check HPA Status

```bash
kubectl get hpa -n my-team-prod
kubectl describe hpa my-service -n my-team-prod
```

---

## Custom Metrics Autoscaling (KEDA)

For scaling on application-specific metrics (e.g., queue depth, request latency), the platform integrates with **KEDA** (Kubernetes Event-driven Autoscaling).

```yaml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: my-service-scaledobject
  namespace: my-team-prod
spec:
  scaleTargetRef:
    name: my-service
  minReplicaCount: 1
  maxReplicaCount: 20
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus.monitoring.svc:9090
        metricName: http_requests_in_flight
        threshold: "50"
        query: sum(http_requests_in_flight{service="my-service"})
```

---

## Vertical Pod Autoscaler (VPA)

VPA automatically adjusts resource requests and limits based on observed usage. It is recommended for right-sizing during initial rollout.

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-service-vpa
  namespace: my-team-prod
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-service
  updatePolicy:
    updateMode: "Off"   # "Off" = recommendations only; "Auto" = live adjustment
```

> **Warning:** Do not run HPA (on CPU/memory) and VPA in `Auto` mode simultaneously on the same deployment — they will conflict. Use HPA for replica count and VPA in `Off` mode for recommendations only.

---

## Autoscaling Tips

- Always set `minReplicas: 2` in production to ensure high availability during scale-down events.
- Set conservative `maxReplicas` limits to protect downstream dependencies and avoid runaway cost.
- Tune HPA cooldown periods with `--horizontal-pod-autoscaler-downscale-stabilization` if you see flapping.
- Check HPA events for scaling activity: `kubectl describe hpa my-service -n my-team-prod`

---

*Documentation version: 0.1.0 — May 2026*
*Maintained by the Platform Engineering team.*
