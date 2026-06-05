# Monitoring

> **Prototype Notice:** This platform is currently in prototype status. Features and APIs may change between releases.

---

## Overview

The platform ships with a pre-configured observability stack:

| Tool | Purpose | Internal URL |
| ------ | --------- | ------------- |
| **Prometheus** | Metrics collection & alerting | `https://prometheus.<cluster>.internal.example.com` |
| **Grafana** | Dashboards & visualisation | `https://grafana.<cluster>.internal.example.com` |
| **Loki** | Log aggregation (via Grafana) | Available as a data source within Grafana |

---

## Exposing Metrics from Your Service

The platform expects services to expose Prometheus metrics on `/metrics` at the service port.

### Enable via Helm Chart

```yaml
# values.yaml — kdp/microservice chart
metrics:
  enabled: true
    port: 8080
    path: /metrics
```

### Annotate Your Pod Spec Directly

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
    prometheus.io/path: "/metrics"
```

---

## Pre-built Grafana Dashboards

The platform team maintains a dashboard library. Import from the Grafana UI under **Dashboards → Browse → KDP Library**:

| Dashboard | Description |
| ----------- | ------------- |
| **KDP: Service Overview** | Request rate, error rate, latency (RED metrics) |
| **KDP: JVM / Node.js Runtime** | Heap, GC, event loop metrics |
| **KDP: Pod Resources** | CPU/memory usage vs. limits |
| **KDP: HPA Status** | Replica counts and scaling events |

---

## Alerting

Alerts are defined as `PrometheusRule` resources. Add custom alerts for your service:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: my-service-alerts
  namespace: my-team-prod
  labels:
    release: kube-prometheus-stack   # required for Prometheus to pick up the rule
spec:
  groups:
    - name: my-service
      rules:
        - alert: HighErrorRate
          expr: |
            rate(http_requests_total{service="my-service", status=~"5.."}[5m])
            /
            rate(http_requests_total{service="my-service"}[5m]) > 0.05
          for: 2m
          labels:
            severity: warning
            team: my-team
          annotations:
            summary: "High error rate on my-service"
            description: "Error rate is {{ $value | humanizePercentage }} over the last 5 minutes."
```

Alerts route to PagerDuty, Slack, or email based on the `team` label. Configure your routing in the platform portal under **Alerting → Notification Policies**.

---

## Viewing Logs

### Query via Grafana (Loki)

Go to **Grafana → Explore → Loki** and use LogQL:

```logql
{namespace="my-team-prod", app="my-service"} |= "ERROR"
```

### Stream Logs via kubectl

```bash
# Stream logs from a deployment
kubectl logs -n my-team-prod deploy/my-service -f --tail=100

# Stream logs from all pods matching a label
kubectl logs -n my-team-prod -l app=my-service --all-containers=true
```

---

## Default Platform Alerts

The platform includes built-in alerts for common failure conditions:

- Pod crash-looping (`CrashLoopBackOff`)
- High pod restart count
- Node memory pressure
- Node disk pressure
- Persistent volume claim unbound

These fire into the `#platform-alerts` Slack channel by default.

---

*Documentation version: 0.1.0 — May 2026*
*Maintained by the Platform Engineering team.*
