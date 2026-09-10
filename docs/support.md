# Support

The Nimbus Software support team handles Orion Platform requests for the
Business and Enterprise editions. This page describes the contact channels,
the response times, and the information that lets us resolve a problem on the
first pass.

## Contact channels

| Channel | Address | Use for | Availability |
| --- | --- | --- | --- |
| Support portal | <https://support.nimbus.example.com> | All requests, full correspondence history | 24/7 |
| Email | <support@nimbus.example.com> | P2 to P4 requests, commercial questions | business days 08:00-18:00 |
| On-call phone | number from your support contract | P1 requests only | 24/7 (Enterprise) |
| Status page | <https://status.orion.example.com> | Managed service outages, maintenance windows | 24/7 |

!!! tip "Before you open a request"

    Check the [FAQ](faq.md), the [error codes](api/rest/error-codes.md), and the
    status page. Roughly a third of all requests describe situations already
    covered by the [runbooks](operations/runbooks/index.md).

## Severity and response times

| Severity | Definition | Response time | Working mode |
| --- | --- | --- | --- |
| P1 | Complete API outage or data loss in production | 30 minutes | continuous until a workaround is in place |
| P2 | Serious degradation: SLO breach, webhooks not delivered | 4 hours | business days, prioritized |
| P3 | Single feature broken with a workaround | 1 business day | scheduled into the next release |
| P4 | Question, documentation, cosmetic defect | 3 business days | standard queue |

Response time is the time to the first substantive reply from an engineer, not
the time to resolution. The requester sets the initial severity, and the support
team may adjust it after triage.

## How to report a problem

1. Determine whether the problem affects production, staging, or the sandbox.
2. Collect the request id (`X-Orion-Request-Id` or `error.request_id`) for at
   least one failed call.
3. Note the exact time of the first occurrence in UTC and the affected time
   range.
4. Generate a diagnostic bundle:

    ```bash
    orion diag bundle --since 2h --output /tmp/orion-diag.tar.gz
    ```

5. Open a request in the portal, attach the bundle, and describe both the
   observed and the expected behavior.

!!! danger "Never send secrets"

    The diagnostic bundle masks the `Authorization` and `X-Orion-Key` headers,
    but always review the attachments before sending them. Never transmit
    `ok_live_...` keys, `whsec_...` webhook secrets, or database passwords. If a
    credential is exposed, revoke it immediately. See
    [API keys](guides/authentication/api-keys.md).

## Request checklist

- [ ] Platform version (`orion version` or the `orion_build_info` metric)
- [ ] Deployment method: Helm, operator, Docker Compose, system packages
- [ ] Environment: production, staging, or sandbox
- [ ] Request id and the `ORN-*` error code
- [ ] Time range in UTC and the scale of the problem (share of requests, number of tenants)
- [ ] Recent changes: deployment, migration, configuration change
- [ ] Dashboard screenshots or PromQL queries that confirm the symptom
- [ ] Diagnostic bundle

## Scope of support

**Covered**

- Releases within their support window: 4.2 LTS (until 2027-06-30) and 4.0
  (security fixes until 2026-09-30).
- Installations that meet the [Requirements](getting-started/requirements.md).
- The official client libraries for Java and Python.

**Not covered**

- Releases past end of life and builds with modified sources.
- Community libraries and clients generated from OpenAPI.
- Deployments running the `dev` profile in production.
- Third-party systems such as payment gateways, WMS, and ERP. We help diagnose
  the integration, but we do not own the behavior of those systems.

## Escalation

If a P1 request has no reply within 30 minutes, or the agreed action plan stalls,
use the **Escalate** button in the portal. Escalation notifies the on-call
support manager. Enterprise customers also have the shared chat channel agreed
during onboarding.

## See also

- [FAQ](faq.md)
- [Runbooks](operations/runbooks/index.md)
- [Error codes](api/rest/error-codes.md)
- [Changelog](changelog.md)
