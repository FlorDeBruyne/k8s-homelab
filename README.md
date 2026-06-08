# k8s-homelab

GitOps-managed Kubernetes homelab cluster running on bare-metal HP ProDesk machines.

## Cluster

| Hostname     | IP               | Role          |
|--------------|------------------|---------------|
| hpprodesk01  | 192.168.100.128  | control-plane |
| hpprodesk02  | 192.168.100.129  | worker        |
| hpprodesk03  | 192.168.100.130  | worker        |
| hpprodesk04  | 192.168.100.131  | worker        |

## Stack

| Component             | Purpose                        |
|-----------------------|-------------------------------|
| Kubernetes 1.32       | Container orchestration        |
| Cilium                | CNI (VXLAN mode)               |
| Longhorn              | Distributed block storage      |
| ingress-nginx         | Ingress controller             |
| FluxCD v2             | GitOps continuous delivery     |
| CloudNativePG         | PostgreSQL operator            |
| kube-prometheus-stack | Metrics, alerting, dashboards  |
| metrics-server        | Resource metrics (CPU/MEM)     |
| SOPS + Age            | Secrets encryption             |

## Repository Structure

```
k8s-homelab/
├── .sops.yaml                  # SOPS encryption config
├── cluster/homelab/            # FluxCD entry point
│   └── flux-system/            # FluxCD self-management + SOPS patch
├── infrastructure/             # Core cluster dependencies
│   ├── cloudnative-pg/         # PostgreSQL operator (cluster-wide)
│   ├── longhorn/               # Distributed block storage (~1TB per node)
│   ├── ingress-nginx/          # Ingress controller (NodePort 30080/30443)
│   └── metrics-server/         # Resource metrics
├── monitoring/                 # Observability
│   └── kube-prometheus-stack/  # Prometheus + Grafana
└── apps/                       # Workloads
    └── fitness-coach/          # AI fitness coaching platform
        ├── open-wearables/     # Wearable data platform (namespace: open-wearables)
        │   ├── postgres/       # PostgreSQL cluster via CloudNativePG
        │   ├── redis/          # Redis message broker
        │   ├── svix/           # Webhook delivery service
        │   ├── backend/        # Open Wearables FastAPI backend
        │   ├── celery/         # Celery worker + beat scheduler
        │   ├── flower/         # Celery monitoring dashboard
        │   └── frontend/       # Open Wearables React dashboard
        └── ai-agent/           # AI coaching agent (namespace: fitness-coach) [WIP]
```

## Prerequisites

- FluxCD CLI installed
- `kubectl` configured with cluster access
- `age` and `sops` installed
- GitHub personal access token with `repo` scope

## Bootstrap

```bash
export GITHUB_TOKEN=<your-token>

flux bootstrap github \
  --owner=FlorDeBruyne \
  --repository=k8s-homelab \
  --branch=main \
  --path=cluster/homelab \
  --personal
```

After bootstrapping, create the SOPS Age secret in the cluster:

```bash
age-keygen -o age.agekey
cat age.agekey | kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=/dev/stdin
rm age.agekey
```

> Store the Age private key somewhere safe (e.g. a password manager). It cannot be recovered if lost.

## Secrets

Secrets are encrypted with SOPS + Age before committing. The `apps-sync.yaml` and `infrastructure-sync.yaml` both have SOPS decryption enabled via the `sops-age` secret in `flux-system`.

To edit an encrypted secret:

```bash
sops path/to/secret.sops.yaml
```

SOPS will decrypt, open your `$EDITOR`, and re-encrypt on save. Never commit unencrypted secret files.

## Accessing Services

| Service            | URL                                          |
|--------------------|----------------------------------------------|
| Longhorn UI        | http://longhorn.homelab.local:30080          |
| Grafana            | http://grafana.homelab.local:30080           |
| Open Wearables     | http://open-wearables.homelab.local:30080    |
| Open Wearables API | http://api.open-wearables.homelab.local:30080|
| Flower (Celery)    | http://flower.open-wearables.homelab.local:30080 |

> DNS records are managed via Pi-hole local DNS. Ingress runs on NodePort 30080.

## Apps

### Fitness Coach

An AI-powered fitness coaching platform built on top of [Open Wearables](https://github.com/the-momentum/open-wearables). It collects health data from wearables (Apple Watch via iOS SDK) and uses an AI agent to generate personalized training plans and daily coaching via Telegram.

**Open Wearables** is a self-hosted platform that unifies wearable health data through a single API. It handles Apple Health sync, data normalization, and webhook delivery.

The custom Docker images for Open Wearables are built and pushed to `ghcr.io/flordebruyne` via GitHub Actions on every push to the [fork](https://github.com/FlorDeBruyne/open-wearables).

**AI Agent** (WIP) — A Python-based agentic RAG system that:
- Polls the Open Wearables API daily for sleep, HRV, and workout data
- Uses Claude API to analyze data and generate personalized coaching plans
- Maintains long-term memory via pgvector embeddings
- Delivers coaching messages and plan updates via Telegram Bot

## Infrastructure

The cluster nodes are provisioned using a separate [Ansible repository](https://github.com/FlorDeBruyne/ansible-homelab), which handles OS-level configuration, Kubernetes installation, and firewall rules.