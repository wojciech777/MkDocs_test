# Customers

The `customer` resource represents a buyer within a single tenant. A customer is required to create an order and acts as the anchor point for addresses, tax data, and integration metadata. The key-value `metadata` object is carried over into the `customer.created` and `customer.updated` events.

## Object model

| Field | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | string | read only | a `cus_...` identifier |
| `email` | string | yes | email address, unique within the tenant |
| `name` | string | yes | company name or full name, 1 to 200 characters |
| `tax_id` | string | no | tax or VAT ID, validated against the country of the billing address |
| `phone` | string | no | number in E.164 format, for example `+48221234567` |
| `default_currency` | string | no | `PLN`, `EUR`, or `USD`; defaults to `PLN` |
| `addresses[]` | array | no | list of addresses, see below |
| `metadata` | object | no | up to 20 custom keys |
| `created_at` | string | read only | ISO-8601 UTC timestamp |
| `updated_at` | string | read only | ISO-8601 UTC timestamp |
| `tenant` | string | read only | a `ten_...` identifier |

A single address contains the fields `type` (`billing` or `shipping`), `line1`, `line2`, `city`, `postal_code`, `country` (ISO 3166-1 alpha-2), and `default`.

```json title="Complete customer object"
{
  "id": "cus_01HQ8ZV3KXN4M2T9YB7C6D",
  "object": "customer",
  "email": "orders@northwindbakery.example.com",
  "name": "Northwind Bakery Ltd",
  "tax_id": "PL5261040828",
  "phone": "+48221234567",
  "default_currency": "PLN",
  "addresses": [
    {
      "type": "billing",
      "line1": "14 Sandhill Road",
      "line2": "Unit 3",
      "city": "Warsaw",
      "postal_code": "61-001",
      "country": "PL",
      "default": true
    }
  ],
  "metadata": {
    "segment": "b2b",
    "crm_id": "CRM-40921"
  },
  "created_at": "2026-03-02T11:14:52Z",
  "updated_at": "2026-04-11T08:02:19Z",
  "tenant": "ten_01HQ8ZV3KXN4M2T9YB7C6D"
}
```

## Endpoints

### POST /v1/customers

Creates a customer. Requires the `customers:write` scope.

| Body field | Type | Required | Description |
| --- | --- | --- | --- |
| `email` | string | yes | unique within the tenant |
| `name` | string | yes | 1 to 200 characters |
| `tax_id` | string | no | tax identifier |
| `phone` | string | no | E.164 format |
| `default_currency` | string | no | `PLN`, `EUR`, `USD` |
| `addresses` | array | no | up to 10 entries |
| `metadata` | object | no | up to 20 keys |

=== "curl"

    ```bash
    curl -X POST https://api.orion.example.com/v1/customers \
      -H "X-Orion-Key: ok_live_9f2c41ab7d5e" \
      -H "Idempotency-Key: cus-crm-40921" \
      -H "Content-Type: application/json" \
      -d '{"email":"orders@northwindbakery.example.com","name":"Northwind Bakery Ltd","default_currency":"PLN"}'
    ```

=== "Python"

    ```python
    customer = client.customers.create(
        email="orders@northwindbakery.example.com",
        name="Northwind Bakery Ltd",
        default_currency="PLN",
        idempotency_key="cus-crm-40921",
    )
    ```

=== "Java"

    ```java
    Customer customer = client.customers().create(
        CustomerCreateParams.builder()
            .email("orders@northwindbakery.example.com")
            .name("Northwind Bakery Ltd")
            .defaultCurrency("PLN")
            .build());
    ```

```json title="201 Created response"
{
  "id": "cus_01HQ8ZV3KXN4M2T9YB7C6D",
  "object": "customer",
  "email": "orders@northwindbakery.example.com",
  "name": "Northwind Bakery Ltd",
  "default_currency": "PLN",
  "addresses": [],
  "metadata": {},
  "created_at": "2026-04-12T09:31:04Z",
  "updated_at": "2026-04-12T09:31:04Z",
  "tenant": "ten_01HQ8ZV3KXN4M2T9YB7C6D"
}
```

Possible errors: `ORN-1001`, `ORN-1004`, `ORN-1007`, `ORN-2003`, `ORN-4002`, `ORN-3001`.

