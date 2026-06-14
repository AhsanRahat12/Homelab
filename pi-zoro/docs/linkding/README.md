# 🔖 Linkding
Self-hosted bookmark manager deployed on Kubernetes via GitOps. [sissbruecker/linkding](https://github.com/sissbruecker/linkding)

Three-stage storage migration from SD card to fully automated iSCSI failover. No nodeSelector, no single point of failure.

**Live at** [links.rahatahsan.com](https://links.rahatahsan.com)

> **TL;DR:** Migrated a stateful app through three storage architectures — SD card → node-pinned static PV → fully automated iSCSI failover with no nodeSelector. Diagnosed and fixed an iSCSI volume deadlock during rolling restarts, then implemented Pod Security Admission `restricted` from the ground up.

![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-326CE5) ![Flux](https://img.shields.io/badge/Flux-GitOps-5468FF) ![SOPS](https://img.shields.io/badge/SOPS-Age%20Encrypted-FFB000) ![Cloudflare](https://img.shields.io/badge/Cloudflare-Tunnel-F38020) ![PSA](https://img.shields.io/badge/PodSecurity-Restricted-CC0000) ![Renovate](https://img.shields.io/badge/Renovate-Automated%20Updates-1A1F6C)

---

## Architecture

<p align="center">
  <img src="architecture.svg" alt="Linkding production architecture diagram" width="680"/>
</p>

---

## Stack

| Concern | Solution |
|---------|----------|
| Storage | democratic-csi → iSCSI LUN on QNAP, no nodeSelector — pod runs on either node, storage follows |
| External access | Cloudflare Tunnel → [links.rahatahsan.com](https://links.rahatahsan.com) — no open ports |
| Security | PSA `restricted` enforced — non-root, capabilities dropped, seccomp, read-only filesystem |
| Deploy strategy | `Recreate` — RWO iSCSI requires single-pod exclusive access, RollingUpdate causes deadlock |
| Health checks & resources | Readiness/liveness probes on `/health`; CPU/memory requests and limits tuned from 30 days of Prometheus data |

---

## 📁 Repo Structure

```
apps/
  base/linkding/              ← deployment, service, PVC (shared across environments)
  staging/linkding/           ← namespace, encrypted secrets, PVC size patch
  production/linkding/        ← iSCSI PV, Cloudflare tunnel, encrypted secrets
clusters/
  staging/                    ← Flux entry point, SOPS config
infrastructure/
  controllers/
    base/democratic-csi/      ← HelmRelease, StorageClass, CHAP secret
docs/
  linkding/README.md          ← you are here
  migrations/migrate-linkding.yaml
```

Base defines what linkding needs to run. Staging stamps the namespace, reduces PVC size for SD card constraints, and injects environment-specific secrets. Production points at the same base and overrides storage to iSCSI — the delta lives entirely in the environment overlay, base is untouched.

---

## 🔒 Security

Security is implemented in layers. Each layer assumes the previous one failed.

### Pod Security Admission — namespace enforced

The `linkding-prod` namespace enforces the `restricted` Pod Security Standard. Any pod that does not meet the standard is rejected at admission time before it ever runs.

```yaml
labels:
  pod-security.kubernetes.io/enforce: restricted
  pod-security.kubernetes.io/warn: restricted
  pod-security.kubernetes.io/warn-version: latest
```

`warn` is kept alongside `enforce` so future violations surface immediately as warnings.

### Pod Security Context

The initial foundation was `runAsUser: 33`, `runAsGroup: 33`, `fsGroup: 33` (www-data), and `allowPrivilegeEscalation: false`. Implementing PSA restricted surfaced three additional gaps which were added deliberately.

```yaml
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 33
    runAsGroup: 33
    fsGroup: 33
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
| `runAsUser: 33` | Process runs as `www-data`, not root |
| `capabilities.drop: ALL` | All Linux kernel capabilities stripped — no raw sockets, no module loading, no ptrace |
| `seccompProfile: RuntimeDefault` | Blocks dangerous kernel syscalls used in container breakout attacks |
| `allowPrivilegeEscalation: false` | Process cannot gain more privileges mid-run — sudo and setuid binaries are blocked |
| `readOnlyRootFilesystem: true` | Container cannot write to its own filesystem — no persistence for an attacker |

`readOnlyRootFilesystem: true` required mounting an `emptyDir` at `/tmp` — linkding's process manager and web server both write pid files and temporary data there at startup. All application data writes go to the dedicated iSCSI-backed PVC.

---

## 🧠 Key Engineering Decisions

**Storage evolution — three stages to get it right.**

```
Stage 1 — local-path (SD card)
  Data lives on the Pi's SD card
  SD cards are unreliable under write-heavy SQLite workloads
  No failover — pod and data are on the same node

Stage 2 — Static PV + manual iSCSI mount (node affinity)
  Data moved off the Pi to QNAP NAS via iSCSI LUN
  LUN manually mounted on Zoro via fstab, static PV with nodeSelector: zoro
  Problem: pod is pinned to Zoro — if Zoro goes down, linkding goes down

Stage 3 — democratic-csi (current)
  Data still on QNAP NAS via iSCSI LUN
  democratic-csi manages the full attach/detach lifecycle
  No nodeSelector — pod runs on either Zoro or Luffy
  Node failure is now survivable — pod reschedules, LUN follows automatically
```

**iSCSI volume deadlock during rolling restarts.** The PVC is `ReadWriteOnce` on iSCSI — only one pod can hold the volume attachment at a time. During a rolling restart the old crashing pod kept the iSCSI lock — it was in CrashLoopBackOff, so Kubernetes kept restarting it before it fully released the attachment. The new pod got stuck in `ContainerCreating` indefinitely. Force-deleting the old pod didn't help — the ReplicaSet immediately spawned a replacement which claimed the lock again. Fix: scale to zero, wait for democratic-csi to complete the detach, scale back to one. This is what eventually drove `strategy: Recreate` for this deployment.

**Implementing Pod Security Admission `restricted`.** Started with `warn` mode so violations surfaced without breaking the running pod. Three gaps were found: unnecessary Linux capabilities, `runAsNonRoot` not set despite a non-root user being configured, and no `seccompProfile`. Making the filesystem read-only also crashed the app — linkding writes pid files and temp data at startup. Fixed with a small `emptyDir` at `/tmp`. Once the pod ran clean with zero warnings, the namespace was flipped from `warn` to `enforce`.

**A resource patch silently surfaced a year-old PodSecurity gap — June 2026.** Adding resource limits to `cloudflared` changed its pod template hash and triggered a rollout. The new ReplicaSet couldn't create pods: `FailedCreate: violates PodSecurity "restricted:latest"` — missing `allowPrivilegeEscalation: false`, capability drops, `runAsNonRoot`, and `seccompProfile`. The *old* pod (61 days old, 16 restarts) kept serving traffic because PodSecurity admission only checks pods at *creation* time — it had been grandfathered in since before the namespace enforced `restricted`. Root cause: `cloudflare.yaml` for linkding had never had a `securityContext` defined, unlike the other two apps' tunnel manifests. Fixed by adding the same restricted-compliant securityContext used elsewhere; the new ReplicaSet came up `2/2` immediately and the old one scaled to 0.

<details>
<summary><strong>Additional implementation notes</strong></summary>

**Container ran as root.** Applied `runAsUser`, `runAsGroup`, and `fsGroup: 33` (www-data) at the pod level. `fsGroup` was the critical one — without it the mounted volume wasn't writable despite running as the correct user.

**Secrets in Git.** Kubernetes secrets are base64, not encrypted. Chose SOPS + Age for Flux's native integration — Flux decrypts at deploy time, the private key never touches the repo.

**Admin user bootstrapping.** Injected `LD_SUPERUSER_NAME` and `LD_SUPERUSER_PASSWORD` via encrypted secret and `envFrom.secretRef` to eliminate manual `createsuperuser` after every deploy.

**Live data migration.** Scaled staging to 0 to stop writes, ran a migration pod mounting both the old hostPath and new iSCSI PVC simultaneously, copied data with `rsync -av`. Cross-namespace PVC limitation meant PVCs couldn't be mounted directly — used hostPath volumes pointing at the underlying filesystem paths instead.

**Cloudflare tunnel conflict.** One tunnel, two environments trying to claim it. Scaled staging Cloudflare to 0, deployed production pointing at the full cluster DNS (`linkding.linkding-prod.svc.cluster.local:9090`), removed Cloudflare from staging.

**Base PVC size.** Reduced to 200Mi — production data lives on QNAP, SD card space is limited. Production patches up to 1Gi via `patch-pvc.yaml`.

**Readiness and liveness probes added — June 2026.** Linkding exposes `/health` on port 9090 — confirmed against the live pod, returns `{"version": "1.45.0", "status": "healthy"}`. Startup confirmed at ~7s from timestamped logs. No startupProbe needed. Readiness at 10s, liveness at 40s — readiness always fires first (golden rule). Liveness period kept at 30s to avoid aggressive restarts on a stateful SQLite app.

**Resource requests/limits added — June 2026.** Sized from 30 days of `container_memory_working_set_bytes` and `container_cpu_usage_seconds_total` in Prometheus, cross-checked against `kubectl get pods/rs` to separate the current pod from stale historical ReplicaSets. Current pod ran ~166–172MB / ~0.036 cores; a previous generation peaked at ~204MB on the same shape of workload — used as the sizing ceiling. Zero OOM events recorded; there was no existing limit, so this was sizing from scratch.

- **Memory:** request 180Mi, limit 320Mi (204MB ceiling + ~50% headroom).
- **CPU:** limit 200m (~5x the observed ~0.036-core peak) — unlike audiobookshelf, linkding has no extreme startup spike, so a real limit was set.
- **cloudflared sidecar:** requests 10m/32Mi, limit 64Mi memory.

Applied via Kustomize JSON6902 `op: add` patches (the base deployment had no existing `resources` field).

</details>

---

## 🔄 Failover

Pod successfully rescheduled from Zoro to Luffy with data intact. democratic-csi detached the iSCSI LUN from Zoro and reattached to Luffy automatically. No manual intervention required.

```
Before: linkding running on Zoro
Cordon Zoro → delete pod → reschedule
After:  linkding running on Luffy — same data, same URL, zero downtime
```

Staging intentionally kept on local-path — no failover, no democratic-csi. The contrast between staging and production demonstrates the architectural difference clearly.

---

## 🚀 What's Next

| Item | Status |
|------|--------|
| PostgreSQL backend | Replace SQLite with PostgreSQL via CloudNative PG. Enables stateless pods, horizontal scaling, and cleaner failover. Next major architectural change. |

---

## 🔗 Related

- [Homelab Overview](https://github.com/AhsanRahat12/Homelab)
- [GitHub Profile](https://github.com/AhsanRahat12)
