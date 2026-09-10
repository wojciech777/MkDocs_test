# Scaling and performance

Orion Platform 4.2 LTS scales horizontally in the stateless layer (`orion-gateway`, `orion-core`, `orion-ledger`, `orion-worker`) and vertically in the stateful layer (PostgreSQL 15, Redis 7, Kafka 3.6). Overall system throughput is usually limited by the number of partitions on the `orion.events.v1` topic and by the database connection limit. This page gives the recommended deployment sizes, the autoscaling configuration and how to recognize bottlenecks.

## Scaling dimensions

Stateless components
:   `orion-gateway`, `orion-core`, `orion-ledger` and `orion-console` scale linearly with the replica count, up to the boundary set by the database connection pool.

Kafka partitions
:   `orion-worker` in the `orion-worker` group cannot have more active consumers than the topic has partitions. Additional replicas stay idle.

Database connection pools
:   Every replica maintains its own pool. The sum of all replica pools must fit within the PostgreSQL `max_connections`, leaving headroom for administrative work.

Redis
:   Scales through cluster mode for rate limits and the cache; a single instance handles up to roughly 2000 orders per minute.

`orion-scheduler`
:   Always runs as a single replica with leader election. Increasing the replica count does not increase recurring job throughput.

## Recommended deployment sizes

| Parameter | up to 50 orders/min | up to 500 orders/min | up to 5000 orders/min |
| --- | --- | --- | --- |
| `orion-gateway` (replicas) | 2 | 4 | 12 |
| `orion-core` | 2 | 4 | 14 |
| `orion-ledger` | 1 | 3 | 8 |
| `orion-worker` | 2 | 6 | 24 |
| `orion-scheduler` | 1 | 1 | 1 |
| `orion-console` | 1 | 2 | 3 |
| CPU / RAM per gateway replica | 0.5 vCPU / 512 MiB | 1 vCPU / 1 GiB | 2 vCPU / 2 GiB |
| CPU / RAM per core replica | 1 vCPU / 1 GiB | 2 vCPU / 2 GiB | 4 vCPU / 4 GiB |
| CPU / RAM per worker replica | 0.5 vCPU / 512 MiB | 1 vCPU / 1 GiB | 2 vCPU / 3 GiB |
| `orion.events.v1` partitions | 6 | 12 | 24 |
| `orion.webhooks.v1` partitions | 3 | 6 | 12 |
| PostgreSQL (vCPU / RAM) | 4 / 16 GiB | 8 / 32 GiB | 32 / 128 GiB |
| `max_connections` | 200 | 400 | 1200 |
| Database size after 12 months | up to 40 GiB | up to 400 GiB | up to 4 TiB |
| Redis | 1 instance, 2 GiB | 1 instance, 8 GiB | 6-node cluster, 8 GiB |

!!! tip "Increase partitions ahead of time"
    In Kafka the partition count can only be increased, and the change breaks event ordering within a key during the rebalance. Plan the target partition count based on the peak over the last 12 months and apply changes in a maintenance window.

## Autoscaling on Kubernetes

The gateway scales on CPU usage, and the worker on queue lag.

```yaml title="hpa-orion-worker.yaml" hl_lines="9 18 19 20"
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: orion-worker
  namespace: orion
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: orion-worker
  minReplicas: 4
  maxReplicas: 24
  metrics:
    - type: Pods
      pods:
        metric:
          name: orion_queue_lag
        target:
          type: AverageValue
          averageValue: "250"
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 600
      policies:
        - type: Pods
          value: 2
          periodSeconds: 120
```

!!! warning "maxReplicas and partitions"
    Set `maxReplicas` equal to the number of partitions on the `orion.events.v1` topic. A higher value creates idle pods that still hold database connections and can trigger the `OrionDbPoolSaturated` alert.

The `orion_queue_lag` value is exposed by the Prometheus adapter. To verify that the metric is available:

```bash
kubectl get --raw \
  "/apis/custom.metrics.k8s.io/v1beta1/namespaces/orion/pods/*/orion_queue_lag" | jq '.items[0]'
```

## Connection pools

The following rule applies:

```text
sum_replicas(component) × pool_size(component) ≤ max_connections − reserve
```

The reserve is 25 connections for administrative operations, replication and backup jobs.

Example for a deployment sized for 500 orders per minute (`max_connections = 400`):

| Component | Replicas | `pool_size` | Total |
| --- | --- | --- | --- |
| `orion-core` | 4 | 25 | 100 |
| `orion-ledger` | 3 | 20 | 60 |
| `orion-worker` | 6 | 10 | 60 |
| `orion-gateway` | 4 | 8 | 32 |
| `orion-scheduler` | 1 | 10 | 10 |
| **Total** | — | — | **262** |

The headroom to the limit is `400 − 25 − 262 = 113` connections, which allows the worker to scale up to 17 replicas without changing the database configuration.

Monitor both pool metrics:

```promql
max by (component) (orion_db_pool_in_use)
max by (component) (orion_db_pool_waiting)
```

`orion_db_pool_waiting` staying above zero means the pool is saturated, see the [database outage runbook](runbooks/database-outage.md). Once the sum of the pools exceeds 60% of `max_connections`, introduce pgBouncer in `transaction` mode.

## Bottlenecks

