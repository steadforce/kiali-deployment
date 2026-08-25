# Kiali Deployment

[![Helm unittest](https://github.com/steadforce/kiali-deployment/actions/workflows/helm-unittest.yaml/badge.svg)](https://github.com/steadforce/kiali-deployment/actions/workflows/helm-unittest.yaml)
[![Trufflehog](https://github.com/steadforce/kiali-deployment/actions/workflows/trufflehog.yaml/badge.svg)](https://github.com/steadforce/kiali-deployment/actions/workflows/trufflehog.yaml)
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

Helm chart that installs [Kiali](https://kiali.io/) through the
[`kiali-operator`](https://kiali.io/docs/installation/installation-guide/install-with-helm/) chart, plus the
supporting resources it needs to run on the Steadforce platform: ingress routing, secret provisioning, and
autoscaling opt-outs.

> [!WARNING]
> Never install the content of this repository on a cluster manually. Deployment happens exclusively through
> Argo CD.

## Table of Contents

- [Overview](#overview)
- [Chart Structure](#chart-structure)
- [Argo CD Sync Waves](#argo-cd-sync-waves)
- [Configuration](#configuration)
- [Dependencies](#dependencies)
- [Prerequisites](#prerequisites)
- [Rendering Templates Locally](#rendering-templates-locally)
- [Running Unit Tests](#running-unit-tests)
- [Continuous Integration](#continuous-integration)

## Overview

This is an umbrella chart around the upstream `kiali-operator` chart. It configures:

- the `kiali-operator` deployment itself, through subchart value overrides
- a `Kiali` custom resource that the operator reconciles into a running Kiali instance
- ingress and service-mesh wiring so Kiali is reachable through Istio
- Kyverno policies that copy the secrets Kiali depends on before it starts
- `VerticalPodAutoscaler` resources that opt both deployments out of automatic resizing

## Chart Structure

| Template | Resource | Purpose | Required API |
| --- | --- | --- | --- |
| `kiali.yaml` | `Kiali` | Configures the Kiali instance the operator reconciles | `kiali.io/v1alpha1` |
| `forecastle-app.yaml` | `ForecastleApp` | Adds a Forecastle link to the Kiali UI | - |
| `virtual-service.yaml` | `VirtualService` | Exposes Kiali through the Istio ingress gateway | `networking.istio.io/v1beta1` |
| `default-sidecar.yaml` | `Sidecar` | Restricts the default Istio sidecar egress scope | `networking.istio.io/v1beta1` |
| `kiali-vpa.yaml` | `VerticalPodAutoscaler` | Disables VPA resizing for the Kiali deployment | `autoscaling.k8s.io/v1` |
| `kialioperator-vpa.yaml` | `VerticalPodAutoscaler` | Disables VPA resizing for the operator deployment | `autoscaling.k8s.io/v1` |
| `copy-grafana-admin.yaml` | Kyverno `ClusterPolicy` | Copies the Grafana admin secret into the namespace | `kyverno.io/v1` |
| `copy-oidc-client-token.yaml` | Kyverno `ClusterPolicy` | Copies the OIDC client secret into the namespace | `kyverno.io/v1` |
| `namespace.yaml` | `Namespace` | Creates the namespace with Istio sidecar injection enabled | - |

Templates with a `Required API` only render when Argo CD (or `helm template --api-versions`) reports that API as
available, so a CRD that is not installed yet never blocks the rest of the chart from syncing.

## Argo CD Sync Waves

- `copy-grafana-admin.yaml` and `copy-oidc-client-token.yaml` run at wave `-5`, so their secrets exist before Kiali
  starts.
- `kiali.yaml` runs at wave `100`, so the `Kiali` custom resource is created only after every other resource in this
  chart has synced.
- Every other template uses the default wave (`0`).

## Configuration

| File | Purpose |
| --- | --- |
| `values.yaml` | Chart defaults, also used as-is for the local cluster's ACME domain |
| `values-development.yaml` | Overrides the ACME domain for the development cluster |
| `values-production.yaml` | Overrides the ACME domain for the production cluster |
| `values-sf-k8s03-dev.yaml` | Overrides the ACME domain for the k8s03-dev cluster |
| `values-sf-k8s04-dev.yaml` | Overrides the ACME domain for the k8s04-dev cluster |
| `values-subchart-overrides.yaml` | Tunes the `kiali-operator` subchart (image pull policy, ad hoc images, resources) |
| `values-local.yaml` | Zeroes resource requests/limits and disables TLS verification for local clusters |

Argo CD applies `values.yaml`, then `values-subchart-overrides.yaml`, then the environment file; for local clusters, use `values-local.yaml` instead of the environment file (and apply it last so its overrides win).

## Dependencies

This chart pulls in `kiali-operator` as a dependency. The version used is pinned in `Chart.yaml`, in the
`dependencies` section. After changing that version, refresh the vendored copy under `charts/` and commit the
result alongside the updated `Chart.yaml` and `Chart.lock`:

```sh
 docker run \
   --rm \
   -u $(id -u) \
   -e HOME=/tmp \
   -v $(pwd):/apps \
   -w /apps \
   alpine/helm dependency update .
```

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies) for details.

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/), to run every command below in the same containers Argo CD and CI
  use.

## Rendering Templates Locally

The following commands render the chart the same way Argo CD does, so you can validate the output before pushing.

### Local

```sh
 docker run \
   --rm \
   -u $(id -u) \
   -e HOME=/tmp \
   -v $(pwd):/apps \
   -w /apps \
   alpine/helm template . \
   --api-versions autoscaling.k8s.io/v1 \
   --api-versions kiali.io/v1alpha1 \
   --api-versions kyverno.io/v1 \
   --api-versions networking.istio.io/v1beta1 \
   --include-crds \
   --namespace kiali-operator \
   --output-dir _local \
   --release-name kiali \
   --skip-tests \
   --values values-subchart-overrides.yaml \
   --values values-local.yaml
```

### Development

```sh
 docker run \
   --rm \
   -u $(id -u) \
   -e HOME=/tmp \
   -v $(pwd):/apps \
   -w /apps \
   alpine/helm template . \
   --api-versions autoscaling.k8s.io/v1 \
   --api-versions kiali.io/v1alpha1 \
   --api-versions kyverno.io/v1 \
   --api-versions networking.istio.io/v1beta1 \
   --include-crds \
   --namespace kiali-operator \
   --output-dir _development \
   --release-name kiali \
   --skip-tests \
   --values values-development.yaml
```

### Production

```sh
 docker run \
   --rm \
   -u $(id -u) \
   -e HOME=/tmp \
   -v $(pwd):/apps \
   -w /apps \
   alpine/helm template . \
   --api-versions autoscaling.k8s.io/v1 \
   --api-versions kiali.io/v1alpha1 \
   --api-versions kyverno.io/v1 \
   --api-versions networking.istio.io/v1beta1 \
   --include-crds \
   --namespace kiali-operator \
   --output-dir _production \
   --release-name kiali \
   --skip-tests \
   --values values-production.yaml
```

### k8s03-dev

```sh
 docker run \
   --rm \
   -u $(id -u) \
   -e HOME=/tmp \
   -v $(pwd):/apps \
   -w /apps \
   alpine/helm template . \
   --api-versions autoscaling.k8s.io/v1 \
   --api-versions kiali.io/v1alpha1 \
   --api-versions kyverno.io/v1 \
   --api-versions networking.istio.io/v1beta1 \
   --include-crds \
   --namespace kiali-operator \
   --output-dir _sf-k8s03-dev \
   --release-name kiali \
   --skip-tests \
   --values values-sf-k8s03-dev.yaml
```

### k8s04-dev

```sh
 docker run \
  --rm \
  -u $(id -u) \
  -e HOME=/tmp \
  -v $(pwd):/apps \
  -w /apps \
  alpine/helm template . \
  --api-versions autoscaling.k8s.io/v1 \
  --api-versions kiali.io/v1alpha1 \
  --api-versions kyverno.io/v1 \
  --api-versions networking.istio.io/v1beta1 \
  --include-crds \
  --namespace kiali-operator \
  --output-dir _sf-k8s04-dev \
  --release-name kiali \
  --skip-tests \
  --values values-sf-k8s04-dev.yaml
```

> [!NOTE]
> `--api-versions` must list every capability an `if .Capabilities.APIVersions.Has` check in this chart looks for.
> Helm templating works offline, so it cannot discover cluster capabilities on its own; omitting one of these flags
> silently skips the resources that depend on it.

## Running Unit Tests

```sh
 docker run \
   --rm \
   -u $(id -u) \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   -v $(pwd):/apps \
   -w /apps \
   helmunittest/helm-unittest .
```

Update the stored snapshots after an intentional rendering change:

```sh
 docker run \
   --rm \
   -u $(id -u) \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   -v $(pwd):/apps \
   -w /apps \
   helmunittest/helm-unittest -u .
```

> [!IMPORTANT]
> `tests/__snapshot__/` is not committed to this repository. Regenerating it locally is expected and does not need
> to be staged.

## Continuous Integration

Every push runs two reusable workflows from
[`steadforce/steadops-workflows`](https://github.com/steadforce/steadops-workflows):

- **Helm unittest** — runs the suites in `tests/` against the rendered chart.
- **Trufflehog** — scans the repository for committed secrets on pushes and pull requests to `main`.
