# Kiali-Operator and Kiali

Repository containing manifests for
[kiali-operator plus kiali](https://kiali.io/docs/installation/installation-guide/install-with-helm/)
installation. Never install the content of this repo on our clusters manually. This is all done by argo-cd.

## Dependencies

This chart pulls in `kiali-operator` as a dependency. The version
used is specified in `Chart.yaml` in the `dependencies` section.
If you change the version in there, you need to then run

    $ helm dependency update

in order to have the chart downloaded to the `charts` directory
and then also commit that new version alongside with the altered
`Chart.yaml` file.

See the [Helm docs](https://helm.sh/docs/topics/charts/#chart-dependencies)
for details.


## Render all manifests locally

```shell
 helm dependency update && \
 for cluster in $(yq '.environments | keys[]' helm-config.yaml); do
    helm template \
      -a "$(cluster=$cluster yq '.environments.[env(cluster)].apis | @csv' helm-config.yaml)" \
      -f "$(cluster=$cluster yq '.environments.[env(cluster)].valueFiles | @csv' helm-config.yaml)" \
      -n $(yq 'explode(.) | .namespace // ""' helm-config.yaml) \
      --output-dir _local/$cluster \
      --include-crds \
      --release-name $(yq 'explode(.) | .releaseName // ""' helm-config.yaml) \
      --skip-tests \
      .
 done
```

# Testing

## values-subchart-overrides.yaml

The `values-subchart-overrides.yaml` file is used to override values in the postgres-operator chart.
We have to separate the values for the subcharts from the values for the main chart, to be able to
unit test for incompatible changes in values of the subcharts. This is necessary because helm does not allow
switching off the usage of values.yaml. Now it's possible to test if we use the same registry and repository
for images as the subcharts are using.

## run helm unittests

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest .
```

Or with output in JUnit format:

```shell
 docker run --pull=always -ti --rm -v "$(pwd):/apps" -u $(id -u) helmunittest/helm-unittest -o test-output.xml .
```

## Run act pipeline locally

To run the pipeline in local environment, start up the workbench, cd into the folder containing this
`README.md` and execute the following command:

```shell
  act
```

On first execution you're asked which flavour of the act image should be used. Using the default `medium`
is a good starting point.

## Hydration Workflow

This repository implements a **GitOps Hydration Pattern**.
The `helm-hydration.yaml` workflow is triggered by pushes to the `main` branch. It renders the Helm charts into static Kubernetes manifests and opens automated Pull Requests targeting the specific environment branches (e.g., `environments/local`, `environments/production`) defined in `helm-config.yaml`.

### API Capabilities Configuration
Because the hydration process runs in a CI environment without access to a live Kubernetes cluster, it must **mock** the cluster's available APIs (CRDs). This is controlled via the `apis` list in `helm-config.yaml`.

If a chart (or its dependencies) uses conditional logic like `if .Capabilities.APIVersions.Has "..."`, and the specific API is missing from `helm-config.yaml`, the resource will **not** be rendered in the final manifest.

### dependency Scanning
To ensure all conditional resources are correctly rendered, use the provided static analysis tool:

```bash
./scan-helm-capabilities.sh
```

This script:
1.  Downloads and extracts all chart dependencies locally.
2.  Recursively scans all templates (`.yaml`, `.tpl`) in your chart and its sub-charts.
3.  Identifies every instance of `.Capabilities.APIVersions.Has`.
4.  Outputs the exact list of API strings (Groups and Kinds) required in your `helm-config.yaml`.