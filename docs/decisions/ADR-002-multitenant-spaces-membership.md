# ADR-002: Multi-tenant spaces with multi-space membership

- Status: Accepted (owner-confirmed 2026-07-11)
- Deciders: Owner (Saber5656), Fable (design agent)

## Context

The owner wants famchat to be SaaS-capable (many families on one instance)
even though v1 operates as an invite-gated closed beta. Real families overlap:
a grandparent belongs to two children's family spaces; shared-custody
arrangements exist. Retrofitting a 1-user-1-tenant model later would be a
near-rewrite of auth, notifications, and every query.

## Decision

Slack-style tenancy from day one:

- `spaces` = tenant; every content row carries `space_id`.
- `users` are global to the instance; `memberships` join users to spaces with
  a role (`guardian` / `adult` / `child`) and exactly one `is_owner` guardian
  per space.
- Adults may hold memberships in any number of spaces. Child users are
  restricted to exactly one space in v1 by application logic (not schema), so
  shared-custody multi-space children remain possible in v2 without migration.
- All API authorization resolves (user, space) → membership; there is no
  instance-wide user-facing query surface.

## Alternatives considered

| Alternative | Why rejected |
|---|---|
| 1 user = 1 space | Breaks the grandparent use case; forces duplicate accounts; migration later is destructive |
| Instance-per-family only (no tenancy) | Contradicts owner's SaaS goal; closed beta with several families on one VPS is the validation plan |
| Org→nested-groups tenancy | Over-modeled; families don't need sub-organizations |

## Consequences

- Every query must filter by `space_id` via membership checks — enforced by a
  shared tRPC middleware; integration tests must include cross-tenant denial
  cases (attempting to read another space's room must 403).
- UI needs a space switcher (kept minimal in v1).
- Notification fanout and WS channels are keyed per space/user, which is also
  what a future multi-instance deployment needs.
