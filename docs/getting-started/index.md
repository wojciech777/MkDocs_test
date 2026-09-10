# Getting started

This section takes you from an empty environment to your first paid order in Orion Platform 4.2 LTS. Follow the steps in the given order, because each one assumes the previous one succeeded. The whole path is identical for the Business and Enterprise tiers; the differences between installation methods are described on the [Installation](installation/index.md) page.

## Deployment path

- [ ] **Step 1. Check the requirements**: resources, dependency versions, open ports and PostgreSQL permissions. See [Requirements](requirements.md). *Estimated time: 30 min.*
- [ ] **Step 2. Choose an installation method and start the components**: Docker Compose, Helm, the operator or system packages. See [Installation](installation/index.md). *Estimated time: 20 min to 2 h.*
- [ ] **Step 3. Configure the instance**: the `orion.yaml` file, the `ORION_*` variables and the choice of profile. See [Configuration](configuration/index.md). *Estimated time: 45 min.*
- [ ] **Step 4. Set up authentication**: an OAuth2 client in the `orion` realm or an `ok_live_...` API key. See [Authentication](../guides/authentication/index.md). *Estimated time: 30 min.*
- [ ] **Step 5. Run your first order end to end**: customer, order, payment, webhook. See [Your first order](first-order.md). *Estimated time: 20 min.*

Total time with a healthy environment and dependencies already in place: **2.5 to 4 hours**. A production deployment including monitoring and backup configuration usually takes 2 to 3 business days.

## What you get once the path is complete

| Item | State after step 5 |
| --- | --- |
| The `orion-gateway`, `orion-core`, `orion-ledger`, `orion-worker`, `orion-scheduler`, `orion-console` components | running and healthy |
| The `core`, `ledger`, `audit` schemas in the `orion` database | migrated to version 4.2.1 |
| The `orion.events.v1` topic and the `orion.events.v1.dlq` DLQ | created and consumed by `orion-worker` |
| One `ord_...` order | in status `paid` |
| One `whk_...` webhook delivery | acknowledged with code 200 |

!!! tip "Start with the sandbox if you are only integrating"
    If your only goal is an API integration, skip steps 1 to 3. The `https://api.sandbox.orion.example.com/v1` environment is ready to use with an `ok_test_...` key, has a limit of 60 requests per minute and requires no installation of your own. Go straight to [Your first order](first-order.md).

!!! warning "Do not start with the prod profile"
    Run the first start-up with the `dev` or `staging` profile. The `prod` profile enforces TLS, disables automatic migrations and blocks the payment sandbox, which makes diagnosing the first configuration errors harder. See [Configuration profiles](configuration/profiles.md).

## Pages in this section

- [Requirements](requirements.md)
- [Installation](installation/index.md)
- [Configuration](configuration/index.md)
- [Your first order](first-order.md)

## See also

- [Architecture](../introduction/architecture.md)
- [Monitoring](../operations/monitoring/index.md)
- [Frequently asked questions](../faq.md)
