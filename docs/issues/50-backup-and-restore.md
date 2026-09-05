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

1. `scripts/backup.sh` (bash, `set -euo pipefail`, `umask 077`,
   host-run next to the compose file):
   - Layout + naming (canonical, consumed by restore/verify):
     `backups/pg/famchat-<YYYYMMDDTHHMMSSZ>.dump` (+`.age` when
     encrypted), `backups/minio/` mirror,
     `backups/manifests/famchat-<stamp>.json`.
   - `pg_dump -Fc` via `docker compose exec -T postgres`.
   - MinIO data: `mc mirror --overwrite` via a transient pinned `mc`
     container.
   - **Manifest JSON schema (exact)**: `{ stamp, pgDumpFile,
     pgDumpSha256, rowCounts: { spaces, users, messages, attachments },
     sampleObject: { key, sha256 } | null, bytes: { pgDump, minio } }`
     — row counts via `psql -tAc`; no URLs, credentials, or env values
     ever appear in the manifest or logs.
   - Retention prune: keep 14 daily + 8 weekly (ISO week stamp dedupe);
     deletes only within `backups/` (path-guard asserted).
   - **Offsite replication is part of the script, not an afterthought**:
     when `BACKUP_RCLONE_REMOTE` is set (e.g. `b2:famchat-backups`),
     `rclone sync backups/ $BACKUP_RCLONE_REMOTE` runs after a
     successful local backup (rclone via pinned container). The beta
     instance MUST set it (runbook + 54 checklist item) — RPO 24 h
     against full-disk loss depends on it; `.env` and secrets remain
     excluded by design (operator stores secrets separately).
   - Permissions: `backups/` 0700, files 0600 (enforced by the script).
   - Non-zero exit on any failure; final line `BACKUP OK <sizes>`.
   - Encryption hook: if `BACKUP_AGE_RECIPIENT` set, dump piped through
     `age` → `.dump.age` (restore decrypts when `BACKUP_AGE_KEY_FILE`
     set).
2. `scripts/restore.sh --dump <path> [--yes]`: interactive typed
   `RESTORE` confirm unless `--yes`; decrypts `.age` inputs when
   `BACKUP_AGE_KEY_FILE` is set; stops app services (api/worker/web),
   restores via `pg_restore --clean --if-exists`, `mc mirror` back into
   minio, runs the `migrate` service (forward-only safety per 49's
   runbook), restarts, waits `/readyz`, prints post-restore spot-check
   (row counts vs the matching manifest).
3. `scripts/verify-restore.sh` (the drill): spins an isolated compose
   project (`-p famchat-restore-test`, separate volumes/ports) from the
   latest backup, restores into it, then asserts: migrations current;
   row counts equal the manifest exactly; the manifest's sampleObject
   exists in restored minio with matching sha256; **no `.env`/secret
   files present anywhere under `backups/`** (find-based assertion);
   tears down. Runs on the VPS off-peak (cron monthly) and manually
   before release (54 gate).
4. Cron installation: `docs/ops/backup.md` — crontab block including
   `CRON_TZ=Asia/Tokyo` (with the UTC-equivalent line for hosts without
   CRON_TZ support), nightly 03:30 backup + monthly verify, disk sizing
   guidance, rclone remote setup (required for beta), encryption
   guidance, RPO 24 h / RTO 4 h statement + what would violate them.
5. Runbook `docs/ops/restore.md`: full-loss scenario (new VPS →
   install → restore → DNS), partial scenarios (bad deploy rollback via
   restore, single-space accidental deletion — point at grace period
   36 first), the forward-only-migration caveat.
6. Failure-path tests (scripted; this issue provides
   `scripts/test-backup-cycle.sh` runnable locally against the dev
   stack — **CI wiring is explicitly issue 51's**, which adds it as a
   scheduled job): backup.sh produces dump + manifest; verify-restore.sh
   passes against it; corrupted-dump path fails loudly (inject
   truncation).

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
