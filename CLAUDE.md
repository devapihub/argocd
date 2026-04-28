# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a GitOps repository managed by ArgoCD. It contains Kubernetes manifests for the `devapihub` platform. Changes pushed to `main` are automatically applied to the cluster via ArgoCD.

## Applying Changes

```bash
# Validate a manifest before committing
kubectl apply --dry-run=client -f <manifest.yaml>

# Check sync status in ArgoCD
argocd app list
argocd app sync <app-name>
```

## Image Bump Workflow

The primary task is updating container image tags when a new image is built. The image field is the only field that changes in most commits:

```yaml
image: trivip002/<service-name>:<git-sha>
```

Commit message convention: `chore(<service-name>): bump image to <sha>`

## Architecture

All services live under the `www.devapihub.com` host and are routed via an nginx ingress with `ingressClassName: nginx-application`.

| App directory | Namespace | Port | Ingress path | Notes |
|---|---|---|---|---|
| `ecomerce-shop-dev` | `ecomerce-shop-dev` | 3000 | `/v1/api` | Node.js, 2 replicas, DB secrets via `ecomerce-shop-dev-db` Secret |
| `ecommerce-shop-dev-client` | `ecommerce-shop-dev-client` | 5200 | — | Frontend, 1 replica, NodePort 30100 (no ingress) |
| `admin-api` | `admin-api` | 8080 | `/api` | Spring Boot, 2 replicas, health at `/api/actuator/health` |
| `notification` | `notification` | 8081 | `/notification-service` | Spring Boot, 2 replicas, health at `/notification-service/actuator/health` |

Each app directory under `app/` contains a `k8s/` subdirectory with deployment, service, and (where applicable) ingress manifests.

## TLS / SSL

All ingresses for `www.devapihub.com` are configured with Let's Encrypt via cert-manager.

- **cert-manager version:** v1.15.3 (compatible with K8s 1.29)
- **ClusterIssuer:** `letsencrypt-prod` (file: `cluster-issuer.yaml` at repo root)
- **TLS secret name:** `www-devapihub-com-tls` (shared across namespaces via ingress annotation)
- **Solver:** HTTP-01 challenge via `ingressClassName: nginx`

Every ingress for `www.devapihub.com` must include:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
    - hosts:
        - www.devapihub.com
      secretName: www-devapihub-com-tls
```

To reinstall cert-manager on a new cluster (K8s 1.29):

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.15.3/cert-manager.yaml
kubectl wait --namespace cert-manager --for=condition=ready pod \
  --selector=app.kubernetes.io/instance=cert-manager --timeout=120s
kubectl apply -f cluster-issuer.yaml
```

### Certificate expiry & renewal

Let's Encrypt certificates expire after **90 days**. cert-manager **auto-renews at 30 days before expiry** — no manual action needed under normal conditions.

Check certificate status and expiry date:

```bash
# List all certificates and their ready status
kubectl get certificate -A

# Check expiry date of a specific cert
kubectl get certificate www-devapihub-com-tls -n ecomerce-shop-dev \
  -o jsonpath='{.status.notAfter}'
```

If a certificate is stuck or not renewing (Status = False):

```bash
# 1. Check what went wrong
kubectl describe certificate www-devapihub-com-tls -n ecomerce-shop-dev
kubectl describe certificaterequest -n ecomerce-shop-dev
kubectl describe challenge -n ecomerce-shop-dev

# 2. Force renew by deleting the TLS secret — cert-manager will recreate it
kubectl delete secret www-devapihub-com-tls -n ecomerce-shop-dev

# 3. Verify ClusterIssuer is still Ready
kubectl describe clusterissuer letsencrypt-prod
```

If ClusterIssuer is not Ready (e.g. after cluster rebuild), reapply it:

```bash
kubectl apply -f cluster-issuer.yaml
```

## Key Conventions

- Each service lives in its own namespace matching the app name.
- Spring Boot services (`admin-api`, `notification`) have `readinessProbe`, `livenessProbe`, and `startupProbe` configured — keep these in sync when adding new services.
- Resource defaults: `requests: {cpu: "200m", memory: "512Mi"}`, `limits: {memory: "1Gi"}` for backend services; frontend uses `requests: {cpu: "100m", memory: "256Mi"}`, `limits: {memory: "512Mi"}`.
- The `ecomerce-shop-dev` deployment has Elastic log annotations (`co.elastic.logs/*`) for JSON log ingestion.
- Sensitive values (DB credentials) are injected via Kubernetes Secrets referenced with `secretKeyRef`, never hardcoded.
