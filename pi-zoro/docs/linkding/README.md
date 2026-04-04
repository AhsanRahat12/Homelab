# 🔖 Linkding

Self-hosted bookmark manager deployed on Kubernetes via GitOps. [sissbruecker/linkding](https://github.com/sissbruecker/linkding)

---

## Stack

| Concern | Solution |
|---------|----------|
| Deployment | Flux GitOps — no manual `kubectl apply` |
| Secrets | SOPS + Age encryption, safe to store in public Git |
| Storage | PersistentVolumeClaim on local-path (NFS planned for production) |
| External access | Cloudflare Tunnel — no open ports or port forwarding |
| Local access | Traefik Ingress + Cloudflare DNS A record |
| Image updates | Renovate CronJob — automated PRs on new releases |
| Security | Non-root container (www-data, UID 33), privilege escalation blocked |

---

## 📁 Repo Structure

```
Homelab/
  pi-zoro/
    apps/
      base/linkding/          ← environment-agnostic: deployment, service, PVC, namespace
      staging/linkding/       ← staging-specific: encrypted secrets, kustomization
    clusters/
      staging/                ← Flux entry point, SOPS config
    infrastructure/           ← Renovate and other controllers
    monitoring/               ← kube-prometheus-stack
  docs/
    linkding/
      README.md               ← you are here
  README.md                   ← homelab overview
```

Base defines what linkding needs to run. The staging layer stamps the namespace and adds environment-specific resources on top. The separation means a production environment can point at the same base without touching any shared files — only the delta lives in the environment layer.

---

## 🧠 Problems & Decisions

**Data lost on pod restart.** Container storage is ephemeral by default. Added a PVC mounted at `/etc/linkding/data` and attached it to the deployment — data now survives pod restarts and redeployments.

**Container ran as root.** Confirmed via `id` inside the container — `uid=0`. Identified `www-data` (UID 33) as the intended user via `/etc/passwd`. Applied `runAsUser`, `runAsGroup`, and `fsGroup` at the pod level and `allowPrivilegeEscalation: false` at the container level. `fsGroup` was specifically required to make the mounted volume writable — without it the app couldn't write to its own data directory despite running as the correct user.

**Secrets weren't safe to commit.** Kubernetes secrets are base64-encoded, not encrypted — effectively plain text in Git. Chose SOPS + Age because Flux has native integration requiring no extra tooling. Flux holds the private key inside the cluster and decrypts at deploy time. The key never touches the repo.

**Admin user required manual intervention after every deploy.** Had to exec into the pod and run `createsuperuser` by hand — breaking the GitOps goal of full automation. Solved by injecting `LD_SUPERUSER_NAME` and `LD_SUPERUSER_PASSWORD` via an encrypted secret and `envFrom.secretRef`. The app now bootstraps itself on first boot.

**Two access patterns with different risk profiles.** External traffic uses Cloudflare Tunnel — outbound-only, no open ports. Local traffic uses Traefik Ingress with a Cloudflare DNS A record pointing to the Pi's private IP (proxy off — Cloudflare can't proxy to a private IP). Local traffic never leaves the network.

---

## 🚀 What's Next — Production Readiness

**NFS-backed storage.** `local-path` binds the PVC to one node's disk — if the pod reschedules it loses access to its data. The fix is NFS dynamic provisioning against the QNAP NAS, removing the node dependency and moving data off the Pi's SD card. Implemented as a Kustomize patch so staging is unaffected.

**Resource limits and HPA.** An unbounded container can starve other workloads on the node. Defining requests and limits sets the guaranteed floor and ceiling. HPA builds on top, scaling replicas automatically using those values as the signal.

**Readiness and liveness probes.** Without them Kubernetes sends traffic to a pod the moment it starts, before the app is ready. Readiness holds traffic back, liveness restarts the pod if it stops responding.

**TLS with cert-manager.** The current self-signed cert triggers browser warnings and requires manual rotation. cert-manager with Let's Encrypt automates the full certificate lifecycle.

**Secrets provider upgrade.** SOPS + Age is solid for a homelab. In a team environment, AWS KMS or HashiCorp Vault is the right call — centralised key management, audit logs, and proper access policies.

---

## 🔗 Related

- [Homelab Overview](https://github.com/AhsanRahat12/Homelab)
- [GitHub Profile](https://github.com/AhsanRahat12)
