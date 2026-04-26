# Alerting

## Purpose

This document records the first custom alert rule for the homelab monitoring stack.

The default kube-prometheus-stack chart already installs many alert rules. Custom homelab-owned rules live under:

```text
apps/monitoring/alerts/
```

## First Custom Alert

The first custom alert is:

```text
HomelabNodeDown
```

It is defined in:

```text
apps/monitoring/alerts/node-down.yaml
```

The alert fires when a node-exporter target is down for more than 2 minutes:

```promql
up{job="node-exporter"} == 0
```

The rule has this label so the Prometheus instance managed by kube-prometheus-stack selects it:

```yaml
release: kube-prometheus-stack
```

## Slack Notifications

Alertmanager sends homelab alerts to Slack through this config:

```text
apps/monitoring/alerts/slack-alerts.yaml
```

The Slack webhook URL is not stored in Git. Create it as a Kubernetes Secret before applying the alerting manifests:

```bash
kubectl create secret generic alertmanager-slack-webhook \
  --namespace monitoring \
  --from-literal=webhook-url='https://hooks.slack.com/services/REPLACE/ME'
```

The `AlertmanagerConfig` routes alerts with `severity=warning` or `severity=critical` to the `#homelab-alerts` Slack channel. It also sends resolved notifications so Slack shows when an alert recovers.

Alerts created by kube-prometheus-stack use the main Alertmanager route in:

```text
apps/monitoring/helm-values.yaml
```

Those alerts are sent to the `#kube-alerts` Slack channel through this Secret:

```text
kube-alertmanager-slack-webhook
```

That Secret is mounted into the Alertmanager pod and read with `api_url_file`, so the kube-prometheus-stack Slack webhook also stays out of Git.

## Helm Values

The `alertmanager.config` block in `apps/monitoring/helm-values.yaml` replaces the default chart route that sent everything to the `null` receiver.

The default kube-prometheus-stack inhibition rules are still kept. Those rules reduce noise by suppressing lower-severity alerts when a higher-severity alert with the same `namespace` and `alertname` is already firing.

The main route now uses:

```yaml
receiver: slack-kube-alerts
```

That means kube-prometheus-stack alerts go to the `#kube-alerts` Slack channel by default. The `Watchdog` alert still goes to the `null` receiver because it is intentionally always firing.

The Slack webhook is configured with:

```yaml
api_url_file: /etc/alertmanager/secrets/kube-alertmanager-slack-webhook/webhook-url
```

That file path comes from this value:

```yaml
alertmanagerSpec:
  secrets:
    - kube-alertmanager-slack-webhook
```

The Prometheus Operator mounts that Secret into the Alertmanager pod. This keeps the Slack webhook URL out of Git while still allowing the chart-managed Alertmanager config to send Slack notifications.

## Apply

Install kube-prometheus-stack first so the `PrometheusRule` and `AlertmanagerConfig` CRDs exist.

Create the Secret for kube-prometheus-stack alerts if it does not already exist:

```bash
kubectl create secret generic kube-alertmanager-slack-webhook \
  --namespace monitoring \
  --from-literal=webhook-url='https://hooks.slack.com/services/REPLACE/ME'
```

Create the Secret for homelab-owned alerts:

```bash
kubectl create secret generic alertmanager-slack-webhook \
  --namespace monitoring \
  --from-literal=webhook-url='https://hooks.slack.com/services/REPLACE/ME'
```

Upgrade kube-prometheus-stack so Alertmanager gets the `#kube-alerts` receiver and mounts the `kube-alertmanager-slack-webhook` Secret:

```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --values apps/monitoring/helm-values.yaml
```

Apply the custom alerting manifests:

```bash
kubectl apply -k apps/monitoring/alerts
```

## Verify

Confirm the alert rule and Slack route exist:

```bash
kubectl get prometheusrule homelab-node-alerts -n monitoring
kubectl get alertmanagerconfig homelab-slack-alerts -n monitoring
```

Open Prometheus rules:

```text
https://prometheus.tagawa.ca/rules
```

Search for:

```text
HomelabNodeDown
```

Open Alertmanager and confirm the Slack route is loaded:

```text
https://alertmanager.tagawa.ca
```

The main route should send kube-prometheus-stack alerts to `#kube-alerts`. The homelab route should send alerts with `route=homelab` to `#homelab-alerts`.

## Test Result

The alert expression can be tested in Prometheus:

```promql
up{job="node-exporter"} == 0
```

Current expected result:

```text
no data
```

`no data` means all node-exporter targets are currently up, so the alert is loaded but not firing.

To test a firing condition later, stop or block node-exporter on one node, wait at least 2 minutes, and confirm `HomelabNodeDown` moves into the firing state. Alertmanager should then send the firing notification to Slack.

When node-exporter is healthy again, Alertmanager should send a resolved notification to Slack.

The first Slack firing and resolved notifications are recorded here:

```text
images/first-slack-alert.png
images/resolved-alert.png
```

If Slack does not receive the alert, check whether Alertmanager accepted the generated configuration:

```bash
kubectl logs -n monitoring statefulset/alertmanager-kube-prometheus-stack-alertmanager
```
