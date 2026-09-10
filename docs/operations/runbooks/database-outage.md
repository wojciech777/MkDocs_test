# Runbook: PostgreSQL outage

This runbook covers unavailability or degradation of the PostgreSQL 15 instance serving the `orion` database (the `core`, `ledger` and `audit` schemas). It applies to the `OrionDbPoolSaturated`, `OrionDbReplicationLag` and `OrionDiskFilling` alerts. The database is the only element whose loss immediately halts writes across the whole platform.

## Symptoms

- `503` responses with the `ORN-5002` code on `POST /orders` and `POST /payments`, occasionally `ORN-5009` when the wait timeout is exceeded.
- `orion_db_pool_waiting` rises above 5 and does not return to zero; `orion_db_pool_in_use` sits at the pool maximum.
- The `/readyz` probes of `orion-core` and `orion-ledger` return `503`, and pods drop out of the service endpoints.
- Repeated `msg="db connection acquire timeout"` entries with `err_code="ORN-5009"` in the `orion-core` logs.
- Orders stall in the `pending_payment` status and `orion_orders_processed_total` stops rising.
- With a WAL problem: growing usage of the archive volume and `archive_command failed` warnings.

## Impact and severity

| Scope | Severity | Rationale |
| --- | --- | --- |
| Database unavailable, no writes and no reads | P1 | Complete API unavailability |
| Primary unavailable, replica serving reads | P1 | Loss of write capability for all tenants |
| Saturated connection pool, some requests get through | P2 | Degradation with an SLO breach |
| Replication lag above 60 s | P2 | Risk of exceeding the 5 min RPO |
| Suspected data corruption | P1 | Potential data loss |

Response time for P1: 30 minutes. Notify `#orion-oncall`, appoint an incident commander and update `https://status.orion.example.com`.

## Quick diagnosis

State of the database instances in the data namespace:

```bash
kubectl get pods -n orion-data -l app.kubernetes.io/name=postgresql -o wide
```

```text
NAME              READY   STATUS    RESTARTS   AGE    ROLE
pg-orion-0        1/1     Running   0          31d    primary
pg-orion-1        1/1     Running   0          31d    replica
pg-orion-2        0/1     Running   4 (2m ago) 31d    replica
```

A summary from the application's point of view:

```bash
kubectl exec -n orion deploy/orion-core -- orion db status
```

```text
connection      pg-orion-0.orion-data:5432   FAILED (timeout after 5s)
schema versions unknown
pool            in_use=25/25  waiting=41
last success    2026-02-11T09:08:41Z
```

Active connections and wait states:

```sql
SELECT state, wait_event_type, count(*)
FROM pg_stat_activity
WHERE datname = 'orion'
GROUP BY 1, 2
ORDER BY 3 DESC;
```

```text
       state        | wait_event_type | count
--------------------+-----------------+-------
 idle in transaction| Client          |   118
 active             | Lock            |    46
 active             |                 |     9
 idle               | Client          |     7
```

Longest-running queries:

```sql
SELECT pid, now() - query_start AS duration, wait_event, left(query, 60) AS query
FROM pg_stat_activity
WHERE datname = 'orion' AND state <> 'idle'
ORDER BY duration DESC
LIMIT 10;
```

Replication state:

```sql
SELECT client_addr, state, sync_state,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replay_bytes,
       replay_lag
FROM pg_stat_replication;
```

```text
 client_addr  |  state  | sync_state | replay_bytes | replay_lag
--------------+---------+------------+--------------+------------
 10.42.3.17   | streaming | sync     |       182144 | 00:00:01.2
 10.42.3.18   | catchup   | async    |    918273645 | 00:04:51.7
```

Disk space and WAL archiving backlog:

```bash
kubectl exec -n orion-data pg-orion-0 -- df -h /var/lib/postgresql/data
kubectl exec -n orion-data pg-orion-0 -- \
  psql -U postgres -Atc "SELECT last_failed_wal, failed_count FROM pg_stat_archiver;"
```

```text
Filesystem      Size  Used Avail Use% Mounted on
/dev/nvme1n1    500G  478G   22G  96% /var/lib/postgresql/data
0000000100000000000000B7|312
```

!!! warning "Volume 96% full"
    Above 90% usage PostgreSQL still runs, but filling the `pg_wal` directory stops all writes and may prevent the instance from starting. Treat this as the priority path (variant C below).

## Procedure

### Step 1: limit the damage

