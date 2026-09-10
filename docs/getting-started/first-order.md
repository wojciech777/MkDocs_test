# Your first order

This tutorial walks through the full path from fetching a token to a confirmed payment and a received webhook. All examples use the sandbox environment `https://api.sandbox.orion.example.com/v1`; in production change the address to `https://api.orion.example.com/v1`. We assume you have an OAuth2 client in the `orion` realm with the `orders:write`, `payments:write` and `customers:write` scopes. The whole tutorial takes about 20 minutes.

## Step 1. Fetch a token

The token is issued in the client credentials flow and is valid for 3600 seconds. Request only the scopes you are actually going to use.

=== "curl"

    ```bash
    curl -s -X POST https://api.sandbox.orion.example.com/v1/oauth/token \
      -H "Content-Type: application/x-www-form-urlencoded" \
      -d "grant_type=client_credentials" \
      -d "client_id=orion-integration" \
      -d "client_secret=$ORION_CLIENT_SECRET" \
      -d "scope=customers:write orders:write payments:write"
    ```

=== "Java"

    ```java title="TokenClient.java"
    var form = "grant_type=client_credentials"
        + "&client_id=orion-integration"
        + "&client_secret=" + URLEncoder.encode(clientSecret, UTF_8)
        + "&scope=" + URLEncoder.encode("customers:write orders:write payments:write", UTF_8);

    var request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.sandbox.orion.example.com/v1/oauth/token"))
        .header("Content-Type", "application/x-www-form-urlencoded")
        .POST(HttpRequest.BodyPublishers.ofString(form))
        .build();

    var response = HttpClient.newHttpClient()
        .send(request, HttpResponse.BodyHandlers.ofString());
    var accessToken = new ObjectMapper()
        .readTree(response.body())
        .get("access_token").asText();
    ```

=== "Python"

    ```python title="token_client.py"
    import os
    import requests

    resp = requests.post(
        "https://api.sandbox.orion.example.com/v1/oauth/token",
        data={
            "grant_type": "client_credentials",
            "client_id": "orion-integration",
            "client_secret": os.environ["ORION_CLIENT_SECRET"],
            "scope": "customers:write orders:write payments:write",
        },
        timeout=10,
    )
    resp.raise_for_status()
    access_token = resp.json()["access_token"]
    ```

Response:

```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6Im9yaW9uLTIwMjUtMDkifQ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "customers:write orders:write payments:write"
}
```

!!! tip "Cache the token"
    Do not fetch a token before every request. Every call to `/oauth/token` counts towards the limit of 60 requests per minute in the sandbox. Refresh the token when less than 60 seconds remain until it expires.

## Step 2. Create a customer

=== "curl"

    ```bash
    curl -s -X POST https://api.sandbox.orion.example.com/v1/customers \
      -H "Authorization: Bearer $ACCESS_TOKEN" \
      -H "Content-Type: application/json" \
      -H "Idempotency-Key: create-customer-2025-11-04-0001" \
      -d '{
        "email": "anna.kowalska@example.com",
        "name": "Anna Kowalska",
        "sales_channel": "web",
        "billing_address": {
          "line1": "51 Prosta Street",
          "city": "Warsaw",
          "postal_code": "00-838",
          "country": "PL"
        }
      }'
    ```

=== "Java"

    ```java title="CreateCustomer.java"
    var body = """
        {
          "email": "anna.kowalska@example.com",
          "name": "Anna Kowalska",
          "sales_channel": "web",
          "billing_address": {
            "line1": "51 Prosta Street",
            "city": "Warsaw",
            "postal_code": "00-838",
            "country": "PL"
          }
        }
        """;

    var request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.sandbox.orion.example.com/v1/customers"))
        .header("Authorization", "Bearer " + accessToken)
        .header("Content-Type", "application/json")
        .header("Idempotency-Key", "create-customer-2025-11-04-0001")
        .POST(HttpRequest.BodyPublishers.ofString(body))
        .build();
    ```

=== "Python"

    ```python title="create_customer.py"
    customer = requests.post(
        "https://api.sandbox.orion.example.com/v1/customers",
        headers={
            "Authorization": f"Bearer {access_token}",
            "Idempotency-Key": "create-customer-2025-11-04-0001",
        },
        json={
            "email": "anna.kowalska@example.com",
            "name": "Anna Kowalska",
            "sales_channel": "web",
            "billing_address": {
                "line1": "51 Prosta Street",
                "city": "Warsaw",
                "postal_code": "00-838",
                "country": "PL",
            },
        },
        timeout=10,
    ).json()
    customer_id = customer["id"]
    ```

A `201 Created` response:

```json
{
  "id": "cus_01HQ8ZV3KXN4M2T9YB7C6D",
  "tenant_id": "ten_01HQ8ZTP4M7RN2VC5XK9BW",
  "email": "anna.kowalska@example.com",
  "name": "Anna Kowalska",
  "sales_channel": "web",
  "created_at": "2025-11-04T10:12:44Z"
}
```

## Step 3. Create an order

The order is created with status `draft`. Amounts are given in the smallest unit of the currency, so 24900 means 249.00 PLN.