| Symptom | Likely cause | Action |
| --- | --- | --- |
| p95 for `GET` rises while gateway CPU is < 40% | Saturated connection pool or slow queries | Increase `pool_size` or add pgBouncer |
| p95 for `POST /orders` > 600 ms while `orion_payment_latency_seconds` is stable | Lock contention on `core.orders` | Shorten transactions, check the index on `tenant, status` |
| `orion_queue_lag` rises while worker CPU is < 50% | Slow customer webhook endpoint | Limit concurrency per tenant, see [queue backlog](runbooks/queue-backlog.md) |
| `orion_queue_lag` rises while worker CPU is > 85% | Too few replicas or partitions | Scale workers up to the partition count, then add partitions |
| `orion_worker_retries_total` rises without DLQ growth | Transient dependency errors (`ORN-5002`) | Check the availability of Redis and `orion-ledger` |
| Increase in `ORN-3001` responses | Client exceeding the limit, or a Redis restart reset the counters | Verify the [rate limits](../api/rest/rate-limits.md) |
| `orion_scheduler_job_duration_seconds` grows linearly | Growing tables without partitioning | Enable monthly partitioning of `audit.events` |
| Database CPU > 80% under steady traffic | Missing `ANALYZE`, stale plans | Run `VACUUM ANALYZE`, check `pg_stat_statements` |
| Growing Redis memory usage | Missing TTL on cache keys | Verify against the TTL table below |

## Load testing

Run load tests in a `staging` environment with the same partition count as production. The scenario below reproduces typical traffic: 70% reads, 25% order creation, 5% payments.

```javascript title="k6/orders.js"
import http from 'k6/http';
import { check, group } from 'k6';

export const options = {
  scenarios: {
    steady: { executor: 'constant-arrival-rate', rate: 500, timeUnit: '1m',
              duration: '20m', preAllocatedVUs: 50, maxVUs: 200 },
  },
  thresholds: {
    'http_req_duration{name:get_order}': ['p(95)<250'],
    'http_req_duration{name:create_order}': ['p(95)<600'],
    'http_req_failed': ['rate<0.005'],
  },
};

const base = 'https://api.orion.example.com/v1';
const params = { headers: { 'X-Orion-Key': __ENV.ORION_KEY,
                            'Content-Type': 'application/json' } };

export default function () {
  group('orders', () => {
    const res = http.post(`${base}/orders`,
      JSON.stringify({ customer_id: 'cus_demo', currency: 'PLN',
                       items: [{ sku: 'SKU-1', qty: 1, price: 4900 }] }),
      Object.assign({ tags: { name: 'create_order' } }, params));
    check(res, { 'created': (r) => r.status === 201 });
    if (res.status === 201) {
      const id = res.json('id');
      const get = http.get(`${base}/orders/${id}`,
        Object.assign({ tags: { name: 'get_order' } }, params));
      check(get, { 'read': (r) => r.status === 200 });
    }
  });
}
```

A run counts as healthy when all of the following hold at the same time:

- p95 for `GET` < 250 ms and p95 for `POST /orders` < 600 ms throughout the test,
- the share of 5xx responses stays below 0.5%, with no `ORN-5009` entries,
- `orion_queue_lag` never exceeds 1000 and returns to zero within 5 minutes of the test finishing,
- `orion_db_pool_waiting` remains at zero,
- no events land in `orion.events.v1.dlq`.

## Caching in Redis

| Key | Contents | TTL | Effect of loss |
| --- | --- | --- | --- |
| `orion:tenant:<id>:config` | Tenant configuration | 300 s | An extra query to `core` |
| `orion:catalog:<sku>` | Item price and availability | 60 s | Higher p95 for `POST /orders` |
| `orion:auth:jwks` | Keycloak public keys | 3600 s | A lookup against the `orion` realm |
| `orion:ratelimit:<key>` | Sliding window counter | 60 s | Reset of the `ORN-3001` limits |
| `orion:idem:<key>` | Request idempotency keys | 86400 s | Risk of a duplicate order |
| `orion:webhook:backoff:<tenant>` | Delivery backoff state | 900 s | Immediate retries |

!!! danger "Idempotency keys"
    `orion:idem:*` is the only category whose loss has a business impact: a retried `POST /orders` with the same key may create a second order. In the `prod` profile, enable AOF persistence on the Redis instance that serves idempotency.

## Degraded mode

When load exceeds capacity, the platform degrades in a defined order instead of refusing all traffic.

1. Queueing in the gateway: requests wait up to 2 s for a free connection; exceeding that returns the `ORN-5009` code.
2. Rate limiting: clients above the limit receive `429` with the `ORN-3001` code and a `Retry-After` header.
3. Shedding non-critical features: reports in `orion-console` and CSV exports are suspended.
4. Event prioritization: `orion.webhooks.v1` yields to `orion.events.v1`; webhook deliveries are delayed, not dropped.
5. Database protection: at `orion_db_pool_waiting > 20` the gateway switches to read-only mode and returns `503` with `ORN-5002` on write operations.

You set the degraded mode thresholds in `/etc/orion/orion.yaml`:

```yaml title="/etc/orion/orion.yaml"
gateway:
  overload:
    queue_wait_timeout: 2s
    read_only_pool_waiting: 20
    shed_non_critical: true
  rate_limit:
    default_rpm: 6000
    burst: 200
```

## See also

- [Metrics](monitoring/metrics.md)
- [Runbook: event queue backlog](runbooks/queue-backlog.md)
- [Event queues](../guides/events/queues.md)
- [API rate limits](../api/rest/rate-limits.md)
