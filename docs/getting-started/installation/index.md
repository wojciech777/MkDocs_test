# Installation

Orion Platform 4.2 LTS can be deployed in four ways: with Docker Compose, with a Helm chart, with the Kubernetes operator, or from system packages. All methods install the same set of components in version 4.2.1 and use the same `orion.yaml` configuration. The choice depends on the target environment, the acceptable deployment time and the license variant.

## Installation methods

| Method | When to use | Deployment time | License variants |
| --- | --- | --- | --- |
| [Docker Compose](docker.md) | `dev` environment, workshops, demos, integration tests | 20-40 min | Community, Business, Enterprise |
| [Helm](kubernetes/helm.md) | `staging` and `prod` in an existing cluster, GitOps-managed deployments | 1-2 h | Community, Business, Enterprise |
| [Operator](kubernetes/operator.md) | `prod` with multiple instances, automatic migrations and backups | 1-3 h | Enterprise only |
| [System packages](bare-metal.md) | physical or virtual machines without containers, regulated environments | 2-4 h | Business, Enterprise |

## Support matrix for version 4.2

| Method | Installation | Upgrade without downtime | Automatic migrations | Backups in the lifecycle | Multi-tenancy |
| --- | --- | --- | --- | --- | --- |
| Docker Compose | yes | no | yes (`dev` profile) | no | yes |
| Helm | yes | yes | as a `pre-upgrade` hook | no | yes |
| Operator | yes | yes | yes, driven by the `Migrating` phase | yes | yes |
| `.deb` / `.rpm` packages | yes | no | no | no | yes |

## What to prepare regardless of the method

- Working external dependencies: PostgreSQL 15, Redis 7, Kafka 3.6, S3/MinIO, Keycloak 24. See [Requirements](../requirements.md).
- Credentials for the `registry.nimbus.example.com` registry (the images are not public).
- The `ORION_DB_URL` value and the broker list in `ORION_KAFKA_BROKERS`.
- The chosen profile: `dev`, `staging` or `prod`. See [Configuration profiles](../configuration/profiles.md).
- The webhook signing secret and the key used to encrypt sensitive data in the database.

!!! danger "Do not mix installation methods"
    A single Orion instance must be managed by exactly one method. Using the Helm chart and the `OrionCluster` operator in parallel in the same namespace leads to a fight over resource ownership: the operator recreates objects deleted by Helm, and Helm marks them as unmanaged on the next `helm upgrade`. A typical result is two `orion-scheduler` leaders running at once and billing jobs executed twice.

!!! note "Switching the installation method"
    You switch between methods by fully stopping the old installation, taking a database backup and starting the new installation against the same `orion` database. Data in PostgreSQL, Kafka and S3 is independent of the deployment method, so the migration does not require exporting application data.

??? info "Verification after every method"
    Whichever method you choose, the installation is considered complete when:

    - `GET /healthz` on port 8080 returns `"status": "ok"` for all dependencies,
    - `orion db migrate` reports schema version `4.2.1`,
    - `orion config validate` finishes without errors,
    - the `orion_queue_lag` metric is readable on port 9100.

## See also

- [Requirements](../requirements.md)
- [Install with Docker](docker.md)
- [Deploy on Kubernetes](kubernetes/index.md)
- [Install on a host](bare-metal.md)
