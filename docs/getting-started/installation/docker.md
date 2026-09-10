# Install with Docker

Docker Compose is the fastest way to bring up the full Orion Platform 4.2.1 stack together with its dependencies on a single machine. The method targets `dev` environments, workshops and integration tests, so do not use it in production. The set of files below starts six Orion components plus PostgreSQL, Redis, Kafka and MinIO.

## Requirements

| Element | Version |
| --- | --- |
| Docker Engine | 24.0 or newer |
| Docker Compose | v2.20 or newer (the `docker compose` plugin) |
| Free RAM | 8 GiB |
| Free disk space | 20 GiB |
| Free ports | 3000, 5432, 6379, 8080-8083, 9000, 9092, 9100 |

Access to the image registry:

```bash
docker login registry.nimbus.example.com
```

## The compose.yaml file

```yaml title="compose.yaml" hl_lines="9 10 33 34"
name: orion

x-orion-common: &orion-common
  restart: unless-stopped
  env_file: .env
  environment:
    ORION_PROFILE: dev
    ORION_LOG_LEVEL: debug
    ORION_DB_URL: postgres://orion_app@postgres:5432/orion
    ORION_KAFKA_BROKERS: kafka:9092
    ORION_REDIS_URL: redis://redis:6379/0
    ORION_S3_ENDPOINT: http://minio:9000
    ORION_S3_BUCKET: orion-artifacts
  depends_on:
    postgres:
      condition: service_healthy
    redis:
      condition: service_started
    kafka:
      condition: service_healthy

services:
  orion-gateway:
    <<: *orion-common
    image: registry.nimbus.example.com/orion/orion-gateway:4.2.1
    ports:
      - "8080:8080"
    environment:
      ORION_PROFILE: dev
      ORION_CORE_URL: http://orion-core:8081
      ORION_LEDGER_URL: http://orion-ledger:8082
      ORION_OIDC_ISSUER: http://keycloak:8080/realms/orion
      ORION_RATE_LIMIT_RPM: "600"
    healthcheck:
      test: ["CMD", "curl", "-fsS", "http://localhost:8080/healthz"]
      interval: 15s
      timeout: 5s
      retries: 6

  orion-core:
    <<: *orion-common
    image: registry.nimbus.example.com/orion/orion-core:4.2.1
    expose:
      - "8081"

  orion-ledger:
    <<: *orion-common
    image: registry.nimbus.example.com/orion/orion-ledger:4.2.1
    expose:
      - "8082"

  orion-worker:
    <<: *orion-common
    image: registry.nimbus.example.com/orion/orion-worker:4.2.1
    expose:
      - "9100"
    environment:
      ORION_PROFILE: dev
      ORION_WORKER_CONCURRENCY: "4"
      ORION_EVENTS_TOPIC: orion.events.v1
      ORION_EVENTS_DLQ_TOPIC: orion.events.v1.dlq

  orion-scheduler:
    <<: *orion-common
    image: registry.nimbus.example.com/orion/orion-scheduler:4.2.1
    expose:
      - "8083"

  orion-console:
    image: registry.nimbus.example.com/orion/orion-console:4.2.1
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      ORION_API_URL: http://orion-gateway:8080/v1
      ORION_OIDC_ISSUER: http://keycloak:8080/realms/orion

  postgres:
    image: postgres:15.8
    restart: unless-stopped
    environment:
      POSTGRES_DB: orion
      POSTGRES_USER: orion_app
      POSTGRES_PASSWORD: ${ORION_DB_PASSWORD}
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U orion_app -d orion"]
      interval: 10s
      retries: 10

  redis:
    image: redis:7.2
    restart: unless-stopped
    command: ["redis-server", "--maxmemory", "512mb", "--maxmemory-policy", "allkeys-lru"]
    ports:
      - "6379:6379"

  kafka:
    image: apache/kafka:3.6.1
    restart: unless-stopped
    environment:
      KAFKA_NODE_ID: "1"
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: "1"
    ports:
      - "9092:9092"
    volumes:
      - kafkadata:/var/lib/kafka/data
    healthcheck:
      test: ["CMD-SHELL", "/opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list"]
      interval: 15s
      retries: 10

  minio:
    image: minio/minio:RELEASE.2025-03-12T18-04-18Z
    restart: unless-stopped
    command: ["server", "/data", "--console-address", ":9001"]
    environment:
      MINIO_ROOT_USER: ${ORION_S3_ACCESS_KEY}
      MINIO_ROOT_PASSWORD: ${ORION_S3_SECRET_KEY}
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - miniodata:/data

volumes:
  pgdata:
  kafkadata:
  miniodata:
```

