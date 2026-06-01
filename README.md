
# Infra Compose

這個 repo 提供兩套本地基礎設施骨架：

- `podman-compose` 本機開發堆疊
- `k3s` 可直接套用的 Kubernetes manifests

目前包含：

- Redis + RedisInsight
- Kafka + Kafka UI
- PostgreSQL
- Headlamp

`healamp` 這裡我先視為 `Headlamp`，因為它是最常見的 Kubernetes Web UI，且適合搭配 k3s。

## Installation of Podman

```shell
sudo apt update
sudo apt install podman podman-compose -y

podman --version
podman-compose version
```

## Local Stack With Podman Compose

先建立環境檔：

```shell
cp .env.example .env
```

啟動基礎服務：

```shell
podman-compose up -d
```

如果你也要把 Headlamp 一起跑起來，需要先準備 kubeconfig，然後開 profile：

```shell
mkdir -p .kube
cp ~/.kube/config .kube/config
podman-compose --profile k8s-ui up -d
```

停止服務：

```shell
podman-compose down
```

服務入口：

- Kafka UI: `http://localhost:8080`
- RedisInsight: `http://localhost:5540`
- Headlamp: `http://localhost:4466`
- Redis: `localhost:6379`
- PostgreSQL: `localhost:5432`
- Kafka external listener: `localhost:9094`

說明：

- Kafka 在 compose 中使用單節點 KRaft 模式，適合本機開發，不是高可用配置。
- Kafka UI 會透過內部網路連到 `kafka:9092`。
- Headlamp 在 compose 中預設為 optional profile，因為它需要 kubeconfig 掛載。

## Deploy To k3s

可以，這些服務都能部署在 k3s 上。這個 repo 已經提供 `k8s/` manifests。

套用方式：

```shell
kubectl apply -k k8s
kubectl -n middle-platform get pods
```

對外存取方式：

- Kafka UI: `http://<k3s-node-ip>:30080`
- RedisInsight: `http://<k3s-node-ip>:30540`
- Headlamp: `http://<k3s-node-ip>:30466`

叢集內部服務：

- Redis: `redis.middle-platform.svc.cluster.local:6379`
- PostgreSQL: `postgres.middle-platform.svc.cluster.local:5432`
- Kafka: `kafka.middle-platform.svc.cluster.local:9092`

如果你要從本機連 k3s 裡的 Kafka，建議使用 port-forward：

```shell
kubectl -n middle-platform port-forward svc/kafka 9094:9094
```

之後本機 client 可用 `localhost:9094`。

## k3s Notes

- 這份 k3s 配置是單節點、偏開發用途，不是 production HA 配置。
- PVC 會依賴 k3s 預設的 storage class，通常是 `local-path`。
- Headlamp 在 `k8s/headlamp.yaml` 內使用 `cluster-admin` 綁定，這對本機或 lab 環境方便，但不適合直接照搬到正式環境。