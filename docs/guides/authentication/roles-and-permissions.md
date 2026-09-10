# Roles and permissions

Authorization in Orion Platform is based on tenant-scoped RBAC: permissions are grouped into roles, roles are assigned to subjects, and every subject acts in the context of exactly one tenant. The permission check happens in `orion-gateway` before the request is forwarded to `orion-core` or `orion-ledger`. Every change to assignments is recorded in the `audit` schema.

## Model

The hierarchy has three levels:

- **Tenant** (`ten_...`) - the boundary for data and configuration.
- **Subject** (`usr_...`, an OAuth2 client, an API key) - belongs to exactly one tenant.
- **Role** (`rol_...`) - a named set of permissions assigned to a subject.

Alongside the role there is a set of scopes carried in the token (`orders:write`, `admin`, ...).

The decision is a conjunction: a request passes only when the token carries the matching scope **and** the subject's role contains the matching permission.

## Built-in roles

| Role | Purpose | Notes |
| --- | --- | --- |
| `owner` | tenant owner, full control including billing | exactly one subject per tenant, cannot be deleted |
| `admin` | manages configuration, users, webhooks | cannot see the tenant's billing data |
| `operator` | day-to-day handling of orders and payments | can cancel and refund, does not change configuration |
| `support` | customer support, reads and puts orders on hold | no access to refunds |
| `integration` | role for OAuth2 clients and API keys | no console access |
| `viewer` | read only | default role for a new user |

## Permission matrix

Legend: **yes** - allowed, **no** - forbidden, **cond.** - conditional (see the footnotes below the table).

| Operation | `owner` | `admin` | `operator` | `support` | `integration` | `viewer` |
| --- | --- | --- | --- | --- | --- | --- |
| Read orders | yes | yes | yes | yes | yes | yes |
| Create an order | yes | yes | yes | no | yes | no |
| Change status to `paid` | yes | yes | yes | no | yes | no |
| Put on hold (`on_hold`) | yes | yes | yes | yes | cond.[^1] | no |
| Cancel an order | yes | yes | yes | cond.[^2] | cond.[^1] | no |
| Read payments | yes | yes | yes | yes | yes | yes |
| Authorize a payment | yes | no | yes | no | yes | no |
| Capture a payment | yes | no | yes | no | yes | no |
| Refund a payment | yes | no | yes | no | cond.[^1] | no |
| Read customers | yes | yes | yes | yes | yes | yes |
| Write customers | yes | yes | yes | cond.[^2] | yes | no |
| List webhooks | yes | yes | yes | no | yes | yes |
| Create a webhook | yes | yes | no | no | no | no |
| Rotate a webhook secret | yes | yes | no | no | no | no |
| Replay events from the DLQ | yes | yes | cond.[^3] | no | no | no |
| Read tenant configuration | yes | yes | yes | no | no | yes |
| Write tenant configuration | yes | yes | no | no | no | no |
| Manage API keys | yes | yes | no | no | no | no |
| Manage roles | yes | yes | no | no | no | no |
| Access billing | yes | no | no | no | no | no |

## Custom roles

You define a custom role as a list of permissions. The platform assigns the `rol_...` identifier.

```yaml title="role-refunds.yaml" hl_lines="4 8 9"
apiVersion: orion.nimbus.example.com/v1
kind: Role
metadata:
  name: refunds-level-2
  tenant: ten_01HQ8ZV3KXN4M2T9YB7C6D
spec:
  inherits: support
  permissions:
    - payments.refund
    - orders.hold
  constraints:
    payments.refund:
      max_amount_minor: 50000
      currency: PLN
```

The API response after creation:

```json title="201 Created"
{
  "id": "rol_01HQ8ZV3KXN4M2T9YB7C6D",
  "name": "refunds-level-2",
  "inherits": "support",
  "permissions": ["payments.refund", "orders.hold"],
  "tenant": "ten_01HQ8ZV3KXN4M2T9YB7C6D",
  "created_at": "2026-02-11T09:31:04Z"
}
```

