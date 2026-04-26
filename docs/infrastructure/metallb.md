# MetalLB

## Purpose

MetalLB gives this bare-metal k3s cluster `LoadBalancer` support.

In a cloud Kubernetes cluster, a cloud provider usually assigns external IPs to `LoadBalancer` services. This homelab does not have that provider layer, so MetalLB assigns IPs from a reserved LAN range and advertises them on the local network.

MetalLB only handles LAN IP assignment and advertisement. Traefik still handles HTTP routing, HTTPS, certificates, and application ingress.

Related infrastructure docs:

- [Load Balancer Address Pool](load-balancer-address-pool.md)
- [Ingress Baseline](ingress-baseline.md)

## Cluster Assumptions

k3s ServiceLB is disabled because MetalLB owns `LoadBalancer` behavior in this cluster.

The bundled k3s Traefik deployment is also disabled because Traefik is managed through this repository with Helm values in `apps/traefik/values.yaml`.

The k3s server config uses:

```yaml
disable:
  - servicelb
  - traefik
```

After changing `/etc/rancher/k3s/config.yaml`, restart k3s:

```bash
sudo systemctl restart k3s
```

## Address Pool

The MetalLB pool is:

```text
10.0.200.10-10.0.200.100
```

This range is reserved for Kubernetes load balancer services and must stay outside DHCP and static host assignments.

The pool is defined in:

```text
apps/metallb/ip-address-pool.yaml
```

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

## L2 Advertisement

Layer 2 advertisement tells MetalLB to announce addresses from the pool on the LAN.

It is defined in:

```text
apps/metallb/l2-advertisement.yaml
```

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

## Repository Layout

```text
apps/metallb/
  namespace.yaml
  ip-address-pool.yaml
  l2-advertisement.yaml
  kustomization.yaml
```

## Apply

Install the upstream MetalLB components first:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/main/config/manifests/metallb-native.yaml
```

Apply the repository-managed MetalLB configuration:

```bash
kubectl apply -k apps/metallb
```

## Validation

Verify the MetalLB pods:

```bash
kubectl get pods -n metallb-system
```

Verify the custom resources:

```bash
kubectl get ipaddresspools,l2advertisements -n metallb-system
```

Verify Traefik received an address from the pool:

```bash
kubectl get svc -n traefik-system
```

Expected:

```text
NAME      TYPE           EXTERNAL-IP
traefik   LoadBalancer   10.0.200.x
```

## Smoke Test

Create a temporary test service:

```bash
kubectl create namespace lb-test
kubectl create deployment nginx --image=nginx -n lb-test
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

Test from a LAN client:

```bash
curl http://<ASSIGNED_LOADBALANCER_IP>
```

Clean up:

```bash
kubectl delete namespace lb-test
```

## Troubleshooting

If a service does not receive an external IP, check the MetalLB pods and custom resources:

```bash
kubectl get pods -n metallb-system
kubectl get ipaddresspools,l2advertisements -n metallb-system
```

If the IP is assigned but unreachable from the LAN, check for DHCP overlap, static IP conflicts, subnet mismatch, or network equipment blocking ARP traffic.
