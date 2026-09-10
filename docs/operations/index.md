# Operations

This section covers running Orion Platform 4.2 LTS in the `staging` and `prod` environments. It collects procedures for monitoring, backups, scaling, security hardening and incident runbooks. The material is intended for SRE teams and administrators maintaining installations on Kubernetes or on machines provisioned from system packages.

## Who this section is for

The operations documents assume access to the `orion` namespace, Grafana credentials (`https://grafana.internal.example.com`) and permissions on the `orion` database in PostgreSQL 15. If you are looking for a description of what the product does, start with [what Orion is](../introduction/what-is-orion.md) and the [architecture](../introduction/architecture.md).

## Operational areas

| Area | Scope | Document |
| --- | --- | --- |
| Monitoring and observability | Prometheus metrics, JSON logs, OTLP traces, dashboards | [operations/monitoring](monitoring/index.md) |
| Metrics | Naming conventions, PromQL queries, recording rules | [monitoring/metrics](monitoring/metrics.md) |
| Logs | Format, `request_id` correlation, masking, retention | [monitoring/logs](monitoring/logs.md) |
| Alerts | Prometheus rules, routing, silences | [monitoring/alerts](monitoring/alerts.md) |
| Backups | Schedule, PITR, restore tests | [backups](backups.md) |
| Scaling | Deployment sizes, HPA, connection pools | [scaling](scaling.md) |
| Security | TLS, secrets, hardening, audit, compliance | [security](security.md) |
| Runbooks | Alert response procedures | [runbooks](runbooks/index.md) |

## Service level objectives

The following values are binding for production installations and form the basis for alert thresholds.

| Indicator | Objective | Measurement window |
| --- | --- | --- |
| API availability | 99.9% | calendar month |
| p95 response time for `GET` | < 250 ms | 5 min |
| p95 response time for `POST /orders` | < 600 ms | 5 min |
| Webhook delivery | < 60 s in 99% of cases | 24 h |
| Event queue lag | < 1000 events | 5 min |
| Database RPO | 5 min | — |
| Service RTO | 60 min | — |

!!! note "Error budget"
    Availability of 99.9% means roughly 43 minutes of downtime per month. Consuming 50% of the budget freezes deployments of non-critical changes until the end of the accounting period.

## Team working rhythm

The "Platform Core" team keeps a fixed schedule of operational activities.

Weekly review
:   Monday 10:00 UTC. Error budget analysis, review of alerts without a runbook, inspection of the `orion.events.v1.dlq` queue.

Maintenance window
:   Tuesday and Thursday 22:00-00:00 UTC. Deployments of the `nimbus/orion` Helm chart, database migrations, certificate rotation.

SRE on-call
:   Weekly shift, handover on Monday 09:00 UTC in the `#orion-oncall` channel. The on-call engineer responds to P1 within 30 minutes.

Backup restore test
:   First business day of the quarter, following the task list in [backups](backups.md).

!!! tip "Customer reports"
    Incidents reported by customers arrive through the `https://support.nimbus.example.com` portal. Public status is communicated on `https://status.orion.example.com`, see [support](../support.md).

## See also

- [Runbooks](runbooks/index.md)
- [Monitoring and observability](monitoring/index.md)
- [Environment requirements](../getting-started/requirements.md)
- [Support](../support.md)
