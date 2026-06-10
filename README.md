# 🏠 Homelab
Self-hosted infrastructure running on a 2-node Raspberry Pi k3s cluster, managed with Flux for GitOps continuous deployment.

---

## Philosophy

Everything is defined as code, version controlled, and applied declaratively. No manual `kubectl apply`, no SSH-ing into nodes to make changes. The Git repo is the single source of truth for what runs in the cluster — Flux handles the rest.

Storage is deliberately separated from compute. Persistent data lives on a QNAP NAS, not on the nodes. A CSI driver manages the attach/detach lifecycle automatically — if a node fails, the pod reschedules to the other node and storage follows. Single node failure is survivable.

Stateful workloads run on a dedicated HA PostgreSQL cluster. Backups are continuous, off-cluster, and independent of Kubernetes — if the cluster dies, the backup target stays alive.

---

## Architecture

![Homelab architecture diagram](HomeLab_Architecture.svg)

---

## 🖥️ Hardware

| Node | Hardware | Role |
|------|----------|------|
| `zoro` | Raspberry Pi 5 (16GB) | Control plane |
| `luffy` | Raspberry Pi 5 (16GB) | Worker node |
| `nas` | QNAP TS-464 | Network attached storage — iSCSI LUNs, NFS shares, Minio S3 |

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
| CloudNative-PG | PostgreSQL operator — HA cluster, streaming replication, automated failover |
| Barman Cloud | PostgreSQL backup tool — WAL archiving and base backups to S3 |
| cert-manager | TLS certificate management for cluster-internal services |
| Minio | S3-compatible object storage on QNAP — backup target for CNPG |
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
    ├── databases/         ← CNPG cluster, scheduled backups, SOPS-encrypted secrets
    ├── clusters/
    ├── docs/
    ├── infrastructure/    ← democratic-csi, cert-manager, CNPG operator, Renovate
    └── monitoring/        ← kube-prometheus-stack
```

Flux watches `clusters/staging/` and follows the chain of Kustomization files down to the actual resources. All secrets are encrypted with SOPS before being committed. Each app has its own README with a detailed breakdown of decisions, problems solved, and architecture.

Staging and production are separate overlays pointing at the same base. Staging uses local-path storage on the SD card — intentionally kept simple for contrast. Production uses NAS-backed storage with no nodeSelector — pods run on either node and storage follows automatically.

---

## 🚀 Projects

### 🔖 [Linkding](./pi-zoro/docs/linkding/README.md)
Self-hosted bookmark manager. Accessible at `links.rahatahsan.com` via Cloudflare Tunnel.

Storage went through three stages — SD card → static iSCSI PV pinned to Zoro → democratic-csi managed iSCSI with no nodeSelector. Production data lives on a QNAP iSCSI LUN. Pod runs on either node, failover tested and proven. Migration to CNPG PostgreSQL backend planned.

### 📚 [Audiobookshelf](./pi-zoro/docs/audiobookshelf/README.md)
Self-hosted audiobook and podcast server. Accessible at `audiobooks.rahatahsan.com` via Cloudflare Tunnel.

Four volumes with a deliberate storage split — config and metadata on iSCSI LUNs (democratic-csi, block storage, RWO), audiobooks and podcasts on NFS shares (RWX, mounts on any node with no detach needed). Zero volumes on SD card in production. Full failover tested and proven.

### 🏠 [Homepage](./pi-zoro/docs/homepage/README.md)
Cluster startpage and app dashboard. Accessible at `homepage.rahatahsan.com` via Cloudflare Tunnel.

Single pod, no persistence — config lives entirely in Git via ConfigMap. Integrates with the Kubernetes API via a dedicated ClusterRole to display live cluster metrics, node CPU and memory, and pod counts. All apps in the cluster are linked from a single dashboard. PSA baseline enforced on the namespace, restricted audited.

---

## 📊 Infrastructure & Data

**Storage:** QNAP TS-464 serves iSCSI LUNs (via democratic-csi), NFS shares, and Minio S3 (via Docker). iSCSI is used for databases and structured data. NFS is used for media. Minio is used as the S3 backup target for PostgreSQL. Storage is completely off the cluster nodes.

### 🐘 [CloudNative-PG](https://github.com/AhsanRahat12/Homelab/tree/main/pi-zoro/docs/cnpg)
Two-instance HA PostgreSQL cluster managed by the CloudNative-PG operator. Primary on zoro, replica on luffy, with streaming replication and automated failover. Built to replace per-app SQLite databases — SQLite on iSCSI is one dropped connection away from corruption.

Continuous WAL archiving to Minio via the Barman Cloud plugin. Daily base backups at 02:00 UTC with 30-day retention, preferring the standby to avoid load on the primary. The backup target lives on the QNAP outside Kubernetes — if the cluster dies, the backups survive.

HA failover was validated under a real iSCSI session drop during initial deployment — the primary lost storage, the replica was promoted within seconds, no data loss.

| Component | Storage | Size |
|-----------|---------|------|
| Primary (zoro) | iSCSI LUN on QNAP | 25Gi |
| Replica (luffy) | iSCSI LUN on QNAP | 25Gi |
| Backup target | Minio bucket on QNAP | unbounded, 30d retention |

### 📈 [kube-prometheus-stack](https://github.com/AhsanRahat12/Homelab/tree/main/pi-zoro/docs/Kube-Prometheus-Stack)
Full observability stack — Prometheus for metrics collection, Grafana for dashboards and visualisation, Alertmanager for alert routing. Accessible at `grs.rahatahsan.com` on the local network.

All three components were originally running on ephemeral emptyDir storage backed by SD cards — Prometheus TSDB is one of the worst workloads for SD card longevity, and every pod restart wiped all history. Migrated all three to dedicated iSCSI LUNs on QNAP via democratic-csi. Metrics and dashboards now survive restarts and redeployments. Write pressure is off the SD cards.

| Component | Storage | Size |
|-----------|---------|------|
| Prometheus | iSCSI LUN on QNAP | 10Gi |
| Grafana | iSCSI LUN on QNAP | 2Gi |
| Alertmanager | iSCSI LUN on QNAP | 1Gi |

**Automated Updates:** Renovate runs as a Kubernetes CronJob, scanning the repo every hour and opening Pull Requests for outdated images or Helm chart versions.


---

## 🌐 Connect

[LinkedIn](https://www.linkedin.com/in/rahatahsan/) &nbsp;•&nbsp; [Twitter/X](https://x.com/RahatAhsan20) &nbsp;•&nbsp; [GitHub (Main Profile)](https://github.com/AhsanRahat12) &nbsp;•&nbsp; [Medium](https://medium.com/@s.rahatahsan)
