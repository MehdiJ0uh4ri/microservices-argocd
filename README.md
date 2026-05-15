# DevSecOps CI/CD Lab — Complete Setup Guide

> **Overview:** A production-ready DevSecOps lab combining an **observability stack** (Prometheus, Grafana, Loki, Jaeger, OTel) on Server A with a **Kubernetes microservices application** (Google Online Boutique) on Server B (k3s), orchestrated via GitHub Actions and ArgoCD.

---

## 📋 Architecture Overview

```
GitHub Repository
       │
       ├─→ push to main
       │
       ├─ Workflow 1: deploy-observability.yml
       │   ├─ Trivy scan (IaC + secrets)
       │   ├─ Security gate (SARIF upload)
       │   └─ SSH deploy to Server A
       │       └─ docker compose up
       │
       └─ Workflow 2: deploy-boutique.yml
           ├─ Trivy scan (Kubernetes config)
           ├─ Security gate (SARIF upload)
           └─ SSH deploy to Server B
               ├─ Install k3s + ArgoCD
               └─ Apply ArgoCD Application
                   └─ Sync Online Boutique from public repo
```

---

## 🗂️ Project Structure

```
├── observability-stack/           (Server A deployment)
│   ├── docker-compose.yml         (Traefik, Prometheus, Grafana, Loki, Jaeger, OTel)
│   ├── .env.example
│   ├── traefik/
│   │   ├── traefik.yml
│   │   └── dynamic/routes.yml     (routes to Server B boutique)
│   ├── prometheus/
│   │   └── prometheus.yml
│   ├── loki/
│   │   └── loki-config.yml
│   ├── otel/
│   │   └── otel-collector-config.yml
│   └── grafana/
│       └── provisioning/
│           ├── datasources/datasources.yml
│           └── dashboards/dashboards.yml
│
├── microservices-argocd/          (Repository root for CI/CD)
│   ├── .github/workflows/
│   │   ├── deploy-observability.yml
│   │   └── deploy-boutique.yml
│   ├── trivy.yaml
│   ├── trivy-secret.yaml
│   ├── SETUP.md
│   ├── boutique/
│   │   ├── argocd/
│   │   │   ├── install-argocd.sh
│   │   │   └── boutique-app.yml    (ArgoCD Application manifest)
│   │   └── manifests/
│   │       ├── boutique-namespace.yml
│   │       └── frontend-nodeport.yml
```

---

## 🚀 Quick Start

### Step 1: Prepare Infrastructure

You need **two servers**:
- **Server A (Observability):** 2+ vCPU, 4GB RAM, Docker + Docker Compose
- **Server B (Application):** 2+ vCPU, 4GB RAM, Linux (Ubuntu 20.04+)

### Step 2: Configure GitHub Actions Secrets

In your repository, go to **Settings → Secrets and variables → Actions**, add:

```plaintext
OBS_SSH_HOST     = <Server A IP>
OBS_SSH_USER     = ubuntu
OBS_SSH_KEY      = <private key for Server A>

APP_SSH_HOST     = <Server B IP>
APP_SSH_USER     = ubuntu
APP_SSH_KEY      = <private key for Server B>

SLACK_WEBHOOK    = https://hooks.slack.com/... (optional, for notifications)
```

### Step 3: Prepare Server A (Observability)

```bash
# SSH into Server A
ssh ubuntu@<SERVER_A_IP>

# Create directories
mkdir -p /opt/observability/certs
cd /opt/observability

# Generate self-signed TLS certificates (dev only)
openssl req -x509 -newkey rsa:4096 \
  -keyout certs/key.pem \
  -out certs/cert.pem \
  -days 365 -nodes \
  -subj "/CN=obs.local"

# Copy your observability-stack files here
# (Pipeline will do this automatically on push)
```

### Step 4: Prepare Server B (Application)

```bash
# SSH into Server B
ssh ubuntu@<SERVER_B_IP>

# Install Docker (optional, for debugging)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# k3s will be installed automatically by the pipeline
# Or manually:
curl -sfL https://get.k3s.io | sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

### Step 5: Configure Local DNS (your machine)

Add to `/etc/hosts` (Linux/Mac) or `C:\Windows\System32\drivers\etc\hosts` (Windows):

```
<SERVER_A_IP>   prometheus.obs.local grafana.obs.local loki.obs.local jaeger.obs.local boutique.obs.local
```

### Step 6: Update Traefik Routes

Edit `observability-stack/traefik/dynamic/routes.yml`:

```yaml
services:
  boutique-backend:
    loadBalancer:
      servers:
        - url: "http://<YOUR_SERVER_B_IP>:30080"
