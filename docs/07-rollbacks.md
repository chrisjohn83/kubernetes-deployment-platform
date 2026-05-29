# Rollbacks

> **Prototype Notice:** This platform is currently in prototype status. Features and APIs may change between releases.

---

## Overview

Every Helm upgrade is recorded in the cluster as a versioned release revision. If a deployment introduces a regression, you can roll back to any prior revision in seconds.

---

## Viewing Release History

```bash
helm history my-service -n my-team-prod
```

Example output:

```bash
REVISION  UPDATED                   STATUS     CHART               APP VERSION  DESCRIPTION
1         2026-05-01 09:12:00 UTC   superseded microservice-2.4.1  1.0.0        Install complete
2         2026-05-15 14:30:00 UTC   superseded microservice-2.4.1  1.1.0        Upgrade complete
3         2026-05-26 11:00:00 UTC   deployed   microservice-2.4.1  1.2.0        Upgrade complete

```

---

## Rolling Back with Helm

```bash
# Roll back to the immediately previous revision
helm rollback my-service -n my-team-prod

# Roll back to a specific revision number
helm rollback my-service 2 -n my-team-prod
```

Helm re-applies the exact Kubernetes manifests from the target revision. The rollback itself is recorded as a new revision entry.

---

## Rolling Back with kubectl

If you deployed directly with `kubectl` rather than Helm:

```bash
# Undo the most recent rollout
kubectl rollout undo deployment/my-service -n my-team-prod

# Undo to a specific revision
kubectl rollout undo deployment/my-service -n my-team-prod --to-revision=3

# View rollout history
kubectl rollout history deployment/my-service -n my-team-prod
```

---

## Monitoring a Rollback

Watch pod replacement in real time:

```bash
kubectl rollout status deployment/my-service -n my-team-prod -w
```

---

## Automated Rollback in CI/CD

The platform's reusable GitHub Action supports automatic rollback on health-check failure:

```yaml
with:
  rollback_on_failure: true
  health_check_url: https://my-service.internal.example.com/health
  health_check_timeout: 120   # seconds to wait before declaring failure
```

---

## Atomic Upgrades

Always use `--atomic` during automated Helm upgrades. This automatically rolls back if any resource fails to become ready within the timeout:

```bash
helm upgrade my-service kdp/microservice \
  --atomic \
  --timeout 5m \
  --namespace my-team-prod \
  --values values.yaml
```

---

## Best Practices

- **Tag every image with an immutable identifier** (Git SHA or semantic version) so rollbacks retrieve the exact same binary.
- **Test the rollback path** in staging before every major release.
- **Retain at least 10 Helm revisions** (the default) to give yourself room to roll back across several releases.
- **Use `--atomic`** in CI to prevent partially broken releases from sitting in a degraded state.
- **Set a health check endpoint** (`/health` or `/readiness`) in your service so rollback automation has something concrete to test against.

---

*Documentation version: 0.1.0 — May 2026*
*Maintained by the Platform Engineering team.*
