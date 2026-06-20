# Actual Budget — Deployment Session Notes

**Date:** 2026-06-20  
**Style:** Step-by-step lecture notes, like working through a problem

---

## 0. What Are We Doing and Why?

Actual Budget is a self-hosted personal finance app. It uses **envelope budgeting** — you assign every pound/dollar to a category before spending it. You access it through a web browser.

Why self-host instead of using a cloud service?
- Your financial data stays on your own hardware
- No subscription fees
- It fits the homelab GitOps pattern we already have

The goal: deploy it the same way we deploy every other app — base manifests, production overlay, iSCSI storage, Cloudflare Tunnel for external access, SOPS-encrypted secrets.

---

## 1. Pre-Deployment — What You Need Before Writing Any Code

Before touching the repo, three things had to exist in the real world:

### 1a. iSCSI LUN on QNAP
Actual Budget stores all data in a SQLite file. SQLite is a single file on disk — so we need a persistent block storage volume. We created a **2Gi iSCSI LUN** on the QNAP NAS.

Key details noted:
```
LUN ID:  8
Name:    actual_budget_data
IQN:     iqn.2004-04.com.qnap:ts-464:iscsi.target-0.88196c  (same as all other LUNs)
Portal:  192.168.1.153:3260
```

> **Why iSCSI and not NFS?** iSCSI gives us a block device (like a hard drive). NFS gives us a network share (like a folder). SQLite needs exclusive block access — it uses file locks that don't work reliably over NFS. So database-style workloads always go on iSCSI here.

### 1b. Cloudflare Tunnel
We need the app accessible at `budget.rahatahsan.com` without opening any firewall ports. Cloudflare Tunnel creates an outbound-only connection from the cluster to Cloudflare's edge.

```bash
cloudflared tunnel create actual_budget
# Output:
# Tunnel ID: 2b57ff16-92b4-4d6a-919a-e674cd8f292f
# Credentials: ~/.cloudflared/2b57ff16-92b4-4d6a-919a-e674cd8f292f.json
```

The JSON file contains the tunnel's authentication credentials. This gets stored as a Kubernetes Secret (SOPS-encrypted).

### 1c. Cloudflare DNS
Last step after everything is deployed — point `budget.rahatahsan.com` at the tunnel in Cloudflare Zero Trust dashboard. Done last so traffic only routes once the app is actually running.

---

## 2. The Base Layer — What Every Environment Shares

Following the repo's existing pattern, we create `apps/base/actual-budget/` with three files that describe the app with no environment-specific details.

### 2a. deployment.yaml

The deployment answers: *what container runs, as which user, with which volumes, with what health checks?*

```yaml
spec:
  strategy:
    type: Recreate        # ← IMPORTANT: explained below
  template:
    spec:
      securityContext:
        runAsUser: 1000   # node user inside the image
        runAsGroup: 1000
        fsGroup: 1000
        runAsNonRoot: true
      containers:
        - name: actual-budget
          image: actualbudget/actual-server:25.6.1
          ports:
            - containerPort: 5006
          volumeMounts:
            - mountPath: /data
              name: actual-budget-data
```

> **Why `Recreate` strategy?** The iSCSI LUN is `ReadWriteOnce` — only one pod can mount it at a time. With `RollingUpdate`, Kubernetes tries to start the new pod before killing the old one. Both pods try to mount the same LUN at the same time — the new pod hangs indefinitely waiting. `Recreate` kills the old pod first, *then* starts the new one. We learned this lesson the hard way with audiobookshelf.

### 2b. service.yaml

Exposes the pod inside the cluster on port 5006. Type `ClusterIP` means it's only reachable inside the cluster — Cloudflare Tunnel reaches it this way.

### 2c. actual-budget-data-pvc.yaml

A generic PVC template — just says "I need 2Gi, ReadWriteOnce." No storage class specified yet. The production overlay will patch in the specific iSCSI binding.

### 2d. kustomization.yaml

Tells Kustomize which files belong to the base:
```yaml
resources:
  - deployment.yaml
  - service.yaml
  - actual-budget-data-pvc.yaml
```

**Commit pattern:** one commit per file. Small, frequent, traceable.

---

## 3. The Production Overlay — What Makes This Real

`apps/production/actual-budget/` adds everything environment-specific.

### 3a. namespace.yaml

Every app gets its own namespace. Namespaces provide isolation — one app's pods can't accidentally touch another's.

```yaml
metadata:
  name: actual-budget-prod
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

`restricted` is the strictest Pod Security Standard. It enforces:
- No root containers
- No privilege escalation
- Capabilities dropped
- Seccomp profile required

> **This label matters a lot later.** Keep it in mind.

### 3b. pv.yaml — The Persistent Volume

The PVC says "I need storage." The PV says "here is that exact storage." They need to match and bind together.

```yaml
spec:
  capacity:
    storage: 2Gi
  csi:
    driver: org.democratic-csi.node-manual
    volumeHandle: actual-budget-data
    volumeAttributes:
      portal: 192.168.1.153:3260
      iqn: iqn.2004-04.com.qnap:ts-464:iscsi.target-0.88196c
      lun: "8"                          # ← the LUN we created on QNAP
