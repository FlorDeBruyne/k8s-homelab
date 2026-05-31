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

## Repository Structure

```
k8s-homelab/
├── cluster/homelab/        # FluxCD entry point
│   └── flux-system/        # FluxCD self-management
├── infrastructure/         # Core cluster dependencies
│   ├── longhorn/           # Distributed block storage
│   ├── ingress-nginx/      # Ingress controller
│   └── metrics-server/     # Resource metrics
├── monitoring/             # Prometheus, Grafana
│   └── kube-prometheus-stack/
└── apps/                   # Workloads
```

## Prerequisites

- FluxCD CLI installed
- `kubectl` configured with cluster access
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

After bootstrapping, Flux automatically reconciles all resources defined in this repository.

## Accessing Services

| Service      | URL                                  |
|--------------|--------------------------------------|
| Longhorn UI  | http://longhorn.homelab.local:30080  |
| Grafana      | http://grafana.homelab.local:30080   |

> DNS records are managed via Pi-hole local DNS. Ingress runs on NodePort 30080.

## Infrastructure

The cluster nodes are provisioned using a separate [Ansible repository](https://github.com/FlorDeBruyne/ansible-homelab), which handles OS-level configuration, Kubernetes installation, and firewall rules.