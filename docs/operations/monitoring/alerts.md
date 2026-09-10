# Alerts

Alerts in Orion Platform 4.2 LTS are derived from service level objectives, not from raw values of individual metrics. Every alert carries a severity of P1 to P3 and a runbook describing the response. This page contains the rule catalogue, the complete Prometheus rule file, Alertmanager routing and the silencing policy.

## Thresholding principles

- Alert on a symptom the customer feels (errors, latency, missed delivery), not on resource utilization.
- Derive thresholds from the error budget: alert at a burn rate that would exhaust the budget before the end of the month.
- Always set `for`; momentary spikes must not wake the on-call engineer. The minimum for P1 is 2 minutes.
- One symptom means one alert. Do not duplicate the same event across three component levels.
- An alert without a runbook must not be routed as P1 or P2.

!!! warning "Alerts without a runbook"
    A rule without a `runbook_url` annotation is rejected by CI validation and will not be deployed. The "Platform Core" weekly review removes rules that have not led to a single corrective action in 90 days.

## Alert catalogue

| Name | Expression (abbreviated) | Threshold | `for` | Severity | Runbook |
| --- | --- | --- | --- | --- | --- |
| `OrionApiDown` | `up{job="orion",component="gateway"}` | == 0 on all replicas | 2m | P1 | [runbooks](../runbooks/index.md) |
| `OrionHighErrorRate` | share of `status=~"5.."` | > 2% | 5m | P1 | [runbooks](../runbooks/index.md) |
| `OrionLatencyBudgetBurn` | `orion:http_get_latency:p95_5m` | > 0.25 s | 10m | P2 | [runbooks](../runbooks/index.md) |
| `OrionQueueLagHigh` | `orion_queue_lag{topic="orion.events.v1"}` | > 1000 | 10m | P2 | [queue backlog](../runbooks/queue-backlog.md) |
| `OrionDlqGrowing` | `rate(orion_events_dlq_total[15m])` | > 0 | 15m | P2 | [queue backlog](../runbooks/queue-backlog.md) |
| `OrionWebhookFailures` | `orion:webhooks:success_ratio_rate30m` | < 0.99 | 15m | P2 | [queue backlog](../runbooks/queue-backlog.md) |
| `OrionDbPoolSaturated` | `orion_db_pool_waiting` | > 5 | 5m | P1 | [database outage](../runbooks/database-outage.md) |
| `OrionDbReplicationLag` | `pg_replication_lag_seconds` | > 60 s | 5m | P2 | [database outage](../runbooks/database-outage.md) |
| `OrionCertExpiring` | `certmanager_certificate_expiration_timestamp_seconds` | < 14 days | 1h | P3 | [security](../security.md) |
| `OrionSchedulerStalled` | `time() - orion_scheduler_last_success_timestamp_seconds` | > 3600 s | 15m | P2 | [runbooks](../runbooks/index.md) |
| `OrionDiskFilling` | WAL volume usage forecast | < 4 h until full | 15m | P1 | [database outage](../runbooks/database-outage.md) |
| `OrionRateLimitSpike` | share of responses with `ORN-3001` | > 10% | 15m | P3 | [rate limits](../../api/rest/rate-limits.md) |

## Rule file

```yaml title="orion-rules.yaml"
groups:
  - name: orion.availability
    rules:
      - alert: OrionApiDown
        expr: |
          sum(up{job="orion",component="gateway"}) == 0
        for: 2m
        labels:
          severity: P1
          service: orion
          component: gateway
        annotations:
          summary: "No healthy orion-gateway replicas"
          description: >-
            Prometheus sees no healthy orion-gateway target.
            The API at https://api.orion.example.com/v1 is unavailable to all tenants.
          runbook_url: "https://docs.orion.example.com/operations/runbooks/"

      - alert: OrionHighErrorRate
        expr: |
          (
            sum(rate(orion_http_requests_total{component="gateway",status=~"5.."}[5m]))
            / sum(rate(orion_http_requests_total{component="gateway"}[5m]))
          ) > 0.02
        for: 5m
        labels:
          severity: P1
          service: orion
        annotations:
          summary: "5xx error rate above 2%"
          description: >-
            The share of 5xx responses is {{ $value | humanizePercentage }}.
            Check the ORN-5002 and ORN-5009 codes in the gateway logs.
          runbook_url: "https://docs.orion.example.com/operations/runbooks/"

  - name: orion.events
    rules:
      - alert: OrionQueueLagHigh
        expr: |
          max by (topic, group) (orion_queue_lag{topic="orion.events.v1"}) > 1000
        for: 10m
        labels:
          severity: P2
          service: orion
          component: worker
        annotations:
          summary: "Consumer lag for {{ $labels.group }} exceeds 1000 events"
          description: >-
            Current lag: {{ $value }} events in topic {{ $labels.topic }}.
            The SLO target is below 1000 events; webhooks may exceed 60 s.
          runbook_url: "https://docs.orion.example.com/operations/runbooks/queue-backlog/"

      - alert: OrionDlqGrowing
        expr: |
          sum(rate(orion_events_dlq_total{topic="orion.events.v1.dlq"}[15m])) > 0
        for: 15m
        labels:
          severity: P2
          service: orion
        annotations:
          summary: "Events are landing in the DLQ"
          runbook_url: "https://docs.orion.example.com/operations/runbooks/queue-backlog/"

  - name: orion.database
    rules:
      - alert: OrionDbPoolSaturated
        expr: |
          max by (component) (orion_db_pool_waiting) > 5
        for: 5m
        labels:
          severity: P1
          service: orion
        annotations:
          summary: "Connection pool for component {{ $labels.component }} is saturated"
          description: >-
            {{ $value }} requests are waiting for a free connection to the orion database.
            Expect 503 responses with the ORN-5009 code.
          runbook_url: "https://docs.orion.example.com/operations/runbooks/database-outage/"
```

