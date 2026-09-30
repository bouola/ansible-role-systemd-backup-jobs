# Ansible role - systemd_backup_jobs
[![Maintainer](https://img.shields.io/badge/maintained%20by-bouola-e00000?style=flat-square)](https://github.com/bouola)
[![License](https://img.shields.io/github/license/bouola/ansible-role-systemd_backup_jobs?style=flat-square)](LICENSE)
[![Release](https://img.shields.io/github/v/release/bouola/ansible-role-systemd_backup_jobs?style=flat-square)](https://github.com/bouola/ansible-role-systemd_backup_jobs/releases)
[![Status](https://img.shields.io/github/actions/workflow/status/bouola/ansible-role-systemd_backup_jobs/ci.yml?style=flat-square&label=tests&branch=main)](https://github.com/bouola/ansible-role-systemd_backup_jobs/actions?query=workflow%3A%22CI%22)
[![Ansible version](https://img.shields.io/badge/ansible-%3E%3D2.15-black.svg?style=flat-square&logo=ansible)](https://github.com/ansible/ansible)
[![Ansible Galaxy](https://img.shields.io/badge/ansible-galaxy-black.svg?style=flat-square&logo=ansible)](https://galaxy.ansible.com/bouola/systemd_backup_jobs)

Run scheduled backup commands with systemd timers, and publish only outputs that are complete and verified.

Galaxy FQCN: `bouola.systemd_backup_jobs`

Each job is a command you write: a `pg_dump` in a throwaway container attached to your service's Compose network, a
`gitlab-backup` on the host, an `rsync` that pulls another host's backups. The role runs it on its own systemd timer
and wraps it in the same safeguards every time:

- one job at a time per host, with a per-job timeout;
- a free-space check against the size of the previous output;
- the command writes into a temporary directory, never over the previous output;
- the output is published only when the command succeeds and wrote at least one non-empty file, with a
  `manifest.json` (sizes and SHA-256 checksums), then replaces the previous output in a single rename;
- a failed run keeps the previous output and reports the failure;
- Prometheus metrics through node-exporter's textfile collector.

The role keeps only the **latest** output of each job. Pair it with a tool that keeps history (restic, borg) on the
host that collects the outputs.

## Requirements

- Ansible 2.15 or newer
- Debian 12, Debian 13, Ubuntu 22.04, or Ubuntu 24.04, with systemd
- Python 3 on the managed host (already required by Ansible)
- Docker, only when a job uses `mode: docker`. The role does not install it.

## Role Variables

| Variable | Type | Default | Description |
| --- | --- | --- | --- |
| `systemd_backup_jobs_definitions` | list | `[]` | Backup jobs to install. See [Job definition](#job-definition). |
| `systemd_backup_jobs_output_dir` | string | `/var/backups/systemd-backup-jobs` | Directory where each job publishes its latest output, in `<output_dir>/<job name>/`. |
| `systemd_backup_jobs_output_group` | string | `backup-reader` | Group that owns the outputs and may read them. Created when missing. |
| `systemd_backup_jobs_output_group_members` | list | `[]` | Existing users added to the output group, for example the account a remote collector uses. The role never creates these users. |
| `systemd_backup_jobs_textfile_dir` | string | `""` | node-exporter textfile collector directory. Metrics are not written when empty. |
| `systemd_backup_jobs_min_free_ratio` | number | `1.5` | Free space required before a job starts, relative to the size of its previous output. `0` disables the check. |
| `systemd_backup_jobs_default_timeout` | string | `1h` | Timeout of jobs that do not set their own, for example `90s`, `30m`, `2h` or `1d`. |
| `systemd_backup_jobs_prune_orphan_outputs` | boolean | `false` | Remove the outputs of jobs that are no longer defined. Timers of undefined or disabled jobs are always removed. |
| `systemd_backup_jobs_bin_path` | string | `/usr/local/bin/systemd-backup-jobs` | Path of the runner. |
| `systemd_backup_jobs_config_dir` | string | `/etc/systemd-backup-jobs` | Runner and job configuration directory. |
| `systemd_backup_jobs_state_dir` | string | `/var/lib/systemd-backup-jobs` | Runner state (last results, lock file). |
| `systemd_backup_jobs_systemd_dir` | string | `/etc/systemd/system` | Directory where the service template and timers are installed. |

### Job definition

| Key | Required | Mode | Description |
| --- | --- | --- | --- |
| `name` | yes | both | Lowercase slug (letters, digits, dashes). Names the output directory, the timer and the metrics. |
| `mode` | yes | both | `docker`: run the command in a throwaway container. `host`: run it on the host. |
| `schedule` | yes | both | systemd [calendar expression](https://www.freedesktop.org/software/systemd/man/latest/systemd.time.html#Calendar%20Events), validated with `systemd-analyze calendar`. |
| `command` | yes | both | A string runs with `/bin/sh -c`. A list runs as is, without a shell. |
| `timeout` | no | both | Maximum run time, for example `30m`. Defaults to `systemd_backup_jobs_default_timeout`. |
| `randomized_delay` | no | both | Random delay added to the schedule, for example `10m`, to spread jobs out. |
| `env_file` | no | both | Absolute path of a `KEY=VALUE` file on the host, such as the service's existing `.env`. Quotes around values are removed as Docker Compose does. |
| `enabled` | no | both | `false` removes the job's timer and definition but keeps its last output. Defaults to `true`. |
| `image` | yes | docker | Image providing the backup tool. Use the same major version as the server, for example `postgres:18-alpine` for a PostgreSQL 18 database. |
| `network` | no | docker | Docker network to attach the container to, so it reaches the service by name. The job fails when the network does not exist. |
| `volumes` | no | docker | Extra `--volume` values, for example `taiga-media:/data:ro` to back up a named volume. |
| `user` | no | docker | `--user` value for images that must not run as root. The output directory is writable by root only. |

**Where the command writes:**

- `docker` mode: into `/backup` inside the container.
- `host` mode: into the directory in the `BACKUP_OUTPUT` environment variable.

Commands run as root. `env_file` values are passed to the container through the Docker CLI's environment, so they
never appear on a command line.

## How it works

```text
systemd-backup-jobs@<name>.timer   (OnCalendar=<schedule>, Persistent=true)
        |
systemd-backup-jobs@<name>.service -> systemd-backup-jobs run <name>
        |
        |- wait for the host lock (one job at a time)
        |- check free space
        |- run the command into <output_dir>/.<name>.tmp
        |- check the exit code and that a non-empty file was written
        |- write manifest.json, set root:<output_group> 0750/0640
        |- replace <output_dir>/<name> with the new output in one rename
        '- write metrics
```

`Persistent=true` runs a job at boot when the host was off at its scheduled time. Two jobs whose schedules overlap
never run at the same time: the second one waits for the first.

Run a job now, or check what a collector received:

```bash
systemctl start systemd-backup-jobs@keycloak-postgres.service
journalctl -u systemd-backup-jobs@keycloak-postgres.service
systemd-backup-jobs verify /var/backups/systemd-backup-jobs --max-age 20h
```

`verify` checks every `manifest.json` under a directory: the output must be younger than `--max-age`, and every file
must match its checksum. It exits non-zero otherwise.

### Output

```text
/var/backups/systemd-backup-jobs/          root:backup-reader 0750
└── keycloak-postgres/
    ├── keycloak.dump                      root:backup-reader 0640
    └── manifest.json
```

```json
{
  "job": "keycloak-postgres",
  "mode": "docker",
  "image": "postgres:18-alpine",
  "host": "vm-keycloak-01",
  "started_at": "2026-09-30T23:00:04.120331+00:00",
  "finished_at": "2026-09-30T23:00:41.908112+00:00",
  "duration_seconds": 37.788,
  "total_size": 48213456,
  "files": [{"path": "keycloak.dump", "size": 48213456, "sha256": "9f2c..."}],
  "runner_version": "1"
}
```

### Metrics

Written to `<textfile_dir>/systemd_backup_job_<name>.prom` when `systemd_backup_jobs_textfile_dir` is set:

| Metric | Description |
| --- | --- |
| `systemd_backup_job_last_run_timestamp_seconds{job}` | Start time of the last run. |
| `systemd_backup_job_last_status{job}` | `1` when the last run succeeded, `0` otherwise. |
| `systemd_backup_job_last_duration_seconds{job}` | Duration of the last run. |
| `systemd_backup_job_last_success_timestamp_seconds{job}` | Start time of the last successful run. Kept after a failure. |
| `systemd_backup_job_last_size_bytes{job}` | Size of the last published output. |

node-exporter must be started with `--collector.textfile.directory=<textfile_dir>`. Example alerts:

```yaml
- alert: BackupJobStale
  expr: time() - systemd_backup_job_last_success_timestamp_seconds > 26 * 3600
- alert: BackupJobFailed
  expr: systemd_backup_job_last_status == 0
```

## Dependencies

None.

## Example Playbook

PostgreSQL dump of a Docker Compose service, reusing the service's `.env` file:

```yaml
---
- name: "Back up Keycloak"
  hosts: keycloak
  become: true
  roles:
    - role: "bouola.systemd_backup_jobs"
      vars:
        systemd_backup_jobs_output_group_members: ["backup-collector"]
        systemd_backup_jobs_textfile_dir: "/var/lib/node_exporter/textfile_collector"
        systemd_backup_jobs_definitions:
          - name: "keycloak-postgres"
            mode: "docker"
            schedule: "*-*-* 01:00:00"
            randomized_delay: "5m"
            timeout: "30m"
            image: "postgres:18-alpine"
            network: "keycloak_default"
            env_file: "/app/keycloak/postgres.env"
            command: >-
              PGPASSWORD="$POSTGRES_PASSWORD" pg_dump -h postgres -U "$POSTGRES_USER" -d "$POSTGRES_DB"
              --format=custom --compress=0 --file=/backup/keycloak.dump
```

Files of a named volume, and GitLab omnibus on the host:

```yaml
systemd_backup_jobs_definitions:
  - name: "taiga-media"
    mode: "docker"
    schedule: "*-*-* 01:40:00"
    image: "alpine:3.20"
    volumes: ["taiga_taiga-media-data:/data:ro"]
    command: "tar -cf /backup/media.tar -C /data ."

  - name: "gitlab"
    mode: "host"
    schedule: "*-*-* 01:00:00"
    timeout: "3h"
    command: >-
      gitlab-backup create SKIP=registry,artifacts,builds BACKUP=latest GZIP_RSYNCABLE=yes
      && mv /var/opt/gitlab/backups/latest_gitlab_backup.tar "$BACKUP_OUTPUT"/
      && cp /etc/gitlab/gitlab-secrets.json /etc/gitlab/gitlab.rb "$BACKUP_OUTPUT"/
```

The same role on a collector host: pull other hosts' outputs, then keep history with restic.

```yaml
systemd_backup_jobs_definitions:
  - name: "pull-db-01"
    mode: "host"
    schedule: "*-*-* 03:00:00"
    command: >-
      rsync -a --delete --exclude '.*' --link-dest=/var/backups/systemd-backup-jobs/pull-db-01
      backup-collector@10.0.30.21:/var/backups/systemd-backup-jobs/ "$BACKUP_OUTPUT"/
      && systemd-backup-jobs verify "$BACKUP_OUTPUT" --max-age 20h

  - name: "restic-push"
    mode: "host"
    schedule: "*-*-* 04:00:00"
    timeout: "3h"
    env_file: "/etc/restic/restic.env"   # RESTIC_REPOSITORY, RESTIC_PASSWORD_FILE
    command: >-
      restic backup /var/backups/systemd-backup-jobs --exclude '/var/backups/systemd-backup-jobs/restic-*'
      && date --iso-8601=seconds > "$BACKUP_OUTPUT/pushed-at"
```

Jobs that produce no backup file, like `restic-push`, write a small marker file so that their runs are validated
and reported like any other job.

## Molecule Scenarios

| Scenario | Coverage |
| --- | --- |
| `default` | `host` jobs: timers, schedules, published output and permissions, output group access, manifest, metrics, failed run keeping the previous output, empty output rejected, `verify` detecting a modified file, removal of disabled and stale timers. |
| `docker` | `docker` jobs: network attachment, named volumes, exec-form commands, `env_file`, missing network, container cleanup. |

```bash
molecule test -s default
molecule test -s docker
```

## License

MIT
