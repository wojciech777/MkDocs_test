# Webhooks

Webhooks are the asynchronous integration surface of Orion Platform 4.2 LTS. Instead of polling the REST API for changes, you register an HTTPS address and `orion-worker` pushes a signed `POST` request to it every time a matching event occurs. A single tenant may keep up to 200 active endpoints, and each of them has its own subscription list, its own secret, and its own delivery statistics.

Polling `GET /v1/orders` in a loop consumes the tenant's rate limit and adds latency equal to half the polling interval. Webhooks remove both problems: the delivery attempt starts within a second of the event being published to `orion.events.v1`.

## Registering an endpoint

An endpoint is created with `POST /v1/webhooks`. The call requires the `webhooks:write` scope, or an API key with that scope assigned.

```bash title="Registering a production endpoint"
curl -X POST https://api.orion.example.com/v1/webhooks \
  -H "X-Orion-Key: ok_live_9f2c41ab7d5e" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://shop.example.com/webhooks/orion",
    "events": ["order.paid", "order.cancelled", "payment.failed"],
    "description": "ERP integration (production)",
    "active": true,
    "api_revision": "2026-04-01"
  }'
```

```json title="201 Created response" hl_lines="3 9"
{
  "id": "whk_01HQ9B4TMZP7R3XK2VD8FE",
  "secret": "whsec_9f2c7ab41de84c05b6e3a71d05",
  "url": "https://shop.example.com/webhooks/orion",
  "events": ["order.paid", "order.cancelled", "payment.failed"],
  "description": "ERP integration (production)",
  "active": true,
  "api_revision": "2026-04-01",
  "created_at": "2026-05-14T09:12:44Z"
}
```

## Configuration fields

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `url` | string | yes | Receiver address. HTTPS only; a port other than 443 is allowed. Plain `http://` is rejected with `ORN-1004`. |
| `events[]` | array | yes | Event types or patterns such as `order.*`. At least one entry. See [Event types](event-types.md). |
| `description` | string | no | Free text, up to 200 characters, shown on the endpoint list in `orion-console`. |
| `secret` | string | no | The `whsec_...` value used to sign deliveries. If omitted, it is generated and returned only once, in the `201` response. |
| `active` | boolean | no | Whether deliveries are attempted. Defaults to `true`. |
| `api_revision` | string | no | Payload schema revision, for example `2026-04-01`. Defaults to the revision currently assigned to the tenant. |

!!! danger "The secret is returned only once"
    The `secret` field appears in the `201 Created` response and never again. `GET /v1/webhooks/{id}` returns only the last four characters. If you lose the value, you cannot read it back: the only way forward is the rotation procedure described in [Signature verification](signature-verification.md).

## Managing endpoints

| Operation | Method and path | Required scope | Notes |
| --- | --- | --- | --- |
| List endpoints | `GET /v1/webhooks` | `webhooks:write` | Cursor pagination, `limit` defaults to 25 and maxes out at 100. See [Pagination, filtering, and sorting](../rest/pagination.md). |
| Read one endpoint | `GET /v1/webhooks/{id}` | `webhooks:write` | Includes delivery statistics from the last 24 h: attempts, successes, failures, and the last response code. |
| Update an endpoint | `PATCH /v1/webhooks/{id}` | `webhooks:write` | JSON Merge Patch, same fields as on creation. Changing `events[]` replaces the whole list. |
| Delete an endpoint | `DELETE /v1/webhooks/{id}` | `webhooks:write` | Returns `204`. Delivery history is kept for 30 days, so you can still audit past attempts. |
| Send a test event | `POST /v1/webhooks/{id}/test` | `webhooks:write` | Emits `webhook.test`, returns a `dlv_...` delivery identifier and the response code observed at your endpoint. |
| Rotate the secret | `POST /v1/webhooks/{id}/rotate-secret` | `admin` | Returns a new `whsec_...` value together with the expiry of the previous one. |

```bash title="Reactivating a deactivated endpoint"
curl -X PATCH https://api.orion.example.com/v1/webhooks/whk_01HQ9B4TMZP7R3XK2VD8FE \
  -H "X-Orion-Key: ok_live_9f2c41ab7d5e" \
  -H "Content-Type: application/json" \
  -d '{"active": true}'
```

```bash title="Sending a test event"
curl -X POST https://api.orion.example.com/v1/webhooks/whk_01HQ9B4TMZP7R3XK2VD8FE/test \
  -H "X-Orion-Key: ok_live_9f2c41ab7d5e"
```

```json title="200 OK response to the test call"
{
  "delivery_id": "dlv_01HQ9B7XKMR4P2TVD6BZ8N",
  "event_type": "webhook.test",
  "response_code": 202,
  "duration_ms": 143
}
```

## Delivery rules

Every delivery is a `POST` request with a JSON body containing exactly one event envelope. The body is never wrapped in an array and never batched.

