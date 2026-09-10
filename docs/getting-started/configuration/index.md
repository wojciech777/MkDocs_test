# Configuration

Orion Platform reads its configuration from several sources with a fixed precedence: profile defaults, the `orion.yaml` file, `ORION_*` environment variables, command line flags and the settings saved by an operator in the console. All components use the same file and the same set of keys; they differ only in which sections they read. This page describes the shared rules, and the details are on the subpages.

## Precedence order

Sources are merged from the lowest priority to the highest:

1. **Defaults built into the component** - fixed for release 4.2.1.
2. **Profile defaults** - `dev`, `staging` or `prod`, see [Configuration profiles](profiles.md).
3. **The `orion.yaml` file** - `/etc/orion/orion.yaml` by default, together with the fragments from `conf.d/` merged in alphabetical order of file names.
4. **`ORION_*` environment variables** - see [Environment variables](environment-variables.md).
5. **Command line flags** - for example `--listen`, `--config`, `--log-level`.
6. **Values saved in the `orion-console` panel** - only keys marked as dynamic, stored in the database and shared by all replicas.

!!! note "Merging, not replacing"
    Merging happens at the level of individual keys, not whole sections. Setting `ORION_LOG_LEVEL=debug` does not erase the remaining keys of the `logging` section from `orion.yaml`. Lists are the exception: a list provided by a higher-priority source replaces the list entirely.

## Example orion.yaml

```yaml title="/etc/orion/orion.yaml" hl_lines="1 6 22 34"
profile: prod

server:
  # The listen address is overridden by the --listen flag in the systemd units
  host: 0.0.0.0
  publicUrl: https://api.orion.example.com/v1
  requestTimeout: 30s

database:
  url: postgres://postgres.internal:5432/orion
  username: orion_app
  # The _FILE suffix points to a file holding the secret, not to the value itself
  passwordFile: /etc/orion/secrets/db-password
  poolSize: 40
  statementTimeout: 30s
  schemas:
    core: core
    ledger: ledger
    audit: audit

kafka:
  brokers: [kafka-1.internal:9092, kafka-2.internal:9092, kafka-3.internal:9092]
  topic: orion.events.v1
  dlqTopic: orion.events.v1.dlq
  producerAcks: all

redis:
  url: redis://redis.internal:6379/0
  passwordFile: /etc/orion/secrets/redis-password

rateLimit:
  # 600 requests per minute per key, with a short-term excess of up to 100
  requestsPerMinute: 600
  burst: 100

auth:
  oidc:
    issuer: https://sso.example.com/realms/orion
    clientId: orion-gateway
    clientSecretFile: /etc/orion/secrets/oidc-client-secret
    tokenTtl: 3600s
  apiKeys:
    enabled: true
    header: X-Orion-Key

webhooks:
  signingSecretFile: /etc/orion/secrets/webhook-signing-secret
  maxRetries: 5
  timeout: 10s

objectStorage:
  endpoint: https://s3.internal:9000
  bucket: orion-artifacts

logging:
  level: info
  format: json

observability:
  metricsEnabled: true
  tracing:
    enabled: true
    samplingRatio: 0.05

features:
  partialCapture: true
  eventReplayApi: false
```

## Configuration validation

Run the validation before every start and after every change to the file. The command checks the syntax, the types, the interdependencies between keys and the presence of the secret files.

```bash
orion config validate --config /etc/orion/orion.yaml
```

Result for a correct configuration:

```text
config validate: OK (profile=prod, 61 keys resolved, 4 secret files readable)
```

Result when there are errors:

```text
config validate: FAILED (3 errors, 1 warning)

ORN-1201  database.poolSize: value 800 exceeds maximum 500
          source: /etc/orion/orion.yaml:14

ORN-1204  kafka.brokers: host "kafka-4.internal:9092" is not resolvable
          source: environment ORION_KAFKA_BROKERS

ORN-1207  webhooks.signingSecretFile: /etc/orion/secrets/webhook-signing-secret
          is not readable by user "orion" (mode 0600, owner root:root)

WARN      auth.oidc.tokenTtl: 7200s exceeds recommended 3600s

exit status 1
```

!!! tip "Validation in a CI pipeline"
    `orion config validate` exits with code 1 on the first error, so it works well as a deployment gate. Add the `--strict` flag to make warnings produce a non-zero exit code as well.

## Reloading without a restart

Some keys are read on every request and can be changed on a running instance. A reload is triggered by the `SIGHUP` signal (`systemctl reload orion-core`) or by saving a value in the `orion-console` panel.

| Key | Dynamic | How to change it | Notes |
| --- | --- | --- | --- |
| `logging.level` | yes | `SIGHUP`, console | the change is visible within a few seconds |
| `rateLimit.requestsPerMinute` | yes | console | counters in Redis are not reset |
| `rateLimit.burst` | yes | console | |
| `webhooks.maxRetries` | yes | `SIGHUP`, console | applies to new deliveries |
| `webhooks.timeout` | yes | `SIGHUP` | |
| `observability.tracing.samplingRatio` | yes | `SIGHUP`, console | |
| `features.*` | yes | console | see [Configuration profiles](profiles.md) |
| `server.requestTimeout` | yes | `SIGHUP` | applies to new connections |
| `database.url` | no | restart | changing the connection pool requires a start |
| `database.poolSize` | no | restart | |
| `kafka.brokers` | no | restart | the producer and consumer are created at startup |
| `auth.oidc.issuer` | no | restart | public key cache |
| `profile` | no | restart | affects the defaults of the whole configuration |
| `objectStorage.endpoint` | no | restart | |

!!! warning "Changes from the console have the highest priority"
    A value set in the console overrides the file and the environment variables. If a key still has its old value after you deploy a new `orion.yaml`, check the overrides in the console; `orion config validate` reports them with the source `console`.

## Section subpages

- [Environment variables](environment-variables.md)
- [Configuration profiles](profiles.md)

## See also

- [Requirements](../requirements.md)
- [Installation](../installation/index.md)
- [Security](../../operations/security.md)
- [Logs](../../operations/monitoring/logs.md)
