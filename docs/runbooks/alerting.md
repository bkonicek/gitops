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

## Editing the AlertmanagerConfig itself

This is different from adding a `PrometheusRule` (below) — most people will
never need to touch `alertmanagerconfig-slack.yaml` itself. This section is
for when you do.

Setting `alertmanagerConfiguration.name` to point at our own `AlertmanagerConfig`
means we **fully replace** `kube-prometheus-stack`'s own default routing and
inhibition, not extend it. Out of the box (i.e. if you never touch
`alertmanager.config` at all), the chart ships with:

```
helm show values prometheus-community/kube-prometheus-stack | less
# search for "alertmanager:" -> "config:"
```

which is worth actually running before changing this file — it's the ground
truth for what "sane defaults" means here, and the reference this file's
`route`/`inhibitRules` are deliberately reproducing. The two load-bearing
things it revealed, that aren't obvious from first principles:

- **The chart's own default root receiver is `null`.** Out of the box, nothing
  gets a real notification at all — the only reason `Watchdog` gets an
  explicit null route in the default config is that it needs one *regardless*
  of what the root receiver is (it fires forever, every `repeat_interval`).
  Everything else silently falls through to the same null default. The moment
  *we* add a real catch-all receiver (`slack-alerts`), every previously
  invisible-by-default alert becomes visible unless explicitly excluded again
  — which is exactly what bit us with `InfoInhibitor`.
- **`inhibit_rules` has 4 entries by default**, not just the obvious
  critical-suppresses-warning one — two of them exist purely to keep the
  synthetic `Watchdog`/`InfoInhibitor` alerts (see below) from ever being
  useful *targets* of a real notification. Diff this file's `inhibitRules`
  against that reference whenever you touch either.

**How to find "plumbing" alerts that need a null route**, like `Watchdog` and
`InfoInhibitor`: query Prometheus's rules API and look for `type: alerting`
rules whose `severity` label isn't `critical`/`warning`/`info`:

```
kubectl port-forward -n monitoring svc/sandbox-oci-prometheus-ope-prometheus 9090:9090 &
curl -s localhost:9090/api/v1/rules | jq -r '
  .data.groups[].rules[]
  | select(.type == "alerting" and (.labels.severity as $s | ["critical","warning","info"] | index($s) | not))
  | "\(.name) | severity: \(.labels.severity)"'
```

(Filtering on `type == "alerting"` matters — recording rules also show up in
this API and also lack a `severity` label, but they never produce alerts at
all, so they're not relevant here.) As of writing this only turns up
`Watchdog` and `InfoInhibitor`, both from the bundled `general.rules` group —
if `kube-prometheus-stack` ever ships another one, or you enable additional
default rule groups, this is how to notice before it pages you unexpectedly
instead of after.

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
