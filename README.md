# Homelab

Self-hosted infrastructure running on a Raspberry Pi 5 cluster, managed with Flux for GitOps continuous deployment.

---

## Philosophy

Everything is defined as code, version controlled, and applied declaratively. No manual `kubectl apply`, no SSH-ing into nodes to make changes. The Git repo is the single source of truth for what runs in the cluster — Flux handles the rest.

---

## Hardware

| Node | Hardware | Role |
|---|---|---|
| `pi-zoro` | Raspberry Pi 5 (16GB) | Primary cluster node |
| `pi-luffy` | Raspberry Pi 5 (16GB) | Secondary cluster node (coming soon) |
| `nas` | QNAP TS-464 | Network attached storage |

**Network:** Private overlay network via Tailscale

---

## Stack

| Tool | Purpose |
|---|---|
| K3S | Lightweight Kubernetes distribution |
| Flux | GitOps controller — syncs cluster state with this repo |
| Kustomize | Environment and namespace configuration |

---

## Structure

```
homelab/
  pi-zoro/
    linkding/         ← self-hosted bookmark manager
    audiobookshelf/   ← self-hosted audiobook server (coming soon)
  pi-luffy/           ← coming soon
```

Each folder contains the Kubernetes manifests for that project. Flux watches this repo and automatically applies any changes to the cluster.

---

## Projects

- **[Linkding](./pi-zoro/linkding)** — Self-hosted bookmark manager running on `pi-zoro`

---

---

*Part of my broader DevOps learning journey — [main profile](https://github.com/AhsanRahat12)*