1. Switch `orion-gateway` to read-only mode to stop rejected writes from piling up.

    ```bash
    kubectl set env -n orion deploy/orion-gateway ORION_READ_ONLY=true
    ```

    Checkpoint: `POST /orders` returns `503` with `ORN-5002` immediately, without the 5-second wait; `orion_db_pool_waiting` stops rising.

2. Determine the failure variant from the diagnosis and move to the matching section: A (connection overload), B (primary down), C (full WAL volume), D (data corruption).

### Variant A: connection overload

1. Terminate connections that have been idle in transaction for more than 5 minutes.

    ```sql
    SELECT pg_terminate_backend(pid), pid, application_name
    FROM pg_stat_activity
    WHERE datname = 'orion'
      AND state = 'idle in transaction'
      AND now() - state_change > interval '5 minutes';
    ```

    Checkpoint: the number of `idle in transaction` connections drops below 10 and `orion_db_pool_waiting` decreases.

2. Identify the component responsible for the connection leak from the `application_name` field and perform a rolling restart.

    ```bash
    kubectl rollout restart -n orion deploy/orion-ledger
    kubectl rollout status -n orion deploy/orion-ledger --timeout=180s
    ```

    Checkpoint: `orion_db_pool_in_use` for `ledger` falls back to the baseline and stays stable for 5 minutes.

3. If the sum of the pools exceeds 60% of `max_connections`, reduce the replica count or lower `pool_size` according to the formula in [scaling](../scaling.md).

    Checkpoint: `sum(orion_db_pool_in_use) < max_connections − 25`.

### Variant B: primary down and failover

1. Confirm that the primary is genuinely unavailable and not merely overloaded.

    ```bash
    kubectl exec -n orion-data pg-orion-1 -- pg_isready -h pg-orion-0.orion-data -p 5432
    ```

    ```text
    pg-orion-0.orion-data:5432 - no response
    ```

    Checkpoint: no response on two attempts 30 s apart.

2. Check which replica has the smallest replay lag and perform the switchover.

    ```bash
    kubectl exec -n orion-data pg-orion-1 -- \
      psql -U postgres -Atc "SELECT pg_last_wal_replay_lsn();"
    kubectl exec -n orion-data pg-orion-1 -- patronictl -c /etc/patroni.yml failover --candidate pg-orion-1
    ```

    Checkpoint: `patronictl list` shows `pg-orion-1` in the `Leader` role with status `running`.

3. Confirm that the application connected to the new leader. The service entry is updated automatically; if nothing happens, perform a rolling restart of the components.

    ```bash
    kubectl exec -n orion deploy/orion-core -- orion db status
    ```

    Checkpoint: `connection ... OK`, `schema versions core=4.2.1 ledger=4.2.1 audit=4.2.1`.

4. Rebuild the former primary as a replica only after service is restored, not during the incident.

### Variant C: full WAL volume

1. Free up space immediately by removing segments that have already been archived.

    ```bash
    kubectl exec -n orion-data pg-orion-0 -- \
      psql -U postgres -c "CHECKPOINT;"
    kubectl exec -n orion-data pg-orion-0 -- pg_archivecleanup /var/lib/postgresql/data/pg_wal 0000000100000000000000B0
    ```

    Checkpoint: volume usage drops below 85%.

2. Establish why archiving is failing; most often it is a lack of access to `s3://orion-backups/pg/wal/` or an inactive replication slot.

    ```sql
    SELECT slot_name, active, pg_size_pretty(
             pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
    FROM pg_replication_slots;
    ```

    Checkpoint: no inactive slots holding back more than 10 GiB of WAL.

3. Drop only the slots confirmed to be abandoned.

    ```sql
    SELECT pg_drop_replication_slot('pg_orion_2_stale');
    ```

    Checkpoint: `pg_stat_archiver.failed_count` stops rising and volume usage decreases.

4. Extend the volume if usage still exceeds 80% after the cleanup.

    Checkpoint: the fill forecast is beyond 14 days and the `OrionDiskFilling` alert resolves.

### Variant D: data corruption

!!! danger "Do not try to repair data in place"
    On `invalid page header` or `could not read block` errors, or an inconsistency between `core.orders` and `ledger.payments`, do not run `UPDATE` or `DELETE` in production. Stop writes and move to a restore following the PITR procedure in [backups](../backups.md).

