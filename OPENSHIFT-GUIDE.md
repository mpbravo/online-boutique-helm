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
    url: https://mpbravo.github.io/online-boutique-helm/
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
# Option A – OpenShift Route (recommended for on-prem/local OpenShift):
frontend:
  externalService: false   # disable the pending LoadBalancer service
  platform: local          # local | gcp | aws | azure | onprem | alibaba
  route:
    create: true
    host: ""               # auto-assigned: frontend-<ns>.apps.<domain>
    tls:
      enabled: true
      termination: edge
      insecureEdgeTerminationPolicy: Redirect
# Option B – LoadBalancer (cloud-based OpenShift: ROSA, ARO, etc.):
# frontend:
#   externalService: true
#   route:
#     create: false

# Cart database
cartDatabase:
  type: redis          # redis | spanner
  inClusterRedis:
    create: true

# Disable services you don't need
loadGenerator:
  create: false

# OpenShift-compatible security — REQUIRED on OpenShift
securityContext:
  enable: true
  openshift: true      # omits runAsUser/runAsGroup/fsGroup; lets OpenShift assign UID
seccompProfile:
  enable: false        # set true on OCP 4.11+ (restricted-v2 SCC)
```

6. Set a **Release Name** (e.g., `onlineboutique`) and click **Install**.

---

## OpenShift-Specific Considerations

### Security Context Constraints (SCCs)

OpenShift enforces SCCs on every pod. The two relevant SCCs for this chart are:

| SCC | OCP version | Enforces |
|---|---|---|
| `restricted` | 4.x | No root, no privilege escalation, assigned UID range |
| `restricted-v2` | 4.11+ | Same as above + requires `seccompProfile` |

#### Required: enable OpenShift mode

Set **both** flags to deploy cleanly on OpenShift:

```yaml
securityContext:
  enable: true      # applies runAsNonRoot, readOnlyRootFilesystem, dropped caps
  openshift: true   # omits runAsUser / runAsGroup / fsGroup
```

Without `securityContext.openshift: true`, every pod will fail admission with an error similar to:

```
unable to validate against any security context constraint:
  [spec.containers[0].securityContext.runAsUser: Invalid value: 1000:
   must be in the ranges: [1000650000, 1000659999]]
```

**Why?** OpenShift's `restricted` SCC allocates a UID range per namespace (e.g. `1000650000–1000659999`). Specifying `runAsUser: 1000` falls outside that range and is rejected. With `openshift: true`, only `runAsNonRoot: true` is set — OpenShift then automatically assigns a UID from the project's allowed range.

#### restricted-v2 (OCP 4.11+)

If your cluster uses `restricted-v2`, also enable the seccomp profile:

```yaml
securityContext:
  enable: true
  openshift: true
seccompProfile:
  enable: true
  type: RuntimeDefault
```

#### Granting a less-restrictive SCC (not recommended)

If you cannot use the recommended settings, you can bind a more permissive SCC to the service accounts — but this weakens cluster security:

```bash
# Example: grant anyuid to all service accounts in the namespace (NOT recommended)
oc adm policy add-scc-to-group anyuid system:serviceaccounts:<namespace>
```

### Routes vs. LoadBalancer Services

On OpenShift, the standard way to expose services externally is via an **OpenShift Route** (`route.openshift.io/v1`). A `LoadBalancer` service only works when a cloud load balancer provisioner is available (ROSA on AWS, ARO on Azure, etc.). On bare-metal, local, or on-prem clusters it stays `<pending>` indefinitely.

#### Recommended: use the built-in Route (on-prem / local OpenShift)

Disable the LoadBalancer service and enable the Route instead:

```yaml
frontend:
  externalService: false   # disable the pending LoadBalancer service
  route:
    create: true           # create an OpenShift Route
    host: ""               # leave empty to auto-assign: frontend-<ns>.apps.<domain>
    tls:
      enabled: true        # terminate TLS at the router (HTTPS)
      termination: edge
      insecureEdgeTerminationPolicy: Redirect
```

Or via CLI:

```bash
helm upgrade onlineboutique ./online-boutique \
  --set frontend.externalService=false \
  --set frontend.route.create=true \
  --set frontend.route.tls.enabled=true \
  -n onlineboutique
```

After deploying, get the assigned URL:

```bash
oc get route frontend -n onlineboutique
```

#### Cloud-based OpenShift (ROSA, ARO): keep LoadBalancer

If your cluster has a cloud load balancer provisioner, keep `frontend.externalService: true` and leave `frontend.route.create: false`.

### Image Pull Policies

The default image repository (`us-central1-docker.pkg.dev/google-samples/microservices-demo`) is publicly accessible. If deploying in an air-gapped environment, mirror the images to an internal registry and set:

```yaml
images:
  repository: registry.example.com/my-org/microservices-demo
  tag: "v0.10.4"
```

---

## Deployment Scenarios

### Scenario 1: Minimal Demo (on-prem OpenShift, no load generator)

```yaml
loadGenerator:
  create: false
securityContext:
  enable: true
  openshift: true
frontend:
  externalService: false
  platform: local
  route:
    create: true
    tls:
      enabled: true
      termination: edge
      insecureEdgeTerminationPolicy: Redirect
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

### Install with overrides (on-prem OpenShift with Route)

```bash
helm upgrade onlineboutique ./online-boutique \
  --install \
  --namespace onlineboutique \
  --create-namespace \
  --set frontend.externalService=false \
  --set frontend.route.create=true \
  --set frontend.route.tls.enabled=true \
  --set frontend.platform=local \
  --set securityContext.enable=true \
  --set securityContext.openshift=true \
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

### Check the frontend service and route

```bash
oc get svc frontend -n onlineboutique
oc get route frontend -n onlineboutique
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

### Frontend service stuck in `<pending>` / not accessible externally

A `LoadBalancer` service stays `<pending>` on clusters without a cloud load balancer provisioner (bare-metal, local, on-prem OpenShift).

**Fix:** disable the LoadBalancer service and use an OpenShift Route instead:

```bash
helm upgrade onlineboutique ./online-boutique \
  --set frontend.externalService=false \
  --set frontend.route.create=true \
  --set frontend.route.tls.enabled=true \
  -n onlineboutique

oc get route frontend -n onlineboutique
```

If you prefer a one-off manual route without upgrading the chart:

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
