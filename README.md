# kafka-podman-compose

3-node ZooKeeper + 3-broker Kafka HA cluster with Kafka UI, tuned for local Podman Compose (also works with Docker Compose).

## Stack

| Service | Host access | Notes |
|---------|-------------|--------|
| ZooKeeper 1–3 | `localhost:2181` (ZK-1 only) | Ensemble quorum on the compose network |
| Kafka 1 | `localhost:9091` | External `PLAINTEXT_HOST` |
| Kafka 2 | `localhost:9092` | External `PLAINTEXT_HOST` |
| Kafka 3 | `localhost:9093` | External `PLAINTEXT_HOST` |
| Kafka UI | [http://localhost:8083](http://localhost:8083) | Cluster: `ha-cluster` |

Listeners:

- **Inter-broker / in-network:** `kafka-N:29092` (`PLAINTEXT`)
- **From your host:** `localhost:9091,9092,9093` (`PLAINTEXT_HOST`)

Defaults: RF=3, `min.insync.replicas=2`, 24h log retention, sized JVM heaps for a laptop/lab host.

## Prerequisites

- [Podman](https://podman.io/) 4+ with Compose (`podman compose` / `podman-compose`), **or**
- Docker Compose V2 (`docker compose`)

## Quick start

```bash
podman compose up -d
# or: docker compose up -d
```

Wait until brokers are healthy, then open http://localhost:8083

```bash
podman compose ps
```

Host bootstrap:

```text
localhost:9091,localhost:9092,localhost:9093
```

Create a topic:

```bash
podman exec kafka-1 kafka-topics --bootstrap-server kafka-1:29092 \
  --create --topic demo --partitions 3 --replication-factor 3
```

## Tuning

Copy or edit `.env` before starting:

| Variable | Default | Purpose |
|----------|---------|---------|
| `ZK_HEAP_OPTS` | `-Xms128m -Xmx256m` | ZooKeeper JVM heap |
| `ZK_MEM_LIMIT` | `512m` | ZooKeeper container memory cap |
| `KAFKA_HEAP_OPTS` | `-Xms512m -Xmx768m` | Broker JVM heap |
| `KAFKA_MEM_LIMIT` | `1g` | Broker container memory cap |
| `KAFKA_LOG_RETENTION_HOURS` | `24` | Topic log retention |
| `UI_JAVA_OPTS` / `UI_MEM_LIMIT` | `512m` / `768m` | Kafka UI sizing |

Recreate after changing env:

```bash
podman compose up -d --force-recreate
```

## What was optimized

- Bounded JVM heaps + container `mem_limit` (Confluent defaults are multi‑GB)
- Healthchecks (`cub zk-ready` / `cub kafka-ready`) and ordered startup
- Consistent listeners: internal `29092`, host ports `9091–9093` 1:1
- Faster consumer rebalance for labs (`group.initial.rebalance.delay.ms=0`)
- Shorter retention + smaller log segments for disk churn
- ZooKeeper autopurge + higher init/sync limits for slower local disks
- Disabled Confluent phone-home metrics
- Pinned Kafka UI image (`v0.7.2`)
- Raised `nofile` ulimits

Rootless Podman may leave health status on `starting` until a check runs; `podman healthcheck run <name>` (or enabling the user podman timer) marks services healthy. Docker Compose runs checks automatically.

## Stop / clean up

```bash
podman compose down
podman compose down -v   # also wipe ZK + Kafka data
```

## Notes

- Image names are fully qualified (`docker.io/...`) for Podman without short-name registries.
- `ZOOKEEPER_SERVERS` must stay semicolon-separated with **no** spaces after `;`.
- Host clients must use `9091–9093`, not the internal `29092` listener.
- Plaintext lab layout only — do not expose on untrusted networks.
