# ADR-006: Closed beta via operator-issued instance invites; compliance posture

- Status: Accepted (owner-confirmed 2026-07-11)
- Deciders: Owner (Saber5656), Fable (design agent)

## Context

The owner chose "multi-tenant SaaS in view" but v1 must stay operable by one
person. Public signup for a children's service immediately triggers heavy
obligations: abuse/impersonation programs, legal review (APPI/COPPA/GDPR),
support processes. Beta participants will be families personally known to the
operator.

## Decision

- Space creation requires an **instance invite code** issued by the operator
  (`pnpm ops instance-invite create`); there is no public signup endpoint at
  all in v1 — not merely disabled, but absent from the API surface.
- Everything else (space invites for adults, child device links) is
  guardian-driven within a space.
- No billing in v1.
- Compliance baseline: Japanese APPI, closed-circle beta; in-repo bilingual
  Terms/Privacy templates with explicit server-readability disclosure and
  guardian consent checkbox at signup. Public launch is gated behind a v2
  re-assessment (store policies, COPPA/GDPR, abuse program).

## Alternatives considered

| Alternative | Why rejected |
|---|---|
| Public free signup in v1 | Abuse/impersonation and legal workload dominate v1; contradicts one-person ops |
| Paid SaaS in v1 | Adds commerce law obligations (特定商取引法) and billing infra prematurely |
| Waitlist + manual approval | More moving parts than invite codes for identical closed-beta effect |

## Consequences

- Growth is intentionally capped; feedback loop is high-trust families.
- Abuse-report tooling can stay minimal (operator suspend + guardian tools).
- The invite gate must be removable in v2 without schema change (it is: it
  only guards `spaces.create`).
