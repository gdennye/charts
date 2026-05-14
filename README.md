<!-- markdownlint-disable MD041 -->
<p align="center">
    <img width="320px" height="auto" src="https://raw.githubusercontent.com/your-org/charts/main/assets/logo.png" />
</p>

<p align="center">
    <a href="https://github.com/your-org/charts"><img src="https://badgen.net/github/stars/your-org/charts?icon=github" /></a>
    <a href="https://github.com/your-org/charts"><img src="https://badgen.net/github/forks/your-org/charts?icon=github" /></a>
    <a href="https://github.com/your-org/charts/actions/workflows/release.yaml"><img src="https://github.com/your-org/charts/actions/workflows/release.yaml/badge.svg" /></a>
</p>

Production-ready Helm charts for deploying applications on [Kubernetes](https://kubernetes.io), optimized for [Google Kubernetes Engine (GKE)](https://cloud.google.com/kubernetes-engine). Ready to launch using [Helm](https://github.com/helm/helm).

## TL;DR

```console
helm repo add your-org https://your-org.github.io/charts
helm install my-release your-org/<chart>
```

## Available Charts

| Chart | Description |
| ----- | ----------- |
| [app-base](./app-base/) | Generic chart for backend applications: REST APIs, workers, and microservices. Supports Cloud SQL Auth Proxy sidecar, file-based secret mounts, and KEDA autoscaling. |
| [app-front](./app-front/) | Generic chart for frontend SPAs where the web server is bundled inside the Docker image. Minimal footprint with full probe and autoscaling support. |
| [app-front-nginx](./app-front-nginx/) | Generic chart for frontend SPAs where NGINX is configured by the chart via a generated ConfigMap. Supports HTTP-only and HTTPS/SSL modes. |

## Prerequisites

- Kubernetes 1.25+
- Helm 3.10+

## Setting up a Kubernetes Cluster

The quickest way to get a Kubernetes cluster up and running is via a managed service:

- [Get Started with GKE](https://cloud.google.com/kubernetes-engine/docs/quickstart)
- [Get Started with EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
- [Get Started with AKS](https://learn.microsoft.com/en-us/azure/aks/learn/quick-kubernetes-deploy-portal)

For on-premises setups, refer to the Kubernetes [getting started guide](https://kubernetes.io/docs/setup/).

## Installing Helm

Helm is a tool for managing Kubernetes charts. Charts are packages of pre-configured Kubernetes resources.

To install Helm, refer to the [Helm install guide](https://helm.sh/docs/intro/install/) and ensure that the `helm` binary is in the `PATH` of your shell.

## Using Helm

Once you have installed the Helm client, you can deploy a chart into a Kubernetes cluster.

Please refer to the [Quick Start guide](https://helm.sh/docs/intro/quickstart/) if you wish to get running in just a few commands, otherwise the [Using Helm Guide](https://helm.sh/docs/intro/using_helm/) provides detailed instructions on how to use the Helm client to manage packages on your Kubernetes cluster.

Useful Helm client commands:

- Add this repo: `helm repo add your-org https://your-org.github.io/charts`
- Install a chart: `helm install my-release your-org/<chart>`
- Upgrade your application: `helm upgrade my-release your-org/<chart>`
- List all releases: `helm list`

## Optional infrastructure dependencies

All charts in this repository share a common set of optional infrastructure components:

| Component | Purpose | Required by |
| --------- | ------- | ----------- |
| [KEDA](https://keda.sh) | CPU/Memory-based pod autoscaling via `ScaledObject` | All charts (`scaledObject.enabled=true`) |
| [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/) | Standard Kubernetes Ingress routing | All charts (`ingress.*.enabled=true`) |
| [GKE Gateway API](https://cloud.google.com/kubernetes-engine/docs/concepts/gateway-api) | HTTPRoute-based routing with native GKE load balancers | All charts (when using `ingress.*.sectionNames`) |
| [Stakater Reloader](https://github.com/stakater/Reloader) | Automatic pod restart when Secrets or ConfigMaps change | All charts (optional) |
| [GKE GCPBackendPolicy CRD](https://cloud.google.com/kubernetes-engine/docs/how-to/configure-gateway-backendpolicies) | GCP-native backend timeout and logging config | All charts (GKE only) |

## Contributing

We welcome contributions! To contribute a fix or improvement:

1. Fork this repository
2. Create a feature branch: `git checkout -b feat/my-change`
3. Bump the `version` field in the affected chart's `Chart.yaml`
4. Update the `README.md` parameter table if you changed `values.yaml`
5. Run `helm lint ./<chart-name>` to validate your changes
6. Open a Pull Request

## License

Copyright &copy; 2024 your-org

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

<http://www.apache.org/licenses/LICENSE-2.0>

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
