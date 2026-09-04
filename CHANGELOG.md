# Changelog

All notable changes to this role are recorded here.
The format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).
Role versions are tagged in lockstep with the on-prem bundle they were tested against.

## [0.2.0] - Unreleased

### Added

- `olakai_runtime: podman-rootless` is a working install mode. The role passes `--runtime=podman-rootless --service-user=<olakai_service_user>` to `get.sh` (2026.09.03.2 or later), which prepares the host itself: Podman 5 + `podman-docker` from the distro repositories, the `docker-compose` provider pinned by SHA256 and registered in `/etc/containers/containers.conf.d/olakai-compose.conf`, the service user with a free 65536-wide subordinate id range (overlap-checked), systemd linger, `net.ipv4.ip_unprivileged_port_start=80`, the user-scope `podman.socket` and `podman-restart.service`, then hands the install directory to the service user (`.olakai-runtime` marker, Podman overlay) and runs `install.sh` through `runuser`. Qualified on Oracle Linux 9.8 (Podman 5.8.2); other RHEL 9 family distributions are accepted.
- Preflight for `podman-rootless`: systemd, cgroup v2, unprivileged user namespaces, `runuser`, a service user name `get.sh` accepts, and a pinned `olakai_version` of 1.5.10 or later (the bundle floor for rootless Podman; checked before the license key is consumed).
- `upgrade.yml` detects the runtime from `<olakai_install_dir>/.olakai-runtime` when `olakai_runtime` is `auto`, and runs `upgrade.sh` as the service user (with `XDG_RUNTIME_DIR` and the D-Bus address of that user) under rootless Podman. An explicit `olakai_runtime` that contradicts the marker fails before `upgrade.sh` runs.
- `olakai_env_overrides` (dict): `.env` values for managed-state deployments (customer-managed PostgreSQL, Redis, object storage). Written to a root-owned `0600` temporary file with `no_log`, passed as `get.sh --env-file=`, removed after the run. Keys `get.sh` writes itself are refused by the preflight. Needs `get.sh` 2026.09.04 or later.
- `olakai_private_ca_pem` / `olakai_private_ca_file`: a private CA (PEM) passed as `get.sh --private-ca=`; `get.sh` installs it at `<olakai_install_dir>/certs/private-ca.pem` and activates the private-CA overlay. Needs `get.sh` 2026.09.04 or later.
- README section "Managed state" with the `.env` keys, the DNS-name and `sslmode=verify-full` guidance and the private-CA guidance from the bundle README.
- The setup-URL retrieval hints and the success message name the service user form (`sudo runuser -u <user> -- ...`) under rootless Podman.

### Changed

- `runtime_podman_rootless.yml` no longer installs Podman, downloads Compose, creates the service user, writes `/etc/subuid` and `/etc/subgid`, enables linger, sets the sysctl, or enables user units. `get.sh` does all of it, with the subordinate id overlap check that is tested against the bundle. The role keeps a read-only preflight and the firewall rules.
- `olakai_runtime: auto` still resolves to `docker` on every family for a fresh install. `podman-rootless` stays explicit opt-in.
- Compose binary constants in `vars/main.yml` are now used by the Amazon Linux docker path only.
- README: the runtime section describes what `get.sh` changes on the host under `podman-rootless`, the bundle 1.5.10 floor, the automatic service-user upgrade, and the security model (a container escape yields the service user, not root). The installer compatibility banner names `get.sh` 2026.09.03.2 for rootless Podman and 2026.09.04 for `--env-file` / `--private-ca`.

### Removed

- `olakai_allow_preview_runtime`, `olakai_subid_start` and `olakai_unprivileged_port_start`. `get.sh` picks the subordinate id range (first free 65536-wide range at or above 524288) and always sets `net.ipv4.ip_unprivileged_port_start=80` under `podman-rootless`.
- `olakai_podman_packages` (`vars/Debian.yml`, `vars/RedHat.yml`) and `olakai_compose_bin_path`.
- The `ansible.posix.sysctl` use; `ansible.posix` is still required for `firewalld`.

## [0.1.0] - Unreleased

### Added

- `install.yml` and `upgrade.yml` playbooks that wrap `get.sh` and `upgrade.sh`.
- `olakai` role: preflight checks, Docker or rootless Podman runtime preparation, firewall ports, install, upgrade.
- Example inventory and `group_vars` with an Ansible Vault reference for the license key.
- ansible-lint and yamllint CI on pull requests, with pinned tool versions, on ansible-core 2.15 (the declared floor, yamllint and syntax checks) and 2.19 (with ansible-lint).
- `olakai_print_setup_url` (default `false`). The one-time setup URL is written to `<olakai_install_dir>/.olakai-setup-url` (mode `0600`) instead of the job log; set the variable to `true` to print it with a warning.

### Security

- Removed a customer name and internal plan identifiers from comments. They remain in the git history of this private repository. History to be squashed before the repository goes public.

### Changed

- `olakai_runtime: auto` resolves to `docker` on every family, including the RedHat family and Oracle Linux, until the installer accepts `--runtime=podman-rootless`. `podman-rootless` is explicit opt-in.
- `podman-rootless` is prepared but not functional: the task file ends with a `fail` unless `olakai_allow_preview_runtime: true`. The Olakai installer does not accept a Podman runtime yet (`docker --version` under the podman-docker shim reports the Podman version, which `get.sh` rejects as "Docker 24+ required").
- `olakai_subid_start` default moved from 100000 to 524288, and the role fails when the range overlaps any existing `/etc/subuid` or `/etc/subgid` entry. 100000 is the range `useradd` gives the first interactive user (`ubuntu`, `opc`, `ec2-user`).
- `podman-restart.service` (a oneshot unit) is enabled only, no longer started on every run.
- `olakai_unprivileged_port_start: ""` skips the sysctl and leaves the kernel default (1024) alone.
- README: installer compatibility banner moved to the top; `get.sh` described as fetched over HTTPS and not itself signed (the bundle is cosign-verified); egress list now includes GitHub and Sigstore hosts; the placeholder documentation link is gone.

### Fixed

- Re-running `install.yml` on a host that already has `.env` no longer reads the registered installer result (a skipped task has no `stdout` or `rc`; a `creates:` skip has a placeholder `stdout`). Every task after `get.sh` is guarded with `olakai_install_ran`, and the second run prints "already installed here, no action".
- `upgrade.yml` tolerates a skipped `upgrade.sh` (check mode) instead of failing on an undefined `stdout`.
- The stderr redaction no longer inserts `[redacted]` between every character when the license key is empty.
- `--check` no longer reports a fake success: the download and the run are skipped in check mode and the play prints "check mode: get.sh not executed".
- The setup URL is never stored in a fact (a fact cache can persist `set_fact` values even with `no_log`). The `copy` task reads it from the registered stdout. The redacted stderr tail is cleared at the end of the play.
- "No `OLAKAI_SETUP_URL=` line" (get.sh older than 2026.09.03) and "line present but empty" (URL not readable within 30 s, or `--quiet`) are reported with two different messages. The retrieval command uses the compose service name `bootstrap`.
- The 409 detection matches `handshake rejected (409)` or `(409)`, not any `409` in stderr.
- `get.sh` and the get.docker.com script are downloaded into a root-only `mkdtemp` directory (`ansible.builtin.tempfile`), not a fixed path under `/tmp`.
- Docker runtime: the role prints the same trust-boundary warning `get.sh` prints before it runs the get.docker.com script. On Oracle Linux (which get.docker.com rejects) the role leaves the Docker install to `get.sh --auto-install-docker`.
