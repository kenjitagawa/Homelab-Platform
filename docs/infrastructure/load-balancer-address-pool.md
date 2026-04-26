# Load Balancer Address Pool

## Purpose

MetalLB needs a dedicated LAN address pool for Kubernetes `LoadBalancer` services.

The pool must be:

- on the same LAN as the cluster nodes
- outside the DHCP range
- outside any static host assignments
- reserved for Kubernetes load balancer use

## Selected Pool

The configured pool is:

```text
10.0.200.10-10.0.200.100
```

It is defined in:

```text
apps/metallb/ip-address-pool.yaml
```

The resource name is:

```text
homelab-lan-pool
```

## Advertisement

Layer 2 advertisement is configured in:

```text
apps/metallb/l2-advertisement.yaml
```

The advertisement resource is:

```text
homelab-l2-advertisement
```

It advertises addresses from:

```text
homelab-lan-pool
```

## Validation

Verify the pool and advertisement exist:

```bash
kubectl get ipaddresspools,l2advertisements -n metallb-system
```

Verify a `LoadBalancer` service receives an address from the pool:

```bash
kubectl get svc -A
```

The Traefik service should receive an external IP from `10.0.200.10-10.0.200.100`.