| Header | Example | Description |
| --- | --- | --- |
| `X-Orion-Event` | `order.paid` | Event type, identical to the `type` field in the body. |
| `X-Orion-Signature` | `t=1778490764,v2=...` | HMAC signature over the raw body. See [Signature verification](signature-verification.md). |
| `X-Orion-Delivery` | `dlv_01HQ9B7XKMR4P2TVD6BZ8N` | Identifier of this delivery attempt. Quote it in support tickets. |

- **Expected response.** Any `2xx` status within a **10 s** timeout counts as success. A redirect is not followed, and a `3xx` response is treated as a failure.
- **At-least-once semantics.** The same `evt_...` may arrive more than once, for example when your `200` response does not reach `orion-worker` before the timeout. Deduplicate on the envelope's `id`.
- **Ordering.** Deliveries are not guaranteed to arrive in the order in which the events were created. Compare the `sequence` field with the last value you processed for that resource and ignore lower values.
- **Retries.** Failed deliveries are retried with exponential backoff and jitter. The schedule, the classification of transient and permanent errors, and the circuit breaker are documented in [Retries and idempotency](../../guides/events/retries.md).
- **Suspension.** After repeated failures across the full retry cycle, the endpoint is flipped to `active: false` and stops receiving traffic. `orion-console` shows the reason, and you resume deliveries with a `PATCH` that sets `active` back to `true`.
- **Dead letter queue.** Deliveries that exhaust all attempts land in `orion.events.v1.dlq` and increment the `orion_events_dlq_total` counter, which is wired to an alert described on the [Alerts](../../operations/monitoring/alerts.md) page.

!!! tip "Answer first, process later"
    Persist the raw body, respond with `202 Accepted`, and do the business work in a background worker. Endpoints that call a payment provider or an ERP synchronously inside the handler are the most common cause of timeouts and needless retries.

```http title="A delivery as it arrives at the endpoint"
POST /webhooks/orion HTTP/1.1
Host: shop.example.com
Content-Type: application/json
Content-Length: 412
User-Agent: orion-worker/4.2.1
X-Orion-Event: order.paid
X-Orion-Delivery: dlv_01HQ9B7XKMR4P2TVD6BZ8N
X-Orion-Signature: t=1778490764,v2=8f3c1a9d47b6e05c2f8a1d3b9e7c40f6a2d5b8c1e4f7a0d3b6c9e2f5a8d1b4c7
```

The body is the event envelope documented in [Event types](event-types.md), with the schema selected by the endpoint's `api_revision`.

## Inspecting deliveries

`GET /v1/webhooks/{id}` returns a summary of the last 24 h next to the configuration, which is usually enough to tell a receiver problem apart from a subscription problem.

```json title="Delivery statistics in the endpoint response"
{
  "id": "whk_01HQ9B4TMZP7R3XK2VD8FE",
  "active": true,
  "deliveries_24h": {
    "attempted": 1842,
    "succeeded": 1839,
    "failed": 3,
    "last_response_code": 202,
    "last_success_at": "2026-05-14T09:12:45Z"
  }
}
```

- `attempted` counts retries as separate attempts, so it can exceed the number of events.
- A `last_response_code` of `401` almost always means a signature problem on the receiver side.
- A growing `failed` count with no successes means the endpoint is heading for suspension.

The same view is available in `orion-console`, together with the response body of the last failed attempt. Delivery level troubleshooting uses the `X-Orion-Delivery` identifier and the log search described in the [Alerts](../../operations/monitoring/alerts.md) page runbooks.

## Receiver requirements

- [ ] The address is reachable over HTTPS with a certificate issued by a public CA.
- [ ] The signature is verified before the body is parsed or trusted.
- [ ] The raw request bytes are available to the verification code, without re-serialization.
- [ ] A `2xx` response is returned in under 10 s, ideally in under 1 s.
- [ ] Processed `evt_...` identifiers are stored for at least 7 days and repeats are rejected.
- [ ] Out-of-order deliveries are handled by comparing `sequence`.
- [ ] The `whsec_...` secret is loaded from a secret store, not from the repository. See [Operational security](../../operations/security.md).
- [ ] Two secrets can be accepted at once, so a rotation does not require a deployment.
- [ ] Unknown event types and unknown fields are ignored rather than rejected.

## Limits

| Limit | Value | Scope |
| --- | --- | --- |
| Active endpoints | 200 | tenant |
| Subscriptions per endpoint | 50 entries in `events[]` | endpoint |
| `description` length | 200 characters | endpoint |
| Delivery timeout | 10 s | delivery attempt |
| Delivery history retention | 30 days | tenant |

Outgoing deliveries do not consume the tenant's inbound request limit, as noted in [Rate limits](../rest/rate-limits.md).

## See also

- [Event types](event-types.md)
- [Signature verification](signature-verification.md)
- [Retries and idempotency](../../guides/events/retries.md)
- [Event processing](../../guides/events/index.md)
- [REST API basics](../rest/index.md)
