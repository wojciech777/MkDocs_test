# Install with Helm

The `nimbus/orion` chart in version 4.2.1 installs all six Orion Platform components along with the `Service`, `Ingress`, `ServiceMonitor` and `HorizontalPodAutoscaler` objects. The chart does not install external dependencies, so PostgreSQL, Redis, Kafka, MinIO and Keycloak must already be available. Create the secrets described in [Deploy on Kubernetes](index.md) before you install.

## Adding the repository

```bash
helm repo add nimbus https://charts.nimbus.example.com
helm repo update nimbus
helm search repo nimbus/orion --versions
```

Expected output:

```text
NAME            CHART VERSION   APP VERSION   DESCRIPTION
nimbus/orion    4.2.1           4.2.1         Orion Platform - orders, payments, events
nimbus/orion    4.2.0           4.2.0         Orion Platform - orders, payments, events
nimbus/orion    4.0.7           4.0.7         Orion Platform - orders, payments, events
nimbus/orion    3.8.14          3.8.14        Orion Platform - orders, payments, events (LTS)
```

## The values.yaml file

```yaml title="values.yaml" hl_lines="2 6 20 21 39"
global:
  profile: prod
  image:
    registry: registry.nimbus.example.com
    repository: orion
    tag: "4.2.1"
    pullPolicy: IfNotPresent
  imagePullSecrets:
    - name: orion-registry
  logLevel: info

database:
  existingSecret: orion-db
  schemas:
    core: core
    ledger: ledger
    audit: audit

migrations:
  enabled: true
  asHook: true

redis:
  existingSecret: orion-redis

kafka:
  existingSecret: orion-kafka
  topic: orion.events.v1
  dlqTopic: orion.events.v1.dlq

objectStorage:
  existingSecret: orion-s3

oidc:
  existingSecret: orion-oidc
  realm: orion

signing:
  existingSecret: orion-signing

gateway:
  replicaCount: 3
  rateLimit:
    requestsPerMinute: 600
    burst: 100
  resources:
    requests: { cpu: 250m, memory: 512Mi }
    limits: { cpu: "1", memory: 1Gi }
  ingress:
    enabled: true
    className: nginx
    host: api.orion.example.com
    tls:
      enabled: true
      issuer: letsencrypt-prod

core:
  replicaCount: 3
  autoscaling:
    enabled: true
    minReplicas: 3
    maxReplicas: 12
    targetCPUUtilizationPercentage: 70

ledger:
  replicaCount: 2

worker:
  replicaCount: 6
  concurrency: 4
  maxRetries: 5

scheduler:
  replicaCount: 2

console:
  replicaCount: 2
  ingress:
    enabled: true
    className: nginx
    host: console.orion.example.com

monitoring:
  serviceMonitor:
    enabled: true
    interval: 30s
```

## Installation

```bash
helm upgrade --install orion nimbus/orion \
  --version 4.2.1 \
  --namespace orion --create-namespace \
  --values values.yaml \
  --wait --timeout 15m
```

!!! tip "Always use upgrade --install"
    This form is idempotent: on the first run it creates the release, on later runs it updates it. That makes it easy to wire the installation into a CI/CD pipeline without branching between `helm install` and `helm upgrade`.

Previewing the manifests without deploying:

```bash
helm template orion nimbus/orion --version 4.2.1 \
  --namespace orion --values values.yaml > /tmp/orion-manifests.yaml
```

## Key values

| Key | Type | Default | Description |
| --- | --- | --- | --- |
| `global.profile` | string | `staging` | Configuration profile: `dev`, `staging` or `prod` |
| `global.image.tag` | string | `4.2.1` | Image tag for all components |
| `global.logLevel` | string | `info` | Log level passed as `ORION_LOG_LEVEL` |
| `global.imagePullSecrets` | list | `[]` | Secrets granting access to the image registry |
| `database.existingSecret` | string | `""` | Name of the `Secret` holding the `orion` database credentials |
| `migrations.enabled` | bool | `true` | Runs `orion db migrate` as a `Job` |
| `migrations.asHook` | bool | `true` | Runs the migrations as a `pre-install`/`pre-upgrade` hook |
| `kafka.topic` | string | `orion.events.v1` | Domain event topic |
| `kafka.dlqTopic` | string | `orion.events.v1.dlq` | Dead letter queue topic |
| `gateway.replicaCount` | int | `2` | Number of `orion-gateway` replicas |
| `gateway.rateLimit.requestsPerMinute` | int | `600` | Rate limit per key in a one-minute window |
| `gateway.rateLimit.burst` | int | `100` | Allowed short-term excess of requests |
| `gateway.ingress.enabled` | bool | `false` | Creates an `Ingress` object for the API |
| `gateway.ingress.host` | string | `""` | API host name, for example `api.orion.example.com` |
| `core.autoscaling.enabled` | bool | `false` | Enables the `HorizontalPodAutoscaler` for `orion-core` |
| `core.autoscaling.maxReplicas` | int | `10` | Upper scaling bound for `orion-core` |
| `worker.concurrency` | int | `2` | Number of events processed in parallel per replica |
| `worker.maxRetries` | int | `5` | Number of retries before an event is moved to the DLQ |
| `scheduler.replicaCount` | int | `1` | `orion-scheduler` replicas; the extra ones stay on standby |
| `console.ingress.host` | string | `""` | Console host name, for example `console.orion.example.com` |
| `monitoring.serviceMonitor.enabled` | bool | `false` | Creates a `ServiceMonitor` for the Prometheus Operator |
| `podDisruptionBudget.enabled` | bool | `true` | Protects availability while nodes are drained |