### GET /v1/customers/{id}

Returns a single customer. Requires the `customers:read` scope.

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `id` | path | yes | a `cus_...` identifier |
| `expand` | query | no | `addresses`, `orders` |

```bash title="Request"
curl "https://api.orion.example.com/v1/customers/cus_01HQ8ZV3KXN4M2T9YB7C6D" \
  -H "X-Orion-Key: ok_live_9f2c41ab7d5e"
```

Possible errors: `ORN-2001`, `ORN-2003`, `ORN-5002`.

### GET /v1/customers

Returns a list of customers in descending order by `created_at`. Cursor pagination is described in [Pagination](../pagination.md).

| Parameter | Type | Required | Description |
| --- | --- | --- | --- |
| `email` | query | no | exact match, case insensitive |
| `created_after` | query | no | ISO-8601 UTC timestamp |
| `limit` | query | no | defaults to 25, max. 100 |
| `starting_after` | query | no | cursor for the next page |
| `ending_before` | query | no | cursor for the previous page |

??? example "Complete 200 OK response"

    ```json
    {
      "data": [
        {
          "id": "cus_01HQ8ZV3KXN4M2T9YB7C6D",
          "object": "customer",
          "email": "orders@northwindbakery.example.com",
          "name": "Northwind Bakery Ltd",
          "created_at": "2026-03-02T11:14:52Z"
        }
      ],
      "has_more": true,
      "next_cursor": "Y3VzXzAxSFE4WlYzS1hOTTJU",
      "total_estimate": 1842
    }
    ```

Possible errors: `ORN-1012`, `ORN-3004`, `ORN-3001`.

### PATCH /v1/customers/{id}

Updates selected fields following JSON Merge Patch semantics. The `id`, `tenant`, and `created_at` fields cannot be changed.

```json title="Request"
{
  "phone": "+48221234599",
  "metadata": {
    "segment": "b2b_key"
  }
}
```

Possible errors: `ORN-1001`, `ORN-1004`, `ORN-1007`, `ORN-2003`.

### DELETE /v1/customers/{id}

Starts customer anonymization. The response is `202 Accepted` with an `anonymize_after` field.

!!! warning "Anonymization instead of deletion"
    Because of accounting and GDPR requirements the record is not physically deleted. The `email`, `name`, `phone`, and `tax_id` fields and the addresses are overwritten with placeholder values after 30 days, and the related orders keep the `cus_...` identifier without any personal data. The operation is irreversible.

```json title="202 Accepted response"
{
  "id": "cus_01HQ8ZV3KXN4M2T9YB7C6D",
  "object": "customer",
  "anonymize_after": "2026-05-12T09:31:04Z",
  "status": "anonymization_scheduled"
}
```

Possible errors: `ORN-2003`, `ORN-4005` (the customer has orders in the `pending_payment` status), `ORN-5002`.

### POST /v1/customers/{id}/addresses

Adds an address to a customer. At most 10 addresses per customer.

| Body field | Type | Required | Description |
| --- | --- | --- | --- |
| `type` | string | yes | `billing` or `shipping` |
| `line1` | string | yes | street and number |
| `line2` | string | no | address complement |
| `city` | string | yes | city |
| `postal_code` | string | yes | postal code |
| `country` | string | yes | ISO 3166-1 alpha-2 |
| `default` | boolean | no | marks the default address for the given type |

```json title="201 Created response"
{
  "type": "shipping",
  "line1": "14 Sandhill Road",
  "city": "Warsaw",
  "postal_code": "61-001",
  "country": "PL",
  "default": false
}
```

Possible errors: `ORN-1004`, `ORN-1001`, `ORN-4005` (address limit exceeded), `ORN-2003`.

## Metadata

The `metadata` object is meant for storing references to external systems.

- at most **20 keys** per object,
- key: up to 40 characters, `[a-z0-9_]`,
- value: a string of up to **500 characters**,
- `PATCH` merges keys; passing `null` as the value removes the key.

!!! danger "Do not store sensitive data"
    Metadata ends up in webhook payloads and application logs. Never put card numbers, passwords, or health data in it.

## See also

- [Orders](orders.md)
- [Pagination, filtering, and sorting](../pagination.md)
- [Error codes](../error-codes.md)
- [Event types](../../webhooks/event-types.md)
