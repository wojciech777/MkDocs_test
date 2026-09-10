# Multi-tenancy

Orion Platform 4.2 LTS is multi-tenant by design: a single installation serves many independent organizations that cannot see each other's data. Isolation rests on a shared database with a `tenant_id` column and RLS policies in PostgreSQL 15, reinforced by separate prefixes in Kafka and S3. This page describes the isolation model, the tenant lifecycle, and the limits and usage metering.

## Isolation model

Every table in the `core`, `ledger`, and `audit` schemas has a `tenant_id` column. `orion-gateway` sets the session variable `app.tenant_id` from the `ten` claim in the token or from the `X-Orion-Tenant` header, and PostgreSQL enforces the filtering through an RLS policy.

```sql title="RLS policy for core.orders" hl_lines="4 9 10"
ALTER TABLE core.orders ENABLE ROW LEVEL SECURITY;
ALTER TABLE core.orders FORCE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON core.orders
    USING (tenant_id = current_setting('app.tenant_id', true))
    WITH CHECK (tenant_id = current_setting('app.tenant_id', true));

GRANT SELECT, INSERT, UPDATE ON core.orders TO orion_app;
REVOKE ALL ON core.orders FROM PUBLIC;
```

`FORCE ROW LEVEL SECURITY` matters: without it the table owner bypasses the policy, which for migrations run under the owner role would grant access to every tenant.

!!! danger "A query without tenant_id is a data leak"
    Every hand-written query, whether in a migration, a reporting script, or a diagnostic tool,
    must include a `tenant_id = ...` condition or run in a session with `app.tenant_id` set.
    A query such as `SELECT * FROM core.orders WHERE status = 'paid'` executed under the owner role
    returns data for ==all tenants==. For analysis, use only the `orion_readonly` role,
    which is subject to RLS.

```sql title="A safe diagnostic session"
SET ROLE orion_readonly;
SET app.tenant_id = 'ten_01HQ8ZV3KXN4M2T9YB7C6D';
SELECT id, status, total_minor FROM core.orders WHERE status = 'paid' LIMIT 10;
```

## Creating a tenant

=== "CLI"

    ```bash
    orion tenant create --id ten_01HQ8ZV3KXN4M2T9YB7C6D \
      --name "Nordis Store" --profile prod \
      --owner-email admin@nordis.example.com

    orion config validate --file /etc/orion/orion.yaml
    ```

=== "API"

    ```bash
    curl -sS -X POST https://api.orion.example.com/v1/tenants \
      -H "Authorization: Bearer $TOKEN" \
      -H "Content-Type: application/json" \
      -H "Idempotency-Key: tenant-nordis-2026-02-11" \
      -d '{
            "name": "Nordis Store",
            "profile": "prod",
            "owner_email": "admin@nordis.example.com",
            "default_currency": "PLN"
          }'
    ```

=== "Response"

    ```json
    {
      "id": "ten_01HQ8ZV3KXN4M2T9YB7C6D",
      "name": "Nordis Store",
      "profile": "prod",
      "status": "active",
      "owner": "usr_01HQ8ZV3KXN4M2T9YB7C6D",
      "limits": {
        "requests_per_minute": 600,
        "webhooks": 20,
        "event_retention_days": 30,
        "attachment_size_mb": 25
      },
      "created_at": "2026-02-11T10:02:44Z"
    }
    ```

Creating a tenant requires the `admin` scope and the `owner` role at the installation level. The operation is idempotent on `Idempotency-Key`; a repeat with a different body returns `ORN-4002`.

## Selecting the tenant in a request

The tenant context is resolved in this order:

1. The `ten` claim in the JWT - binding, it cannot be overridden.
2. The `X-Orion-Tenant` header - used when the token carries no `ten` (multi-tenant tokens issued to operator tooling).
3. Neither present -> `ORN-1004`.

| Situation | Result |
| --- | --- |
| token with `ten`, no header | context from the token |
| token with `ten`, matching header | context from the token |
| token with `ten`, different header | `ORN-2003` |
| token without `ten`, header present | context from the header, entry in `audit` |
| token without `ten`, no header | `ORN-1004` |

!!! warning "The header does not escalate permissions"
    Passing `X-Orion-Tenant` for another tenant with a token that has no `ten` claim works only when
    the subject holds the `owner` role at the installation level. For ordinary OAuth2 clients it results in
    `ORN-2003`.

