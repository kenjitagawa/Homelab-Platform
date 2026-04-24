# Homelab Platform 

A homelab Kubernetes platform project focused on organizing deployments, infrastructure configuration, and observability in a clean repository.

This repository includes documentation primarily intended as a personal reference for future use. It has been refined with the help of AI to improve clarity and structure, and should be sufficiently detailed for others who wish to follow along and build a similar homelab platform. Where relevant, the documentation also provides deeper explanations of specific configurations, such as Traefik and MetalLB, to reinforce understanding of how these components operate.

## Goals

- create a clean structure for Kubernetes-related resources
- deploy and manage workloads in the homelab
- add observability with Prometheus and Grafana
- document architecture and operational decisions
- prepare for future GitOps adoption with ArgoCD

## Planned Phases

1. Repository foundation
2. Cluster baseline
3. Observability
4. Portfolio polish
5. Future GitOps expansion

## Repository Structure

- `apps/` — application manifests and related configuration
- `infra/` — infrastructure-level Kubernetes resources and shared config
- `monitoring/` — observability stack configuration
- `docs/` — architecture, setup notes, and project documentation

## Status

This repository is currently in the foundation phase.
