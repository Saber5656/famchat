# Issue 48: EAS internal distribution (TestFlight / Play internal)

## Summary

Make the mobile app installable by beta families: EAS build profiles,
app identifiers and store records, push credentials, universal/app-link
association, the build+submit runbook, versioning policy, and the
per-release smoke checklist.

## Context

ADR-004: v1 stops at internal distribution — TestFlight and Play
internal testing; public store review (Kids Category / Families) is v2.
Owner-manual steps (accounts, credentials) are explicitly separated from
agent-executable steps per the repo's security rules (agents never
handle store credentials).

## Scope

In scope: `eas.json`, app.config finalization (ids, icons, splash,
permissions rationale strings), credentials documentation, associated-
domain files, runbook, versioning, smoke checklist, optional CI trigger.
Out of scope: store listing/public review (v2), push transport code
(46), Expo Updates OTA (v2 — note below).

## Detailed Requirements

0. **Owner-inputs precondition** (this issue cannot finish without
   them; the runbook's owner-manual section collects them): final
   bundle id / Android package name, Apple team id + Developer Program
   membership, App Store Connect app record, Play Console app +
   internal testers, EAS project link, APNs key + FCM service account
   uploaded via `eas credentials`, Android signing SHA-256
   fingerprints, `EXPO_ACCESS_TOKEN` (worker) and `EXPO_TOKEN` (CI —
   distinct secrets, named exactly so), and the beta API/web URLs for
   the preview profile. The agent-executable work below uses loud
   `TODO(owner)` placeholders until these exist.
1. `eas.json` profiles:
   - `development`: dev client, internal distribution, simulators
     enabled.
   - `preview`: internal distribution; iOS → TestFlight (submit),
     Android → Play internal testing track (submit); channel `preview`;
     `EXPO_PUBLIC_API_URL/WS_URL` pointed at the beta VPS.
   - `production` profile stubbed but unused (v2 marker).
2. `app.config.ts` finalization: real bundle id / package name
   (owner-chosen, placeholder `com.OWNER.famchat` with a loud TODO the
   runbook resolves), app icons + splash from the 38 lettermark set
   (adaptive icon for Android), iOS `NSCameraUsageDescription` /
   `NSPhotoLibraryUsageDescription` etc. localized rationale strings
   (ja/en via config plugins), `associatedDomains` + Android intent
   filters for `${APP_BASE_URL}` (activates 42/46 links) with the
   well-known files served by the web app from
   `apps/web/public/.well-known/`:
   `apple-app-site-association` — JSON template
   `{"applinks":{"details":[{"appIDs":["<TEAMID>.<bundleId>"],
   "components":[{"/":"/invite/*"},{"/":"/link"}]}]}}` (no file
   extension; must be served `application/json`) — and
   `assetlinks.json` —
   `[{"relation":["delegate_permission/common.handle_all_urls"],
   "target":{"namespace":"android_app","package_name":"<pkg>",
   "sha256_cert_fingerprints":["<SHA256>"]}}]` — placeholders filled
   from the req-0 owner inputs; a web unit test asserts both routes
   return 200 + `content-type: application/json`.
3. `docs/mobile-release.md` runbook with two clearly separated
   sections:
   - **Owner-manual (once)**: Apple Developer Program enrollment, App
     Store Connect app record + TestFlight internal group, Play Console
     account + app + internal testers list, EAS project link
     (`eas init`), push credentials (`eas credentials`: APNs key
     upload, FCM service account), `EXPO_ACCESS_TOKEN` secret for the
     worker, well-known file values (team id, SHA-256 fingerprints).
   - **Per-release (agent-executable)**: version bump rules, then per
     platform: `eas build --profile preview --platform ios` /
     `--platform android`, `eas submit --profile preview --platform ios
     --latest` (TestFlight internal group) / `… --platform android
     --latest` (Play internal track), smoke checklist, tagging.
4. Versioning: `version` (semver, user-facing) + auto-increment
   `buildNumber`/`versionCode` via EAS (`autoIncrement`); runtime
   version policy `appVersion` (no OTA updates in v1 — expo-updates
   disabled; noted as v2 with rationale: update auditability for a
   kids' app).
5. Smoke checklist `docs/mobile-release-checklist.md` (executed per
   build, both platforms): install via TestFlight/internal track, adult
   login, child QR link, chat send/receive + push (app killed), image
   round-trip, board post, report flow, quiet-hours lock, deep link
   from push, version/build visible in settings.
6. Optional CI: manual-dispatch GitHub Actions workflow
   `mobile-build.yml` running `eas build --non-interactive` with
   `EXPO_TOKEN` secret — **documented but disabled by default**
   (workflow present with `workflow_dispatch` only; secret setup is an
   owner step).
7. Tests/validation: `eas.json` + app.config schema-validated
   (`expo config --type public` parses); web unit test for the
   well-known routes (200 + `application/json` + parseable JSON); the
   rest is the checklist by nature.

## Acceptance Criteria

- [ ] A beta family member with no dev tools installs via TestFlight /
      Play internal link and passes the smoke checklist (evidence:
      completed checklist + screenshots in PR).
- [ ] Push works in the distributed build (46 checklist re-run on the
      TestFlight build specifically).
- [ ] Universal/app links open the app from invite/room URLs on both
      platforms.
- [ ] Runbook lets the owner rotate a release end-to-end without agent
      improvisation (dry-run review by owner).

## Validation

```bash
pnpm -w typecheck && pnpm -w lint
pnpm --filter @famchat/mobile exec expo config --type public   # parses clean
pnpm --filter @famchat/web test -- -t well-known
# then: runbook per-release section executed once end-to-end
```

## Dependencies

46, 47 (feature-complete app), 38 (web serves static well-known
files) + the owner-inputs precondition (req 0). The preview profile's
API URL is a config value the owner supplies (typically the 49 beta
host, but not a build-order dependency).

## Acceptance split

Agent-complete: eas.json, app.config, well-known templates + tests,
runbook, checklist documents. Owner-gated: store records, credentials,
the actual TestFlight/Play install evidence (owner executes the
runbook and attaches the completed checklist).

## Non-goals

Public store review, Kids Category compliance work, OTA updates, CI
auto-release (all v2).

## Design References

- DESIGN §17 (distribution), ADR-004;
  research/platform-constraints.md §2–3
