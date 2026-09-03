# olakai-ansible

Ansible playbooks to install and upgrade [Olakai](https://olakai.ai) on-prem on one Linux host.

The role is a thin wrapper. Ansible prepares the operating system (container runtime, service user, firewall ports). The Olakai scripts `get.sh` and `upgrade.sh` do the trust-bearing work: license handshake, bundle download, SHA256 and cosign verification, `.env` generation, and the container bring-up. This repository contains no product code.

## Installer compatibility (read first)

> **This role requires `get.sh` 2026.09.03 or later.** It relies on three behaviours of that version: the `OLAKAI_LICENSE_KEY` environment variable, the `--force-overlay` flag, and the `OLAKAI_SETUP_URL=` line on stdout. Until `https://get.olakai.ai/get.sh` serves that version, the install task fails with `missing required input 'LICENSE_KEY'`. To review the role before then, point `olakai_get_sh_url` at a pre-release copy (an HTTPS URL, or a `file://` path on the target host) and pin it with `olakai_get_sh_sha256`.

| Role version | Minimum get.sh | Minimum bundle | Notes |
| --- | --- | --- | --- |
| 0.1.0 (unreleased) | 2026.09.03 (`OLAKAI_LICENSE_KEY`, `--force-overlay`, `OLAKAI_SETUP_URL=` line) | 1.5.x (`upgrade.sh --yes --to=`) | Written against an installer change that is in progress. |

What the role expects from `get.sh`:

1. `OLAKAI_LICENSE_KEY` in the environment is accepted instead of `--license-key=` (the flag wins when both are set).
2. `--force-overlay` (or `OLAKAI_FORCE_OVERLAY=true`) proceeds when the install directory is not empty.
3. A single line `OLAKAI_SETUP_URL=<url>` on stdout after a successful install. The line is present but empty when `get.sh` could not read the URL within 30 seconds or when `--quiet` is passed.

With an older `get.sh`: (1) the run fails with `missing required input 'LICENSE_KEY'`; (2) a non-empty install directory without `.env` aborts, so empty the directory first; (3) the role prints the `docker compose logs` command to retrieve the setup URL instead of writing the URL file.

```yaml
# group_vars/olakai.yml: review against a pre-release get.sh
olakai_get_sh_url: https://example.internal/olakai/get.sh
olakai_get_sh_sha256: "<sha256 of the reviewed get.sh>"
```

Tagged releases of this repository track the on-prem bundle versions they were tested against.

## What it does

`install.yml`:

1. Preflight: Linux, x86_64 or aarch64, 8 GB RAM, `python3`, a distribution the installer accepts.
2. Runtime: installs Docker Engine + Compose v2, or prepares rootless Podman (packages, service user, subordinate ids, linger, sysctl for ports 80/443, user-scope Podman socket). The rootless Podman path stops before `get.sh` today; see "Runtime choice".
3. Firewall: opens TCP 80 and 443 in firewalld or ufw when one of them is present.
4. Install: downloads `get.sh` into a root-only temporary directory, runs it non-interactively with the license key in its environment, writes the one-time setup URL to a `0600` file on the host, deletes the script.

`upgrade.yml`:

1. Preflight (same checks).
2. Runs `<install_dir>/upgrade.sh --yes --to=<version|latest>` and reports its verdict.

## Requirements

Control node:

- ansible-core 2.15 or newer. CI runs the lint and syntax checks on 2.15 and on the current release.
- Collections `ansible.posix` and `community.general` (`ansible-galaxy collection install --no-cache -r requirements.yml`).

Target host:

- One VM, 8 GB RAM, 80 GB free disk, x86_64 or aarch64.
- Target hosts accepted by the installer: Ubuntu 22.04 or newer, Debian 12 or newer, RHEL 9 or newer (Rocky and AlmaLinux 9 or newer), Oracle Linux 9 or newer, Amazon Linux 2023. The role's preflight applies the same list as `get.sh`. This is not a support statement. Refer to Olakai's platform support documentation for the qualified list.
- SSH access with password-less sudo, and `python3` on the host.
- A public DNS A record for `olakai_domain` that points at the VM before you run the install (Let's Encrypt needs it).
- Outbound HTTPS from the host to:
  - `get.olakai.ai` (installer) and `relay.olakai.ai` (license handshake, bundle download URL);
  - `public.ecr.aws` (container images);
  - `github.com` and `objects.githubusercontent.com` (the cosign binary that `get.sh` bootstraps, and the Docker Compose binary the role downloads for Amazon Linux and rootless Podman);
  - Sigstore, for online cosign verification: `rekor.sigstore.dev`, `fulcio.sigstore.dev`, `tuf-repo-cdn.sigstore.dev`. Not needed when `olakai_rekor_bundle` points at an offline bundle;
  - `get.docker.com` and `download.docker.com` when Docker is auto-installed (`olakai_auto_install_docker: true`).
- Inbound TCP 80 and 443 from the internet, open in your cloud security group. The role opens them on the host firewall only.

## Quick start

```bash
git clone https://github.com/olakai-ai/olakai-ansible.git
cd olakai-ansible
ansible-galaxy collection install --no-cache -r requirements.yml

cp inventory.example.ini inventory.ini            # set ansible_host and ansible_user
cp group_vars/olakai.example.yml group_vars/olakai.yml

# Put the license key in Ansible Vault and paste the output into group_vars/olakai.yml
ansible-vault encrypt_string 'OLK-your-license-key' --name olakai_license_key

# Edit olakai_domain and olakai_admin_email in group_vars/olakai.yml, then:
ansible-playbook -i inventory.ini install.yml --ask-vault-pass
```

The last task prints the dashboard URL and the path of the fallback setup link: `<olakai_install_dir>/.olakai-setup-url` on the host, mode `0600`. The admin also receives a magic-link e-mail.

The setup link is a one-time admin token, so the role does not print it. Read it on the host and delete the file after the first login:

```bash
sudo cat /opt/olakai-onprem/.olakai-setup-url
sudo rm /opt/olakai-onprem/.olakai-setup-url
```

Set `olakai_print_setup_url: true` to print the link in the play output anyway (for example on a laptop with no shared job log). Ansible and AWX job logs are retained and often readable by more people than the operator, so the default is `false`.

`ansible-playbook --check` runs the preflight and reports the OS changes it would make. It does not download or run `get.sh`; the play prints `check mode: get.sh not executed`.

## Upgrades

```bash
# Latest servable release
ansible-playbook -i inventory.ini upgrade.yml

# Pin a version
ansible-playbook -i inventory.ini upgrade.yml -e olakai_version=v1.6.0
```

No license key is needed. `upgrade.sh` authenticates to the Olakai relay with the deployment bearer stored in `<install_dir>/.env`.

If Olakai marks a release as containing a destructive migration, `upgrade.sh` refuses to run it unattended. Read the release note, then set `olakai_accept_data_loss: true` for that one run. This gate is separate from `--yes` on purpose.

## Runtime choice

`olakai_runtime` selects the container runtime.

| Value | Behaviour |
| --- | --- |
| `auto` (default) | `docker` on every family (Ubuntu, Debian, RHEL, Rocky, AlmaLinux, CentOS Stream, Oracle Linux, Amazon Linux) until the installer accepts `--runtime=podman-rootless`. |
| `docker` | Docker Engine 24+ with Compose v2. Installed from `get.docker.com` when missing and `olakai_auto_install_docker` is true (Amazon Linux uses the distro package; Oracle Linux is left to `get.sh`, which installs from Docker's CentOS package repository because `get.docker.com` rejects `ID=ol`). |
| `podman-rootless` | Explicit opt-in. Prepared but not functional until the installer accepts `--runtime=podman-rootless`. The role installs Podman with the `podman-docker` shim and a standalone Compose v2 binary, creates an unprivileged `olakai` service user with subordinate ids, and sets `net.ipv4.ip_unprivileged_port_start=80`. Then it stops with a `fail`. |

On Oracle Linux the role does not run `get.docker.com` (it rejects `ID=ol`). It delegates the Docker CE install to `get.sh --auto-install-docker`, which installs from Docker's CentOS package repository over HTTPS. Docker CE on Oracle Linux is community-supported: Docker does not list Oracle Linux as a supported platform for Docker Engine, and the CentOS packages are used as-is.

Why `podman-rootless` stops: `get.sh` does not accept a `--runtime=` flag yet (rootless Podman support in the Olakai installer is in progress), and no shipped `get.sh` can pass its own Docker check under the shim. With `podman-docker`, `docker --version` prints `podman version 5.x`; `get.sh` reads the third field as the Docker major version, sees `5 < 24` and dies with `Docker 24+ required`. On top of that, `install.yml` runs `get.sh` as root while `upgrade.yml` runs `upgrade.sh` as the service user, so file ownership does not line up until the installer owns the runtime switch. `olakai_allow_preview_runtime: true` skips the final `fail` for installer development only. When the installer ships the flag, pass it with `olakai_get_sh_extra_args: ["--runtime=podman-rootless"]`; the OS preparation is already what it expects.

The default `olakai_subid_start` is 524288, not 100000. `useradd` gives the first interactive user (`ubuntu`, `opc`, `ec2-user`) the range `100000:65536`, and two users must never share a subordinate range. The role reads `/etc/subuid` and `/etc/subgid` and fails, naming the owner, when the chosen range overlaps any existing entry.

## Idempotency

Run the install playbook twice and the second run reports zero changes. The guard is `<olakai_install_dir>/.env`: `get.sh` writes it on success, and the role does not download or run `get.sh` while it exists. The second run prints `.env exists: Olakai is already installed here` and touches nothing else; every task that reads the installer's output runs only when `get.sh` actually ran. OS preparation tasks (packages, users, sysctl, firewall rules) are native Ansible modules and converge on their own.

A failed install keeps its partial state in the install directory, as `get.sh` does. Fix the cause and run the playbook again. `get.sh` never overwrites an existing `.env`.

To re-install on purpose (for example after `.env` was removed), remember that the install directory is not empty: `.olakai-setup-url` and the extracted bundle are still there. `get.sh` refuses a non-empty directory unless `olakai_force_overlay: true` (`--force-overlay`), or you empty the directory first.

## What the playbook does not do

- It does not re-implement the installer. The license handshake, the bundle download, the SHA256 and cosign checks, and the compose bring-up stay inside `get.sh` and `upgrade.sh`.
- It does not migrate data between runtimes. Switching an existing Docker install to Podman (or back) is a manual procedure with Olakai support.
- It does not install or enable a firewall. It only adds rules to firewalld or ufw when one is already there.
- It does not manage DNS, TLS certificates, load balancers, or cloud security groups.
- It does not run backups or restores. Use the bundle's own `backup.sh` and `restore.sh`.

For the installer itself, see Olakai's on-prem install documentation.

## Security notes

- `get.sh` is fetched over HTTPS and is not itself signed. The bundle it downloads is cosign-verified (keyless, against the publisher's GitHub workflow identity), and `upgrade.sh` comes from that verified bundle. Pin `olakai_get_sh_sha256` to the checksum of the `get.sh` you reviewed; the download then fails on any mismatch.
- The license key is single-use. The relay rejects a second handshake with HTTP 409; the role reports that with a hint to contact Olakai sales.
- The license key reaches `get.sh` through the `OLAKAI_LICENSE_KEY` environment variable, never as a command-line argument, so it is not visible in `ps` or in the Ansible task line.
- The `get.sh` task runs with `no_log: true`. Ansible never records its command line or output. On failure a separate task prints the last lines of stderr with the license key replaced by `[redacted]`, and clears that text from host vars afterwards.
- The `OLAKAI_SETUP_URL` line is a one-time admin bootstrap token. The role writes it to `<olakai_install_dir>/.olakai-setup-url` (mode `0600`, owner `root`, or the service user under rootless Podman) and prints only the path. The URL is never stored in a fact, because a fact cache (jsonfile, redis, AWX) can persist `set_fact` values even with `no_log`. It appears in the play output, and so in the job log, only with `olakai_print_setup_url: true`. Delete the file after the first login.
- `get.sh` and the get.docker.com script are downloaded into a root-only `mkdtemp` directory, not a fixed path under `/tmp`, so another local user cannot swap the file between download and run.
- Docker auto-install runs the get.docker.com script as root. That script is not signature-verified; it extends the install's trust boundary to docker.com's HTTPS-served script. The role prints the same warning `get.sh` prints. The conservative option is `olakai_auto_install_docker: false` and Docker Engine 24+ with Compose v2 installed from Docker's signed package repositories before you run the playbook.
- Keep `olakai_license_key` in Ansible Vault. `group_vars/olakai.yml` is git-ignored in this repository.
- `olakai_skip_verify` disables all cosign verification. Use it only for air-gapped installs where you verified the bundle out of band.
- Under rootless Podman, `net.ipv4.ip_unprivileged_port_start=80` is a host-wide setting: every unprivileged process on the host may then bind ports 80 to 1023, not only Caddy. Once the installer supports it, the alternative is Caddy on 8080/8443 behind a customer load balancer with `olakai_unprivileged_port_start` left unset (`""`), which leaves the kernel default of 1024 in place.
- Under rootless Podman, in-container root maps to the unprivileged `olakai` user on the host. The support-bundle sidecar's Podman socket mount grants that user's rights only, not host root.

## Variables

All variables, with comments, are in `roles/olakai/defaults/main.yml`. The ones you normally set are in `group_vars/olakai.example.yml`.

## Development

```bash
pip install ansible-core ansible-lint yamllint
ansible-galaxy collection install --no-cache -r requirements.yml
yamllint --strict .
ansible-lint --offline
ansible-playbook --syntax-check -i inventory.example.ini install.yml
ansible-playbook --syntax-check -i inventory.example.ini upgrade.yml
```

CI runs the same commands on every pull request, with pinned tool versions, on ansible-core 2.15 (the declared floor) and on the current release.

## License

Apache-2.0. See `LICENSE`.
