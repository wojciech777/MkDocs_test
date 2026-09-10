# Environment variables

Every configuration key can be set with an environment variable prefixed with `ORION_`. The variable name matches the key path from `orion.yaml` written in upper case with an underscore in place of each nesting level, so `database.poolSize` becomes `ORION_DB_POOL_SIZE`. Variables have a higher priority than the configuration file but a lower priority than CLI flags and console overrides.

## Basics

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `ORION_PROFILE` | enum | `staging` | Configuration profile: `dev`, `staging`, `prod` |
| `ORION_CONFIG` | path | `/etc/orion/orion.yaml` | Location of the configuration file |
| `ORION_PUBLIC_URL` | URL | - | Public API address, for example `https://api.orion.example.com/v1` |
| `ORION_LISTEN_ADDR` | string | `0.0.0.0` | Listen address of the component |
| `ORION_REQUEST_TIMEOUT` | duration | `30s` | Time limit for handling a single HTTP request |
| `ORION_CORE_URL` | URL | `http://orion-core:8081` | Address of the orders engine used by the gateway |
| `ORION_LEDGER_URL` | URL | `http://orion-ledger:8082` | Address of the payment ledger used by the gateway |
| `ORION_SHUTDOWN_GRACE` | duration | `20s` | Grace period for closing connections during shutdown |

## Database

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `ORION_DB_URL` | URL | - | Address of the `orion` database, for example `postgres://postgres.internal:5432/orion`; required |
| `ORION_DB_USERNAME` | string | `orion_app` | Application role used at runtime |
| `ORION_DB_PASSWORD` | secret | - | Password of the application role |
| `ORION_DB_MIGRATOR_USERNAME` | string | `orion_migrator` | Role used only by `orion db migrate` |
| `ORION_DB_MIGRATOR_PASSWORD` | secret | - | Password of the migrator role |
| `ORION_DB_POOL_SIZE` | int | `20` | Maximum number of connections in the pool per replica (limit 500) |
| `ORION_DB_STATEMENT_TIMEOUT` | duration | `30s` | Time limit for a single SQL query |
| `ORION_DB_SSL_MODE` | enum | `prefer` | `disable`, `prefer`, `require`, `verify-full`; the `prod` profile forces `require` |
| `ORION_DB_SCHEMA_CORE` | string | `core` | Schema name of the orders engine |
| `ORION_DB_SCHEMA_LEDGER` | string | `ledger` | Schema name of the payment ledger |
| `ORION_DB_SCHEMA_AUDIT` | string | `audit` | Schema name of the audit log |

## Kafka

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `ORION_KAFKA_BROKERS` | list | - | Comma-separated broker list; required |
| `ORION_EVENTS_TOPIC` | string | `orion.events.v1` | Domain event topic |
| `ORION_EVENTS_DLQ_TOPIC` | string | `orion.events.v1.dlq` | Dead letter queue topic |
| `ORION_KAFKA_CONSUMER_GROUP` | string | `orion-worker` | Consumer group of the `orion-worker` component |
| `ORION_KAFKA_SASL_MECHANISM` | enum | `none` | `none`, `plain`, `scram-sha-512` |
| `ORION_KAFKA_SASL_USERNAME` | string | - | SASL user name |
| `ORION_KAFKA_SASL_PASSWORD` | secret | - | SASL password |
| `ORION_KAFKA_PRODUCER_ACKS` | enum | `all` | Producer acknowledgement level: `1` or `all` |
| `ORION_WORKER_CONCURRENCY` | int | `2` | Number of events processed in parallel per replica |
| `ORION_WORKER_MAX_RETRIES` | int | `5` | Number of retries before an event is moved to the DLQ |

## Redis

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `ORION_REDIS_URL` | URL | - | Redis address, for example `redis://redis.internal:6379/0`; required |
| `ORION_REDIS_PASSWORD` | secret | - | Redis password, if one is required |
| `ORION_REDIS_TLS` | bool | `false` | Forces a TLS connection |
| `ORION_IDEMPOTENCY_TTL` | duration | `24h` | Retention period for responses stored for `Idempotency-Key` |
| `ORION_RATE_LIMIT_RPM` | int | `600` | Rate limit per minute for a single key |
| `ORION_RATE_LIMIT_BURST` | int | `100` | Allowed short-term excess of requests |

