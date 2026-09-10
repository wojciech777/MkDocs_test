# REST API basics

The Orion Platform REST API is exposed by the `orion-gateway` component (port `8080`) and is the only public entry point to the order engine and the payment ledger. The interface is resource oriented and accepts and returns `application/json` encoded as UTF-8. This page describes the elements common to all resources: addresses, headers, methods, idempotency, and health probes.

## Base address

```text title="Base addresses"
https://api.orion.example.com/v1            # production
https://api.sandbox.orion.example.com/v1    # sandbox
```

All paths in the resource documentation are given in full form, for example `POST /v1/orders`. Connections other than TLS 1.2+ are rejected at the edge layer.

## Authentication

Two mechanisms are supported:

- OAuth2 client credentials, `POST /oauth/token`, a Bearer token with a TTL of `3600` s, scopes `orders:read`, `orders:write`, `payments:read`, `payments:write`, `customers:read`, `customers:write`, `webhooks:write`, `admin`.
- An API key in the `X-Orion-Key` header (`ok_live_...` or `ok_test_...`).

Full description: [OAuth2](../../guides/authentication/oauth2.md) and [API keys](../../guides/authentication/api-keys.md).

## Request headers

| Header | Required | Description |
| --- | --- | --- |
| `Authorization` | yes, with OAuth2 | `Bearer <token>` |
| `X-Orion-Key` | yes, with an API key | `ok_live_...` / `ok_test_...` key |
| `Content-Type` | for `POST`/`PATCH` | `application/json` |
| `Idempotency-Key` | recommended for `POST` | unique string of up to 255 characters |
| `X-Orion-Api-Revision` | no | revision date, for example `2026-04-01` |
| `X-Orion-Tenant` | `admin` scope only | a `ten_...` identifier |
| `X-Orion-Request-Id` | no | your own correlation identifier |

## Response headers

| Header | Description |
| --- | --- |
| `X-Orion-Request-Id` | the request identifier echoed back or generated server side |
| `X-RateLimit-Limit` | the limit in the current window |
| `X-RateLimit-Remaining` | number of remaining requests |
| `X-RateLimit-Reset` | Unix timestamp at which the window resets |
| `Retry-After` | number of seconds to wait on `429` and `503` |
| `X-Orion-Api-Revision` | the revision used to handle the request |

!!! tip "Always record `X-Orion-Request-Id`"
    This identifier is the only key by which support can find your request in the logs. See [Logs](../../operations/monitoring/logs.md).

## Methods and semantics

| Method | Use | Idempotent | Request body |
| --- | --- | --- | --- |
| `GET` | reading a resource or a list | yes | no |
| `POST` | creating a resource or running an action | not by definition, see below | yes |
| `PATCH` | partial update | yes | yes |
| `DELETE` | deletion or anonymization | yes | no |

The `PUT` and `HEAD` methods are not supported and return `405 Method Not Allowed`.

## Idempotency of POST requests

The `Idempotency-Key` header lets you safely retry resource creating requests after a network error or a timeout.

- The key is remembered for **24 hours** together with a fingerprint of the request body and the result.
- Repeating the same key with an **identical** body returns the stored response, including the original `201` code.
- Repeating the same key with a **different** body fails with `ORN-4002` (`409 Conflict`).
- The key is scoped to the pair: API key plus resource path.

=== "curl"

    ```bash
    curl -X POST https://api.orion.example.com/v1/orders \
      -H "X-Orion-Key: ok_live_9f2c41ab7d5e" \
      -H "Idempotency-Key: order-2026-04-12-8831" \
      -H "Content-Type: application/json" \
      -d '{"customer":"cus_01HQ8ZV3KXN4M2T9YB7C6D","currency":"PLN"}'
    ```

=== "Python"

    ```python
    resp = client.post(
        "/v1/orders",
        json={"customer": "cus_01HQ8ZV3KXN4M2T9YB7C6D", "currency": "PLN"},
        headers={"Idempotency-Key": "order-2026-04-12-8831"},
    )
    ```

=== "Java"

    ```java
    OrderCreateParams params = OrderCreateParams.builder()
        .customer("cus_01HQ8ZV3KXN4M2T9YB7C6D")
        .currency("PLN")
        .build();
    Order order = client.orders().create(params, RequestOptions.idempotencyKey("order-2026-04-12-8831"));
    ```

!!! danger "Do not generate a random key on retry"
    The key must stay constant for a single logical business operation. A new key on retry will create a second order.

## Response expansion

The `expand` parameter replaces relation identifiers with full objects. At most two levels of nesting are allowed.

```bash title="Expanding the customer and the line items"
curl "https://api.orion.example.com/v1/orders/ord_01HQ8ZV3KXN4M2T9YB7C6D?expand=customer,items" \
  -H "X-Orion-Key: ok_live_9f2c41ab7d5e"
```

Expansion increases the response time and counts double against the query point limit. An unknown expansion path returns `ORN-1004`.

## Field selection

The `fields` parameter narrows the response down to the listed fields. The `id` and `object` fields are always returned.

```bash title="Narrowing the response"
curl "https://api.orion.example.com/v1/orders?fields=id,status,amount_total" \
  -H "X-Orion-Key: ok_live_9f2c41ab7d5e"
```

## Partial updates with PATCH

`PATCH` follows JSON Merge Patch semantics (RFC 7386):

- an omitted field stays unchanged,
- a field set to `null` is removed,
- the `metadata` object is merged key by key rather than replaced.

```json title="PATCH /v1/customers/cus_01HQ8ZV3KXN4M2T9YB7C6D request" hl_lines="2 4"
{
  "phone": null,
  "metadata": {
    "segment": "b2b"
  }
}
```

## Health probes

Probes require no authentication and do not count against the rate limits.

### GET /healthz

Liveness probe for the `orion-gateway` process. It does not check dependencies.

```json title="200 OK response"
{
  "status": "ok",
  "component": "orion-gateway",
  "version": "4.2.1"
}
```

### GET /readyz

Readiness probe. It verifies PostgreSQL, Redis, and Kafka. It returns `503` with `ORN-5002` if any dependency is unavailable.

```json title="503 Service Unavailable response"
{
  "status": "degraded",
  "checks": {
    "postgres": "ok",
    "redis": "ok",
    "kafka": "unavailable"
  }
}
```

## HTTP status codes

| Code | Meaning | Typical error class |
| --- | --- | --- |
| `200` | operation completed | — |
| `201` | resource created | — |
| `202` | accepted for asynchronous processing | — |
| `204` | completed, no content | — |
| `400` | malformed request | `ORN-1xxx` |
| `401` | missing or invalid credentials | `ORN-2xxx` |
| `403` | missing scope or permission | `ORN-2xxx` |
| `404` | resource does not exist in the tenant context | `ORN-1xxx` |
| `405` | method not supported | — |
| `409` | state or idempotency conflict | `ORN-4xxx` |
| `422` | syntactically valid request, rejected semantically | `ORN-1xxx` |
| `429` | rate limit exceeded | `ORN-3xxx` |
| `500` | internal error | `ORN-5xxx` |
| `503` | dependency unavailable | `ORN-5xxx` |
| `504` | timeout | `ORN-5xxx` |

## See also

- [Error codes](error-codes.md)
- [Rate limits](rate-limits.md)
- [Pagination, filtering, and sorting](pagination.md)
- [Orders](resources/orders.md)
