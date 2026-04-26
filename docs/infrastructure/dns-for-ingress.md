# DNS for Homelab Ingress

## Purpose

Ingress hostnames must resolve to the Traefik `LoadBalancer` IP.

MetalLB assigns the Traefik service a LAN IP. DNS points application hostnames to that IP, and Traefik routes requests by hostname.

## Hostname Pattern

Application hostnames use:

```text
<service>.tagawa.ca
```

Examples:

```text
traefik.tagawa.ca
it-tools.tagawa.ca
grafana.tagawa.ca
prometheus.tagawa.ca
alertmanager.tagawa.ca
```

## DNS Records

After Traefik receives its external IP, DNS records point each hostname to that IP.

Example:

```text
traefik.tagawa.ca       -> 10.0.200.11
it-tools.tagawa.ca      -> 10.0.200.11
grafana.tagawa.ca       -> 10.0.200.11
prometheus.tagawa.ca    -> 10.0.200.11
alertmanager.tagawa.ca  -> 10.0.200.11
```

A wildcard record can be used for the homelab zone:

```text
*.tagawa.ca -> 10.0.200.11
```

## Validation

Check the Traefik external IP:

```bash
kubectl get svc -n traefik-system
```

Verify DNS resolution from a LAN client:

```bash
dig it-tools.tagawa.ca
```

Verify the hostname reaches Traefik:

```bash
curl -I https://it-tools.tagawa.ca
```
