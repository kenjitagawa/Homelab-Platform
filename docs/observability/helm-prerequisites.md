# Helm Prerequisites for Observability

## Overview

Helm is required for installing observability tooling such as Prometheus and Grafana through the `kube-prometheus-stack` chart.

The observability stack uses the `prometheus-community` Helm repository.

## Install Helm

On Linux, install Helm with the official install script:

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash

Fedora:
sudo dnf install helm


MacOS:
brew install helm
```

Verify Helm is installed:

```bash
helm version --short
```

Expected:

```text
v4.x.x+...
```

## Add Required Repositories

Add the Prometheus Community Helm repository:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
```

Update local chart metadata:

```bash
helm repo update
```

Verify the repository is configured:

```bash
helm repo list
```

Expected:

```text
NAME                    URL
prometheus-community    https://prometheus-community.github.io/helm-charts
```

## Verify Chart Access

Confirm the `kube-prometheus-stack` chart is available:

```bash
helm search repo prometheus-community/kube-prometheus-stack
```

Expected:

```text
NAME                                             CHART VERSION   APP VERSION
prometheus-community/kube-prometheus-stack       ...
```

After these steps, the local machine is ready to install observability tooling with Helm.
