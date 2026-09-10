# Event types

Orion Platform 4.2 LTS emits 14 event types covering the lifecycle of orders, payments and customers. Every event uses the same envelope, and the differences concern only the contents of `data.object`. The type catalog is additive: new types appear in minor releases, and removals happen only on a major version change, with a withdrawal period.

## Naming convention

An event name has the form `resource.action`, where the resource is singular (`order`, `payment`, `customer`) and the action is a verb in the past tense (`created`, `paid`, `refunded`). Names are entirely lowercase, without underscores and without a version in the name. The exception is `webhook.test`, which does not correspond to any business resource.

!!! note "Why there is no version in the name"
    Payload versioning happens through the `api_revision` field in the envelope and through the endpoint's `api_revision` attribute. Thanks to that, the same event name can have different schemas for different subscriptions.

## Envelope structure

| Field | Type | Always present | Description |
| --- | --- | --- | --- |
| `id` | string | yes | Event identifier `evt_...`, the deduplication key. |
| `type` | string | yes | Event type, for example `order.paid`. |
| `api_revision` | string | yes | Payload schema revision, for example `2026-04-01`. |
| `occurred_at` | string | yes | The moment the event occurred, ISO-8601 UTC with seconds. |
| `tenant` | string | yes | Tenant identifier `ten_...`. |
| `sequence` | number | yes | Monotonic counter within the tenant. Used for ordering. |
| `data.object` | object | yes | Snapshot of the resource at `occurred_at`. |
| `data.previous` | object | no | Only for `*.updated` events: the changed fields before the modification. |

```json title="Complete event envelope" hl_lines="2 6 7"
{
  "id": "evt_01HQ8ZV3KXN4M2T9YB7C6D",
  "type": "order.paid",
  "api_revision": "2026-04-01",
  "occurred_at": "2026-05-14T09:12:44Z",
  "tenant": "ten_01HQ8ZV3KXN4M2T9YB7C6D",
  "sequence": 884213,
  "data": {
    "object": {
      "id": "ord_01HQ9A2FVJ8D6M3RTP5KWX",
      "status": "paid"
    }
  }
}
```

## Event catalog

| Type | When it is emitted | `data.object` | `data.previous` | Since version |
| --- | --- | --- | --- | --- |
| `order.created` | An order is created in the `draft` or `pending_payment` state. | `order` | no | 3.0 |
| `order.updated` | A line item, address, metadata or a secondary status (`on_hold`) changes. | `order` | yes | 3.0 |
| `order.paid` | The order moves to the `paid` status after the payment is captured. | `order` | no | 3.0 |
| `order.cancelled` | Cancellation by the customer, an operator or a risk rule. | `order` | no | 3.0 |
| `order.fulfilled` | All line items have been assembled, status `fulfilled`. | `order` | no | 3.4 |
| `order.expired` | The payment window set by `orion-scheduler` has expired. | `order` | no | 4.0 |
| `payment.authorized` | Authorization obtained from the provider, status `authorized`. | `payment` | no | 3.0 |
| `payment.captured` | Funds captured, status `captured`. | `payment` | no | 3.0 |
| `payment.failed` | Authorization or capture rejected, status `failed`. | `payment` | no | 3.0 |
| `payment.refunded` | Full or partial refund (`refunded`, `partially_refunded`). | `payment` | no | 3.2 |
| `customer.created` | A new customer registers. | `customer` | no | 3.0 |
| `customer.updated` | Contact details, address or consents change. | `customer` | yes | 3.0 |
| `customer.anonymized` | Anonymization of personal data has completed. | `customer` | no | 4.1 |
| `webhook.test` | Manual call to `POST /v1/webhooks/{id}/test`. | `webhook` | no | 3.6 |

!!! tip "Subscribe to the minimal set"
    An endpoint listening to everything (`order.*`, `payment.*`, `customer.*`) receives several times more traffic than it needs at high volume. Limit `events[]` to the types you actually react to.

## Example payloads

??? example "order.created"

    ```json
    {
      "id": "evt_01HQ9A5TKMR2P8XVD3BC7Y",
      "type": "order.created",
      "api_revision": "2026-04-01",
      "occurred_at": "2026-05-14T09:10:02Z",
      "tenant": "ten_01HQ8ZV3KXN4M2T9YB7C6D",
      "sequence": 884199,
      "data": {
        "object": {
          "id": "ord_01HQ9A2FVJ8D6M3RTP5KWX",
          "status": "pending_payment",
          "customer_id": "cus_01HQ7YB4NDR6K2WMZP9TXV",
          "currency": "PLN",
          "amount_total": 24990,
          "items": [
            {
              "sku": "NB-14-PRO",
              "name": "Notebook 14 Pro",
              "quantity": 1,
              "unit_amount": 24990
            }
          ],
          "created_at": "2026-05-14T09:10:02Z"
        }
      }
    }
    ```

??? example "order.paid"

    ```json
    {
      "id": "evt_01HQ9AB7WZP4N6TKD2XR8M",
      "type": "order.paid",
      "api_revision": "2026-04-01",
      "occurred_at": "2026-05-14T09:12:44Z",
      "tenant": "ten_01HQ8ZV3KXN4M2T9YB7C6D",
      "sequence": 884213,
      "data": {
        "object": {
          "id": "ord_01HQ9A2FVJ8D6M3RTP5KWX",
          "status": "paid",
          "customer_id": "cus_01HQ7YB4NDR6K2WMZP9TXV",
          "currency": "PLN",
          "amount_total": 24990,
          "amount_paid": 24990,
          "payment_id": "pay_01HQ9AA3RKM7D5XVTB2NPZ",
          "paid_at": "2026-05-14T09:12:43Z"
        }
      }
    }
    ```

