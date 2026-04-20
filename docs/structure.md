# Repository Structure

## Purpose

This repository manages the **Kubernetes platform layer** of my homelab environment.

It is responsible for defining and maintaining:

* Kubernetes namespaces
* Kubernetes manifests
* Helm values and configurations
* Application/workload definitions
* Cluster-level services (e.g., ingress, monitoring)


## Boundary of Responsibility

This repository **does not** handle infrastructure provisioning or cluster creation.

The following responsibilities are managed in the **Ansible-K3S-Cluster** repository:

* Operating system provisioning
* K3S installation
* Node preparation and configuration
* Host-level networking and system setup

### Summary

| Responsibility                                    | Repository          |
| ------------------------------------------------- | ------------------- |
| Cluster creation (K3S, nodes, OS)                 | Ansible-K3S-Cluster |
| Cluster configuration (apps, manifests, services) | Homelab-Platform    |


## Repository Layout

The repository follows a structure designed to support future GitOps adoption.

```sh
.
├── apps
│   ├── traefik/
│   ├── metallb/
│   ├── kube-prometheus-stack/
│   └── sample-app/
├── docs
│   └── structure.md
├── infra/
└── README.md
```

### Directory Overview

#### `apps/`

Contains application and platform service definitions.

Each application or service has its own directory, which may include:

* Kubernetes manifests
* Helm values (`helm-values.yaml`)
* Supporting configuration files

Examples:

* `traefik/` - ingress controller
* `metallb/` - load balancer
* `kube-prometheus-stack/` - monitoring stack
* `sample-app/` - test workload


#### `infra/`

Reserved for cluster-level infrastructure components that do not belong to a single app.

Examples may include:

* shared resources
* cluster-wide configurations
* future GitOps bootstrap resources (e.g., ArgoCD root apps)


#### `docs/`

Contains all documentation related to the repository.


## Naming Conventions

### Files and Directories

* Use **lowercase**
* Use **kebab-case** (hyphen-separated)
* Use **descriptive names**

#### Examples

```text
helm-values.yaml
deployment-k8s-nextjs-app.yaml
ingress-grafana.yaml
```


### Kubernetes Resource Names

* Use **lowercase kebab-case**
* Keep names **short and explicit**
* Names should reflect the **purpose of the resource**

#### Examples

```text
monitoring
sample-app
traefik-system
kube-prometheus-stack
```


## Namespace Conventions

Namespaces are kept **simple and readable**, optimized for homelab usage.

### Rules

* Each application or major service should have its own namespace
* Use lowercase kebab-case
* Keep names short and meaningful

### Examples

```text
monitoring
argocd
traefik
sample-app
```


## Manifest Naming Conventions

All Kubernetes manifests follow a predictable naming pattern.

### Pattern

```text
<resource-type>-<resource-name>.yaml
```

### Examples

```text
namespace-monitoring.yaml
service-sample-app.yaml
ingress-grafana.yaml
deployment-sample-app.yaml
```


## Helm-Based Applications

For applications deployed via Helm:

* Store configuration inside the app directory
* Use a standard filename:

```text
helm-values.yaml
```

### Example

```text
apps/kube-prometheus-stack/helm-values.yaml
```

This ensures consistency across all Helm-managed services.


## Adding a New Application or Service

To add a new application:

1. Create a new directory under `apps/`
2. Add Kubernetes manifests and/or Helm values
3. Follow naming conventions for all files and resources
4. Assign a dedicated namespace if applicable

Example:

```text
apps/my-app/
  deployment-my-app.yaml
  service-my-app.yaml
  ingress-my-app.yaml
```


## Future Direction (GitOps)

This repository is structured to support future integration with tools like:

* ArgoCD - Still deciding, but probably going with ArgoCD
* Flux

The `apps/` structure allows easy transition to:

* app-of-apps patterns
* environment-based deployments
* declarative synchronization
