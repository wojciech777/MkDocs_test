# Runbook: event queue backlog

This runbook covers a growing consumer lag for the `orion-worker` group on the `orion.events.v1` topic and its consequences: delayed webhooks and a rising number of events in `orion.events.v1.dlq`. It applies to the `OrionQueueLagHigh`, `OrionDlqGrowing` and `OrionWebhookFailures` alerts. A backlog does not cause data loss, but it breaches the 60 s webhook delivery objective.

## Symptoms

- `orion_queue_lag{topic="orion.events.v1"}` exceeds 1000 and rises monotonically across consecutive scrape intervals.
- Webhooks delivered with more than 60 s of delay; `orion_webhook_delivery_latency_seconds` shifts into the highest buckets.
- `orion_events_dlq_total` rises, and the `orion-worker` logs repeat `msg="event moved to dlq"` entries.
- `orion_worker_retries_total` grows faster than `orion_webhook_deliveries_total`.
- Tenants report missing notifications about order status changes from `paid` to `fulfilled`.
- With a poison event: the offset of one partition stands still while the others process normally.

## Impact and severity

| Scope | Severity | Rationale |
| --- | --- | --- |
| Lag above 1000 and rising, webhooks delayed | P2 | Delivery SLO breach, API working |
| Lag rising for all tenants, delay above 30 min | P1 | Effect close to the feature being unavailable |
| Events landing in the DLQ without a retry | P2 | Risk of permanently missed notifications |
| Lag limited to one tenant with a slow endpoint | P3 | Single integration broken |
| One partition stalled (poison event) | P2 | Event ordering blocked for a subset of keys |

Reads and writes through the API remain healthy, so do not switch the gateway to read-only mode.

## Quick diagnosis

State of the consumer group:

```bash
kafka-consumer-groups.sh \
  --bootstrap-server kafka-0.orion-data:9093 \
  --command-config /etc/kafka/client.properties \
  --describe --group orion-worker
```

```text
GROUP         TOPIC              PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG    CONSUMER-ID
orion-worker  orion.events.v1    0          4821904         4822011         107    orion-worker-3-a1f2
orion-worker  orion.events.v1    1          4818377         4819002         625    orion-worker-1-77bd
orion-worker  orion.events.v1    2          4790112         4841566        51454   orion-worker-5-9c04
orion-worker  orion.events.v1    3          4820551         4820702         151    orion-worker-2-3ee8
orion-worker  orion.events.v1    4          4819883         4820014         131    orion-worker-6-c5a1
orion-worker  orion.events.v1    5          4817440         4817612         172    orion-worker-4-b820
```

A distribution like the one above, one partition with enormous lag while the rest are healthy, points to a poison event or a very slow receiver for the keys in that partition. Lag spread evenly across all partitions points to a throughput shortfall.

Logs of the worker handling the stalled partition:

```bash
kubectl logs -n orion -l app.kubernetes.io/name=orion-worker --tail=100 \
  | jq -c 'select(.level=="warn" or .level=="error") | {ts,msg,err_code,tenant,order_id}'
```

```text
{"ts":"2026-02-11T10:02:14.118Z","msg":"webhook delivery failed","err_code":"ORN-5009","tenant":"acme-retail","order_id":"ord_01HQ8ZK4"}
{"ts":"2026-02-11T10:02:44.221Z","msg":"webhook retry scheduled","err_code":"ORN-5009","tenant":"acme-retail","order_id":"ord_01HQ8ZK4"}
{"ts":"2026-02-11T10:03:44.402Z","msg":"event processing failed","err_code":"ORN-5009","tenant":"acme-retail","order_id":"ord_01HQ8ZK4"}
```

The same `order_id` appearing repeatedly with a rising retry count is the signature of a poison event.

Worker resource usage, which distinguishes a capacity shortfall from a blockage:

```bash
kubectl top pods -n orion -l app.kubernetes.io/name=orion-worker
```

```text
NAME                            CPU(cores)   MEMORY(bytes)
orion-worker-6d4c8f7b9d-2kq8x   47m          312Mi
orion-worker-6d4c8f7b9d-5rt2v   51m          298Mi
orion-worker-6d4c8f7b9d-9c04m   38m          341Mi
```

