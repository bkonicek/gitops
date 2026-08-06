# Alerting conventions

## Slack routing

All alerts route to `#alerts` via one global `AlertmanagerConfig`:
[`charts/prometheus-operator/templates/alertmanagerconfig-slack.yaml`](../../charts/prometheus-operator/templates/alertmanagerconfig-slack.yaml),
set as the top-level config via `alertmanagerConfiguration.name` in
[`values.yaml`](../../charts/prometheus-operator/values.yaml). Webhook URL comes
from 1Password via the `alertmanager-secrets` `ExternalSecret` — never in git.

Channel/receiver/secret name are chart values (`alerting.slack.*`). Nothing to
configure per-app — any `PrometheusRule` that fires routes to Slack automatically.

## Editing the AlertmanagerConfig itself

`alertmanagerConfiguration.name` fully replaces kube-prometheus-stack's default
routing/inhibition. Before changing this file, diff against the actual default:

```
helm show values prometheus-community/kube-prometheus-stack | less   # search "config:"
```

To find plumbing alerts (like `Watchdog`/`InfoInhibitor`) that need a null route
— `type: alerting` rules with a non-standard severity:

```
kubectl port-forward -n monitoring svc/sandbox-oci-prometheus-ope-prometheus 9090:9090 &
curl -s localhost:9090/api/v1/rules | jq -r '
  .data.groups[].rules[]
  | select(.type == "alerting" and (.labels.severity as $s | ["critical","warning","info"] | index($s) | not))
  | "\(.name) | severity: \(.labels.severity)"'
```

## Adding a PrometheusRule for an app

**Where:** a template in that app's own wrapper chart —
`charts/<app>/templates/prometheusrule.yaml`. See
[`charts/actual-budget/templates/prometheusrule.yaml`](../../charts/actual-budget/templates/prometheusrule.yaml).

**Why not a loose file under `environments/<cluster>/<app>/`:** ArgoCD's
`ops-tools` ApplicationSet only syncs the rendered Helm chart output, not
arbitrary files in the environment dir — they're never applied.

**No wrapper chart yet?** Create one first (`charts/<app>/`, matching
`charts/actual-budget`) — there's no other way to attach extra resources to an
app deployed straight from an upstream chart.

**No labels required** — `ruleSelectorNilUsesHelmValues: false` means Prometheus
picks up every `PrometheusRule` in the cluster.

**Check the defaults first** — `kube-prometheus-stack` ships many default rules
(`KubePodCrashLooping`, `KubeDeploymentReplicasMismatch`, etc.). Only add a
custom rule for something app-specific those don't cover.

### Standard rule shape

```yaml
{{- if .Values.alerting.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: <app>
spec:
  groups:
    - name: <app>
      rules:
        - alert: <CamelCaseAlertName>
          expr: |
            <promql expression>
          for: {{ .Values.alerting.someDuration }}
          labels:
            severity: {{ .Values.alerting.severity | quote }}  # critical | warning | info
          annotations:
            summary: "<one line>"
            description: "<what's happening and why it matters>"
            runbook_url: {{ .Values.alerting.runbookUrl | quote }}
{{- end }}
```

```yaml
alerting:
  enabled: true
  severity: critical
  someDuration: 5m
  runbookUrl: https://github.com/bkonicek/gitops/blob/main/docs/runbooks/<app>.md
```

- `runbook_url`: link to `docs/runbooks/<app>.md`, anchored to the relevant
  section if possible. Drop the annotation if no runbook exists yet.
- `alerting.enabled` toggle keeps it easy to disable per-environment.
