# Backup and restore

Orion Platform 4.2 LTS keeps persistent state in PostgreSQL 15 (the `orion` database with the `core`, `ledger` and `audit` schemas) and in S3 object storage (the `orion-artifacts` bucket). Backups must guarantee an RPO of 5 minutes and an RTO of 60 minutes. This page describes the backup scope, the schedule, the point-in-time recovery procedure and the mandatory quarterly tests.

## Backup scope

| Resource | Method | Frequency | Retention | RPO |
| --- | --- | --- | --- | --- |
| PostgreSQL 15 (`orion` database) | `pg_basebackup` + WAL archiving | full at 01:00 UTC, WAL every 5 min | 30 days | 5 min |
| S3 `orion-artifacts` | versioned snapshot + cross-region replication | weekly (Sun 03:00 UTC) | 90 days | 7 days |
| Secrets (Vault / ExternalSecrets) | encrypted export to `s3://orion-backups/secrets/` | after every change | 365 days | 0 |
| Kafka configuration (topics, ACLs, offsets) | `orion backup create --scope kafka` | daily at 01:30 UTC | 30 days | 24 h |
| Webhook and tenant definitions | included in the `core` schema | same as the database | 30 days | 5 min |
| `/etc/orion/orion.yaml` files, Helm values | Git repository + copy in S3 | on change | indefinite | 0 |

!!! note "Redis is not backed up"
    Redis 7 acts purely as a cache and a rate limit store. Losing the instance causes a temporary rise in latency and resets the `ORN-3001` counters, but no data loss. Details in [scaling](scaling.md).

## Schedule

| Time UTC | Job | Executed by |
| --- | --- | --- |
| 01:00 daily | full PostgreSQL backup to `s3://orion-backups/pg/base/` | `orion-scheduler`, `pg-basebackup` job |
| every 5 min | archiving of WAL segments to `s3://orion-backups/pg/wal/` | PostgreSQL `archive_command` |
| 01:30 daily | backup of Kafka metadata and configuration | `orion-scheduler`, `kafka-meta` job |
| 03:00 on Sunday | snapshot of the `orion-artifacts` bucket | S3 lifecycle policy |
| 04:00 on the first day of the month | restore drill into the `restore-test` environment | `orion-scheduler`, `restore-drill` job |

## Creating a backup

A logical backup from the CLI, to be used before a migration or a larger change:

```bash
orion backup create \
  --scope db,kafka,config \
  --label pre-upgrade-4.2.1 \
  --destination s3://orion-backups/manual/
```

```text
snapshot id     bkp_01HQ9F3M7T4XZ8
scope           db, kafka, config
db schemas      core (18.2 GiB), ledger (7.4 GiB), audit (31.9 GiB)
wal position    0/8A3C1D40
checksum        sha256:9c1f...4ab2
duration        11m42s
status          COMPLETED
```

A physical backup taken directly on the database host:

```bash
pg_basebackup \
  --host=pg-primary.internal.example.com \
  --username=orion_backup \
  --pgdata=/var/backups/orion/base-$(date -u +%Y%m%d) \
  --wal-method=stream \
  --checkpoint=fast \
  --progress --verbose \
  --compress=zstd:6
```

The state of the WAL chain and the oldest available recovery point:

```bash
orion db status --backups
```

```text
schema versions   core=4.2.1  ledger=4.2.1  audit=4.2.1
latest base       2026-02-11T01:00:04Z  (s3://orion-backups/pg/base/20260211/)
wal continuity    OK  (0/8A3C1D40 .. 0/9F02B118, gaps: 0)
earliest PITR     2026-01-12T01:00:04Z
newest PITR       2026-02-11T09:20:00Z
```

## Verifying backups

A backup without a confirmed restore is not a backup. Two levels of verification apply.

1. Automatic, after every run: verification of the `sha256` checksum of the S3 object and of WAL continuity.
2. Monthly, via the `restore-drill` job: restoring the full database into the `restore-test` environment and comparing control counts.

