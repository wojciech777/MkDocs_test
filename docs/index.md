# Orion Platform 4.2 LTS

Orion Platform handles orders, payments, and events for online stores and
marketplaces. This documentation covers release **4.2 LTS** (latest patch
release **4.2.1**) and spans installation, configuration, API integration, and
day-to-day operations in production.

!!! info "Sample content"

    Everything in this documentation set, including the product, the company,
    the hostnames, the versions, and the error codes, is **fictional**. The set
    exists as a multi-level example project built with MkDocs and the Material
    theme.

## Where to start

<div class="grid cards" markdown>

-   :material-rocket-launch-outline:{ .lg .middle } __Getting started__

    ---

    Requirements, installation with Docker or Kubernetes, configuration, and
    your first order created through the API in under an hour.

    [:octicons-arrow-right-24: Run Orion](getting-started/index.md)

-   :material-book-open-variant:{ .lg .middle } __Introduction__

    ---

    What Orion is, how the platform is built, which component owns which
    responsibility, and what the terms in this documentation mean.

    [:octicons-arrow-right-24: Learn the platform](introduction/index.md)

-   :material-api:{ .lg .middle } __API__

    ---

    REST API reference, webhooks including signature verification, and the
    official client libraries for Java and Python.

    [:octicons-arrow-right-24: Integrate](api/index.md)

-   :material-server-network:{ .lg .middle } __Operations__

    ---

    Monitoring, alerts, backups, scaling, security, and runbooks for on-call
    engineers.

    [:octicons-arrow-right-24: Run production](operations/index.md)

</div>

## What's in the docs

| Section | Contents | Audience |
| --- | --- | --- |
| [Introduction](introduction/index.md) | Product scope, architecture, glossary | Everyone |
| [Getting started](getting-started/index.md) | Requirements, installation, configuration, tutorial | Administrators, integrators |
| [Guides](guides/index.md) | Authentication, events, migrations, multi-tenancy | Integrators, architects |
| [API](api/index.md) | REST, webhooks, SDKs, error codes | Integration developers |
| [Operations](operations/index.md) | Observability, backups, runbooks | SRE, administrators |
| [Help](faq.md) | FAQ, changelog, contact | Everyone |

## Highlights in 4.2

- Cursor pagination is the default for every collection. See
  [Pagination, filtering, and sorting](api/rest/pagination.md).
- New `v2` webhook signature format (HMAC-SHA256); the `v1` signature is
  deprecated. See [Signature verification](api/webhooks/signature-verification.md).
- Kubernetes operator with the `OrionCluster` resource in version `v1alpha2`.
  See [Install with the operator](getting-started/installation/kubernetes/operator.md).
- Tenant isolation backed by RLS policies in PostgreSQL. See
  [Multi-tenancy](guides/multi-tenancy.md).

The [changelog](changelog.md) lists every change, including breaking ones. If
you are upgrading from 3.8 LTS, work through the
[migration guide](guides/migrations/from-3-8-to-4-2.md).

## Releases and support

| Release | Status | Supported until |
| --- | --- | --- |
| 4.2 LTS | Recommended | 2027-06-30 |
| 4.0 | Security fixes only | 2026-09-30 |
| 3.8 LTS | End of life | 2026-03-31 |

!!! tip "Need help?"

    Check the [FAQ](faq.md) first. If that does not resolve the problem, get in
    touch. The [Support](support.md) page lists the contact channels and
    response times.
