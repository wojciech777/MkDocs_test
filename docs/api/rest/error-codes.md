# Error codes

Orion Platform returns errors in a uniform format regardless of the resource, the method, and the component that detected the problem. Every error carries a code from the `ORN-` family assigned to one of five classes, plus a request identifier that allows correlation with the logs. This page contains the full list of codes, their mapping to HTTP status codes, and guidance on retrying.

## Error object structure

```json title="400 Bad Request response"
{
  "error": {
    "code": "ORN-1004",
    "message": "The 'currency' field is required.",
    "type": "validation_error",
    "field": "currency",
    "request_id": "req_01HQ8ZV3KXN4M2T9YB7C6D",
    "doc_url": "https://docs.orion.example.com/api/rest/error-codes/"
  }
}
```

| Field | Type | Description |
| --- | --- | --- |
| `error.code` | string | an `ORN-xxxx` code, stable across revisions |
| `error.message` | string | a message for the operator, its wording may change |
| `error.type` | string | `validation_error`, `auth_error`, `rate_limit_error`, `conflict_error`, `internal_error` |
| `error.field` | string \| null | name of the request field, if the error concerns a single field |
| `error.request_id` | string | request identifier, the same as `X-Orion-Request-Id` |
| `error.doc_url` | string | link to the relevant documentation page |

!!! danger "Branch your logic on `error.code`"
    Do not parse `error.message`, its wording is localized and subject to change. The only stable contract is `error.code`.

## Error classes

| Class | Meaning | Typical HTTP code | Retryable |
| --- | --- | --- | --- |
| `ORN-1xxx` | request validation | `400`, `404`, `422` | no, fix the request |
| `ORN-2xxx` | authentication and authorization | `401`, `403` | no, fix the credentials |
| `ORN-3xxx` | rate limits | `429` | yes, after `Retry-After` |
| `ORN-4xxx` | state and idempotency conflict | `409` | not without a state change |
| `ORN-5xxx` | internal and dependency errors | `500`, `503`, `504` | yes, with exponential backoff |

## ORN-1xxx, validation

| Code | HTTP | Message | Cause | Recommended action |
| --- | --- | --- | --- | --- |
| `ORN-1001` | `400` | Invalid JSON | the body is not valid JSON or the wrong encoding was used | fix the serialization, send UTF-8 without a BOM |
| `ORN-1004` | `400` | Missing required field | a mandatory field was omitted; the name is in `error.field` | fill in the field and repeat |
| `ORN-1007` | `400` | Invalid currency | a currency outside `PLN`, `EUR`, `USD`, or one not enabled for the tenant | use a currency enabled in the console |
| `ORN-1012` | `400` | Invalid cursor | the cursor is corrupted, expired, or inconsistent with `sort` | start the iteration from the beginning |
| `ORN-1015` | `422` | Unknown field in the request body | a field was sent that is not supported in this revision | remove the field or raise `X-Orion-Api-Revision` |
| `ORN-1019` | `400` | Amount outside the allowed range | `amount` is negative, zero, or above 99,999,999 | send an integer amount in cents |
| `ORN-1023` | `404` | Resource does not exist | the identifier does not belong to the tenant context | check the identifier and the `X-Orion-Tenant` header |

## ORN-2xxx, authentication and authorization

| Code | HTTP | Message | Cause | Recommended action |
| --- | --- | --- | --- | --- |
| `ORN-2001` | `401` | Invalid token | the token has expired, has a bad signature, or belongs to the wrong realm | fetch a new token from `POST /oauth/token` |
| `ORN-2003` | `403` | Missing required scope | the token does not carry the scope, for example `orders:write` | add the scope in the Keycloak client configuration |
| `ORN-2005` | `401` | API key revoked | the key was revoked or used in the wrong environment | generate a new key for the correct address |
| `ORN-2008` | `401` | Missing credentials | both `Authorization` and `X-Orion-Key` were omitted | include one of the authentication mechanisms |
| `ORN-2011` | `403` | Tenant header not allowed | `X-Orion-Tenant` used without the `admin` scope | remove the header or use an administrative key |
| `ORN-2014` | `403` | IP address outside the allow list | the key is restricted to specific CIDR ranges | complete the list in the console |

## ORN-3xxx, limits

| Code | HTTP | Message | Cause | Recommended action |
| --- | --- | --- | --- | --- |
| `ORN-3001` | `429` | Rate limit exceeded | the 60 s window or the hourly limit is exhausted | wait out `Retry-After`, apply jitter |
| `ORN-3004` | `400` | Page size too large | `limit` above 100 | reduce `limit` to 100 |
| `ORN-3007` | `413` | Request body too large | 1 MiB exceeded | split the request or use batch mode |
| `ORN-3009` | `429` | Too many items in the batch | more than 50 items in `POST /v1/orders/batch` | split the batch |
| `ORN-3012` | `429` | Too many concurrent connections | more than 64 connections per key | reduce the connection pool size |

