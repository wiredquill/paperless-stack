# Paperless Stack

Document management system with AI-powered auto-tagging, OCR, and local LLM inference.

## Project Goals

- **Centralized document management**: Paperless-ngx as the core document repository
- **AI-powered auto-tagging**: Automatic metadata extraction using local LLMs (Ollama)
- **Vision OCR**: Intelligent document scanning via Paperless-GPT (minicpm-v model)
- **Unified access**: Single nginx reverse proxy with per-service subdomains
- **Zero external AI dependencies**: All ML runs locally via Ollama (GPU optional)
- **Fleet-managed**: GitOps deployment via Rancher Fleet with automatic reconciliation

## Architecture

```
                        ┌─────────────────────────────────────────┐
                        │         nginx LoadBalancer               │
                        │         (single IP / subdomains)         │
                        └────────┬──────┬──────┬──────┬───────────┘
                                 │      │      │      │
                          ┌──────┴┐ ┌──┴──┐ ┌─┴───┐ ┌┴────┐
                          │Paperless│ │Open │ │AI-  │ │GPT  │
                          │-ngx:8000│ │WebUI│ │AI:3000│ │:8080│
                          └───┬────┘ └──┬──┘ └──┬──┘ └──┬──┘
                              │         │       │       │
                        ┌─────┴──────┐   │       │       │
                        │   Postgres  │   │       │       │
                        │   :5432     │   │       │       │
                        └─────┬──────┘   │       │       │
                              │          │       │       │
                        ┌─────┴──────┐ ┌─┴─┐ ┌──┴──┐ ┌┴────┐
                        │   Redis     │ │Ti │ │Got │ │Ollama│
                        │   :6379     │ │ka │ │ben │ │:11434│
                        └─────────────┘ │:99│ │:300│ │      │
                                        └─┬99│ └──┬─┘ └──┬──┘
                                          └──┘    └────┘    └─────┘
```

## Services

| Service | Port | Purpose |
|---------|------|---------|
| paperless | 8000 | Document management UI/API |
| open-webui | 8080 | Web interface for Ollama |
| paperless-ai | 3000 | Metadata suggestions & classification |
| paperless-gpt | 8080 | Vision OCR & auto-tagging |
| tika | 9998 | Document text extraction |
| gotenberg | 3000 | PDF/document conversion |
| ollama | 11434 | Local LLM inference (GPU optional) |

## Prerequisites

- **Kubernetes cluster**: v1.28+ (tested on RKE2 v1.35.6)
- **Helm**: v3.12+
- **Harvester storage class**: `harvester` (default)
- **GPU node (optional)**: For Ollama with GPU acceleration
  - NVIDIA drivers installed
  - `nvidia.com/gpu` label on node
  - GPU tolerations applied

## Quick Start

### 1. Clone the repository

```bash
git clone https://github.com/wiredquill/paperless-stack.git
cd paperless-stack
```

### 2. Configure values

Edit `values.yaml` with your settings:

```yaml
# Required: Set passwords
paperless:
  env:
    PAPERLESS_ADMIN_PASSWORD: "your-secure-password"
    PAPERLESS_DBPASS: "your-db-password"
postgres:
  env:
    POSTGRES_PASSWORD: "your-db-password"

# Optional: Configure domains
global:
  domain: "paperless.wiredquill.com"
```

Or use the `questions.yaml` file as a checklist.

### 3. Deploy via Helm

```bash
# Create namespace
kubectl create namespace paperless

# Install the chart
helm install paperless-stack . \
  --namespace paperless \
  --create-namespace \
  --wait \
  --timeout 10m
```

### 4. Get the LoadBalancer IP

```bash
kubectl get svc -n paperless nginx
# Get the EXTERNAL-IP and configure DNS
```

### 5. Access the services

```
http://<EXTERNAL-IP>           -> Paperless-ngx
http://<EXTERNAL-IP>           -> Open WebUI
http://<EXTERNAL-IP>           -> Paperless-AI
http://<EXTERNAL-IP>           -> Paperless-GPT
```

Or configure DNS records for subdomains:
- `paperless.wiredquill.com`
- `open-webui.wiredquill.com`
- `paperless-ai.wiredquill.com`
- `paperless-gpt.wiredquill.com`

### 6. Initial setup

1. Log into Paperless-ngx with admin credentials
2. Generate an API token: Settings → API Tokens → Add Token
3. Set the token in `values.yaml`:
   ```yaml
   paperlessAi:
     env:
       PAPERLESS_API_TOKEN: "your-token"
   paperlessGpt:
     env:
       PAPERLESS_API_TOKEN: "your-token"
   ```
4. Deploy Ollama and pull models:
   ```bash
   kubectl exec -it ollama-0 -n paperless -- ollama pull llama3.2:3b
   kubectl exec -it ollama-0 -n paperless -- ollama pull minicpm-v:8b
   ```

