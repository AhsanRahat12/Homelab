# 🔖 Linkding
Self-hosted bookmark manager deployed on Kubernetes via GitOps. [sissbruecker/linkding](https://github.com/sissbruecker/linkding)

---

## Stack

| Concern | Solution |
|---------|----------|
| Deployment | Flux GitOps — no manual `kubectl apply` |
| Secrets | SOPS + Age encryption, safe to store in public Git |
| Storage (staging) | PersistentVolumeClaim on local-path (200Mi — SD card constrained) |
| Storage (production) | iSCSI LUN on QNAP NAS — 1Gi Thick provisioned, data off the Pi entirely |
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
      production/linkding/    ← production-specific: iSCSI PV/PVC, Cloudflare tunnel, encrypted secrets
    clusters/
      staging/                ← Flux entry point, SOPS config
      staging/apps-production.yaml ← bootstraps production Kustomization from within staging cluster dir
    infrastructure/           ← Renovate and other controllers
    monitoring/               ← kube-prometheus-stack
  docs/
    linkding/
      README.md               ← you are here
    migrations/
      migrate-linkding.yaml   ← migration pod used to transfer data from staging to production PVC
  README.md                   ← homelab overview
```

Base defines what Linkding needs to run. The staging layer stamps the namespace, reduces PVC size for SD card constraints, and adds environment-specific resources. The production layer points at the same base and overrides storage to iSCSI — the delta lives entirely in the environment layer, base is untouched.

---

## 🧠 Problems & Decisions

**Container ran as root.** Applied `runAsUser`, `runAsGroup`, and `fsGroup: 33` (www-data) at the pod level. `fsGroup` was the critical one — without it the mounted volume wasn't writable despite running as the correct user.

**Secrets in Git.** Kubernetes secrets are base64, not encrypted. Chose SOPS + Age for Flux's native integration — Flux decrypts at deploy time, the private key never touches the repo.

**Admin user bootstrapping.** Injected `LD_SUPERUSER_NAME` and `LD_SUPERUSER_PASSWORD` via encrypted secret and `envFrom.secretRef` to eliminate manual `createsuperuser` after every deploy.

**Storage bound to SD card.** `local-path` ties the PVC to the node's SD card — constrained and unreliable for SQLite. Migrated production to an iSCSI LUN on QNAP NAS — block storage over the network, safe for databases, data fully off the Pi.

**Live data migration.** Scaled staging to 0 to stop writes, ran a migration pod mounting both the old hostPath and new iSCSI mount simultaneously, copied data with `rsync -av`. Cross-namespace PVC limitation meant PVCs couldn't be mounted directly — used hostPath volumes pointing at the underlying filesystem paths instead.

**Cloudflare tunnel conflict.** One tunnel, two environments trying to claim it. Scaled staging Cloudflare to 0, deployed production pointing at the full cluster DNS (`linkding.linkding-prod.svc.cluster.local:9090`), removed Cloudflare from staging. Staging is now internal only via port-forward.

**Base PVC size.** Reduced to 200Mi — production data lives on QNAP, SD card space is limited. Production patches up to 1Gi via `patch-pvc.yaml`.

---

## 🚀 What's Next — Production Readiness

| Item | Status |
|------|--------|
| ✅ Migrate storage from SD card to iSCSI on QNAP | `local-path` binds the PVC to the node's SD card — constrained, unreliable for databases. Migrated to an iSCSI LUN on QNAP NAS. Block storage over the network, data fully off the Pi. |
| Resource limits and HPA | An unbounded container can starve other workloads on the node. Defining requests and limits sets the guaranteed floor and ceiling. HPA builds on top, scaling replicas automatically using those values as the signal. |
| Readiness and liveness probes | Without them Kubernetes sends traffic to a pod the moment it starts, before the app is ready. Readiness holds traffic back, liveness restarts the pod if it stops responding. |
| TLS with cert-manager | The current self-signed cert triggers browser warnings and requires manual rotation. cert-manager with Let's Encrypt automates the full certificate lifecycle. |
| Secrets provider upgrade | SOPS + Age is solid for a homelab. In a team environment, AWS KMS or HashiCorp Vault is the right call — centralised key management, audit logs, and proper access policies. |

---

## 🔗 Related

- [Homelab Overview](https://github.com/AhsanRahat12/Homelab)
- [GitHub Profile](https://github.com/AhsanRahat12)
