# Issue 50: Backup + tested restore

## Summary

Implement the backup/restore story: nightly host-cron backups of
PostgreSQL and MinIO with retention, a guided restore script, a scripted
restore verification against a scratch stack, and the runbook with
RPO/RTO statements.

## Context

DESIGN §18.2: RPO 24 h / RTO 4 h for beta; "restore is tested" is an
acceptance criterion, not a hope. Family photos are irreplaceable — this
is a trust feature (ADR-003 posture).

## Scope

In scope: backup.sh, restore.sh, verify-restore.sh, cron installation
docs, optional encryption/offsite guidance, runbook.
Out of scope: continuous WAL archiving (v2), monitoring/alerting on
backup failure beyond exit codes + log (v2 — documented gap), space
export (36, user-facing).

## Detailed Requirements

1. `scripts/backup.sh` (bash, `set -euo pipefail`, host-run next to the
   compose file):
   - `pg_dump -Fc` via `docker compose exec -T postgres` → 
     `backups/pg/famchat-<UTC timestamp>.dump`.
   - MinIO data: `mc mirror --overwrite` via a transient `minio/mc`
     container → `backups/minio/` (bucket-level, preserves keys).
   - Retention prune: keep 14 daily + 8 weekly (ISO week stamp
     dedupe); deletes only within `backups/` (path-guard asserted).
   - `.env` and secrets are **excluded by design** (documented: operator
     stores secrets separately, e.g. password manager).
   - Non-zero exit on any failure; final line `BACKUP OK <sizes>`
     (cron mail/log friendly).
   - Optional encryption hook: if `BACKUP_AGE_RECIPIENT` set, pipe dump
     through `age` (documented, off by default).
2. `scripts/restore.sh`: interactive (typed `RESTORE` confirm) or
   `--yes`; stops app services (api/worker/web), restores the chosen
   dump via `pg_restore --clean --if-exists`, `mc mirror` back into
   minio, runs `migrate` service (forward-only safety per 49's
   runbook), restarts, waits `/readyz`, prints post-restore checklist
   (spot-check counts).
3. `scripts/verify-restore.sh` (the drill): spins an isolated compose
   project (`-p famchat-restore-test`, separate volumes/ports) from the
   latest backup, restores into it, then asserts: migrations current,
   row counts (spaces/users/messages/attachments) within tolerance of
   values recorded at backup time (backup.sh writes a manifest JSON
   with counts), a sampled attachment object exists in restored minio
   and its sha256 matches the manifest sample; tears down. Designed to
   run on the VPS off-peak (cron monthly) and manually before
   release (54 gate).
4. Cron installation: `docs/ops/backup.md` — crontab lines (nightly
   03:30 JST backup; monthly verify), disk sizing guidance, offsite
   options (rclone to object storage — documented pattern, not
   implemented), encryption guidance, RPO 24 h / RTO 4 h statement +
   what would violate them.
5. Runbook `docs/ops/restore.md`: full-loss scenario (new VPS →
   install → restore → DNS), partial scenarios (bad deploy rollback via
   restore, single-space accidental deletion — point at grace period
   36 first), the forward-only-migration caveat.
6. Failure-path tests (scripted, run in CI on a schedule-label — not
   every PR): backup.sh on the dev stack produces dump + manifest;
   verify-restore.sh passes against it; corrupted-dump path fails
   loudly (inject truncation).

## Acceptance Criteria

- [ ] Nightly backup produces dump + mirror + manifest with retention
      pruning (evidence: two consecutive runs' listings).
- [ ] `verify-restore.sh` passes on the dev stack AND on the beta VPS
      (transcript in PR / ops log).
- [ ] Full restore drill executed once end-to-end on a scratch VM
      following only the runbook (evidence: transcript + timing vs
      RTO).
- [ ] Backups contain no `.env`/secrets (assertion in verify script).

## Validation

```bash
bash scripts/backup.sh && bash scripts/verify-restore.sh
# drill: docs/ops/restore.md on a scratch VM
```

## Dependencies

49 (compose stack + runbooks).

## Non-goals

WAL/PITR, managed backup services, backup monitoring/alerting (v2 —
listed as known gap in the runbook), space-level export (36).

## Design References

- DESIGN §18.2 (backup/restore, RPO/RTO), §19.6 (secrets exclusion);
  ADR-005
