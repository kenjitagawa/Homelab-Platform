# Homelab Infrastructure

## Purpose

This document describes the homelab environment that backs the Kubernetes platform in this repository.

The cluster is a four-node k3s deployment running on the `10.0.1.0/24` LAN. Platform services, application manifests, ingress, TLS, and observability configuration are managed from this repository.

## Node Inventory

| Node       | IP Address | Role                      | Responsibility                         |
| ---------- | ---------- | ------------------------- | -------------------------------------- |
| homelab-01 | 10.0.1.101 | control-plane, etcd, master | Kubernetes API, scheduling, etcd       |
| homelab-02 | 10.0.1.102 | control-plane, etcd, master | Kubernetes API, scheduling, etcd       |
| homelab-03 | 10.0.1.103 | control-plane, etcd, master | Kubernetes API, scheduling, etcd       |
| homelab-04 | 10.0.1.104 | worker                    | Application and platform workloads     |

All nodes are running Ubuntu 24.04 LTS with k3s `v1.32.13+k3s1`.

## Cluster Networking

The Kubernetes API is reached through the cluster endpoint:

```text
10.0.1.210:6443
```

Cluster-internal services use the k3s service network. For example, the default Kubernetes service is `10.43.0.1`.

MetalLB provides LAN-facing `LoadBalancer` IPs from the reserved pool:

```text
10.0.200.10-10.0.200.100
```

Traefik currently receives:

```text
10.0.200.11
```

## DNS Approach

Homelab services use hostnames under:

```text
tagawa.ca
```

Ingress hostnames point to the Traefik `LoadBalancer` IP:

```text
10.0.200.11
```

Current ingress hostnames include:

```text
traefik.tagawa.ca
it-tools.tagawa.ca
grafana.tagawa.ca
prometheus.tagawa.ca
alertmanager.tagawa.ca
```

Traefik receives the request and routes it to the correct Kubernetes service based on the hostname.

## Ingress Approach

Traefik is the cluster ingress controller and is managed through:

```text
apps/traefik/
```

MetalLB exposes Traefik with a `LoadBalancer` service:

```text
traefik-system/traefik -> 10.0.200.11
```

The shared ingress path is:

```text
LAN client
  -> DNS hostname
  -> Traefik LoadBalancer IP
  -> Traefik
  -> Kubernetes Service
  -> Pod
```

TLS is handled by Traefik with Let's Encrypt certificates issued through the Cloudflare DNS challenge.

## Current Exposed Services

| Service      | Hostname                  | Namespace      | Backend Service                          |
| ------------ | ------------------------- | -------------- | ---------------------------------------- |
| Traefik      | traefik.tagawa.ca         | traefik-system | api@internal                             |
| IT-Tools     | it-tools.tagawa.ca        | it-tools       | it-tools                                 |
| Grafana      | grafana.tagawa.ca         | monitoring     | kube-prometheus-stack-grafana            |
| Prometheus   | prometheus.tagawa.ca      | monitoring     | kube-prometheus-stack-prometheus         |
| Alertmanager | alertmanager.tagawa.ca    | monitoring     | kube-prometheus-stack-alertmanager       |

## Related Docs

- [Repository Structure](structure.md)
- [Ingress Baseline](infrastructure/ingress-baseline.md)
- [MetalLB](infrastructure/metallb.md)
- [Traefik](infrastructure/traefik.md)
- [DNS for Homelab Ingress](infrastructure/dns-for-ingress.md)
- [TLS Certificate Strategy](infrastructure/tls-certificate-strategy.md)
