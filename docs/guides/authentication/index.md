# Authentication and authorization

Orion Platform 4.2 LTS recognizes three authentication mechanisms: OAuth2 client credentials, API keys, and `orion-console` session tokens. All of them reach the same filter in `orion-gateway`, which converts them into an internal security context made up of a tenant, a subject, and a set of scopes. Authorization happens later, at the level of the role assigned to the subject.

## Three mechanisms

OAuth2 client credentials
:   The primary way to build machine-to-machine integrations. The client exchanges `client_id` and `client_secret`
    for a short-lived JWT in Keycloak 24 (realm `orion`).

API keys
:   A simpler mechanism for scripts, recurring jobs, and quick prototypes. The secret is long-lived
    and passed in the `X-Orion-Key` header.

Console session tokens
:   Issued only by `orion-console` after a user signs in through OIDC. They are not meant
    for programmatic calls and cannot be obtained outside a browser.

## Comparison

| Mechanism | Use case | Rotation | Permission reach |
| --- | --- | --- | --- |
| OAuth2 client credentials | production integrations, backend services | client secret every 90 days, token every 3600 s | scopes from the request, capped by the client's list |
| API keys | scripts, cron jobs, prototypes | manual, with an overlap period | scopes assigned to the key |
| Console session token | people working in `orion-console` | automatic, 8 h session | the user's full role |

## Principle of least privilege

Every subject should hold only the scopes it needs to do its job. The available scopes are `orders:read`, `orders:write`, `payments:write`, `customers:read`, `customers:write`, and `admin`.

- [x] A separate OAuth2 client for each integration, not one shared client.
- [x] `admin` only for administrative tooling, never for application traffic.
- [x] Read and write separated when an integration only reads data.
- [ ] Sharing a single key across environments - not acceptable.

!!! warning "The `admin` scope does not bypass roles"
    A token with the `admin` scope is still subject to the role assigned to the subject. A subject with the `viewer` role
    and the `admin` scope receives `ORN-2003` when it attempts a write.

## Choosing a mechanism

| Caller | Environment | Mechanism |
| --- | --- | --- |
| human | any | `orion-console` |
| backend service | production | OAuth2 client credentials |
| backend service | sandbox | OAuth2 client credentials |
| one-off script | sandbox | `ok_test_...` key |
| recurring job | production | `ok_live_...` key with an IP allowlist |

## Subpages

| Page | Contents |
| --- | --- |
| [OAuth2 client credentials](oauth2.md) | client registration, token request, JWT structure, secret rotation |
| [API keys](api-keys.md) | key format, creation, restriction, rotation without downtime |
| [Roles and permissions](roles-and-permissions.md) | RBAC model, built-in roles, permission matrix, audit |

## See also

- [Error codes](../../api/rest/error-codes.md)
- [Rate limits](../../api/rest/rate-limits.md)
- [Security](../../operations/security.md)
