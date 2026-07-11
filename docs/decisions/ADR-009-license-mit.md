# ADR-009: License — MIT

- Status: **Accepted** (owner decision, 2026-07-11)
- Deciders: Owner (Saber5656)

## Context

famchat is public OSS and simultaneously the basis of an operator-run
SaaS. License choice affects who can run competing hosted instances and
whether their modifications must be shared. The design agent proposed
AGPL-3.0 (network copyleft) and presented alternatives; the owner
decided.

## Decision

**MIT License** for the entire repository.

- Rationale (owner call): maximum simplicity and adoption; no barrier
  for families, tinkerers, or companies to use, embed, or host famchat;
  the project's trust posture rests on transparency and self-hostability
  (ADR-003), not on copyleft leverage.
- Header policy: repo-level `LICENSE` file only; no per-file license
  headers. `license: "MIT"` set in every package.json.
- Dependency policy: CI license checker (issue 57) allows
  MIT/Apache-2.0/BSD/ISC-class production dependencies; copyleft
  (GPL/AGPL/SSPL-class) production dependencies are rejected.

## Alternatives considered

| Option | Why not chosen |
|---|---|
| AGPL-3.0 (agent's proposal) | Strongest protection against silent hosted forks, but narrows contributor/adopter pool; owner prioritizes adoption and simplicity |
| Apache-2.0 | Patent grant is nice-to-have, but MIT's simplicity preferred at this project size |
| Source-available (BUSL-class) | Not open source; contradicts the project's framing |

## Consequences

- Anyone — including large platforms — may run closed, commercial hosted
  forks of famchat. Accepted knowingly.
- Contributions are MIT-licensed by default (inbound=outbound); no CLA.
- Issue 57 implements: `LICENSE` (MIT, owner as copyright holder),
  package.json fields, README licensing section, and the CI license
  checker with the policy above.
- Until issue 57 merges the LICENSE file, the public repository remains
  formally all-rights-reserved; this window is acceptable to the owner.
