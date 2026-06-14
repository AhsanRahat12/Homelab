# 📚 Audiobookshelf
Self-hosted audiobook and podcast manager deployed on Kubernetes via GitOps. [advplyr/audiobookshelf](https://github.com/advplyr/audiobookshelf)

Four volumes, two storage backends (iSCSI + NFS), tested node failover, zero data on SD card in production.

**Live at** [audiobooks.rahatahsan.com](https://audiobooks.rahatahsan.com)

> **TL;DR:** Designed a split storage architecture (iSCSI for databases, NFS for media) across two nodes with tested failover. Diagnosed and recovered from a real filesystem corruption incident caused by a RollingUpdate/RWO deadlock — including root-causing a "1/1 Running but dead inside" pod with no liveness probe.

![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-326CE5) ![Flux](https://img.shields.io/badge/Flux-GitOps-5468FF) ![SOPS](https://img.shields.io/badge/SOPS-Age%20Encrypted-FFB000) ![Cloudflare](https://img.shields.io/badge/Cloudflare-Tunnel-F38020) ![PSA](https://img.shields.io/badge/PodSecurity-Restricted-CC0000) ![NFS](https://img.shields.io/badge/Storage-iSCSI%20%2B%20NFS-0078D4) ![Renovate](https://img.shields.io/badge/Renovate-Automated%20Updates-1A1F6C)

---

## Architecture

<p align="center">
  <img src="architecture.svg" alt="Audiobookshelf production architecture diagram" width="680"/>
</p>

---

## Stack

| Concern | Solution |
|---------|----------|
| Storage split | iSCSI LUNs (config + metadata, RWO) for the database; NFS shares (audiobooks + podcasts, RWX) for media |
| External access | Cloudflare Tunnel → [audiobooks.rahatahsan.com](https://audiobooks.rahatahsan.com) — no open ports |
| Security | PSA `restricted` enforced — non-root, capabilities dropped, seccomp, read-only filesystem |
| Deploy strategy | `Recreate` — RWO iSCSI volumes require single-pod exclusive access, RollingUpdate causes deadlock |
| Health checks & resources | Readiness/liveness probes on `/healthcheck`; CPU/memory requests and limits tuned from 30 days of Prometheus data |

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

## 🔒 Security

Security is implemented in layers. Each layer assumes the previous one failed.

### Pod Security Admission — namespace enforced

The `audiobookshelf-prod` namespace enforces the `restricted` Pod Security Standard. Any pod that does not meet the standard is rejected at admission time before it ever runs.

```yaml
labels:
  pod-security.kubernetes.io/enforce: restricted
  pod-security.kubernetes.io/warn: restricted
  pod-security.kubernetes.io/warn-version: latest
```

`warn` is kept alongside `enforce` so future violations surface immediately as warnings.

### Pod Security Context

The initial foundation was `runAsUser: 1000`, `runAsGroup: 1000`, `fsGroup: 1000`, and `allowPrivilegeEscalation: false`. Implementing PSA restricted surfaced three additional gaps which were added deliberately.

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 1000
    fsGroup: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop:
        - ALL
```

| Setting | What It Does |
|---------|-------------|
| `runAsNonRoot: true` | Kubernetes rejects the pod if the process would run as root |
| `runAsUser: 1000` | Process runs as the `node` user, not root |
| `capabilities.drop: ALL` | All Linux kernel capabilities stripped — no raw sockets, no module loading, no ptrace |
| `seccompProfile: RuntimeDefault` | Blocks dangerous kernel syscalls used in container breakout attacks |
| `allowPrivilegeEscalation: false` | Process cannot gain more privileges mid-run — sudo and setuid binaries are blocked |
| `readOnlyRootFilesystem: true` | Container cannot write to its own filesystem — no persistence for an attacker |

All writes go to dedicated mounted volumes (PVCs), not the container filesystem.

---

## 🧠 Key Engineering Decisions

**RollingUpdate + RWO caused deadlock and filesystem corruption — June 2026.** No explicit deploy strategy meant Kubernetes defaulted to RollingUpdate. Renovate bumped the image, the new pod sat in `ContainerCreating` for 24 hours because the RWO iSCSI volumes were held by the old pod. `kubectl rollout restart` forced both pods to die simultaneously — the unclean detach corrupted the ext4 journal on the metadata LUN. Every write threw `EIO`. The pod showed `1/1 Running` with zero restarts while completely dead inside — there was no liveness probe to catch it. Recovery came when Flux reconciled and democratic-csi provisioned a fresh device attachment. Fix: `strategy: Recreate` added explicitly. The corrupted metadata LUN still needs `fsck` at the next maintenance window.

**Why iSCSI for config/metadata, NFS for audiobooks/podcasts.** Config holds the SQLite database and app settings; metadata holds book covers and thumbnails — both write-heavy, structured, and need exclusive block device access (iSCSI). Audiobooks and podcasts are large, read-heavy, append-only media files with no concurrent-write risk — NFS (ReadWriteMany) lets both nodes mount the share simultaneously with no attach/detach needed during failover.

```
/config     → iSCSI LUN 1 (2Gi)  — block storage, RWO, exclusive access
/metadata   → iSCSI LUN 2 (2Gi)  — block storage, RWO, exclusive access
/audiobooks → NFS share (10Gi)   — network share, RWX, mounts on any node
/podcasts   → NFS share (10Gi)   — network share, RWX, mounts on any node
```

**Implementing Pod Security Admission `restricted`.** Started with `warn` mode to identify violations without breaking the running pod. Three gaps were found: `capabilities.drop: ALL` missing, `runAsNonRoot: true` not set despite `runAsUser: 1000` being present, and no `seccompProfile`. Each was added deliberately, then the namespace was flipped from `warn` to `enforce` once the pod ran clean.

**Resource limits — deliberately leaving CPU uncapped.** 30 days of Prometheus data showed the current pod's steady-state at ~58–61MB / <2% of a core, but four separate historical ReplicaSets all showed the same ~1.0–1.14 core spike on startup (library scan/indexing) — a recurring pattern, not noise. A CPU limit would need to sit above that spike to avoid throttling every restart — on a 4-core Pi that's ~25% of total node CPU for a limit that does nothing 99% of the time. Concluded the limit wouldn't meaningfully constrain anything: memory got request 80Mi / limit 160Mi (sized off a ~110MB observed ceiling), CPU got a 20m request for scheduling and no limit. Caveat: if multiple apps' startup spikes ever overlap (e.g. after a node reboot), the uncapped spike could briefly starve the Pi — not currently a problem, worth revisiting if the cluster grows.

<details>
<summary><strong>Additional implementation notes</strong></summary>

**Container ran as root.** Confirmed via `cat /etc/passwd` inside the container. Identified `node` (UID 1000, GID 1000) as the intended user. `fsGroup` was specifically required to make mounted volumes writable — without it the app gets permission denied despite running as the correct user.

**Secrets weren't safe to commit.** Kubernetes secrets are base64-encoded, not encrypted. Chose SOPS + Age for Flux's native integration — Flux holds the private key inside the cluster and decrypts at deploy time.

**Storage architecture evolution.**

```
Stage 1 — local-path (SD card): all four volumes on the Pi's SD card, no failover
Stage 2 — democratic-csi + NFS (current): config/metadata on iSCSI, audiobooks/
  podcasts on NFS, no nodeSelector, single node failure is survivable
```

**fstab cleanup before democratic-csi.** Both iSCSI LUNs were previously mounted on Zoro via fstab from manual setup. fstab and democratic-csi cannot both manage the same block device — entries were removed from Zoro before Flux applied the PVs.

**QNAP NFS fsid cache bug.** Renaming an NFS share does not generate a new fsid — QNAP caches the original, causing inconsistent mounts between nodes. Fix: delete the share completely, restart the NFS service on QNAP, recreate fresh.

**PVC accessMode immutability.** Changing base PVCs from `ReadWriteOnce` to `ReadWriteMany` caused Flux to fail — Kubernetes does not allow mutating accessModes on a bound PVC. Fix: scale deployment to 0, remove finalizers, force delete PVCs, force Flux reconcile.

**Cloudflare tunnel moved from staging to production.** Staging had the only Cloudflare tunnel; moved to production as part of this migration.

**Readiness and liveness probes added — June 2026.** ABS exposes `/healthcheck` on port 3005. Startup confirmed at 613ms from timestamped logs — no startupProbe needed. Readiness at 5s, liveness at 30s — readiness always fires first. Liveness period kept generous at 20s to avoid false positives during library scans.

**Resource sizing detail.** cloudflared sidecar: requests 10m/32Mi, limit 64Mi memory — same small, stable footprint observed across all three apps' tunnels. Applied via Kustomize JSON6902 `op: add` patches (base deployment had no existing `resources` field). A fresh restart post-deploy showed CPU ~15.1m and memory climbing to ~86MB — comfortably within the new limits.

</details>

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

## 🚀 What's Next

Resource limits have been set (see Key Engineering Decisions). No outstanding items currently tracked for this app.

---

## 🔗 Related

- [Homelab Overview](https://github.com/AhsanRahat12/Homelab)
- [GitHub Profile](https://github.com/AhsanRahat12)
