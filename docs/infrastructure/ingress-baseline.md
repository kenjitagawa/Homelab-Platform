# Ingress Baseline

## Purpose

The ingress baseline restores north-south traffic into the rebuilt k3s cluster.

It is built from:

- MetalLB for `LoadBalancer` IP assignment
- Traefik for HTTP and HTTPS ingress routing
- internal DNS records for homelab hostnames
- Let's Encrypt certificates through Traefik and Cloudflare DNS challenge

## Traffic Flow

```text
LAN client
  -> service hostname
  -> Traefik LoadBalancer IP from MetalLB
  -> Traefik
  -> Kubernetes Service
  -> application Pod
```

Example:

```text
it-tools.tagawa.ca
  -> 10.0.200.11
  -> Traefik
  -> it-tools service
  -> it-tools pod
```

## Repository Resources

MetalLB configuration:

```text
apps/metallb/
  namespace.yaml
  ip-address-pool.yaml
  l2-advertisement.yaml
  kustomization.yaml
```

Traefik configuration:

```text
apps/traefik/
  namespace.yaml
  values.yaml
  dashboard-ingressroute.yaml
  kustomization.yaml
```

## Validation

Verify MetalLB resources:

```bash
kubectl get ipaddresspools,l2advertisements -n metallb-system
```

Verify Traefik receives an external IP:

```bash
kubectl get svc -n traefik-system
```

Verify routed HTTPS access with an exposed application:

```bash
curl -I https://it-tools.tagawa.ca
```

Expected result:

```text
HTTP/2 200
```
