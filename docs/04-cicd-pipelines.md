# CI/CD Pipelines

> **Prototype Notice:** This platform is currently in prototype status. Features and APIs may change between releases.

---

## Overview

The platform integrates with GitHub Actions and GitLab CI out of the box. The recommended pipeline pattern is:

```text
build → test → push image → deploy to dev → promote to staging → promote to production
```

---

## GitHub Actions Integration

The platform provides reusable workflow actions published at `kdp-actions/deploy`. A typical pipeline:

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: |
          docker build -t $IMAGE_NAME:$GITHUB_SHA .
          docker push $IMAGE_NAME:$GITHUB_SHA
        env:
          IMAGE_NAME: registry.internal.example.com/my-team/my-service

  deploy-dev:
    needs: build-and-test
    if: github.event_name == 'push'
    uses: kdp-actions/deploy/.github/workflows/helm-deploy.yml@v1
    with:
      cluster: dev-us-east-1
      namespace: my-team-dev
      release: my-service
      chart: kdp/microservice
      image_tag: ${{ github.sha }}
    secrets:
      KUBECONFIG: ${{ secrets.KDP_DEV_KUBECONFIG }}

  deploy-prod:
    needs: deploy-dev
    environment: production           # requires manual approval in GitHub
    uses: kdp-actions/deploy/.github/workflows/helm-deploy.yml@v1
    with:
      cluster: prod-us-east-1
      namespace: my-team-prod
      release: my-service
      chart: kdp/microservice
      image_tag: ${{ github.sha }}
    secrets:
      KUBECONFIG: ${{ secrets.KDP_PROD_KUBECONFIG }}
```

---

## GitLab CI Integration

```yaml
# .gitlab-ci.yml
stages:
  - build
  - deploy-dev
  - deploy-prod

build:
  stage: build
  script:
    - docker build -t $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA

deploy-dev:
  stage: deploy-dev
  script:
    - kdp-cli deploy --cluster dev-us-east-1 --namespace my-team-dev
        --release my-service --image-tag $CI_COMMIT_SHA
  only:
    - main

deploy-prod:
  stage: deploy-prod
  script:
    - kdp-cli deploy --cluster prod-us-east-1 --namespace my-team-prod
        --release my-service --image-tag $CI_COMMIT_SHA
  when: manual
  only:
    - main
```

---

## Pipeline Best Practices

- **Never deploy untagged images to production.** Use the Git SHA or a semantic version tag.
- **Gate production deployments behind a manual approval step** or a change-management ticket reference.
- **Run `helm diff`** (using the `helm-diff` plugin) before every upgrade to preview changes before they apply.
- **Store kubeconfig credentials as encrypted CI secrets**, never in source code.
- **Keep dev and prod pipelines symmetric** — what runs in dev is exactly what runs in prod, only the cluster and values file differ.

---

## Viewing Deployment History

```bash
helm history my-service -n my-team-prod
```

---

*Documentation version: 0.1.0 — May 2026*
*Maintained by the Platform Engineering team.*
