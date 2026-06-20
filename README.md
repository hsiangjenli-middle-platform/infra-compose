# Infra Compose

A local simulation of a **middle-platform infrastructure** for learning and development. It provides shared infrastructure services (cache, message broker, database, observability) deployed locally via `podman-compose`, integrated with a local Kubernetes cluster (k3s) through Helm charts.

## Prerequisites

- [Podman](https://podman.io/) and `podman-compose`
- A local Kubernetes cluster (e.g., [k3s](https://k3s.io/))
- [Helm](https://helm.sh/) for deploying in-cluster components

Install Podman on Ubuntu/Debian:

```shell
sudo apt update
sudo apt install podman podman-compose -y

podman --version
podman-compose version
```

## Current Stack

The `compose.yaml` file defines the following services:

| Service        | Port(s)              | Purpose                                              | Profile    |
| -------------- | -------------------- | ---------------------------------------------------- | ---------- |
| `redis`        | `6379`               | Shared in-memory cache / message broker              | (default)  |
| `redisinsight` | `5540`               | Redis management UI                                  | (default)  |
| `kafka`        | `9094`               | Shared event streaming platform (KRaft mode)         | (default)  |
| `kafka-ui`     | `8080`               | Kafka management UI                                  | (default)  |
| `postgres`     | `5432`               | Shared relational database                           | (default)  |
| `headlamp`     | `4466`               | Kubernetes cluster observability UI                  | `k8s-ui`   |
| `graylog`      | `9000`, `12201/udp`, `1514/udp` | Centralized log aggregation                 | `graylog`  |
| `grafana`      | `3000`               | Metrics visualization and dashboards                 | `grafana`  |

## Quick Start

Start the core infrastructure:

```shell
podman-compose -f compose.yaml up -d
```

Start optional profiles:

```shell
podman-compose -f compose.yaml --profile k8s-ui up -d
podman-compose -f compose.yaml --profile graylog up -d
podman-compose -f compose.yaml --profile grafana up -d
```

> **Note:** `podman-compose` requires the `-f compose.yaml` flag **before** `--profile`.

## Access URLs

| Service        | URL                         | Default Credentials            |
| -------------- | --------------------------- | ------------------------------ |
| RedisInsight   | http://localhost:5540       | —                              |
| Kafka UI       | http://localhost:8080       | —                              |
| Headlamp       | http://localhost:4466       | —                              |
| Graylog        | http://localhost:9000       | `admin` / `admin`              |
| Grafana        | http://localhost:3000       | `admin` / `admin`              |

## Configuration

Copy the example environment file and adjust values as needed:

```shell
cp .env.example .env
```

Key environment variables:

| Variable | Description | Default |
| -------- | ----------- | ------- |
| `POSTGRES_DB` / `POSTGRES_USER` / `POSTGRES_PASSWORD` | PostgreSQL database credentials | `app` / `app` / `app-secret` |
| `GRAYLOG_PASSWORD_SECRET` | Graylog password pepper | `somepasswordpepper` |
| `GRAYLOG_ROOT_PASSWORD_SHA2` | SHA-256 of the Graylog root password | `admin` |
| `GRAYLOG_OPENSEARCH_ADMIN_PASSWORD` | OpenSearch initial admin password (must contain uppercase, lowercase, digit, and special char) | `Graylog@OpenSearch1` |
| `GRAYLOG_OPENSEARCH_JAVA_OPTS` | OpenSearch JVM options | `-Xms512m -Xmx512m` |
| `GRAFANA_ADMIN_USER` / `GRAFANA_ADMIN_PASSWORD` | Grafana admin credentials | `admin` / `admin` |
| `GRAFANA_PROVISIONING_PATH` | Path to Grafana provisioning directory | `./grafana/provisioning` |
| `KUBECONFIG_PATH` | Path to kubeconfig for Headlamp | `./.kube/config` |

## Kubernetes Integration

In-cluster components (Fluent Bit, Prometheus, Harbor, etc.) are deployed via Helm charts under the `helm/` directory, targeting the `middle-platform` namespace by default.

Render a chart before installing:

```shell
helm template <release> helm/<chart> --namespace middle-platform
```

Install or upgrade:

```shell
helm upgrade --install <release> helm/<chart> --namespace middle-platform --create-namespace
```

## Validation

Validate the Compose file:

```shell
podman-compose -f compose.yaml config
```

Validate a Helm chart:

```shell
helm lint helm/<chart>
helm template <release> helm/<chart> --namespace middle-platform
```

## Project Structure

```
.
├── compose.yaml          # Podman Compose stack
├── AGENTS.md             # Agent instructions and architecture guidance
├── README.md             # This file
├── helm/                 # Helm charts for in-cluster components
├── grafana/provisioning/ # Grafana datasources and dashboards
└── docs/                 # Architecture diagrams and documentation
```

## Notes

- This is a learning project; keep the architecture simple and avoid over-engineering.
- Do not commit sensitive values (passwords, tokens, kubeconfig files) to the repository.
- Prefer declarative configuration and infrastructure-as-code practices.
