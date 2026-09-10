# API

Orion Platform 4.2 LTS exposes three integration surfaces: the synchronous REST API, asynchronous webhooks, and the official SDK libraries. All three share the same data model, the same identifiers, and the same error structure. This section describes the conventions common to the entire programming interface.

## Integration surfaces

| Surface | Nature | When to use it | Documentation |
| --- | --- | --- | --- |
| REST API | synchronous, HTTP/1.1 and HTTP/2 | creating and reading orders, payments, customers | [REST API basics](rest/index.md) |
| Webhooks | asynchronous, `POST` to the receiver address | reacting to state changes without polling | [Webhooks](webhooks/index.md) |
| SDK | client libraries | fast integration with retries and types | [SDK](sdk/index.md) |

## Environments

| Feature | Production | Sandbox |
| --- | --- | --- |
| Base address | `https://api.orion.example.com/v1` | `https://api.sandbox.orion.example.com/v1` |
| API key prefix | `ok_live_` | `ok_test_` |
| Rate limit | 600/min per key | 60/min per key |
| Burst | 100 | 100 |
| Test data | none, live operations | test cards, simulated BLIK |
| Data retention | as agreed in the contract | 30 days, periodic cleanup |

!!! warning "Keys are strictly separated"
    An `ok_test_...` key will never work against the production address, and the other way round you will receive `ORN-2005`. See [Error codes](rest/error-codes.md) for details.

## Versioning and deprecation

- The major version lives in the path: `/v1`. Backward incompatible changes only ever land in a new major version.
- A backward compatible revision is selected with the `X-Orion-Api-Revision` header carrying a date, for example `2026-04-01`. Without the header, the revision assigned to the API key on first use applies.
- Deprecation policy: every revision is supported for at least **12 months** from the announcement of the next one. Announcements are published in the [changelog](../changelog.md).

## Common conventions

Time
:   All timestamps use ISO-8601 in the UTC time zone, for example `2026-04-12T09:31:04Z`.

Amounts
:   Integers in the smallest unit of the currency (cents) in the `amount` field, always together with the `currency` field (`PLN`, `EUR`, `USD`).

Field names
:   `snake_case` in requests and responses. Unknown fields in a request are rejected with a validation error.

Identifiers
:   A resource type prefix plus 22 characters, for example `cus_01HQ8ZV3KXN4M2T9YB7C6D`. Treat them as opaque strings.

Tenant
:   The multi-tenant context is derived from the key or token; the `X-Orion-Tenant` header is reserved for keys with the `admin` scope.

## Pages in this section

- [REST API basics](rest/index.md), headers, methods, idempotency, probes.
- [Pagination, filtering, and sorting](rest/pagination.md)
- [Rate limits](rest/rate-limits.md)
- [Error codes](rest/error-codes.md)
- [Event types](webhooks/event-types.md) and [signature verification](webhooks/signature-verification.md)
- [Java SDK](sdk/java.md), [Python SDK](sdk/python.md)

## See also

- [Authentication](../guides/authentication/index.md)
- [First order](../getting-started/first-order.md)
- [Architecture](../introduction/architecture.md)
