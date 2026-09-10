# Metrics

All Orion Platform 4.2 LTS components expose metrics in Prometheus format under `/metrics` on port 9100. Names always start with the `orion_` prefix, and units are encoded in the suffix (`_seconds`, `_total`, `_bytes`). This page contains the full metric catalogue, the scrape configuration, ready-made PromQL queries and recording rules.

## Naming and label conventions

A metric name describes the measured object in the plural and ends with a unit or with the `_total` suffix for counters. Labels are restricted to closed sets of values.

| Label | Values | Notes |
| --- | --- | --- |
| `tenant` | tenant identifier | always present on business metrics |
| `component` | `gateway`, `core`, `ledger`, `worker`, `scheduler`, `console` | set by the process, not by relabeling |
| `endpoint` | route pattern, for example `/orders/{id}` | never a full URL containing an identifier |
| `status` | `2xx`, `4xx`, `5xx` or the exact code | depends on the metric |
| `method` | `GET`, `POST`, `PUT`, `PATCH`, `DELETE` | uppercase |

!!! warning "Cardinality"
    Do not add a label carrying an order identifier, a customer identifier or a `request_id`. An `order_id` label on `orion_orders_processed_total` generates millions of time series in a production environment and takes Prometheus down within hours. Identifiers belong in logs and traces, not in metrics.

## Metric catalogue

| Metric | Type | Labels | Description |
| --- | --- | --- | --- |
| `orion_http_requests_total` | counter | `component`, `method`, `endpoint`, `status`, `tenant` | Number of HTTP requests served |
| `orion_http_request_duration_seconds` | histogram | `component`, `method`, `endpoint` | HTTP request handling time |
| `orion_orders_processed_total` | counter | `tenant`, `status` | Orders processed by `orion-core`, broken down by target status |
| `orion_payment_latency_seconds` | histogram | `tenant`, `provider` | Payment settlement time in `orion-ledger` |
| `orion_queue_lag` | gauge | `topic`, `group` | Kafka consumer lag in events |
| `orion_worker_retries_total` | counter | `tenant`, `reason` | Event processing retries by `orion-worker` |
| `orion_events_dlq_total` | counter | `topic`, `reason` | Events parked in `orion.events.v1.dlq` |
| `orion_webhook_deliveries_total` | counter | `tenant`, `status`, `event_type` | Webhook delivery attempts |
| `orion_webhook_delivery_latency_seconds` | histogram | `tenant`, `event_type` | Time from event publication to delivery acknowledgement |
| `orion_db_pool_in_use` | gauge | `component`, `schema` | Active connections in the pool to the `orion` database |
| `orion_db_pool_waiting` | gauge | `component` | Requests waiting for a free connection |
| `orion_scheduler_job_duration_seconds` | histogram | `job`, `tenant` | Duration of a recurring job |
| `orion_build_info` | gauge | `version`, `commit`, `component` | Always `1`; carries version information |

??? info "Histogram buckets"
    `orion_http_request_duration_seconds` uses the buckets `0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10`. `orion_webhook_delivery_latency_seconds` uses the buckets `1, 5, 10, 30, 60, 120, 300`, aligned with the 60 s SLO target. Changing the buckets requires rebuilding the recording rules.

## Scrape configuration

=== "Kubernetes"

    ```yaml title="servicemonitor.yaml" hl_lines="10 11 14"
    apiVersion: monitoring.coreos.com/v1
    kind: ServiceMonitor
    metadata:
      name: orion
      namespace: orion
      labels:
        release: kube-prometheus-stack
    spec:
      selector:
        matchLabels:
          app.kubernetes.io/part-of: orion
      namespaceSelector:
        matchNames: [orion]
      endpoints:
        - port: metrics
          interval: 30s
          scrapeTimeout: 10s
          honorLabels: false
          relabelings:
            - sourceLabels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
              targetLabel: component
    ```

=== "System packages"

    ```yaml title="prometheus.yml" hl_lines="3 4 9"
    scrape_configs:
      - job_name: orion
        scrape_interval: 30s
        scrape_timeout: 10s
        metrics_path: /metrics
        static_configs:
          - targets:
              - orion-gateway-1.internal.example.com:9100
              - orion-gateway-2.internal.example.com:9100
              - orion-core-1.internal.example.com:9100
              - orion-ledger-1.internal.example.com:9100
              - orion-worker-1.internal.example.com:9100
              - orion-scheduler-1.internal.example.com:9100
        relabel_configs:
          - source_labels: [__address__]
            regex: 'orion-([a-z]+)-.*'
            target_label: component
            replacement: '$1'
    ```

