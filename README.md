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
| FluxCD                | GitOps continuous delivery     |
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
│   ├── longhorn/               # Distributed block storage (~1TB per node)
│   ├── ingress-nginx/          # Ingress controller (NodePort 30080/30443)
│   └── metrics-server/         # Resource metrics
├── monitoring/                 # Observability
│   └── kube-prometheus-stack/  # Prometheus + Grafana
└── apps/                       # Workloads
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

Secrets are encrypted with SOPS + Age before committing. To encrypt a secret:

```bash
sops --encrypt secret.yaml > secret.enc.yaml
```

Never commit unencrypted secret files. Flux automatically decrypts them during reconciliation.

## Accessing Services

| Service      | URL                                  |
|--------------|--------------------------------------|
| Longhorn UI  | http://longhorn.homelab.local:30080  |
| Grafana      | http://grafana.homelab.local:30080   |

> DNS records are managed via Pi-hole local DNS. Ingress runs on NodePort 30080.

## Infrastructure

The cluster nodes are provisioned using a separate [Ansible repository](https://github.com/FlorDeBruyne/ansible-homelab), which handles OS-level configuration, Kubernetes installation, and firewall rules.