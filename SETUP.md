# DevSecOps CI/CD Lab — Setup Instructions

This repository contains the complete setup for the DevSecOps CI/CD lab with observability stack and Google Online Boutique microservices on k3s.

## Quick Start

### 1. GitHub Actions Secrets

Add these secrets in **Settings → Secrets → Actions**:

| Secret | Value |
|---|---|
| `OBS_SSH_HOST` | Server A IP |
| `OBS_SSH_USER` | e.g. `ubuntu` |
| `OBS_SSH_KEY` | Private key (Server A) |
| `APP_SSH_HOST` | Server B IP |
| `APP_SSH_USER` | e.g. `ubuntu` |
| `APP_SSH_KEY` | Private key (Server B) |
| `SLACK_WEBHOOK` | Slack incoming webhook URL (optional) |

### 2. Server A (Observability Stack)

**Prerequisites:**
- Docker & Docker Compose installed
- TLS certificates at `/opt/observability/certs/{cert.pem,key.pem}`

**Manual Deployment (for testing):**

```bash
cd observability-stack
docker compose up -d
docker compose ps  # verify all services healthy
```

**Access URLs:**
- Grafana: `https://grafana.obs.local` (admin / changeme)
- Prometheus: `https://prometheus.obs.local`
- Loki: `https://loki.obs.local`
- Jaeger: `https://jaeger.obs.local`

**Local DNS Setup:** Add to `/etc/hosts`:
```
SERVER_A_IP   prometheus.obs.local grafana.obs.local loki.obs.local jaeger.obs.local boutique.obs.local
```

### 3. Server B (k3s + ArgoCD)

**Prerequisites:**
- Linux (Ubuntu 20.04+)
- SSH access from GitHub Actions runner

**Bootstrap (via pipeline or manually):**

```bash
# Install k3s
curl -sfL https://get.k3s.io | sh -

# Set KUBECONFIG
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml

# Install ArgoCD
bash ./install-argocd.sh

# Apply boutique manifests
kubectl apply -f boutique-namespace.yml
kubectl apply -f frontend-nodeport.yml
```

**Verify:**
```bash
kubectl get pods -n boutique
kubectl get svc -n boutique frontend-external
```

### 4. Traefik Configuration

Update `observability-stack/traefik/dynamic/routes.yml` with Server B's IP:

```yaml
services:
  boutique-backend:
    loadBalancer:
      servers:
        - url: "http://YOUR_SERVER_B_IP:30080"
```

Then restart Traefik:
```bash
cd observability-stack
docker compose restart traefik
```

## Pipeline Workflows

### Deploy Observability Stack
- **Trigger:** Push to `observability/` or `.github/workflows/deploy-observability.yml`
- **Actions:**
  1. Run Trivy scan (IaC + secrets)
  2. Upload SARIF to GitHub Security tab
  3. Copy files to Server A via SCP
  4. Deploy via SSH (docker compose up)
  5. Notify Slack

### Deploy Online Boutique
- **Trigger:** Push to `boutique/` or `.github/workflows/deploy-boutique.yml`
- **Actions:**
  1. Run Trivy scan (Kubernetes manifests)
  2. Upload SARIF to GitHub Security tab
  3. Bootstrap k3s + ArgoCD on Server B
  4. Apply ArgoCD Application manifest
  5. Wait for sync completion
  6. Notify Slack

## Observability Architecture

```
┌────────────────────────────────────────────────┐
│ Server A (Observability)                       │
│ ┌──────────────────────────────────────────┐  │
│ │ Traefik (reverse proxy)                  │  │
│ │ ├─ prometheus.obs.local → Prometheus    │  │
│ │ ├─ grafana.obs.local → Grafana          │  │
│ │ ├─ loki.obs.local → Loki                │  │
│ │ ├─ jaeger.obs.local → Jaeger            │  │
│ │ └─ boutique.obs.local → Server B:30080  │  │
│ └──────────────────────────────────────────┘  │
│ ┌──────────────────────────────────────────┐  │
│ │ Backend Services                         │  │
│ │ ├─ Prometheus (metrics storage)          │  │
│ │ ├─ Grafana (visualization)               │  │
│ │ ├─ Loki (log aggregation)                │  │
│ │ ├─ Jaeger (distributed tracing)          │  │
│ │ └─ OTel Collector (telemetry gateway)    │  │
│ └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
           ↕ (SSH + reverse proxy)
┌────────────────────────────────────────────────┐
│ Server B (Application)                         │
│ ┌──────────────────────────────────────────┐  │
│ │ k3s Cluster                              │  │
│ │ ├─ ArgoCD (GitOps controller)            │  │
│ │ └─ boutique namespace                    │  │
│ │    ├─ frontend (NodePort 30080)          │  │
│ │    ├─ backend services                   │  │
│ │    └─ OTel instrumentation               │  │
│ └──────────────────────────────────────────┘  │
└────────────────────────────────────────────────┘
```

## Troubleshooting

### Traefik not routing to Boutique
- Verify `routes.yml` has correct `SERVER_B_IP`
- Check: `docker compose logs traefik`
- Restart: `docker compose restart traefik`

### ArgoCD not syncing
- Check ArgoCD logs: `kubectl -n argocd logs -f deploy/argocd-application-controller`
- Verify `boutique-app.yml` manifest applied: `kubectl get applications -n argocd`

### OTel data not reaching Jaeger
- Verify OTel ports exposed: `docker compose ps otel-collector`
- Check OTel config: `docker compose logs otel-collector`

### Certificate issues
- Generate self-signed certs (dev only):
  ```bash
  mkdir -p /opt/observability/certs
  openssl req -x509 -newkey rsa:4096 -keyout /opt/observability/certs/key.pem -out /opt/observability/certs/cert.pem -days 365 -nodes
  ```

## Next Steps (Lab Extensions)

- **Falco** — runtime threat detection on Server B (k3s DaemonSet)
- **Kyverno** — policy engine enforcing no `latest` tags, mandatory resource limits
- **Vault** — inject secrets into pods via the Vault Agent sidecar or External Secrets Operator
- **Alertmanager** — wire Prometheus alerts → Slack/PagerDuty
- **Promtail / Alloy** — deploy on Server B to ship k3s pod logs to Loki on Server A
- **Grafana Tempo** — drop-in Jaeger replacement with better scalability
- **mTLS via Linkerd** — service mesh for zero-trust inside the boutique namespace
