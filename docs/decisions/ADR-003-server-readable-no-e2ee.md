# ADR-003: Server-readable content; no end-to-end encryption

- Status: Accepted (owner-confirmed 2026-07-11)
- Deciders: Owner (Saber5656), Fable (design agent)

## Context

famchat's core value is guardian oversight plus server-side safety features
(NG-word filtering on write, flag queues, future search). These require the
server to process plaintext. E2EE with guardian-key escrow was evaluated: it
would move filtering client-side (trivially bypassable by a modified client),
break server-composed push previews and future search, and add key-recovery
failure modes that non-technical families cannot handle. The operator-trust
problem this creates is real: on a SaaS instance the operator can technically
read family content.

## Decision

Content (messages, board, media) is stored server-readable. Compensating
controls are mandatory, not optional:

1. TLS 1.2+ for all transport; HSTS.
2. Encryption at rest at the volume/disk layer of the host (documented in
   selfhost guide) — application-layer field encryption is not used in v1.
3. No operator-facing content APIs exist (`/admin/v1` returns metadata only).
4. Emergency operator content access = direct DB access, governed by the
   Operator Data Access Policy: logged and disclosed to the affected space.
5. Full audit logging of privileged in-app access (guardian deletions, admin
   actions).
6. The Privacy Notice states plainly that the operator's infrastructure can
   read content and that guardians can read children's content.

E2EE is a **rejected** direction for famchat, not a deferred one: the product
premise (structural guardian oversight + server-side safety) is incompatible
with it. Revisiting requires a new ADR superseding this one.

## Alternatives considered

| Alternative | Why rejected |
|---|---|
| E2EE with guardian key escrow | Client-side-only filtering is bypassable; key loss bricks family history; complexity explodes v1 |
| Application-layer encryption with server-held keys | Theater against the operator threat (same key holder); real cost in every feature; deferred as not worth it for v1 |
| E2EE-ready schema constraints | Permanent complexity tax for a path we reject |

## Consequences

- Trust posture must be earned via transparency: OSS code, audit logs, policy
  docs, self-hosting as the ultimate opt-out. This is stated in README/docs.
- Push notifications may contain message previews (server composes them).
- A future breach of the host exposes content — backup encryption and host
  hardening guidance are part of the ops docs; this residual risk is accepted
  and disclosed.