=== "curl"

    ```bash
    curl -s -X POST https://api.sandbox.orion.example.com/v1/orders \
      -H "Authorization: Bearer $ACCESS_TOKEN" \
      -H "Content-Type: application/json" \
      -H "Idempotency-Key: create-order-2025-11-04-0001" \
      -d '{
        "customer_id": "cus_01HQ8ZV3KXN4M2T9YB7C6D",
        "currency": "PLN",
        "sales_channel": "web",
        "order_lines": [
          { "sku": "KB-87-PL", "name": "Mechanical keyboard 87 keys", "quantity": 1, "unit_amount": 24900 },
          { "sku": "MP-XL-BLK", "name": "XL mouse pad", "quantity": 2, "unit_amount": 4900 }
        ]
      }'
    ```

=== "Java"

    ```java title="CreateOrder.java"
    var body = """
        {
          "customer_id": "cus_01HQ8ZV3KXN4M2T9YB7C6D",
          "currency": "PLN",
          "sales_channel": "web",
          "order_lines": [
            { "sku": "KB-87-PL", "name": "Mechanical keyboard 87 keys", "quantity": 1, "unit_amount": 24900 },
            { "sku": "MP-XL-BLK", "name": "XL mouse pad", "quantity": 2, "unit_amount": 4900 }
          ]
        }
        """;

    var request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.sandbox.orion.example.com/v1/orders"))
        .header("Authorization", "Bearer " + accessToken)
        .header("Content-Type", "application/json")
        .header("Idempotency-Key", "create-order-2025-11-04-0001")
        .POST(HttpRequest.BodyPublishers.ofString(body))
        .build();
    ```

=== "Python"

    ```python title="create_order.py"
    order = requests.post(
        "https://api.sandbox.orion.example.com/v1/orders",
        headers={
            "Authorization": f"Bearer {access_token}",
            "Idempotency-Key": "create-order-2025-11-04-0001",
        },
        json={
            "customer_id": customer_id,
            "currency": "PLN",
            "sales_channel": "web",
            "order_lines": [
                {"sku": "KB-87-PL", "name": "Mechanical keyboard 87 keys",
                 "quantity": 1, "unit_amount": 24900},
                {"sku": "MP-XL-BLK", "name": "XL mouse pad",
                 "quantity": 2, "unit_amount": 4900},
            ],
        },
        timeout=10,
    ).json()
    order_id = order["id"]
    ```

A `201 Created` response:

```json
{
  "id": "ord_01HQ8ZW7PM5N3T8YC4B2FR",
  "tenant_id": "ten_01HQ8ZTP4M7RN2VC5XK9BW",
  "customer_id": "cus_01HQ8ZV3KXN4M2T9YB7C6D",
  "status": "draft",
  "currency": "PLN",
  "total_amount": 34700,
  "order_lines": [
    { "sku": "KB-87-PL", "quantity": 1, "unit_amount": 24900, "line_amount": 24900 },
    { "sku": "MP-XL-BLK", "quantity": 2, "unit_amount": 4900, "line_amount": 9800 }
  ],
  "created_at": "2025-11-04T10:13:02Z"
}
```

At this point the `order.created` event has been published.

## Step 4. Initiate a payment

Creating a payment moves the order from status `draft` to `pending_payment` and authorizes the funds.

=== "curl"

    ```bash
    curl -s -X POST https://api.sandbox.orion.example.com/v1/payments \
      -H "Authorization: Bearer $ACCESS_TOKEN" \
      -H "Content-Type: application/json" \
      -H "Idempotency-Key: create-payment-2025-11-04-0001" \
      -d '{
        "order_id": "ord_01HQ8ZW7PM5N3T8YC4B2FR",
        "amount": 34700,
        "currency": "PLN",
        "method": "card",
        "capture_mode": "manual"
      }'
    ```

=== "Java"

    ```java title="CreatePayment.java"
    var body = """
        {
          "order_id": "ord_01HQ8ZW7PM5N3T8YC4B2FR",
          "amount": 34700,
          "currency": "PLN",
          "method": "card",
          "capture_mode": "manual"
        }
        """;

    var request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.sandbox.orion.example.com/v1/payments"))
        .header("Authorization", "Bearer " + accessToken)
        .header("Content-Type", "application/json")
        .header("Idempotency-Key", "create-payment-2025-11-04-0001")
        .POST(HttpRequest.BodyPublishers.ofString(body))
        .build();
    ```

=== "Python"

    ```python title="create_payment.py"
    payment = requests.post(
        "https://api.sandbox.orion.example.com/v1/payments",
        headers={
            "Authorization": f"Bearer {access_token}",
            "Idempotency-Key": "create-payment-2025-11-04-0001",
        },
        json={
            "order_id": order_id,
            "amount": 34700,
            "currency": "PLN",
            "method": "card",
            "capture_mode": "manual",
        },
        timeout=10,
    ).json()
    payment_id = payment["id"]
    ```

A `201 Created` response:

