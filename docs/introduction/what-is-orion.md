# What is Orion Platform

Orion Platform is a multi-tenant platform for handling orders, payments and events for online shops and marketplaces. It is built by Nimbus Software sp. z o.o. The platform exposes a REST API, an event bus and an operations console, and it runs both as a self-hosted deployment and in the vendor's cloud.

## Key capabilities

- Recording and versioning orders with a full history of status transitions `draft` → `pending_payment` → `paid` → `fulfilled` → `closed`.
- A separate payment ledger (`orion-ledger`) that tracks authorizations, captures, refunds and settlements per tenant.
- An idempotent HTTP API driven by the `Idempotency-Key` header, so retrying a request never creates a duplicate order.
- An event bus on Apache Kafka (topic `orion.events.v1`) with a dead letter queue `orion.events.v1.dlq` and the `orion events replay` mechanism.
- Webhooks signed with the `X-Orion-Signature` header for ten domain event types.
- Tenant data isolation (`ten_...`) at the level of queries, API keys and rate limits.
- OAuth2 client credentials authentication through Keycloak 24 (realm `orion`) or API keys in the `X-Orion-Key` header.
- Observability: Prometheus metrics with the `orion_` prefix, structured JSON logs, correlation through `X-Orion-Request-Id`.
- The `orion-console` panel for browsing orders, retrying webhooks and inspecting queue lag.

## Typical use cases

### Online shop with a single sales channel

The shop keeps its own frontend and product catalog while Orion takes over the order lifecycle and settlements. The frontend calls `POST /v1/orders` and, once the payment is authorized, receives an `order.paid` webhook and releases the goods for shipping.

### Marketplace with many sellers

Every seller is a separate tenant (`ten_...`) with its own API keys and limits. The payment ledger keeps settlements split per tenant, and the marketplace operator has access to an aggregate view through the `admin` scope.

### Migration from a monolithic system

Orion runs alongside the old system: new orders go to `orion-core`, while events from the `orion.events.v1` topic are consumed by an adapter on the legacy side. Once the migration is complete, the adapter is switched off.

## What Orion does not do

The platform deliberately leaves several areas out of scope and integrates with specialized systems instead.

| Orion **is not** | What it does instead |
| --- | --- |
| A CMS or a product catalog | Stores only the SKU reference and the line item name passed in the request |
| A warehouse management system (WMS) | Publishes `order.paid` and `order.fulfilled` events that the WMS consumes |
| A payment gateway or an acquirer | Calls the gateway and persists its result in the `orion-ledger` ledger |
| An accounting or ERP system | Provides a ledger export to the `orion-artifacts` bucket |
| An email and SMS delivery tool | Emits events that an external messaging system reacts to |

!!! warning "The product catalog stays on your side"
    Orion does not validate that a SKU exists or that its catalog price is correct. The price of an order line item is taken from the request and persisted unchanged. Validate prices before you call `POST /v1/orders`.

## Licence tiers

| Feature | Community | Business | Enterprise |
| --- | --- | --- | --- |
| Number of tenants | 1 | 25 | unlimited |
| Orders per month | 50,000 | 2,000,000 | negotiated |
| API rate limit | 600/min | 600/min per key | configurable |
| Installation with Docker Compose | yes | yes | yes |
| Installation with Helm | yes | yes | yes |
| `OrionCluster` operator | no | no | yes |
| `.deb` / `.rpm` packages | no | yes | yes |
| Multi-tenancy with schema isolation | no | yes | yes |
| Vendor support | community | P2/P3 on business days | P1 to P3 according to the SLA |
| LTS release support | 12 months | until the EOL date | until the EOL date plus extension |
| Price | free | on request | on request |

## Release lifecycle

| Release | Type | Release date | End of support (EOL) | Status |
| --- | --- | --- | --- | --- |
| 3.8 | LTS | 2024-04-15 | 2026-03-31 | security fixes only |
| 4.0 | standard | 2025-02-10 | 2026-09-30 | limited support |
| 4.2 | LTS | 2025-09-30 | 2027-06-30 | recommended, actively developed |

!!! note "Why LTS"
    LTS releases receive security and bug fixes for the whole support period without backwards-incompatible changes to the REST API or to the `orion.events.v1` event schema. Standard releases may change contracts between minor versions. Use the **4.2 LTS** line for production deployments.

!!! danger "Release 3.8 is approaching EOL"
    After 2026-03-31 the 3.8 line will receive no fixes at all, including critical ones. Plan the upgrade according to [Migration from 3.8 to 4.2](../guides/migrations/from-3-8-to-4-2.md).

??? info "Patch release numbering"
    The `MAJOR.MINOR.PATCH` number matches the container image tag, for example `registry.nimbus.example.com/orion/orion-gateway:4.2.1`. All components of a single instance must carry an identical tag, because mixing patch versions is not supported.

## See also

- [Architecture](architecture.md)
- [Glossary](glossary.md)
- [Getting started](../getting-started/index.md)
- [Changelog](../changelog.md)