## Per-tenant limits

| Limit | `dev` profile | `staging` profile | `prod` profile |
| --- | --- | --- | --- |
| Requests per minute | 60 | 60 | 600 (burst 100) |
| Number of webhooks | 5 | 10 | 20 |
| Event retention (days) | 7 | 14 | 30 |
| Max. attachment size | 5 MB | 10 MB | 25 MB |
| Active API keys | 3 | 5 | 25 |
| OAuth2 clients | 2 | 5 | 50 |

Limits are raised by filing a request at `https://support.nimbus.example.com`. Exceeding the rate limit returns `ORN-3001` with the `X-RateLimit-Reset` header; the details are described in [rate limits](../api/rest/rate-limits.md).

## Isolation in Kafka and S3

Kafka
:   All tenants share the `orion.events.v1` topic. Separation is provided by the `tenant` field
    in the envelope and by the partitioning key, which contains the order identifier and is globally
    unique. `orion-worker` rejects an event whose `tenant` does not exist or is suspended.

S3 / MinIO
:   A shared `orion-artifacts` bucket with a per-tenant prefix:
    `orion-artifacts/ten_01HQ8ZV3KXN4M2T9YB7C6D/invoices/2026/02/`.
    The access policy binds the prefix to the IAM role assigned to the tenant, so an object outside the prefix
    is unreachable even if its path is known.

Redis
:   Cache and rate limit counter keys use the prefix `orion:{tenant}:`, for example
    `orion:ten_01HQ8ZV3KXN4M2T9YB7C6D:ratelimit:ok_live_9F4KQ2ZT`.

??? info "Why not a topic per tenant"
    A separate Kafka topic for each tenant raises the partition count linearly with the number of tenants
    and quickly exhausts broker limits (metadata, open files, rebalance time). With 500 tenants
    that would mean 12,000 partitions instead of 24.

## Reporting and usage metering

`orion-scheduler` aggregates usage hourly into the `ledger.usage_hourly` table.

| Column | Description |
| --- | --- |
| `tenant_id` | tenant |
| `bucket_start` | start of the hour in UTC |
| `api_requests` | number of requests counted toward the limit |
| `orders_created` | number of orders leaving `draft` |
| `payments_captured_minor` | sum of captured amounts |
| `webhook_deliveries` | number of deliveries, including retries |
| `storage_bytes` | size of the objects under the tenant's prefix |

```sql title="Monthly usage for a tenant"
SELECT date_trunc('month', bucket_start) AS month,
       sum(api_requests)                 AS requests,
       sum(orders_created)               AS orders,
       sum(payments_captured_minor)/100.0 AS captured
FROM ledger.usage_hourly
WHERE tenant_id = 'ten_01HQ8ZV3KXN4M2T9YB7C6D'
  AND bucket_start >= date_trunc('month', now()) - interval '3 months'
GROUP BY 1
ORDER BY 1;
```

Data from `ledger.usage_hourly` is also exposed as the metrics `orion_http_requests_total` and `orion_orders_processed_total` with a `tenant` label, which lets you build per-tenant dashboards.

## Migrating a tenant between installations

1. Suspend the tenant (`status: suspended`): writes return `ORN-4005`, reads keep working.
2. Wait for `orion_queue_lag` to drop to zero and for the DLQ to be empty for that tenant.
3. Export the data: `pg_dump` filtered by `tenant_id`, a copy of the S3 prefix, the list of webhooks and API keys.
4. Create the tenant on the target installation with the same `ten_...` (`orion tenant create --id`).
5. Import the data and run `orion db migrate status` to confirm the schema versions match.
6. Repoint the integration DNS or update the base URL on the tenant's side.
7. Issue new API keys and OAuth2 client secrets; the old ones are not transferred.
8. Lift the suspension on the target installation and leave the tenant suspended on the source one for 14 days.

!!! tip "Keep the tenant identifier"
    The `ten_...` identifier appears in event envelopes, S3 prefixes, and integration logs.
    Changing the identifier during a migration forces a rewrite of historical data and breaks correlation
    in the tenant's own tooling.

## See also

- [Roles and permissions](authentication/roles-and-permissions.md)
- [Event processing](events/index.md)
- [Rate limits](../api/rest/rate-limits.md)
- [Security](../operations/security.md)