```

The `democratic-csi` driver handles attaching/detaching this iSCSI LUN from whichever node the pod lands on. No nodeSelector needed — the LUN follows the pod automatically.

### 3c. patch-pvc.yaml — Binding the PVC to the PV

Without this patch, the PVC floats looking for *any* matching storage. With it, it binds to our specific PV:

```yaml
spec:
  storageClassName: qnap-iscsi-manual
  volumeName: actual-budget-data-pv    # ← exact PV name
```

### 3d. Cloudflare Secret + Tunnel Deployment

The tunnel credentials JSON (from step 1b) goes into a Kubernetes Secret. We write the Secret manifest, paste in the JSON, then encrypt it with SOPS before committing:

```bash
sops --encrypt --encrypted-regex '^(data|stringData)$' --in-place cloudflare-secret.yaml
```

After encryption, the file in Git looks like:
```yaml
stringData:
  credentials.json: ENC[AES256_GCM,data:74O/7y4Oye...]
```

Safe to commit. Flux decrypts it at apply time using the cluster's Age private key.

The tunnel deployment (`cloudflare.yaml`) runs 2 cloudflared replicas (HA) and a ConfigMap routes traffic:
```yaml
ingress:
- hostname: budget.rahatahsan.com
  service: http://actual-budget.actual-budget-prod.svc.cluster.local:5006
```

### 3e. kustomization.yaml — Wires Everything Together

Lists all resources and applies patches:
```yaml
namespace: actual-budget-prod     # stamps all resources with this namespace
resources:
  - ../../base/actual-budget/     # pulls in the base
  - namespace.yaml
  - pv.yaml
  - actual-budget-secret.yaml
  - cloudflare-secret.yaml
  - cloudflare.yaml
patches:
  - path: patch-pvc.yaml          # binds PVC to PV
```

---

## 4. Wiring Into Flux

Add `./actual-budget` to `apps/production/kustomization.yaml`. Flux already watches this file — adding it here means Flux picks up actual-budget on the next reconcile. No other changes needed.

---

## 5. First Deployment — Two Bugs Back to Back

We pushed and forced a reconcile:
```bash
flux reconcile kustomization apps-production --with-source
kubectl get pods -n actual-budget-prod
```

Pod went into `Error` state immediately. Logs showed:

```
EACCES: permission denied, mkdir '/data/server-files'
```

### Bug 1: Wrong UID (1001 instead of 1000)

We initially set `runAsUser: 1001`. But the `actualbudget/actual-server` image's Node.js process runs as the built-in `node` user — **UID 1000, GID 1000**. Setting 1001 overrides that with a user that doesn't match the image's intended identity.

Fix: change to `runAsUser: 1000`. Simple one-line patch, one commit.

But the pod still crashed with the same error. Bug 2 was underneath Bug 1.

---

## 6. The Real Problem — iSCSI + Non-Root + Fresh Filesystem

This is the most important lesson from the whole session.

### What's happening inside Kubernetes when a pod mounts an iSCSI volume?

1. democratic-csi attaches the iSCSI LUN to the node as a block device (like `/dev/sdb`)
2. The node mounts the block device's filesystem at a path
3. The pod's container sees that path as its `/data` directory

### What does a freshly created iSCSI LUN look like?

When you create a new LUN on QNAP and it gets mounted for the first time, the OS formats an **ext4 filesystem** on it. That fresh ext4 filesystem has its root directory owned by:
```
drwxr-xr-x  root:root  (UID 0, GID 0, mode 755)
```

### What does Kubernetes `fsGroup` actually do?

`fsGroup: 1000` in the pod spec tells Kubernetes: *"change the group ownership of mounted volumes to GID 1000."*

But **it only changes the GROUP, not the USER**.

So after Kubernetes applies fsGroup, the `/data` directory becomes:
```
drwxr-xr-x  root:1000  (UID still 0, GID now 1000, mode still 755)
```

Now the container runs as UID 1000. It tries to `mkdir /data/server-files`. Who can write to `/data`?

- **Owner (root, UID 0):** `rwx` — can write. But we're not root.
- **Group (GID 1000):** `r-x` — can read and execute, **cannot write**.
- **Others:** `r-x` — can read and execute, **cannot write**.

UID 1000 is in group 1000, but the group permission is `r-x` not `rwx`. **Write denied.** That's the `EACCES`.

### Why didn't linkding and audiobookshelf have this problem?

Great question. The answer: those apps originally ran as **root** when first deployed, before they were hardened to non-root.

When root first touched those volumes, root created all the necessary directories — and root-owned directories are writable by root. Later, when we switched to non-root, the directories already existed with the right structure. The non-root user didn't need to create anything new.

Actual Budget started non-root from day one. It hit the raw `root:root` filesystem immediately on first run with no existing directory structure.

---

## 7. The Fix — initContainer

The right solution: run a temporary container as root *before* the main container starts, to fix the permissions on `/data`. This is called an **initContainer**.

```yaml
initContainers:
  - name: fix-data-permissions
    image: busybox:1.37.0
    command: ["sh", "-c", "chown 1000:1000 /data"]
    securityContext:
      runAsUser: 0              # root
      runAsNonRoot: false
      capabilities:
        drop: [ALL]
        add: [CHOWN]            # need this to actually chown
