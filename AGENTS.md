# Agent Instructions

## Project Overview

This repository (`infra-compose`) is a local simulation of a **middle-platform infrastructure** for learning and development. It provides shared infrastructure services used by multiple downstream applications, deployed locally via `podman-compose` and integrated with a local Kubernetes cluster (k3s).

## Current Stack

The `compose.yaml` file defines the following services:

| Service      | Purpose                                              |
| ------------ | ---------------------------------------------------- |
| `redis`      | Shared in-memory cache / message broker              |
| `redisinsight` | Redis management UI                                  |
| `kafka`      | Shared event streaming platform (KRaft mode)         |
| `kafka-ui`   | Kafka management UI                                  |
| `postgres`   | Shared relational database                           |
| `headlamp`   | Kubernetes cluster observability UI (profile: k8s-ui) |
| `harbor`     | Local container image and Helm chart registry        |

## Planned Services

The following services are intended to be added to the stack:

- **Graylog** — centralized log aggregation and analysis.
- **Grafana** — metrics visualization and dashboards.

Both services should be configured to **automatically discover and collect data from pods running on the Kubernetes cluster**.

## Architecture Goals

- Keep the architecture simple and suitable for personal learning.
- Prefer GitHub Actions over Tekton for CI/CD.
- Use Graylog as the centralized logging solution.
- Maintain a single infrastructure repository.
- Use `podman` and `podman-compose` as the local container runtime.

## Kubernetes Integration

- The local Kubernetes cluster is managed via **Helm** charts installed with `helm install` / `helm upgrade`.
- Helm charts and their values live under the `helm/` directory and target the `middle-platform` namespace by default.
- `headlamp` consumes a kubeconfig file mounted from `${KUBECONFIG_PATH:-./.kube/config}`.

## Artifact Registry

- **Harbor** is used as the local container image and Helm chart registry.
- Harbor runs as a Compose service and is the default target for building, pushing, and pulling images and charts used by the cluster.
- Configure downstream applications and CI/CD pipelines to push artifacts to Harbor instead of external registries.

## Conventions

- Compose file: `compose.yaml`
- Validate compose config: `podman-compose -f compose.yaml config`
- Helm release namespace: `middle-platform`
- Validate / render Helm charts: `helm template <release> helm/<chart> --namespace middle-platform`
- Default Harbor project: `middle-platform`

## Guidance for Adding Graylog and Grafana

When adding Graylog and Grafana to this stack, follow these principles:

1. **Auto-discovery from Kubernetes**
   - Graylog should collect container logs from Kubernetes pods. Prefer a lightweight log shipper (e.g., Fluent Bit or Promtail) deployed as a DaemonSet in the `middle-platform` namespace, forwarding logs to the Graylog instance.
   - Grafana should discover Prometheus metrics endpoints exposed by pods via Kubernetes ServiceMonitor or PodMonitor resources. A Prometheus instance (or Grafana Agent / Alloy) should be deployed in-cluster to scrape pod targets and act as a Grafana data source.

2. **Local-first configuration**
   - Keep the local Docker Compose stack simple. Graylog and Grafana can run as Compose services, while their collectors/agents run inside Kubernetes.
   - Use environment variables and mounted config files for service configuration.

3. **Service dependencies**
   - Graylog requires MongoDB and Elasticsearch/OpenSearch. Consider whether to run these as additional Compose services or to reuse existing infrastructure.
   - Grafana requires a Prometheus data source. Document how Prometheus is deployed and configured.

4. **Documentation**
   - Update this file and `README.md` with setup steps, required environment variables, and access URLs.
   - Provide Helm charts under `helm/` for log shippers, Prometheus, ServiceMonitors, and any other in-cluster components.
   - Document how to push images and charts to Harbor and how workloads consume them.

5. **Validation**
   - After changes, run `podman-compose -f compose.yaml config` to validate the Compose file.
   - Render Helm charts with `helm template <release> helm/<chart> --namespace middle-platform` before installing.
   - Validate chart values and dependencies with `helm lint helm/<chart>`.

## Notes

- This is a learning project; avoid over-engineering.
- Do not commit sensitive values (passwords, tokens, kubeconfig files) to the repository.
- Prefer declarative configuration and infrastructure-as-code practices.
