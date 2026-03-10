# Online Boutique – OpenShift Deployment Guide

## Overview

This guide covers how to deploy the **Online Boutique** microservices demo application on **Red Hat OpenShift** using its Helm chart, both from the OpenShift Web Console and the CLI.

Online Boutique is a cloud-native demo e-commerce application composed of 11 microservices. The chart has been modified from the [upstream Google Microservices Demo](https://github.com/GoogleCloudPlatform/microservices-demo) to integrate cleanly with OpenShift's security model.

---

## Prerequisites

- OpenShift 4.10+ cluster with cluster-admin or project-admin access
- Helm 3.x (`helm version`)
- OpenShift CLI (`oc version`)
- A project/namespace to deploy into

---

## Quick Start: Deploy from the OpenShift Web Console

### Step 1: Add the Helm Repository

#### Option A: Web Console

1. Switch to the **Administrator** perspective.
2. Navigate to **Helm** → **Helm Repositories**.
3. Click **Create** → **HelmChartRepository** (cluster-wide) or **ProjectHelmChartRepository** (project-scoped).
4. Paste the configuration from `openshift-helm-repo.yaml` and click **Create**.

#### Option B: CLI

```bash
oc apply -f openshift-helm-repo.yaml
```

Or apply it inline:

```bash
oc apply -f - <<EOF
apiVersion: helm.openshift.io/v1beta1
kind: ProjectHelmChartRepository
metadata:
  name: online-boutique-charts
spec:
  connectionConfig:
    url: https://mpbravo.github.io/helm-charts/
  name: Online Boutique Charts
  description: Helm chart for the Online Boutique microservices demo
EOF
```

---

### Step 2: Deploy from the Web Console

1. Switch to the **Developer** perspective.
2. Select or create the target project/namespace.
3. Click **+Add** → **Helm Chart**.
4. Search for **"onlineboutique"** and click the chart card.
5. Click **Install Helm Chart**.

#### Using the Form View

The chart ships with a `values.schema.json` that drives OpenShift's built-in form view. All parameters are grouped into collapsible sections:

| Section | Key parameters |
|---|---|
| **Container Images** | Image repository, image tag |
| **Service Accounts** | Auto-create, cart-only annotations (Workload Identity) |
| **Network Policies** | Enable fine-grained NetworkPolicies per service |
| **Istio Sidecars / AuthorizationPolicies** | Enable Istio resources per service |
| **OpenTelemetry Collector** | Deploy collector, GCP project ID |
| **Google Cloud Operations** | Cloud Profiler, Trace, Monitoring |
| **Seccomp Profile** | Enable, profile type |
| **Security Context** | Enable non-root / read-only container contexts |
| **Per-service settings** | Deploy toggle, service name, CPU/memory requests & limits |
| **Frontend** | External service, Cymbal branding, platform, Istio VirtualService |
| **Load Generator** | Deploy toggle, frontend readiness init container |
| **Product Catalog Service** | Artificial latency injection |
| **Cart Database** | Type (redis/spanner), connection string, in-cluster Redis, external TLS |

#### Using the YAML View

Switch to **YAML view** for full control. Key parameters to configure:

```yaml
# Image source (optional — defaults to Google's public registry)
images:
  repository: us-central1-docker.pkg.dev/google-samples/microservices-demo
  tag: ""  # defaults to chart appVersion

# Expose the frontend externally
frontend:
  externalService: true
  platform: local  # local | gcp | aws | azure | onprem | alibaba

# Cart database
cartDatabase:
  type: redis          # redis | spanner
  inClusterRedis:
    create: true

# Disable services you don't need
loadGenerator:
  create: false

# OpenShift-friendly security (non-root, no privilege escalation)
securityContext:
  enable: true
seccompProfile:
  enable: false        # set true if your cluster allows it
```

6. Set a **Release Name** (e.g., `onlineboutique`) and click **Install**.

---

## OpenShift-Specific Considerations

### Security Context Constraints (SCCs)

OpenShift's default `restricted` SCC prevents containers from running as root. This chart's `securityContext.enable: true` setting aligns with that constraint — all containers run as non-root with a read-only root filesystem and dropped Linux capabilities.

> **Do not set `runAsUser`** in values when deploying on OpenShift. OpenShift automatically assigns a UID from the project's allocated range. Setting an explicit UID will conflict with the `restricted` SCC.

If your cluster uses **restricted-v2** (OCP 4.11+), no additional SCC configuration is needed.

### Routes vs. LoadBalancer Services

OpenShift Routes are the preferred way to expose services externally. After installing the chart, create a Route for the frontend:

```bash
oc expose service frontend -n <your-namespace>
oc get route frontend -n <your-namespace>
```

Alternatively, keep `frontend.externalService: true` to create a `LoadBalancer` service if your cluster has a cloud load balancer provider.

### Image Pull Policies

The default image repository (`us-central1-docker.pkg.dev/google-samples/microservices-demo`) is publicly accessible. If deploying in an air-gapped environment, mirror the images to an internal registry and set:

```yaml
images:
  repository: registry.example.com/my-org/microservices-demo
  tag: "v0.10.4"
```

---

## Deployment Scenarios

### Scenario 1: Minimal Demo (no load generator)

```yaml
loadGenerator:
  create: false
securityContext:
  enable: true
frontend:
  externalService: true
  platform: local
```

### Scenario 2: With Istio / OpenShift Service Mesh

```yaml
networkPolicies:
  create: true
sidecars:
  create: true
authorizationPolicies:
  create: true
frontend:
  externalService: false
  virtualService:
    create: true
    hosts:
      - "boutique.apps.example.com"
    gateway:
      name: ingressgateway
      namespace: istio-system
      labelKey: istio
      labelValue: ingressgateway
```

### Scenario 3: External Cloud Spanner Cart Database

```yaml
cartDatabase:
  type: spanner
  connectionString: "projects/my-project/instances/onlineboutique/databases/carts"
  inClusterRedis:
    create: false
serviceAccounts:
  create: true
  annotationsOnlyForCartservice: true
  annotations:
    iam.gke.io/gcp-service-account: spanner-db-user@my-project.iam.gserviceaccount.com
```

### Scenario 4: Custom Internal Image Registry

```yaml
images:
  repository: quay.io/myorg/microservices-demo
  tag: "v0.10.4"
cartDatabase:
  inClusterRedis:
    publicRepository: false  # uses images.repository for Redis too
```

### Scenario 5: Google Cloud Observability Enabled

```yaml
googleCloudOperations:
  profiler: true
  tracing: true
  metrics: true
opentelemetryCollector:
  create: true
  projectId: "my-gcp-project"
```

---

## CLI Deployment

### Install with default values

```bash
helm upgrade onlineboutique ./online-boutique \
  --install \
  --namespace onlineboutique \
  --create-namespace
```

### Install with overrides

```bash
helm upgrade onlineboutique ./online-boutique \
  --install \
  --namespace onlineboutique \
  --create-namespace \
  --set frontend.externalService=true \
  --set frontend.platform=local \
  --set securityContext.enable=true \
  --set loadGenerator.create=false
```

### Install with a custom values file

```bash
helm upgrade onlineboutique ./online-boutique \
  --install \
  --namespace onlineboutique \
  --create-namespace \
  -f my-values.yaml
```

---

## Upgrading

### From the Web Console

1. Go to **Helm** in the left sidebar.
2. Find your release and click the three-dot menu → **Upgrade**.
3. Modify values in Form View or YAML View.
4. Click **Upgrade**.

### From the CLI

```bash
helm upgrade onlineboutique ./online-boutique \
  --namespace onlineboutique \
  --set images.tag="v0.10.5"
```

---

## Uninstalling

### From the Web Console

1. Go to **Helm** in the left sidebar.
2. Find your release and click the three-dot menu → **Uninstall Helm Release**.
3. Confirm deletion.

### From the CLI

```bash
helm uninstall onlineboutique --namespace onlineboutique
```

> This removes all chart resources. There are no PersistentVolumeClaims by default (Redis uses `emptyDir`), so no data cleanup is needed unless you configured external storage.

---

## Verifying the Deployment

### Check pod status

```bash
oc get pods -n onlineboutique
```

All pods should reach `Running` state within a few minutes.

### Check the frontend service

```bash
oc get svc frontend -n onlineboutique
oc get route frontend -n onlineboutique  # if you created a route
```

### View logs for a specific service

```bash
oc logs -f deployment/frontend -n onlineboutique
```

### Port-forward for local testing

```bash
oc port-forward service/frontend 8080:80 -n onlineboutique
```

Then open [http://localhost:8080](http://localhost:8080).

---

## Troubleshooting

### Pods stuck in `Pending`

Check for resource quota or node capacity issues:

```bash
oc describe pod <pod-name> -n onlineboutique
oc get events -n onlineboutique --sort-by='.lastTimestamp'
```

### Pods failing with `Permission denied` or SCC errors

Check SCC violations:

```bash
oc describe pod <pod-name> -n onlineboutique | grep -i scc
```

Ensure `securityContext.enable: true` so containers run as non-root. Do **not** set an explicit `runAsUser` value on OpenShift.

### `ImagePullBackOff`

Verify the image repository is reachable from the cluster:

```bash
oc get events -n onlineboutique | grep -i pull
```

For air-gapped clusters, mirror images and set `images.repository` accordingly.

### Cart service can't connect to Redis

Check the Redis pod is running and the connection string matches:

```bash
oc get pods -n onlineboutique | grep redis
oc logs deployment/cartservice -n onlineboutique
```

Default connection string: `redis-cart:6379`. If using an external Redis, verify `cartDatabase.connectionString` is correct.

### Frontend not accessible externally

If using `frontend.externalService: true`, check the service:

```bash
oc get svc frontend -n onlineboutique
```

If no external IP is assigned (no cloud LB), create an OpenShift Route instead:

```bash
oc expose svc frontend -n onlineboutique
oc get route frontend -n onlineboutique
```

---

## Additional Resources

- [Online Boutique – GitHub Repository](https://github.com/GoogleCloudPlatform/microservices-demo)
- [Online Boutique – Helm Chart Issue Tracker](https://github.com/GoogleCloudPlatform/microservices-demo/issues/1319)
- [OpenShift Documentation](https://docs.openshift.com/)
- [Helm Documentation](https://helm.sh/docs/)
- [Helm Chart Values Schema (JSON Schema draft-07)](https://helm.sh/docs/topics/charts/#schema-files)
