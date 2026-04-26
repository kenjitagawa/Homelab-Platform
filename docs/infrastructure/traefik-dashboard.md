# Traefik Dashboard

## Purpose

The Traefik dashboard provides visibility into routers, services, middleware, TLS configuration, and ingress health.

In this homelab, the dashboard is exposed through Traefik itself over HTTPS.

## Repository Configuration

The dashboard route is defined in:

```text
apps/traefik/dashboard-ingressroute.yaml
```

The hostname is:

```text
traefik.tagawa.ca
```

The route targets Traefik's internal API service:

```text
api@internal
```

TLS is issued through the configured Let's Encrypt resolver:

```yaml
tls:
  certResolver: letsencrypt
```

## Security Notes

The Traefik Helm values disable insecure dashboard exposure:

```yaml
api:
  dashboard: true
  insecure: false
```

This avoids exposing the dashboard through Traefik's insecure API port.

For a production environment, the dashboard should also be protected with authentication middleware or a stronger access control layer. In this homelab, access is intended to stay limited to the private network.

## Validation

Verify the IngressRoute exists:

```bash
kubectl get ingressroute -n traefik-system
```

Verify HTTPS access:

```bash
curl -I https://traefik.tagawa.ca
```

Open the dashboard in a browser:

```text
https://traefik.tagawa.ca
```
