# Retries and idempotency

Event delivery in Orion Platform has at-least-once semantics, so every consumer must cope with duplicates and with temporary unavailability of its dependencies. `orion-worker` retries processing according to a fixed policy of exponential backoff with jitter, and once the attempts are exhausted it moves the message to `orion.events.v1.dlq`. This page describes the policy, the classification of errors, and the idempotency mechanisms available on the consumer side.

## Retry policy

| Attempt | Delay before the attempt | Time since the first attempt |
| --- | --- | --- |
| 1 | 0 s | 0 s |
| 2 | 1 s | 1 s |
| 3 | 2 s | 3 s |
| 4 | 4 s | 7 s |
| 5 | 8 s | 15 s |
| 6 | 16 s | 31 s |
| 7 | 32 s | 1 min 3 s |
| 8 | 64 s | 2 min 7 s |
| - | move to the DLQ | approx. 2 min 10 s |

## Exponential backoff with jitter

The delay of the nth attempt is given by the full jitter formula:

```text
delay(n) = random(0, min(base * 2^(n-1), max_delay))
base = 1000 ms, max_delay = 64000 ms
```

Example for attempt 5: `min(1000 * 2^4, 64000) = 16000 ms`, so the actual delay is drawn from the range 0-16000 ms. The values in the table above are upper bounds.

!!! tip "Why jitter"
    Without randomization, all `orion-worker` replicas that failed in the same second
    (for example because of a database restart) would retry at exactly the same moment and trigger another
    overload. Jitter spreads the traffic out over time.

## Error classification

Only transient errors are retried. Permanent ones send the message straight to the DLQ, without waiting for 8 attempts.

| Code | Nature | Retried | Rationale |
| --- | --- | --- | --- |
| `ORN-5002` | transient | yes | a dependency (PostgreSQL, Redis, Kafka) is temporarily unavailable |
| `ORN-3001` | transient | yes | rate limit of the webhook recipient; retry after `X-RateLimit-Reset` |
| recipient HTTP 502/503/504 | transient | yes | failure on the recipient side |
| timeout exceeded (10 s) | transient | yes | no response within the delivery window |
| `ORN-4002` | permanent | no | idempotency conflict, the event has already been processed |
| `ORN-4005` | permanent | no | disallowed status transition, a retry changes nothing |
| `ORN-1004` | permanent | no | a required field is missing, the data is malformed |
| `ORN-2001` / `ORN-2003` | permanent | no | access configuration error; requires intervention |

!!! warning "ORN-4002 is not a failure"
    An idempotency conflict on a repeated delivery means the target system correctly
    recognized a duplicate. `orion-worker` treats such a delivery as ==successful==
    and commits the offset without writing the message to the DLQ.

## Idempotency on the consumer side

Deduplication relies on the envelope's `id` field (`evt_...`) and on the `Idempotency-Key` header for callbacks into the Orion API.

```sql title="core.idempotency_keys"
CREATE TABLE core.idempotency_keys (
    tenant_id     text        NOT NULL,
    key           text        NOT NULL,
    request_hash  text        NOT NULL,
    response_code smallint    NOT NULL,
    response_body jsonb,
    created_at    timestamptz NOT NULL DEFAULT now(),
    expires_at    timestamptz NOT NULL DEFAULT now() + interval '24 hours',
    PRIMARY KEY (tenant_id, key)
);

CREATE INDEX idx_idempotency_expires ON core.idempotency_keys (expires_at);
```

| Column | Role |
| --- | --- |
| `key` | the `Idempotency-Key` header value, max. 255 characters |
| `request_hash` | SHA-256 of the request body; a mismatch under the same key yields `ORN-4002` |
| `response_code` / `response_body` | the stored response returned on a repeat |
| `expires_at` | 24 h TTL; `orion-scheduler` removes expired entries every hour |

A repeated request with the same key and the same body returns the stored response without executing the operation again. The same key with a different body returns `ORN-4002` and HTTP status 409.

## Handling retries in code