```bash
orion backup verify --snapshot bkp_01HQ9F3M7T4XZ8 --deep
```

```text
manifest        ok (412 objects)
checksums       ok (412/412)
wal continuity  ok (0 gaps)
restore probe   ok (pg_verifybackup: backup successfully verified)
```

The control counts compared after a restore:

```sql
SELECT 'orders' AS rel, count(*), max(updated_at) FROM core.orders
UNION ALL
SELECT 'payments', count(*), max(updated_at) FROM ledger.payments
UNION ALL
SELECT 'audit_events', count(*), max(created_at) FROM audit.events;
```

## Point-in-time recovery

The PITR procedure for a logical corruption scenario (for example a bad migration or a mass data deletion). Expected duration: 35 to 55 minutes.

1. **Stop writes.** Scale the writing components to zero and switch the gateway to read-only mode.

    ```bash
    kubectl scale -n orion deploy/orion-core deploy/orion-ledger deploy/orion-worker --replicas=0
    kubectl set env -n orion deploy/orion-gateway ORION_READ_ONLY=true
    ```

    Checkpoint: `POST /orders` returns `503` with the `ORN-5002` code, and `orion_db_pool_in_use` for `core` drops to zero.

2. **Determine the target moment.** Pick a timestamp immediately preceding the event, using the `audit` schema.

    ```sql
    SELECT created_at, actor, action, entity_id
    FROM audit.events
    WHERE created_at > '2026-02-11T08:00:00Z'
    ORDER BY created_at
    LIMIT 20;
    ```

    Checkpoint: you have a single `recovery_target_time` value accurate to the second.

3. **Restore the base backup.** Unpack the last full backup preceding the target moment into an empty data directory.

    ```bash
    orion backup restore \
      --snapshot bkp_01HQ9F3M7T4XZ8 \
      --target-time '2026-02-11T08:41:30Z' \
      --pgdata /var/lib/postgresql/15/restore \
      --no-start
    ```

    Checkpoint: the directory contains `backup_label` and `postgresql.auto.conf` with `restore_command`.

4. **Start WAL recovery.** Start the instance and watch the progress.

    ```bash
    systemctl start postgresql@15-restore
    journalctl -u postgresql@15-restore -f | grep -E 'recovery|consistent'
    ```

    ```text
    LOG:  starting point-in-time recovery to 2026-02-11 08:41:30+00
    LOG:  restored log file "0000000100000000000000A3" from archive
    LOG:  redo done at 0/A31F7B88
    LOG:  recovery stopping before commit of transaction 918442
    ```

    Checkpoint: `SELECT pg_is_in_recovery();` returns `t`, with no `FATAL` errors.

5. **Verify the data before promotion.** Compare the control counts from the verification step and check order consistency.

    ```sql
    SELECT status, count(*) FROM core.orders GROUP BY status ORDER BY 2 DESC;
    SELECT count(*) FROM core.orders o
      LEFT JOIN ledger.payments p ON p.order_id = o.id
      WHERE o.status = 'paid' AND p.id IS NULL;
    ```

    Checkpoint: the second query returns `0`. Any non-zero value calls for escalation instead of promotion.

6. **Promote the instance.** Finish recovery and switch traffic over.

    ```bash
    psql -h localhost -c "SELECT pg_wal_replay_resume();"
    psql -h localhost -c "SELECT pg_promote(wait => true, wait_seconds => 60);"
    ```

    Checkpoint: `pg_is_in_recovery()` returns `f`.

7. **Bring the application back.** Update `ORION_DB_URL`, run migrations in check mode and restore the replicas.

    ```bash
    orion db status
    kubectl set env -n orion deploy/orion-gateway ORION_READ_ONLY=false
    kubectl scale -n orion deploy/orion-core --replicas=4
    kubectl scale -n orion deploy/orion-ledger --replicas=3
    kubectl scale -n orion deploy/orion-worker --replicas=6
    ```

    Checkpoint: `orion_http_requests_total{status=~"5.."}` falls back below 0.5% and `orion_queue_lag` decreases.