!!! warning "Custom permissions do not widen scopes"
    The `refunds-level-2` role has no effect if the token does not carry the `payments:write` scope.
    The scope constrains the role, not the other way around.

## Assigning a role

=== "CLI"

    ```bash
    orion tenant create --id ten_01HQ8ZV3KXN4M2T9YB7C6D --name "Nordis" --profile prod
    orion config validate --file /etc/orion/orion.yaml
    ```

=== "API"

    ```bash
    curl -sS -X POST https://api.orion.example.com/v1/users/usr_01HQ8ZV3KXN4M2T9YB7C6D/roles \
      -H "Authorization: Bearer $TOKEN" \
      -H "X-Orion-Tenant: ten_01HQ8ZV3KXN4M2T9YB7C6D" \
      -H "Content-Type: application/json" \
      -H "Idempotency-Key: role-assign-2026-02-11-01" \
      -d '{"role_id": "rol_01HQ8ZV3KXN4M2T9YB7C6D"}'
    ```

=== "Response"

    ```json
    {
      "user": "usr_01HQ8ZV3KXN4M2T9YB7C6D",
      "roles": ["rol_01HQ8ZV3KXN4M2T9YB7C6D"],
      "effective_permissions": [
        "orders.read", "orders.hold", "payments.read", "payments.refund",
        "customers.read"
      ],
      "updated_at": "2026-02-11T09:33:12Z"
    }
    ```

## Inheritance and exceptions

- A custom role may inherit from exactly one built-in role (`inherits`); chains longer than one level are rejected with `ORN-1004`.
- Inherited permissions cannot be subtracted. To obtain a narrower set, inherit from `viewer` and add the permissions you need.
- A subject may hold several roles; the effective set is the union of their permissions.
- Constraints (`constraints`) from different roles are merged by taking the most permissive value. A role with `max_amount_minor: 50000` combined with a role without that constraint results in no constraint.
- The `owner` role ignores custom constraints.
- API keys and OAuth2 clients may only receive roles that inherit from `integration` or `viewer`.

??? info "Checking the effective set before rolling it out"
    The `GET /v1/users/{id}/permissions?simulate_role=rol_...` endpoint returns the set of permissions the
    subject would gain after the role is added, without applying the change. It is useful during access reviews.

## Auditing changes

Every operation on roles writes a row into the `audit.permission_changes` table.

| Column | Type | Description |
| --- | --- | --- |
| `id` | `text` | entry identifier |
| `tenant_id` | `text` | tenant |
| `actor` | `text` | `usr_...` or the `sub` of an OAuth2 client |
| `action` | `text` | `role.assign`, `role.revoke`, `role.create`, `role.update` |
| `subject` | `text` | the subject the change applies to |
| `before` / `after` | `jsonb` | the set of roles before and after |
| `request_id` | `text` | the `X-Orion-Request-Id` value |
| `created_at` | `timestamptz` | time of the change in UTC |

```sql title="Permission changes from the last 7 days"
SELECT created_at, actor, action, subject, after->'roles' AS roles
FROM audit.permission_changes
WHERE tenant_id = 'ten_01HQ8ZV3KXN4M2T9YB7C6D'
  AND created_at > now() - interval '7 days'
ORDER BY created_at DESC;
```

Entries in the `audit` schema are insert-only; the application role has no `UPDATE` or `DELETE` permissions. Retention is 24 months.

[^1]: The `integration` role performs the operation only when the token carries the matching scope (`orders:write` or `payments:write`) and the client has state-changing operations explicitly enabled.
[^2]: The `support` role may cancel an order in the `draft` or `pending_payment` status and update customer contact details; orders in `paid` and later statuses require the `operator` role.
[^3]: The `operator` role may replay events from the DLQ only for partitions assigned to its own tenant and only within a 72-hour window.

## See also

- [Authentication and authorization](index.md)
- [OAuth2 client credentials](oauth2.md)
- [Multi-tenancy](../multi-tenancy.md)
- [Security](../../operations/security.md)
