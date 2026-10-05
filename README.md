# Kiali Deployment

[![Helm unittest][unittest-badge]][unittest-workflow]
[![Trufflehog][trufflehog-badge]][trufflehog-workflow]
[![License: Apache 2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

[unittest-badge]: https://github.com/steadforce/kiali-deployment/actions/workflows/helm-unittest.yaml/badge.svg
[unittest-workflow]: https://github.com/steadforce/kiali-deployment/actions/workflows/helm-unittest.yaml
[trufflehog-badge]: https://github.com/steadforce/kiali-deployment/actions/workflows/trufflehog.yaml/badge.svg
[trufflehog-workflow]: https://github.com/steadforce/kiali-deployment/actions/workflows/trufflehog.yaml

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
- [Prerequisites](#prerequisites)
- [Repository Layout](#repository-layout)
- [Setup](#setup)
- [Rendering Templates Locally](#rendering-templates-locally)
- [Testing](#testing)
- [CI/CD](#cicd)
- [Dependency Updates](#dependency-updates)

## Overview

This is an umbrella chart around the upstream `kiali-operator` chart from `https://kiali.org/helm-charts`. The
subchart version follows the chart's `appVersion` through a YAML anchor in `Chart.yaml`. The chart configures:

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
| `virtual-service.yaml` | `VirtualService` | Exposes Kiali via the ingress gateway | `networking.istio.io/v1beta1` |
| `default-sidecar.yaml` | `Sidecar` | Restricts the default sidecar egress scope | `networking.istio.io/v1beta1` |
| `kiali-vpa.yaml` | `VerticalPodAutoscaler` | Disables Kiali VPA resizing | `autoscaling.k8s.io/v1` |
| `kialioperator-vpa.yaml` | `VerticalPodAutoscaler` | Disables operator VPA resizing | `autoscaling.k8s.io/v1` |
| `copy-grafana-admin.yaml` | Kyverno `ClusterPolicy` | Copies the Grafana admin secret | `kyverno.io/v1` |
| `copy-oidc-client-token.yaml` | Kyverno `ClusterPolicy` | Copies the OIDC client secret | `kyverno.io/v1` |
| `namespace.yaml` | `Namespace` | Creates the namespace with Istio sidecar injection enabled | - |

Templates with a `Required API` only render when Argo CD (or `helm template --api-versions`) reports that API as
available, so a CRD that is not installed yet never blocks the rest of the chart from syncing.

## Argo CD Sync Waves

- `copy-grafana-admin.yaml` and `copy-oidc-client-token.yaml` run at wave `-5`, so their secrets exist before Kiali
  starts.
- `kiali.yaml` runs at wave `100`, so the `Kiali` custom resource is created only after every other resource in this
  chart has synced.
- Every other template uses the default wave (`0`).

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/), to run every command below in a container.

All commands run from the repository root.

## Repository Layout

| Path | Purpose |
| --- | --- |
| `Chart.yaml` | Chart metadata and the `kiali-operator` dependency |
| `Chart.lock` | Committed lock file that pins the resolved subchart version |
| `charts/` | Subchart archive; `kiali-operator-<version>.tgz` is committed |
| `templates/` | Resources the umbrella chart adds (see [Chart Structure](#chart-structure)) |
| `values.yaml` | Chart defaults, including the local cluster's ACME domain |
| `values-subchart-overrides.yaml` | Tunes the `kiali-operator` subchart (pull policy, ad hoc images, resources) |
| `values-development.yaml` | Overrides the ACME domain for the development cluster |
| `values-production.yaml` | Overrides the ACME domain for the production cluster |
| `values-sf-k8s03-dev.yaml` | Overrides the ACME domain for the k8s03-dev cluster |
| `values-sf-k8s04-dev.yaml` | Overrides the ACME domain for the k8s04-dev cluster |
| `values-local.yaml` | Local clusters: zeroes requests and CPU limits, drops pod annotations, skips OIDC TLS checks |
| `tests/` | helm-unittest suites; `tests/__snapshot__/` is gitignored |
| `renovate.json` | Renovate configuration |

`values.yaml` sets `global.acme.domain`, `global.istio.ingressGateway.namespace`, the `kiali` block (OpenID TLS
verification, sidecar pod annotations, resources), and `subDomain`. The rendering commands below layer
`values-subchart-overrides.yaml` first, then the environment file; k8s03-dev and k8s04-dev add their cluster file
on top of `values-development.yaml`, and local clusters use `values-local.yaml` instead of an environment file.

## Setup

The subchart archive is committed, so a fresh clone renders right away. To restore `charts/` to the version
`Chart.lock` pins, for example after a pull that changes `Chart.lock`, register the dependency repositories from
`Chart.yaml` and build the dependencies:

```sh
 docker run \
   -e HOME=/tmp \
   --entrypoint sh \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm -c '
     yq "explode(.) | .dependencies[] | select(.repository == \"http*\") | .name + \" \" + .repository" Chart.yaml |
       while read -r name repo; do helm repo add --force-update "$name" "$repo"; done &&
     helm dependency build .
   '
```

## Rendering Templates Locally

Render the chart per environment to validate the output before pushing. The output lands in a gitignored
`_<environment>` directory.

### Local

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template . \
   --api-versions autoscaling.k8s.io/v1 \
   --api-versions kiali.io/v1alpha1 \
   --api-versions kyverno.io/v1 \
   --api-versions networking.istio.io/v1beta1 \
   --include-crds \
   --namespace kiali-operator \
   --output-dir _local \
   --skip-tests \
   --values values-subchart-overrides.yaml \
   --values values-local.yaml
```

### Development

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template . \
   --api-versions autoscaling.k8s.io/v1 \
   --api-versions kiali.io/v1alpha1 \
   --api-versions kyverno.io/v1 \
   --api-versions networking.istio.io/v1beta1 \
   --include-crds \
   --namespace kiali-operator \
   --output-dir _development \
   --skip-tests \
   --values values-subchart-overrides.yaml \
   --values values-development.yaml
```

### Production

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template . \
   --api-versions autoscaling.k8s.io/v1 \
   --api-versions kiali.io/v1alpha1 \
   --api-versions kyverno.io/v1 \
   --api-versions networking.istio.io/v1beta1 \
   --include-crds \
   --namespace kiali-operator \
   --output-dir _production \
   --skip-tests \
   --values values-subchart-overrides.yaml \
   --values values-production.yaml
```

### k8s03-dev

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template . \
   --api-versions autoscaling.k8s.io/v1 \
   --api-versions kiali.io/v1alpha1 \
   --api-versions kyverno.io/v1 \
   --api-versions networking.istio.io/v1beta1 \
   --include-crds \
   --namespace kiali-operator \
   --output-dir _sf-k8s03-dev \
   --skip-tests \
   --values values-subchart-overrides.yaml \
   --values values-development.yaml \
   --values values-sf-k8s03-dev.yaml
```

### k8s04-dev

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm template . \
   --api-versions autoscaling.k8s.io/v1 \
   --api-versions kiali.io/v1alpha1 \
   --api-versions kyverno.io/v1 \
   --api-versions networking.istio.io/v1beta1 \
   --include-crds \
   --namespace kiali-operator \
   --output-dir _sf-k8s04-dev \
   --skip-tests \
   --values values-subchart-overrides.yaml \
   --values values-development.yaml \
   --values values-sf-k8s04-dev.yaml
```

> [!NOTE]
> `--api-versions` must list every capability an `if .Capabilities.APIVersions.Has` check in this chart looks for.
> Helm templating works offline, so it cannot discover cluster capabilities on its own; omitting one of these flags
> silently skips the resources that depend on it.

## Testing

Run the helm-unittest suites in `tests/`, including the tests of subcharts under `charts/` (enabled by default):

```sh
 docker run \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest .
```

Update the stored snapshots after an intentional rendering change:

```sh
 docker run \
   -e HELM_CACHE_HOME=/tmp/helm/.config \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   helmunittest/helm-unittest -u .
```

> [!IMPORTANT]
> `tests/__snapshot__/` is not committed to this repository. Regenerating it locally is expected and does not need
> to be staged.

To write a JUnit report like CI does, add `-t JUnit -o test-output.xml`; without `-t`, helm-unittest writes XUnit.

CI also lints the chart:

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm lint .
```

## CI/CD

Both workflows call reusable workflows from
[`steadforce/steadops-workflows`](https://github.com/steadforce/steadops-workflows):

- **Helm unittest** (`helm-unittest.yaml@v4.2.0`) runs on every push. It builds the dependencies from `Chart.lock`
  (falling back to `helm dependency update` with a warning when no lock file exists), runs
  `helm unittest --with-subchart=true -t JUnit -o test-output.xml`, publishes the results as a check run, and runs
  `helm lint`.
- **Trufflehog** (`trufflehog-oss.yaml@v4.2.0`) scans the commit range of pushes and pull requests to `main`, and
  can be started manually. The job fails when Trufflehog detects a secret.

On branches starting with `renovate/`, the Helm unittest workflow posts the result to Microsoft Teams. Both
secrets are optional:

| Repository secret | Passed as | Receives |
| --- | --- | --- |
| `STEADOPS_HELM_RENOVATION_MS_TEAMS_WEBHOOK` | `steadops-helm-renovation-ms-teams-webhook` | Successes |
| `STEADOPS_HELM_RENOVATION_ERROR_MS_TEAMS_WEBHOOK` | `steadops-helm-renovation-ms-teams-error-webhook` | Failures |

Failures go to the regular webhook when the error secret is not set, and no notification is sent when neither
secret is set. Both must be Microsoft Teams Workflows webhook URLs; legacy Office 365 connector URLs no longer work.

## Dependency Updates

Renovate extends `config:base` and keeps a dependency dashboard issue. It opens pull requests for new
`kiali-operator` versions without automerging them, and `helmUpdateSubChartArchives` refreshes the committed
archive under `charts/` in the same pull request.

To change the dependency manually, edit `Chart.yaml`, then regenerate `Chart.lock` and the archive:

```sh
 docker run \
   -e HOME=/tmp \
   --rm \
   -u $(id -u) \
   -v "$(pwd):/apps" \
   -w /apps \
   alpine/helm dependency update .
```

Commit the new `Chart.lock` and the archive under `charts/` together with `Chart.yaml`. `charts/` is gitignored,
so stage a new archive with `git add -f`. See the
[Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies) for details.
