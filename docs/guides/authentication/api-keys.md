# API keys

API keys are a simplified alternative to OAuth2: a single long-lived secret passed in the `X-Orion-Key` header, with no separate token exchange step. They work well for scripts, recurring jobs, and internal tools where maintaining a token cache is disproportionate effort. You pay for that convenience with weaker accountability, because a key carries no JWT claims and its validity does not end on its own.

## When to use a key and when to use OAuth2

| Situation | Recommendation |
| --- | --- |
| Backend service serving customer traffic | [OAuth2](oauth2.md) |
| Cron job exporting orders once a day | `ok_live_...` key |
| Prototype and manual testing | `ok_test_...` key |
| Integration that needs different scopes on different paths | [OAuth2](oauth2.md) |
| Platform operator CLI tool | `ok_live_...` key with an IP restriction |

## Key format

A key consists of an environment prefix and a 32-character random part:

```text
ok_live_9F4KQ2ZT7MXB3NRV6HCD8SPJ5WYA1EGL   # production
ok_test_2QDH8LMN4TXV7BZC5RKP9SFJ3WYA6EGU   # sandbox
```

`ok_live_`
:   works only against `https://api.orion.example.com/v1`.

`ok_test_`
:   works only against `https://api.sandbox.orion.example.com/v1`, limit 60 requests/min.

!!! note "Environment detection"
    `orion-gateway` rejects an `ok_test_` key in production with code `ORN-2001`, even if the key
    is spelled correctly. This prevents accidentally pointing at the wrong base URL.

## Creating a key

=== "Console"

    1. Open `https://console.orion.example.com` and go to **Integrations -> API keys**.
    2. Choose **New key** and give it a descriptive name (for example `nightly-reports`).
    3. Select the scopes: `orders:read`, `customers:read`.
    4. Set an expiry date and an optional list of IP addresses.
    5. Copy the secret; it will not be visible once you close the dialog.

=== "CLI"

    ```bash
    orion tenant create --id ten_01HQ8ZV3KXN4M2T9YB7C6D --name "Nordis" --profile prod

    orion config validate --file /etc/orion/orion.yaml
    ```

=== "Result"

    ```json title="Console response"
    {
      "id": "usr_01HQ8ZV3KXN4M2T9YB7C6D",
      "name": "nightly-reports",
      "prefix": "ok_live_9F4KQ2ZT",
      "secret": "****************************************",
      "scopes": ["orders:read", "customers:read"],
      "expires_at": "2026-08-31T23:59:59Z",
      "tenant": "ten_01HQ8ZV3KXN4M2T9YB7C6D"
    }
    ```

## Using the header

```bash title="Reading orders with an API key"
curl -sS "https://api.orion.example.com/v1/orders?limit=50" \
  -H "X-Orion-Key: $ORION_API_KEY" \
  -H "X-Orion-Tenant: ten_01HQ8ZV3KXN4M2T9YB7C6D" \
  -H "X-Orion-Request-Id: $(uuidgen)"
```

The `Authorization` and `X-Orion-Key` headers are mutually exclusive. If you send both, the gateway returns `ORN-1004`.

## Restricting a key

| Restriction | Field | Example | Effect of a violation |
| --- | --- | --- | --- |
| Scopes | `scopes` | `["orders:read"]` | `ORN-2003` |
| IP allowlist | `ip_allowlist` | `["203.0.113.0/24"]` | `ORN-2001` |
| Expiry | `expires_at` | `2026-08-31T23:59:59Z` | `ORN-2001` |
| Tenant | `tenant` | `ten_01HQ8ZV3KXN4M2T9YB7C6D` | `ORN-2003` |
| Rate limit | `rate_limit` | `120` | `ORN-3001` |

!!! tip "Always set an expiry date"
    A key without `expires_at` is valid indefinitely and usually stays in circulation long after the
    integration that used it has been retired. The maximum recommended lifetime is 12 months.

## Rotation without downtime

Orion Platform allows two active keys to exist at once under one name, which gives you an overlap period.

1. Create a new key with an identical set of scopes and the name `nightly-reports-v2`.
2. Store the secret in your secret store. Do not delete the old value yet.
3. Deploy the application with the new key on a single instance and check the logs for the absence of `ORN-2001`.
4. Roll the deployment out to the remaining instances. Keep the overlap period at ==7 days==.
5. After 7 days, check the **Last used** column for the old key in the console. If it is older than 72 hours, continue.
6. Revoke the old key and remove its secret from the store.

## Storing secrets

!!! danger "Never put a key in a repository"
    Treat an `ok_live_...` key in a source file, `docker-compose.yml`, an `.env` file, or your command
    history as disclosed the moment it was written, even if the commit was later removed. Git history
    retains objects after a file changes, and an internal repository is not a security boundary.

Recommended storage locations:

- [x] A Kubernetes secret mounted as the `ORION_API_KEY` variable.
- [x] An external secret store with access auditing.
- [x] An encrypted configuration file with the key kept outside the repository.
- [ ] A variable in a CI job definition that is visible in the logs.
- [ ] An `orion.yaml` file inside the container image.

## Detecting disclosure and revoking

The console flags a key as **suspicious** when at least one of the following occurs:

- requests from more than 10 different autonomous systems within an hour,
- calls from an address outside `ip_allowlist` (even rejected ones),
- a sharp increase in `orion_http_requests_total` with that key's label above five times the median.

Response procedure:

1. In the console, use **Revoke immediately**. The key stops working within 5 seconds.
2. Review the operations performed with the key since the moment of suspicion in the `audit` schema.
3. Report an incident to `support@nimbus.example.com` (P1, 30 min response) if any write occurred.
4. Issue a new key and rotate it as described above, skipping the overlap period.

## Rate limits

A production key is subject to a limit of 600 requests/min with a burst of 100; a sandbox key is limited to 60 requests/min. Responses include `X-RateLimit-Limit`, `X-RateLimit-Remaining`, and `X-RateLimit-Reset`. The algorithm and the behavior on exceeding the limit are described in [rate limits](../../api/rest/rate-limits.md).

## See also

- [OAuth2 client credentials](oauth2.md)
- [Roles and permissions](roles-and-permissions.md)
- [Rate limits](../../api/rest/rate-limits.md)
- [Security](../../operations/security.md)
