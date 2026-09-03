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

### Fixed

- Re-running `install.yml` on a host that already has `.env` no longer reads the registered installer result (a skipped task has no `stdout` or `rc`; a `creates:` skip has a placeholder `stdout`). Every task after `get.sh` is guarded with `olakai_install_ran`, and the second run prints "already installed here, no action".
- `upgrade.yml` tolerates a skipped `upgrade.sh` (check mode) instead of failing on an undefined `stdout`.
- The stderr redaction no longer inserts `[redacted]` between every character when the license key is empty.
