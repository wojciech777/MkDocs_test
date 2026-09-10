# Requirements

Before you start Orion Platform 4.2 LTS, make sure the environment meets the hardware requirements, the external dependency versions and the network access rules described below. The values apply to a single instance handling up to 2,000,000 orders per month. For larger volumes, follow the guidance in the [Scaling](../operations/scaling.md) section.

## Hardware requirements

| Environment | CPU (cores) | RAM | Disk (SSD) | Component replicas |
| --- | --- | --- | --- | --- |
| `dev` | 4 | 8 GiB | 40 GiB | 1 of each component |
| `staging` | 8 | 16 GiB | 200 GiB | gateway 2, core 2, ledger 1, worker 2, scheduler 1, console 1 |
| `prod` | 16 | 48 GiB | 1 TiB plus a backup volume | gateway 3, core 3, ledger 2, worker 6, scheduler 2, console 2 |

Demand per single pod or process:

| Component | CPU request / limit | RAM request / limit |
| --- | --- | --- |
| `orion-gateway` | 250m / 1000m | 512 MiB / 1 GiB |
| `orion-core` | 500m / 2000m | 1 GiB / 2 GiB |
| `orion-ledger` | 250m / 1000m | 768 MiB / 1.5 GiB |
| `orion-worker` | 250m / 1500m | 512 MiB / 1 GiB |
| `orion-scheduler` | 100m / 500m | 256 MiB / 512 MiB |
| `orion-console` | 100m / 500m | 256 MiB / 512 MiB |

!!! note "The database disk grows with the number of events"
    The `audit` schema accounts for roughly 60% of database growth. With a 90-day retention period and 2,000,000 orders per month, plan for around 350 GiB for the audit log alone. Retention is set by the profile, see [Configuration profiles](configuration/profiles.md).

## Dependency versions

| Dependency | Minimum | Recommended | Unsupported |
| --- | --- | --- | --- |
| PostgreSQL | 15.4 | 15.8 | 13.x and older, 16.x, 17.x |
| Redis | 7.0 | 7.2 | 6.x and older, 8.x |
| Apache Kafka | 3.5 | 3.6 | 2.x, 3.0 to 3.4 |
| Keycloak | 24.0 | 24.0.5 | 22.x and older, 25.x |
| S3 / MinIO | S3v4 API | MinIO RELEASE.2025-03 | the S3v2 interface |
| Docker Engine | 24.0 | 26.1 | 20.10 and older |
| Docker Compose | v2.20 | v2.29 | v1.x |
| Kubernetes | 1.28 | 1.30 | 1.26 and older, 1.32+ |
| Helm | 3.13 | 3.15 | 2.x |
| cert-manager | 1.14 | 1.15 | 1.11 and older |

!!! danger "Kafka 3.0 to 3.4 is not supported"
    Release 4.2 requires a transactional producer with exactly-once semantics in the API version available from Kafka 3.5 onwards. On older brokers `orion-core` refuses to start with error `ORN-5011`.

## Supported operating systems

| System | Versions | Installation methods |
| --- | --- | --- |
| Ubuntu Server | 22.04 LTS, 24.04 LTS | `.deb` packages, Docker, Kubernetes |
| Red Hat Enterprise Linux | 9.2 to 9.4 | `.rpm` packages, Docker, Kubernetes |
| Rocky Linux / AlmaLinux | 9.x | `.rpm` packages, Docker, Kubernetes |
| Debian | 12 | `.deb` packages, Docker |
| Windows Server | 2022 | Docker Desktop in a `dev` environment only |

Processor architectures: `linux/amd64` and `linux/arm64`. Images are published as a multi-architecture manifest.

## Supported browsers for orion-console

| Browser | Minimum version |
| --- | --- |
| Google Chrome / Chromium | 120 |
| Microsoft Edge | 120 |
| Mozilla Firefox | 121 |
| Safari | 17 |

## Ports