## Security

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `ORION_OIDC_ISSUER` | URL | - | Token issuer address, for example `https://sso.example.com/realms/orion` |
| `ORION_OIDC_CLIENT_ID` | string | `orion-gateway` | OAuth2 client identifier |
| `ORION_OIDC_CLIENT_SECRET` | secret | - | Secret of the confidential client |
| `ORION_OIDC_REALM` | string | `orion` | Realm in Keycloak 24 |
| `ORION_TOKEN_TTL` | duration | `3600s` | Expected lifetime of an access token |
| `ORION_API_KEYS_ENABLED` | bool | `true` | Allows authentication with the `X-Orion-Key` key |
| `ORION_ENCRYPTION_KEY` | secret | - | 32-byte key (base64) used to encrypt sensitive data in the database |
| `ORION_WEBHOOK_SIGNING_SECRET` | secret | - | Signing secret for the `X-Orion-Signature` header |
| `ORION_TLS_ENABLED` | bool | `false` | TLS termination inside the component; the `prod` profile forces `true` |
| `ORION_TLS_CERT_FILE` | path | - | Server certificate in PEM format |
| `ORION_TLS_KEY_FILE` | path | - | Private key in PEM format |
| `ORION_CORS_ALLOWED_ORIGINS` | list | `[]` | Allowed origins for the console and browser applications |

## Observability

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `ORION_LOG_LEVEL` | enum | `info` | `trace`, `debug`, `info`, `warn`, `error` |
| `ORION_LOG_FORMAT` | enum | `json` | `json` or `text`; `text` only for the `dev` profile |
| `ORION_METRICS_ENABLED` | bool | `true` | Exposes Prometheus metrics with the `orion_` prefix |
| `ORION_METRICS_PORT` | int | `9100` | Metrics port for `orion-worker` |
| `ORION_TRACING_ENABLED` | bool | `false` | Exports traces over OTLP |
| `ORION_TRACING_ENDPOINT` | URL | - | OTLP collector, for example `http://otel-collector:4317` |
| `ORION_TRACING_SAMPLING_RATIO` | float | `0.05` | Fraction of sampled requests, range 0.0 - 1.0 |
| `ORION_AUDIT_RETENTION_DAYS` | int | `90` | Retention period for entries in the `audit` schema |

## Object storage

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `ORION_S3_ENDPOINT` | URL | - | Address of the S3-compatible service |
| `ORION_S3_BUCKET` | string | `orion-artifacts` | Bucket for ledger exports and backups |
| `ORION_S3_ACCESS_KEY` | secret | - | S3v4 access key |
| `ORION_S3_SECRET_KEY` | secret | - | S3v4 secret key |
| `ORION_S3_REGION` | string | `eu-central-1` | Region required by the S3v4 signature |
| `ORION_S3_FORCE_PATH_STYLE` | bool | `true` | Required for MinIO |

## Secrets

Do not bake secrets into the container image or into an `orion.yaml` file kept in a repository. Every variable marked with the `secret` type has a counterpart with the `_FILE` suffix that takes the path to a file holding the value. The component reads the file at startup and on a configuration reload.

```bash
ORION_DB_PASSWORD_FILE=/etc/orion/secrets/db-password
ORION_WEBHOOK_SIGNING_SECRET_FILE=/etc/orion/secrets/webhook-signing-secret
ORION_ENCRYPTION_KEY_FILE=/etc/orion/secrets/encryption-key
ORION_OIDC_CLIENT_SECRET_FILE=/etc/orion/secrets/oidc-client-secret
```

Rules:

- The secret file must have mode `0640` and owner `root:orion`; broader permissions end with the `ORN-1207` error.
- Setting both `ORION_X` and `ORION_X_FILE` is a configuration error (`ORN-1209`), not a silent preference for one of the values.
- In Kubernetes, mount secrets as a volume rather than as `env`, so that a change to the value in the `Secret` object becomes visible after a reload without recreating the pod.
- `ORION_ENCRYPTION_KEY` is immutable for the lifetime of the installation. Rotation requires the re-encryption procedure described in [Security](../../operations/security.md).

!!! danger "Secrets never reach the logs"
    Components mask `secret` values in the logs and in the output of `orion config validate`, printing `***` instead. Do not log them yourself on the integration side; an entry in `audit` contains only the key identifier, never its value.

## Example .env file

```ini title=".env"
ORION_PROFILE=staging
ORION_PUBLIC_URL=https://api.sandbox.orion.example.com/v1
ORION_LOG_LEVEL=debug

ORION_DB_URL=postgres://postgres.internal:5432/orion
ORION_DB_USERNAME=orion_app
ORION_DB_PASSWORD_FILE=/run/secrets/orion-db-password
ORION_DB_POOL_SIZE=30
ORION_DB_SSL_MODE=require

ORION_KAFKA_BROKERS=kafka-1.internal:9092,kafka-2.internal:9092
ORION_EVENTS_TOPIC=orion.events.v1
ORION_EVENTS_DLQ_TOPIC=orion.events.v1.dlq
ORION_WORKER_CONCURRENCY=4

ORION_REDIS_URL=redis://redis.internal:6379/0
ORION_RATE_LIMIT_RPM=60
ORION_RATE_LIMIT_BURST=100

ORION_OIDC_ISSUER=https://sso.example.com/realms/orion
ORION_OIDC_CLIENT_ID=orion-gateway
ORION_OIDC_CLIENT_SECRET_FILE=/run/secrets/orion-oidc-secret
ORION_WEBHOOK_SIGNING_SECRET_FILE=/run/secrets/orion-webhook-secret
ORION_ENCRYPTION_KEY_FILE=/run/secrets/orion-encryption-key

ORION_S3_ENDPOINT=https://s3.internal:9000
ORION_S3_BUCKET=orion-artifacts
ORION_S3_FORCE_PATH_STYLE=true

ORION_METRICS_ENABLED=true
ORION_TRACING_ENABLED=true
ORION_TRACING_ENDPOINT=http://otel-collector:4317
ORION_TRACING_SAMPLING_RATIO=0.10
```

## Variables deprecated in 4.2

!!! warning "The old names keep working until the 4.2 line goes out of support"
    Variables from the "old name" column are still accepted, but they emit the `ORN-1250` warning at startup and will be removed in release 5.0. Update your configuration during the next maintenance window.

| Old name (up to 4.0) | New name (from 4.2) | Notes |
| --- | --- | --- |
| `ORION_DATABASE_DSN` | `ORION_DB_URL` | name change only |
| `ORION_DB_MAX_CONNECTIONS` | `ORION_DB_POOL_SIZE` | limit lowered from 1000 to 500 |
| `ORION_BROKERS` | `ORION_KAFKA_BROKERS` | name change only |
| `ORION_TOPIC_EVENTS` | `ORION_EVENTS_TOPIC` | name change only |
| `ORION_TOPIC_DEAD_LETTER` | `ORION_EVENTS_DLQ_TOPIC` | name change only |
| `ORION_CACHE_URL` | `ORION_REDIS_URL` | name change only |
| `ORION_THROTTLE_LIMIT` | `ORION_RATE_LIMIT_RPM` | the value is now interpreted per minute, not per second |
| `ORION_KEYCLOAK_URL` | `ORION_OIDC_ISSUER` | the new value must include the realm path |
| `ORION_HMAC_SECRET` | `ORION_WEBHOOK_SIGNING_SECRET` | the signing algorithm is unchanged |
| `ORION_DEBUG` | `ORION_LOG_LEVEL` | `ORION_DEBUG=true` corresponds to `ORION_LOG_LEVEL=debug` |

## See also

- [Configuration](index.md)
- [Configuration profiles](profiles.md)
- [Security](../../operations/security.md)
- [Install with Docker](../installation/docker.md)
