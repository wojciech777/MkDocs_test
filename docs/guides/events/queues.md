# Queues and topics

Orion Platform 4.2 LTS uses Apache Kafka 3.6 as its event bus. An installation creates four topics: the domain stream, the dead letter queue, the webhook delivery queue, and the audit stream. The topics are created by `orion-scheduler` on first start, provided the technical role holds the `CREATE` permission on the cluster.

## Topics

| Topic | Partitions | Retention | Producer | Consumer |
| --- | --- | --- | --- | --- |
| `orion.events.v1` | 24 | 7 days | `orion-core`, `orion-ledger` | `orion-worker` |
| `orion.events.v1.dlq` | 24 | 30 days | `orion-worker` | operator (`orion events replay`) |
| `orion.webhooks.v1` | 12 | 3 days | `orion-worker` | `orion-worker` (delivery pool) |
| `orion.audit.v1` | 6 | 90 days | all components | `orion-worker` (writes to the `audit` schema) |

!!! note "Partition count and installation size"
    24 partitions handle up to roughly 3000 events per second with a typical payload distribution.
    For smaller installations (the `dev` and `staging` profiles) 6 partitions are enough, which reduces the number
    of open files on the broker.

## Consumer configuration

```yaml title="orion.yaml" hl_lines="6 7 11 14"
events:
  bootstrapServers: kafka-0.orion.svc:9092,kafka-1.orion.svc:9092
  topic: orion.events.v1
  dlqTopic: orion.events.v1.dlq
  consumer:
    groupId: orion-worker-prod
    maxPollRecords: 200
    maxPollIntervalMs: 300000
    sessionTimeoutMs: 45000
    autoOffsetReset: earliest
    enableAutoCommit: false
    isolationLevel: read_committed
  processing:
    concurrency: 8
    commitBatchSize: 50
```

The equivalent environment variables use the `ORION_` prefix, for example `ORION_EVENTS_CONSUMER_GROUP_ID`. The full list is documented in [environment variables](../../getting-started/configuration/environment-variables.md).

!!! warning "enableAutoCommit must stay disabled"
    `orion-worker` commits offsets after the projection has been written to PostgreSQL. Automatic committing
    leads to lost events when a pod restarts, because the offset advances before the write.

## Consumer groups and scaling

All `orion-worker` replicas belong to one group (`groupId`), so partitions are distributed among them.

| `orion-worker` replicas | Partitions per replica | Assessment |
| --- | --- | --- |
| 4 | 6 | starting configuration |
| 8 | 3 | recommended for production |
| 12 | 2 | used during sales peaks |
| 24 | 1 | effective maximum |
| 32 | 0.75 | 8 replicas idle, wasted resources |

The rule: the number of replicas should not exceed the number of partitions. Adding replicas beyond that count does not increase throughput, because Kafka will not assign a partition to two consumers in the same group.

```bash title="Inspecting the partition assignment"
kafka-consumer-groups.sh --bootstrap-server kafka-0.orion.svc:9092 \
  --group orion-worker-prod --describe
```

## Monitoring lag

The key metric is `orion_queue_lag`, the number of uncommitted messages per partition, exposed by `orion-worker` on port 9100.

| Threshold | Assessment | Response |
| --- | --- | --- |
| `< 1 000` | normal | none |
| `1 000 - 10 000` | warning (5 min) | check webhook response latency |
| `> 10 000` for 10 min | critical alert | increase the replica count, follow the runbook |
| growing linearly for 30 min | overload | throttle the producer, scale out |

```promql title="Alert rule"
sum by (topic) (orion_queue_lag{topic="orion.events.v1"}) > 10000
```

The procedure for clearing a backlog is described in the [queue backlog runbook](../../operations/runbooks/queue-backlog.md).

## Handling the DLQ

An event lands in `orion.events.v1.dlq` once retries are exhausted. The envelope is preserved unchanged, and the cause is added to the message headers: `x-orion-error-code`, `x-orion-error-message`, `x-orion-attempts`, `x-orion-original-partition`.

```bash title="Inspecting the DLQ contents"
kafka-console-consumer.sh --bootstrap-server kafka-0.orion.svc:9092 \
  --topic orion.events.v1.dlq --from-beginning --max-messages 20 \
  --property print.headers=true
```

Once the cause has been removed, replay the events:

```bash title="Replaying events from the DLQ"
orion events replay --topic orion.events.v1.dlq \
  --since 2026-02-11T00:00:00Z \
  --until 2026-02-11T06:00:00Z \
  --tenant ten_01HQ8ZV3KXN4M2T9YB7C6D \
  --dry-run

orion events replay --topic orion.events.v1.dlq \
  --since 2026-02-11T00:00:00Z \
  --until 2026-02-11T06:00:00Z \
  --tenant ten_01HQ8ZV3KXN4M2T9YB7C6D
```

`--dry-run` prints the count and types of events without publishing them. Replayed events return to `orion.events.v1` with their original `id`, so deduplication on the consumer side still works.

??? info "How you know the DLQ is growing"
    The `orion_events_dlq_total` counter increases with every move. Recommended alert:
    an increase of more than 10 messages in 15 minutes for any tenant.

## Compaction and retention

| Topic | `cleanup.policy` | Rationale |
| --- | --- | --- |
| `orion.events.v1` | `delete` | a stream of events, not state; 7 day retention |
| `orion.events.v1.dlq` | `delete` | 30 days to diagnose and replay |
| `orion.webhooks.v1` | `delete` | deliveries are short-lived |
| `orion.audit.v1` | `delete` | 90 days in Kafka, ultimately the `audit` schema (24 months) |

Compaction (`compact`) is not used on any topic. Events do not represent the current state of a key, so removing older entries for the same order would make it impossible to reconstruct the history.

!!! danger "Changing the partition count is irreversible"
    Increasing the partition count of the `orion.events.v1` topic changes how keys map to partitions.
    Events for one order published before and after the change end up on different partitions,
    which ==breaks the ordering guarantee== for orders in flight. Perform this operation only
    during a maintenance window, after stopping the producers and driving `orion_queue_lag` to zero.
    Decreasing the partition count is not possible in Kafka; it requires creating a new topic.

## See also

- [Event processing](index.md)
- [Retries and idempotency](retries.md)
- [Metrics](../../operations/monitoring/metrics.md)
- [Queue backlog](../../operations/runbooks/queue-backlog.md)
