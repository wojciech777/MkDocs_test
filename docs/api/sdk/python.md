# Python library

`orion-sdk` version 4.2.1 is the official client library for Orion Platform 4.2 LTS. It requires Python 3.10 or newer, ships inline type hints (the package is marked with `py.typed`, so `mypy` and `pyright` see the full signatures) and is built on `httpx`, which gives the same feature set in the synchronous and the `asyncio` variant. The library is a thin wrapper over the HTTP interface described in [REST API basics](../rest/index.md): every method maps to one endpoint, and the response objects keep the field names used in the JSON payloads.

## Installation

=== "pip"

    ```bash
    python -m pip install "orion-sdk==4.2.1"
    ```

=== "Poetry"

    ```toml title="pyproject.toml"
    [tool.poetry.dependencies]
    python = "^3.10"
    orion-sdk = "4.2.1"
    ```

The `4.2.x` line supports Orion Platform 4.2 LTS and API revision `2026-04-01`. Patch releases within a line never change signatures, and unknown response fields are ignored, so a platform upgrade does not force an immediate library upgrade. The full compatibility matrix is in [Client libraries](index.md).

!!! tip "Pin the exact version"
    Use `orion-sdk==4.2.1` rather than `orion-sdk>=4.2`, and commit the lock file. A floating requirement lets a minor release land in a production build without review, which is the most common cause of unexplained changes in retry and idempotency behavior between deployments.

## Initializing the client

`OrionClient` is created once and reused. The constructor takes the base address, the credentials, the tenant context, the timeouts and the retry budget.

=== "API key"

    ```python title="orion_config.py"
    import os

    from orion_sdk import OrionClient


    def create_client() -> OrionClient:
        return OrionClient(
            api_key=os.environ["ORION_API_KEY"],
            base_url="https://api.orion.example.com/v1",
            tenant="ten_01HQ8ZV3KXN4M2T9YB7C6D",
            connect_timeout=3.0,
            read_timeout=15.0,
            max_retries=3,
        )
    ```

=== "OAuth2 client credentials"

    ```python title="orion_config.py" hl_lines="8 9 10 11"
    import os

    from orion_sdk import OrionClient


    def create_client() -> OrionClient:
        return OrionClient(
            client_id="nordis-checkout",
            client_secret=os.environ["ORION_CLIENT_SECRET"],
            scopes=["orders:read", "orders:write", "payments:write"],
            base_url="https://api.orion.example.com/v1",
            tenant="ten_01HQ8ZV3KXN4M2T9YB7C6D",
            read_timeout=15.0,
        )
    ```

The `api_key` value is an `ok_live_...` key sent in the `X-Orion-Key` header. Read it from the environment (`ORION_API_KEY`) and never from a source file, a `.env` file committed to the repository or a CI job definition; the reasoning and the rotation procedure are in [API keys](../../guides/authentication/api-keys.md).

With `client_id` and `client_secret` the client fetches a token on the first request and refreshes it before the 3600 s TTL elapses, so application code never touches `POST /oauth/token`. Scope names and the claims carried by the token are described in [OAuth2 client credentials](../../guides/authentication/oauth2.md).

`tenant` fills the `X-Orion-Tenant` header and is accepted only for credentials that carry the `admin` scope; without it the request fails with `ORN-2011`. For the sandbox, pass `base_url="https://api.sandbox.orion.example.com/v1"` and an `ok_test_...` key.

### Constructor arguments

| Argument | Default | Description |
| --- | --- | --- |
| `base_url` | `https://api.orion.example.com/v1` | Base API address. |
| `api_key` | `None` | An `ok_live_...` or `ok_test_...` key, sent in `X-Orion-Key`. |
| `client_id`, `client_secret` | `None` | OAuth2 client credentials, mutually exclusive with `api_key`. |
| `scopes` | all assigned | Scopes requested when fetching the token. |
| `tenant` | `None` | Value of the `X-Orion-Tenant` header. |
| `connect_timeout` | `5.0` | Connection handshake limit, in seconds. |
| `read_timeout` | `30.0` | Limit for waiting on a response, in seconds. |
| `max_retries` | `2` | Number of retries for `429` and `5xx`. |
| `retry_backoff` | `0.2` | Base delay in seconds, multiplied exponentially with jitter. |
| `max_connections` | `32` | Size of the HTTP connection pool. |
| `idempotency_key_factory` | UUID v4 | Callable returning `Idempotency-Key` values. |
| `debug` | `False` | Logging of requests and responses at the `DEBUG` level. |

