<!--- app-name: app-front-nginx -->

# app-front-nginx

`app-front-nginx` is a generic Helm chart for deploying **frontend Single Page Applications (SPA)** on Kubernetes where **NGINX is configured by the chart itself** via a dynamically generated ConfigMap. You do not need to bake an `nginx.conf` into your Docker image.

[Overview of app-front-nginx](https://github.com/your-org/charts)

The chart operates in one of two modes, controlled by `ingress.external.enabled`:

| Mode | `ingress.external.enabled` | NGINX listens on | Use case |
| ---- | -------------------------- | ---------------- | -------- |
| **HTTP mode** | `true` | Port `80` | External Ingress or Gateway handles TLS termination |
| **HTTPS/SSL mode** | `false` | Port `443` | TLS terminated at the pod; two TLS Secrets required in cluster |

> **Don't need chart-managed NGINX?** Use [`app-front`](../app-front/) instead.

## TL;DR

```console
helm repo add your-org https://your-org.github.io/charts
helm install my-release your-org/app-front-nginx \
  --set name=my-spa \
  --set namespace=my-namespace \
  --set deployment.image.repository=nginx \
  --set deployment.image.tag=1.27-alpine \
  --set ingress.external.host=app.example.com
```

## Introduction

This chart bootstraps an **app-front-nginx** deployment on a [Kubernetes](https://kubernetes.io) cluster using the [Helm](https://helm.sh) package manager.

It provisions the following Kubernetes resources:

- `ConfigMap` — NGINX `default.conf` generated from chart values (HTTP or HTTPS mode)
- `Deployment` — mounts the ConfigMap and, in HTTPS mode, two TLS certificate Secrets
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
- [Stakater Reloader](https://github.com/stakater/Reloader) *(optional)*
- `certificate-ssl` TLS Secret *(required in **HTTPS/SSL mode** only)*
- `certificate-ssl-default` TLS Secret *(required in **HTTPS/SSL mode** only)*

## Installing the Chart

To install the chart with the release name `my-release`:

```console
helm repo add your-org https://your-org.github.io/charts
helm install my-release your-org/app-front-nginx
```

> **Tip**: List all releases using `helm list`

## Uninstalling the Chart

To uninstall/delete the `my-release` deployment:

```console
helm uninstall my-release
```

The command removes all Kubernetes components associated with the chart and deletes the release.

## Configuration and installation details

### HTTP mode vs HTTPS/SSL mode

The mode is determined by `ingress.external.enabled`:

**HTTP mode** (`ingress.external.enabled: true`) — simplest setup. The external Ingress or Gateway terminates TLS, and NGINX only serves on port 80 with CORS headers restricted to the configured `ingress.external.host`:

```yaml
ingress:
  external:
    enabled: true          # HTTP mode
    host: app.example.com
    tls:
      secretCertificate: app-tls-secret
```

**HTTPS/SSL mode** (`ingress.external.enabled: false`) — NGINX terminates TLS directly inside the pod on port 443. This is suitable for internal-only deployments where no external Ingress or Gateway is handling TLS. You must create two TLS Secrets in the namespace before deploying:

```console
kubectl create secret tls certificate-ssl \
  --cert=wildcard.crt --key=wildcard.key \
  -n my-namespace

kubectl create secret tls certificate-ssl-default \
  --cert=default.crt --key=default.key \
  -n my-namespace
```

```yaml
ingress:
  external:
    enabled: false         # HTTPS/SSL mode
    host: app.internal.example.com
  internal:
    enabled: true
    host: app.internal.example.com
    sectionNames:
      - https-443
```

In HTTPS/SSL mode, NGINX automatically adds an HTTP→HTTPS redirect on port 80.

### Generated NGINX configuration

The chart generates the `nginx.conf` for you. To further customize NGINX behavior, use `deployment.nginx.extraConfig` to inject additional directives inside the `server` block:

```yaml
deployment:
  nginx:
    clientMaxBodySize: "50M"
    gzip: true
    extraConfig: |
      add_header X-Frame-Options SAMEORIGIN;
      add_header X-Content-Type-Options nosniff;
```

### Resource requests and limits

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

### Configuring KEDA autoscaling

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
| `deployment.strategy.rollingUpdate.maxUnavailable` | Maximum pods that can be unavailable during update. | `1` |
| `deployment.image.repository` | Container image repository (must include NGINX). **Required.** | `""` |
| `deployment.image.tag` | Container image tag. **Required.** | `""` |
| `deployment.image.pullPolicy` | Image pull policy. Allowed values: `Always`, `IfNotPresent`, `Never`. | `IfNotPresent` |
| `deployment.resources` | Resource requests and limits for the main container. | `{}` |
| `deployment.ports.http.containerPort` | HTTP port that the container listens on. | `80` |
| `deployment.terminationGracePeriodSeconds` | Seconds to wait for graceful termination. | `120` |
| `deployment.nodeSelector.nodeTypePoolSpec` | Node pool label value for pod scheduling. Leave empty to skip. | `""` |
| `deployment.tolerations` | List of pod scheduling tolerations. | `[]` |
| `deployment.extraVolumeMounts` | Extra volume mounts for the main container. | `[]` |
| `deployment.extraVolumes` | Extra volumes to add to the pod spec. | `[]` |

### NGINX configuration parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `deployment.nginx.workerProcesses` | NGINX `worker_processes` directive. | `auto` |
| `deployment.nginx.clientMaxBodySize` | Maximum allowed client request body size. | `100M` |
| `deployment.nginx.gzip` | Enable gzip compression in the NGINX server block. | `false` |
| `deployment.nginx.extraConfig` | Extra NGINX directives appended inside the `server` block. | `""` |

### SSL volume mount parameters

Only applicable in **HTTPS/SSL mode** (`ingress.external.enabled: false`).

| Name | Description | Value |
| ---- | ----------- | ----- |
| `deployment.volumeMounts.ssl.mountPath` | Mount path for the `certificate-ssl` TLS Secret. | `/etc/nginx/ssl` |
| `deployment.volumeMounts.ssl-default.mountPath` | Mount path for the `certificate-ssl-default` TLS Secret. | `/etc/nginx/ssl-default` |

### Probe parameters

| Name | Description | Value |
| ---- | ----------- | ----- |
| `deployment.probes.readiness.path` | HTTP path for the readiness probe. | `/healthcheck` |
| `deployment.probes.readiness.port` | Port for the readiness probe. | `80` |
| `deployment.probes.readiness.failureThreshold` | Consecutive failures before the probe is considered failed. | `4` |
| `deployment.probes.readiness.periodSeconds` | Frequency (seconds) at which the probe is performed. | `15` |
| `deployment.probes.readiness.timeoutSeconds` | Seconds after which the probe times out. | `10` |
| `deployment.probes.startup.path` | HTTP path for the startup probe. | `/healthcheck` |
| `deployment.probes.startup.failureThreshold` | Consecutive failures before the startup probe fails. | `8` |
| `deployment.probes.startup.periodSeconds` | Frequency (seconds) at which the startup probe is performed. | `10` |
| `deployment.probes.liveness.path` | HTTP path for the liveness probe. | `/healthcheck` |
| `deployment.probes.liveness.failureThreshold` | Consecutive failures before the liveness probe fails. | `4` |
| `deployment.probes.liveness.periodSeconds` | Frequency (seconds) at which the liveness probe is performed. | `15` |
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
| `ingress.external.enabled` | `true` = HTTP mode; `false` = HTTPS/SSL mode. Also controls which NGINX config is generated. | `true` |
| `ingress.external.host` | Primary hostname used for CORS headers and NGINX `server_name`. **Required.** | `""` |
| `ingress.external.hosts` | Additional hostnames (overrides `host` when set). | `[]` |
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
    --set deployment.image.repository=nginx \
    --set deployment.image.tag=1.27-alpine \
    --set ingress.external.host=app.example.com \
    your-org/app-front-nginx
```

Alternatively, provide a YAML file:

```console
helm install my-release -f my-values.yaml your-org/app-front-nginx
```

> **Tip**: You can use the default [values.yaml](./values.yaml) as a starting point.

## Upgrading

### To 1.0.0

Initial public release. Tolerations are now a configurable list (`deployment.tolerations: []`) instead of being hard-coded in the Deployment template.

## License

Copyright &copy; 2024 your-org

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

<http://www.apache.org/licenses/LICENSE-2.0>

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
