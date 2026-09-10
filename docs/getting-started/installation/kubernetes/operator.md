# Install with the operator

The Orion operator manages platform instances declaratively, through the `OrionCluster` custom resource in the `orion.nimbus.example.com/v1alpha2` group. It is available only in the Enterprise license variant. The operator runs in the `orion-system` namespace and can serve many instances spread across the cluster.

## Why an operator

- **Self-healing** - every reconciliation restores the state declared in `spec`, so manually editing a `Deployment` or accidentally deleting a `Service` is repaired automatically.
- **Phase-driven migrations** - the operator holds traffic, runs `orion db migrate` in the `Migrating` phase and admits new pods only after the schema version is confirmed.
- **Backups in the lifecycle** - the schedule for database dumps and ledger exports to the `orion-artifacts` bucket is part of the resource rather than a separate process.
- **Safe version upgrades** - changing `spec.version` triggers a rolling update in a fixed order: `orion-core`, `orion-ledger`, `orion-worker`, `orion-scheduler`, `orion-gateway`, `orion-console`.
- **Consistency across instances** - a single operator keeps an identical base configuration for all tenants that have separate instances.

## Installing the operator

```bash
kubectl apply -f https://charts.nimbus.example.com/operator/4.2.1/orion-operator.yaml
kubectl -n orion-system rollout status deployment/orion-operator
kubectl get crd orionclusters.orion.nimbus.example.com
```

Expected output of the last command:

```text
NAME                                          CREATED AT
orionclusters.orion.nimbus.example.com        2025-11-04T09:12:47Z
```

!!! note "The operator manifest requires cluster-admin permissions"
    The manifest creates the CRD, a `ClusterRole`, a `ClusterRoleBinding` and a validating webhook. A cluster administrator performs this installation once; creating `OrionCluster` resources afterwards only requires permissions in the instance namespace.

## The OrionCluster resource

```yaml title="orioncluster.yaml" hl_lines="1 6 7 33 46"
apiVersion: orion.nimbus.example.com/v1alpha2
kind: OrionCluster
metadata:
  name: orion-prod
  namespace: orion
spec:
  version: "4.2.1"
  profile: prod
  imageRegistry: registry.nimbus.example.com/orion
  imagePullSecrets:
    - name: orion-registry
  logLevel: info

  database:
    secretRef: orion-db
    schemas: [core, ledger, audit]
    migrationPolicy: Automatic

  kafka:
    secretRef: orion-kafka
    topic: orion.events.v1
    dlqTopic: orion.events.v1.dlq

  redis:
    secretRef: orion-redis

  objectStorage:
    secretRef: orion-s3

  oidc:
    secretRef: orion-oidc
    realm: orion

  signing:
    secretRef: orion-signing

  components:
    gateway:
      replicas: 3
      rateLimit:
        requestsPerMinute: 600
        burst: 100
      ingress:
        enabled: true
        className: nginx
        host: api.orion.example.com
        tlsIssuer: letsencrypt-prod
    core:
      replicas: 3
      autoscaling:
        enabled: true
        minReplicas: 3
        maxReplicas: 12
    ledger:
      replicas: 2
    worker:
      replicas: 6
      concurrency: 4
      maxRetries: 5
    scheduler:
      replicas: 2
    console:
      replicas: 2
      ingress:
        enabled: true
        className: nginx
        host: console.orion.example.com

  backup:
    enabled: true
    schedule: "0 2 * * *"
    retentionDays: 30
    destination: s3://orion-artifacts/backups/orion-prod

  monitoring:
    serviceMonitor: true
    scrapeInterval: 30s

  updateStrategy:
    type: RollingUpdate
    maxUnavailable: 1
```

Applying it:

```bash
kubectl apply -f orioncluster.yaml
kubectl -n orion get orioncluster
```

## spec fields

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `spec.version` | string | - | Platform version, which is also the image tag; required |
| `spec.profile` | enum | `staging` | Configuration profile: `dev`, `staging`, `prod` |
| `spec.imageRegistry` | string | `registry.nimbus.example.com/orion` | Image repository prefix |
| `spec.logLevel` | enum | `info` | Log level for all components |
| `spec.database.secretRef` | string | - | `Secret` with the `orion` database credentials; required |
| `spec.database.migrationPolicy` | enum | `Automatic` | `Automatic` or `Manual`; with `Manual` the operator waits for an approval annotation |
| `spec.kafka.secretRef` | string | - | `Secret` with the broker list and SASL credentials; required |
| `spec.kafka.dlqTopic` | string | `orion.events.v1.dlq` | Dead letter queue topic |
| `spec.redis.secretRef` | string | - | `Secret` with the Redis address; required |
| `spec.objectStorage.secretRef` | string | - | `Secret` with access to the `orion-artifacts` bucket; required |
| `spec.oidc.realm` | string | `orion` | Keycloak realm used to validate tokens |
| `spec.components.<name>.replicas` | int | component-specific | Number of replicas of the given component |
| `spec.components.core.autoscaling.maxReplicas` | int | `10` | Upper bound for automatic scaling of `orion-core` |
| `spec.components.worker.maxRetries` | int | `5` | Number of retries before an event is moved to the DLQ |
| `spec.backup.schedule` | string | `""` | Cron schedule for dumps; empty disables backups |
| `spec.backup.retentionDays` | int | `14` | Retention period for backups in S3 |
| `spec.updateStrategy.maxUnavailable` | int | `1` | Maximum number of unavailable pods during an update |
| `spec.monitoring.serviceMonitor` | bool | `false` | Creates a `ServiceMonitor` object |

