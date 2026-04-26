# kube-prometheus-stack

## Overview

`kube-prometheus-stack` installs Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics, default dashboards, and default alerting rules.

This repository stores the namespace manifest and Helm values under:

```text
apps/monitoring/
  namespace.yaml
  helm-values.yaml
  kustomization.yaml
```

## Namespace

Create the monitoring namespace before installing the chart:

```bash
kubectl apply -k apps/monitoring
```

Verify it exists:

```bash
kubectl get namespace monitoring
```

## Install

Update Helm chart metadata:

```bash
helm repo update
```

Install or upgrade the stack:

```bash
helm upgrade --install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  --version 84.1.0 \
  -f apps/monitoring/helm-values.yaml
```

## Verify Workloads

Check that the monitoring pods are running:

```bash
kubectl get pods -n monitoring
```

Check the Helm release:

```bash
helm status kube-prometheus-stack -n monitoring
```

## Ingress Access

This setup exposes Grafana, Prometheus, and Alertmanager through Traefik for homelab debugging convenience.

```text
https://grafana.tagawa.ca
https://prometheus.tagawa.ca
https://alertmanager.tagawa.ca
```

Internal DNS should point each hostname to the Traefik LoadBalancer IP.

Prometheus and Alertmanager do not provide strong built-in authentication in this configuration. This is acceptable for a private homelab, but this access pattern should not be used for a production environment without adding proper authentication, authorization, and network controls.

## Verify Prometheus Targets

Open Prometheus:

```text
https://prometheus.tagawa.ca/targets
```

If ingress is not available yet, use [port-forwarding](../kubernetes-learnings/port-forwarding.md):

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Confirm baseline targets are up, including:

- Prometheus
- kube-state-metrics
- node-exporter
- kubelet
- Grafana

Any down targets should be reviewed before relying on the stack for cluster monitoring.

## Grafana Access

Retrieve the Grafana admin password from the chart-managed secret:

```bash
kubectl get secret kube-prometheus-stack-grafana \
  -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

The default username is:

```text
admin
```

## Alertmanager Access

Open Alertmanager:

```text
https://alertmanager.tagawa.ca
```

If ingress is not available yet, use port-forwarding:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
```
