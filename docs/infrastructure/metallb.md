# MetalLB Setup Guide

## Overview

MetalLB provides `LoadBalancer` service support for bare-metal Kubernetes clusters.

In cloud environments, services of type `LoadBalancer` usually receive an external IP from the provider. In this homelab k3s cluster, MetalLB fills that role by assigning IPs from a reserved LAN range and advertising those IPs on the local network.

MetalLB only handles external IP assignment and LAN advertisement. It does not replace ingress routing, TLS termination, or application-level load balancing.

## Prerequisites

### Disable k3s ServiceLB

MetalLB replaces the default k3s ServiceLB implementation.

On each server node, update `/etc/rancher/k3s/config.yaml`:

```yaml
disable:
  - servicelb
```

If Traefik will also be managed through this repository, disable the bundled Traefik deployment at the same time:

```yaml
disable:
  - servicelb
  - traefik
```

Restart k3s after changing the node configuration:

```bash
sudo systemctl restart k3s
```

### Reserve a LAN IP Range

The MetalLB pool must be in the same subnet as the cluster nodes, outside the DHCP range, and not already assigned to another device.

This repository currently uses:

```text
10.0.200.10-10.0.200.100
```

## Installation

Install the upstream MetalLB components before applying the resources in this repository:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-native.yaml
```

Verify the controller and speaker pods are running:

```bash
kubectl get pods -n metallb-system
```

## Repository Layout

```text
apps/
  metallb/
    namespace.yaml
    ip-address-pool.yaml
    l2-advertisement.yaml
    kustomization.yaml
```

## Configuration

`apps/metallb/namespace.yaml` creates the `metallb-system` namespace used by the MetalLB resources.

`apps/metallb/ip-address-pool.yaml` defines the LAN address pool MetalLB can assign to `LoadBalancer` services:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: homelab-lan-pool
  namespace: metallb-system
spec:
  addresses:
    - 10.0.200.10-10.0.200.100
```

`apps/metallb/l2-advertisement.yaml` enables Layer 2 advertisement for that pool:

```yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: homelab-l2-advertisement
  namespace: metallb-system
spec:
  ipAddressPools:
    - homelab-lan-pool
```

`apps/metallb/kustomization.yaml` groups the resources so they can be applied together:

```bash
kubectl apply -k apps/metallb
```

## Apply

Apply the repository-managed MetalLB configuration:

```bash
kubectl apply -k apps/metallb
```

Verify the custom resources:

```bash
kubectl get ipaddresspools -n metallb-system
kubectl get l2advertisements -n metallb-system
```

## Smoke Test

Create a temporary namespace and deploy nginx:

```bash
kubectl create namespace lb-test
kubectl create deployment nginx --image=nginx -n lb-test
kubectl scale deployment nginx --replicas=2 -n lb-test
```

Expose the deployment as a `LoadBalancer` service:

```bash
kubectl expose deployment nginx \
  --name nginx \
  --port 80 \
  --target-port 80 \
  --type LoadBalancer \
  -n lb-test
```

Confirm MetalLB assigns an address from the configured pool:

```bash
kubectl get svc -n lb-test
```

Example output:

```text
NAME    TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)
nginx   LoadBalancer   10.x.x.x       10.0.200.10   80:xxxxx/TCP
```

Test from another machine on the LAN:

```bash
curl http://10.0.200.10
```

Clean up the smoke test when finished:

```bash
kubectl delete namespace lb-test
```

## Troubleshooting

If no external IP is assigned, check that MetalLB is running and that the service is actually `type: LoadBalancer`:

```bash
kubectl get pods -n metallb-system
kubectl get svc -n lb-test
```

If the assigned IP is not reachable from the LAN, check for DHCP overlap, an address already in use, the wrong subnet, or network equipment blocking ARP traffic.

If the service stays pending, confirm the MetalLB CRDs are installed and k3s ServiceLB is disabled.