!!! tip "One client per process"
    `OrionClient` is thread-safe and owns the connection pool and the token cache. Build it once, at startup, and inject it into your services. A client created per request refetches the token every time, exhausts the connection budget of 64 per key and produces `ORN-3001` and `ORN-3012` errors. Close it on shutdown, or use it as a context manager, to release the pool.

## Synchronous and asynchronous usage

Both variants expose the same resource namespaces (`customers`, `orders`, `payments`) and the same method names. `AsyncOrionClient` accepts exactly the arguments listed above and returns awaitables.

=== "Sync"

    ```python title="checkout.py"
    from orion_sdk import OrionClient
    from orion_sdk.models import Order

    from orion_config import create_client


    def place_order() -> Order:
        with create_client() as client:
            customer = client.customers.create(
                email="orders@northwindbakery.example.com",
                name="Northwind Bakery Ltd",
                default_currency="PLN",
                idempotency_key="cus-crm-40921",
            )

            order = client.orders.create(
                customer=customer.id,
                currency="PLN",
                channel="web",
                items=[
                    {
                        "sku": "OVEN-KM-200",
                        "name": "KM-200 convection oven",
                        "quantity": 1,
                        "unit_amount": 1299000,
                    }
                ],
                idempotency_key="erp-ORD-2026-04-118",
            )

            return client.orders.retrieve(order.id, expand=["customer", "items"])
    ```

=== "Async (asyncio)"

    ```python title="checkout_async.py"
    import asyncio
    import os

    from orion_sdk import AsyncOrionClient
    from orion_sdk.models import Order


    async def place_order() -> Order:
        async with AsyncOrionClient(
            api_key=os.environ["ORION_API_KEY"],
            base_url="https://api.orion.example.com/v1",
            tenant="ten_01HQ8ZV3KXN4M2T9YB7C6D",
            max_retries=3,
        ) as client:
            customer = await client.customers.create(
                email="orders@northwindbakery.example.com",
                name="Northwind Bakery Ltd",
                default_currency="PLN",
                idempotency_key="cus-crm-40921",
            )

            order = await client.orders.create(
                customer=customer.id,
                currency="PLN",
                channel="web",
                idempotency_key="erp-ORD-2026-04-118",
            )

            return await client.orders.retrieve(order.id)


    if __name__ == "__main__":
        print(asyncio.run(place_order()).id)
    ```

Amounts are integers in the smallest unit of the currency: `1299000` means 12,990.00 PLN. Passing a float raises a local `ValueError` before the request leaves the process, so the server never has to answer with `ORN-1019`.

## Idempotency

Every write method accepts `idempotency_key`, which becomes the `Idempotency-Key` header. When the argument is omitted the client generates a UUID v4, which protects its own automatic retries but not a retry performed by your code after a process restart.

```python title="submit_order.py" hl_lines="9"
from orion_sdk import OrionClient
from orion_sdk.models import Payment


def pay(client: OrionClient, order_id: str, amount: int) -> Payment:
    payment = client.payments.create(
        order=order_id,
        amount=amount,
        currency="PLN",
        idempotency_key=f"pay-{order_id}",
    )
    return client.payments.capture(
        payment.id,
        amount=amount,
        idempotency_key=f"capture-{payment.id}",
    )
```

!!! danger "Derive the key from the business operation"
    The key is remembered for 24 hours together with a fingerprint of the request body. A stable key such as `pay-ord_01HQ8ZV3KXN4M2T9YB7C6D` makes a retry return the original `201` response instead of creating a second payment. A random key generated on retry creates a duplicate; the same key with a changed body fails with `ORN-4002`.

## Pagination

`list()` returns a page object with the `data`, `has_more`, `next_cursor` and `total_estimate` fields, exactly as documented in [Pagination, filtering, and sorting](../rest/pagination.md). `auto_paging_iter()` wraps that page in a generator that follows `next_cursor` for you and yields items one at a time.

