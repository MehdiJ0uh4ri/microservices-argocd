# Observability Stack — Server A Deployment

> Complete observability solution with **Traefik**, **Prometheus**, **Grafana**, **Loki**, **Jaeger**, and **OpenTelemetry Collector**.

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────┐
│ Traefik (Reverse Proxy & Load Balancer)     │
│ Ports: 80/443                               │
│ Routes:                                     │
│  - prometheus.obs.local → Prometheus:9090   │
│  - grafana.obs.local → Grafana:3000         │
│  - loki.obs.local → Loki:3100               │
│  - jaeger.obs.local → Jaeger:16686          │
│  - boutique.obs.local → Server B:30080      │
└─────────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────────────────────┐
│ Observability Services (Docker Network: obs)             │
├──────────────────────────────────────────────────────────┤
│ ✓ Prometheus (v2.54.1)         — Metrics storage (15d)   │
│ ✓ Grafana (v11.2.1)            — Visualization           │
│ ✓ Loki (v3.2.0)                — Log aggregation         │
│ ✓ Jaeger (v1.61.0)             — Distributed tracing     │
│ ✓ OTel Collector (v0.110.0)    — Telemetry gateway       │
│                                                           │
│ Ports Exposed:                                            │
│  - 80/443   (Traefik)                                    │
│  - 4317    (OTel gRPC)                                   │
│  - 4318    (OTel HTTP)                                   │
└──────────────────────────────────────────────────────────┘
```

---

## 🚀 Quick Start

### Prerequisites

```bash
# Install Docker & Docker Compose
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Verify installation
docker --version
docker compose --version
```

### 1. Prepare TLS Certificates

```bash
# Create certificate directory
mkdir -p /opt/observability/certs

# Generate self-signed certificates (dev only)
openssl req -x509 -newkey rsa:4096 \
  -keyout /opt/observability/certs/key.pem \
  -out /opt/observability/certs/cert.pem \
  -days 365 -nodes \
  -subj "/CN=obs.local"

# Verify
ls -la /opt/observability/certs/
```

### 2. Deploy Stack

```bash
cd /opt/observability

# Copy docker-compose.yml and all config directories here
# (structure: docker-compose.yml, traefik/, prometheus/, loki/, otel/, grafana/)

# Create environment file (optional)
cp .env.example .env
# Edit .env if needed: vim .env

# Start services
docker compose up -d

# Verify all services are healthy
docker compose ps
docker compose logs
```

### 3. Verify Services

```bash
# Check individual services
docker compose ps
docker compose logs prometheus | tail -20
docker compose logs grafana | tail -20
docker compose logs jaeger | tail -20
docker compose logs otel-collector | tail -20
docker compose logs loki | tail -20
docker compose logs traefik | tail -20
```

### 4. Configure Local DNS

On your client machine, add to `/etc/hosts`:

```
<SERVER_A_IP>   prometheus.obs.local grafana.obs.local loki.obs.local jaeger.obs.local boutique.obs.local
```

### 5. Access Services

| Service | URL | Notes |
|---------|-----|-------|
| **Grafana** | `https://grafana.obs.local` | Default: `admin` / `changeme` |
| **Prometheus** | `https://prometheus.obs.local` | Query metrics |
| **Loki** | `https://loki.obs.local` | Query logs |
| **Jaeger** | `https://jaeger.obs.local` | View traces |
| **Traefik Dashboard** | `https://traefik.obs.local/dashboard/` | (disabled by default) |

---

## 📋 Configuration Files

### `docker-compose.yml`
- Defines all services, networks, volumes
- Service dependencies
- Traefik labels for routing
- Resource limits

### `traefik/traefik.yml`
- Global settings (log level, API, TLS)
- Entry points (HTTP → HTTPS redirect, HTTPS)
- Providers (Docker, File)
- Certificate paths

### `traefik/dynamic/routes.yml`
- HTTP routers (rules, entry points, TLS)
- Service definitions (load balancer backends)
- Middleware (security headers)
- **Important:** Update `SERVER_B_IP` to Server B's actual IP

### `prometheus/prometheus.yml`
- Global scrape interval (15s)
- Scrape configs for:
  - Prometheus itself
  - OTel Collector metrics
  - k3s node metrics (Server B)
  - kube-state-metrics
  - Boutique services

### `loki/loki-config.yml`
- Authentication disabled (for lab)
- TSDB backend with filesystem storage
- 7-day retention (configurable)
- Query caching

### `otel/otel-collector-config.yml`
- Receivers: OTLP (gRPC, HTTP)
- Processors: batch, memory limiter, resource detection
- Exporters: Jaeger (traces), Prometheus (metrics), Loki (logs)
- Pipelines: traces → Jaeger, metrics → Prometheus, logs → Loki

### `grafana/provisioning/datasources/datasources.yml`
- Auto-provisions Prometheus, Loki, Jaeger
- All datasources non-editable

### `grafana/provisioning/dashboards/dashboards.yml`
- Auto-loads dashboards from `/dashboards` directory

