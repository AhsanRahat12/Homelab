# PodNotReadyExtended Paging Every ~15 Minutes — Renovate CronJob Pods

**Date:** 2026-07-05

---

## What Happened

The `PodNotReadyExtended` alert (added the previous day, see [2026-07-04-cnpg-stuck-primary-alerting.md](./2026-07-04-cnpg-stuck-primary-alerting.md)) started firing to Slack repeatedly, roughly every 15 minutes, since the day it was set up.

---

## Root Cause

The alert expression:

```
max by (namespace, pod) (kube_pod_status_ready{condition="false"}) == 1
for: 15m
```

matches **any** pod with `Ready=false` sustained for 15+ minutes — with no exclusion for pods that finished intentionally. The `renovate` CronJob runs `@hourly`; once a job pod completes, Kubernetes marks it `Ready=false, reason: PodCompleted` and leaves a few of them around (job history limit). That condition never clears — a completed pod is never going to become Ready again — so exactly 15 minutes after each renovate run finished, the alert fired.

Confirmed via Alertmanager's API (`/api/v2/alerts`) that three separate `PodNotReadyExtended` alerts were active simultaneously, all against old `renovate-*` pods, each `startsAt` timestamped 15 minutes after its pod's completion time. Cluster and CNPG postgres were otherwise fully healthy — no crash-looping pods, CNPG `2/2 ready`.

---

## The Fix

Gated the expression on pod phase so only pods that are actually `Running` (and stuck not-ready) can trigger it:

```yaml
- alert: PodNotReadyExtended
  expr: |
    (max by (namespace, pod) (kube_pod_status_ready{condition="false"}) == 1)
    and on (namespace, pod) (kube_pod_status_phase{phase="Running"} == 1)
  for: 15m
```

File: `pi-zoro/monitoring/configs/staging/kube-prometheus-stack/alerting-rules.yaml`

---

## Verification

Ran both the old and new PromQL directly against the live Prometheus pod (`prometheus-kube-prometheus-stack-prometheus-0`):

- Old expression matched the three completed `renovate-*` pods.
- New expression (with the `phase="Running"` gate) returned an **empty result set** — correctly excludes them, and correctly finds nothing else wrong right now, matching what `kubectl get pods -A` showed independently.

---

## Lesson

Any alert built on a "sticky" per-pod condition (readiness, last-terminated-reason, etc.) needs to explicitly account for pods that are supposed to end up in that state permanently — completed Jobs/CronJobs being the most common case in this cluster. The existing `PodOOMKilled` alert already handles a similar sticky-metric problem with an `unless (... running == 1)` clause; this is the same class of bug, just missed on `PodNotReadyExtended` because it was written and tested against the CNPG incident only, not against the cluster's Job-based workloads.