Low CPU with a large lag means the workers are waiting for an external response rather than running out of capacity.

Delivery success rate per tenant:

```promql
sum by (tenant) (rate(orion_webhook_deliveries_total{status="failed"}[15m]))
```

Contents of the dead letter queue:

```bash
kafka-console-consumer.sh \
  --bootstrap-server kafka-0.orion-data:9093 \
  --consumer.config /etc/kafka/client.properties \
  --topic orion.events.v1.dlq --from-beginning --max-messages 5 \
  --property print.headers=true
```

## Procedure

### Step 1: establish the type of backlog

1. Compare the lag distribution across partitions with the worker CPU usage.

    Checkpoint: you have an unambiguous classification, variant A (throughput shortfall), B (slow receiver) or C (poison event).

### Variant A: throughput shortfall

1. Increase the worker replica count without exceeding the topic's partition count.

    ```bash
    kubectl get kafkatopic orion-events-v1 -n orion-data -o jsonpath='{.spec.partitions}{"\n"}'
    kubectl scale -n orion deploy/orion-worker --replicas=12
    kubectl rollout status -n orion deploy/orion-worker --timeout=180s
    ```

    Checkpoint: the replica count is less than or equal to the partition count, and `kafka-consumer-groups.sh --describe` shows every consumer assigned to a partition (no consumers without an assignment).

    !!! warning "Replicas above the partition count"
        Consumers beyond the partition count stay idle while still holding database connections, and they can trigger the `OrionDbPoolSaturated` alert. If 12 replicas are not enough, first increase the partition count in a maintenance window, see [scaling](../scaling.md).

2. Check whether the backlog is secondary to a database problem (`orion_db_pool_waiting` above zero).

    Checkpoint: `orion_db_pool_waiting` is at zero. Otherwise, move to the [database outage runbook](database-outage.md).

3. Watch the lag trend for 10 minutes.

    Checkpoint: `orion_queue_lag` drops by at least 10% in each successive 5-minute interval.

### Variant B: unavailable or slow customer endpoint

1. Identify the tenant and the endpoint responsible for the blockage.

    ```promql
    topk(5, sum by (tenant) (rate(orion_webhook_deliveries_total{status="failed"}[15m])))
    ```

    Checkpoint: one or two tenants account for the majority of the failures.

2. Limit concurrency and extend the backoff for that tenant so it does not block the others.

    ```bash
    kubectl exec -n orion deploy/orion-worker -- \
      orion config set webhooks.tenant.acme-retail.max_inflight 2 --ttl 2h
    ```

    Checkpoint: the lag for the remaining tenants decreases and `orion_webhook_delivery_latency_seconds` for the other `tenant` values returns below 60 s.

3. Temporarily disable non-essential subscriptions (reports, informational notifications) while keeping the order and payment status events.

    ```bash
    kubectl exec -n orion deploy/orion-worker -- \
      orion config set webhooks.disabled_types "report.ready,catalog.synced" --ttl 4h
    ```

    Checkpoint: `orion_queue_lag` decreases; the list of event types is in [event types](../../api/webhooks/event-types.md).

4. Contact the tenant through the support portal. No events are lost: they stay in the topic until retention expires and will be delivered once the endpoint is back.

    Checkpoint: a ticket has been created and the tenant's fix time has been agreed.

### Variant C: poison event

1. Identify the offset and key of the blocking event.

    ```bash
    kafka-console-consumer.sh \
      --bootstrap-server kafka-0.orion-data:9093 \
      --consumer.config /etc/kafka/client.properties \
      --topic orion.events.v1 --partition 2 --offset 4790112 --max-messages 1 \
      --property print.key=true --property print.headers=true
    ```

    ```text
    request_id:5f2c1b64-...,tenant:acme-retail,event_type:order.updated
    ord_01HQ8ZK4  {"id":"evt_01HQ9...","type":"order.updated","order_id":"ord_01HQ8ZK4","payload":{...}}
    ```

    Checkpoint: you know the partition, the offset, the `tenant` and the `order_id`.

