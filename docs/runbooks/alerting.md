# Alerting conventions

How Alertmanager gets alerts to Slack, and the convention for adding a new
`PrometheusRule` for an app without re-deriving the pattern each time.

## Slack routing

Every alert in the cluster routes to the `#alerts` Slack channel via a single
global `AlertmanagerConfig` resource:
[`charts/prometheus-operator/templates/alertmanagerconfig-slack.yaml`](../../charts/prometheus-operator/templates/alertmanagerconfig-slack.yaml).
It's referenced as the top-level config via
`kube-prometheus-stack.alertmanager.alertmanagerSpec.alertmanagerConfiguration.name`
in [`charts/prometheus-operator/values.yaml`](../../charts/prometheus-operator/values.yaml),
which is what lets it define the whole route/receiver tree instead of just
contributing a namespaced sub-route. The Slack webhook URL itself comes from
1Password via the `alertmanager-secrets` `ExternalSecret`
([`charts/prometheus-operator/templates/externalsecret-alertmanager.yaml`](../../charts/prometheus-operator/templates/externalsecret-alertmanager.yaml)) —
it's referenced by name/key in the `AlertmanagerConfig`, never written into git.

Channel, receiver name, and secret name/key are all chart-level values
(`alerting.slack.*` in `charts/prometheus-operator/values.yaml`) so a
different environment could override just the channel without redefining the
whole routing tree — Helm doesn't text-template `values.yaml`, so keeping the
structure in a `templates/` file (parameterized by `.Values`) rather than
inline in a values block is what makes that possible.

There's nothing to configure per-app to get an alert routed to Slack — any
`PrometheusRule` that fires, fires into this one route automatically.

## Adding a PrometheusRule for an app

**Where it goes:** as a template in that app's own chart, at
`charts/<app>/templates/prometheusrule.yaml` — see
[`charts/actual-budget/templates/prometheusrule.yaml`](../../charts/actual-budget/templates/prometheusrule.yaml)
for a working example.

**Why not a loose file under `environments/<cluster>/<app>/`:** ArgoCD's
`ops-tools` ApplicationSet ([`applications/appset-ops-tools.yaml`](../../applications/appset-ops-tools.yaml))
only ever syncs the *rendered Helm output* of `charts/<app>`, with values
layered from `base/<app>.yaml` and `environments/<cluster>/<app>/values.yaml`.
It does not sync arbitrary files sitting in the environment directory — a
loose `prometheusrule.yaml` dropped there would never actually be applied.

**If the app doesn't have a wrapper chart yet** (i.e. its `config.yaml` points
straight at an upstream chart via `repoURL`/`chartName`, no `charts/<app>/`
directory exists): create a thin wrapper chart first, matching
`charts/actual-budget` or `charts/prometheus-operator` — a `Chart.yaml`
declaring the upstream chart as a `dependencies` entry, plus a `templates/`
directory for the extra resource. There's no other way to attach a
`PrometheusRule` (or any other extra resource) to an app deployed this way.

**No labels required:** `charts/prometheus-operator/values.yaml` sets
`prometheus.prometheusSpec.ruleSelectorNilUsesHelmValues: false`, so Prometheus
picks up every `PrometheusRule` in the cluster — no `release: <helm-release>`
label needed on the rule, consistent with how ServiceMonitors/PodMonitors
already work here.

**Check the defaults first:** `kube-prometheus-stack` ships a large set of
default rules (`KubePodCrashLooping`, `KubeDeploymentReplicasMismatch`,
`KubePodNotReady`, etc.), enabled out of the box, no config needed. Only add a
custom `PrometheusRule` for something app-specific those don't already cover
— e.g. a business-logic condition, not generic pod health.

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

with matching values in `charts/<app>/values.yaml`:

```yaml
alerting:
  enabled: true
  severity: critical
  someDuration: 5m
  runbookUrl: https://github.com/bkonicek/gitops/blob/main/docs/runbooks/<app>.md
```

- `severity`: `critical` / `warning` / `info` — drives the `inhibitRules` in
  the global `AlertmanagerConfig` (a firing `critical` alert suppresses a
  `warning` alert for the same `alertname`+`namespace`), and shows up in the
  Slack message.
- `runbook_url`: link to `docs/runbooks/<app>.md` when one exists, ideally
  anchored to the specific section describing what to do — gives every Slack
  alert an actionable next step instead of just a name. If there's no runbook
  yet for the app, write one, or drop the annotation rather than link to
  nothing.
- Gating the whole rule behind an `alerting.enabled` value keeps it easy to
  temporarily disable per-environment without deleting the template.
