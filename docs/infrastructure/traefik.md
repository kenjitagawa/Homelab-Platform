# Traefik Setup Guide

## Overview

Traefik is used as the Kubernetes ingress controller and reverse proxy for the homelab platform.

Its responsibilities are:

- receive HTTP and HTTPS traffic
- route requests to Kubernetes services
- manage TLS certificates
- provide a dashboard for ingress visibility
- expose metrics for monitoring

In this setup, Traefik is exposed through a Kubernetes `LoadBalancer` service. MetalLB assigns the Traefik service a LAN IP.

## Architecture

```text
Client
  -> DNS name
  -> Traefik LoadBalancer IP from MetalLB
  -> Traefik
  -> Kubernetes Service
  -> Pod
```

Example:

```text
whoami.tagawa.ca
  -> 10.0.200.11
  -> Traefik
  -> whoami service
  -> whoami pod
```

## Important DNS Note

After Traefik receives a `LoadBalancer` IP from MetalLB, update the internal Bind9 DNS instance to point application hostnames to that IP.

Example:

```text
traefik.tagawa.ca  -> 10.0.200.11
whoami.tagawa.ca   -> 10.0.200.11
```

All ingress hostnames can point to the same Traefik LoadBalancer IP. Traefik then routes traffic based on the hostname.

## Repository Structure

```text
apps/
  traefik/
    namespace.yaml
    values.yaml
    dashboard-ingressroute.yaml
    kustomization.yaml
```

## Files

### `namespace.yaml`

Creates the namespace for Traefik.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: traefik-system
  labels:
    app.kubernetes.io/name: traefik
    app.kubernetes.io/part-of: homelab-platform
```

### `values.yaml`

Contains the Helm values used to install and redeploy Traefik.

Key configuration:

* `service.type: LoadBalancer`

  * allows MetalLB to assign Traefik an external LAN IP

* `ports.web`

  * exposes HTTP on port 80
  * redirects HTTP traffic to HTTPS

* `ports.websecure`

  * exposes HTTPS on port 443
  * enables TLS using the `letsencrypt` certificate resolver

* `providers.kubernetesIngress`

  * enables standard Kubernetes `Ingress` resources

* `providers.kubernetesCRD` (Custom Resource Definitions)

  * enables Traefik CRDs such as `IngressRoute`

* `persistence`

  * stores ACME certificate data in `/data/acme.json`

* `certificatesResolvers.letsencrypt`

  * configures Let's Encrypt with Cloudflare DNS challenge

* `metrics.prometheus`

  * enables Prometheus metrics

* `logs.access`

  * enables access logs

### `dashboard-ingressroute.yaml`

Exposes the Traefik dashboard through Traefik itself.

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: traefik-dashboard
  namespace: traefik-system
  labels:
    app.kubernetes.io/name: traefik-dashboard
    app.kubernetes.io/part-of: homelab-platform
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`traefik.tagawa.ca`)
      kind: Rule
      services:
        - name: api@internal
          kind: TraefikService
  tls:
    certResolver: letsencrypt
```

### `kustomization.yaml`

Groups the Traefik Kubernetes manifests.

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - namespace.yaml
  - dashboard-ingressroute.yaml
```

This allows the manifests to be applied with:

```bash
kubectl apply -k apps/traefik
```

## Cloudflare Secret

The Cloudflare API token is required for the DNS challenge.

Do not commit the real token to Git.

Create the secret manually:

```bash
kubectl create secret generic cloudflare-secret \
  --namespace traefik-system \
  --from-literal=api-token='YOUR_CLOUDFLARE_API_TOKEN'
```

## Installation Steps

### 1. Add the Traefik Helm repo

```bash
helm repo add traefik https://traefik.github.io/charts
helm repo update
```

### 2. Apply the namespace and dashboard manifests

```bash
kubectl apply -k apps/traefik
```

### 3. Install Traefik

```bash
helm upgrade --install traefik traefik/traefik \
  --namespace traefik-system \
  --create-namespace \
  -f apps/traefik/values.yaml
