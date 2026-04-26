# Traefik

## Purpose

Traefik is the ingress controller for this homelab cluster.

It receives HTTP and HTTPS traffic on the MetalLB-assigned LAN IP, then routes requests to Kubernetes services based on hostname.

Traefik is managed in this repository with Helm values and a small set of Kubernetes manifests.

Related infrastructure docs:

- [Ingress Baseline](ingress-baseline.md)
- [DNS for Homelab Ingress](dns-for-ingress.md)
- [TLS Certificate Strategy](tls-certificate-strategy.md)
- [Traefik Dashboard](traefik-dashboard.md)

## Traffic Flow

```text
LAN client
  -> DNS hostname
  -> Traefik LoadBalancer IP
  -> Traefik
  -> Kubernetes Service
  -> Pod
```

Example:

```text
it-tools.tagawa.ca
  -> 10.0.200.11
  -> Traefik
  -> it-tools service
  -> it-tools pod
```

## Repository Layout

```text
apps/traefik/
  namespace.yaml
  values.yaml
  dashboard-ingressroute.yaml
  kustomization.yaml
```

## Helm Values

Traefik is installed with:

```text
apps/traefik/values.yaml
```

Important settings:

- `service.type: LoadBalancer` exposes Traefik through MetalLB.
- `ingressClass.enabled: true` creates the Traefik ingress class.
- `ingressClass.isDefaultClass: true` makes Traefik the default ingress controller.
- `providers.kubernetesIngress.enabled: true` enables standard Kubernetes `Ingress` resources.
- `providers.kubernetesCRD.enabled: true` enables Traefik CRDs such as `IngressRoute`.
- `ports.web` exposes HTTP on port `80`.
- `ports.websecure` exposes HTTPS on port `443`.
- HTTP traffic redirects to HTTPS.
- `certificatesResolvers.letsencrypt` uses Let's Encrypt with Cloudflare DNS challenge.
- `persistence` stores ACME certificate state at `/data/acme.json`.
- `metrics.prometheus.enabled: true` exposes Traefik metrics.
- `logs.access.enabled: true` enables access logs.

## TLS

Traefik uses the `letsencrypt` certificate resolver.

The Cloudflare API token is provided by this secret:

```text
cloudflare-secret
```

Expected key:

```text
api-token
```

Create it manually:

```bash
kubectl create secret generic cloudflare-secret \
  --namespace traefik-system \
  --from-literal=api-token='YOUR_CLOUDFLARE_API_TOKEN'
```

The real token is not committed to Git.

## Dashboard

The dashboard route is defined in:

```text
apps/traefik/dashboard-ingressroute.yaml
```

It exposes:

```text
https://traefik.tagawa.ca
```

The route targets Traefik's internal service:

```text
api@internal
```

The dashboard uses HTTPS through the `letsencrypt` resolver.

The insecure dashboard API is disabled:

```yaml
api:
  dashboard: true
  insecure: false
```

## Apply

Apply the namespace and dashboard route:

```bash
kubectl apply -k apps/traefik
```

Install or upgrade Traefik:

```bash
helm upgrade --install traefik traefik/traefik \
  --namespace traefik-system \
  --create-namespace \
  -f apps/traefik/values.yaml
```

## Validation

Verify the Traefik pod:

```bash
kubectl get pods -n traefik-system
```

Verify the `LoadBalancer` service:

```bash
kubectl get svc -n traefik-system
```

Expected:

```text
NAME      TYPE           EXTERNAL-IP    PORT(S)
traefik   LoadBalancer   10.0.200.x     80/TCP,443/TCP
```

Verify the certificate storage PVC:

```bash
kubectl get pvc -n traefik-system
```

Verify dashboard routing:

```bash
curl -I https://traefik.tagawa.ca
```

Verify an application route:

```bash
curl -I https://it-tools.tagawa.ca
```

## Troubleshooting Notes

The Traefik chart expects HTTP redirection and TLS settings under `http`.

Correct structure:

```yaml
ports:
  web:
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true

  websecure:
    http:
      tls:
        enabled: true
        certResolver: letsencrypt
```

Traefik persists ACME state in the `data` volume:

```yaml
persistence:
  enabled: true
  name: data
  path: /data
```

The init container prepares `/data/acme.json` with the permissions Traefik expects:

```yaml
deployment:
  initContainers:
    - name: volume-permissions
      image: busybox:latest
      command:
        - sh
        - -c
        - touch /data/acme.json && chmod 600 /data/acme.json
      volumeMounts:
        - name: data
          mountPath: /data
```

The homelab uses a MetalLB-backed `LoadBalancer` service instead of fixed NodePorts.
