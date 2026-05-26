# Secrets Management

> **Prototype Notice:** This platform is currently in prototype status. Features and APIs may change between releases.

---

## Overview

Secrets are managed through **HashiCorp Vault** (for storage and rotation) and synced into Kubernetes using the **External Secrets Operator (ESO)**. Do not store credentials, API keys, or certificates in Helm values files, environment variables committed to Git, or ConfigMaps.

---

## How It Works

```
Vault (source of truth)
    ↓  External Secrets Operator syncs on schedule
Kubernetes Secret (native)
    ↓  Referenced by pod spec
Running Pod (env var or volume mount)
```

1. A secret is stored in Vault at a well-known path.
2. An `ExternalSecret` resource in your namespace declares which Vault path to fetch.
3. The External Secrets Operator syncs the value into a native Kubernetes `Secret`.
4. Your pod references the Kubernetes `Secret` — no application code changes required.

---

## Storing a Secret in Vault

Access Vault via the platform portal or CLI:

```bash
vault login -method=oidc   # authenticate with your SSO credentials

vault kv put secret/my-team/my-service/prod \
  DB_PASSWORD="s3cur3P@ssw0rd" \
  API_KEY="abc123xyz"
```

Vault paths follow the convention:

```
secret/<team>/<service>/<environment>
```

---

## Creating an ExternalSecret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-service-secrets
  namespace: my-team-prod
spec:
  refreshInterval: 1h          # how often to re-sync from Vault
  secretStoreRef:
    name: vault-backend        # ClusterSecretStore provisioned by the platform
    kind: ClusterSecretStore
  target:
    name: my-service-secrets   # name of the resulting Kubernetes Secret
    creationPolicy: Owner
  data:
    - secretKey: DB_PASSWORD   # key in the Kubernetes Secret
      remoteRef:
        key: secret/my-team/my-service/prod
        property: DB_PASSWORD
    - secretKey: API_KEY
      remoteRef:
        key: secret/my-team/my-service/prod
        property: API_KEY
```

Apply and verify:

```bash
kubectl apply -f external-secret.yaml -n my-team-prod
kubectl get externalsecret my-service-secrets -n my-team-prod
```

---

## Consuming Secrets in Your Deployment

### Inject All Keys as Environment Variables

```yaml
spec:
  containers:
    - name: my-service
      envFrom:
        - secretRef:
            name: my-service-secrets
```

### Reference Individual Keys

```yaml
env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: my-service-secrets
        key: DB_PASSWORD
```

---

## Secret Rotation

When a secret is updated in Vault, the External Secrets Operator re-syncs it on the next refresh cycle (default: 1 hour). To force an immediate sync:

```bash
kubectl annotate externalsecret my-service-secrets \
  -n my-team-prod \
  force-sync=$(date +%s) --overwrite
```

After rotation, restart the deployment to pick up new environment variable values:

```bash
kubectl rollout restart deployment/my-service -n my-team-prod
```

---

## TLS Certificates

TLS certificates are managed by **cert-manager** and provisioned automatically for any Ingress with the appropriate annotation:

```yaml
# For internet-facing services
annotations:
  cert-manager.io/cluster-issuer: "letsencrypt-prod"

# For internal services
annotations:
  cert-manager.io/cluster-issuer: "internal-ca"
```

---

## Best Practices

- **Grant Vault policies with least privilege** — each service should only have read access to its own secret path.
- **Never log secrets.** Audit your application's startup logs and error handlers.
- **Rotate secrets on a schedule.** The platform supports automated Vault lease renewal for database credentials.
- **Do not bake secrets into Docker images.** Treat images as public artifacts.
- **Use separate Vault paths per environment** (`/dev`, `/staging`, `/prod`) so a compromised dev credential cannot access production.
- **Audit Vault access logs** periodically via the platform portal under **Security → Vault Audit**.

---

*Documentation version: 0.1.0 — May 2026*
*Maintained by the Platform Engineering team.*