## The reconciliation loop

1. The operator receives a change event for the `OrionCluster` resource or an event from a watched child object.
2. It validates `spec` against the CRD schema and checks that every referenced `Secret` object exists; a missing secret results in the `Degraded` phase.
3. It verifies connectivity to PostgreSQL, Redis, Kafka, S3 and Keycloak and reads the current database schema version.
4. If the schema version is lower than `spec.version`, it enters the `Migrating` phase and runs the `orion db migrate` job as the migrator role.
5. It creates or updates the `ConfigMap` holding the generated `orion.yaml`, plus the `Deployment` objects for the six components.
6. It creates `Service`, `Ingress`, `ServiceMonitor`, `PodDisruptionBudget` and `HorizontalPodAutoscaler` objects according to `spec.components`.
7. It performs a rolling update in the fixed order, respecting `spec.updateStrategy.maxUnavailable`.
8. It polls `/healthz` on every component and records the result in `status.conditions`.
9. It sets `status.phase` to `Ready` once all replicas are ready and the schema version matches `spec.version`.
10. It reconciles again every 60 seconds, or immediately after a watched object changes.

## Phases in status.phase

| Phase | Meaning | Operator behavior |
| --- | --- | --- |
| `Pending` | Resource accepted, validation and dependency checks in progress | waits for secrets and for dependencies to become available |
| `Migrating` | Schema migrations are running | holds back the startup of new application pods |
| `Ready` | All components ready, schema matches `spec.version` | maintains the state, reconciles every 60 s |
| `Degraded` | Some replicas unavailable or a dependency unreachable | retries the reconciliation with exponential backoff, emits an event |
| `Upgrading` | A rolling update is running after a `spec.version` change | replaces pods component by component |

## Inspecting the state

```bash
kubectl -n orion describe orioncluster orion-prod
```

```text
Name:         orion-prod
Namespace:    orion
API Version:  orion.nimbus.example.com/v1alpha2
Kind:         OrionCluster
Spec:
  Version:  4.2.1
  Profile:  prod
Status:
  Phase:            Ready
  Observed Version: 4.2.1
  Schema Version:   4.2.1
  Ready Components: 6/6
  Last Backup:      2025-11-04T02:00:11Z
  Conditions:
    Type            Status   Reason               Last Transition
    ----            ------   ------               ---------------
    Validated       True     SpecValid            2025-11-04T09:13:02Z
    Migrated        True     SchemaUpToDate       2025-11-04T09:14:38Z
    Available       True     AllComponentsReady   2025-11-04T09:17:05Z
    BackupHealthy   True     LastBackupSucceeded  2025-11-04T02:00:11Z
Events:
  Type    Reason              Age   From            Message
  ----    ------              ----  ----            -------
  Normal  MigrationStarted    9m    orion-operator  running orion db migrate for 4.2.1
  Normal  MigrationSucceeded  8m    orion-operator  schema core/ledger/audit at 4.2.1
  Normal  ComponentsReady     6m    orion-operator  6/6 components report healthz ok
  Normal  BackupSucceeded     4m    orion-operator  dump uploaded to s3://orion-artifacts/backups/orion-prod
```

A quick overview of all instances:

```bash
kubectl get orioncluster -A
```

```text
NAMESPACE   NAME          VERSION   PROFILE   PHASE   READY   AGE
orion       orion-prod    4.2.1     prod      Ready   6/6     31d
orion-uat   orion-uat     4.2.1     staging   Ready   6/6     31d
```

!!! danger "Do not combine the operator with the Helm chart"
    The operator considers itself the sole owner of the objects in the instance namespace. If a Helm release of the `nimbus/orion` chart exists in the same namespace, both controllers will overwrite the `Deployment` and `ConfigMap` objects in turn. A typical result is two active `orion-scheduler` leaders and billing jobs executed twice in the ledger. Before moving to the operator, remove the Helm release with `helm uninstall orion -n orion` and confirm that no objects labeled `app.kubernetes.io/managed-by=Helm` are left.

??? info "Migrations in Manual mode"
    With `spec.database.migrationPolicy: Manual`, the operator stops in the `Pending` phase and waits for approval:

    ```bash
    kubectl -n orion annotate orioncluster orion-prod \
      orion.nimbus.example.com/approve-migration="4.2.1" --overwrite
    ```

    This mode is used in environments where schema changes require a maintenance window.

## See also

- [Deploy on Kubernetes](index.md)
- [Install with Helm](helm.md)
- [Backups](../../../operations/backups.md)
- [Alerts](../../../operations/monitoring/alerts.md)
