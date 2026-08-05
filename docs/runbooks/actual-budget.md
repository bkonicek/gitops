# Actual Budget: Backup & Restore Runbook

Actual Budget is deployed via [`charts/actual-budget`](../../charts/actual-budget), a local
wrapper chart around the upstream `actualbudget` chart (see
[`environments/sandbox-oci/actual-budget`](../../environments/sandbox-oci/actual-budget) for the
ArgoCD config). This document covers how backups work, how to restore, and what to do if the
node holding its storage disappears.

## Storage

`/data` is a 4Gi PVC (`actualbudget-data`) provisioned by `local-path-provisioner` — the
cluster's default StorageClass. This means the data lives on a specific node's local disk, is
**not replicated**, and the PV's `nodeAffinity` is pinned to that node. See
[Recovering from a lost or replaced node](#recovering-from-a-lost-or-replaced-node) below for
what that implies.

## Backup

[`templates/cronjob-backup.yaml`](../../charts/actual-budget/templates/cronjob-backup.yaml) runs
every 3 days (`backup.schedule` in [`values.yaml`](../../charts/actual-budget/values.yaml)):

1. A `busybox` init container tars `/data`, but first checks that `/data/user-files` is
   non-empty. If it's empty (data volume looks uninitialized — e.g. right after a node
   replacement recreated the PV from scratch), it exits non-zero and the Job fails loudly
   instead of uploading a near-empty archive. (`account.sqlite` was deliberately **not** used
   for this check — `actual-server` creates it lazily on the first request that touches account
   state, so it exists within seconds of a fresh empty volume, before any real budget data does.)
2. An `oci-cli` container (`ghcr.io/oracle/oci-cli`) uploads the tarball to
   `benkonicek-backups-us-ashburn-1` under a **fixed object name**
   (`actual-budget/actualbudget-backup.tar.gz`), overwriting the previous backup each run.

Retention is handled entirely on the OCI side: the bucket has versioning enabled with a 30-day
lifecycle policy on old versions, so there's no pruning logic in the CronJob itself.

To check backup history:

```
kubectl get jobs -n actual -l job-name --sort-by=.metadata.creationTimestamp | grep actualbudget-backup
kubectl logs -n actual job/<job-name>
```

## Restore

[`templates/cronjob-restore.yaml`](../../charts/actual-budget/templates/cronjob-restore.yaml)
defines `actualbudget-restore`, a `CronJob` with `suspend: true` — it never runs on its own. It
downloads the current backup object and extracts it over `/data`, **wiping existing contents
first** (clean restore, not a merge).

To restore:

```
# 1. Stop the app so it isn't writing to /data during the restore
kubectl scale deployment actualbudget -n actual --replicas=0

# 2. Trigger the restore Job
kubectl create job --from=cronjob/actualbudget-restore -n actual manual-restore

# 3. Watch it finish
kubectl logs -n actual job/manual-restore -f

# 4. Bring the app back up
kubectl scale deployment actualbudget -n actual --replicas=1
```

## Recovering from a lost or replaced node

Because storage is node-pinned (see [Storage](#storage) above), if the node backing
`actualbudget-data` is deleted (node pool cycling, manual replacement, hardware failure), the
PV's `nodeAffinity` points at a node that no longer exists. The PVC stays `Bound` — Kubernetes
does not detect or heal this on its own — so the actualbudget pod sits `Pending` indefinitely
(`0/N nodes are available: node(s) didn't match Pod's node affinity`).

Recovery is a single manual step, since the default StorageClass uses `reclaimPolicy: Delete`:

```
kubectl delete pvc actualbudget-data -n actual
```

ArgoCD's `selfHeal: true` sync policy then recreates the PVC from the chart automatically, and a
fresh (empty) PV gets provisioned on whichever node the pod actually lands on. At that point,
follow the [Restore](#restore) steps above to repopulate it from the latest backup — today this
is a manual step you have to remember to do.

## Future work

### Auto-restore on pod startup

Planned improvement (not yet implemented): add an `initContainer` to the actualbudget
`Deployment` itself (the upstream chart already exposes `initContainers: []` as a values hook)
that runs the same "is `/data/user-files` empty?" check used by the backup guard, inverted — if
empty, it automatically downloads and extracts the latest backup *before* the app container
starts.

This would close the gap in the recovery flow above: after `kubectl delete pvc`, the pod would
restore itself on boot instead of starting with an empty database and waiting on a human to
notice and trigger the restore Job. The `actualbudget-restore` CronJob would remain useful as a
manual-override tool (e.g. restoring to recover from in-app data corruption, not just node loss),
but would no longer be the primary recovery path.

Deliberately **not** planned: an automated controller to detect the stuck-`Pending` state and
delete the stale PVC itself. Kubernetes has no built-in primitive for this, cluster-autoscaler
already defaults to not evicting pods with local-path PVCs (so this only happens on deliberate
events, not routine autoscaler churn), and a real watch-loop controller (plus RBAC) is a lot of
surface area for something this infrequent and already a single manual `kubectl delete pvc` away
from fixed.

### Alerting on pod pending/unavailable

Implemented: [`charts/actual-budget/templates/prometheusrule.yaml`](../../charts/actual-budget/templates/prometheusrule.yaml)
fires `ActualBudgetDeploymentUnavailable` when the actualbudget `Deployment` has had zero
available replicas for more than `alerting.unavailableFor` (default `5m`), so a lost/replaced node
surfaces as a Slack message in `#alerts` instead of relying on someone noticing the app is down.
See [`docs/runbooks/alerting.md`](alerting.md) for how the Slack routing and the PrometheusRule
convention work, and for the pattern to follow when adding alerts for other apps.
