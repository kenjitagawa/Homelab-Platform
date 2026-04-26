# Homelab Platform 

A homelab Kubernetes platform project focused on organizing deployments, infrastructure configuration, and observability in a clean repository.

This repository is my reference for running the Kubernetes platform layer of my homelab. The docs explain the current setup and the decisions behind it, especially around MetalLB, Traefik, ingress, TLS, and observability.

## Goals

- create a clean structure for Kubernetes-related resources
- deploy and manage workloads in the homelab
- add observability with Prometheus and Grafana
- document architecture and operational decisions
- prepare for GitOps adoption with ArgoCD

## Planned Phases

1. Repository foundation
2. Cluster baseline
3. Observability
4. Portfolio polish
5. Future GitOps expansion

## Repository Structure

- `apps/` — application manifests and related configuration
- `infra/` — infrastructure-level Kubernetes resources and shared config
- `docs/` — architecture, setup notes, and project documentation
