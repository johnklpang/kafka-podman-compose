# kafka-podman-compose

3-node ZooKeeper + 3-broker Kafka HA cluster with Kafka UI, runnable via Podman Compose (or Docker Compose).

## Stack

| Service | Host access | Notes |
|---------|-------------|--------|
| ZooKeeper 1–3 | `localhost:2181` (ZK-1 only) | Ensemble quorum over the compose network |
| Kafka 1 | `localhost:9091` | External `PLAINTEXT_HOST` listener |
| Kafka 2 | `localhost:9092` | Maps host `9092` → container `29092` |
| Kafka 3 | `localhost:9093` | External `PLAINTEXT_HOST` listener |
| Kafka UI | [http://localhost:8083](http://localhost:8083) | Cluster name: `ha-cluster` |

Brokers advertise:

- **Inter-broker / in-network:** `kafka-N:9092` (`PLAINTEXT`)
- **From your host:** `localhost:9091`, `localhost:9092`, `localhost:9093` (`PLAINTEXT_HOST`)

Replication defaults: RF=3, `min.insync.replicas=2`.

## Prerequisites

- [Podman](https://podman.io/) 4+ with Compose support (`podman compose`), **or**
- Docker with Compose V2 (`docker compose`)

## Quick start

```bash
# Podman
podman compose up -d

# Docker
docker compose up -d
```

Check status:

```bash
podman compose ps
# or
docker compose ps
```

Open Kafka UI: http://localhost:8083

Host client bootstrap servers:

```text
localhost:9091,localhost:9092,localhost:9093
```

Example topic create (from any Kafka container):

```bash
podman exec kafka-1 kafka-topics --bootstrap-server kafka-1:9092 \
  --create --topic demo --partitions 3 --replication-factor 3
```

## Stop / clean up

```bash
podman compose down
# remove volumes (wipes ZK + Kafka data)
podman compose down -v
```

## Notes

- Named volumes persist ZooKeeper and Kafka data across restarts.
- `ZOOKEEPER_SERVERS` must stay semicolon-separated with **no** spaces after `;`.
- External clients on the host must use the `PLAINTEXT_HOST` ports (`9091`–`9093`), not the internal `9092` listener.
- This is a local / lab HA layout (plaintext, no TLS/SASL). Do not expose it on untrusted networks.
