# Introduction

This section explains what Orion Platform 4.2 LTS is, how the platform is built and what vocabulary we use to describe its parts. Read it before your first deployment, because the concepts introduced here come back in every later part of the documentation. The material is independent of the installation method and the licence tier.

## Who you are, where to start

| Role | Your goal | Start with |
| --- | --- | --- |
| Integrator / developer | Connect a shop to the orders and payments API | [Your first order](../getting-started/first-order.md), [REST API](../api/rest/index.md) |
| System administrator | Deploy and configure an instance | [Requirements](../getting-started/requirements.md), [Installation](../getting-started/installation/index.md) |
| SRE / operations team | Monitor, scale and respond to incidents | [Monitoring](../operations/monitoring/index.md), [Runbooks](../operations/runbooks/index.md) |
| Product owner | Understand the functional scope and the release lifecycle | [What is Orion Platform](what-is-orion.md), [Changelog](../changelog.md) |
| Architect | Evaluate the deployment model and system boundaries | [Architecture](architecture.md), [Multi-tenancy](../guides/multi-tenancy.md) |
| Security team | Review the authentication and permission model | [Authentication](../guides/authentication/index.md), [Security](../operations/security.md) |

## Pages in this section

[What is Orion Platform](what-is-orion.md)
:   Functional scope, typical use cases, licence tiers and the release lifecycle of 3.8, 4.0 and 4.2 LTS.

[Architecture](architecture.md)
:   Components, ports, the request flow for creating an order, the data model and architecture decisions ADR-001 to ADR-003.

[Glossary](glossary.md)
:   Definitions of the domain and technical terms used throughout the documentation and in API messages.

## Before you continue

!!! tip "Try the sandbox before you deploy"
    The `https://api.sandbox.orion.example.com/v1` environment exposes the full set of endpoints with a limit of 60 requests per minute. It requires no installation of your own and no licence agreement, just a test key `ok_test_...`.

!!! note "Version covered by this documentation"
    The whole documentation set covers the **4.2 LTS** line (latest patch release **4.2.1**, supported until **2027-06-30**). Passages specific to earlier releases are clearly marked.

## See also

- [What is Orion Platform](what-is-orion.md)
- [Getting started](../getting-started/index.md)
- [Frequently asked questions](../faq.md)
- [Support](../support.md)
