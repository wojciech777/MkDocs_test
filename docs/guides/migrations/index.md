# Migrations and upgrades

Orion Platform is released in version lines, some of which are marked as LTS and receive extended support. An upgrade covers three things: the component images, the database schema, and the `orion.yaml` configuration. This page describes the versioning rules, the supported upgrade paths, and the `orion db migrate` tool.

## Versioning

Version numbers have the form `MAJOR.MINOR.PATCH`. The currently supported lines are:

| Version | Type | End of support |
| --- | --- | --- |
| 3.8 | LTS | 2026-03-31 |
| 4.0 | standard | 2026-09-30 |
| 4.2 | LTS | 2027-06-30 |

A breaking change is:

- removing a field from a REST response or changing its meaning,
- changing the default value of a configuration parameter in a way that affects behavior,
- removing an `ORION_*` environment variable without an alias,
- renaming a Kafka topic or changing the event envelope format,
- raising the minimum version of an external dependency (for example PostgreSQL).

It is **not** a breaking change to add a new field to a response, a new event type, a new OAuth2 scope, or a new error code within the existing `ORN-*` family.

!!! note "Patch releases"
    Within `4.2.x` we introduce no breaking changes and no irreversible migrations. Upgrading
    from `4.2.0` to `4.2.1` amounts to swapping the image and running `orion db migrate`.

## Supported upgrade paths

| From version | To version | Intermediate steps | Notes |
| --- | --- | --- | --- |
| 4.2.0 | 4.2.1 | none | rolling upgrade, no downtime |
| 4.0.x | 4.2.1 | none | requires a 30 min maintenance window |
| 3.8.x | 4.2.1 | none (direct path) | see [migrate from 3.8 to 4.2](from-3-8-to-4-2.md) |
| 3.6.x | 4.2.1 | 3.8 LTS | two-stage upgrade |
| 3.4.x or older | 4.2.1 | 3.8 LTS, then 4.2 | data migration requires vendor support |

Upgrading while skipping a required intermediate step is blocked: `orion db migrate` aborts with code `ORN-4005` and does not modify the schema.

## Maintenance window and read-only mode

Before a schema migration, switch tenants into read-only mode. `orion-gateway` then rejects write operations with code `ORN-4005` while keeping reads and the console available.

```bash title="Read-only mode"
orion config validate --file /etc/orion/orion.yaml
kubectl -n orion set env deployment/orion-gateway ORION_MAINTENANCE_READ_ONLY=true
kubectl -n orion rollout status deployment/orion-gateway
```

The order in which components are shut down and started:

1. `orion-gateway` -> read-only mode (do not stop it).
2. `orion-scheduler` -> scale to 0 replicas.
3. `orion-worker` -> wait until `orion_queue_lag` drops to zero, then 0 replicas.
4. `orion-core`, `orion-ledger` -> 0 replicas.
5. Schema migration.
6. Start the components in reverse order.

!!! warning "Stop orion-worker third"
    Stopping `orion-worker` before the queue has drained does not lose events (offsets
    stay in Kafka), but it lengthens the maintenance window: after the restart the worker has to process the backlog
    against the old and the new projection schema at the same time.

## orion db migrate

```bash title="Order of invocations"
orion db migrate status
orion db migrate --dry-run --target 4.2.1
orion db migrate --target 4.2.1
orion db migrate status
```

| Command | Action |
| --- | --- |
| `orion db migrate status` | lists applied and pending migrations and the current schema version |
| `orion db migrate --dry-run` | prints the SQL that would run without modifying the database |
| `orion db migrate --target <version>` | applies migrations up to the given version |
| `orion db migrate --lock-timeout 30s` | how long to wait for a table lock |

Migrations operate on the `core`, `ledger`, and `audit` schemas of the `orion` database in PostgreSQL 15. Each runs inside a transaction, except for the creation of `CONCURRENTLY` indexes, which require autocommit mode and are marked in the `--dry-run` output with the annotation `-- non-transactional`.

## Rolling back changes

Reversible
:   Swapping the image back to the previous patch version, `orion.yaml` configuration changes, added
    optional columns, new indexes, new Kafka topics.

Irreversible
:   Dropping a column, changing a column type with a loss of precision, rewriting data into a new format,
    increasing the partition count of a Kafka topic, migrating the webhook signature format.

!!! danger "Without a backup there is no rollback"
    Irreversible migrations have no backward script. The only way back is to restore
    a backup taken before the migration, which means losing the data written after it ran.
    Take the backup and ==verify== it immediately before the maintenance window starts, as described
    in [backups](../../operations/backups.md).

## Subpages

| Page | Contents |
| --- | --- |
| [Migrate from 3.8 LTS to 4.2 LTS](from-3-8-to-4-2.md) | breaking changes, checklist, procedure, rollback plan |

## See also

- [Backups](../../operations/backups.md)
- [Changelog](../../changelog.md)
- [Requirements](../../getting-started/requirements.md)
- [Support](../../support.md)
