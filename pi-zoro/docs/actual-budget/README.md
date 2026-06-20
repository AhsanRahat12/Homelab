# Actual Budget

Self-hosted personal finance app using envelope budgeting methodology.

- **Namespace**: `actual-budget-prod`
- **Image**: `actualbudget/actual-server`
- **Port**: 5006
- **URL**: https://budget.rahatahsan.com
- **Storage**: 2Gi iSCSI LUN on QNAP (LUN 8)
- **Auth**: Password set on first login via browser UI

## Storage

Uses a single RWO iSCSI LUN (`actual-budget-data-pv`) mounted at `/data`. The LUN holds the SQLite database and all budget files.

## First-time Setup (Volume Permissions)

When the iSCSI LUN is freshly formatted, the filesystem root is owned by `root:root` (mode 755). The app runs as UID 1000 and cannot write to it without a one-time permission fix.

**If you ever need to redo this** (e.g. after replacing the LUN), temporarily add the following initContainer to `apps/base/actual-budget/deployment.yaml`, set the namespace PSA to `baseline`, deploy once, then remove the initContainer and restore PSA to `restricted`:

```yaml
initContainers:
  - name: fix-data-permissions
    image: busybox:1.37.0
    command: ["sh", "-c", "chown 1000:1000 /data"]
    securityContext:
      runAsUser: 0
      runAsNonRoot: false
      allowPrivilegeEscalation: false
      capabilities:
        drop:
          - ALL
        add:
          - CHOWN
    volumeMounts:
      - mountPath: /data
        name: actual-budget-data
```

Also temporarily change `namespace.yaml`:
```yaml
pod-security.kubernetes.io/enforce: baseline
pod-security.kubernetes.io/warn: baseline
```

Once the pod starts successfully, revert both changes and redeploy.

## Cloudflare Tunnel

Tunnel name: `actual_budget`  
Tunnel ID: `2b57ff16-92b4-4d6a-919a-e674cd8f292f`  
Credentials secret: `actual-budget-tunnel-credentials`
