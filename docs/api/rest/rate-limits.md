# Rate limits

Rate limits protect the shared `orion-gateway` infrastructure against uneven load and provide predictable response times for all tenants. Limits are enforced at the edge, before resource level authentication, and rely on counters kept in Redis 7. This page describes the limiting model, the informational headers, the recommended client strategy, and batch mode.

## Limiting model

- **60 second sliding window**, the counter does not reset on the full minute but reflects the last 60 seconds.
- **Burst of 100**, a token bucket allows a momentary spike of up to 100 requests and is refilled evenly up to the base limit.
- **Limit key**, the pair of API key (or the `client_id` of an OAuth2 token) plus the endpoint pattern. Individual endpoints have independent counters, but they all add up to the global limit of the key.
- **Probes**, `GET /healthz` and `GET /readyz` are not counted.
- **Outgoing webhooks** do not consume the tenant's inbound limit.

## Limits per environment

| Environment | Global limit per key | Burst | Hourly limit |
| --- | --- | --- | --- |
| Production | 600 requests/min | 100 | 10,000 requests/h |
| Sandbox | 60 requests/min | 100 | 10,000 requests/h |

## Limits per endpoint

| Endpoint | Limit | Notes |
| --- | --- | --- |
| `POST /v1/orders` | 120/min | counted together with `POST /v1/orders/batch` |
| `POST /v1/payments` | 60/min | shared across all payment methods |
| any `GET` | 600/min | a shared read counter |
| `POST /oauth/token` | 30/min | cache the token for its full TTL of 3600 s |

!!! warning "Do not fetch a token before every request"
    The most common cause of `ORN-3001` reports is a missing OAuth2 token cache. See [OAuth2](../../guides/authentication/oauth2.md).

## Informational headers

| Header | Example | Description |
| --- | --- | --- |
| `X-RateLimit-Limit` | `600` | the limit in effect for this counter |
| `X-RateLimit-Remaining` | `43` | requests remaining in the window |
| `X-RateLimit-Reset` | `1776072664` | Unix timestamp of a full window refill |
| `Retry-After` | `12` | number of seconds until the next allowed attempt |

The `X-RateLimit-*` headers are returned with every response, including `2xx`.

## The 429 response

```http title="429 Too Many Requests response" hl_lines="2 5"
HTTP/1.1 429 Too Many Requests
Retry-After: 12
X-RateLimit-Limit: 120
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1776072664
Content-Type: application/json
```

```json title="ORN-3001 error body"
{
  "error": {
    "code": "ORN-3001",
    "message": "Rate limit exceeded for the POST /v1/orders endpoint.",
    "type": "rate_limit_error",
    "field": null,
    "request_id": "req_01HQ8ZV3KXN4M2T9YB7C6D",
    "doc_url": "https://docs.orion.example.com/api/rest/rate-limits/"
  }
}
```

## Recommended client strategy

Use exponential backoff with jitter and respect `Retry-After` whenever it is present. At most 5 attempts, with an upper delay bound of 32 seconds. Always retry `POST` requests with the same `Idempotency-Key` header.

=== "Python"

    ```python
    import random, time
    import httpx

    def send(client, request, max_attempts=5):
        for attempt in range(max_attempts):
            response = client.send(request)
            if response.status_code != 429:
                return response
            retry_after = response.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2 ** attempt, 32)
            time.sleep(delay + random.uniform(0, 0.5))
        raise RuntimeError("Rate limit did not clear after 5 attempts")
    ```

=== "Java"

    ```java
    HttpResponse<String> send(HttpClient client, HttpRequest request) throws Exception {
        for (int attempt = 0; attempt < 5; attempt++) {
            HttpResponse<String> response = client.send(request, BodyHandlers.ofString());
            if (response.statusCode() != 429) {
                return response;
            }
            long delay = response.headers().firstValue("Retry-After")
                .map(Long::parseLong)
                .orElse(Math.min(1L << attempt, 32L));
            Thread.sleep(delay * 1000 + ThreadLocalRandom.current().nextInt(500));
        }
        throw new IllegalStateException("Rate limit did not clear after 5 attempts");
    }
    ```

!!! tip "Jitter is not optional"
    Without random spread, all client instances retry at the same moment and keep the limit permanently exhausted.

## Long term limits

| Limit | Value | Scope |
| --- | --- | --- |
| Requests per hour | 10,000 | API key |
| Active webhooks | 200 | tenant |
| Orders in batch mode | 100/s | tenant |
| Concurrent connections | 64 | API key |
| Request body size | 1 MiB | request |

Exceeding the hourly limit returns `ORN-3001` with `Retry-After` pointing at the end of the current hour.

## Batch mode

`POST /v1/orders/batch` accepts up to **50** orders in a single request and returns `202 Accepted` with a list of results in input order. Items are processed independently: an error in one does not invalidate the others.

```json title="POST /v1/orders/batch request"
{
  "items": [
    {"customer": "cus_01HQ8ZV3KXN4M2T9YB7C6D", "currency": "PLN", "channel": "web"},
    {"customer": "cus_01HQ8ZV3KXN4M2T9YB7C7E", "currency": "EUR", "channel": "marketplace"}
  ]
}
```

```json title="202 Accepted response"
{
  "results": [
    {"index": 0, "status": "created", "id": "ord_01HQ8ZV3KXN4M2T9YB7C6D"},
    {"index": 1, "status": "error", "error": {"code": "ORN-1007", "message": "The EUR currency is not enabled for this tenant."}}
  ],
  "created": 1,
  "failed": 1
}
```

A batch request consumes one token of the global limit plus as many tokens of the `POST /v1/orders` counter as it contains items.

## Monitoring consumption

- The `https://console.orion.example.com` console shows limit consumption in the Integrations section over 1 min, 1 h, and 24 h windows for every key.
- The `orion_http_requests_total` metric with the `endpoint`, `status`, and `tenant` labels lets you build an alert on the share of `429` responses. See [Metrics](../../operations/monitoring/metrics.md) for details.
- Recommended alert threshold: a share of `429` above 1 percent of requests over a 5 minute period.

## Raising a limit

A request to raise a limit requires the `ten_...` tenant identifier, the API key prefix, the target limit value, and a volume justification (average and peak requests per minute). A decision is issued within 3 business days. Form and contact channels: [Support](../../support.md).

## See also

- [Error codes](error-codes.md)
- [REST API basics](index.md)
- [Scaling](../../operations/scaling.md)
- [Alerts](../../operations/monitoring/alerts.md)