---

## 🔄 Common Operations

### Check Logs

```bash
# All services
docker compose logs -f

# Specific service
docker compose logs -f traefik
docker compose logs -f otel-collector

# Last 50 lines
docker compose logs -n 50 prometheus
```

### Stop & Start

```bash
# Stop all services
docker compose down

# Start all services
docker compose up -d

# Restart specific service
docker compose restart prometheus
docker compose restart traefik
```

### Update Configuration

```bash
# Edit configuration (e.g., prometheus.yml)
vim prometheus/prometheus.yml

# Reload Prometheus (without restart)
docker compose exec prometheus curl -X POST http://localhost:9090/-/reload

# Or restart the service
docker compose restart prometheus
```

### Check Service Health

```bash
# Prometheus health
curl -k https://prometheus.obs.local/-/healthy

# Grafana health
curl -k https://grafana.obs.local/api/health

# Loki health
curl -k https://loki.obs.local/loki/api/v1/status/buildinfo

# Jaeger health
curl -k https://jaeger.obs.local/
```

### View Metrics

```bash
# Prometheus API
curl -k https://prometheus.obs.local/api/v1/query?query=up

# OTel Collector metrics
curl http://otel-collector:8888/metrics
```

---

## 🔐 Security Considerations

### For Production

1. **TLS Certificates:** Use valid certificates (Let's Encrypt, CA-signed)
2. **Authentication:** Enable Grafana authentication, consider Traefik basic auth
3. **Network:** Restrict Prometheus/Loki to private network (no direct exposure)
4. **Secrets:** Use Docker secrets or .env files (never commit credentials)
5. **Updates:** Pin image versions, use digest-based pins (`image@sha256:...`)
6. **Backups:** Regular backups of Grafana dashboards, Prometheus data

### Current Lab Setup

- ⚠️ Self-signed certificates (dev only)
- ⚠️ Grafana default password (change via .env)
- ⚠️ Loki auth disabled (use network isolation)
- ⚠️ Prometheus publicly exposed (use firewall rules)

---

## 🛠️ Troubleshooting

### Service Not Starting

```bash
# Check logs
docker compose logs <service_name>

# Check port conflicts
docker ps | grep -E "80|443|9090|3000"
lsof -i :80  # if available

# Verify files exist
ls -la traefik/traefik.yml
ls -la prometheus/prometheus.yml
```

### Certificate Errors (HTTPS)

```bash
# Verify certificate exists
ls -la /opt/observability/certs/

# Check certificate validity
openssl x509 -in /opt/observability/certs/cert.pem -text -noout

# Regenerate if needed
rm /opt/observability/certs/*
openssl req -x509 -newkey rsa:4096 \
  -keyout /opt/observability/certs/key.pem \
  -out /opt/observability/certs/cert.pem \
  -days 365 -nodes -subj "/CN=*.obs.local"

# Restart Traefik
docker compose restart traefik
```

### Traefik Not Routing

```bash
# Check Traefik logs
docker compose logs traefik | tail -50

# Test DNS resolution
nslookup grafana.obs.local <SERVER_A_IP>

# Verify routes.yml syntax
cat traefik/dynamic/routes.yml

# Check Traefik dashboard (if enabled)
# Uncomment in traefik.yml: api.dashboard: true
```

### OTel Not Receiving Data

```bash
# Check OTel logs
docker compose logs otel-collector

# Verify ports are listening
docker compose port otel-collector 4317
docker compose port otel-collector 4318

# Test OTel health
curl http://localhost:13133/  # OTel health check port

# Check gRPC connectivity
grpcurl -plaintext localhost:4317 list  # if grpcurl available
```

### Prometheus Not Scraping

```bash
# Check Prometheus logs
docker compose logs prometheus

# Check targets in Prometheus UI
# https://prometheus.obs.local/targets

# Test connectivity to targets
docker compose exec prometheus curl -v http://otel-collector:8888/metrics
```

---

## 📊 Data Retention

| Component | Default | Config Key | Notes |
|-----------|---------|-----------|-------|
| Prometheus | 15 days | `--storage.tsdb.retention.time=15d` | Edit docker-compose.yml |
| Loki | 7 days | `retention_days: 7` | Edit loki-config.yml |
| Jaeger | In-memory (ephemeral) | — | Use external backend for persistence |
| Grafana | Per datasource | — | Data stored in SQLite (grafana_data volume) |

---

## 🔗 Integration with Server B (k3s)

Update `traefik/dynamic/routes.yml` to route boutique traffic:

```yaml
services:
  boutique-backend:
    loadBalancer:
      servers:
        - url: "http://SERVER_B_IP:30080"  # NodePort 30080 from Server B
```

Restart Traefik:

```bash
docker compose restart traefik
```

---

## 📚 References

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Loki Documentation](https://grafana.com/docs/loki/)
- [Jaeger Documentation](https://www.jaegertracing.io/docs/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)

---

**Happy observing! 📊**