## The .env file

```ini title=".env"
ORION_DB_PASSWORD=zmien_to_haslo
ORION_S3_ACCESS_KEY=orion-local
ORION_S3_SECRET_KEY=orion-local-secret
ORION_WEBHOOK_SIGNING_SECRET=whsec_local_dev_environment
ORION_ENCRYPTION_KEY=ZmFrZV9rZXlfMzJfYnl0ZXNfZGV2X29ubHk=
ORION_OIDC_CLIENT_ID=orion-gateway
ORION_OIDC_CLIENT_SECRET=change_me_in_keycloak
```

!!! danger "The .env file does not belong in the repository"
    Add `.env` to `.gitignore`. The `ORION_ENCRYPTION_KEY` and `ORION_WEBHOOK_SIGNING_SECRET` values from this example are meant for the local environment only. Secret management rules are described in [Security](../../operations/security.md).

## Starting the stack

=== "Linux / macOS"

    ```bash
    docker compose pull
    docker compose up -d
    docker compose ps
    ```

=== "Windows (PowerShell)"

    ```powershell
    docker compose pull
    docker compose up -d
    docker compose ps
    ```

## Schema migrations

Run the database migrations before you use the API for the first time. The container exits once the job is finished.

=== "Linux / macOS"

    ```bash
    docker compose run --rm orion-core orion db migrate
    ```

=== "Windows (PowerShell)"

    ```powershell
    docker compose run --rm orion-core orion db migrate
    ```

Expected output:

```text
[core]   applied 42 migrations, schema version 4.2.1
[ledger] applied 18 migrations, schema version 4.2.1
[audit]  applied  7 migrations, schema version 4.2.1
migrate completed in 6.4s
```

## Verification

=== "Linux / macOS"

    ```bash
    curl -s http://localhost:8080/healthz | jq
    ```

=== "Windows (PowerShell)"

    ```powershell
    curl.exe -s http://localhost:8080/healthz | ConvertFrom-Json | ConvertTo-Json -Depth 5
    ```

Expected response:

```json
{
  "status": "ok",
  "version": "4.2.1",
  "profile": "dev",
  "checks": {
    "postgres": "ok",
    "redis": "ok",
    "kafka": "ok",
    "s3": "ok",
    "keycloak": "ok"
  },
  "schema": {
    "core": "4.2.1",
    "ledger": "4.2.1",
    "audit": "4.2.1"
  }
}
```

The operator console is available at `http://localhost:3000`.

## Troubleshooting

### The orion-core container restarts in a loop

Symptom: `docker compose ps` shows the `restarting` status and `ORN-5003` repeats in the logs.

```bash
docker compose logs --tail 50 orion-core
```

The most common cause is missing migrations or a database role without permissions on the schemas. Run `docker compose run --rm orion-core orion db migrate` and check that `ORION_DB_URL` points at the `orion` database. If the error concerns configuration validation (`ORN-1201`), run `docker compose run --rm orion-core orion config validate`.

### No connection to Kafka

Symptom: `orion-worker` logs `ORN-5011 kafka: connection refused` and `orion_queue_lag` is not reported.

```bash
docker compose exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

If the topic list is empty, the broker has not initialized yet, so wait until its healthcheck passes. If the broker is running, check that `ORION_KAFKA_BROKERS` is set to `kafka:9092` and not `localhost:9092`; inside the Compose network the service name applies.

### Port 8080 is already in use

Symptom: `Error response from daemon: failed to bind host port 0.0.0.0:8080`.

=== "Linux / macOS"

    ```bash
    ss -ltnp | grep 8080
    ```

=== "Windows (PowerShell)"

    ```powershell
    Get-NetTCPConnection -LocalPort 8080 | Select-Object OwningProcess, State
    ```

Free the port or change the mapping in `compose.yaml`, for example to `"18080:8080"`. Remember to update `ORION_API_URL` in the `orion-console` service if you refer to the address from the host.

## Stopping the stack and deleting data

Stopping without losing data:

```bash
docker compose stop
```

Removing the whole stack together with its volumes:

```bash
docker compose down -v
```

!!! warning "down -v deletes the database irreversibly"
    The `-v` flag deletes the `pgdata`, `kafkadata` and `miniodata` volumes together with the orders, the ledger and the events. Take a dump first, as described in [Backups](../../operations/backups.md).

## See also

- [Installation](index.md)
- [Environment variables](../configuration/environment-variables.md)
- [First order](../first-order.md)
- [Deploy on Kubernetes](kubernetes/index.md)
