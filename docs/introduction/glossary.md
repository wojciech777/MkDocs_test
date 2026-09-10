# Glossary

The definitions below apply across the whole Orion Platform documentation, in REST API messages and in the `orion-console` panel. The domain terms match the field names in the API, so it is worth aligning them in your own integration as well. If the documentation and a field name in the API disagree, the resource specification wins.

## Domain concepts

authorization
:   A reservation of funds at the payment provider without collecting them. Results in a `payment.authorized` event. An authorization has a limited validity imposed by the provider, and once it expires a new authorization is required.

capture
:   Collection of previously reserved funds, in full or in part. Results in a `payment.captured` event and moves the order to status `paid`.

cart
:   An order in status `draft` that has not yet been sent to payment. Carts are deleted by `orion-scheduler` after an inactivity window that depends on the profile.

customer
:   The party placing orders within a single tenant, identified by the `cus_` prefix. The same person at two tenants is two independent customer records.

ledger
:   An append-only set of financial entries maintained by the `orion-ledger` component in the `ledger` schema of the `orion` database. There is no operation to update or delete an entry.

order
:   A persisted intent to purchase, belonging to one customer and one tenant, identified by the `ord_` prefix. It follows the path `draft` → `pending_payment` → `paid` → `fulfilled` → `closed`, with the side states `on_hold`, `cancelled` and `refunded`. An order's line items become immutable once it leaves status `draft`.

order line item
:   A single row of an order with a SKU reference, a name, a quantity and a unit price supplied by the integration. Orion does not check the price against a catalog, it persists the value received in the request.

payment
:   A record in the `orion-ledger` ledger linked to a single order, identified by the `pay_` prefix. It aggregates authorization, capture and refund operations.

refund
:   Returning captured funds to the customer. Results in a `payment.refunded` event and moves the order to status `refunded`. In the ledger a refund is a separate offsetting entry, not a modification of the original entry.

sales channel
:   A marker for the source of an order within a tenant, for example an own shop, a mobile app or a marketplace. It is used for reporting and for selecting settlement rules; it does not affect the order lifecycle.

tenant
:   A logically isolated space for data and configuration, identified by the `ten_` prefix. Every API request runs in the context of exactly one tenant, determined by the token or the API key. Tenant data is never returned together outside the `admin` scope.

## Technical concepts

API key
:   An authentication secret passed in the `X-Orion-Key` header, in the form `ok_live_...` for production or `ok_test_...` for the sandbox. An alternative to the OAuth2 client credentials flow, bound to a single tenant and a set of scopes.

DLQ (dead letter queue)
:   The `orion.events.v1.dlq` topic, which receives events that cannot be processed after the retries are exhausted. Events in the DLQ are not consumed automatically and require an operator decision.

event
:   An immutable message describing a fact that has already happened, identified by the `evt_` prefix. Published to the Kafka topic `orion.events.v1`. Available types include `order.created`, `order.paid`, `payment.captured` and `customer.updated`.

idempotency
:   The property of a write operation for which a repeated execution with the same `Idempotency-Key` header produces the same effect and the same response as the first execution. A body conflict under the same key returns an error in the `ORN-4xxx` range.

migration
:   A versioned database schema change applied with the `orion db migrate` command. Migrations are forward-only, so rolling back means restoring a backup.

profile
:   A named set of default configuration values: `dev`, `staging` or `prod`. Activated with the `ORION_PROFILE` variable and overridden by the `orion.yaml` file.

queue lag
:   The number of events waiting in the topic partitions to be processed by the `orion-worker` consumer group. Reported by the `orion_queue_lag` metric; a sustained increase means too few replicas or a consumer bug.

rate limit
:   The maximum number of API requests in a one-minute window per key: 600 in production and 60 in the sandbox, with an allowed burst of 100. The current state is returned by the `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers.

realm
:   An identity space in Keycloak 24 containing OAuth2 clients, users and scope mappings. Orion Platform uses the `orion` realm.

replay
:   Reprocessing events from a range of offsets or timestamps, performed with the `orion events replay` command. Used after fixing a consumer bug or after draining the DLQ.

webhook
:   An HTTPS address configured by the tenant to which `orion-worker` delivers events, identified by the `whk_` prefix. Every request is signed with the `X-Orion-Signature` header.

## Abbreviations

| Abbreviation | Expansion | Meaning in Orion |
| --- | --- | --- |
| API | Application Programming Interface | the REST API exposed by `orion-gateway` under `/v1` |
| DLQ | Dead Letter Queue | the `orion.events.v1.dlq` topic |
| OIDC | OpenID Connect | the identity protocol implemented by Keycloak 24 |
| RPO | Recovery Point Objective | the acceptable data loss counted from the last backup |
| RTO | Recovery Time Objective | the acceptable time to restore the service after an outage |
| SLA | Service Level Agreement | support response times: P1 30 min, P2 4 h, P3 1 business day |
| LTS | Long Term Support | a release line with long support, currently 4.2 |
| CRD | Custom Resource Definition | the definition of the `OrionCluster` resource used by the operator |

!!! tip "Field names versus concept names"
    The API uses English naming and `snake_case` notation, for example `order_lines`, `tenant_id`, `captured_amount`. The terms in this glossary are descriptive labels used in the documentation only.

## See also

- [Architecture](architecture.md)
- [Error codes](../api/rest/error-codes.md)
- [Event types](../api/webhooks/event-types.md)
- [Multi-tenancy](../guides/multi-tenancy.md)
