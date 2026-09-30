# Introduction
In this document, a list of relevant settings for hardening systemd_backup_jobs is provided.
This document is non-exhaustive, however, it provides a solid base of security hardening
measures.

## Output access

- Outputs are owned by `root:<systemd_backup_jobs_output_group>` with modes `0750` for directories and `0640` for
  files. Only root writes them.
- Add to `systemd_backup_jobs_output_group_members` only the accounts that must read backups, such as the account a
  collector host connects with. Membership grants read access to every job's output, including secrets such as
  database dumps or GitLab's `gitlab-secrets.json`.
- Prefer pulling outputs from a collector host over pushing them: the backed-up host then has no access to the
  backup storage, so a compromised host cannot delete its backups.

## Job definitions

- Job definitions live in `/etc/systemd-backup-jobs/jobs/` with mode `0600`, because commands may reference secrets.
- Keep secrets out of `command`. Reference them through `env_file` and environment variables instead: values from
  `env_file` are passed to containers through the Docker CLI environment, never on a command line.
- Pin `image` to a version or a digest. A floating tag can change the backup tool without notice.

## Execution

- Jobs run as root, because they need the Docker socket or host backup tools. Treat anyone who can change
  `systemd_backup_jobs_definitions` as root on the host.
- `docker` jobs mount only `/backup` read-write. Mount service volumes read-only (`:ro`).
- The service runs with `PrivateTmp=true` and a lower CPU and I/O priority.

## Retention and integrity

- The role keeps only the latest output. Keep history, encryption and integrity checks in a dedicated tool (restic,
  borg) on the collector.
- Run `systemd-backup-jobs verify <dir> --max-age <age>` after transferring outputs to catch stale or corrupted
  copies.
- Alert on `systemd_backup_job_last_status == 0` and on a `systemd_backup_job_last_success_timestamp_seconds` older
  than the expected interval.