8. **Replay the outstanding events.** Replay events published after the target moment from Kafka.

    ```bash
    orion events replay --topic orion.events.v1 \
      --from '2026-02-11T08:41:30Z' --to '2026-02-11T09:20:00Z' --dry-run
    ```

    Checkpoint: the `--dry-run` report shows no idempotency conflicts. Details in [retries](../guides/events/retries.md).

!!! danger "Backup taken before a database migration"
    Before every `orion db migrate`, take a backup labelled `pre-upgrade-<version>` and confirm that its status is `COMPLETED`. The 4.2.x migrations include irreversible type changes in the `ledger` schema, so once they are applied, restoring an earlier state requires a full PITR and the loss of transactions written after the migration.

## Restoring a single tenant

Data corruption affecting one tenant does not require restoring the entire installation.

```bash
orion backup restore \
  --snapshot bkp_01HQ9F3M7T4XZ8 \
  --tenant acme-retail \
  --target-time '2026-02-11T08:41:30Z' \
  --into-schema core_restore
```

```text
restoring tenant acme-retail scope: core, ledger
core.orders            12 841 rows
core.customers          3 208 rows
ledger.payments         9 774 rows
target schema          core_restore (isolated)
status                 COMPLETED (4m11s)
```

After restoring into an isolated schema, compare the data and move only the missing records. Tenant isolation rules are described in [multi-tenancy](../guides/multi-tenancy.md).

## RPO and RTO per scenario

| Scenario | RPO | RTO | Procedure |
| --- | --- | --- | --- |
| Disk failure on a database node | 0 | 10 min | Failover to the synchronous replica |
| Logical data corruption | 5 min | 55 min | PITR following the procedure above |
| Loss of an entire region | 5 min | 60 min | Restore from the replicated backup in the standby region |
| Operator error (tenant deleted) | 5 min | 30 min | Single-tenant restore |
| Loss of the `orion-artifacts` bucket | 7 days | 45 min | Versioned snapshot + replication |
| Loss of the Kafka cluster | 24 h | 40 min | Metadata restore, `orion events replay` |

## Backup encryption

- Objects in `s3://orion-backups/` are encrypted with SSE-KMS using the `alias/orion-backup` key.
- The key is rotated automatically once every 12 months; older versions remain available to decrypt the archive.
- Only the `orion-backup-writer` (write) and `orion-restore-operator` (read) roles have access to the key.
- Secret exports are additionally encrypted with `age`, using a key stored outside the cloud provider.

```bash
aws kms describe-key --key-id alias/orion-backup --query 'KeyMetadata.[KeyId,KeyState,KeyRotationEnabled]'
```

## Restore drill: quarterly checklist

- [ ] Pick a random backup from the last 30 days and confirm `orion backup verify --deep`.
- [ ] Restore the full database into the `restore-test` environment and measure the actual time to promotion.
- [ ] Compare the control counts for `core.orders`, `ledger.payments` and `audit.events`.
- [ ] Perform a PITR to a point halfway through the WAL window and confirm it stops at the right transaction.
- [ ] Test a single-tenant restore into an isolated schema.
- [ ] Check that the restored instance passes `orion config validate` and `orion db status`.
- [ ] Verify access to the KMS key from the `orion-restore-operator` role.
- [ ] Update the measured RTO in the scenario table and report deviations above 20%.
- [ ] Delete the `restore-test` environment and confirm that no snapshots were left behind.

## See also

- [Runbook: PostgreSQL outage](runbooks/database-outage.md)
- [Operational security](security.md)
- [Migrating from 3.8 to 4.2](../guides/migrations/from-3-8-to-4-2.md)
- [Operations](index.md)