```

### 4. Verify the pod

```bash
kubectl get pods -n traefik-system
```

Expected:

```text
NAME                       READY   STATUS    RESTARTS
traefik-xxxxxxxxxx-xxxxx   1/1     Running   0
```

### 5. Verify the LoadBalancer service

```bash
kubectl get svc -n traefik-system
```

Expected:

```text
NAME      TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)
traefik   LoadBalancer   10.x.x.x       10.0.200.11    80/TCP,443/TCP
```

The `EXTERNAL-IP` should come from the MetalLB IP pool.

### 6. Verify the PVC

```bash
kubectl get pvc -n traefik-system
```

Expected:

```text
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
traefik   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx     1Gi        RWO            local-path
```

## Test Application

The `whoami` application is included as a simple smoke test for Traefik routing and TLS, not as a production workload.

A sample app can be deployed with manifests under:

```text
apps/
  whoami/
    namespace.yaml
    deployment.yaml
    service.yaml
    whoami-ingress.yaml
    kustomization.yaml
```

Apply it:

```bash
kubectl apply -k apps/whoami
```

Verify:

```bash
kubectl get pods -n whoami
kubectl get svc -n whoami
kubectl get ingress -n whoami
```

Test:

```bash
curl -k https://whoami.tagawa.ca
```

Expected response includes information from the `whoami` pod.

## Issues Encountered

### 1. Helm schema error with old values format

Initial error:

```text
Error: values don't meet the specifications of the schema(s)
- at '/ports/web': additional properties 'redirections' not allowed
- at '/ports/websecure': additional properties 'tls' not allowed
```

Cause:

The older values file used this structure:

```yaml
ports:
  web:
    redirections:
      entryPoint:
        to: websecure

  websecure:
    tls:
      enabled: true
```

The newer Traefik Helm chart expects those settings under `http`.

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

### 2. CreateContainerConfigError

The Traefik pod initially showed:

```text
CreateContainerConfigError
```

The logs command returned:

```text
container "traefik" in pod is waiting to start: CreateContainerConfigError
```

This usually means Kubernetes could not build the container configuration.

Useful debug commands:

```bash
kubectl describe pod -n traefik-system -l app.kubernetes.io/name=traefik
```

```bash
kubectl get deployment traefik -n traefik-system -o yaml
```

Likely causes include:

* missing secret
* invalid volume name
* PVC not bound
* init container referencing a volume that does not exist

In this setup, the values were adjusted to ensure the persistence volume was named `data`:

```yaml
persistence:
  enabled: true
  name: data
  path: /data
```

The init container then mounts the same volume:

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

### 3. NodePort values removed

The previous setup used fixed NodePorts:

```yaml
nodePort: 30000
nodePort: 30001
```

These were removed because Traefik is now exposed with:

```yaml
service:
  type: LoadBalancer
```

MetalLB provides the external LAN IP, so direct NodePort access is no longer needed.

Current traffic flow:

```text
Client
  -> MetalLB IP
  -> Traefik LoadBalancer service
  -> Traefik pod
```

## Labeling Convention

The manifests use Kubernetes recommended labels instead of simple custom labels.

Preferred:

```yaml
app.kubernetes.io/name: whoami
app.kubernetes.io/part-of: homelab-platform
```

Instead of:

```yaml
app: whoami
```

Both work, but the `app.kubernetes.io/*` labels are standardized across the Kubernetes ecosystem.

Benefits:

* clearer resource ownership
* better compatibility with Helm, ArgoCD, dashboards, and monitoring tools
* easier filtering and debugging
* more professional and production-like manifests

Example:

```bash
kubectl get all -l app.kubernetes.io/name=whoami -n whoami
```

Another useful grouping label:

```yaml
app.kubernetes.io/part-of: homelab-platform
```

This can later be used to find resources that belong to the overall platform:

```bash
kubectl get all -A -l app.kubernetes.io/part-of=homelab-platform
```

Important rule:

Selectors must match pod labels exactly.

Example:

```yaml
selector:
  app.kubernetes.io/name: whoami
```

must match:

```yaml
labels:
  app.kubernetes.io/name: whoami
```

## Final Validation

Traefik was successfully deployed with:

```bash
kubectl get pods -n traefik-system
```

Result:

```text
traefik-xxxxxxxxxx-xxxxx   1/1   Running
```

The service was successfully assigned a MetalLB IP:

```bash
kubectl get svc -n traefik-system
```

Result:

```text
traefik   LoadBalancer   ...   10.0.200.11   80/TCP,443/TCP
```

The PVC was successfully bound:

```bash
kubectl get pvc -n traefik-system
```

Result:

```text
traefik   Bound   ...   1Gi   RWO   local-path
```
