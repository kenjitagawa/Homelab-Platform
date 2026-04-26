# Namespace Strategy

## Purpose

Namespaces are used to keep the homelab platform organized and easier to troubleshoot.

Each application or platform service gets its own namespace. The matching folder under `apps/` keeps that service's manifests, Helm values, and supporting configuration together.

This keeps failures easier to isolate. If something goes wrong with a service, the namespace and folder usually point to the same ownership boundary.

## Current Namespaces

| Namespace      | App Directory      | Purpose                              |
| -------------- | ------------------ | ------------------------------------ |
| `traefik-system` | `apps/traefik/`    | Ingress controller and dashboard     |
| `metallb-system` | `apps/metallb/`    | LoadBalancer IP allocation           |
| `monitoring`   | `apps/monitoring/` | Prometheus, Grafana, and Alertmanager |
| `it-tools`     | `apps/it-tools/`   | Self-hosted IT-Tools application     |
| `whoami`       | `apps/whoami/`     | Sample Traefik routing test app      |

## Naming Approach

Application namespaces use the app name:

```text
it-tools
whoami
```

Platform services use the conventional namespace expected by the tool or chart:

```text
traefik-system
metallb-system
monitoring
```

## Troubleshooting

The namespace layout makes common checks predictable:

```bash
kubectl get pods -n it-tools
kubectl get svc -n monitoring
kubectl get ingress -n monitoring
kubectl get pods -n traefik-system
kubectl get pods -n metallb-system
```

For labels that span the platform, resources use:

```text
app.kubernetes.io/part-of: homelab-platform
```

This allows platform-wide filtering:

```bash
kubectl get all -A -l app.kubernetes.io/part-of=homelab-platform
```

## Future Workloads

New apps should get their own directory and namespace:

```text
apps/my-app/
  namespace.yaml
  deployment.yaml
  service.yaml
  ingress.yaml
  kustomization.yaml
```

Shared platform services should keep a clear namespace tied to their operational role.
