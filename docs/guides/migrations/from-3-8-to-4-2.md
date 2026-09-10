# Migrate from 3.8 LTS to 4.2 LTS

Upgrading from the 3.8 LTS line to 4.2.1 is a direct path; it does not require going through 4.0. It does, however, involve breaking changes in the API, the webhook signature format, the Kafka topic names, and the minimum PostgreSQL version. Plan a maintenance window of 3 to 4 hours for an installation with up to 20 million orders; the `core` schema migration rewrites data and scales linearly with the row count.

## Breaking changes

### 1. Removed `order.legacy_total` field

The field is gone from REST responses and from the event envelope. It is replaced by the pair `total_minor` (an integer in the smallest currency unit) and `currency`.

### 2. New webhook signature format

In 3.8, the `X-Orion-Signature` header contained a bare HMAC-SHA256 in hexadecimal form. In 4.2 it has the form `t=<epoch>,v1=<hmac>`, and the signed string is `<epoch>.<raw body>`. Verification with the old code will always fail.

### 3. Cursor pagination by default

The default list mode is cursor pagination (`cursor`, `has_more`). The `page` and `offset` parameters are still accepted, but only when the request also carries `pagination=offset`. Without that parameter, `page=2` is ignored.

### 4. Retired `ORION_LEGACY_*` variables

All variables with the `ORION_LEGACY_` prefix have been removed without aliases. Their presence makes `orion config validate` return `ORN-1004` and the component fails to start.

### 5. PostgreSQL 15 required

Version 3.8 ran on PostgreSQL 13 and 14. Version 4.2 requires PostgreSQL 15: it uses RLS policies with `NULLS NOT DISTINCT` in unique constraints and will not start on an older server.

### 6. Kafka topic `orion.events.v1`

The `orion-events` topic from 3.8 has been replaced by `orion.events.v1`, and the `orion-events-dead` DLQ by `orion.events.v1.dlq`. The event envelope has a new `version` field, and `tenant_id` has been renamed to `tenant`.

### 7. Mandatory `audit` schema and the `admin` scope

Auditing is no longer optional (`ORION_LEGACY_AUDIT_DISABLED` has been removed): the application role must hold `INSERT` on the `audit` schema. At the same time the `superuser` OAuth2 scope was removed; clients must request `admin` instead, otherwise they receive `ORN-2003`.

## Pre-migration checklist

- [ ] PostgreSQL upgraded to 15 (`SELECT version();`), Redis to 7, Kafka to 3.6, Keycloak to 24.
- [ ] Backup of the `orion` database taken and **restored into a test environment**.
- [ ] Topics `orion.events.v1` and `orion.events.v1.dlq` created with 24 partitions.
- [ ] All integrations prepared for the new `X-Orion-Signature` format.
- [ ] OAuth2 clients switched from the `superuser` scope to `admin`.
- [ ] `ORION_LEGACY_*` variables removed from manifests and configuration files.
- [ ] `orion db migrate --dry-run --target 4.2.1` executed against a copy of production.
- [ ] Maintenance window announced to tenants, status published at `https://status.orion.example.com`; on-call team and P1 channel confirmed at `https://support.nimbus.example.com`.

## Procedure

1. **Freeze writes.** Enable read-only mode in `orion-gateway`.

    ```bash
    kubectl -n orion set env deployment/orion-gateway ORION_MAINTENANCE_READ_ONLY=true
    kubectl -n orion rollout status deployment/orion-gateway
    ```

    Checkpoint: `POST /v1/orders` returns 409 with code `ORN-4005`.

2. **Drain the queue.** Wait until `orion_queue_lag` drops to zero for `orion-events` and holds that value for 2 minutes.

3. **Stop the components.**

    ```bash
    kubectl -n orion scale deployment/orion-scheduler --replicas=0
    kubectl -n orion scale deployment/orion-worker --replicas=0
    kubectl -n orion scale deployment/orion-core --replicas=0
    kubectl -n orion scale deployment/orion-ledger --replicas=0
    ```

4. **Point-in-time backup.** Checkpoint: `pg_restore --list` can read the file header.

    ```bash
    pg_dump --format=custom --dbname=orion --file=/backup/orion-pre-4.2.dump
    ```

5. **Swap the images.**

    ```bash
    helm repo add nimbus https://charts.nimbus.example.com
    helm repo update
    helm upgrade orion nimbus/orion --version 4.2.1 \
      --namespace orion --values values-prod.yaml --wait --timeout 15m
    ```

6. **Validate the configuration.** Checkpoint: no warnings about `ORION_LEGACY_*` and no `ORN-1004`.

    ```bash
    orion config validate --file /etc/orion/orion.yaml
    ```