2. Move the event to the DLQ to unblock the partition.

    ```bash
    kubectl exec -n orion deploy/orion-worker -- \
      orion events replay --to-dlq \
        --topic orion.events.v1 --partition 2 --offset 4790112 --reason "poison: schema mismatch"
    ```

    ```text
    event evt_01HQ9F8K2 moved to orion.events.v1.dlq
    consumer group orion-worker: offset advanced to 4790113 on partition 2
    ```

    Checkpoint: `CURRENT-OFFSET` for partition 2 starts rising and the lag begins to fall.

3. Establish why the event was rejected. The most common causes: a schema mismatch after a migration, a missing tenant, an invalid order status.

    Checkpoint: the cause is recorded in the timeline, and a bug is filed if the application is at fault.

4. After the fix, replay the events from the DLQ, starting with a dry run.

    ```bash
    orion events replay --topic orion.events.v1.dlq \
      --from '2026-02-11T09:45:00Z' --to '2026-02-11T10:30:00Z' --dry-run
    ```

    ```text
    matched events        184
    idempotency conflicts 0
    would republish to    orion.events.v1
    ```

    Then run it for real without `--dry-run`.

    Checkpoint: `orion_events_dlq_total` stops rising and every replayed event has a confirmed delivery. Retry policy: [retries](../../guides/events/retries.md).

### Final step: return to normal

1. Roll back the temporary limits once the lag has stabilized.

    ```bash
    kubectl exec -n orion deploy/orion-worker -- orion config unset webhooks.disabled_types
    kubectl exec -n orion deploy/orion-worker -- orion config unset webhooks.tenant.acme-retail.max_inflight
    kubectl scale -n orion deploy/orion-worker --replicas=6
    ```

    Checkpoint: `orion_queue_lag` below 100 for 15 minutes at the nominal replica count.

2. Confirm delivery of the outstanding webhooks.

    ```promql
    sum(rate(orion_webhook_delivery_latency_seconds_bucket{le="60"}[30m]))
      / sum(rate(orion_webhook_delivery_latency_seconds_count[30m]))
    ```

    Checkpoint: the value is above 0.99.

## Escalation

| Condition | Recipient | Deadline |
| --- | --- | --- |
| Lag does not decrease after 30 minutes despite scaling | The incident commander raises it to P1 | immediately |
| Lag exceeds 100,000 events | The "Platform Core" team (Marek Debski) | within 15 minutes |
| The backlog approaches the topic retention (72 h) | P1, risk of permanent event loss | immediately |
| The poison event stems from a 4.2.1 application bug | Ticket at `https://support.nimbus.example.com` with an `orion diag bundle` | within 4 h |
| The tenant does not restore its endpoint for more than 24 h | Product owner (Anna Kowalska) | within 24 h |

## After the incident

- [ ] Confirm that `orion.events.v1.dlq` is empty or contains only events that were deliberately rejected and documented.
- [ ] Check that every temporary `orion config set` entry has expired or been removed.
- [ ] Verify the worker replica count against the nominal value for the given deployment size.
- [ ] Agree with the affected tenants whether events need to be redelivered.
- [ ] Check the consistency of orders whose status changed during the backlog.
- [ ] Complete the timeline and update `https://status.orion.example.com`.
- [ ] Schedule the postmortem within 5 business days.
- [ ] Assess whether the `OrionQueueLagHigh` and `OrionDlqGrowing` thresholds fired with enough lead time.

## Prevention

- Keep the `OrionQueueLagHigh` alert with a threshold of 1000 and a `for: 10m` window, complemented by an alert on the lag growth rate.
- Set the worker HPA `maxReplicas` equal to the topic's partition count; as you approach the ceiling, add partitions in a maintenance window.
- Limit webhook delivery retries to 8 attempts with exponential backoff and a hard 24 h cap, after which the event goes to the DLQ.
- Apply a per-tenant concurrency limit so that one slow receiver cannot block the whole group.
- Enforce event schema validation on publication rather than on consumption; poison events should never reach the topic.
- Run quarterly throughput tests following the scenario in [scaling](../scaling.md).
- Review the DLQ contents at the weekly team meeting; a non-empty queue is a task, not background noise.

## See also

- [Runbooks](index.md)
- [Event queues](../../guides/events/queues.md)
- [Scaling and performance](../scaling.md)
- [Alerts](../monitoring/alerts.md)
