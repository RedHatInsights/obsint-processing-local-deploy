# ObsInt Processing Local Deploy - Project Context

## Overview
Local containerized deployment of Observability Intelligence Processing Team services.
Two pipelines: `external/` (external data pipeline) and `internal/` (internal data pipeline).

## Platform
- **OS**: Fedora Linux (amd64/x86_64)
- **Container runtime**: Podman (not Docker)
- **Compose**: `podman compose` (uses podman-compose provider)
- Remove any `docker-compose` binary from PATH — it uses different auth credentials than Podman

## Key Commands
```bash
podman compose up -d      # start services (from external/ or internal/ directory)
podman compose down       # stop services
podman compose logs -f    # follow all logs
podman logs <container>   # single container logs
```

## Podman/SELinux Gotchas
- All bind-mount volumes MUST use `:z` suffix for SELinux relabeling (e.g., `./config.yaml:/config.yaml:z`)
- Do NOT use Docker Swarm `deploy.restart_policy` — use `restart: on-failure` instead
- `platform: linux/amd64` is set on services with Red Hat registry images that may be multi-arch

## Image Registry Conventions
- **docker.io**: Used for open-source images (kafka, redis, minio, mc, prometheus, pushgateway, kcat)
- **quay.io/redhat-services-prod/obsint-processing-tenant/**: Private Red Hat images (requires `podman login quay.io`)
- **quay.io/ccxdev/**: Internal dev images (e.g., kafka-no-zk for internal pipeline)
- **localhost/ingress**: Locally-built ingress image

## Ingress Image
Built from https://github.com/RedHatInsights/insights-ingress-go:
```bash
git clone https://github.com/RedHatInsights/insights-ingress-go.git
cd insights-ingress-go
podman build . -t ingress:latest
```
- The ingress binary defaults to `kafka:29092` (hardcoded in `internal/config/config.go:114`)
- The env var `INGRESS_KAFKA_BROKERS` can override this but Viper may not parse slice types from env correctly
- Safest approach: ensure Kafka listens on port 29092

## Kafka Configuration

### External pipeline (`external/docker-compose.yaml`)
- Uses `docker.io/bitnami/kafka:3.8` (KRaft mode, no ZooKeeper)
- Listeners: INTERNAL on 29092, EXTERNAL on 9092, CONTROLLER on 9093
- All services connect via `kafka:29092`

### Internal pipeline (`internal/docker-compose.yaml`)
- Uses `quay.io/ccxdev/kafka-no-zk:latest` (wurstmeister-style)
- Dual listeners: PLAINTEXT on 9092, INTERNAL on 29092
- Ingress connects on `kafka:29092`, other services use `kafka:9092`
- Topics auto-created via `KAFKA_CREATE_TOPICS` env var
- `kafka-topics-waiter` (kcat) init container waits for topics before dependent services start

## Service Credentials (Local Dev)
| Service | User/Key | Password/Secret |
|---------|----------|-----------------|
| MinIO | `minio` | `minio123` |
| PostgreSQL | `user` | `password` (admin: `admin`) |
| Redis | — | `password` |

## Internal Pipeline Services
| Service | Image | Notes |
|---------|-------|-------|
| ingress | localhost/ingress | Needs kafka:29092 |
| kafka | quay.io/ccxdev/kafka-no-zk | Ports 9092 + 29092 |
| archive-sync | ccx-messaging | Syncs archives from external S3 to internal |
| rules-processing | ccx-messaging | Uses `ccx_messaging` plugins (not `ccx_rules_ocp`) |
| rules-uploader | ccx-messaging | Uploads rule results to S3 |
| parquet-factory | parquet-factory | Generates parquet aggregations |
| minio | docker.io/minio/minio | S3-compatible storage |
| pushgateway | docker.io/prom/pushgateway | Metrics |
| prometheus | docker.io/prom/prometheus | Monitoring |

## Known Issues
- **Startup race condition**: Services depending on `kafka-topics-waiter` may still hit timing issues. If archive-sync or rules-processing exit on first start, restart them: `podman compose up -d <service>`
- **rules-processing image**: Originally used `data-pipeline` image (has `ccx_rules_ocp`). Changed to `ccx-messaging` because `data-pipeline` is not pullable from `quay.io/redhat-services-prod`. Config updated to use `ccx_messaging` plugins instead.
- **parquet-factory**: Has `restart: always` but may exit before Kafka is ready on first run. Will auto-restart.

## MinIO Buckets
### External pipeline
- `insights-upload-perma`, `insights-upload-rejected`

### Internal pipeline
- `external-storage`, `internal-storage-io`, `ccx-bucket`
