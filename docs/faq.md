# FAQ

The questions the Nimbus Software support team receives most often. The answers
apply to release 4.2 LTS. If you do not find a solution here, head to the
[Support](support.md) page.

## Installation and deployment

??? question "Which installation method should I use in production?"

    Kubernetes is the recommended choice: [Helm](getting-started/installation/kubernetes/helm.md)
    for deployments managed by a platform team, or the
    [operator](getting-started/installation/kubernetes/operator.md) if you want
    self-healing and automated migrations. Docker Compose is meant for
    development and demo environments.

??? question "Can I run Orion without Kafka?"

    No. Kafka is the event bus for the whole platform. Without it, webhooks,
    projections, and the audit trail do not work. See
    [Event processing](guides/events/index.md) for details.

??? question "What are the minimum resources for a development environment?"

    4 CPU cores, 8 GB of RAM, and 20 GB of disk. The full breakdown is in
    [Requirements](getting-started/requirements.md).

??? question "Is PostgreSQL 14 good enough?"

    No, release 4.2 requires PostgreSQL 15. The migrations use constructs that
    are unavailable in 14, and the RLS policies assume the behavior of 15.

## Configuration

??? question "An environment variable has no effect. Why?"

    The usual causes: a typo in the `ORION_` prefix, the variable set in only one
    container, or a value from the console overriding it. The precedence order of
    configuration sources is described in
    [Configuration](getting-started/configuration/index.md), and the full list of
    variables in
    [Environment variables](getting-started/configuration/environment-variables.md).

??? question "How do I check that the configuration is valid before restarting?"

    Run `orion config validate --file /etc/orion/orion.yaml`. The command returns
    a list of errors together with the key path and a validation code.

??? question "Can I use the `dev` profile in production?"

    No. The `dev` profile enables automatic migrations, drops the TLS
    requirement, and raises the log level to `DEBUG`. See
    [Configuration profiles](getting-started/configuration/profiles.md).

## API integration

??? question "OAuth2 or an API key?"

    For server-to-server integration inside your own infrastructure, an
    [API key](guides/authentication/api-keys.md) is enough. For external
    integrations, and wherever short-lived credentials are required, use
    [OAuth2 client credentials](guides/authentication/oauth2.md).

??? question "I get ORN-2003. What should I do?"

    The token is valid but lacks the required scope. Check the scopes assigned to
    the client and the permission matrix in
    [Roles and permissions](guides/authentication/roles-and-permissions.md).

??? question "How do I avoid creating an order twice when a request is retried?"

    Send an `Idempotency-Key` header with a unique value per business operation.
    The idempotency window is 24 hours. See
    [REST API basics](api/rest/index.md).

??? question "Should I poll the API every second or use webhooks?"

    Webhooks. Polling a collection in a tight loop burns through the
    [rate limits](api/rest/rate-limits.md) quickly. Endpoint registration is
    described in [Webhooks](api/webhooks/index.md).

??? question "Why do I receive the same event twice?"

    Webhook delivery has at-least-once semantics. Deduplicate on the event `id`
    (`evt_...`) and order by the `sequence` field. See
    [Event types](api/webhooks/event-types.md).

??? question "A cursor from a previous page stopped working (ORN-1012)"

    Cursors are opaque and bound to the query parameters. Changing the sort order
    or the filters invalidates the cursor, so start traversing the collection
    again. See [Pagination, filtering, and sorting](api/rest/pagination.md).

## Operations

??? question "How fast can the database be restored after an outage?"

    The target RTO is 60 minutes and the RPO is 5 minutes. The point-in-time
    recovery procedure is in [Backup and restore](operations/backups.md), and the
    steps to take during an outage are in the
    [PostgreSQL runbook](operations/runbooks/database-outage.md).

??? question "Queue lag is growing. Should I scale the workers?"

    Yes, but only up to the number of partitions. Additional replicas beyond that
    do not add throughput. See the
    [queue backlog runbook](operations/runbooks/queue-backlog.md) and
    [Scaling and performance](operations/scaling.md).

??? question "Which metrics should I watch while on call?"

    The 5xx error rate, p95 response time, `orion_queue_lag`,
    `orion_events_dlq_total`, and webhook delivery success. The alert set is
    described in [Alerts](operations/monitoring/alerts.md).

??? question "How do I correlate a customer report with the logs?"

    Ask for the `X-Orion-Request-Id` value or the `error.request_id` from the
    error response, then search for it in the logs. See
    [Logs](operations/monitoring/logs.md).

## Licensing and versions

??? question "How do the Community, Business, and Enterprise editions differ?"

    In tenant limits, availability of the Kubernetes operator, and the scope of
    support. The comparison table is in
    [What is Orion Platform](introduction/what-is-orion.md).

??? question "Can I jump straight from 3.8 to 4.2?"

    Yes, this is a supported path, but it requires the full procedure from the
    [migration guide](guides/migrations/from-3-8-to-4-2.md). The upgrade needs a
    maintenance window.

??? question "How long is an LTS release supported?"

    18 months from publication. For 4.2 LTS, support ends on 2027-06-30. The
    dates for the other releases are in the [changelog](changelog.md).

## See also

- [Support](support.md)
- [Changelog](changelog.md)
- [Error codes](api/rest/error-codes.md)
- [Glossary](introduction/glossary.md)
