# CNPG Stuck Primary — Root Cause and PodNotReadyExtended Alert

**Date:** 2026-07-04

---

## What Happened

On 2026-07-02, an iSCSI session drop on `luffy` left the CNPG primary (`cnpg-cluster-2`) pointing at storage that no longer existed. The pod never crashed and never restarted — it just sat at `0/1 Ready` for two full days, silently failing every query.

CNPG never failed over on its own. Nobody noticed until this session, two days later.

---

## Root Cause

CNPG's liveness probe (`/healthz`) only checks that the Postgres process is alive — not that it can actually read or write data. That kept passing the entire time (0 restarts). Readiness (`/readyz`) correctly failed the entire time, but nothing was watching readiness:

- `PodCrashLooping` never tripped — the pod wasn't restarting, it was just stuck.
- The built-in `KubePodNotReady` rule only fires on phase `Pending`/`Unknown` — this pod's phase stayed `Running` throughout.

So there was a real gap: "Running but readiness failing for an extended period" had no alert covering it at all.

Also discovered while investigating: the alert that was *supposed* to catch iSCSI problems, `iSCSISessionLost`, was keyed off `node_iscsi_session_count` — a metric no exporter or textfile collector anywhere ever actually produced. It had been sitting in the ruleset for weeks looking like coverage that didn't exist, and was structurally incapable of ever firing.

---

## The Fix

Removed the dead `iSCSISessionLost` alert. Replaced it with `PodNotReadyExtended`, which watches the actual symptom (pod readiness) instead of a storage-layer signal:

```yaml
- alert: PodNotReadyExtended
  expr: |
    max by (namespace, pod) (kube_pod_status_ready{condition="false"}) == 1
  for: 15m
  labels:
    severity: critical
  annotations:
    summary: "Pod {{ $labels.pod }} in {{ $labels.namespace }} not Ready for 15+ minutes"
```

15 minutes was chosen to filter out normal slow startups (linkding alone has taken up to ~40s to become ready).

File: `pi-zoro/monitoring/configs/staging/kube-prometheus-stack/alerting-rules.yaml`

---

## Verification

Manually deleted the stuck primary pod to test the actual failover path (not just the alert):

- CNPG promoted the replica and had the cluster fully healthy again in **under a minute**.
- Confirms the failover mechanism itself works fine — it just needs a clear signal to act on, which a "stuck, not crashed" pod never gave it.

Decision: for now this stays a **human-in-the-loop** step (alert fires → we decide → we act) rather than fully automated. Auto-deleting the pod the moment the alert fires is a reasonable next step, but risky if the alert ever fires on something merely slow rather than truly stuck — deferred rather than rushed.

Docs updated: `pi-zoro/docs/cnpg/README.md` and the incidents list in the main `README.md`.

---

## What We Did Not Account For

The `PodNotReadyExtended` expression matches on pod readiness alone, with no check on pod **phase**. That includes any pod whose `Ready` condition is `false` — which is also true, permanently, for any completed Job/CronJob pod (reason `PodCompleted`).

This wasn't caught at the time because nothing prompted checking it against Job pods specifically — the alert was written and verified against the CNPG incident shape (Running, not-Ready), not against the cluster's other workloads. The `renovate` CronJob runs `@hourly` and leaves completed pods around, each of which tripped this exact alert starting the next day. See the follow-up note: [2026-07-05-renovate-alert-misfire.md](./2026-07-05-renovate-alert-misfire.md).
