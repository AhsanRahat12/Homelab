# 🏠 Homelab

Self-hosted infrastructure running on a 2-node Raspberry Pi k3s cluster, managed with Flux for GitOps continuous deployment.

---

## Philosophy

Everything is defined as code, version controlled, and applied declaratively. No manual `kubectl apply`, no SSH-ing into nodes to make changes. The Git repo is the single source of truth for what runs in the cluster — Flux handles the rest.

Storage is deliberately separated from compute. Persistent data lives on a QNAP NAS, not on the nodes. A CSI driver manages the attach/detach lifecycle automatically — if a node fails, the pod reschedules to the other node and storage follows. Single node failure is survivable.

---

## 🖥️ Hardware

| Node | Hardware | Role |
|------|----------|------|
| `zoro` | Raspberry Pi 5 (16GB) | Control plane |
| `luffy` | Raspberry Pi 5 (16GB) | Worker node |
| `nas` | QNAP TS-464 | Network attached storage — iSCSI LUNs and NFS shares |

**Network:** Private overlay network via Tailscale — secure remote access without VPN configuration. External DNS managed via Cloudflare — services exposed via Cloudflare Tunnel (no open ports, no port forwarding).

---

## 🧰 Stack

| Tool | Purpose |
|------|---------|
| K3s | Lightweight Kubernetes distribution |
| Flux | GitOps controller — syncs cluster state with this repo |
| Kustomize | Environment-specific configuration and overlays |
| Helm | Package manager for Kubernetes applications |
| Traefik | Ingress controller |
| SOPS + Age | Secrets encryption — safe to commit encrypted secrets to Git |
| Renovate | Automated dependency updates — opens PRs for outdated images and charts |
| democratic-csi | CSI driver — manages iSCSI attach/detach between nodes automatically |
| Prometheus | Metrics collection and alerting |
| Grafana | Metrics visualisation and dashboards |
| Cloudflare Tunnel | Secure public exposure of services without opening firewall ports |
| Tailscale | Private overlay network — secure remote access to the cluster |

---

## 📁 Repository Structure

```
Homelab/
├── renovate.json
├── README.md
└── pi-zoro/
    ├── apps/
    │   ├── base/          ← environment-agnostic manifests
    │   ├── staging/       ← staging overlays — local-path storage, internal only
    │   └── production/    ← production overlays — NAS storage, Cloudflare Tunnel
    ├── clusters/
    ├── docs/
    ├── infrastructure/    ← democratic-csi, Renovate
    └── monitoring/        ← kube-prometheus-stack
```

Flux watches `clusters/staging/` and follows the chain of Kustomization files down to the actual resources. All secrets are encrypted with SOPS before being committed. Each app has its own README with a detailed breakdown of decisions, problems solved, and architecture.

Staging and production are separate overlays pointing at the same base. Staging uses local-path storage on the SD card — intentionally kept simple for contrast. Production uses NAS-backed storage with no nodeSelector — pods run on either node and storage follows automatically.

---

## 🚀 Projects

### 🔖 [Linkding](./pi-zoro/docs/linkding/README.md)
Self-hosted bookmark manager. Accessible at `links.rahatahsan.com` via Cloudflare Tunnel.

Storage went through three stages — SD card → static iSCSI PV pinned to Zoro → democratic-csi managed iSCSI with no nodeSelector. Production data lives on a QNAP iSCSI LUN. Pod runs on either node, failover tested and proven.

### 📚 [Audiobookshelf](./pi-zoro/docs/audiobookshelf/README.md)
Self-hosted audiobook and podcast server. Accessible at `audiobooks.rahatahsan.com` via Cloudflare Tunnel.

Four volumes with a deliberate storage split — config and metadata on iSCSI LUNs (democratic-csi, block storage, RWO), audiobooks and podcasts on NFS shares (RWX, mounts on any node with no detach needed). Zero volumes on SD card in production. Full failover tested and proven.

---

## 📊 Infrastructure & Monitoring

**Storage:** QNAP TS-464 serves both iSCSI LUNs (via democratic-csi) and NFS shares. iSCSI is used for database and structured data volumes. NFS is used for media. Storage is completely off the cluster nodes.

**Monitoring:** Full observability stack via kube-prometheus-stack — Prometheus for metrics collection, Grafana for visualisation, accessible at `grs.rahatahsan.com`.

**Automated Updates:** Renovate runs as a Kubernetes CronJob, scanning the repo every hour and opening Pull Requests for outdated images or Helm chart versions.

---

*Part of my broader DevOps learning journey — [main profile](https://github.com/AhsanRahat12)*