=== "Python"

    ```python title="consumer.py" hl_lines="8 15 20"
    import random, time
    import psycopg

    TRANSIENT = {"ORN-5002", "ORN-3001"}
    MAX_ATTEMPTS = 8

    def handle(event: dict) -> None:
        if already_processed(event["id"], event["tenant"]):
            return

        for attempt in range(1, MAX_ATTEMPTS + 1):
            try:
                deliver(event, idempotency_key=event["id"])
                mark_processed(event["id"], event["tenant"])
                return
            except OrionError as exc:
                if exc.code not in TRANSIENT or attempt == MAX_ATTEMPTS:
                    raise
                cap = min(1.0 * 2 ** (attempt - 1), 64.0)
                time.sleep(random.uniform(0, cap))
    ```

=== "Java"

    ```java title="EventConsumer.java" hl_lines="10 16"
    private static final Set<String> TRANSIENT = Set.of("ORN-5002", "ORN-3001");
    private static final int MAX_ATTEMPTS = 8;

    void handle(OrionEvent event) throws OrionException {
        if (store.alreadyProcessed(event.id(), event.tenant())) {
            return;
        }
        for (int attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
            try {
                client.deliver(event, event.id());
                store.markProcessed(event.id(), event.tenant());
                return;
            } catch (OrionException exc) {
                if (!TRANSIENT.contains(exc.code()) || attempt == MAX_ATTEMPTS) {
                    throw exc;
                }
                long cap = Math.min(1000L << (attempt - 1), 64_000L);
                Thread.sleep(ThreadLocalRandom.current().nextLong(cap));
            }
        }
    }
    ```

## Protection against retry storms

`orion-worker` maintains a separate circuit breaker for each webhook target address.

| Parameter | Default value | Meaning |
| --- | --- | --- |
| `failureRateThreshold` | 50% | failure share that opens the circuit |
| `slidingWindowSize` | 100 deliveries | evaluation window |
| `minimumNumberOfCalls` | 20 | minimum number of attempts before evaluation |
| `waitDurationInOpenState` | 60 s | time spent in the open state |
| `permittedCallsInHalfOpenState` | 5 | probe attempts after the circuit opened |
| `slowCallDurationThreshold` | 5 s | a delivery above this threshold counts as slow |

In the open state, deliveries to that address are rejected without an HTTP call and counted toward the retries. This prevents flooding a recipient that is already unresponsive and blocking threads in the delivery pool.

## Moving to the DLQ and resuming

1. After the eighth failed attempt, `orion-worker` publishes the envelope to `orion.events.v1.dlq` with the headers `x-orion-error-code`, `x-orion-attempts`, and `x-orion-original-partition`.
2. The offset in `orion.events.v1` is committed, so processing of subsequent events is not blocked.
3. The `orion_events_dlq_total` counter grows, which fires an alert once 10 messages in 15 minutes are exceeded.
4. Once the cause has been removed, run `orion events replay --topic orion.events.v1.dlq --since <timestamp>`, with `--dry-run` first.
5. Replayed events keep their `id`, so deduplication at the recipient prevents double processing.

- [x] `x-orion-error-code` checked for a representative sample of messages in the DLQ.
- [x] Availability of the webhook recipient confirmed.
- [x] `--dry-run` executed before the replay.
- [ ] A replay without a time range restriction - do not do this across the full 30 day retention.

## Metrics

`orion_worker_retries_total`
:   Counter of retries with the labels `topic`, `event_type`, `error_code`, `attempt`. Growth while
    `orion_orders_processed_total` stays flat points to a degraded dependency, not to increased traffic.

`orion_events_dlq_total`
:   Counter of messages moved to the DLQ with the labels `event_type` and `error_code`.

```promql title="Share of retries in processing"
sum(rate(orion_worker_retries_total[5m]))
  / sum(rate(orion_orders_processed_total[5m])) > 0.05
```

## See also

- [Queues and topics](queues.md)
- [Event processing](index.md)
- [Error codes](../../api/rest/error-codes.md)
- [Alerts](../../operations/monitoring/alerts.md)
