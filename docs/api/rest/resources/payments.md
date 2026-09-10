# Payments

The `payment` resource represents a single attempt to collect funds for an order. Payments are managed by the `orion-ledger` component (port `8082`), which keeps the ledger in the `ledger` schema of the `orion` database. The payment lifecycle is independent of the order lifecycle, but capturing funds moves the order to the `paid` status.

## Object model

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | string | read only | a `pay_...` identifier |
| `order` | string | yes | an `ord_...` identifier |
| `status` | string | read only | `initiated`, `authorized`, `captured`, `failed`, `refunded`, `partially_refunded`, `expired` |
| `method` | string | yes | `card`, `blik`, `transfer`, `bnpl`, `wallet` |
| `amount` | integer | yes | amount in the smallest unit of the currency |
| `currency` | string | yes | `PLN`, `EUR`, `USD` |
| `captured_amount` | integer | read only | the captured amount |
| `refunded_amount` | integer | read only | total of the refunds |
| `provider_reference` | string | read only | reference assigned by the payment provider |
| `failure_code` | string | read only | decline code, `null` on success |
| `three_ds` | object | read only | result of 3-D Secure authentication |
| `created_at` | string | read only | ISO-8601 UTC timestamp |

```json title="Complete payment object"
{
  "id": "pay_01HQ8ZV3KXN4M2T9YB7C6D",
  "object": "payment",
  "order": "ord_01HQ8ZV3KXN4M2T9YB7C6D",
  "status": "authorized",
  "method": "card",
  "amount": 1299000,
  "currency": "PLN",
  "captured_amount": 0,
  "refunded_amount": 0,
  "provider_reference": "NBX-773401-99",
  "failure_code": null,
  "three_ds": {
    "required": true,
    "status": "authenticated",
    "redirect_url": null
  },
  "created_at": "2026-04-12T09:33:12Z"
}
```

## Payment methods

| Method | Currencies | Deferred authorization | Partial capture | Expiration time |
| --- | --- | --- | --- | --- |
| `card` | PLN, EUR, USD | yes | yes | 7 days |
| `blik` | PLN | no | no | 2 minutes |
| `transfer` | PLN, EUR | no | no | 72 hours |
| `bnpl` | PLN | yes | no | 30 days |
| `wallet` | PLN, EUR, USD | yes | yes | 24 hours |

!!! note "Deferred authorization"
    For methods with deferred authorization the payment stops in the `authorized` status until `capture` is called. Methods without that capability move from `initiated` directly to `captured`.

## Endpoints

### POST /v1/payments

Initiates a payment for an order in the `pending_payment` status. Requires the `payments:write` scope.

| Body field | Type | Required | Description |
| --- | --- | --- | --- |
| `order` | string | yes | an `ord_...` identifier |
| `method` | string | yes | one of the methods from the table above |
| `amount` | integer | yes | must equal the `amount_total` of the order |
| `currency` | string | yes | must match the currency of the order |
| `return_url` | string | for `card`, `bnpl` | return address after 3-D Secure |
| `capture` | boolean | no | `true` means an immediate capture |

=== "curl"

    ```bash
    curl -X POST https://api.orion.example.com/v1/payments \
      -H "X-Orion-Key: ok_live_9f2c41ab7d5e" \
      -H "Idempotency-Key: pay-ord-118-1" \
      -H "Content-Type: application/json" \
      -d '{"order":"ord_01HQ8ZV3KXN4M2T9YB7C6D","method":"card","amount":1299000,"currency":"PLN","return_url":"https://shop.example.com/return"}'
    ```

=== "Python"

    ```python
    payment = client.payments.create(
        order="ord_01HQ8ZV3KXN4M2T9YB7C6D",
        method="card",
        amount=1299000,
        currency="PLN",
        return_url="https://shop.example.com/return",
        idempotency_key="pay-ord-118-1",
    )
    ```

=== "Java"

    ```java
    Payment payment = client.payments().create(
        PaymentCreateParams.builder()
            .order("ord_01HQ8ZV3KXN4M2T9YB7C6D")
            .method(PaymentMethod.CARD)
            .amount(1_299_000L)
            .currency("PLN")
            .returnUrl("https://shop.example.com/return")
            .build());
    ```

```json title="201 Created response"
{
  "id": "pay_01HQ8ZV3KXN4M2T9YB7C6D",
  "object": "payment",
  "status": "initiated",
  "method": "card",
  "amount": 1299000,
  "currency": "PLN",
  "three_ds": {
    "required": true,
    "status": "pending",
    "redirect_url": "https://acs.bank.example.com/3ds/NBX-773401-99"
  }
}
```

