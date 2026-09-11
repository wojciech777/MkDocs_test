# Changelog

Orion Platform releases, newest first. The numbering follows semantic
versioning: breaking changes land only in major releases, and patch releases
(`x.y.Z`) are always safe to deploy without integration changes.

!!! note "Labels"

    **Added** for a new feature · **Changed** for a behavior change ·
    **Deprecated** for something that still works but disappears in the next
    major release · **Removed** for a breaking change · **Fixed** for a bug fix ·
    **Security** for a vulnerability fix.

## 4.2.1 - 2026-08-18

Patch release. Recommended for every 4.2 installation. Editing by IDE rehearsal. Edit by Github Web Api rehearsal.

**Fixed**

- `orion-worker` did not release PostgreSQL connections after a failed webhook
  delivery, which exhausted the pool after a few hours (`orion_db_pool_waiting`
  grew linearly).
- The pagination cursor returned `has_more: true` for the last full page when the
  record count was an exact multiple of `limit`.
- Orders in the `on_hold` status were incorrectly expired by `orion-scheduler`
  after 30 minutes.
- `orion db migrate --dry-run` returned exit code 0 even when the migration plan
  contained an irreversible step.

**Security**

- Base image dependency update (CVE in a compression library); affects all
  `4.2.0` images.
- The `X-Orion-Key` header is no longer echoed in validation messages written to
  the logs at the `DEBUG` level.

## 4.2.0 - 2026-06-02 · LTS

Long-term support release. Supported until **2027-06-30**.

**Added**

- Kubernetes operator with the `OrionCluster` resource in version `v1alpha2`:
  database migrations as part of reconciliation, self-healing, and scheduled
  backups. See [Install with the operator](getting-started/installation/kubernetes/operator.md).
- Tenant isolation based on RLS policies in PostgreSQL, plus the
  `X-Orion-Tenant` header. See [Multi-tenancy](guides/multi-tenancy.md).
- New event types: `order.expired`, `customer.anonymized`, `webhook.test`.
- Batch mode `POST /v1/orders/batch` (up to 50 items per request).
- The `orion_webhook_delivery_latency_seconds`, `orion_events_dlq_total`, and
  `orion_db_pool_waiting` metrics, along with ready-made alert rules.
- The `orion diag bundle` and `orion events replay` commands.

**Changed**

- Cursor pagination is now the default for every collection; offset parameters
  work only in compatibility mode. See
  [Pagination, filtering, and sorting](api/rest/pagination.md).
- The `v2` webhook signature (HMAC-SHA256) is the default, with a timestamp
  tolerance of 300 s. See [Signature verification](api/webhooks/signature-verification.md).
- The default log level in the `prod` profile is `INFO`, and the log format is
  JSON.
- Rate limits are counted in a sliding window with a burst of 100 instead of a
  fixed window. See [Rate limits](api/rest/rate-limits.md).

**Deprecated**

- The `v1` webhook signature (HMAC-SHA1), scheduled for removal in 5.0.
- The `ORION_LEGACY_*` variables. The mapping to the new names is described in
  the [migration guide](guides/migrations/from-3-8-to-4-2.md).
- The `page` and `per_page` pagination parameters.

**Removed**

- The `order.legacy_total` field, replaced by `amount_total`.
- The `orion.events` Kafka topic, superseded by `orion.events.v1`.
- Support for PostgreSQL 13 and 14.

## 4.0.7 - 2026-04-14

**Fixed**

- `payment.refunded` events could be emitted twice for a partial refund that
  ended with a network error.
- `helm upgrade` did not wait for the migration hook to finish when `--atomic`
  was enabled.

**Security**

- Fixed `redirect_url` validation in the 3-D Secure flow.

## 4.0.0 - 2025-11-20

**Added**

- New order engine with an explicit state machine and the `on_hold` status.
- Dedicated `orion-ledger` component for payments and settlement (ADR-003).
- Idempotency for `POST` requests at the `orion-gateway` level (ADR-002).

**Removed**

- The built-in inventory module. The capability moved to WMS integrations.

## 3.8.12 - 2026-02-10

**Fixed**

- Final patch release of the 3.8 LTS line. Stability fixes for the event
  consumer and report exports.

!!! warning "3.8 LTS is end of life"

    Support for the 3.8 line ended on **2026-03-31**. Installations on 3.8 no
    longer receive security fixes, so plan the
    [migration to 4.2 LTS](guides/migrations/from-3-8-to-4-2.md).

## 3.8.0 - 2025-03-05 · LTS

**Added**

- First release with webhooks signed using a `whsec_...` secret.
- Version 2 of the `orion-console` panel.

## See also

- [Migrations and upgrades](guides/migrations/index.md)
- [Migrate from 3.8 LTS to 4.2 LTS](guides/migrations/from-3-8-to-4-2.md)
- [What is Orion Platform](introduction/what-is-orion.md)
- [Support](support.md)