```python title="order_report.py"
from collections.abc import Iterator

from orion_sdk import OrionClient
from orion_sdk.models import Order


def paid_orders(client: OrionClient, since: str) -> Iterator[Order]:
    page = client.orders.list(
        status="paid",
        created_after=since,
        sort="created_at:asc",
        limit=100,
    )
    yield from page.auto_paging_iter()


def total_paid(client: OrionClient, since: str) -> int:
    return sum(order.amount_total for order in paid_orders(client, since))
```

Passing `starting_after` explicitly is still possible when you persist a cursor between runs:

```python title="incremental_export.py"
cursor: str | None = load_cursor()

while True:
    page = client.orders.list(limit=100, status="paid", starting_after=cursor)
    for order in page.data:
        export(order)
    if not page.has_more:
        break
    cursor = page.next_cursor
    save_cursor(cursor)
```

!!! note "Page limit and cursor lifetime"
    The default `limit` is 25 and the maximum is 100; a larger value returns `ORN-3004`. Cursors are opaque and valid for 24 hours, and changing `sort` mid-iteration invalidates them with `ORN-1012`. `auto_paging_iter()` performs as many requests as the filter needs, so narrow wide time ranges to stay inside the budget described in [Rate limits](../rest/rate-limits.md).

## Error handling

Every non-success response is raised as an exception derived from `OrionError`. The subclasses follow the `ORN-*` classes, and each instance exposes `code`, `status_code`, `request_id` and, where relevant, `field` and `retry_after`.

```text title="Exception hierarchy"
OrionError
 |- OrionValidationError    ORN-1xxx
 |- OrionNotFoundError      ORN-1023
 |- OrionAuthError          ORN-2xxx
 |- OrionRateLimitError     ORN-3xxx
 |- OrionConflictError      ORN-4xxx
 |- OrionServerError        ORN-5xxx
```

```python title="error_handling.py" hl_lines="18 24"
import logging

from orion_sdk import OrionClient
from orion_sdk.errors import (
    OrionAuthError,
    OrionConflictError,
    OrionError,
    OrionNotFoundError,
    OrionRateLimitError,
    OrionValidationError,
)

log = logging.getLogger(__name__)


def create_order(client: OrionClient, **params: object) -> str | None:
    try:
        return client.orders.create(**params).id
    except OrionValidationError as exc:
        log.warning("Invalid input data: %s (%s, field=%s)", exc, exc.code, exc.field)
        return None
    except OrionAuthError as exc:
        log.error("Credentials rejected: %s request_id=%s", exc.code, exc.request_id)
        raise
    except OrionNotFoundError as exc:
        log.warning("Referenced resource missing: %s", exc.code)
        return None
    except OrionConflictError as exc:
        log.warning("Conflict %s, reading the existing resource", exc.code)
        return None
    except OrionRateLimitError as exc:
        log.warning("Rate limit exceeded, retry after %s s", exc.retry_after)
        raise
    except OrionError as exc:
        log.error(
            "API error %s status=%s request_id=%s",
            exc.code,
            exc.status_code,
            exc.request_id,
        )
        raise
```

| Code | HTTP status | Exception | Recommended reaction |
| --- | --- | --- | --- |
| `ORN-1004` | 400 | `OrionValidationError` | fix the payload, do not retry |
| `ORN-1023` | 404 | `OrionNotFoundError` | check the identifier and the tenant context |
| `ORN-2001` | 401 | `OrionAuthError` | refresh the token or check the API key |
| `ORN-2003` | 403 | `OrionAuthError` | add the missing scope to the OAuth2 client |
| `ORN-3001` | 429 | `OrionRateLimitError` | retry after `retry_after` seconds |
| `ORN-4002` | 409 | `OrionConflictError` | use a new `Idempotency-Key` or read the existing resource |
| `ORN-5002` | 503 | `OrionServerError` | retry with exponential backoff, report it if the error persists |

Branch your logic on `exc.code` only. The `str(exc)` text comes from `error.message`, whose wording may change between releases. The full list is in [Error codes](../rest/error-codes.md).

The `request_id` attribute carries the value of the `X-Orion-Request-Id` header and is required for support tickets.

## Retries and rate limits

With the default `max_retries=2` the client retries by itself:

- `429` responses (`ORN-3001`), waiting out `Retry-After` when the header is present,
- `503` and `504` responses (`ORN-5002`, `ORN-5005`, `ORN-5009`),
- connection errors and read timeouts on `GET` requests.

Delays follow exponential backoff seeded from `retry_backoff`, with random jitter added so that many instances do not retry in the same instant, and are capped at 32 seconds. Retried `POST` requests reuse the `Idempotency-Key` of the original attempt, so a repeat never creates a second order. `400`, `401`, `403`, `404`, `409` and `422` responses are never retried, because the request itself has to change first.

When the budget is exhausted the last error is raised, carrying `retry_after` and the `X-RateLimit-Limit`, `X-RateLimit-Remaining` and `X-RateLimit-Reset` values from the final response, which is enough to drive a circuit breaker in your own code.

!!! warning "Use `max_retries=0` in tests"
    With retries enabled a test asserting on a `429` or `503` path waits out the backoff and then sees the response of the last attempt, which makes the test both slow and misleading. Set `max_retries=0` in fixtures so that one call means exactly one request and the mock assertion counts match.

## Webhook verification with FastAPI

`WebhookVerifier` implements the `v2` signature algorithm with constant-time comparison and a 300 s tolerance window. It needs the exact bytes the platform signed, so read the body with `await request.body()` before anything parses it.

```python title="main.py" hl_lines="20 23 24 27"
import os

from fastapi import FastAPI, Header, HTTPException, Request
from orion_sdk.webhooks import WebhookVerifier

app = FastAPI()

verifier = WebhookVerifier(
    secrets=[
        os.environ["ORION_WEBHOOK_SECRET"],
        os.environ.get("ORION_WEBHOOK_SECRET_PREVIOUS"),
    ],
    tolerance_seconds=300,
)


@app.post("/webhooks/orion", status_code=202)
async def receive(
    request: Request,
    x_orion_signature: str = Header(..., alias="X-Orion-Signature"),
    x_orion_event_id: str = Header(..., alias="X-Orion-Event-Id"),
) -> dict[str, str]:
    raw_body = await request.body()
    if not verifier.is_valid(x_orion_signature, raw_body):
        raise HTTPException(status_code=400, detail="Invalid signature")

    event = verifier.parse(raw_body)
    await queue.enqueue_if_absent(x_orion_event_id, event)
    return {"status": "accepted", "event": event.type}
```

Passing two secrets covers the overlap period after a rotation. A `None` entry is skipped, so a missing `ORION_WEBHOOK_SECRET_PREVIOUS` variable does not break startup.

!!! danger "Order of operations"
    Do not declare a Pydantic model in the handler signature. FastAPI reads and deserializes the body to build that model before your code runs, and re-serializing the parsed object produces different bytes (reordered keys, changed whitespace, normalized numbers), so the signature will never match. Take `Request`, call `await request.body()` first, verify, and only then call `request.json()` or validate the model yourself. The same rule applies to any middleware that touches the body. The algorithm and the header format are described in [Signature verification](../webhooks/signature-verification.md).

The `event.type` value is one of the domain event names, for example `order.created`, `order.paid` or `payment.refunded`. Deliveries are retried by the platform, so deduplicate on the `evt_...` identifier from `X-Orion-Event-Id` as shown above; the delivery model is described in [Webhooks](../webhooks/index.md).

## Testing

Point the tests at the sandbox `https://api.sandbox.orion.example.com/v1` with an `ok_test_...` key. Test keys never trigger real operations at the payment provider, and the sandbox rejects an `ok_live_...` key with `ORN-2005`.

```python title="conftest.py"
from collections.abc import Iterator

import pytest
from orion_sdk import OrionClient


@pytest.fixture(scope="session")
def orion_client() -> Iterator[OrionClient]:
    with OrionClient(
        api_key="ok_test_2QDH8LMN4TXV7BZC5RKP9SFJ3WYA6EGU",
        base_url="https://api.sandbox.orion.example.com/v1",
        tenant="ten_01HQ8ZV3KXN4M2T9YB7C6D",
        max_retries=0,
    ) as client:
        yield client
```

Unit tests should not generate network traffic at all. Because the client uses `httpx`, `respx` can intercept the transport layer and let you assert on the headers the library actually sends.

