# Deployment Specification – Biteasy Platform

> Status: Draft v0.1 – 2025-06-29  
> Author: DevOps Architect  
> Scope: Container, CI/CD, and cloud topology for production & staging.

---

## 0  Objectives

* Provide reproducible builds via **multi-stage Dockerfile** (Python 3.12-slim, poetry).  
* Use **GitHub Actions** to lint, test, build, push to GHCR.  
* Deploy via **Kubernetes** (EKS) with Helm charts; one release per environment.  
* Leverage **Prometheus Operator** & **Grafana** for metrics.  
* Secrets managed by **AWS Secrets Manager**; injected with External Secrets.

---

## 1  Docker Image

| Stage | Purpose | Size Target |
|-------|---------|------------|
| base | python:3.12-slim + system deps (gcc, blas) | < 350 MB |
| builder | poetry install –no-dev + cvxpy & torch wheels | 800 MB |
| runtime | copy venv + src, drop build cache | < 600 MB |

Entrypoint `/usr/local/bin/biteasy` invoking orchestrator.

---

## 2  GitHub Actions Workflow

```yaml
name: CI
on:
  push:
    branches: [main]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with: {python-version: '3.12'}
      - run: pip install poetry && poetry install -E torch
      - run: poetry run pytest -q
  docker:
    needs: test
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
      - name: Build & Push
        uses: docker/build-push-action@v5
        with:
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ghcr.io/org/biteasy:latest
```

---

## 3  Kubernetes Topology

* Namespace `biteasy-{env}` (prod/stage).  
* Deployments: `orchestrator`, `risk-updater`, `alpaca-adapter`, `prometheus` sidecar.  
* HPA on CPU (target 70 %).  
* Secrets mounted via CSI driver.  
* PersistentVolume for logs (/var/log/biteasy).

---

## 4  Helm Values Snippet

```yaml
orchestrator:
  image: ghcr.io/org/biteasy:latest
  replicaCount: 2
  resources:
    limits: {cpu: '1', memory: 2Gi}
    requests: {cpu: 200m, memory: 512Mi}
  env:
    - name: CONFIG_PATH
      value: /config/agents.yaml
``` 

---

## 5  Milestones

| ID | Task | ETA |
|----|------|-----|
| DP-1 | Multi-stage Dockerfile + local run | W1D1 |
| DP-2 | GitHub Actions test+build+push | W1D2 |
| DP-3 | Helm chart + dev namespace deploy | W1D3 |
| DP-4 | Prom/Grafana stack integration | W1D4 |

---

*End of deployment spec.* 