```

### Step 7: Push to Main

```bash
git add .
git commit -m "Initial DevSecOps lab setup"
git push -u origin main
```

**GitHub Actions will automatically:**
1. Scan code with Trivy
2. Deploy observability stack to Server A
3. Bootstrap k3s + ArgoCD on Server B
4. Deploy Google Online Boutique via ArgoCD
5. Send Slack notifications (if configured)

---

## 🔍 Access the Services

### Observability Stack (Server A)

| Service | URL | Default Creds |
|---------|-----|---|
| **Grafana** | `https://grafana.obs.local` | `admin` / `changeme` |
| **Prometheus** | `https://prometheus.obs.local` | (no auth) |
| **Loki** | `https://loki.obs.local` | (no auth) |
| **Jaeger** | `https://jaeger.obs.local` | (no auth) |

### Online Boutique (Server B via Traefik)

| Service | URL |
|---------|-----|
| **Boutique Frontend** | `https://boutique.obs.local` |

---

## 🛠️ Troubleshooting

### ❌ Traefik not routing to Boutique

```bash
# SSH into Server A
docker compose logs traefik | tail -20
docker compose ps traefik

# Verify routes config
cat traefik/dynamic/routes.yml

# Restart Traefik
docker compose restart traefik
```

### ❌ k3s not starting on Server B

```bash
# SSH into Server B
k3s --version
kubectl get nodes
journalctl -u k3s -n 20
```

### ❌ ArgoCD not syncing

```bash
# SSH into Server B
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
kubectl get applications -n argocd
kubectl describe application boutique -n argocd
```

### ❌ OTel not receiving telemetry

```bash
# SSH into Server A
docker compose logs otel-collector | tail -20

# Verify ports
docker compose port otel-collector 4317
docker compose port otel-collector 4318
```

### ❌ Certificate issues (HTTPS errors)

Generate new self-signed certificates:

```bash
# SSH into Server A
rm -rf /opt/observability/certs/*
openssl req -x509 -newkey rsa:4096 \
  -keyout /opt/observability/certs/key.pem \
  -out /opt/observability/certs/cert.pem \
  -days 365 -nodes \
  -subj "/CN=*.obs.local"

# Restart Traefik
docker compose restart traefik
```

---

## 📊 Observability Features

### Metrics (Prometheus)
- Container metrics (docker-compose)
- k3s node metrics (Node Exporter)
- kube-state-metrics (Kubernetes objects)
- Application metrics (OTel instrumentation)

### Logs (Loki)
- Docker container logs
- Pod logs from k3s cluster
- Application logs

### Traces (Jaeger)
- Distributed tracing via OTel
- Google Online Boutique spans
- Service dependencies

### Visualization (Grafana)
- Pre-configured datasources (Prometheus, Loki, Jaeger)
- Dashboard support (add your own in provisioning/dashboards/)

---

## 🔐 Security Features

| Feature | Where |
|---------|-------|
| **Shift-left scanning** | Trivy in GitHub Actions (before deployment) |
| **Secret detection** | Trivy secret rules (trivy-secret.yaml) |
| **IaC scanning** | Trivy on docker-compose, Kubernetes manifests |
| **SARIF reporting** | Auto-uploaded to GitHub Security tab |
| **TLS everywhere** | Traefik terminates HTTPS for all services |
| **GitOps audit trail** | Every deployment tracked in Git history |
| **Immutable infra** | Docker images pinned by version (upgrade: use digest pins) |
| **RBAC (k3s)** | Default ServiceAccounts for ArgoCD namespace |

---

## 📈 Next Steps & Extensions

- **[Falco](https://falco.org/)** — Runtime threat detection on k3s (DaemonSet)
- **[Kyverno](https://kyverno.io/)** — Policy engine (no `latest` tags, resource limits)
- **[HashiCorp Vault](https://www.vaultproject.io/)** — Secret injection via Vault Agent
- **[Alertmanager](https://prometheus.io/docs/alerting/latest/overview/)** — Alerts to Slack/PagerDuty
- **[Promtail](https://grafana.com/docs/loki/latest/clients/promtail/)** — Ship k3s logs to Loki
- **[Grafana Tempo](https://grafana.com/oss/tempo/)** — Drop-in Jaeger replacement
- **[Linkerd](https://linkerd.io/)** — Service mesh for mTLS + observability

---

## 📚 References

- [LAB.md](../LAB.md) — Detailed architecture and configuration
- [SETUP.md](./SETUP.md) — Step-by-step setup instructions
- [observability-stack/.env.example](../observability-stack/.env.example) — Environment variables template
- [Trivy Documentation](https://aquasecurity.github.io/trivy/)
- [k3s Documentation](https://docs.k3s.io/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)

---

## 💡 Key DevSecOps Concepts

1. **Shift-left Security** — Scan code before deployment, not after
2. **Infrastructure as Code** — All configuration in version control
3. **GitOps** — Single source of truth in Git; ArgoCD keeps cluster in sync
4. **Observability** — Logs + Metrics + Traces for full visibility
5. **Immutable Infrastructure** — Containers replaced, never patched
6. **Audit Trail** — Every change tracked; easy rollback via Git
7. **Automation** — CI/CD pipelines reduce manual errors
8. **Defense in Depth** — Multiple security layers (scanning, policies, RBAC, mTLS)

---

**Happy labbing! 🚀**