7. **Migrate the schema.**

    ```bash
    orion db migrate status
    orion db migrate --dry-run --target 4.2.1 > /tmp/migration-4.2.1.sql
    orion db migrate --target 4.2.1 --lock-timeout 60s
    orion db migrate status
    ```

    Checkpoint: `status` reports schema version `4.2.1` and zero pending migrations.

8. **Start the components.**

    ```bash
    kubectl -n orion scale deployment/orion-ledger --replicas=2
    kubectl -n orion scale deployment/orion-core --replicas=3
    kubectl -n orion scale deployment/orion-worker --replicas=8
    kubectl -n orion scale deployment/orion-scheduler --replicas=1
    ```

    Checkpoint: all pods `Ready`, no `ORN-5002` in the logs.

9. **Disable read-only mode.**

    ```bash
    kubectl -n orion set env deployment/orion-gateway ORION_MAINTENANCE_READ_ONLY=false
    ```

10. **Observe for 60 minutes.** Watch `orion_http_requests_total` (the 5xx share), `orion_queue_lag`, and `orion_events_dlq_total`.

## Field mapping

| 3.8 field | 4.2 field | Notes |
| --- | --- | --- |
| `order.legacy_total` | `order.total_minor` + `order.currency` | decimal value -> integer in minor units |
| `order.tenant_id` | `order.tenant` | also in the event envelope |
| `order.state` | `order.status` | values unchanged |
| `payment.txn_ref` | `payment.provider_reference` | max. length 128 characters |
| `customer.email_address` | `customer.email` | RFC 5322 validation |
| `event.body` | `event.data` | `event.version` added, value `1` |
| `webhook.secret_hex` | - | removed; the secret is visible only on rotation |

## Environment variable mapping

| 3.8 variable | 4.2 variable | Notes |
| --- | --- | --- |
| `ORION_LEGACY_DB_URL` | `ORION_DATABASE_URL` | the `postgresql://` scheme is required |
| `ORION_LEGACY_KAFKA_TOPIC` | `ORION_EVENTS_TOPIC` | defaults to `orion.events.v1` |
| `ORION_LEGACY_KAFKA_DLQ` | `ORION_EVENTS_DLQ_TOPIC` | defaults to `orion.events.v1.dlq` |
| `ORION_LEGACY_AUDIT_DISABLED` | - | removed, auditing is mandatory |
| `ORION_LEGACY_PAGE_MODE` | `ORION_API_PAGINATION` | values `cursor` (default) or `offset` |
| `ORION_LEGACY_SIGNATURE_V0` | - | removed along with the old signature format |
| `ORION_LEGACY_REDIS_HOST` | `ORION_REDIS_URL` | a full URL instead of a host and port |

## Rollback plan

!!! danger "The 4.2.1 migration is partly irreversible"
    Step 7 drops the `legacy_total` column and rewrites amounts as integers.
    There is no backward script. Returning to 3.8 requires restoring the backup from step 4, which means
    ==losing every write made after the maintenance window ended==.

| Failure after step | Action |
| --- | --- |
| 1-6 | restore the previous chart (`helm rollback orion`), disable read-only mode; the database is untouched |
| 7 (migration interrupted mid-transaction) | the transaction rolls back automatically; repeat `orion db migrate` once the cause is removed |
| 7 (migration completed) | only a `pg_restore` of the backup; there is no backward schema path |
| 8-10 | stay on 4.2.1 and diagnose the component; do not roll the database back |

```bash title="Restoring the backup (last resort)"
kubectl -n orion scale deployment --all --replicas=0
dropdb orion && createdb orion
pg_restore --dbname=orion --clean --if-exists /backup/orion-pre-4.2.dump
helm rollback orion --namespace orion
```

## Post-migration verification

- [ ] The schema version is `4.2.1`.
- [ ] A new order moves through `draft` -> `pending_payment` -> `paid`.
- [ ] A webhook is delivered and its signature verifies with the new format.
- [ ] `orion_events_dlq_total` is not growing.
- [ ] No `ORN-1004` entries related to `legacy_total` in the `orion-gateway` logs.

```sql title="Checking the amount rewrite"
-- expected: 0
SELECT count(*) FROM core.orders WHERE total_minor IS NULL OR currency IS NULL;
-- expected: 4.2.1
SELECT schema_version FROM core.migrations ORDER BY applied_at DESC LIMIT 1;
```

```bash title="API check"
curl -sS "https://api.orion.example.com/v1/orders?limit=1" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Orion-Tenant: ten_01HQ8ZV3KXN4M2T9YB7C6D" | jq '.data[0] | {id, status, total_minor, currency}'
```

## See also

- [Migrations and upgrades](index.md)
- [Signature verification](../../api/webhooks/signature-verification.md)
- [Pagination](../../api/rest/pagination.md)
- [Backups](../../operations/backups.md)
