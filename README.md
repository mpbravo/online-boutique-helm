# Online Boutique – Helm Chart Repository

This branch (`gh-pages`) is the **Helm chart repository** for the Online Boutique chart. It is managed automatically by the [Chart Releaser Action](https://github.com/helm/chart-releaser-action) and should not be edited manually.

## Using This Repository

Add the repository to Helm:

```sh
helm repo add online-boutique https://mpbravo.github.io/online-boutique-helm/
helm repo update
```

Search for available charts:

```sh
helm search repo online-boutique
```

## Installing Online Boutique

### Default installation

```sh
helm install onlineboutique online-boutique/onlineboutique \
  --namespace onlineboutique \
  --create-namespace
```

### OpenShift-optimised installation

```sh
helm install onlineboutique online-boutique/onlineboutique \
  --namespace onlineboutique \
  --create-namespace \
  --set securityContext.enable=true \
  --set frontend.externalService=true \
  --set frontend.platform=local
```

### Using a custom values file

```sh
helm install onlineboutique online-boutique/onlineboutique \
  --namespace onlineboutique \
  --create-namespace \
  -f my-values.yaml
```

## Available Charts

| Chart | Description | App Version |
|---|---|---|
| `onlineboutique` | Cloud-native microservices demo e-commerce application (11 services) | See [releases](https://github.com/mpbravo/online-boutique-helm/releases) |

## Adding the Repository to OpenShift

Apply the provided `ProjectHelmChartRepository` manifest from the `main` branch:

```sh
oc apply -f https://raw.githubusercontent.com/mpbravo/online-boutique-helm/main/openshift-helm-repo.yaml
```

This makes the chart available in the OpenShift Web Console's **Helm Catalog** with a built-in form view powered by the chart's `values.schema.json`.

## How Releases Are Automated

Every push to the `main` branch triggers the [Release Helm Charts](.github/workflows/release.yml) GitHub Actions workflow:

1. **Detects changed charts** — compares chart versions against existing release tags.
2. **Packages charts** — runs `helm package` on each changed chart.
3. **Creates a GitHub Release** — named `<chart-name>-<version>` (e.g. `onlineboutique-0.10.4`), with the `.tgz` package attached as an artifact.
4. **Updates `index.yaml`** — pushes an updated Helm repository index to this branch, which GitHub Pages serves at `https://mpbravo.github.io/online-boutique-helm/`.

## Source Repository

Chart source, templates, and documentation live on the `main` branch:
[github.com/mpbravo/online-boutique-helm](https://github.com/mpbravo/online-boutique-helm)
