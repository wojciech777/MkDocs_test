# Operational security

This page describes the security requirements for production installations of Orion Platform 4.2 LTS: traffic protection, secrets management, container hardening, network policies, audit, and the obligations arising from GDPR and PCI DSS. The requirements are mandatory in the `staging` and `prod` profiles; in `dev`, the explicitly noted exceptions are allowed. Authentication of API clients is covered separately in the [authentication section](../guides/authentication/index.md).

## Threat model in brief

| Asset | Threat | Control |
| --- | --- | --- |
| Order data in `core` | Cross-tenant access | `tenant` filter at the query level + RLS policies |
| Payment ledger `ledger` | Modification of settlement history | Append-only tables, log in `audit` |
| API keys (`X-Orion-Key`) | Leak from logs or a repository | Masking in logs, storing hashes, rotation |
| Keycloak OAuth2 tokens | Theft and session replay | Short lifetime, `aud` validation, TLS |
| `orion.events.v1` topic | Unauthorized reading of events | SASL/SCRAM + ACLs per consumer group |
| `orion-artifacts` bucket | Public exposure of files | Public access block, signed URLs |
| `/metrics` endpoints (9100) | Disclosure of topology and volumes | Reachable only from the monitoring network |
| Backups in S3 | Backup read by an unauthorized person | SSE-KMS, separate write and read roles |
| `orion-console` panel | Administrator session hijacking | MFA in the `orion` realm, short sessions |

## TLS and certificates

All external traffic and all internal traffic between components is encrypted. Certificates are issued by cert-manager, and renewal happens automatically 30 days before expiry.

| Endpoint | Name | Issuer | Minimum TLS version | Rotation |
| --- | --- | --- | --- | --- |
| API | `api.orion.example.com` | public CA (ACME) | TLS 1.2 | 90 days, automatic |
| Console | `console.orion.example.com` | public CA (ACME) | TLS 1.2 | 90 days, automatic |
| Internal traffic | `*.orion.svc.cluster.local` | internal CA | TLS 1.3 | 30 days, automatic |
| PostgreSQL | `pg-primary.internal.example.com` | internal CA | TLS 1.2 (`verify-full`) | 365 days, manual |
| Kafka (SASL_SSL) | brokers in `ORION_KAFKA_BROKERS` | internal CA | TLS 1.2 | 365 days, manual |

The only permitted ciphers are AEAD suites (`TLS_AES_128_GCM_SHA256`, `TLS_AES_256_GCM_SHA384`, `ECDHE-RSA-AES128-GCM-SHA256`). TLS 1.0 and 1.1 are disabled and cannot be enabled in the `prod` profile.

```bash
kubectl get certificate -n orion
```

```text
NAME                  READY   SECRET                 AGE
orion-api-tls         True    orion-api-tls          63d
orion-console-tls     True    orion-console-tls      63d
orion-internal-tls    True    orion-internal-tls     12d
```

The `OrionCertExpiring` alert with its 14-day threshold is described on the [alerts page](monitoring/alerts.md).

## Secrets management

!!! danger "No secret in an image or in a repository"
    The `registry.nimbus.example.com/orion/*:4.2.1` images contain no credentials. Putting a password into `values.yaml`, `orion.yaml` or an `ORION_DB_URL` variable in a manifest is a policy violation and requires immediate rotation of the exposed secret, regardless of whether the repository is private.

Secrets are delivered from Vault or by the ExternalSecrets controller. Every `ORION_*` variable that accepts a confidential value has a counterpart with the `_FILE` suffix pointing at the path of a mounted file; the process reads it at startup and never writes the value into the environment.

```yaml title="externalsecret-orion.yaml" hl_lines="14 15 19"
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: orion-secrets
  namespace: orion
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: orion-secrets
  data:
    - secretKey: db_url
      remoteRef: { key: orion/prod/postgres, property: url }
    - secretKey: kafka_password
      remoteRef: { key: orion/prod/kafka, property: password }
    - secretKey: webhook_signing_key
      remoteRef: { key: orion/prod/webhooks, property: signing_key }
```

The reference in a component's configuration:

```yaml title="values.yaml"
env:
  ORION_DB_URL_FILE: /etc/orion/secrets/db_url
  ORION_KAFKA_PASSWORD_FILE: /etc/orion/secrets/kafka_password
  ORION_WEBHOOK_SIGNING_KEY_FILE: /etc/orion/secrets/webhook_signing_key
```

Configuration validation checks that the bindings are correct:

```bash
orion config validate --profile prod --strict
```

```text
/etc/orion/orion.yaml            ok
secret refs                      6/6 resolved (0 inline values)
tls minimum version              1.2  ok
metrics exposure                 internal-only  ok
result                           PASS
```

## Credential rotation

| What | How often | Owner | Procedure |
| --- | --- | --- | --- |
| Database user passwords | 90 days | SRE (Marek Debski) | Create a new password in Vault, reload the secret, rolling restart |
| Tenant API keys | 180 days or on request | Tenant, in the console | Issue a second key, migrate, revoke the old one |
| Webhook signing key | 180 days | SRE | 48 h overlap period with two active keys |
| Kafka SASL credentials | 180 days | SRE | Rotation per consumer group, with no consumption gap |
| Backup KMS key | 12 months | SRE | Automatic rotation, older versions retained |
| OAuth2 client secret | 180 days | "Platform Core" team | Rotation in the `orion` realm |
| Internal certificates | 30 days | cert-manager | Automatic |

