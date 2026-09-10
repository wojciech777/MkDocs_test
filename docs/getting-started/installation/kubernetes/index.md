# Deploy on Kubernetes

Kubernetes is the recommended target environment for `staging` and `prod` installations of Orion Platform 4.2 LTS. Two variants are available: a Helm chart and an operator that manages the `OrionCluster` resource. Both install identical images in version 4.2.1 and differ only in how the instance lifecycle is managed.

## Cluster requirements

| Element | Requirement |
| --- | --- |
| Kubernetes version | 1.28 - 1.31 (1.30 recommended) |
| StorageClass | default class with `volumeBindingMode: WaitForFirstConsumer`, support for `ReadWriteOnce` |
| Ingress controller | NGINX Ingress 1.10+ or a controller compatible with Gateway API 1.1 |
| cert-manager | 1.14+ with a `ClusterIssuer` for the `*.orion.example.com` domains |
| Metrics Server | required for `HorizontalPodAutoscaler` |
| Prometheus Operator | optional, required for `ServiceMonitor` objects |
| Registry access | `imagePullSecret` for `registry.nimbus.example.com` |
| Installer permissions | `cluster-admin` for the operator, namespace-scoped permissions for Helm |

## Installation variants

| Feature | [Helm](helm.md) | [Operator](operator.md) |
| --- | --- | --- |
| License variant | Community, Business, Enterprise | Enterprise only |
| Management model | imperative Helm releases | declarative `OrionCluster` resource |
| Database migrations | `pre-upgrade` hook as part of `helm upgrade` | `Migrating` phase in the reconciliation loop |
| Self-healing after a manual change | none, until the next `helm upgrade` | yes, on every reconciliation |
| Backups in the lifecycle | configured separately | built into `spec.backup` |
| Version upgrade | `helm upgrade --install` | change `spec.version` |
| Rolling back a change | `helm rollback` | edit the resource, no automatic rollback of migrations |
| Multiple instances in a cluster | a separate release per namespace | many resources served by a single operator |
| GitOps integration | natural (the chart is an artifact) | natural (the resource is a manifest) |
| Deployment complexity | low | medium |

!!! tip "Which variant to choose"
    If you run one or two instances and already have Helm-based processes, choose [Helm](helm.md). If you manage many instances for different tenants and expect automatic migrations and backups, choose the [operator](operator.md).

## Recommended namespace topology

| Namespace | Contents | Notes |
| --- | --- | --- |
| `orion` | application components, `Service`, `Ingress`, application secrets | the main namespace of the instance |
| `orion-system` | the operator, its `ServiceAccount` and `ClusterRole`, the CRD validating webhook | only for the operator variant |
| `orion-data` | PostgreSQL, Redis, Kafka, MinIO, if they run inside the cluster | separate resource limits and network policies |

```text
cluster
├── orion-system     OrionCluster operator, CRD
├── orion            gateway, core, ledger, worker, scheduler, console
└── orion-data       postgres, redis, kafka, minio
```

!!! note "Dependencies outside the cluster"
    If PostgreSQL, Kafka or Redis are managed services from a cloud provider, the `orion-data` namespace is not needed. Point `ORION_DB_URL` and `ORION_KAFKA_BROKERS` at the service addresses and add a `NetworkPolicy` rule that allows egress traffic from the `orion` namespace.

## Required secrets

Create them in the `orion` namespace before installation; both variants expect the same names.

| Secret | Keys | Purpose |
| --- | --- | --- |
| `orion-db` | `url`, `username`, `password`, `migrator-username`, `migrator-password` | access to the `orion` database |
| `orion-redis` | `url`, `password` | cache and rate limit counter |
| `orion-kafka` | `brokers`, `sasl-username`, `sasl-password` | event bus |
| `orion-s3` | `endpoint`, `bucket`, `access-key`, `secret-key` | the `orion-artifacts` bucket |
| `orion-oidc` | `issuer`, `client-id`, `client-secret` | the `orion` realm in Keycloak |
| `orion-signing` | `webhook-secret`, `encryption-key` | the `X-Orion-Signature` signature, encryption of sensitive data |
| `orion-registry` | `.dockerconfigjson` | pulling images from `registry.nimbus.example.com` |

```bash
kubectl create namespace orion

kubectl -n orion create secret docker-registry orion-registry \
  --docker-server=registry.nimbus.example.com \
  --docker-username='deploy' \
  --docker-password="$REGISTRY_TOKEN"

kubectl -n orion create secret generic orion-db \
  --from-literal=url='postgres://postgres.orion-data:5432/orion' \
  --from-literal=username='orion_app' \
  --from-literal=password="$DB_PASSWORD" \
  --from-literal=migrator-username='orion_migrator' \
  --from-literal=migrator-password="$DB_MIGRATOR_PASSWORD"
```

!!! danger "Secrets do not belong in chart values"
    Do not put passwords in `values.yaml` or in the `OrionCluster` manifest. Both variants accept only references to existing `Secret` objects. If you use GitOps, use the External Secrets Operator or Sealed Secrets.

## See also

- [Install with Helm](helm.md)
- [Install with the operator](operator.md)
- [Requirements](../../requirements.md)
- [Scaling](../../../operations/scaling.md)