| Port | Component | Direction | Protocol | Notes |
| --- | --- | --- | --- | --- |
| 8080 | `orion-gateway` | inbound from the internet | HTTP/1.1, HTTP/2 | behind TLS termination |
| 8081 | `orion-core` | internal | HTTP/1.1 | do not expose publicly |
| 8082 | `orion-ledger` | internal | HTTP/1.1 | do not expose publicly |
| 8083 | `orion-scheduler` | internal | HTTP/1.1 | probes and job inspection |
| 9100 | `orion-worker` | internal | HTTP/1.1 | Prometheus metrics only |
| 3000 | `orion-console` | inbound from the corporate network | HTTP/1.1 | behind TLS termination |
| 5432 | PostgreSQL 15 | outbound from the components | TCP | TLS required in the `prod` profile |
| 6379 | Redis 7 | outbound from `orion-gateway` | TCP | |
| 9092 | Kafka 3.6 | bidirectional | TCP | `orion-core`, `orion-ledger`, `orion-worker`, `orion-scheduler` |
| 9000 | S3 / MinIO | outbound from `orion-ledger` | HTTP/HTTPS | the `orion-artifacts` bucket |
| 8443 | Keycloak 24 | outbound from `orion-gateway` | HTTPS | realm `orion` |

## PostgreSQL permissions

Create one login role for the platform and three schemas with a separate migration owner.

```sql title="orion-bootstrap.sql" hl_lines="4 12 13 14"
CREATE ROLE orion_app WITH LOGIN PASSWORD 'change_me' CONNECTION LIMIT 200;
CREATE ROLE orion_migrator WITH LOGIN PASSWORD 'change_me_too';

CREATE DATABASE orion OWNER orion_migrator ENCODING 'UTF8' LC_COLLATE 'en_US.UTF-8';

\connect orion

CREATE SCHEMA core AUTHORIZATION orion_migrator;
CREATE SCHEMA ledger AUTHORIZATION orion_migrator;
CREATE SCHEMA audit AUTHORIZATION orion_migrator;

GRANT USAGE ON SCHEMA core, ledger, audit TO orion_app;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA core, ledger TO orion_app;
GRANT SELECT, INSERT ON ALL TABLES IN SCHEMA audit TO orion_app;
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA core, ledger, audit TO orion_app;

ALTER DEFAULT PRIVILEGES FOR ROLE orion_migrator IN SCHEMA core, ledger
    GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO orion_app;
ALTER DEFAULT PRIVILEGES FOR ROLE orion_migrator IN SCHEMA audit
    GRANT SELECT, INSERT ON TABLES TO orion_app;

CREATE EXTENSION IF NOT EXISTS pgcrypto;
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
```

!!! warning "The application role must not change the schema"
    `orion_app` deliberately has no `CREATE` privilege in the schemas. Run migrations exclusively as the `orion_migrator` role through `orion db migrate`. Running a migration as the application role fails with error `ORN-5003`.

??? info "Required PostgreSQL server parameters"
    - `max_connections` at least 300 for the `prod` environment
    - `shared_buffers` around 25% of host memory
    - `wal_level = replica` (required for continuous backups)
    - `statement_timeout = 30s` for the `orion_app` role
    - `idle_in_transaction_session_timeout = 60s`

## Pre-installation checklist

- [ ] CPU, RAM and disk resources match the target environment
- [ ] PostgreSQL 15 is running and the `orion` database and three schemas are created
- [ ] The `orion_app` and `orion_migrator` roles have their permissions granted
- [ ] Redis 7 is reachable and has the `maxmemory-policy allkeys-lru` policy set
- [ ] Kafka 3.6 is running and the `orion.events.v1` topic has at least 12 partitions
- [ ] The `orion.events.v1.dlq` topic is created with a 30-day retention period
- [ ] The `orion-artifacts` bucket exists and has S3v4 credentials
- [ ] Keycloak 24 has the `orion` realm and a confidential client
- [ ] A TLS certificate for `api.orion.example.com` and `console.orion.example.com` is ready
- [ ] Firewall rules allow the traffic listed in the port table
- [ ] A backup schedule is planned according to [Backups](../operations/backups.md)
- [ ] An installation method matching the licence tier has been chosen

## See also

- [Installation](installation/index.md)
- [Configuration](configuration/index.md)
- [Scaling](../operations/scaling.md)
- [Backups](../operations/backups.md)