```json
{
  "id": "pay_01HQ8ZX2RN6P4V9ZD5C3GT",
  "order_id": "ord_01HQ8ZW7PM5N3T8YC4B2FR",
  "status": "authorized",
  "amount": 34700,
  "authorized_amount": 34700,
  "captured_amount": 0,
  "currency": "PLN",
  "method": "card",
  "capture_mode": "manual",
  "authorized_at": "2025-11-04T10:13:29Z"
}
```

The order now has status `pending_payment` and a `payment.authorized` event has appeared on the bus.

## Step 5. Capture the funds and verify the event

Capture the authorized funds:

```bash
curl -s -X POST \
  https://api.sandbox.orion.example.com/v1/payments/pay_01HQ8ZX2RN6P4V9ZD5C3GT/capture \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: capture-payment-2025-11-04-0001" \
  -d '{ "amount": 34700 }'
```

```json
{
  "id": "pay_01HQ8ZX2RN6P4V9ZD5C3GT",
  "order_id": "ord_01HQ8ZW7PM5N3T8YC4B2FR",
  "status": "captured",
  "authorized_amount": 34700,
  "captured_amount": 34700,
  "captured_at": "2025-11-04T10:14:05Z"
}
```

Check the state of the order:

```bash
curl -s https://api.sandbox.orion.example.com/v1/orders/ord_01HQ8ZW7PM5N3T8YC4B2FR \
  -H "Authorization: Bearer $ACCESS_TOKEN"
```

```json
{
  "id": "ord_01HQ8ZW7PM5N3T8YC4B2FR",
  "status": "paid",
  "total_amount": 34700,
  "paid_amount": 34700,
  "status_history": [
    { "status": "draft",           "at": "2025-11-04T10:13:02Z" },
    { "status": "pending_payment", "at": "2025-11-04T10:13:29Z" },
    { "status": "paid",            "at": "2025-11-04T10:14:05Z" }
  ]
}
```

A delivery of the `order.paid` event arrives at the configured webhook endpoint:

```json title="POST to the webhook endpoint"
{
  "id": "evt_01HQ8ZY6TP7Q5W2AE6D4HV",
  "type": "order.paid",
  "created_at": "2025-11-04T10:14:05Z",
  "tenant_id": "ten_01HQ8ZTP4M7RN2VC5XK9BW",
  "webhook_id": "whk_01HQ8ZS9JL3KM1TB4WX8AV",
  "data": {
    "order_id": "ord_01HQ8ZW7PM5N3T8YC4B2FR",
    "customer_id": "cus_01HQ8ZV3KXN4M2T9YB7C6D",
    "payment_id": "pay_01HQ8ZX2RN6P4V9ZD5C3GT",
    "status": "paid",
    "currency": "PLN",
    "total_amount": 34700
  }
}
```

Delivery headers:

```text
Content-Type: application/json
X-Orion-Request-Id: req_01HQ8ZY6TQ8R6X3BF7E5JW
X-Orion-Signature: t=1762251245,v1=8f3c1a9d47b6e05c2f8a1d3b9e7c40f6a2d5b8c1e4f7a0d3b6c9e2f5a8d1b4c7
```

Verify the signature before you process the body, the procedure is described in [Signature verification](../api/webhooks/signature-verification.md).

!!! warning "Do not trust the body without checking the signature"
    The webhook endpoint is public. Without verifying the `X-Orion-Signature` header, anyone can send a forged `order.paid` event and trigger a shipment.

## Troubleshooting common errors

| Code | Symptom | What to do |
| --- | --- | --- |
| `ORN-2001` | `401` on every request | the token has expired or `Authorization` has the wrong format; fetch the token again |
| `ORN-2003` | `403` when creating an order | the `orders:write` scope is missing from the token |
| `ORN-1104` | `422` with a list of fields | a request field is missing or invalid, for example `unit_amount` given as a decimal number |
| `ORN-3001` | `429` with an `X-RateLimit-Reset` header | the sandbox limit of 60 requests per minute has been exceeded; wait until the indicated time |
| `ORN-4002` | `409` on a repeated request | the same `Idempotency-Key` was used with a different body; generate a new key |

## What next

- [Order resources](../api/rest/resources/orders.md): the full resource contract and all status transitions.
- [Payment resources](../api/rest/resources/payments.md): partial captures, refunds and settlements.
- [Event types](../api/webhooks/event-types.md): the schemas of the ten domain events.
- [Authentication with API keys](../guides/authentication/api-keys.md): an alternative to OAuth2.
- [Java SDK](../api/sdk/java.md) and [Python SDK](../api/sdk/python.md): client libraries with retry and idempotency support.

!!! tip "Use ok_test_ keys in the sandbox"
    Instead of an OAuth2 token you can authenticate with an API key in the `X-Orion-Key` header. The sandbox accepts only keys with the `ok_test_` prefix; an `ok_live_` key is rejected with error `ORN-2005`. Test keys never trigger real operations at the payment provider.

## See also

- [Getting started](index.md)
- [Error codes](../api/rest/error-codes.md)
- [OAuth2](../guides/authentication/oauth2.md)
- [Rate limits](../api/rest/rate-limits.md)
