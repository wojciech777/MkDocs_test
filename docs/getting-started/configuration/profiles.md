# Configuration profiles

A profile is a named set of default values matched to the purpose of an environment. Orion Platform 4.2 provides three profiles: `dev`, `staging` and `prod`. A profile sets the starting point for several dozen keys and additionally enforces some of them regardless of the configuration file, which applies above all to the security restrictions in the `prod` profile.

## Activating a profile

You select the profile with an environment variable or with a key in the configuration file. The variable has the higher priority.

```bash
export ORION_PROFILE=prod
orion config validate --config /etc/orion/orion.yaml
```

```yaml title="/etc/orion/orion.yaml"
profile: prod
```

The active profile is visible in the `GET /healthz` response, in the `profile` field of every log entry and in the header of the `orion-console` panel.

!!! note "The profile is immutable at runtime"
    The `profile` key is not one of the dynamic keys. Changing the profile requires a restart of all components, because it affects the defaults of the whole configuration, including the connection pool size and the TLS policy.

## Differences between the profiles

| Key / behavior | `dev` | `staging` | `prod` |
| --- | --- | --- | --- |
| `logging.level` | `debug` | `info` | `info` |
| `logging.format` | `text` | `json` | `json` (enforced) |
| Migrations at startup | yes, automatic | no | no (blocked) |
| TLS for inbound traffic | optional | required with a warning | required (enforced) |
| `database.sslMode` | `prefer` | `require` | `require` (enforced) |
| `database.poolSize` | `5` | `20` | `40` |
| `rateLimit.requestsPerMinute` | `60` | `60` | `600` |
| `rateLimit.burst` | `100` | `100` | `100` |
| `observability.tracing.samplingRatio` | `1.0` | `0.25` | `0.05` |
| Retention of the `audit` schema | 7 days | 30 days | 90 days |
| Retention of DLQ entries | 3 days | 14 days | 30 days |
| Payment sandbox | enabled | enabled | disabled (blocked) |
| Cart (`draft`) expiry | after 1 h | after 24 h | after 72 h |
| Error detail in the API response | full call stack | shortened message | the `ORN-*` code only |
| API keys | `ok_test_...` and `ok_live_...` | `ok_test_...` | `ok_live_...` only |
| `webhooks.maxRetries` | `2` | `5` | `5` |
| Verification of the webhook receiver certificate | skipped | required | required (enforced) |

!!! danger "Never run the dev profile in production"
    The `dev` profile prints full call stacks in API responses, skips verification of webhook receiver certificates, allows text-format logs without masking some context fields and runs schema migrations automatically on every component start. In a production environment that means internal details are exposed, webhook deliveries are open to a man-in-the-middle attack, and there is a risk of an unplanned schema change during a pod restart.

## Overriding profile values

Profile values can be overridden in three ways, in order of increasing priority: with a profile file, with keys in `orion.yaml`, and with `ORION_*` variables.

The profile file is named `orion.<profile>.yaml` and sits next to the main file. It is loaded only when it matches the active profile.

```yaml title="/etc/orion/orion.dev.yaml" hl_lines="2 8"
logging:
  level: trace

database:
  poolSize: 10

rateLimit:
  requestsPerMinute: 6000

features:
  eventReplayApi: true
```

Checking which source each value comes from:

```bash
orion config validate --config /etc/orion/orion.yaml --explain
```

```text
profile                              prod       (source: environment ORION_PROFILE)
logging.level                        info       (source: profile:prod)
database.poolSize                    40         (source: orion.yaml:12)
rateLimit.requestsPerMinute          600        (source: profile:prod)
rateLimit.burst                      150        (source: console override)
observability.tracing.samplingRatio  0.05       (source: profile:prod)
features.partialCapture              true       (source: orion.yaml:58)
```

!!! warning "The prod profile rejects some overrides"
    Keys marked as "enforced" in the table cannot be lowered in the `prod` profile. Trying to set `logging.format: text` or `database.sslMode: disable` fails with the `ORN-1212` error and the component does not start. If you need that behavior for diagnostics, use the `staging` profile in an isolated environment.

## Feature flags

Flags in the `features` section enable behaviors that are not the default in the 4.2 line. All of them are dynamic and can be toggled in the `orion-console` panel without a restart.

| Flag | Default (`prod`) | Description | Effect of enabling it |
| --- | --- | --- | --- |
| `feature.partialCapture` | enabled | Capturing part of the authorized amount | `POST /payments/{id}/capture` accepts an `amount` field lower than the authorized one |
| `feature.eventReplayApi` | disabled | Replaying events over REST | exposes a replay endpoint next to the `orion events replay` command |
| `feature.strictIdempotency` | disabled | Stricter checking of the `Idempotency-Key` header | a missing header on a write request returns `ORN-1105` instead of being tolerated |
| `feature.multiCurrencyLedger` | disabled | A multi-currency ledger per tenant | allows different currencies within one tenant; irreversible after the first write |
| `feature.webhookBatching` | disabled | Batching of webhook deliveries | delivers up to 25 events in a single HTTP request; requires a change on the receiver side |

!!! warning "Two flags change the integration contract"
    `feature.strictIdempotency` and `feature.webhookBatching` change behavior that API clients can see. Enable them in `staging` first, and only after confirming with your integrators. `feature.multiCurrencyLedger` is one-way: once the first write in a second currency happens, it cannot be disabled without restoring the ledger from a backup.

??? info "Checking the flag state from the command line"
    ```bash
    orion config validate --config /etc/orion/orion.yaml --explain | grep '^features'
    ```

    ```text
    features.partialCapture         true    (source: profile:prod)
    features.eventReplayApi         false   (source: profile:prod)
    features.strictIdempotency      true    (source: console override)
    features.multiCurrencyLedger    false   (source: profile:prod)
    features.webhookBatching        false   (source: profile:prod)
    ```

## Recommended profile assignment

| Environment | Profile | Rationale |
| --- | --- | --- |
| Workstation, Docker Compose | `dev` | fast diagnostics, automatic migrations, full trace sampling |
| Partner integration environment | `staging` | behavior close to production, `ok_test_...` keys, payment sandbox |
| Pre-production and load tests | `prod` | the same enforcements as production, so test results are meaningful |
| Production | `prod` | full security enforcements and the target limits |

## See also

- [Configuration](index.md)
- [Environment variables](environment-variables.md)
- [Rate limits](../../api/rest/rate-limits.md)
- [Security](../../operations/security.md)