The full list of values is returned by:

```bash
helm show values nimbus/orion --version 4.2.1
```

## Upgrade and rollback

Upgrading to a newer patch release:

```bash
helm repo update nimbus
helm upgrade --install orion nimbus/orion \
  --version 4.2.1 \
  --namespace orion --values values.yaml --wait
```

Release history and rollback:

```bash
helm history orion -n orion
helm rollback orion 3 -n orion --wait
```

!!! danger "helm rollback does not roll back database migrations"
    Schema migrations are forward-only. After a release rollback, images from the older version may encounter a schema newer than expected and fail with `ORN-5003`. Rolling back across minor versions requires restoring the database from a backup, see [Backups](../../../operations/backups.md).

## Verification

```bash
kubectl get pods -n orion
```

```text
NAME                                READY   STATUS      RESTARTS   AGE
orion-console-6c4f7d8b95-2xqzn      1/1     Running     0          4m
orion-console-6c4f7d8b95-lk8vt      1/1     Running     0          4m
orion-core-7d9b5c4f68-4gwpm         1/1     Running     0          4m
orion-core-7d9b5c4f68-9tzkr         1/1     Running     0          4m
orion-core-7d9b5c4f68-hb2ql         1/1     Running     0          4m
orion-gateway-5f8c6d7b44-8mzvd      1/1     Running     0          4m
orion-gateway-5f8c6d7b44-c4rnp      1/1     Running     0          4m
orion-gateway-5f8c6d7b44-w7klx      1/1     Running     0          4m
orion-ledger-84b7f9c6d5-jq3xv       1/1     Running     0          4m
orion-ledger-84b7f9c6d5-r6ptn       1/1     Running     0          4m
orion-migrate-4-2-1-tq8lz           0/1     Completed   0          6m
orion-scheduler-59d8c7f4b6-kx9dm    1/1     Running     0          4m
orion-scheduler-59d8c7f4b6-p2vhs    1/1     Running     0          4m
orion-worker-6b5d94c7f8-2ndqr       1/1     Running     0          4m
orion-worker-6b5d94c7f8-4jwmt       1/1     Running     0          4m
orion-worker-6b5d94c7f8-8xchl       1/1     Running     0          4m
orion-worker-6b5d94c7f8-gt7bz       1/1     Running     0          4m
orion-worker-6b5d94c7f8-mq5vw       1/1     Running     0          4m
orion-worker-6b5d94c7f8-zk3pf       1/1     Running     0          4m
```

Checking health from inside the cluster:

```bash
kubectl -n orion run healthz --rm -it --restart=Never \
  --image=curlimages/curl:8.8.0 -- \
  curl -s http://orion-gateway:8080/healthz
```

## Known pitfalls

**PVCs survive an uninstall.** `helm uninstall orion -n orion` does not delete the persistent volumes created for dependencies running inside the cluster. Check `kubectl get pvc -n orion` and delete them deliberately once the data is no longer needed.

**Migrations as a hook block the deployment.** With `migrations.asHook: true`, the migration job must succeed before the new pods start. If the job gets stuck, `helm upgrade` waits until `--timeout` expires and the release stays in the `pending-upgrade` state. Unblock it with `helm rollback orion -n orion` after removing the cause and reviewing the logs with `kubectl -n orion logs job/orion-migrate-4-2-1`.

**Resource limits cause processes to be killed.** Under a high order volume, `orion-core` exceeds the default 2 GiB memory limit and is killed with exit code 137. The symptom is a rising `orion_worker_retries_total` metric while traffic stays flat. Raise `core.resources.limits.memory` instead of adding replicas.

**An HPA does not work without the Metrics Server.** With `core.autoscaling.enabled: true` and no Metrics Server, the `HorizontalPodAutoscaler` object reports `unknown` in the `TARGETS` column and does not scale the deployment.

??? info "Installing from an archive file"
    In environments without internet access, download the chart and the images ahead of time:

    ```bash
    helm pull nimbus/orion --version 4.2.1 --destination ./charts
    helm upgrade --install orion ./charts/orion-4.2.1.tgz \
      --namespace orion --create-namespace --values values.yaml
    ```

## See also

- [Deploy on Kubernetes](index.md)
- [Install with the operator](operator.md)
- [Environment variables](../../configuration/environment-variables.md)
- [Monitoring](../../../operations/monitoring/index.md)
