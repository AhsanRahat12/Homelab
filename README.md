# 🏠 Homelab

Self-hosted infrastructure running on a Raspberry Pi 5, managed with Flux for GitOps continuous deployment.

---

## Philosophy

Everything is defined as code, version controlled, and applied declaratively. No manual `kubectl apply`, no SSH-ing into nodes to make changes. The Git repo is the single source of truth for what runs in the cluster — Flux handles the rest.

---

## 🖥️ Hardware

| Node | Hardware | Role |
|---|---|---|
| `pi-zoro` | Raspberry Pi 5 (16GB) | Primary cluster node |
| `pi-luffy` | Raspberry Pi 5 (16GB) | Secondary cluster node *(coming soon)* |
| `nas` | QNAP TS-464 | Network attached storage |

**Network:** Private overlay network via Tailscale. External DNS managed via Cloudflare — private services exposed via Cloudflare Tunnel or DNS-only A records (no proxy).

---

## 🧰 Stack

| Tool | Purpose |
|---|---|
| K3s | Lightweight Kubernetes distribution |
| Flux | GitOps controller — syncs cluster state with this repo |
| Kustomize | Environment-specific configuration and overlays |
| Helm | Package manager for Kubernetes applications |
| Traefik | Ingress controller — routes external traffic to services |
| SOPS + AGE | Secrets encryption — safe to commit encrypted secrets to Git |
| Renovate | Automated dependency updates — opens PRs for outdated images and charts |
| Prometheus | Metrics collection and alerting |
| Grafana | Metrics visualisation and dashboards |
| Cloudflare Tunnel | Secure public exposure of services without opening firewall ports |

---

## 📁 Repository Structure

```
Homelab/
├── renovate.json
├── README.md
└── pi-zoro/
    ├── apps/
    ├── clusters/
    ├── docs/
    ├── infrastructure/
    └── monitoring/
```

Flux watches the `clusters/staging/` directory and follows the chain of Kustomization files down to the actual resources. All secrets are encrypted with SOPS before being committed. Each app has its own README with a detailed breakdown of its structure and configuration.

---

## 🚀 Projects

### 🔖 [Linkding](./pi-zoro/docs/linkding/README.md)
Self-hosted bookmark manager running on `pi-zoro`. Accessible externally via Cloudflare Tunnel at `links.rahatahsan.com`. Covers persistent storage, non-root security context, SOPS-encrypted secrets, and dual access patterns via Cloudflare Tunnel and Traefik Ingress.

### 🎧 Audiobookshelf
Self-hosted audiobook server *(coming soon)*

---

## 📊 Infrastructure & Monitoring

**Monitoring:** Full observability stack via kube-prometheus-stack — Prometheus for metrics collection and Grafana for visualisation, accessible privately at `grs.rahatahsan.com`.

**Automated Updates:** Renovate runs as a Kubernetes CronJob, scanning the repo every hour and opening Pull Requests for any outdated images or Helm chart versions.

---

*Part of my broader DevOps learning journey — [main profile](https://github.com/AhsanRahat12)*