1. Stop all writing components.

    ```bash
    kubectl scale -n orion deploy/orion-core deploy/orion-ledger deploy/orion-worker --replicas=0
    ```

    Checkpoint: `pg_stat_activity` contains no connections whose `application_name` starts with `orion-`.

2. Preserve the evidence: collect a diagnostic bundle and take a backup of the current state before making any change.

    ```bash
    orion diag bundle --since 3h --components core,ledger --out /tmp/orion-db-incident.tar.gz
    orion backup create --scope db --label incident-$(date -u +%Y%m%dT%H%MZ)
    ```

    Checkpoint: the backup status is `COMPLETED`.

3. Perform a PITR to a point before the corruption, following the procedure in [backups](../backups.md).

    Checkpoint: the control counts match and `pg_is_in_recovery()` returns `f`.

### Final step: resume writes and verify consistency

1. Restore writes and the replicas.

    ```bash
    kubectl set env -n orion deploy/orion-gateway ORION_READ_ONLY=false
    kubectl scale -n orion deploy/orion-core --replicas=4
    kubectl scale -n orion deploy/orion-ledger --replicas=3
    kubectl scale -n orion deploy/orion-worker --replicas=6
    ```

    Checkpoint: `orion_http_requests_total{status=~"5.."}` stays below 0.5% for 10 minutes.

2. Verify the consistency of orders and payments.

    ```sql
    -- paid orders with no ledger entry
    SELECT count(*) FROM core.orders o
      LEFT JOIN ledger.payments p ON p.order_id = o.id
      WHERE o.status = 'paid' AND p.id IS NULL;

    -- orders stuck in pending_payment for more than an hour
    SELECT count(*) FROM core.orders
      WHERE status = 'pending_payment' AND updated_at < now() - interval '1 hour';

    -- status distribution, compare with the values from before the incident
    SELECT status, count(*) FROM core.orders GROUP BY status ORDER BY 2 DESC;
    ```

    Checkpoint: the first query returns `0`; a non-zero value means the ledger has to be reconciled manually.

3. Replay the events lost during the outage.

    ```bash
    orion events replay --topic orion.events.v1 --from '2026-02-11T09:08:00Z' --dry-run
    ```

    Checkpoint: the report shows no idempotency conflicts, after which you can run it without `--dry-run`.

## Escalation

| Condition | Recipient | Deadline |
| --- | --- | --- |
| Writes not restored after 20 minutes | The incident commander raises it to P1 and calls the on-call DBA | immediately |
| Failover does not succeed after two attempts | The database service provider, ticket with critical priority | within 10 minutes |
| Suspected data corruption or a checksum error | DBA + the "Platform Core" team (Marek Debski) | immediately |
| Restore from backup will exceed the 60 minute RTO | Product owner (Anna Kowalska), announcement to tenants | within 30 minutes |
| Data disclosure or loss of audit entries | Security team, `support@nimbus.example.com` | within 24 h |

## After the incident

- [ ] Complete the timeline with every action taken and its effect.
- [ ] Confirm zero `orion_queue_lag` and an empty `orion.events.v1.dlq`.
- [ ] Check that the next scheduled backup finished with the `COMPLETED` status.
- [ ] Rebuild the former primary as a replica and confirm `sync_state = sync`.
- [ ] Manually reconcile the orders flagged by the consistency queries.
- [ ] Update `https://status.orion.example.com` with a closing post.
- [ ] Schedule the postmortem within 5 business days.
- [ ] Verify whether the alert thresholds detected the event with enough lead time; adjust them in [alerts](../monitoring/alerts.md).
- [ ] Close the tenant tickets on the support portal.

## Prevention

- Keep the sum of the connection pools below 60% of `max_connections`; once it is exceeded, introduce pgBouncer in `transaction` mode.
- Set `idle_in_transaction_session_timeout = 60s` and `statement_timeout = 30s` for the `orion_app` role.
- Monitor `orion_db_pool_waiting` (alert from a value of 5) and the WAL volume fill forecast with 4 h of lead time.
- Keep at least one synchronous replica and test failover once a quarter in a maintenance window.
- Before every `orion db migrate`, create a backup labelled `pre-upgrade-<version>`.
- Enable `pg_stat_statements` and review the 10 most expensive queries at the weekly review.
- Partition `audit.events` monthly to keep index sizes down.

## See also

- [Runbooks](index.md)
- [Backup and restore](../backups.md)
- [Scaling and performance](../scaling.md)
- [Alerts](../monitoring/alerts.md)
