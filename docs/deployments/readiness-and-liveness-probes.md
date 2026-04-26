# Readiness and Liveness Probes

Kubernetes health probes help the cluster understand whether an application is ready to receive traffic and whether it should be restarted.

## Readiness Probe

A readiness probe controls whether a pod is added to a Service.

If the readiness probe fails, Kubernetes keeps the pod running but removes it from Service endpoints. This prevents Traefik and other clients from sending traffic to a pod that is still starting up or temporarily unable to serve requests.

## Liveness Probe

A liveness probe controls whether Kubernetes restarts a container.

If the liveness probe keeps failing, Kubernetes assumes the application is stuck or unhealthy and restarts the container.

## Example

For IT-Tools, the app serves HTTP traffic on port `80`, which is named `http` in the deployment. A simple HTTP probe against `/` is enough for basic health checking.

```yaml
readinessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 5
  periodSeconds: 10
livenessProbe:
  httpGet:
    path: /
    port: http
  initialDelaySeconds: 30
  periodSeconds: 30
```

The readiness probe helps with clean rollouts because Kubernetes waits for the new pod to serve HTTP before routing traffic to it. The liveness probe helps recover from cases where the container is running but the application is no longer responding.
