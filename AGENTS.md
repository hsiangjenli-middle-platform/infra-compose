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
| `graylog`    | Centralized log aggregation (profile: graylog)       |
| `grafana`    | Metrics visualization and dashboards (profile: grafana) |

## Planned Services

The following in-cluster components are intended to be added:

- **Fluent Bit DaemonSet** — forwards pod logs to Graylog.
- **Prometheus** — scrapes pod metrics via ServiceMonitor / PodMonitor resources.
- **Harbor** — local container image and Helm chart registry, deployed on Kubernetes.

Graylog and Grafana should be configured to **automatically discover and collect data from pods running on the Kubernetes cluster**.

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
- Harbor is deployed on Kubernetes via Helm chart (under `helm/harbor` or Bitnami's Harbor chart) rather than as a Compose service.
- Configure downstream applications and CI/CD pipelines to push artifacts to Harbor instead of external registries.

## Conventions

- Compose file: `compose.yaml`
- Validate compose config: `podman-compose -f compose.yaml config`
- Helm release namespace: `middle-platform`
- Validate / render Helm charts: `helm template <release> helm/<chart> --namespace middle-platform`
- Default Harbor project: `middle-platform`

## Service Details

### Graylog

- Runs as a Compose service under the `graylog` profile.
- Requires `graylog-mongodb` and `graylog-opensearch` as backing services.
- OpenSearch 2.12+ requires `OPENSEARCH_INITIAL_ADMIN_PASSWORD` with at least 8 characters, one uppercase letter, one lowercase letter, one digit, and one special character.
- Default access: http://localhost:9000 (admin / admin).
- Log inputs (GELF/UDP, Syslog/UDP) are exposed on `12201/udp` and `1514/udp`.

### Grafana

- Runs as a Compose service under the `grafana` profile.
- Mounts provisioning files from `${GRAFANA_PROVISIONING_PATH:-./grafana/provisioning}`.
- Default access: http://localhost:3000 (admin / admin).
- Prometheus should be deployed in-cluster and configured as a data source via provisioning or the UI.

## Guidance for Adding In-Cluster Components

When adding Fluent Bit, Prometheus, Harbor, and related components, follow these principles:

1. **Auto-discovery from Kubernetes**
   - Graylog should collect container logs from Kubernetes pods. Prefer a lightweight log shipper (e.g., Fluent Bit or Promtail) deployed as a DaemonSet in the `middle-platform` namespace, forwarding logs to the Graylog instance.
   - Grafana should discover Prometheus metrics endpoints exposed by pods via Kubernetes ServiceMonitor or PodMonitor resources. A Prometheus instance (or Grafana Agent / Alloy) should be deployed in-cluster to scrape pod targets and act as a Grafana data source.

2. **Local-first configuration**
   - Keep the local Docker Compose stack simple. Graylog and Grafana run as Compose services, while their collectors/agents run inside Kubernetes.
   - Use environment variables and mounted config files for service configuration.

3. **Service dependencies**
   - Graylog requires MongoDB and Elasticsearch/OpenSearch. These run as additional Compose services.
   - Grafana requires a Prometheus data source. Document how Prometheus is deployed and configured.

4. **Documentation**
   - Update this file and `README.md` with setup steps, required environment variables, and access URLs.
   - Provide Helm charts under `helm/` for log shippers, Prometheus, ServiceMonitors, Harbor, and any other in-cluster components.
   - Document how to push images and charts to Harbor and how workloads consume them.

5. **Validation**
   - After changes, run `podman-compose -f compose.yaml config` to validate the Compose file.
   - Render Helm charts with `helm template <release> helm/<chart> --namespace middle-platform` before installing.
   - Validate chart values and dependencies with `helm lint helm/<chart>`.

## Notes

- This is a learning project; avoid over-engineering.
- Do not commit sensitive values (passwords, tokens, kubeconfig files) to the repository.
- Prefer declarative configuration and infrastructure-as-code practices.