??? example "payment.failed"

    ```json
    {
      "id": "evt_01HQ9AF2DKR8M4TXVP7CB3",
      "type": "payment.failed",
      "api_revision": "2026-04-01",
      "occurred_at": "2026-05-14T09:15:19Z",
      "tenant": "ten_01HQ8ZV3KXN4M2T9YB7C6D",
      "sequence": 884231,
      "data": {
        "object": {
          "id": "pay_01HQ9AE8TVN3K6MRDZ2XPB",
          "status": "failed",
          "order_id": "ord_01HQ9AD4MKP2R7TXVB6NZD",
          "currency": "PLN",
          "amount": 13500,
          "method": "card",
          "failure_code": "issuer_declined",
          "failure_message": "Transaction declined by the card issuer",
          "attempt": 2
        }
      }
    }
    ```

??? example "customer.updated"

    ```json
    {
      "id": "evt_01HQ9AH6NKD3R8MTXVZ2PB",
      "type": "customer.updated",
      "api_revision": "2026-04-01",
      "occurred_at": "2026-05-14T09:18:07Z",
      "tenant": "ten_01HQ8ZV3KXN4M2T9YB7C6D",
      "sequence": 884248,
      "data": {
        "object": {
          "id": "cus_01HQ7YB4NDR6K2WMZP9TXV",
          "email": "anna.kowalska@example.com",
          "phone": "+48512340099",
          "marketing_consent": true,
          "updated_at": "2026-05-14T09:18:07Z"
        },
        "previous": {
          "phone": "+48512340011",
          "marketing_consent": false
        }
      }
    }
    ```

## Ordering and duplicates

Delivery has at-least-once semantics. That has two consequences you must handle in the receiver's code:

1. **Duplicates.** The same `evt_...` may arrive multiple times (for example when your `200` response does not reach `orion-worker` before the timeout). Keep a table of processed identifiers with a retention of at least 7 days and reject repeats.
2. **Ordering.** Events do not have to arrive in the order in which they were created. Compare `sequence` with the last processed value for the given resource and ignore lower values.

```sql title="Idempotent event insert"
INSERT INTO orion_events (event_id, event_type, sequence_no, payload)
VALUES ($1, $2, $3, $4)
ON CONFLICT (event_id) DO NOTHING;
```

!!! warning "Do not rely on occurred_at"
    The `occurred_at` timestamp has one-second resolution, so two events concerning the same order can carry an identical value. The only correct ordering criterion is `sequence`.

## Filtering subscriptions

The `events[]` field accepts exact names as well as patterns with an asterisk at the action level:

| Pattern | Matches |
| --- | --- |
| `order.paid` | only `order.paid` |
| `order.*` | all six events of the `order` resource |
| `payment.*` | all four events of the `payment` resource |
| `*` | all types, including new ones added in future releases |

An asterisk may not appear in the middle of a name: `order.*.paid` is rejected with error `ORN-1004`.

## Test events

`POST /v1/webhooks/{id}/test` emits `webhook.test` with a payload describing the endpoint. The event is not retried and does not affect the deactivation counter, but it is fully signed, so it is suitable for end-to-end verification of an integration.

```json title="webhook.test payload"
{
  "id": "evt_01HQ9AK4TVR7M2XKDPB3NZ",
  "type": "webhook.test",
  "api_revision": "2026-04-01",
  "occurred_at": "2026-05-14T09:20:00Z",
  "tenant": "ten_01HQ8ZV3KXN4M2T9YB7C6D",
  "sequence": 0,
  "data": {
    "object": {
      "id": "whk_01HQ9B4TMZP7R3XK2VD8FE",
      "url": "https://shop.example.com/webhooks/orion",
      "message": "Test delivery from Orion Platform 4.2.1"
    }
  }
}
```

## Types withdrawn in 4.2

Names from the 3.8 line are no longer emitted. Endpoints subscribing to the old names receive no events at all; updating `events[]` through `PATCH /v1/webhooks/{id}` is required.

| Type from 3.8 | Replacement in 4.2 | Notes |
| --- | --- | --- |
| `order.status_changed` | `order.updated` | The previous status is in `data.previous.status`. |
| `order.payment_received` | `order.paid` | The payment identifier is in `data.object.payment_id`. |
| `order.timeout` | `order.expired` | Renamed, payload unchanged. |
| `payment.success` | `payment.captured` | Authorization and capture are split into two events. |
| `payment.error` | `payment.failed` | The reason code is in `data.object.failure_code`. |
| `customer.deleted` | `customer.anonymized` | The record is not deleted, only stripped of personal data. |

The step-by-step mapping together with a migration checklist is described in [Migrate from 3.8 LTS to 4.2 LTS](../../guides/migrations/from-3-8-to-4-2.md).[^1]

[^1]: Emission of the old names was disabled in 4.2.0; in the 4.1 line it ran alongside the new names as duplicated events.

## See also

- [Webhooks](index.md)
- [Signature verification](signature-verification.md)
- [Event processing](../../guides/events/index.md)
- [Migrate from 3.8 LTS to 4.2 LTS](../../guides/migrations/from-3-8-to-4-2.md)
