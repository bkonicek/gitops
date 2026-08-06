# Actual Budget: Backup & Restore Runbook

Deployed via [`charts/actual-budget`](../../charts/actual-budget), a wrapper chart around
upstream `actualbudget`.

## Storage

`/data` is a 4Gi PVC (`actualbudget-data`) on `local-path-provisioner` — node-pinned, not
replicated. See [Recovering from a lost or replaced node](#recovering-from-a-lost-or-replaced-node).

## Backup

[`cronjob-backup.yaml`](../../charts/actual-budget/templates/cronjob-backup.yaml) runs every 3
days (`backup.schedule`):

1. Checks `/data/user-files` is non-empty (guards against backing up an uninitialized volume,
   e.g. right after a node loss); fails loudly if empty.
2. Uploads a tarball to `benkonicek-backups-us-ashburn-1` at a fixed object name, overwriting
   each run. Retention is via OCI bucket versioning (30-day lifecycle), not the CronJob.

```
kubectl get jobs -n actual -l job-name --sort-by=.metadata.creationTimestamp | grep actualbudget-backup
kubectl logs -n actual job/<job-name>
```

## Restore

[`cronjob-restore.yaml`](../../charts/actual-budget/templates/cronjob-restore.yaml) (`suspend:
true`, manual only) downloads the backup and extracts over `/data`, wiping existing contents first.

```
kubectl scale deployment actualbudget -n actual --replicas=0
kubectl create job --from=cronjob/actualbudget-restore -n actual manual-restore
kubectl logs -n actual job/manual-restore -f
kubectl scale deployment actualbudget -n actual --replicas=1
```

## Recovering from a lost or replaced node

If the node backing `actualbudget-data` is gone, the PVC stays `Bound` but its `nodeAffinity`
points nowhere — the pod sits `Pending` forever. `ActualBudgetDeploymentUnavailable`
([`prometheusrule.yaml`](../../charts/actual-budget/templates/prometheusrule.yaml)) alerts to
`#alerts` after 5m — see [alerting.md](alerting.md).

```
kubectl delete pvc actualbudget-data -n actual
```

ArgoCD recreates the PVC/PV. From there the pod auto-restores itself: two `initContainers`
([`values.yaml`](../../charts/actual-budget/values.yaml)) check if `/data/user-files` is empty
and, if so, download + extract the latest backup before the app starts (no-op on normal
restarts). If no backup exists yet, they leave the volume empty rather than failing startup. The
`actualbudget-restore` CronJob remains available as a manual override.

Caveat found during testing: if a client connects while the volume is still empty (before a
restore completes), Actual Budget auto-creates a default "My Finances" budget cached locally in
that browser. A later restore doesn't clean that up, so it can appear alongside the real budget
in the Files list — cosmetic, removable via the file's `⋮` menu, and avoided going forward since
auto-restore means the app never runs against an empty volume a client could reach first.
