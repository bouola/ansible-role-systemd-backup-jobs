# [2.0.0](https://github.com/bouola/ansible-role-systemd_backup_jobs/compare/v1.0.0...v2.0.0) (2026-10-02)


* feat!: label backup metrics with backup_job and remove metrics of inactive jobs ([16a66c8](https://github.com/bouola/ansible-role-systemd_backup_jobs/commit/16a66c87c4e4b173e2a8971f0b866ad937b11473))


### BREAKING CHANGES

* metrics use the backup_job label instead of job. Prometheus renamed
the former job label to exported_job; update queries, alerts and dashboards.

# 1.0.0 (2026-09-30)


### Features

* add systemd_backup_jobs role ([5b52ca3](https://github.com/bouola/ansible-role-systemd_backup_jobs/commit/5b52ca3232d27cd4a4d8f3f347e4f5931aa6a2eb))
* refine job verification and metrics handling in Molecule scenario ([0c1167c](https://github.com/bouola/ansible-role-systemd_backup_jobs/commit/0c1167cc2f72b21b44598bb58687734d0394aae6))
