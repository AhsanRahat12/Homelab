# 🔖 Linkding

Self-hosted bookmark manager deployed on Kubernetes via GitOps. [sissbruecker/linkding](https://github.com/sissbruecker/linkding)

---

## Stack

| Concern | Solution |
|---------|----------|
| Deployment | Flux GitOps — no manual `kubectl apply` |
| Secrets | SOPS + Age encryption, safe to store in public Git |
| Storage (staging) | local-path PVC — 200Mi, SD card constrained, intentional |
| Storage (production) | democratic-csi → iSCSI LUN on QNAP NAS — 1Gi Thick provisioned, data off the Pi entirely |
| CSI Driver | democratic-csi node-manual — handles iSCSI attach/detach between nodes automatically |
| External access | Cloudflare Tunnel — no open ports or port forwarding (production only) |
| Local access | Traefik Ingress + Cloudflare DNS A record |
| Image updates | Renovate CronJob — automated PRs on new releases |
| Security | Non-root container (www-data, UID 33), privilege escalation blocked |

---

## 📁 Repo Structure

```
Homelab/
  pi-zoro/
    apps/
      base/linkding/          ← environment-agnostic: deployment, service, PVC
      staging/linkding/       ← staging-specific: encrypted secrets, PVC size patch, namespace
      production/linkding/    ← production-specific: iSCSI PV, Cloudflare tunnel, encrypted secrets
    clusters/
      staging/                ← Flux entry point, SOPS config
      staging/apps-production.yaml ← bootstraps production Kustomization from within staging cluster dir
    infrastructure/
      controllers/
        base/democratic-csi/  ← CSI driver: HelmRelease, StorageClass, CHAP secret
        staging/democratic-csi/ ← namespace + overlay wiring
    monitoring/               ← kube-prometheus-stack
  docs/
    linkding/
      README.md               ← you are here
    migrations/
      migrate-linkding.yaml   ← migration pod used to transfer data from staging to production PVC
  README.md                   ← homelab overview
```

Base defines what Linkding needs to run. The staging layer stamps the namespace, reduces PVC size for SD card constraints, and adds environment-specific secrets. The production layer points at the same base and overrides storage to iSCSI — the delta lives entirely in the environment overlay, base is untouched.

---

## 🧠 Problems & Decisions

**Container ran as root.** Applied `runAsUser`, `runAsGroup`, and `fsGroup: 33` (www-data) at the pod level. `fsGroup` was the critical one — without it the mounted volume wasn't writable despite running as the correct user.

**Secrets in Git.** Kubernetes secrets are base64, not encrypted. Chose SOPS + Age for Flux's native integration — Flux decrypts at deploy time, the private key never touches the repo.

**Admin user bootstrapping.** Injected `LD_SUPERUSER_NAME` and `LD_SUPERUSER_PASSWORD` via encrypted secret and `envFrom.secretRef` to eliminate manual `createsuperuser` after every deploy.

**Storage evolution — three stages to get it right.**

The storage architecture for linkding went through three distinct stages, each solving a problem the previous stage introduced:

```
Stage 1 — local-path (SD card)
  Data lives on the Pi's SD card
  SD cards are unreliable under write-heavy SQLite workloads
  No failover — pod and data are on the same node

Stage 2 — Static PV + manual iSCSI mount (node affinity)
  Data moved off the Pi to QNAP NAS via iSCSI LUN
  LUN manually mounted on Zoro via fstab
  Static PV with nodeSelector: zoro
  Problem: pod is now pinned to Zoro
  Zoro goes down → linkding goes down, no recovery possible

Stage 3 — democratic-csi (current)
  Data still on QNAP NAS via iSCSI LUN
  democratic-csi manages the full attach/detach lifecycle
  No nodeSelector — pod runs on either Zoro or Luffy
  Node failure is now survivable — pod reschedules, LUN follows
```

**Why democratic-csi instead of a static PV.** A static PV with a local hostPath mount required a `nodeSelector` pinning the pod to Zoro. If Zoro went down, linkding went down with it — no recovery possible. democratic-csi manages the full iSCSI attach/detach lifecycle, removing the nodeSelector entirely. The pod now runs on whichever node Kubernetes picks. If Zoro fails, democratic-csi detaches the LUN from Zoro and reattaches to Luffy automatically.

**Live data migration.** Scaled staging to 0 to stop writes, ran a migration pod mounting both the old hostPath and new iSCSI PVC simultaneously, copied data with `rsync -av`. Cross-namespace PVC limitation meant PVCs couldn't be mounted directly — used hostPath volumes pointing at the underlying filesystem paths instead.

**Cloudflare tunnel conflict.** One tunnel, two environments trying to claim it. Scaled staging Cloudflare to 0, deployed production pointing at the full cluster DNS (`linkding.linkding-prod.svc.cluster.local:9090`), removed Cloudflare from staging. Staging is now internal only via port-forward.

**Base PVC size.** Reduced to 200Mi — production data lives on QNAP, SD card space is limited. Production patches up to 1Gi via `patch-pvc.yaml`.

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
| Resource limits | Pending — set after reviewing 1 week of Prometheus metrics |
| PostgreSQL backend | Pending — replace SQLite with PostgreSQL to enable stateless pods and multiple replicas. SQLite is a single-writer database tied to a single pod. PostgreSQL decouples the database from the application layer — linkding pods become stateless, horizontal scaling becomes possible, and failover is cleaner. Demonstrates how separating state from compute changes the operational model entirely. |
| Readiness and liveness probes | Without them Kubernetes sends traffic to a pod the moment it starts. Readiness holds traffic back, liveness restarts if it stops responding. |
| TLS with cert-manager | Current self-signed cert triggers browser warnings. cert-manager with Let's Encrypt automates the full certificate lifecycle. |
| Secrets provider upgrade | SOPS + Age is solid for homelab. In a team environment, AWS KMS or HashiCorp Vault is the right call — centralised key management, audit logs, proper access policies. |

---

## 🔗 Related

- [Homelab Overview](https://github.com/AhsanRahat12/Homelab)
- [GitHub Profile](https://github.com/AhsanRahat12)
