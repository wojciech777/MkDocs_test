# Monitoring and observability

Orion Platform 4.2 LTS exposes telemetry through three independent channels: Prometheus metrics, JSON logs on stdout and OpenTelemetry traces. All three channels share the `X-Orion-Request-Id` correlation identifier, so a single API request can be traced all the way to webhook delivery. This page describes the required tool stack and the data collection points.

## The three pillars

Metrics
:   Counters, gauges and histograms prefixed with `orion_` on the `/metrics` endpoints. The basis for alerts and SLO dashboards. Details: [metrics](metrics.md).

Logs
:   Structured JSON on standard output, one line per event, with the `request_id`, `trace_id` and `tenant` fields. Details: [logs](logs.md).

Traces
:   OpenTelemetry spans exported over OTLP on port 4317. Default sampling is set by `ORION_TRACE_SAMPLE_RATE` (`0.05` in production).

## Data collection points

| Component | Endpoint | Port | Format |
| --- | --- | --- | --- |
| `orion-gateway` | `/metrics` | 9100 | Prometheus text |
| `orion-gateway` | `/healthz`, `/readyz` | 8080 | JSON |
| `orion-core` | `/metrics` | 9100 | Prometheus text |
| `orion-ledger` | `/metrics` | 9100 | Prometheus text |
| `orion-worker` | `/metrics` | 9100 | Prometheus text |
| `orion-scheduler` | `/metrics` | 9100 | Prometheus text |
| `orion-console` | `/metrics` | 9100 | Prometheus text |
| all components | stdout | — | JSON (one line) |
| all components | OTLP gRPC | 4317 | OpenTelemetry |

## Required stack

- Prometheus 2.5x (2.51 minimum), scrape every 30 s, 15 days of local retention.
- Grafana 11, dashboards and error budget reviews.
- Loki 3.x or Elasticsearch 8.x, log aggregation, 30 days of retention.
- OpenTelemetry Collector 0.9x, OTLP intake, tail sampling, export to the trace backend.
- Alertmanager 0.27, notification routing, see [alerts](alerts.md).

!!! warning "Do not scrape through the gateway"
    The `/metrics` endpoint on port 9100 is not rate limited and not authenticated. It must be reachable only from the internal network, and the traffic rules are described in [security](../security.md).

## Bundled dashboards

The dashboards ship with the `nimbus/orion` Helm chart 4.2.1 as ConfigMaps labelled `grafana_dashboard: "1"`.

| Name | ID | What it shows |
| --- | --- | --- |
| Orion / API Overview | `orion-api` | Traffic, error rate, p95 and p99 per endpoint |
| Orion / SLO Burn | `orion-slo` | Error budget consumption in 1 h and 6 h windows |
| Orion / Orders | `orion-orders` | Order throughput, status distribution |
| Orion / Payments | `orion-pay` | `orion_payment_latency_seconds`, ledger errors |
| Orion / Events & Webhooks | `orion-events` | `orion_queue_lag`, DLQ, delivery success rate |
| Orion / Database | `orion-db` | Connection pools, query time, replication |
| Orion / Scheduler | `orion-sched` | Duration of recurring jobs, skipped runs |

## Subpages

- [Metrics](metrics.md), metric catalogue, PromQL, recording rules.
- [Logs](logs.md), format, correlation, masking, retention.
- [Alerts](alerts.md), rules, thresholds, routing, silences.

## See also

- [Operations](../index.md)
- [Runbooks](../runbooks/index.md)
- [Scaling and performance](../scaling.md)