Possible errors: `ORN-1004`, `ORN-1007`, `ORN-4008` (the order is already paid), `ORN-4005` (the order is not in `pending_payment`), `ORN-4002`, `ORN-3001`.

### GET /v1/payments/{id}

Returns a payment. Requires the `payments:read` scope. The `expand` parameter accepts `order`, `refunds`.

Possible errors: `ORN-2001`, `ORN-2003`, `ORN-5002`.

### POST /v1/payments/{id}/capture

Captures previously authorized funds. Without an `amount` field it captures the full amount.

| Body field | Type | Required | Description |
| --- | --- | --- | --- |
| `amount` | integer | no | partial amount, ≤ `amount` minus `captured_amount` |

```json title="200 OK response, partial capture"
{
  "id": "pay_01HQ8ZV3KXN4M2T9YB7C6D",
  "status": "captured",
  "amount": 1299000,
  "captured_amount": 800000
}
```

Possible errors: `ORN-4005` (the payment is not in `authorized`), `ORN-1004`, `ORN-5009`.

### POST /v1/payments/{id}/refund

Creates a refund with a `ref_...` identifier. The total of the refunds cannot exceed `captured_amount`.

| Body field | Type | Required | Description |
| --- | --- | --- | --- |
| `amount` | integer | no | defaults to the entire captured amount |
| `reason` | string | no | `requested_by_customer`, `duplicate`, `fraud`, `other` |

```json title="201 Created response"
{
  "id": "ref_01HQ8ZV3KXN4M2T9YB7C8F",
  "object": "refund",
  "payment": "pay_01HQ8ZV3KXN4M2T9YB7C6D",
  "amount": 300000,
  "currency": "PLN",
  "status": "processing",
  "reason": "requested_by_customer"
}
```

Once the refund is booked, the payment receives the `partially_refunded` or `refunded` status and the system publishes a `payment.refunded` event.

Possible errors: `ORN-1004`, `ORN-4005`, `ORN-4002`, `ORN-5002`.

### GET /v1/payments

A list of payments with cursor pagination.

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `order` | query | no | an `ord_...` identifier |
| `status` | query | no | for example `status[in]=authorized,captured` |
| `method` | query | no | payment method |
| `created_after` | query | no | ISO-8601 UTC timestamp |
| `limit` | query | no | defaults to 25, max. 100 |

Possible errors: `ORN-1012`, `ORN-3004`, `ORN-3001`.

### POST /v1/payments/{id}/cancel

Reverses an authorization before capture. The payment moves to `failed` with `failure_code: "cancelled_by_merchant"`.

Possible errors: `ORN-4005` (the funds have already been captured), `ORN-2003`, `ORN-5009`.

## 3-D Secure

For the `card` and `wallet` methods the provider may require additional authentication:

1. `POST /v1/payments` returns `three_ds.status: "pending"` and `three_ds.redirect_url`.
2. Redirect the buyer to `redirect_url`. Do not embed that address in a frame.
3. After authentication the buyer returns to `return_url` with the `payment_id` and `result` parameters.
4. Read the payment state with `GET /v1/payments/{id}`; the `result` parameter is only a hint for the interface, the API is the source of truth.

!!! warning "Do not rely on the browser return"
    The buyer may close the tab before returning. The final state is confirmed by the `payment.authorized` or `payment.failed` event. See [Signature verification](../../webhooks/signature-verification.md).

## Decline codes

| `failure_code` | Meaning | Recommended action |
| --- | --- | --- |
| `insuffic_funds` | insufficient funds in the account | offer a different method, do not retry automatically |
| `card_expired` | the card has expired | ask for updated card details |
| `do_not_honor` | issuer refusal without a stated reason | one retry after 24 h, then a different method |
| `3ds_failed` | authentication failed or was interrupted | create a new payment and repeat the redirect |
| `timeout` | no provider response within the time window | check the state with `GET /v1/payments/{id}` before retrying |

## Settlements and payouts

Every capture and refund creates a double entry in the `ledger` schema: a debit of the provider's technical account and a credit of the tenant's account. Entries are grouped into daily batches closed at `23:59Z`, and the payout is executed on a D+2 cycle to the account configured in the console.

!!! danger "Integer amounts and rounding"
    Never send amounts as floating point numbers. Write `12,990.00 PLN` as `1299000`. Tax is computed on the line item amount and rounded to a whole cent using half up rounding, so the sum of the line item taxes may differ by one cent from the tax computed on the order total; the sum of the line items is authoritative.

## See also

- [Orders](orders.md)
- [Error codes](../error-codes.md)
- [Rate limits](../rate-limits.md)
- [Event types](../../webhooks/event-types.md)