After changing the configuration, validate it before reloading:

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -s -X POST http://localhost:9090/-/reload
```

## PromQL queries

5xx error rate on the API:

```promql
sum(rate(orion_http_requests_total{component="gateway",status=~"5.."}[5m]))
  / sum(rate(orion_http_requests_total{component="gateway"}[5m]))
```

p95 response time for `GET` methods (target < 250 ms):

```promql
histogram_quantile(0.95,
  sum by (le) (rate(orion_http_request_duration_seconds_bucket{component="gateway",method="GET"}[5m])))
```

p95 for `POST /orders` (target < 600 ms):

```promql
histogram_quantile(0.95,
  sum by (le) (rate(orion_http_request_duration_seconds_bucket{endpoint="/orders",method="POST"}[5m])))
```

Order throughput per minute, broken down by tenant:

```promql
sum by (tenant) (rate(orion_orders_processed_total{status="paid"}[5m])) * 60
```

Largest event queue lag:

```promql
max by (topic, group) (orion_queue_lag{topic="orion.events.v1"})
```

Database connection pool utilization:

```promql
sum by (component) (orion_db_pool_in_use)
  / sum by (component) (orion_db_pool_in_use + 1e-9 + orion_db_pool_waiting)
```

Webhook delivery success rate over a 24 hour window:

```promql
sum(rate(orion_webhook_deliveries_total{status="success"}[24h]))
  / sum(rate(orion_webhook_deliveries_total[24h]))
```

Share of webhooks delivered within the required 60 s:

```promql
sum(rate(orion_webhook_delivery_latency_seconds_bucket{le="60"}[24h]))
  / sum(rate(orion_webhook_delivery_latency_seconds_count[24h]))
```

Burn rate of the 99.9% availability error budget over an hourly window:

```promql
(1 - orion:api_availability:ratio_rate1h) / (1 - 0.999)
```

Versions running in the cluster (post-deployment verification):

```promql
count by (version, component) (orion_build_info)
```

## Recording rules

Recording rules shorten dashboard queries and stabilize alert thresholds.

```yaml title="orion-recording-rules.yaml"
groups:
  - name: orion.slo
    interval: 30s
    rules:
      - record: orion:api_availability:ratio_rate1h
        expr: |
          sum(rate(orion_http_requests_total{component="gateway",status!~"5.."}[1h]))
            / sum(rate(orion_http_requests_total{component="gateway"}[1h]))
      - record: orion:api_availability:ratio_rate6h
        expr: |
          sum(rate(orion_http_requests_total{component="gateway",status!~"5.."}[6h]))
            / sum(rate(orion_http_requests_total{component="gateway"}[6h]))
      - record: orion:http_get_latency:p95_5m
        expr: |
          histogram_quantile(0.95,
            sum by (le) (rate(orion_http_request_duration_seconds_bucket{component="gateway",method="GET"}[5m])))
      - record: orion:orders:rate5m
        expr: sum by (tenant) (rate(orion_orders_processed_total[5m]))
      - record: orion:webhooks:success_ratio_rate30m
        expr: |
          sum(rate(orion_webhook_deliveries_total{status="success"}[30m]))
            / sum(rate(orion_webhook_deliveries_total[30m]))
```

## Retention and cardinality

| Tier | Resolution | Retention |
| --- | --- | --- |
| Local Prometheus | 30 s | 15 days |
| Long-term storage (remote write) | 30 s to 5 min after 7 days | 400 days |
| SLO recording rules | 30 s | 400 days |

The cardinality budget for a single production installation is 1.5 million active series. Once 80% of the budget is exceeded, the "Platform Core" team removes the highest-cardinality labels. Check the current state with this query:

```promql
topk(10, count by (__name__) ({__name__=~"orion_.*"}))
```

!!! danger "Identifier labels"
    Adding an `order_id`, `payment_id` or `request_id` label to any `orion_*` metric is forbidden in the `staging` and `prod` profiles. Such a change does not pass code review and is blocked by `orion config validate`.

## See also

- [Monitoring and observability](index.md)
- [Alerts](alerts.md)
- [Logs](logs.md)
- [Scaling and performance](../scaling.md)
