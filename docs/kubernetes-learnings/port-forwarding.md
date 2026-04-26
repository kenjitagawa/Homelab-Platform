# Port Forwarding

## Overview

`kubectl port-forward` creates a temporary connection from a local port on your machine to a Kubernetes resource inside the cluster.

It is useful for debugging because it bypasses DNS, Traefik, MetalLB, ingress, and TLS.

## Example

Port-forward the Prometheus service:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

This means:

```text
localhost:9090 on your machine
  -> Kubernetes API tunnel
  -> kube-prometheus-stack-prometheus service
  -> Prometheus pod
```

Then open:

```text
http://localhost:9090
```

## Why It Helps

If this does not work:

```text
https://prometheus.tagawa.ca
```

but port-forwarding does work, Prometheus is responding and the issue is likely in the external routing path:

- DNS
- Traefik
- ingress configuration
- certificate or TLS handling
- service routing

If port-forwarding does not work, the issue is likely closer to the application itself:

- pod is not running
- service is missing
- service selector is wrong
- application is not listening correctly

## Monitoring Examples

Prometheus:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-prometheus 9090:9090
```

Alertmanager:

```bash
kubectl port-forward -n monitoring svc/kube-prometheus-stack-alertmanager 9093:9093
```

Access them locally:

```text
http://localhost:9090
http://localhost:9093
```
