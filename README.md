# olakai-ansible

Ansible playbooks to install and upgrade [Olakai](https://olakai.ai) on-prem on one Linux host.

The role is a thin wrapper. Ansible prepares the operating system (container runtime, service user, firewall ports). The signed Olakai scripts `get.sh` and `upgrade.sh` do the trust-bearing work: license handshake, bundle download, SHA256 and cosign verification, `.env` generation, and the container bring-up. This repository contains no product code.

## What it does

`install.yml`:

1. Preflight: Linux, x86_64 or aarch64, 8 GB RAM, `python3`, supported distribution.
2. Runtime: installs Docker Engine + Compose v2, or prepares rootless Podman (packages, service user, subordinate ids, linger, sysctl for ports 80/443, user-scope Podman socket).
3. Firewall: opens TCP 80 and 443 in firewalld or ufw when one of them is present.
4. Install: downloads `get.sh`, runs it non-interactively with the license key in its environment, writes the one-time setup URL to a `0600` file on the host, deletes the script.

`upgrade.yml`:

1. Preflight (same checks).
2. Runs `<install_dir>/upgrade.sh --yes --to=<version|latest>` and reports its verdict.

## Requirements

Control node:

- ansible-core 2.15 or newer.
- Collections `ansible.posix` and `community.general` (`ansible-galaxy collection install --no-cache -r requirements.yml`).

Target host:

- One VM, 8 GB RAM, 80 GB free disk, x86_64 or aarch64.
- A supported OS: Ubuntu 22.04 / 24.04, Debian 12, RHEL / Rocky / AlmaLinux / CentOS Stream 9 or 10, Oracle Linux 9 or 10, Amazon Linux 2023. The authoritative list is the Olakai platform support matrix: `https://docs.olakai.ai/on-prem/platform-support` (placeholder, page in progress).
- SSH access with password-less sudo, and `python3` on the host.
- A public DNS A record for `olakai_domain` that points at the VM before you run the install (Let's Encrypt needs it).
- Outbound HTTPS to `relay.olakai.ai` (license handshake, bundle download), `get.olakai.ai` (installer), `public.ecr.aws` (container images), and `get.docker.com` when Docker is auto-installed.
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
| `auto` (default) | `docker` on Ubuntu, Debian and Amazon Linux. `podman-rootless` on RHEL, Rocky, AlmaLinux, CentOS Stream and Oracle Linux. |
| `docker` | Docker Engine 24+ with Compose v2. Installed from `get.docker.com` when missing and `olakai_auto_install_docker` is true. |
| `podman-rootless` | Podman with the `podman-docker` shim, a standalone Compose v2 binary, an unprivileged `olakai` service user, and `net.ipv4.ip_unprivileged_port_start=80`. |

Preview status of `podman-rootless`: this role version prepares the OS for rootless Podman, but `get.sh` does not accept a `--runtime=` flag yet. Until the installer ships it, `get.sh` runs as root against the `podman-docker` shim, which brings the stack up under rootful Podman. When the flag lands, pass it with `olakai_get_sh_extra_args: ["--runtime=podman-rootless"]`; the OS preparation is already what it expects. The compatibility table below tracks this.

## Idempotency

Run the install playbook twice and the second run reports zero changes. The guard is `<olakai_install_dir>/.env`: `get.sh` writes it on success, and the role does not download or run `get.sh` while it exists. The second run prints `.env exists: Olakai is already installed here` and touches nothing else; every task that reads the installer's output runs only when `get.sh` actually ran. OS preparation tasks (packages, users, sysctl, firewall rules) are native Ansible modules and converge on their own.

A failed install keeps its partial state in the install directory, as `get.sh` does. Fix the cause and run the playbook again. `get.sh` never overwrites an existing `.env`.

## What the playbook does not do

- It does not re-implement the installer. The license handshake, the bundle download, the SHA256 and cosign checks, and the compose bring-up stay inside the signed `get.sh` and `upgrade.sh`.
- It does not migrate data between runtimes. Switching an existing Docker install to Podman (or back) is a manual procedure with Olakai support.
- It does not install or enable a firewall. It only adds rules to firewalld or ufw when one is already there.
- It does not manage DNS, TLS certificates, load balancers, or cloud security groups.
- It does not run backups or restores. Use the bundle's own `backup.sh` and `restore.sh`.

## Compatibility

| Role version | Minimum get.sh | Minimum bundle | Notes |
| --- | --- | --- | --- |
| 0.1.0 (unreleased) | 2026.09 (`OLAKAI_LICENSE_KEY`, `--force-overlay`, `OLAKAI_SETUP_URL=` line) | 1.5.x (`upgrade.sh --yes --to=`) | Written against an installer change that is in progress. See below. |

This role expects three behaviours in `get.sh` that are being added in parallel with the role:

1. `OLAKAI_LICENSE_KEY` in the environment is accepted instead of `--license-key=` (the flag wins when both are set).
2. `--force-overlay` (or `OLAKAI_FORCE_OVERLAY=true`) proceeds when the install directory is not empty.
3. A single line `OLAKAI_SETUP_URL=<url>` on stdout after a successful install.

With an older `get.sh`: (1) the run fails with `missing required input 'LICENSE_KEY'`; (2) a non-empty install directory without `.env` aborts, so empty the directory first; (3) the role prints the `docker compose logs` command to retrieve the setup URL instead of writing the URL file.

Tagged releases of this repository track the on-prem bundle versions they were tested against.

## Security notes

- The license key is single-use. The relay rejects a second handshake with HTTP 409; the role reports that with a hint to contact Olakai sales.
- The license key reaches `get.sh` through the `OLAKAI_LICENSE_KEY` environment variable, never as a command-line argument, so it is not visible in `ps` or in the Ansible task line.
- The `get.sh` task runs with `no_log: true`. Ansible never records its command line or output. On failure a separate task prints the last lines of stderr with the license key replaced by `[redacted]`.
- The `OLAKAI_SETUP_URL` line is a one-time admin bootstrap token. The role writes it to `<olakai_install_dir>/.olakai-setup-url` (mode `0600`, owner `root`, or the service user under rootless Podman) and prints only the path. It appears in the play output, and so in the job log, only with `olakai_print_setup_url: true`. Delete the file after the first login.
- Keep `olakai_license_key` in Ansible Vault. `group_vars/olakai.yml` is git-ignored in this repository.
- Set `olakai_get_sh_sha256` to pin the installer once Olakai publishes checksums for `get.sh` releases.
- `olakai_skip_verify` disables all cosign verification. Use it only for air-gapped installs where you verified the bundle out of band.
- Under rootless Podman, in-container root maps to the unprivileged `olakai` user on the host. The support-bundle sidecar's Podman socket mount grants that user's rights only, not host root.

## Variables

All variables, with comments, are in `roles/olakai/defaults/main.yml`. The ones you normally set are in `group_vars/olakai.example.yml`.

## Development

```bash
pip install ansible-core ansible-lint yamllint
ansible-galaxy collection install --no-cache -r requirements.yml
yamllint --strict .
ansible-lint
ansible-playbook --syntax-check -i inventory.example.ini install.yml
```

CI runs the same three commands on every pull request.

## License

Apache-2.0. See `LICENSE`.
