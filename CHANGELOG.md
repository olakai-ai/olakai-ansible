# Changelog

All notable changes to this role are recorded here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Role versions are tagged in lockstep with the on-prem bundle they were tested against.

## [0.1.0] - Unreleased

### Added

- `install.yml` and `upgrade.yml` playbooks that wrap the signed `get.sh` and `upgrade.sh`.
- `olakai` role: preflight checks, Docker or rootless Podman runtime preparation, firewall ports, install, upgrade.
- Example inventory and `group_vars` with an Ansible Vault reference for the license key.
- ansible-lint and yamllint CI on pull requests.
- `olakai_print_setup_url` (default `false`). The one-time setup URL is written to `<olakai_install_dir>/.olakai-setup-url` (mode `0600`) instead of the job log; set the variable to `true` to print it with a warning.

### Security

- Removed a customer name and internal plan identifiers from comments. They remain in the git history of this private repository. History to be squashed before the repository goes public.

### Changed

- `podman-rootless` is prepared but not functional: the task file ends with a `fail` unless `olakai_allow_preview_runtime: true`. The Olakai installer does not accept a Podman runtime yet (`docker --version` under the podman-docker shim reports the Podman version, which `get.sh` rejects as "Docker 24+ required").
- `olakai_subid_start` default moved from 100000 to 524288, and the role fails when the range overlaps any existing `/etc/subuid` or `/etc/subgid` entry. 100000 is the range `useradd` gives the first interactive user (`ubuntu`, `opc`, `ec2-user`).
- `podman-restart.service` (a oneshot unit) is enabled only, no longer started on every run.

### Fixed

- Re-running `install.yml` on a host that already has `.env` no longer reads the registered installer result (a skipped task has no `stdout` or `rc`; a `creates:` skip has a placeholder `stdout`). Every task after `get.sh` is guarded with `olakai_install_ran`, and the second run prints "already installed here, no action".
- `upgrade.yml` tolerates a skipped `upgrade.sh` (check mode) instead of failing on an undefined `stdout`.
- The stderr redaction no longer inserts `[redacted]` between every character when the license key is empty.