## Fleet/GitOps Deployment

This chart is designed for Rancher Fleet git-sync. Changes to `values.yaml` are automatically reconciled.

```yaml
# Fleet.yaml (deployed alongside this chart)
apiVersion: fleet.cattle.io/v1alpha1
kind: ClusterGroup
metadata:
  name: paperless-clusters
  namespace: fleet-local
spec:
  selector:
    matchLabels:
      fleet.cattle.io/cluster-join-secret: true
---
apiVersion: fleet.cattle.io/v1alpha1
kind: GitRepo
metadata:
  name: paperless-stack
  namespace: fleet-local
spec:
  repo: https://github.com/wiredquill/paperless-stack.git
  branch: main
  paths:
  - .
  clusterSelector:
    matchLabels:
      cluster-label: paperless-ready
```

## Configuration Reference

### Required Configuration

| Setting | Description | Default |
|---------|-------------|---------|
| `global.domain` | Base domain for the stack | `paperless.wiredquill.com` |
| `paperless.env.PAPERLESS_ADMIN_PASSWORD` | Admin password | `change-me-admin-password` |
| `paperless.env.PAPERLESS_DBPASS` | Database password | `change-me-strong-password` |
| `postgres.env.POSTGRES_PASSWORD` | Database password | `change-me-strong-password` |

### Optional Configuration

| Setting | Description | Default |
|---------|-------------|---------|
| `ollama.enabled` | Enable Ollama (GPU required) | `true` |
| `openWebui.enabled` | Enable Open WebUI | `true` |
| `paperlessAi.enabled` | Enable Paperless-AI | `true` |
| `paperlessGpt.enabled` | Enable Paperless-GPT | `true` |
| `ocr.languages` | OCR languages | `["eng"]` |
| `loadbalancer.enabled` | Enable nginx LoadBalancer | `true` |

## Troubleshooting

### Pods not starting

```bash
# Check pod status
kubectl get pods -n paperless

# Check pod logs
kubectl logs -n paperless <pod-name> --tail=50

# Describe pod for events
kubectl describe pod -n paperless <pod-name>
```

### Ollama not connecting

```bash
# Check Ollama is running
kubectl exec -it ollama-0 -n paperless -- ollama list

# Pull a model
kubectl exec -it ollama-0 -n paperless -- ollama pull llama3.2:3b

# Verify network connectivity
kubectl exec -it paperless-ai-... -n paperless -- curl http://ollama:11434
```

### Redis/PostgreSQL issues

```bash
# Check PostgreSQL connectivity
kubectl exec -it postgres-0 -n paperless -- psql -U paperless -c "SELECT 1;"

# Check Redis connectivity
kubectl exec -it redis-0 -n paperless -- redis-cli PING

# Check Redis persistence status
kubectl exec -it redis-0 -n paperless -- redis-cli INFO persistence
```

### LoadBalancer not getting IP

```bash
# Check if Harvester/Cilium is assigning the IP
kubectl get svc -n paperless nginx -o jsonpath='{.status.loadBalancer}'

# Check Cilium IPAM allocation
cilium ipam status -n paperless

# Wait for IP assignment (can take several minutes)
watch kubectl get svc -n paperless nginx
```

## Backup & Restore

### PVC Snapshots

```bash
# Create Harvester volume snapshot via Rancher UI
# Or use kubectl:
kubectl create snapshot -n paperless <pvc-name>
```

### Paperless Export

```bash
# Generate export from Paperless UI
# Settings → Export → Generate Export
# This creates a .tar.gz importable on fresh install
```

### Database Backup

```bash
# Backup PostgreSQL
kubectl exec -it postgres-0 -n paperless -- pg_dump -U paperless paperless > backup.sql

# Restore from backup
kubectl cp backup.sql paperless-0:/backup.sql -n paperless
kubectl exec -it postgres-0 -n paperless -- psql -U paperless paperless < /backup.sql
```

## Resource Requirements

| Service | CPU Request | Memory Request | Notes |
|---------|-------------|----------------|-------|
| paperless | 50m | 256Mi |  |
| postgres | 50m | 256Mi |  |
| redis | 50m | 256Mi |  |
| gotenberg | 200m | 512Mi |  |
| tika | 200m | 512Mi |  |
| ollama | 2 (CPU) + 1 GPU | 8Gi | GPU optional |
| open-webui | 50m | 256Mi |  |
| paperless-ai | 50m | 256Mi |  |
| paperless-gpt | 50m | 256Mi |  |
| nginx | 50m | 64Mi |  |

**Minimum total**: ~1.5 CPU, ~4.5Gi RAM (without Ollama GPU)
**With Ollama GPU**: ~3.5 CPU, ~12Gi RAM + GPU

## License

MIT