```

InitContainers run to completion before any main container starts. So the sequence becomes:
1. initContainer runs as root → `chown 1000:1000 /data` → exits
2. `/data` is now owned by `1000:1000` on the LUN's filesystem permanently
3. Main container starts as UID 1000 → can write to `/data` because it owns it

> **Why not `chown -R` (recursive)?** The `/data` volume also has a `lost+found` directory created by ext4. That directory is owned by root and protected — even root with CAP_CHOWN can't always chown it. We only need `/data` itself to be writable, not its contents. `chown 1000:1000 /data` (no `-R`) is sufficient.

### The PSA Problem

PSA `restricted` blocks any container from running as root. Our initContainer runs as root. Kubernetes rejects it.

**Solution:** Temporarily relax the namespace PSA to `baseline`.

```yaml
# namespace.yaml — temporary
pod-security.kubernetes.io/enforce: baseline
```

`baseline` still prevents privileged containers, host network access, etc. — it's still meaningful security. It just allows root containers.

### Why `baseline` doesn't need to stay

Once the initContainer runs, the chown is **written to the LUN's ext4 filesystem on disk**. It's permanent storage. The next time the pod starts — even on a different node, even after a cluster restart — `/data` is already owned by `1000:1000`. The initContainer is not needed again.

So the sequence was:
1. Change PSA to `baseline` → commit
2. Add initContainer → commit
3. Push, let Flux deploy → pod starts, initContainer chowns `/data`, main container runs successfully
4. Remove initContainer → commit
5. Restore PSA to `restricted` → commit
6. Push again → pod restarts, `/data` still owned by `1000:1000`, works fine under `restricted`

---

## 8. Documentation

Every non-obvious decision gets documented. The initContainer fix procedure was written into `docs/actual-budget/README.md` in a collapsible details block — so if the LUN is ever replaced, there's a clear step-by-step reference for what to do and why.

This is the difference between a homelab and infrastructure you can actually maintain: **write down the why, not just the what.**

---

## 9. Homepage Integration

Adding a service to the homepage is a ConfigMap change — no Kubernetes restart involved, just a Git commit:

```yaml
- Productivity:
    - Actual Budget:
        href: https://budget.rahatahsan.com
        description: Personal finance
        icon: mdi-cash-multiple
```

First attempt used `icon: actual-budget.png` — the icon pack didn't have it, so nothing showed. Switched to `mdi-cash-multiple` — Material Design Icons are built into Homepage and always available.

The homepage pod caches its config on startup, so a rollout restart was needed to pick up the ConfigMap change:
```bash
kubectl rollout restart deploy/homepage -n homepage
```

---

## Key Takeaways

| Lesson | What It Teaches |
|--------|----------------|
| `fsGroup` changes GROUP, not USER | Kubernetes volume ownership is not the same as Linux chown |
| Fresh iSCSI LUNs are `root:root` | Any new volume needs permission bootstrapping for non-root containers |
| PSA `restricted` is a real gate | You can't just ignore it — you have to work with or around it deliberately |
| initContainers run before main containers | Useful for setup tasks that need different permissions or tools |
| `Recreate` strategy is required for RWO | RollingUpdate + ReadWriteOnce = deadlock |
| Document the non-obvious | Future you will not remember why the initContainer dance happened |

---

## Commit Log (Chronological)

```
ab2eb9e  add actual-budget base deployment
dfada8d  add actual-budget base service
baa6e90  add actual-budget base PVC
e850a93  add actual-budget base kustomization
6f4ee80  add actual-budget production namespace
d9fde9d  add actual-budget production PV (iSCSI LUN 8)
bf5302f  add actual-budget PVC patch to bind production PV
7491e77  add actual-budget Cloudflare tunnel deployment and configmap
1d6e2e6  add actual-budget SOPS-encrypted Cloudflare tunnel credentials
2793fc2  add actual-budget SOPS-encrypted app secret
6843898  add actual-budget production kustomization
55c8c38  register actual-budget in production kustomization
fdc473c  fix actual-budget security context to use node UID/GID 1000
77f9b38  temporarily relax actual-budget namespace PSA to baseline for initContainer
9dd2540  add initContainer to chown /data to UID 1000 on first mount
fc98533  fix initContainer: add CAP_CHOWN and chown only /data root
9cbefbd  add actual-budget docs with volume permission fix procedure
97d182a  remove initContainer now that /data ownership is fixed on LUN
8fdcc4e  restore actual-budget namespace PSA to restricted
8050a7c  add actual-budget architecture diagram
c3d3d76  rewrite actual-budget docs to match audiobookshelf format
f43d40d  add actual-budget to homelab architecture diagram
c0eda25  add actual-budget to homelab README projects and incidents
764de6d  add Actual Budget to homepage dashboard
9da5681  fix actual-budget homepage icon to use mdi-cash-multiple
```
