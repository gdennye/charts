<!--- app-name: app-front -->

# app-front

`app-front` is a generic Helm chart for deploying **frontend Single Page Applications (SPA)** on Kubernetes where the web server is **bundled inside the Docker image** (e.g., an image built with `nginx:alpine` as base). The chart does not manage the web server configuration — it simply deploys the image, exposes it, and wires up autoscaling and routing.

[Overview of app-front](https://github.com/gdennye/charts)

> **Looking for chart-managed NGINX config?** Use [`app-front-nginx`](../app-front-nginx/) instead.

## TL;DR

```console
helm repo add gdennye https://gdennye.github.io/charts
helm install my-release gdennye/app-front \
  --set name=my-spa \
  --set namespace=my-namespace \
  --set deployment.image.repository=my-registry/my-spa \
  --set deployment.image.tag=1.0.0
```

## Introduction

This chart bootstraps an **app-front** deployment on a [Kubernetes](https://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.

It provisions the following Kubernetes resources:

- `Deployment` — with configurable rolling update strategy and health probes
- `Service` — ClusterIP by default, with optional annotations
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
- [Stakater Reloader](https://github.com/stakater/Reloader) *(optional)*

## Installing the Chart

To install the chart with the release name `my-release`:

```console
helm repo add gdennye https://gdennye.github.io/charts
helm install my-release gdennye/app-front
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
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 128Mi
```

### Health probes

By default, all three probes (readiness, startup, and liveness) point to `/healthcheck` on port `80`. This matches the default path served by most SPA-ready NGINX images. Override them if your image uses a different endpoint:

```yaml
deployment:
  probes:
    readiness:
      path: /ping
      port: 8080
    startup:
      path: /ping
      port: 8080
    liveness:
      path: /ping
      port: 8080
```

### Extra environment variables

To pass runtime configuration to the container, use `deployment.extraEnv` or `deployment.extraEnvFrom`:

```yaml
deployment:
  extraEnv:
    - name: API_BASE_URL
      value: "https://api.example.com"
    - name: FEATURE_FLAG_X
      valueFrom:
        configMapKeyRef:
          name: feature-flags
          key: flag-x

  extraEnvFrom:
    - secretRef:
        name: my-spa-runtime-secrets
```

### Extra volumes and volume mounts

For injecting files into the container (e.g., a runtime config file):

```yaml
deployment:
  extraVolumes:
    - name: runtime-config
      configMap:
        name: spa-runtime-config
  extraVolumeMounts:
    - name: runtime-config
      mountPath: /usr/share/nginx/html/config.json
      subPath: config.json
      readOnly: true
```

### Configuring KEDA autoscaling

KEDA ScaledObject is enabled by default with CPU and Memory triggers:

```yaml
scaledObject:
  enabled: true
  minReplicaCount: 2
  maxReplicaCount: 10
  triggers:
    cpu:
      value: 70
    memory:
      value: 80
```

To disable KEDA and use a static replica count:

```yaml
scaledObject:
  enabled: false
deployment:
  replicas: 3
```

### Configuring Ingress

The chart supports both NGINX Ingress Controller and GKE Gateway API HTTPRoute:

```yaml
ingress:
  external:
    enabled: true
    host: app.example.com
    ingressClassName: nginx
    tls:
      secretCertificate: app-tls-secret
  internal:
    enabled: true
    host: app.internal.example.com
    sectionNames:
      - https-443
```

### Node scheduling

```yaml
deployment:
  nodeSelector:
    nodeTypePoolSpec: "frontend-pool"
  tolerations:
    - key: "nodepool"
      operator: "Equal"
      value: "frontend"
      effect: "NoSchedule"
```

## Parameters

### Global parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `name` | Application name. Used as prefix for all Kubernetes resources. **Required.** | `""` |
| `namespace` | Kubernetes namespace to deploy into. **Required.** | `""` |
| `timezone` | Timezone injected as `TZ` environment variable. | `"America/Sao_Paulo"` |

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
| `deployment.extraEnv` | Extra environment variables to inject into the container. | `[]` |
| `deployment.extraEnvFrom` | Additional `envFrom` sources (Secrets, ConfigMaps). | `[]` |
| `deployment.extraVolumeMounts` | Extra volume mounts for the main container. | `[]` |
| `deployment.extraVolumes` | Extra volumes to add to the pod spec. | `[]` |

### Probe parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `deployment.probes.readiness.path` | HTTP path for the readiness probe. | `/healthcheck` |
| `deployment.probes.readiness.port` | Port for the readiness probe. | `80` |
| `deployment.probes.readiness.failureThreshold` | Consecutive failures before the probe is considered failed. | `4` |
| `deployment.probes.readiness.periodSeconds` | Frequency (seconds) at which the probe is performed. | `15` |
| `deployment.probes.readiness.successThreshold` | Consecutive successes for the probe to be considered successful. | `1` |
| `deployment.probes.readiness.timeoutSeconds` | Seconds after which the probe times out. | `10` |
| `deployment.probes.startup.path` | HTTP path for the startup probe. | `/healthcheck` |
| `deployment.probes.startup.port` | Port for the startup probe. | `80` |
| `deployment.probes.startup.failureThreshold` | Consecutive failures before the startup probe is considered failed. | `8` |
| `deployment.probes.startup.periodSeconds` | Frequency (seconds) at which the startup probe is performed. | `10` |
| `deployment.probes.startup.successThreshold` | Consecutive successes for the startup probe. | `1` |
| `deployment.probes.liveness.path` | HTTP path for the liveness probe. | `/healthcheck` |
| `deployment.probes.liveness.port` | Port for the liveness probe. | `80` |
| `deployment.probes.liveness.failureThreshold` | Consecutive failures before the liveness probe is considered failed. | `4` |
| `deployment.probes.liveness.periodSeconds` | Frequency (seconds) at which the liveness probe is performed. | `15` |
| `deployment.probes.liveness.successThreshold` | Consecutive successes for the liveness probe. | `1` |
| `deployment.probes.liveness.timeoutSeconds` | Seconds after which the liveness probe times out. | `10` |

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
| `ingress.internal.enabled` | Enable internal NGINX Ingress and GKE Gateway HTTPRoute. | `false` |
| `ingress.internal.host` | Primary internal hostname. | `""` |
| `ingress.internal.hosts` | Additional internal hostnames. | `[]` |
| `ingress.internal.sectionNames` | GKE Gateway API listener section names. **Required when `internal.enabled=true`.** | `[]` |
| `ingress.internal.ingressClassName` | Ingress class name. | `nginx` |
| `ingress.internal.tls.secretCertificate` | TLS Secret name. | `""` |
| `ingress.internal.tls.configs` | Advanced TLS config list. | `[]` |

### GCP parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `healthCheck.requestPath` | HTTP path used by the GCP `HealthCheckPolicy`. | `/healthcheck` |
| `gcpBackendPolicy.timeoutSec` | Backend timeout in seconds for the `GCPBackendPolicy`. | `360` |
| `gcpBackendPolicy.logging.enabled` | Enable request logging on the GCP backend. | `false` |
| `gcpBackendPolicy.logging.sampleRate` | Log sampling rate (0–1000000). | `500000` |

### PodDisruptionBudget parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `pdb.enabled` | Create a `PodDisruptionBudget` resource for the deployment. | `false` |
| `pdb.minAvailable` | Minimum available pods (integer or percentage string). Required when enabled. | `""` |

### KEDA autoscaling parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `scaledObject.enabled` | Enable KEDA `ScaledObject`. When `true`, replicas are managed by KEDA. | `true` |
| `scaledObject.minReplicaCount` | Minimum number of replicas. | `1` |
| `scaledObject.maxReplicaCount` | Maximum number of replicas. | `10` |
| `scaledObject.triggers.cpu.value` | CPU utilization (%) that triggers a scale-out event. | `80` |
| `scaledObject.triggers.memory.value` | Memory utilization (%) that triggers a scale-out event. | `90` |
| `scaledObject.advanced.horizontalPodAutoscalerConfig.behavior.scaleDown.stabilizationWindowSeconds` | Seconds to wait before allowing a scale-down event. | `600` |
| `scaledObject.advanced.horizontalPodAutoscalerConfig.behavior.scaleDown.policies` | List of scale-down policies. | `[{type: Pods, value: 1, periodSeconds: 120}]` |
| `scaledObject.advanced.horizontalPodAutoscalerConfig.behavior.scaleUp.stabilizationWindowSeconds` | Seconds to wait before allowing a scale-up event. | `0` |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`. For example:

```console
helm install my-release \
    --set deployment.image.repository=my-registry/my-spa \
    --set deployment.image.tag=2.1.0 \
    gdennye/app-front
```

Alternatively, provide a YAML file with the values:

```console
helm install my-release -f my-values.yaml gdennye/app-front
```

> **Tip**: You can use the default [values.yaml](./values.yaml) as a starting point.

## Upgrading

### To 1.0.0

Initial public release. Fixes a bug from prior internal versions where `readinessProbe`, `startupProbe`, and `livenessProbe` were defined in `values.yaml` but not rendered in the Deployment template.

## License

Copyright &copy; 2024 gdennye

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

<http://www.apache.org/licenses/LICENSE-2.0>

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
