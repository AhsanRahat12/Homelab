# 📚 Audiobookshelf

Self-hosted audiobook and podcast manager deployed on Kubernetes via GitOps. [advplyr/audiobookshelf](https://github.com/advplyr/audiobookshelf)

---

## Stack

| Concern | Solution |
|---------|----------|
| Deployment | Flux GitOps — no manual `kubectl apply` |
| Secrets | SOPS + Age encryption, safe to store in public Git |
| Storage | PersistentVolumeClaims on local-path (NFS planned for production) |
| External access | Cloudflare Tunnel — no open ports or port forwarding |
| Image updates | Renovate CronJob — automated PRs on new releases |
| Security | Non-root container (node, UID 1000), privilege escalation blocked |

---

## 📁 Repo Structure

```
Homelab/
  pi-zoro/
    apps/
      base/audiobookshelf/          ← environment-agnostic: deployment, service, PVCs, configmap
      staging/audiobookshelf/       ← staging-specific: encrypted tunnel secret, cloudflared, kustomization
    clusters/
      staging/                      ← Flux entry point, SOPS config
    infrastructure/                 ← Renovate and other controllers
    monitoring/                     ← kube-prometheus-stack
  docs/
    audiobookshelf/
      README.md                     ← you are here
  README.md                         ← homelab overview
```

Base defines what Audiobookshelf needs to run. The staging layer stamps the namespace and adds environment-specific resources on top. The separation means a production environment can point at the same base without touching any shared files — only the delta lives in the environment layer.

---

## 🧠 Problems & Decisions

**Data lost on pod restart.** Container storage is ephemeral by default. Added four PVCs mounted at `/config`, `/metadata`, `/audiobooks`, and `/podcasts` — data now survives pod restarts and redeployments.

**Container ran as root.** Confirmed via `cat /etc/passwd` inside the container. Identified `node` (UID 1000, GID 1000) as the intended user. Applied `runAsUser`, `runAsGroup`, and `fsGroup` at the pod level and `allowPrivilegeEscalation: false` at the container level. `fsGroup` is specifically required to make mounted volumes writable — without it the app gets permission denied errors despite running as the correct user.

**Existing volume had wrong ownership after adding securityContext.** The pod previously ran as root, creating SQLite database files owned by `root:root`. When `runAsUser: 1000` was applied, the node user couldn't write to those files, causing `SQLITE_READONLY` crash loops. In staging, the config PVC was deleted and recreated fresh with correct ownership. In production the fix is an init container that runs as root, chowns the volume, then exits before the main container starts.

**Secrets weren't safe to commit.** Kubernetes secrets are base64-encoded, not encrypted — effectively plain text in Git. Chose SOPS + Age because Flux has native integration requiring no extra tooling. Flux holds the private key inside the cluster and decrypts at deploy time. The key never touches the repo.

**External access without open ports.** Cloudflare Tunnel provides outbound-only connectivity — no port forwarding, no exposed IPs. A dedicated tunnel was created for Audiobookshelf with its own credentials file, encrypted with SOPS and deployed through Flux.

---

## 🚀 What's Next — Production Readiness

**NFS-backed storage.** `local-path` binds PVCs to the node's disk — media libraries belong on the QNAP NAS, not an SD card. The fix is NFS dynamic provisioning via `nfs-subdir-external-provisioner`, implemented as a Kustomize patch so staging is unaffected.

**Resource limits.** An unbounded container can starve other workloads on the node. Defining requests and limits sets the guaranteed floor and ceiling for CPU and memory.

**Readiness and liveness probes.** Without them Kubernetes sends traffic to a pod the moment it starts, before the app is ready. Readiness holds traffic back, liveness restarts the pod if it stops responding.

**TLS with cert-manager.** Current setup relies on Cloudflare Tunnel for HTTPS termination. A local TLS cert via cert-manager and Let's Encrypt would secure internal access as well.

---

## 🔗 Related

- [Homelab Overview](https://github.com/AhsanRahat12/Homelab)
- [GitHub Profile](https://github.com/AhsanRahat12)
