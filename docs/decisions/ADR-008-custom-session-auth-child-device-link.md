# ADR-008: Custom DB-session auth with child QR device-link (no auth provider)

- Status: Accepted
- Deciders: Fable (design agent); conservative default per task instructions

## Context

Children 6–12 have no email address and cannot manage passwords, which breaks
almost every hosted auth provider's account model (and providers like Auth0/
Clerk conflict with self-hosting and with minors-data minimization). Lucia,
the popular DIY-auth library, is deprecated; its author now recommends
implementing sessions directly — a well-documented, small pattern.

## Decision

- Own the auth: argon2id password hashing (`@node-rs/argon2`), opaque
  256-bit session tokens stored hashed in a `sessions` table, cookie (web) /
  bearer (mobile) transport, rotation on login, revocation lists in DB.
- Adults: email + password (+ email password-reset). TOTP 2FA deferred to v2.
- Children: **device-link** flow — guardian generates a one-time 6-digit/QR
  code (10-min TTL); the child device redeems it for a long-lived (180-day
  sliding) revocable device session. Optional 4–6 digit PIN acts as a local
  app lock (sibling deterrence), explicitly not a security boundary; the
  guardian's device-revocation power is the boundary. Because the PIN is a
  deterrent rather than a boundary, offline behavior favors availability:
  the mobile app verifies the PIN server-side when online and accepts a
  cached successful verification for up to 24 h when offline (the
  "offline grace rule"); a revoked device stays revoked regardless.
- No OAuth/social login in any version: it would link children's accounts to
  external identity graphs, contradicting the closed-network principle.

## Alternatives considered

| Alternative | Why rejected |
|---|---|
| Hosted auth (Clerk/Auth0/Firebase) | Self-host breaks; child accounts unsupported; minors' data sent to third party |
| Keycloak/Zitadel self-hosted | Huge operational surface for a family-scale app |
| Lucia | Deprecated |
| Passkeys for children | Device/platform account entanglement children don't control; revisit v2 for guardians |
| Child passwords | Unusable at 6–9; leads to guardian-known passwords anyway — device-link is honest about the real trust model |

## Consequences

- We own session security (fixation, rotation, hashing, expiry) — covered by
  dedicated tests and the §19 checklist.
- Password reset requires SMTP in every deployment (Mailpit in dev).
- Device-link codes are credential-equivalent secrets: hashed at rest,
  short-TTL, one-time, rate-limited, and guardian-notified on use.
