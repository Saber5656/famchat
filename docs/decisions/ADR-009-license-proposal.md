# ADR-009: License — AGPL-3.0 proposed

- Status: **Proposed — requires owner decision before issue 57 (OSS release prep) executes**
- Deciders: Owner (Saber5656) — decision pending

## Context

famchat is intended to be public OSS and simultaneously the basis of an
operator-run SaaS. License choice affects who can run competing hosted
instances and whether their modifications must be shared. This is a legal/
strategic call the design agent must not make unilaterally.

## Proposal

**AGPL-3.0-only** for the application. Rationale: network-use copyleft means
anyone offering famchat as a service must publish their modifications —
protective for a small OSS author against silent hosted forks, and aligned
with the transparency posture of ADR-003 (users can verify what a hosted
instance runs only if operators must publish changes).

## Options for the owner

| Option | Effect | Trade-off |
|---|---|---|
| A: AGPL-3.0 (proposed) | Hosted forks must open their changes | Some companies avoid AGPL; contributor pool slightly narrower |
| B: Apache-2.0 | Maximum adoption, patent grant | Anyone (incl. big platforms) may run closed hosted forks |
| C: MIT | Simplest, maximum permissive | Same as B, no patent grant |
| D: Elastic/BUSL-style source-available | Blocks competing SaaS entirely | Not OSI open source; conflicts with "open-source project" framing |

## Consequences

- `LICENSE` file, SPDX headers policy, and README licensing section are part
  of issue 54 and blocked on this decision.
- Until decided, the repository has **no license**, i.e. all-rights-reserved
  by default — acceptable while private, must be resolved before publication.
- Dependency licenses (all MIT/Apache/BSD-class in the chosen stack) are
  compatible with any option above; CI adds a license checker in issue 54.