## Alertmanager routing

```yaml title="alertmanager.yaml"
route:
  receiver: default-email
  group_by: [alertname, service, component]
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h
  routes:
    - matchers: ['severity="P1"', 'service="orion"']
      receiver: orion-pager
      group_wait: 10s
      repeat_interval: 30m
      continue: true
    - matchers: ['severity="P1"', 'service="orion"']
      receiver: orion-oncall-slack
    - matchers: ['severity="P2"', 'service="orion"']
      receiver: orion-oncall-slack
      repeat_interval: 2h
    - matchers: ['severity="P3"', 'service="orion"']
      receiver: default-email
      repeat_interval: 24h

receivers:
  - name: orion-pager
    pagerduty_configs:
      - routing_key_file: /etc/alertmanager/secrets/pagerduty_key
        severity: critical
        description: '{{ .CommonAnnotations.summary }}'
  - name: orion-oncall-slack
    slack_configs:
      - api_url_file: /etc/alertmanager/secrets/slack_url
        channel: '#orion-oncall'
        title: '[{{ .CommonLabels.severity }}] {{ .CommonLabels.alertname }}'
        text: >-
          {{ .CommonAnnotations.description }}
          Runbook: {{ .CommonAnnotations.runbook_url }}
  - name: default-email
    email_configs:
      - to: support@nimbus.example.com
        send_resolved: true

inhibit_rules:
  - source_matchers: ['alertname="OrionApiDown"']
    target_matchers: ['severity=~"P2|P3"', 'service="orion"']
    equal: [service]
```

!!! note "Top-level inhibition rule"
    `inhibit_rules` prevents a flood of notifications during a full outage: while `OrionApiDown` is active, P2 and P3 alerts for the `orion` service are suppressed. The on-call engineer sees one alert instead of twelve.

## Silencing and maintenance windows

Create a silence for a planned deployment before the work starts, with a named author and a justification:

```bash
amtool silence add \
  --alertmanager.url=https://alertmanager.internal.example.com \
  --author="marek.debski" \
  --comment="Maintenance window: helm upgrade nimbus/orion 4.2.1" \
  --duration=2h \
  service=orion severity=P2
```

```text
silence created: 7c1d5a9e-2f43-4c8e-9b21-0d5a7f6e3c18 (expires 2026-02-12T00:04:11Z)
```

Review active silences before handing over on-call duty:

```bash
amtool silence query --alertmanager.url=https://alertmanager.internal.example.com
```

!!! danger "Never silence P1 indefinitely"
    A silence with the `P1` severity is limited to 4 hours and requires the incident commander's approval. Open P1 silences are detected by the `silence-audit` job in `orion-scheduler` and reported on `#orion-oncall`.

## Testing rules

The rules live in the repository together with `promtool` unit tests.

```yaml title="orion-rules-test.yaml"
rule_files:
  - orion-rules.yaml

evaluation_interval: 1m

tests:
  - interval: 1m
    input_series:
      - series: 'orion_queue_lag{topic="orion.events.v1",group="orion-worker"}'
        values: '900+50x20'
    alert_rule_test:
      - eval_time: 15m
        alertname: OrionQueueLagHigh
        exp_alerts:
          - exp_labels:
              severity: P2
              service: orion
              component: worker
              topic: orion.events.v1
              group: orion-worker
```

```bash
promtool check rules orion-rules.yaml
promtool test rules orion-rules-test.yaml
```

```text
Unit Testing:  orion-rules-test.yaml
  SUCCESS
```

## See also

- [Metrics](metrics.md)
- [Runbooks](../runbooks/index.md)
- [Monitoring and observability](index.md)
- [Operations](../index.md)