## ORN-4xxx, state conflict and idempotency

| Code | HTTP | Message | Cause | Recommended action |
| --- | --- | --- | --- | --- |
| `ORN-4002` | `409` | Idempotency conflict | the same `Idempotency-Key` with a different body within the 24 h window | use a new key for a new operation |
| `ORN-4005` | `409` | Disallowed status transition | a transition not listed in the order or payment transition table | read the current status and pick an allowed action |
| `ORN-4008` | `409` | Order already paid | an attempt to change the amount or cancel after capture | issue a refund with `POST /v1/payments/{id}/refund` |
| `ORN-4011` | `409` | Duplicate customer email address | the email is already taken within the tenant | read the existing customer using the `email` filter |
| `ORN-4014` | `409` | Refund amount exceeds the captured amount | the total of the refunds is greater than `captured_amount` | reduce the refund amount |
| `ORN-4017` | `409` | Idempotent request in progress | a previous request with this key is still being processed | retry after 1 to 2 seconds |

## ORN-5xxx, internal and dependency errors

| Code | HTTP | Message | Cause | Recommended action |
| --- | --- | --- | --- | --- |
| `ORN-5002` | `503` | Dependency unavailable | PostgreSQL, Redis, Kafka, or S3 is not responding | retry with backoff, check the status page |
| `ORN-5009` | `504` | Timeout | the payment provider or `orion-core` did not respond within the window | check the state of the resource before retrying |
| `ORN-5001` | `500` | Internal error | an unhandled exception in `orion-gateway` | report it to support with the `request_id` |
| `ORN-5005` | `503` | Maintenance in progress | a service window for the component | retry after the time given in `Retry-After` |
| `ORN-5012` | `503` | Event queue backlog | the `orion-worker` consumer lag exceeded the threshold | slow down your write rate, check the runbook |

## Whether to retry a request

| Situation | Retry | Safety condition |
| --- | --- | --- |
| `429` with `ORN-3001` | yes | wait out `Retry-After`, add jitter |
| `503` with `ORN-5002` or `ORN-5005` | yes | exponential backoff, max. 5 attempts |
| `504` with `ORN-5009` | yes, after verification | `GET` the resource first to avoid a duplicate |
| `500` with `ORN-5001` | once | the same `Idempotency-Key` |
| `409` with `ORN-4017` | yes | wait 1 to 2 s |
| `409` with `ORN-4002`, `ORN-4005`, `ORN-4008` | no | requires a change to the request or to the resource state |
| `4xx` with `ORN-1xxx` | no | fix the request |
| `401`, `403` with `ORN-2xxx` | no | fix the credentials or the scopes |

Every retry of a `POST` must use an `Idempotency-Key` header identical to the original request.

## Correlating support reports

The `error.request_id` value is identical to the `X-Orion-Request-Id` response header and appears in every log entry generated while handling the request, including entries from `orion-core` and `orion-ledger`.

```text title="An orion-gateway log entry"
2026-04-12T09:31:04.412Z WARN  [orion-gateway] request_id=req_01HQ8ZV3KXN4M2T9YB7C6D tenant=ten_01HQ8ZV3KXN4M2T9YB7C6D
  route=POST /v1/orders status=400 code=ORN-1004 field=currency duration_ms=18
```

Record the identifier for every failed response. Searching by `request_id` is described on the [Logs](../../operations/monitoring/logs.md) page. A report without an identifier extends diagnosis by at least one iteration.

## Examples of complete responses

??? example "Validation error, 400 ORN-1007"

    ```json
    {
      "error": {
        "code": "ORN-1007",
        "message": "The 'CHF' currency is not supported. Allowed: PLN, EUR, USD.",
        "type": "validation_error",
        "field": "currency",
        "request_id": "req_01HQ8ZV3KXN4M2T9YB7C6D",
        "doc_url": "https://docs.orion.example.com/api/rest/error-codes/"
      }
    }
    ```

??? example "Rate limit exceeded, 429 ORN-3001"

    ```json
    {
      "error": {
        "code": "ORN-3001",
        "message": "The limit of 120 requests per minute for POST /v1/orders was exceeded.",
        "type": "rate_limit_error",
        "field": null,
        "request_id": "req_01HQ8ZV3KXN4M2T9YB7C7E",
        "doc_url": "https://docs.orion.example.com/api/rest/rate-limits/"
      }
    }
    ```

??? example "State conflict, 409 ORN-4005"

    ```json
    {
      "error": {
        "code": "ORN-4005",
        "message": "The transition from the 'fulfilled' status to 'pending_payment' is not allowed.",
        "type": "conflict_error",
        "field": "status",
        "request_id": "req_01HQ8ZV3KXN4M2T9YB7C8F",
        "doc_url": "https://docs.orion.example.com/api/rest/resources/orders/"
      }
    }
    ```

## See also

- [REST API basics](index.md)
- [Rate limits](rate-limits.md)
- [Logs](../../operations/monitoring/logs.md)
- [Support](../../support.md)