```python title="test_orders.py"
import httpx
import pytest
import respx
from orion_sdk import OrionClient
from orion_sdk.errors import OrionRateLimitError

BASE_URL = "https://api.sandbox.orion.example.com/v1"


@respx.mock
def test_create_order_sends_idempotency_key(orion_client: OrionClient) -> None:
    route = respx.post(f"{BASE_URL}/orders").mock(
        return_value=httpx.Response(
            201,
            json={
                "id": "ord_01HQ8ZV3KXN4M2T9YB7C6D",
                "object": "order",
                "status": "draft",
                "currency": "PLN",
                "amount_total": 0,
            },
        )
    )

    order = orion_client.orders.create(
        customer="cus_01HQ8ZV3KXN4M2T9YB7C6D",
        currency="PLN",
        idempotency_key="erp-ORD-2026-04-118",
    )

    assert order.id == "ord_01HQ8ZV3KXN4M2T9YB7C6D"
    request = route.calls.last.request
    assert request.headers["Idempotency-Key"] == "erp-ORD-2026-04-118"
    assert request.headers["X-Orion-Tenant"] == "ten_01HQ8ZV3KXN4M2T9YB7C6D"


@respx.mock
def test_rate_limit_is_not_retried_when_disabled(orion_client: OrionClient) -> None:
    route = respx.get(f"{BASE_URL}/orders").mock(
        return_value=httpx.Response(
            429,
            headers={"Retry-After": "12"},
            json={
                "error": {
                    "code": "ORN-3001",
                    "message": "Rate limit exceeded.",
                    "type": "rate_limit_error",
                    "request_id": "req_01HQ8ZV3KXN4M2T9YB7C7E",
                }
            },
        )
    )

    with pytest.raises(OrionRateLimitError) as excinfo:
        orion_client.orders.list(limit=100)

    assert excinfo.value.code == "ORN-3001"
    assert excinfo.value.retry_after == 12
    assert route.call_count == 1
```

A pre-merge checklist that has proved useful:

- [x] `max_retries=0` in every fixture.
- [x] One `respx` route per endpoint, with assertions on `Idempotency-Key`.
- [x] At least one test for each of `ORN-1004`, `ORN-3001` and `ORN-4002`.
- [x] Webhook tests that post raw bytes, not a serialized model.
- [ ] Tests that reach the sandbox from a unit test suite.

## Logging and observability

The library logs through the standard `logging` module under the `orion_sdk` logger. Passing `debug=True` records the method, path, status, duration and `X-Orion-Request-Id` for every call; request bodies are redacted, and the `Authorization` and `X-Orion-Key` headers as well as `whsec_...` values are never written.

```python title="logging_setup.py"
import logging

logging.basicConfig(
    format="%(asctime)s %(levelname)s %(name)s %(message)s",
    level=logging.INFO,
)
logging.getLogger("orion_sdk").setLevel(logging.DEBUG)
logging.getLogger("httpx").setLevel(logging.WARNING)
```

To correlate your own logs with the platform, pass your correlation identifier as `X-Orion-Request-Id` and log the value returned on the response or carried by the exception:

```python title="correlation.py"
order = client.orders.create(
    customer="cus_01HQ8ZV3KXN4M2T9YB7C6D",
    currency="PLN",
    request_id="req_01HQ8ZV3KXN4M2T9YB7C6D",
)
log.info("order created id=%s request_id=%s", order.id, order.request_id)
```

The same value appears in the `request_id` field of every `orion-gateway`, `orion-core` and `orion-ledger` entry produced while handling the call, which turns a single search into the full path from the API to webhook delivery. Query examples are on the [Logs](../../operations/monitoring/logs.md) page.

!!! danger "Do not enable debug logging in production"
    Even with bodies redacted, `debug=True` writes one entry per request and can multiply log volume by tens of times on a busy integration, filling the collector buffer. Enable it temporarily, on a single replica, and prefer `orion_http_requests_total` on the server side for permanent visibility into status distribution.

## See also

- [Client libraries](index.md)
- [Java library](java.md)
- [Signature verification](../webhooks/signature-verification.md)
- [Pagination, filtering, and sorting](../rest/pagination.md)
- [Error codes](../rest/error-codes.md)
