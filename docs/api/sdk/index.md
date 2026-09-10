# Client libraries

Nimbus Software maintains official SDKs for Java and Python, compatible with Orion Platform 4.2 LTS. The libraries wrap the REST API, moving authentication, retries, idempotency and pagination into client code. Integrations in other languages can be based on community libraries or on a generated OpenAPI client.

## Available libraries

| Language | Artifact | Version | Minimum runtime | Support status |
| --- | --- | --- | --- | --- |
| Java | `com.nimbus.orion:orion-sdk-java` (Maven Central) | 4.2.1 | Java 17+ | official, covered by the SLA |
| Python | `orion-sdk` (PyPI) | 4.2.1 | Python 3.10+ | official, covered by the SLA |
| Node.js | `@community/orion-node` | 0.9.x | Node.js 20+ | community, no SLA |
| Go | `github.com/community/orion-go` | 0.4.x | Go 1.22+ | experimental |

!!! warning "Community libraries"
    The Node.js and Go releases are neither built nor audited by Nimbus Software. They are not covered by incident reporting in the support portal, and compatibility with the `2026-04-01` revision is not guaranteed.

## What the SDK does for you

Retries
:   Automatic repeats for `429` and `5xx` with exponential backoff and jitter, honoring the `Retry-After` header.

Idempotency
:   Generation of the `Idempotency-Key` header for every write operation, so a retry does not create a second order or a second payment.

Pagination
:   Iterators that fetch subsequent cursor pages in the background, without passing `cursor` manually.

Webhook verification
:   A ready implementation of the `v2` signature with constant-time comparison and a 300 s tolerance check.

Error mapping
:   Translation of `ORN-*` codes into typed exceptions together with `X-Orion-Request-Id` for support tickets.

Token refresh
:   Fetching and renewing the OAuth2 token before the 3600 s TTL elapses, without any involvement from application code.

## Versioning

The SDK uses semantic versioning tied to the platform line: `4.2.x` supports Orion Platform 4.2 LTS and API revision `2026-04-01`. SDK patch releases are binary compatible with each other and require no code changes. New response fields are ignored by older SDK versions, so upgrading the platform does not force an immediate library upgrade.

| SDK line | Supported platform | API revision | Supported until |
| --- | --- | --- | --- |
| 4.2.x | 4.2 LTS | `2026-04-01` | 2029-06-30 |
| 4.1.x | 4.1 | `2025-10-01` | 2027-03-31 |
| 3.8.x | 3.8 | `2024-11-01` | ended |

!!! tip "Generating your own client"
    The OpenAPI 3.1 specification is published at `https://api.orion.example.com/v1/openapi.json` (test environment: `https://api.sandbox.orion.example.com/v1/openapi.json`). A generated client contains no retry, idempotency or webhook verification logic; you have to implement those parts yourself, following [Signature verification](../webhooks/signature-verification.md).

## Subpages

- [Java library](java.md) - installation, `OrionClientBuilder`, error handling, Spring integration.
- [Python library](python.md) - the synchronous and `asyncio` variants, FastAPI, type hints and `mypy`.

## See also

- [Java library](java.md)
- [Python library](python.md)
- [REST API basics](../rest/index.md)
- [Webhooks](../webhooks/index.md)
