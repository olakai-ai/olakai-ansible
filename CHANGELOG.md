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
