<!--- app-name: app-base -->

# app-base

`app-base` is a generic Helm chart for deploying **backend applications** (REST APIs, gRPC services, workers, and microservices) on Kubernetes. It includes first-class support for KEDA-based autoscaling, GKE-native resources, and an optional Cloud SQL Auth Proxy sidecar.

[Overview of app-base](https://github.com/your-org/charts)

## TL;DR

```console
helm repo add your-org https://your-org.github.io/charts
helm install my-release your-org/app-base \
  --set name=my-api \
  --set namespace=my-namespace \
  --set deployment.image.repository=my-registry/my-api \
  --set deployment.image.tag=1.0.0
```

## Introduction

This chart bootstraps an **app-base** deployment on a [Kubernetes](https://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.

It provisions the following Kubernetes resources:

- `Deployment` — with configurable rolling update strategy, probes, and optional sidecar
- `Service` — ClusterIP by default
- `Ingress` — NGINX Ingress Controller (external and/or internal)
- `HTTPRoute` — GKE Gateway API route (external and/or internal)
- `GCPBackendPolicy` — GKE-native backend timeout and logging policy
- `HealthCheckPolicy` — GKE-native health check for load balancers
- `ScaledObject` — KEDA autoscaler using CPU and Memory triggers
- `PodDisruptionBudget` — optional, for high-availability workloads

## Prerequisites

- Kubernetes 1.25+
- Helm 3.10+
- [KEDA](https://keda.sh) 2.10+ *(required when `scaledObject.enabled=true`)*
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/) *(required when `ingress.*.enabled=true`)*
- [GKE Gateway API CRDs](https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api) *(required when using `ingress.*.sectionNames`)*
- [Stakater Reloader](https://github.com/stakater/Reloader) *(optional, enables automatic pod restart on Secret/ConfigMap changes)*

## Installing the Chart

To install the chart with the release name `my-release`:

```console
helm repo add your-org https://your-org.github.io/charts
helm install my-release your-org/app-base
```

> **Tip**: List all releases using `helm list`

## Uninstalling the Chart

To uninstall/delete the `my-release` deployment:

```console
helm uninstall my-release
```

The command removes all the Kubernetes components associated with the chart and deletes the release.

## Configuration and installation details

### Resource requests and limits

Setting resource requests and limits is strongly recommended for production workloads. Configure them under `deployment.resources`:

```yaml
deployment:
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi
```

### Secret injection via environment variables

By default, the chart injects all keys from a Kubernetes Secret named `config` as environment variables using `envFrom.secretRef`. This secret must exist in the same namespace before deploying:

```console
kubectl create secret generic config \
  --from-literal=DATABASE_URL="postgres://..." \
  --from-literal=API_KEY="..." \
  -n my-namespace
```

To inject additional sources (Secrets or ConfigMaps), use `deployment.extraEnvFrom`:

```yaml
deployment:
  extraEnvFrom:
    - configMapRef:
        name: my-feature-flags
    - secretRef:
        name: my-third-party-keys
```

### Volume mounts for configuration files

Some backend applications require file-based configuration (e.g., `appsettings.json`). Enable the built-in volume mounts:

```yaml
deployment:
  volumeMounts:
    appsettings:
      enabled: true
      secretName: my-app-appsettings   # must exist in the namespace
      mountPath: /app/appsettings.json
      subPath: appsettings.json
    apiKey:
      enabled: true
      secretName: my-api-key
      mountPath: /app/apis-key.json
      subPath: apis-key.json
```

### Cloud SQL Auth Proxy sidecar

For applications connecting to [Google Cloud SQL](https://cloud.google.com/sql), the chart can inject a [Cloud SQL Auth Proxy](https://cloud.google.com/sql/docs/mysql/connect-kubernetes-engine) sidecar container:

```yaml
deployment:
  cloudsqlProxy:
    enabled: true
    instances: "my-project:us-central1:my-db=tcp:5432"
    credentialsSecretName: cloud-sql-sa-key  # Kubernetes Secret with JSON key
```

The sidecar runs as a non-root user and mounts the credentials secret automatically when `credentialsSecretName` is set.

### Extra volumes and volume mounts

For arbitrary file mounts beyond the built-in options, use `extraVolumes` and `extraVolumeMounts`:

```yaml
deployment:
  extraVolumes:
    - name: custom-certs
      secret:
        secretName: custom-ca-bundle
  extraVolumeMounts:
    - name: custom-certs
      mountPath: /etc/ssl/certs/ca-bundle.crt
      subPath: ca-bundle.crt
      readOnly: true
```

### Configuring KEDA autoscaling

KEDA ScaledObject is enabled by default with CPU and Memory triggers. Tune the thresholds and replica counts:

```yaml
scaledObject:
  enabled: true
  minReplicaCount: 2
  maxReplicaCount: 20
  triggers:
    cpu:
      value: 70    # scale out when CPU > 70%
    memory:
      value: 80    # scale out when Memory > 80%
```

To disable KEDA and use a static replica count:

```yaml
scaledObject:
  enabled: false
deployment:
  replicas: 3
```

### Configuring Ingress

The chart supports both NGINX Ingress Controller and GKE Gateway API HTTPRoute simultaneously:

```yaml
ingress:
  external:
    enabled: true
    host: api.example.com
    ingressClassName: nginx
    tls:
      secretCertificate: api-tls-secret
  internal:
    enabled: true
    host: api.internal.example.com
    sectionNames:
      - https-443    # Gateway API listener section name
```

### Node scheduling

To pin pods to a specific node pool:

```yaml
deployment:
  nodeSelector:
    nodeTypePoolSpec: "backend-pool"
  tolerations:
    - key: "nodepool"
      operator: "Equal"
      value: "backend"
      effect: "NoSchedule"
```

## Parameters

### Global parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `name` | Application name. Used as prefix for all Kubernetes resources. **Required.** | `""` |
| `namespace` | Kubernetes namespace to deploy into. **Required.** | `""` |
| `timezone` | Timezone injected as `TZ` environment variable into the pod. | `"America/Sao_Paulo"` |

### Deployment parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `deployment.revisionHistoryLimit` | Number of old ReplicaSets to retain for rollback. | `1` |
| `deployment.progressDeadlineSeconds` | Seconds before a stalled deployment is marked as failed. | `3600` |
| `deployment.strategy.type` | Deployment strategy: `RollingUpdate` or `Recreate`. | `RollingUpdate` |
| `deployment.strategy.rollingUpdate.maxSurge` | Maximum number of pods created above desired during update. | `1` |
| `deployment.strategy.rollingUpdate.maxUnavailable` | Maximum number of pods that can be unavailable during update. | `1` |
| `deployment.image.repository` | Container image repository. **Required.** | `""` |
| `deployment.image.tag` | Container image tag. **Required.** | `""` |
| `deployment.image.pullPolicy` | Image pull policy. Allowed values: `Always`, `IfNotPresent`, `Never`. | `IfNotPresent` |
| `deployment.resources` | Resource requests and limits for the main container. | `{}` |
| `deployment.ports.http.containerPort` | HTTP port that the container listens on. | `80` |
| `deployment.terminationGracePeriodSeconds` | Seconds to wait for graceful termination. | `120` |
| `deployment.nodeSelector.nodeTypePoolSpec` | Node pool label value for pod scheduling. Leave empty to skip. | `""` |
| `deployment.tolerations` | List of pod scheduling tolerations. | `[]` |
| `deployment.extraEnv` | Extra environment variables to inject into the main container. | `[]` |
| `deployment.extraEnvFrom` | Additional `envFrom` sources (Secrets, ConfigMaps). | `[]` |
| `deployment.extraVolumeMounts` | Extra volume mounts for the main container. | `[]` |
| `deployment.extraVolumes` | Extra volumes to add to the pod spec. | `[]` |

### Probe parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `deployment.probes.readiness.path` | HTTP path for the readiness probe. | `/health/ready` |
| `deployment.probes.readiness.port` | Port for the readiness probe. | `80` |
| `deployment.probes.readiness.failureThreshold` | Number of consecutive failures before the probe is considered failed. | `4` |
| `deployment.probes.readiness.periodSeconds` | Frequency (in seconds) at which the probe is performed. | `15` |
| `deployment.probes.readiness.successThreshold` | Minimum consecutive successes for the probe to be considered successful. | `1` |
| `deployment.probes.readiness.timeoutSeconds` | Seconds after which the probe times out. | `10` |
| `deployment.probes.startup.path` | HTTP path for the startup probe. | `/health/live` |
| `deployment.probes.startup.port` | Port for the startup probe. | `80` |
| `deployment.probes.startup.failureThreshold` | Number of consecutive failures before the startup probe is considered failed. | `8` |
| `deployment.probes.startup.periodSeconds` | Frequency (in seconds) at which the startup probe is performed. | `10` |
| `deployment.probes.startup.successThreshold` | Minimum consecutive successes for the startup probe. | `1` |
| `deployment.probes.liveness.path` | HTTP path for the liveness probe. | `/health/live` |
| `deployment.probes.liveness.port` | Port for the liveness probe. | `80` |
| `deployment.probes.liveness.failureThreshold` | Number of consecutive failures before the liveness probe is considered failed. | `4` |
| `deployment.probes.liveness.periodSeconds` | Frequency (in seconds) at which the liveness probe is performed. | `15` |
| `deployment.probes.liveness.successThreshold` | Minimum consecutive successes for the liveness probe. | `1` |
| `deployment.probes.liveness.timeoutSeconds` | Seconds after which the liveness probe times out. | `10` |

### Volume mount parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `deployment.volumeMounts.appsettings.enabled` | Mount an `appsettings.json` file from a Kubernetes Secret. | `false` |
| `deployment.volumeMounts.appsettings.secretName` | Name of the Secret to mount. Required when enabled. | `""` |
| `deployment.volumeMounts.appsettings.mountPath` | Mount path inside the container. | `/app/appsettings.json` |
| `deployment.volumeMounts.appsettings.subPath` | Key (subPath) within the Secret. | `appsettings.json` |
| `deployment.volumeMounts.apiKey.enabled` | Mount an API key file from a Kubernetes Secret. | `false` |
| `deployment.volumeMounts.apiKey.secretName` | Name of the Secret to mount. Required when enabled. | `""` |
| `deployment.volumeMounts.apiKey.mountPath` | Mount path inside the container. | `/app/apis-key.json` |
| `deployment.volumeMounts.apiKey.subPath` | Key (subPath) within the Secret. | `apis-key.json` |

### Cloud SQL Auth Proxy parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `deployment.cloudsqlProxy.enabled` | Enable the Cloud SQL Auth Proxy sidecar container. | `false` |
| `deployment.cloudsqlProxy.image.repository` | Cloud SQL Proxy image repository. | `gcr.io/cloud-sql-connectors/cloud-sql-proxy` |
| `deployment.cloudsqlProxy.image.tag` | Cloud SQL Proxy image tag. | `2.11.4` |
| `deployment.cloudsqlProxy.instances` | Cloud SQL instance connection name (e.g., `project:region:instance=tcp:5432`). Required when enabled. | `""` |
| `deployment.cloudsqlProxy.credentialsSecretName` | Kubernetes Secret containing the GCP service account JSON key file. | `""` |
| `deployment.cloudsqlProxy.resources` | Resource requests and limits for the proxy sidecar. | `{requests: {cpu: 10m, memory: 32Mi}}` |

### Service parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `service.type` | Kubernetes Service type. | `ClusterIP` |
| `service.annotations` | Annotations to add to the Service resource. | `{}` |
| `service.ports.port` | Port exposed by the Service. | `80` |
| `service.ports.targetPort` | Target port on the pod. | `80` |

### Ingress parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `ingress.external.enabled` | Enable external NGINX Ingress and GKE Gateway HTTPRoute. | `false` |
| `ingress.external.host` | Primary external hostname. | `""` |
| `ingress.external.hosts` | Additional external hostnames (overrides `host` when set). | `[]` |
| `ingress.external.sectionNames` | GKE Gateway API listener section names for the external HTTPRoute. | `[]` |
| `ingress.external.ingressClassName` | Ingress class name. | `nginx` |
| `ingress.external.tls.secretCertificate` | TLS Secret name used when `tls.configs` is not set. | `""` |
| `ingress.external.tls.configs` | Advanced TLS config list. Each entry: `{hosts: [], secretName: ""}`. | `[]` |
| `ingress.external.annotations` | Annotations added to the external Ingress resource. | *(nginx proxy settings)* |
| `ingress.internal.enabled` | Enable internal NGINX Ingress and GKE Gateway HTTPRoute. | `false` |
| `ingress.internal.host` | Primary internal hostname. | `""` |
| `ingress.internal.hosts` | Additional internal hostnames. | `[]` |
| `ingress.internal.sectionNames` | GKE Gateway API listener section names. **Required when `internal.enabled=true`.** | `[]` |
| `ingress.internal.ingressClassName` | Ingress class name. | `nginx` |
| `ingress.internal.tls.secretCertificate` | TLS Secret name. | `""` |
| `ingress.internal.tls.configs` | Advanced TLS config list. | `[]` |
| `ingress.internal.annotations` | Annotations added to the internal Ingress resource. | *(nginx proxy settings)* |

### GCP parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `healthCheck.requestPath` | HTTP path used by the GCP `HealthCheckPolicy` for load balancer health checks. | `/health/ready` |
| `gcpBackendPolicy.timeoutSec` | Backend timeout in seconds for the `GCPBackendPolicy`. | `360` |
| `gcpBackendPolicy.logging.enabled` | Enable request logging on the GCP backend. | `false` |
| `gcpBackendPolicy.logging.sampleRate` | Log sampling rate (0–1000000). | `500000` |

### PodDisruptionBudget parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `pdb.enabled` | Create a `PodDisruptionBudget` resource for the deployment. | `false` |
| `pdb.minAvailable` | Minimum number of available pods (integer or percentage string). Required when enabled. | `""` |

### KEDA autoscaling parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `scaledObject.enabled` | Enable KEDA `ScaledObject`. When `true`, the Deployment `replicas` field is managed by KEDA. | `true` |
| `scaledObject.minReplicaCount` | Minimum number of replicas. | `1` |
| `scaledObject.maxReplicaCount` | Maximum number of replicas. | `10` |
| `scaledObject.triggers.cpu.value` | CPU utilization percentage that triggers a scale-out event. | `80` |
| `scaledObject.triggers.memory.value` | Memory utilization percentage that triggers a scale-out event. | `90` |
| `scaledObject.advanced.horizontalPodAutoscalerConfig.behavior.scaleDown.stabilizationWindowSeconds` | Seconds to wait before allowing a scale-down event. | `600` |
| `scaledObject.advanced.horizontalPodAutoscalerConfig.behavior.scaleDown.policies` | List of scale-down policies. | `[{type: Pods, value: 1, periodSeconds: 120}]` |
| `scaledObject.advanced.horizontalPodAutoscalerConfig.behavior.scaleUp.stabilizationWindowSeconds` | Seconds to wait before allowing a scale-up event. | `0` |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example:

```console
helm install my-release \
    --set deployment.image.repository=my-registry/my-api \
    --set deployment.image.tag=1.2.3 \
    your-org/app-base
```

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example:

```console
helm install my-release -f my-values.yaml your-org/app-base
```

> **Tip**: You can use the default [values.yaml](./values.yaml) as a starting point.

## Upgrading

### To 1.0.0

Initial public release. No breaking changes from prior internal versions.

## License

Copyright &copy; 2024 your-org

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

<http://www.apache.org/licenses/LICENSE-2.0>

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
