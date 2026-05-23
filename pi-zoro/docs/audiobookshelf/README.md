# 📚 Audiobookshelf
Self-hosted audiobook and podcast manager deployed on Kubernetes via GitOps. [advplyr/audiobookshelf](https://github.com/advplyr/audiobookshelf)

Four volumes, two storage backends (iSCSI + NFS), tested node failover, zero data on SD card in production.

**Live at** [audiobooks.rahatahsan.com](https://audiobooks.rahatahsan.com)

---

## Architecture

<p align="center">
  <img src="architecture.svg" alt="Audiobookshelf production architecture diagram" width="680"/>
</p>

---

## Stack

| Concern | Solution |
|---------|----------|
| Deployment | Flux GitOps — no manual `kubectl apply` |
| Secrets | SOPS + Age encryption, safe to store in public Git |
| Storage — config + metadata | democratic-csi → iSCSI LUNs on QNAP NAS — 2Gi each, data off the Pi |
| Storage — audiobooks + podcasts | NFS shares on QNAP NAS — 10Gi each, ReadWriteMany |
| CSI Driver | democratic-csi node-manual — handles iSCSI attach/detach between nodes automatically |
| External access | Cloudflare Tunnel → [audiobooks.rahatahsan.com](https://audiobooks.rahatahsan.com) — no open ports or port forwarding (production only) |
| Image updates | Renovate CronJob — automated PRs on new releases |
| Security | Non-root container (node, UID 1000), privilege escalation blocked |
| Staging | local-path for all volumes — kept intentionally for portfolio contrast |

---

## 📁 Repo Structure

```
apps/
  base/audiobookshelf/          ← deployment, service, PVCs, configmap (shared)
  staging/audiobookshelf/       ← local-path, internal only, no Cloudflare
  production/audiobookshelf/    ← iSCSI PVs, NFS PVs, PVC patches, Cloudflare tunnel
clusters/
  staging/                      ← Flux entry point, SOPS config
infrastructure/
  controllers/
    base/democratic-csi/        ← HelmRelease, StorageClass, CHAP secret (shared with linkding)
docs/
  audiobookshelf/README.md      ← you are here
```

Base defines what Audiobookshelf needs to run. The staging layer stamps the namespace with local-path storage — no Cloudflare, no NAS. The production layer points at the same base and overrides all four volumes via patches. The delta lives entirely in the environment overlay, base is untouched.

---

## 🧠 Problems & Decisions

**Container ran as root.** Confirmed via `cat /etc/passwd` inside the container. Identified `node` (UID 1000, GID 1000) as the intended user. Applied `runAsUser`, `runAsGroup`, and `fsGroup` at the pod level and `allowPrivilegeEscalation: false` at the container level. `fsGroup` is specifically required to make mounted volumes writable — without it the app gets permission denied errors despite running as the correct user.

**Secrets weren't safe to commit.** Kubernetes secrets are base64-encoded, not encrypted — effectively plain text in Git. Chose SOPS + Age because Flux has native integration requiring no extra tooling. Flux holds the private key inside the cluster and decrypts at deploy time. The key never touches the repo.

**Why iSCSI for config and metadata, NFS for audiobooks and podcasts.**

Config holds the SQLite database and app settings. Metadata holds book covers and cached thumbnails. Both are write-heavy, structured, and need exclusive block device access — iSCSI is the right storage type. NFS would introduce consistency risks for a live database.

Audiobooks and podcasts are large media files — read-heavy, append-only, no concurrent write risk. NFS is the right storage type here. It is ReadWriteMany, meaning both nodes can mount the share simultaneously with no attach/detach needed during failover.

```
/config     → iSCSI LUN 1 (2Gi)  — block storage, RWO, exclusive access
/metadata   → iSCSI LUN 2 (2Gi)  — block storage, RWO, exclusive access
/audiobooks → NFS share (10Gi)   — network share, RWX, mounts on any node
/podcasts   → NFS share (10Gi)   — network share, RWX, mounts on any node
```

**Storage architecture evolution.**

```
Stage 1 — local-path (SD card)
  All four volumes on the Pi's SD card
  Unreliable for databases, no failover possible

Stage 2 — Production environment with democratic-csi + NFS
  Config and metadata → iSCSI LUNs on QNAP, managed by democratic-csi
  Audiobooks and podcasts → NFS shares on QNAP
  No nodeSelector — pod runs on either Zoro or Luffy
  Single node failure is now survivable
```

**fstab cleanup before democratic-csi.** Both iSCSI LUNs were previously mounted on Zoro via fstab from manual setup. fstab and democratic-csi cannot both manage the same block device — fstab entries were removed from Zoro before Flux applied the PVs. This was the only manual step in the migration.

**QNAP NFS fsid cache bug.** During NFS share setup, renaming a share does not generate a new fsid — QNAP caches the original. This caused inconsistent mount behaviour between nodes. Fix: delete the share completely, restart the NFS service on QNAP, recreate fresh. Both nodes then mounted consistently.

**PVC accessMode immutability.** Changing base PVCs from `ReadWriteOnce` to `ReadWriteMany` caused Flux to fail — Kubernetes does not allow mutating accessModes on a bound PVC. Fix: scale deployment to 0, remove finalizers, force delete PVCs, force Flux reconcile. PVCs were recreated fresh with the correct spec.

**Cloudflare tunnel moved from staging to production.** Staging had the only Cloudflare tunnel. It was moved to production as part of this migration — staging is now internal only. Production is the only environment with external access.

---

## 🔄 Failover

Pod successfully rescheduled from Luffy to Zoro with all four volumes following.

- iSCSI volumes (config, metadata) — democratic-csi detached from Luffy, reattached to Zoro automatically
- NFS volumes (audiobooks, podcasts) — ReadWriteMany, no detach needed, mounted on Zoro immediately

```
Before: audiobookshelf running on Luffy
Cordon Luffy → delete pod → reschedule
After:  audiobookshelf running on Zoro — all data intact, app accessible
```

Staging intentionally kept on local-path — no failover, no NAS. The contrast between staging and production demonstrates the architectural difference clearly.

---

---

## 🚀 What's Next

| Item | Status |
|------|--------|
| Resource limits | Planned — measure with Prometheus before setting |
| Readiness and liveness probes | Planned |

---

## 🔗 Related

- [Homelab Overview](https://github.com/AhsanRahat12/Homelab)
- [GitHub Profile](https://github.com/AhsanRahat12)
