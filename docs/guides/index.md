# Guides

Guides describe how to carry out a specific task in Orion Platform 4.2 LTS from start to finish. Unlike the API reference, which exhaustively documents individual resources and fields, a guide walks through a scenario and explains the design decisions behind it. Every guide assumes you have a working installation and access to the console at `https://console.orion.example.com`.

## Guide or reference

| Question | Go to |
| --- | --- |
| "How do I connect an integration to production?" | guide |
| "What fields does the order object have?" | [REST reference](../api/rest/index.md) |
| "Why did an event end up in the DLQ?" | guide |
| "What does code ORN-4002 mean?" | [error codes](../api/rest/error-codes.md) |
| "How do I install the platform?" | [getting started](../getting-started/index.md) |

## List of guides

| Topic | Audience | Level | Time |
| --- | --- | --- | --- |
| [Authentication and authorization](authentication/index.md) | integrators, administrators | basic | 15 min |
| [OAuth2 client credentials](authentication/oauth2.md) | integrators | basic | 25 min |
| [API keys](authentication/api-keys.md) | integrators | basic | 20 min |
| [Roles and permissions](authentication/roles-and-permissions.md) | tenant administrators | intermediate | 30 min |
| [Event processing](events/index.md) | architects, integrators | intermediate | 20 min |
| [Queues and topics](events/queues.md) | platform operators | intermediate | 35 min |
| [Retries and idempotency](events/retries.md) | integrators, operators | advanced | 40 min |
| [Migrations and upgrades](migrations/index.md) | platform operators | intermediate | 25 min |
| [Migrate from 3.8 LTS to 4.2 LTS](migrations/from-3-8-to-4-2.md) | platform operators | advanced | 4 h + maintenance window |
| [Multi-tenancy](multi-tenancy.md) | architects, administrators | advanced | 45 min |

## Suggested order

If you are deploying Orion Platform for the first time, work through the guides in this order:

1. [Authentication and authorization](authentication/index.md) - obtain a valid token and the minimum set of scopes.
2. [Roles and permissions](authentication/roles-and-permissions.md) - separate access between people and integrations.
3. [Event processing](events/index.md) - understand how state changes reach your systems.
4. [Multi-tenancy](multi-tenancy.md) - only if you serve more than one organization.

!!! tip "Sandbox environment"
    You can safely reproduce every command and call from these guides against
    `https://api.sandbox.orion.example.com/v1` using an `ok_test_...` key.
    The sandbox has a lower rate limit (60/min) but an identical data model.

## Conventions

`<TOKEN>`
:   a Bearer token obtained as described in [OAuth2 client credentials](authentication/oauth2.md).

`<TENANT>`
:   a tenant identifier in the format `ten_01HQ8ZV3KXN4M2T9YB7C6D`.

`orion`
:   the command line interface shipped with the `orion-gateway:4.2.1` image.

## See also

- [First order](../getting-started/first-order.md)
- [REST API reference](../api/rest/index.md)
- [Operations and maintenance](../operations/index.md)
- [Frequently asked questions](../faq.md)
