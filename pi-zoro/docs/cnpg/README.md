# 🐘 CloudNative-PG
Two-instance HA PostgreSQL cluster deployed on Kubernetes via GitOps. Provides a shared database backend for all homelab applications — backed by durable iSCSI storage on QNAP with automated daily backups to Minio via Barman. [cloudnative-pg/cloudnative-pg](https://github.com/cloudnative-pg/cloudnative-pg)

Built to replace per-app SQLite databases that were one iSCSI hiccup away from corruption.

---

## Architecture

<p align="center">
  <img src="architecture.svg" alt="CloudNative-PG production architecture diagram" width="680"/>
</p>

---

## Stack

| Concern | Solution |
|---------|----------|
| Deployment | Flux GitOps — HelmRelease for operator, raw manifests for Cluster resource |
| Operator | CloudNative-PG v0.23.0 — manages pod lifecycle, failover, and replication |
| Backup plugin | Barman Cloud v0.5.0 — WAL archiving and base backups to S3-compatible storage |
| Cert management | cert-manager v1.17.2 — TLS certificates for cluster-internal replication |
| Data storage | democratic-csi → iSCSI LUNs on QNAP NAS — 25Gi primary, 25Gi replica |
| CSI Driver | democratic-csi node-manual — handles iSCSI attach/detach between nodes |
| Backup target | Minio on QNAP (Docker) — S3 API at 192.168.1.153:9000, bucket `cnpg-backups` |
| Offsite backup | Cloudflare R2 — planned, not yet configured |
| Backup schedule | Daily at 02:00 UTC, prefer-standby, 30-day retention |
| HA topology | topologySpreadConstraints — guarantees one instance per node |
| Secrets | SOPS + age encryption — superuser and Barman S3 credentials |
| Image updates | Renovate CronJob — automated PRs on new releases |

---

## 📁 Repo Structure

```
infrastructure/controllers/
  base/cert-manager/                 ← HelmRelease, HelmRepository
  base/cnpg/                         ← CNPG operator HelmRelease, HelmRepository
  base/cnpg-barman-plugin/           ← Barman plugin HelmRelease, HelmRepository
  staging/cert-manager/              ← namespace, kustomization overlay
  staging/cnpg/                      ← namespace, kustomization overlay
  staging/cnpg-barman-plugin/        ← kustomization overlay

databases/
  base/cnpg-cluster/                 ← Cluster resource, ScheduledBackup
  staging/cnpg-cluster/              ← namespace, PVs, SOPS secrets, kustomization
  production/cnpg-cluster/           ← empty, for future promotion

clusters/staging/
  databases.yaml                     ← Flux Kustomization entrypoint

docs/cnpg/
  README.md                          ← you are here
  architecture.svg                   ← architecture diagram
  cnpg-chown-job.yaml                ← bootstrap: volume ownership fix
  pvc-chown-bootstrap.yaml           ← bootstrap: temporary PVC for chown
  cnpg-test-backup.yaml              ← on-demand backup trigger
```

The CNPG operator is installed at the infrastructure layer via HelmRelease. The Cluster resource and its supporting manifests live in a separate `databases/` layer with an explicit Flux dependency on `infrastructure-controllers`.

---

## 🧠 Problems & Decisions

**Fresh iSCSI LUNs are root-owned — PostgreSQL cannot write to them.**

Identical problem to the monitoring stack, different user. `mkfs.ext4` creates a root-owned filesystem. PostgreSQL runs as UID 26 (not 1000 like Prometheus). The initdb process fails immediately with `Permission denied` when trying to create the pgdata directory.

Unlike kube-prometheus-stack, CNPG does not support `spec.initContainers` in its Cluster CRD — the field is not declared in the schema. Attempting to add one causes Flux to fail with a dry-run validation error. This means the Prometheus approach (initContainer with `chown -R`) cannot be replicated inside the CNPG manifest.

The fix is a one-time bootstrap Job that mounts each PVC as root and runs `chown -R 26:26 /data`. Since the PVs use `Retain` policy, the ownership persists across all restarts and redeployments. The Job files live in `docs/cnpg/` and are applied manually — they are not managed by Flux.

```
Prometheus  → initContainer in HelmRelease values    → self-healing on every deploy
CNPG        → one-time bootstrap Job                 → persists via Retain policy
```

**PVs with Retain policy get stuck in Released state.**

When a PVC is deleted (cluster deletion, nuclear reset), the PV enters `Released` state. Kubernetes refuses to rebind it — even to a new PVC with the exact same name and spec. Without intervention, the new cluster hangs forever waiting for storage.

The fix is `claimRef` in the PV spec. Pre-binding each PV to its expected PVC name forces Kubernetes to accept the rebind automatically:

```yaml
spec:
  claimRef:
    name: cnpg-cluster-1
    namespace: cnpg
```

This was not documented in any CNPG guide. Discovered through repeated cluster deletions during debugging.

**iSCSI sessions drop intermittently — causing I/O errors and failover.**

After the cluster was running stable, PostgreSQL began hitting `Input/output error` on random file operations. The CNPG operator detected the storage failure and initiated failover to the replica. Investigation revealed the iSCSI TCP session was dropping and reconnecting.

Root cause: both Raspberry Pi nodes had Linux NIC power management set to `auto`. The kernel would briefly sleep the network interface during idle periods, severing the iSCSI connection. PostgreSQL — which requires constant disk I/O — would immediately panic.

Fixed by setting `power/control` to `on` on both nodes with a systemd service to persist across reboots. See `docs/cnpg/iscsi-fix-and-backup-testing.md` for the full procedure.

The positive outcome: CNPG's HA failover worked exactly as designed. When the primary lost storage, the replica was promoted automatically within seconds. The architecture proved itself under a real failure — just not the kind of failure we expected.

**WAL archiving fails with "Expected empty archive" after cluster recreation.**

When a CNPG cluster is deleted and recreated, it gets a new database system identifier. The Minio backup bucket still contains WAL files from the previous incarnation. Barman's archive check detects the identifier mismatch and refuses to archive new WAL files.

The fix is to empty the `cnpg-backups` bucket before recreating the cluster. The Minio console UI may not fully remove files due to versioning — files remain on disk as delete markers. The reliable method is deleting from the QNAP filesystem directly using a temporary Alpine container running as root:

```bash
docker run --rm -v /share/CACHEDEV1_DATA/minio/data:/data alpine \
  rm -rf /data/cnpg-backups/cnpg-cluster/
```

**Minio runs outside Kubernetes — intentionally.**

Minio is deployed via Docker Compose directly on the QNAP, not inside the k3s cluster. This is a deliberate architectural choice: if the Kubernetes cluster dies, the backup target must still be running. Backups that live and die with the thing they're backing up are not backups.

**One shared CNPG cluster, not per-app databases.**

A single 2-instance cluster serves all applications. Individual app databases are created as PostgreSQL databases within the cluster — not as separate CNPG Cluster resources. This avoids doubling iSCSI LUNs and operator overhead on a 2-node cluster with limited resources.

**Namespace has no PodSecurity labels.**

The `cnpg` namespace does not enforce `restricted` PodSecurity. CNPG's bootstrap process runs root-level init containers (installing the controller manager binary). This matches the approach used for `monitoring` namespace with kube-prometheus-stack — infrastructure components require elevated privileges that application namespaces should not have.

---

## Storage Sizing

| Component | Size | Rationale |
|---|---|---|
| Primary (LUN 7, zoro) | 25Gi | Shared across all app databases. Real usage will grow with linkding, future apps. 25Gi provides significant headroom on a homelab. |
| Replica (LUN 6, luffy) | 25Gi | Mirrors primary via streaming replication. Same size required. |
| Minio backup bucket | Unbounded | Lives on QNAP main storage pool. Base backups ~50–200Mi each. WAL files compressed with gzip. 30-day retention prevents unbounded growth. |

---

## Bootstrap Procedure

First-time setup requires a one-time volume ownership fix before CNPG can initialize. Full procedure documented in `docs/cnpg/cnpg-deployment-guide.md`. Summary:

```
1. Deploy infrastructure (cert-manager, CNPG operator, Barman plugin)
2. Create iSCSI LUNs on QNAP (25Gi each)
3. Format LUNs with ext4 (mkfs.ext4)
4. Suspend Flux databases kustomization
5. Apply bootstrap PVC + chown Job per node
6. Verify ownership (26:26)
7. Clean up Jobs and PVCs
8. Resume Flux — cluster initializes automatically
9. Trigger test backup — verify in Minio
```

---

## 🚀 What's Next

| Item | Status |
|------|--------|
| Cloudflare R2 offsite backup | Planned — second barmanObjectStore target |
| Linkding migration | Planned — migrate from SQLite to this PostgreSQL cluster |
| Grafana migration | Planned — replace SQLite backend with PostgreSQL |
| PodMonitor for Prometheus | Planned — scrape CNPG metrics for dashboards |
| Disaster recovery test | Planned — full restore from Barman backup to validate the chain |
| Connection pooling | Not planned — PgBouncer is overkill for homelab traffic |

---

## 🔗 Related

- [Homelab Overview](https://github.com/AhsanRahat12/Homelab)
- [Monitoring Stack](../Kube-Prometheus-Stack/README.md)
- [GitHub Profile](https://github.com/AhsanRahat12)
