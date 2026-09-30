# AGENTS.md

This file defines how agents should work in this repository.

These instructions apply repo-wide unless the user gives a more specific request.

## Intent

This is the `bouola.systemd_backup_jobs` Ansible role, published as open source. It runs backup commands written by
the user on systemd timers and publishes only complete, verified outputs.

The role does not know any backup tool. Its value is the frame around each command: one job at a time per host,
timeout, free-space check, temporary directory, validation, `manifest.json`, atomic replacement of the previous output,
and metrics. Keep it that way: a new need is solved by a better command or a new generic job key, never by a
tool-specific job type.

## Layout

- `files/systemd-backup-jobs`: the runner, Python 3 standard library only (`run <job>`, `verify <dir>`). It must work
  on the Python shipped by every supported OS (3.10 on Ubuntu 22.04): no newer syntax or modules.
- `templates/`: the `systemd-backup-jobs@.service` template, one timer per job, `config.json`, and one JSON
  definition per job.
- `tasks/`: `validate.yml`, `install.yml`, `jobs.yml`, `cleanup.yml`, included from `main.yml`.

## Variables

- Public variables start with `systemd_backup_jobs_` and are all documented in `defaults/main.yml` and `README.md`.
- Private variables (loop variables, registered results, facts) start with `_systemd_backup_jobs_`. `.ansible-lint`
  enforces this for loop variables.
- Defaults are typed and truthful: every default must be implemented end to end.
- Use YAML booleans `true` and `false`, never `yes` or `no`.

## Task style

- Every task has a sentence-case `name:` in double quotes and uses FQCNs.
- Use modules instead of `command` whenever one exists. When `command` is needed, set `changed_when` explicitly.
- Validation uses `ansible.builtin.assert` with an explicit multiline `fail_msg: >-`.
- Every loop sets `loop_control.loop_var` and `loop_control.label`.
- Role tags are `role-systemd-backup-jobs-validate`, `-install`, `-jobs` and `-cleanup`, applied through
  `include_tasks` with `apply.tags`.

## Safety

- Never delete a published output implicitly. Outputs of undefined jobs are removed only with
  `systemd_backup_jobs_prune_orphan_outputs: true`.
- A failed run must keep the previous output and its last success timestamp.
- Never put secret values on a command line: `env_file` values reach containers through the Docker CLI environment.
- The role must not install Docker, create users other than the output group, or change permissions outside its
  own directories.

## Testing and validation

Before considering a change done, run what is available locally:

- `yamllint .`
- `ansible-lint`
- `python3 -m py_compile files/systemd-backup-jobs`
- `molecule test -s default` and `molecule test -s docker` when Docker access is available

Scenario checks belong in Ansible `molecule/*/verify.yml` playbooks. If a scenario cannot run, say so in the final
report.

Two Molecule settings must stay in place:

- `hostname:` on each platform. Platform names contain `_`, which is invalid in a hostname; without an explicit one,
  `sudo` cannot resolve the host and every task waits about 40 s for a DNS timeout.
- `tmpfs:` for `/var/lib/docker` and `/var/lib/containerd` in the `docker` scenario. The nested Docker daemon cannot
  mount overlay layers on the container's overlay root. The geerlingguy images do not ship Docker; `prepare.yml`
  installs it.

After changing `molecule.yml`, run `molecule destroy` first: an existing container keeps its old settings.

## Documentation

- README examples must match the implemented contract and use the FQCN `bouola.systemd_backup_jobs`.
- Remove documentation for deleted variables and behavior.
- Commits follow Conventional Commits; semantic-release derives the version from them.

## When to ask the user

Ask before changing the public contract: job keys, variable names, output layout, manifest format, metric names, or
the runner's command-line interface.