!!! tip "Key overlap period"
    Always rotate API keys and the webhook signing key with an overlap period. During that time signature verification accepts two values, see [signature verification](../api/webhooks/signature-verification.md).

## Container hardening

All images run as the unprivileged user `10001`, with a read-only filesystem and no kernel capabilities.

```yaml title="values.yaml (securityContext excerpt)"
podSecurityContext:
  runAsNonRoot: true
  runAsUser: 10001
  runAsGroup: 10001
  fsGroup: 10001
  seccompProfile:
    type: RuntimeDefault
securityContext:
  allowPrivilegeEscalation: false
  privileged: false
  readOnlyRootFilesystem: true
  capabilities:
    drop: ["ALL"]
volumes:
  - name: tmp
    emptyDir: { sizeLimit: 256Mi }
volumeMounts:
  - name: tmp
    mountPath: /tmp
```

Additional requirements:

- The `orion` namespace carries the `pod-security.kubernetes.io/enforce: restricted` label.
- No `hostNetwork`, `hostPID`, `hostPath` mounts or Docker socket.
- Resource limits defined for every container; a missing limit blocks the deployment.
- Images pinned by digest, not by a mutable tag.

## Network policies

By default all traffic in the `orion` namespace is denied, and only the required directions are opened.

```yaml title="networkpolicy-orion-core.yaml"
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: orion-core
  namespace: orion
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: orion-core
  policyTypes: [Ingress, Egress]
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app.kubernetes.io/name: orion-gateway
      ports:
        - { protocol: TCP, port: 8081 }
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - { protocol: TCP, port: 9100 }
  egress:
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: orion-data
      ports:
        - { protocol: TCP, port: 5432 }
        - { protocol: TCP, port: 6379 }
        - { protocol: TCP, port: 9093 }
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - { protocol: UDP, port: 53 }
```

## Audit

Every operation that changes data or configuration writes an entry into the `audit` schema and publishes an event on the `orion.audit.v1` topic. Entries are retained for 400 days, and the `audit.events` table is partitioned monthly and append-only.

```sql
SELECT created_at, actor, actor_type, action, entity, entity_id, request_id, ip
FROM audit.events
WHERE tenant = 'acme-retail'
  AND action IN ('order.cancel', 'payment.refund', 'apikey.revoke')
  AND created_at >= now() - interval '7 days'
ORDER BY created_at DESC
LIMIT 50;
```

Audit records include `request_id`, which lets you join an entry with application logs following the rules described in [logs](monitoring/logs.md).

??? info "Who can read the audit log"
    The `orion_auditor` role has `SELECT` on `audit.events` and no rights on `core` or `ledger`. The application role `orion_app` has `INSERT` without `UPDATE` or `DELETE`, which makes it impossible to modify history from the application.

## Compliance

GDPR
:   Customer personal data is stored in `core.customers`. A deletion request is fulfilled by anonymization: the first name, last name, address and email are overwritten with pseudonymous values, while the identifier and the settlement history are preserved. The default retention for closed orders is 5 years, after which records are anonymized by the `gdpr-anonymize` job. The record of processing activities is maintained by Anna Kowalska.

PCI DSS
:   Orion does not store card numbers. `orion-ledger` operates exclusively on payment provider tokens.

!!! danger "Storing a PAN is forbidden"
    The full card number (PAN), the CVV code and magnetic stripe data must never be written to the database, the logs, the queue or an `orion diag bundle`. Finding such data is a P1 security incident: invalidate the affected data, notify the payment provider and report the case to `support@nimbus.example.com` within 24 h.

## Image and vulnerability scanning

Images are scanned at build time and daily in the registry. A deployment carrying a critical vulnerability with an available fix is blocked.

```bash
trivy image --severity HIGH,CRITICAL --ignore-unfixed \
  registry.nimbus.example.com/orion/orion-gateway:4.2.1
```

| Severity | Patch SLA | Path |
| --- | --- | --- |
| Critical | 72 h | Out-of-band patch release |
| High | 14 days | Next patch release |
| Medium | 90 days | Scheduled release |
| Low | next LTS release | Scheduled release |

The history of patch releases is kept in the [changelog](../changelog.md).

## Quarterly review checklist

- [ ] Verify that `orion config validate --strict` reports no confidential values embedded in the configuration.
- [ ] Review the roles and permissions in the `orion` realm, and remove accounts inactive for more than 90 days.
- [ ] Confirm that database passwords, webhook signing keys and Kafka SASL credentials were rotated on time.
- [ ] Check that every `NetworkPolicy` in the `orion` namespace defaults to deny.
- [ ] Scan the 4.2.1 images and confirm there are no open critical or high vulnerabilities past their SLA.
- [ ] Verify the access log for the `alias/orion-backup` KMS key.
- [ ] Check that `audit.events` records are complete for `payment.refund` operations over the last quarter.
- [ ] Confirm that the `/metrics` endpoints are not reachable from the public network.
- [ ] Review the list of active tenant API keys and revoke those unused for more than 180 days.
- [ ] Update the threat model with the architectural changes from the last quarter.

## See also

- [Backup and restore](backups.md)
- [Roles and permissions](../guides/authentication/roles-and-permissions.md)
- [Alerts](monitoring/alerts.md)
- [Operations](index.md)
