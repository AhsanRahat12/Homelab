# 📊 kube-prometheus-stack
Self-hosted observability stack deployed on Kubernetes via GitOps. Provides cluster-wide metrics, dashboards, and alerting — backed by durable iSCSI storage on QNAP. [prometheus-community/kube-prometheus-stack](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)

Migrated all three components off SD cards after 28 combined pod restarts wiped all metrics, dashboards, and alert state. Full alerting suite added on top — Prometheus rules covering storage, pods, nodes, and Flux, routed through Alertmanager to Slack.

**Grafana at** [grs.rahatahsan.com](https://grs.rahatahsan.com) — LAN only, not publicly accessible

> **TL;DR:** Migrated Prometheus, Grafana, and Alertmanager off SD-card storage after 28 combined restarts wiped all metrics and dashboards — moved to dedicated iSCSI LUNs, fixed root-owned volume permissions, and resolved Operator-vs-manual PVC ownership conflicts. Later added a full alerting suite — 11 PrometheusRule alerts for storage, pods, nodes, and Flux, routed to Slack via Alertmanager with an encrypted webhook secret. End-to-end pipeline confirmed: a manual test alert reached Slack within 30 seconds.

![Kubernetes](https://img.shields.io/badge/Kubernetes-K3s-326CE5) ![Flux](https://img.shields.io/badge/Flux-GitOps-5468FF) ![Prometheus](https://img.shields.io/badge/Prometheus-Metrics-E6522C) ![Grafana](https://img.shields.io/badge/Grafana-Dashboards-F46800) ![Alertmanager](https://img.shields.io/badge/Alertmanager-Slack%20Alerts-4A154B) ![SOPS](https://img.shields.io/badge/SOPS-Age%20Encrypted-FFB000) ![Renovate](https://img.shields.io/badge/Renovate-Automated%20Updates-1A1F6C)

---

## Architecture

<p align="center">
  <img src="architecture.svg" alt="kube-prometheus-stack production architecture diagram" width="680"/>
</p>

---

## Stack

| Concern | Solution |
|---------|----------|
| Storage | democratic-csi → iSCSI LUNs on QNAP — Prometheus 10Gi, Grafana 2Gi, Alertmanager 1Gi, 30-day retention |
| Dashboards | Auto-provisioned by chart sidecar — survive restarts and redeployments |
| External access | Grafana via Traefik + TLS at grs.rahatahsan.com — local network only |
| Alerting | PrometheusRule (`homelab-alerts`, 11 rules) → Alertmanager (1 replica) → Slack via encrypted webhook |

---

## 📁 Repo Structure

```
monitoring/
  controllers/
    base/kube-prometheus-stack/      ← HelmRelease, HelmRepository, namespace
    staging/kube-prometheus-stack/   ← kustomization overlay wiring
  configs/
    staging/kube-prometheus-stack/   ← PVs, Grafana PVC, TLS secret, alerting rules, Slack secret, kustomization
docs/
  monitoring/README.md               ← you are here
```

The HelmRelease in `controllers/base` defines the full chart configuration including all storage values. The PVs and Grafana PVC live in `configs/staging` — they are static resources that must exist and be `Bound` before the HelmRelease reconciles.

---

## 🧠 Key Engineering Decisions

**All three components were writing to SD cards with zero persistence.** The default Helm deployment uses emptyDir volumes — ephemeral storage backed by whatever the node has locally. Prometheus TSDB is one of the worst possible workloads for SD card longevity — constant writes, constant compaction, never idle. By the time the migration happened, both SD cards were at 67–69% capacity and the damage was already visible:

| Component | Restarts before migration | Impact |
|---|---|---|
| Prometheus | 8 | 8 complete wipes of all metrics history |
| Grafana | 12 | 12 complete wipes of dashboards and datasource config |
| Alertmanager | 8 | 8 wipes of silence rules and state |

Fixed by migrating all three components to dedicated iSCSI LUNs on QNAP — Prometheus/Alertmanager get PVs only (the Operator creates their PVCs from `volumeClaimTemplate`), Grafana gets a PV+PVC handed to Helm via `existingClaim`. Mixing those up causes the Operator's auto-generated PVCs to get stuck `Pending`.

**Fresh iSCSI LUNs are root-owned — Prometheus cannot write to them.** Block LUNs are formatted with root ownership. Prometheus runs as UID 1000 and gets `permission denied` on every write. Fixing this permanently needs two things: `fsGroup: 2000` so Kubernetes chowns the mount at attach time, *and* a one-time initContainer running as root to `chown -R` the already-formatted volume — `fsGroup` alone doesn't retroactively fix a filesystem that already exists.

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 1000
  runAsGroup: 2000
  fsGroup: 2000
initContainers:
  - name: fix-permissions
    image: busybox
    command: ["sh", "-c", "chown -R 1000:2000 /prometheus"]
    securityContext:
      runAsUser: 0
      runAsNonRoot: false   # required — overrides pod-level non-root policy
```

**Secret encrypted with the wrong Age key — Flux couldn't decrypt it.** The Slack webhook secret was first encrypted with the local workstation's Age key. Flux failed to reconcile with `no identity matched any of the recipients` — Flux decrypts using the private key in `flux-system/sops-age`, a *different* key from the one used for local SOPS operations. SOPS also refuses to re-encrypt a file that already has a `sops:` block, so the file had to be deleted and recreated from scratch against the cluster's actual key:

```bash
kubectl get secret sops-age -n flux-system \
  -o jsonpath='{.data.age\.agekey}' | base64 -d | grep -o 'public key: .*'
```

**Alertmanager stuck in `Init:0/1` after a config update — RWO volume multi-attach.** Alertmanager rescheduled from `luffy` to `zoro`, but its RWO PV was still attached to `luffy` — `FailedAttachVolume: Multi-Attach error`. Scaling to 0 and back didn't help (CSI detach is asynchronous), and force-deleting the pod after suspending Flux didn't help either — the `VolumeAttachment` object itself persisted on `luffy`. The fix was deleting that `VolumeAttachment` object directly; democratic-csi recreated it on the correct node and the pod came up within 2 minutes.

<details>
<summary><strong>Additional implementation notes</strong></summary>

**Retain policy requires manual claimRef cleanup.** When a PVC bound to a `ReclaimPolicy: Retain` PV is deleted, the PV enters `Released` state and Kubernetes won't rebind it automatically — even to a new PVC with the right spec:

```bash
kubectl patch pv <name> -p '{"spec":{"claimRef":null}}'
```

**Grafana SQLite database corrupted during PVC transition.** Grafana served a white page with `database disk image is malformed`. Fixed by scaling down, deleting the corrupted PVC, clearing the PV claimRef, letting Flux recreate it fresh, then scaling back up. No data was lost — 12 prior SD-card restarts had already wiped everything.

**Prometheus retention must be set explicitly.** Without a retention value, TSDB grows until the LUN is full. Set to 30 days in HelmRelease values — real usage is 1–3Gi, 10Gi gives 3–5x headroom.

**Alertmanager must be single replica on a 2-node cluster.** A second replica has no scheduling guarantee and sits `Pending` indefinitely. `replicas: 1` set explicitly.

**`config` and `secrets` placed at the wrong indentation in `release.yaml`.** `alertmanagerSpec` (infrastructure — replicas, storage, secrets to mount) and `config` (Alertmanager's routing config) are siblings under `alertmanager:`. The first attempt placed both as top-level siblings of `alertmanager:` itself — Helm silently ignored both, so Alertmanager started with no Slack routing and no webhook mounted.

**Alertmanager v1 API removed — manual test alert failed.** `/api/v1/alerts` returned `"deprecated... removed as of version 0.28.0"`. Use `/api/v2/alerts`:

```bash
curl -X POST http://localhost:9093/api/v2/alerts \
  -H 'Content-Type: application/json' \
  -d '[{"labels":{"alertname":"TestAlert","severity":"critical"},"annotations":{"summary":"test"}}]'
```

</details>

---

## 🔔 Alerting

**Prometheus evaluates rules, Alertmanager routes and sends them.** Prometheus runs PromQL expressions on a schedule and decides whether a condition is true — it does not notify anyone. Alertmanager receives fired alerts and handles grouping, routing, silencing, and delivery to Slack.

### Rules — `alerting-rules.yaml`

A single `PrometheusRule` (`homelab-alerts`, labelled `release: kube-prometheus-stack` so the Operator picks it up) defines four groups, 11 alerts total:

| Group | Alert | Trigger | Why |
|---|---|---|---|
| `storage.critical` | `PrometheusDataStaleness` | `time() - prometheus_tsdb_head_max_time_seconds > 300` for 5m | The exact failure mode from the May 2026 incident — Prometheus silently stops writing TSDB data (e.g. iSCSI goes read-only) |
| `storage.critical` | `PVCUsageCritical` | any PVC, cluster-wide, > 80% for 5m | A full disk breaks the owning app, in any namespace |
| `storage.critical` | `PVCUsageWarning` | any PVC, cluster-wide, > 60% for 10m | Early warning with runway to act |
| `pods.critical` | `PodCrashLooping` | cluster-wide, > 5 restarts/hour for 5m | Catches kube-system and every app namespace, not just `monitoring` |
| `pods.critical` | `PodOOMKilled` | `last_terminated_reason{reason="OOMKilled"}` `unless` currently running | The terminated-reason metric is sticky — the `unless` clause stops it firing forever once the pod has recovered |
| `pods.critical` | `PodNotReadyExtended` | `kube_pod_status_ready{condition="false"} == 1` for 15m | Catches a pod stuck Running-but-not-Ready with 0 restarts — `PodCrashLooping` needs restarts, and the built-in `KubePodNotReady` only fires on phase Pending/Unknown, not Running. Added 2026-07-05 after `cnpg-cluster-2` sat at 0/1 for 2 days undetected |
| `pods.critical` | `PrometheusTargetDown` | `up == 0` for 15m | 15m, not the usual 5m, to filter out restart/rolling-update flaps |
| `flux.warning` | `FluxReconciliationFailed` | `gotk_reconcile_condition{type="Ready",status="False"} == 1` for 10m | A failed Flux reconciliation means Git changes are silently not applied |
| `nodes.warning` | `NodeMemoryPressure` | available memory < 15% for 5m | Below this the OOM killer becomes likely |
| `nodes.warning` | `NodeCPUSustained` | CPU > 90% for 5m | Sustained load plus thermal throttling degrades everything on that node |
| `nodes.warning` | `NodeTemperatureHigh` | CPU temp > 75°C for 2m | Pi 5 throttles at ~80°C — leaves time to check cooling |
| `nodes.warning` | `NodeDiskPressure` | root filesystem > 80% for 5m | A full SD card means kubelet can't even write logs |

`PVCUsageCritical`, `PVCUsageWarning`, and `PodCrashLooping` are deliberately cluster-wide rather than scoped to `namespace="monitoring"` — a full disk or crash loop in any app's namespace is just as dangerous, and a namespace filter would silently miss every other app.

**`iSCSISessionLost` was removed 2026-07-05.** It was keyed off `node_iscsi_session_count`, a metric no exporter or textfile collector ever actually produced — the alert had been structurally incapable of firing since the day it was written. A 2026-07-02 iSCSI session drop on `luffy` corrupted linkding and the CNPG primary and went undetected for 2 days because of this and the `PodNotReadyExtended` gap above. Replaced with `PodNotReadyExtended`, which watches application health directly instead of a storage-layer signal this repo has no way to provision or track without adding an untracked node-level agent.

### Routing — Slack via Alertmanager

The Slack Incoming Webhook URL is stored as a SOPS+Age encrypted Kubernetes Secret (`alertmanager-slack-webhook`), mounted into the Alertmanager pod as a file and read via `slack_api_url_file` — the URL is never an environment variable and never appears in plaintext.

| Setting | Value | Why |
|---|---|---|
| `group_by` | `[alertname, namespace, severity]` | 10 pods crashing at once produces 1 Slack message, not 10 |
| `group_wait` | 30s | time for related alerts to arrive before the first notification |
| `repeat_interval` (critical) | 1h | stays visible while actively firing |
| `repeat_interval` (warning) | 4h | reduces noise for lower-priority alerts |
| `send_resolved` | true | sends a ✅ follow-up when an alert clears |

---

## Storage Sizing

| Component | Size | Rationale |
|---|---|---|
| Prometheus | 10Gi | 30d retention on a small cluster. Real usage ~1–3Gi. 10Gi gives 3–5x headroom for target growth. Do not go below 5Gi — compaction needs working space. |
| Grafana | 2Gi | SQLite DB + dashboards + plugins. Real usage under 200Mi. 2Gi is effectively infinite. |
| Alertmanager | 1Gi | Silence rules and state only. Real usage under 50Mi. 1Gi is the minimum sensible LUN size. |

---

## 🚀 What's Next

| Item | Status |
|------|--------|
| Grafana PostgreSQL migration | Replace SQLite with PostgreSQL to eliminate corruption risk on storage interruptions. SQLite on iSCSI is fragile — one dropped connection can corrupt the database. |

---

## 🔗 Related

- [Homelab Overview](https://github.com/AhsanRahat12/Homelab)
- [GitHub Profile](https://github.com/AhsanRahat12)
