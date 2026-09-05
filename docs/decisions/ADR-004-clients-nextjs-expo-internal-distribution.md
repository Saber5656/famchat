# ADR-004: Clients — Next.js web PWA + Expo React Native; internal distribution only in v1

- Status: Accepted (owner-confirmed 2026-07-11)
- Deciders: Owner (Saber5656), Fable (design agent)

## Context

The owner wants both web and native mobile in v1. Children's chat apps face
the strictest app-store review tracks (Apple Kids Category, Google Play
Families) requiring privacy labels, parental gates, moderation evidence, and
ongoing policy compliance — heavy, slow, and premature for a closed beta.

## Decision

- Web: Next.js App Router app, installable PWA (manifest + service worker +
  Web Push) so browser-only households still get an app-like experience.
- Mobile: one Expo (React Native) codebase for iOS/Android using expo-router,
  sharing the tRPC types, zod schemas, and i18n catalogs with web.
- v1 distribution stops at **EAS internal channels**: TestFlight (iOS) and
  Play internal testing (Android) for invited beta families. Public store
  publication — including Kids Category / Families program compliance — is a
  v2 gate with its own issue set.

## Alternatives considered

| Alternative | Why rejected |
|---|---|
| Store publication in v1 | Review risk and compliance workload dominate the schedule; beta needs neither |
| Flutter | Splits the codebase from the TS contract-sharing strategy (ADR-001) |
| Capacitor-wrapped web | Weak push/UX payoff; Apple minimum-functionality rejection risk if ever submitted |
| PWA only | Owner explicitly wants native mobile; iOS Web Push remains second-class (see research/platform-constraints.md) |

## Consequences

- TestFlight requires an Apple Developer account ($99/yr) and app records even
  for internal testing; Play internal requires a Play Console account ($25).
  Owner performs these registrations manually (agents never handle store
  credentials).
- Internal-distribution builds still deliver full push via APNs/FCM (Expo
  push service), unlike PWA-on-iOS.
- v2 store submission will likely force feature additions (e.g., in-app
  blocking mechanics, published moderation policy); tracked as deferred scope.
