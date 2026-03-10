# Online Boutique – Helm Chart for OpenShift

This is a Helm chart for deploying [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo), a cloud-native microservices demo e-commerce application, on **Red Hat OpenShift**.

The chart is adapted from the [upstream Google Microservices Demo](https://github.com/GoogleCloudPlatform/microservices-demo/tree/main) with OpenShift-compatible security contexts and a `values.schema.json` that enables the **Form View** in the OpenShift Web Console.

> **Note:** Online Boutique's Helm chart is currently experimental. Feedback and issues can be filed at [GitHub Issue #1319](https://github.com/GoogleCloudPlatform/microservices-demo/issues/1319).

---

## Quick Start

### Deploy with default settings

```sh
helm upgrade onlineboutique ./online-boutique \
    --install \
    --namespace onlineboutique \
    --create-namespace
```

### Deploy with OpenShift-optimised settings (on-prem / local cluster)

```sh
helm upgrade onlineboutique ./online-boutique \
    --install \
    --namespace onlineboutique \
    --create-namespace \
    --set securityContext.enable=true \
    --set securityContext.openshift=true \
    --set frontend.externalService=false \
    --set frontend.route.create=true \
    --set frontend.route.tls.enabled=true \
    --set frontend.platform=local
```

### Deploy an advanced scenario (Istio + Spanner + custom registry)

```sh
helm upgrade onlineboutique ./online-boutique \
    --install \
    --namespace onlineboutique \
    --create-namespace \
    --set images.repository=us-docker.pkg.dev/my-project/microservices-demo \
    --set frontend.externalService=false \
    --set cartDatabase.inClusterRedis.create=false \
    --set cartDatabase.type=spanner \
    --set cartDatabase.connectionString=projects/my-project/instances/onlineboutique/databases/carts \
    --set serviceAccounts.create=true \
    --set serviceAccounts.annotationsOnlyForCartservice=true \
    --set 'serviceAccounts.annotations.iam\.gke\.io/gcp-service-account=spanner-db-user@my-project.iam.gserviceaccount.com' \
    --set authorizationPolicies.create=true \
    --set networkPolicies.create=true \
    --set sidecars.create=true \
    --set frontend.virtualService.create=true \
    -n onlineboutique
```

For the full list of configuration options, see [`online-boutique/values.yaml`](./online-boutique/values.yaml).

---

## OpenShift Web Console Deployment

Add the Helm repository to OpenShift and deploy directly from the **Developer** perspective using the built-in form view:

```sh
oc apply -f openshift-helm-repo.yaml
```

See the [OpenShift Deployment Guide](./OPENSHIFT-GUIDE.md) for step-by-step instructions, deployment scenarios, and troubleshooting tips.

---

## Repository Contents

| Path | Description |
|---|---|
| `online-boutique/` | Helm chart source (templates, values, schema) |
| `online-boutique/values.yaml` | Default configuration values |
| `online-boutique/values.schema.json` | JSON Schema for the OpenShift form view |
| `online-boutique/questions.yaml` | Rancher-compatible question definitions |
| `openshift-helm-repo.yaml` | OpenShift `ProjectHelmChartRepository` manifest |
| `OPENSHIFT-GUIDE.md` | Full OpenShift deployment guide |

---

## Further Reading

- [Online Boutique Helm chart – advanced scenarios with Service Mesh and GitOps](https://medium.com/google-cloud/246119e46d53)
- [gRPC health probes with Kubernetes 1.24+](https://medium.com/google-cloud/b5bd26253a4c)
- [Use Google Cloud Spanner with Online Boutique](https://medium.com/google-cloud/f7248e077339)
