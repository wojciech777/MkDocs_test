# OAuth2 client credentials

The OAuth2 client credentials grant is the recommended way to authenticate production integrations with Orion Platform. The client exchanges a `client_id` / `client_secret` pair for a JWT access token with a lifetime of 3600 seconds and then presents it in the `Authorization: Bearer` header. Tokens are issued by Keycloak 24 running in the `orion` realm, and `orion-gateway` verifies the signature locally against a downloaded set of public keys.

## Client registration

Clients are registered by a user with the `owner` or `admin` role. You can register a client in the console at `https://console.orion.example.com` (section **Integrations -> OAuth2 clients**) or from the CLI.

```bash title="Client registration"
orion tenant create --id ten_01HQ8ZV3KXN4M2T9YB7C6D --name "Nordis Store" \
  --profile prod

orion config validate --file /etc/orion/orion.yaml
```

In the console, fill in:

| Field | Example value | Notes |
| --- | --- | --- |
| Client name | `nordis-checkout` | lowercase letters, digits, and `-` only |
| Type | confidential | public clients are not supported |
| Allowed scopes | `orders:write`, `payments:write` | the maximum set the client may request |
| Tenant | `ten_01HQ8ZV3KXN4M2T9YB7C6D` | written into the `ten` claim |
| Secret rotation | 90 days | email warning 14 days in advance |

!!! danger "The secret is shown only once"
    `client_secret` is displayed only when the client is created and after a rotation. Nimbus Software
    does not store it in a reversible form and cannot recover it for you.

## Requesting a token

=== "curl"

    ```bash
    curl -sS -X POST https://api.orion.example.com/v1/oauth/token \
      -H "Content-Type: application/x-www-form-urlencoded" \
      -d "grant_type=client_credentials" \
      -d "client_id=nordis-checkout" \
      -d "client_secret=$ORION_CLIENT_SECRET" \
      -d "scope=orders:write payments:write"
    ```

=== "Java"

    ```java title="OrionTokenClient.java"
    var form = "grant_type=client_credentials"
        + "&client_id=nordis-checkout"
        + "&client_secret=" + URLEncoder.encode(secret, UTF_8)
        + "&scope=" + URLEncoder.encode("orders:write payments:write", UTF_8);

    var request = HttpRequest.newBuilder()
        .uri(URI.create("https://api.orion.example.com/v1/oauth/token"))
        .header("Content-Type", "application/x-www-form-urlencoded")
        .POST(HttpRequest.BodyPublishers.ofString(form))
        .build();

    var response = HttpClient.newHttpClient()
        .send(request, HttpResponse.BodyHandlers.ofString());
    ```

=== "Python"

    ```python title="orion_token.py"
    import os, requests

    response = requests.post(
        "https://api.orion.example.com/v1/oauth/token",
        data={
            "grant_type": "client_credentials",
            "client_id": "nordis-checkout",
            "client_secret": os.environ["ORION_CLIENT_SECRET"],
            "scope": "orders:write payments:write",
        },
        timeout=10,
    )
    response.raise_for_status()
    token = response.json()["access_token"]
    ```

The response:

```json title="200 OK"
{
  "access_token": "*****************************************************",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "orders:write payments:write",
  "issued_at": "2026-02-11T09:14:22Z"
}
```

## Token structure

The token is a signed JWT (RS256). The claims below are verified by `orion-gateway`.

| Claim | Example | Meaning |
| --- | --- | --- |
| `iss` | `https://api.orion.example.com/v1/oauth` | issuer; must match the gateway configuration |
| `sub` | `nordis-checkout` | OAuth2 client identifier |
| `aud` | `orion-gateway` | intended audience of the token |
| `ten` | `ten_01HQ8ZV3KXN4M2T9YB7C6D` | tenant whose context the token acts in |
| `scope` | `orders:write payments:write` | space-separated scopes |
| `exp` | `1770801262` | expiry timestamp (epoch seconds) |
| `jti` | `evt_01HQ8ZV3KXN4M2T9YB7C6D` | unique token identifier, used in logs |

??? info "Token payload preview"
    ```json
    {
      "iss": "https://api.orion.example.com/v1/oauth",
      "sub": "nordis-checkout",
      "aud": "orion-gateway",
      "ten": "ten_01HQ8ZV3KXN4M2T9YB7C6D",
      "scope": "orders:write payments:write",
      "exp": 1770801262,
      "jti": "evt_01HQ8ZV3KXN4M2T9YB7C6D"
    }
    ```

## Caching and refreshing

The client credentials grant does not issue a `refresh_token`. You obtain a new token by repeating the request, but do not do this on every API call.

!!! warning "Do not fetch a token for every request"
    The `/oauth/token` endpoint counts toward the limit of 600 requests/min per key. An integration that fetches
    a token for every order will exhaust the limit and receive `ORN-3001`. Cache the token in process
    memory and refresh it when fewer than ==300 seconds== remain until `exp`.

```python title="Token cache with a safety margin" hl_lines="6 7"
_cache = {"token": None, "expires_at": 0}

def get_token() -> str:
    now = time.time()
    if _cache["token"] and now < _cache["expires_at"] - 300:
        return _cache["token"]
    payload = fetch_token()
    _cache["token"] = payload["access_token"]
    _cache["expires_at"] = now + payload["expires_in"]
    return _cache["token"]
```

## Scopes and operations

| Scope | Allowed operations |
| --- | --- |
| `orders:read` | read orders and order line items |
| `orders:write` | create orders, change status, cancel |
| `payments:write` | authorize, capture, and refund payments |
| `customers:read` | read customer records |
| `customers:write` | create and update customers |
| `admin` | manage webhooks, roles, keys, and tenant configuration |

## Common errors

| Code | HTTP | Cause | Action |
| --- | --- | --- | --- |
| `ORN-2001` | 401 | token expired, has a bad signature, or the wrong `aud` | fetch a new token, check that `iss` matches |
| `ORN-2003` | 403 | the request requires a scope the token does not carry | add the scope to the client and request it in `scope` |
| `ORN-1004` | 400 | missing `grant_type` or `client_id` parameter | fix the request body |
| `ORN-3001` | 429 | token fetched too frequently | enable token caching |
| `ORN-5002` | 503 | Keycloak unavailable | check status and retry with backoff |

## Rotating the client secret

1. In the console, select the client and use **Generate new secret**. The old secret stays active for 48 hours.
2. Store the new secret in your secret store as `ORION_CLIENT_SECRET_NEXT`.
3. Deploy a configuration in which the application tries `ORION_CLIENT_SECRET_NEXT` first and falls back to `ORION_CLIENT_SECRET`.
4. Confirm in the `orion-gateway` logs that tokens are being issued for the new secret (the `secret_generation` field).
5. Replace `ORION_CLIENT_SECRET` with the new value and remove the temporary variable.
6. Revoke the old secret with the **Revoke previous** button before the 48 hours elapse.

!!! tip "Rotation without downtime"
    Because tokens live for 3600 s, tokens issued before the old secret was revoked keep working
    until they expire naturally. There is no need to restart the integration.

## See also

- [API keys](api-keys.md)
- [Roles and permissions](roles-and-permissions.md)
- [Error codes](../../api/rest/error-codes.md)
- [Rate limits](../../api/rest/rate-limits.md)
