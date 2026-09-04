# olakai-ansible

Ansible playbooks to install and upgrade [Olakai](https://olakai.ai) on-prem on one Linux host.

The role is a thin wrapper. Ansible checks the host, installs Docker when that is the chosen runtime, opens the firewall ports, and runs the Olakai scripts. `get.sh` and `upgrade.sh` do the trust-bearing work: license handshake, bundle download, SHA256 and cosign verification, `.env` generation, the container bring-up, and under rootless Podman the whole host preparation. This repository contains no product code.

## Installer compatibility (read first)

> **This role requires `get.sh` 2026.09.03 or later; 2026.09.03.2 or later for `olakai_runtime: podman-rootless`; 2026.09.04 or later for `olakai_env_overrides` and the private CA. `get.olakai.ai` serves 2026.09.04, so the default `olakai_get_sh_url` meets every floor.** The role relies on the `OLAKAI_LICENSE_KEY` environment variable, the `--force-overlay` flag, the `OLAKAI_SETUP_URL=` line on stdout, the `--runtime=podman-rootless` and `--service-user=` flags, and the `--env-file=` and `--private-ca=` flags. To review the role against a `get.sh` that `get.olakai.ai` does not serve yet, point `olakai_get_sh_url` at a copy (an HTTPS URL, or a `file://` path on the target host) and pin it with `olakai_get_sh_sha256`.

| Role version | Minimum get.sh | Minimum bundle | Notes |
| --- | --- | --- | --- |
| 0.2.1 (unreleased) | as 0.2.0 | 1.5.x for docker; 1.5.8 for `olakai_env_overrides` / private CA; 1.5.10 for `podman-rootless` | Bundle floors checked in preflight. Env values single-quoted. Owned keys aligned with `get.sh` (2026.09.04.1 adds `OLAKAI_SKIP_VERIFY`, `OLAKAI_VERIFY_IMAGES`). |
| 0.2.0 (unreleased) | 2026.09.03.2 for `podman-rootless` (`--runtime=`, `--service-user=`); 2026.09.04 for `olakai_env_overrides` / private CA (`--env-file=`, `--private-ca=`) | 1.5.x for docker; 1.5.10 for `podman-rootless` | Rootless Podman delegated to the installer. Managed-state overrides. |
| 0.1.0 (unreleased) | 2026.09.03 (`OLAKAI_LICENSE_KEY`, `--force-overlay`, `OLAKAI_SETUP_URL=` line) | 1.5.x (`upgrade.sh --yes --to=`) | Docker runtime only. |

What the role expects from `get.sh`:

1. `OLAKAI_LICENSE_KEY` in the environment is accepted instead of `--license-key=` (the flag wins when both are set).
2. `--force-overlay` (or `OLAKAI_FORCE_OVERLAY=true`) proceeds when the install directory is not empty.
3. A single line `OLAKAI_SETUP_URL=<url>` on stdout after a successful install. The line is present but empty when `get.sh` could not read the URL within 30 seconds or when `--quiet` is passed.
4. `--runtime=podman-rootless --service-user=<name>` prepares the host for rootless Podman and runs the bundle as that user (2026.09.03.2 or later). The docker runtime is `get.sh`'s default and the role does not pass `--runtime=docker`, so docker installs keep working with 2026.09.03.
5. `--env-file=<path>` merges a `KEY=VALUE` file into `.env` before the first bring-up, and `--private-ca=<path>` installs a PEM CA at `<install_dir>/certs/private-ca.pem` and activates the private-CA overlay (2026.09.04 or later).

With an older `get.sh`: (1) the run fails with `missing required input 'LICENSE_KEY'`; (2) a non-empty install directory without `.env` aborts, so empty the directory first; (3) the role prints the `docker compose logs` command to retrieve the setup URL instead of writing the URL file; (4) and (5) fail with `unknown option`.

```yaml
# group_vars/olakai.yml: review against a pre-release get.sh
olakai_get_sh_url: https://example.internal/olakai/get.sh
olakai_get_sh_sha256: "<sha256 of the reviewed get.sh>"
```

Tagged releases of this repository track the on-prem bundle versions they were tested against.

## What it does

`install.yml`:

1. Preflight: Linux, x86_64 or aarch64, 8 GB RAM, `python3`, a distribution the installer accepts. Under `podman-rootless`: systemd, cgroup v2, user namespaces, `runuser`, and a pinned `olakai_version` of 1.5.10 or later. With `olakai_env_overrides` or a private CA: a pinned `olakai_version` of 1.5.8 or later, under both runtimes, and every override value a string.
2. Runtime: installs Docker Engine + Compose v2 when the runtime is `docker`. Under `podman-rootless` the role installs nothing; `get.sh` prepares the host (see "Runtime choice").
3. Firewall: opens TCP 80 and 443 in firewalld or ufw when one of them is present.
4. Install: downloads `get.sh` into a root-only temporary directory, runs it as root, non-interactively, with the license key in its environment, writes the one-time setup URL to a `0600` file on the host, deletes the script and its input files.

`upgrade.yml`:

1. Preflight (same checks). With `olakai_runtime: auto` the runtime is read from `<install_dir>/.olakai-runtime`.
2. Runs `<install_dir>/upgrade.sh --yes --to=<version|latest>` and reports its verdict. Under rootless Podman it runs as the service user.

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
  - `github.com` and `objects.githubusercontent.com` (the cosign binary that `get.sh` bootstraps; the Docker Compose binary the role downloads for Amazon Linux; the `docker-compose` provider `get.sh` downloads under rootless Podman);
  - Sigstore, for online cosign verification: `rekor.sigstore.dev`, `fulcio.sigstore.dev`, `tuf-repo-cdn.sigstore.dev`. Not needed when `olakai_rekor_bundle` points at an offline bundle;
  - `get.docker.com` and `download.docker.com` when Docker is auto-installed (`olakai_auto_install_docker: true`);
  - the distribution's package repositories under rootless Podman (`get.sh` installs Podman with `dnf` or `apt-get`).
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

With `olakai_runtime: auto` (the default) the upgrade playbook reads `<install_dir>/.olakai-runtime`, the marker `get.sh` writes under rootless Podman, so you do not repeat the runtime. Under rootless Podman it runs `upgrade.sh` as the service user, with that user's `XDG_RUNTIME_DIR` and D-Bus address, which `systemctl --user` and the Podman socket lookup key on and which a plain `runuser` shell does not set. The full form, the one the role prints and the one to paste on the host by hand:

```bash
cd /opt/olakai-onprem && sudo runuser -u olakai -- env XDG_RUNTIME_DIR=/run/user/$(id -u olakai) DBUS_SESSION_BUS_ADDRESS=unix:path=/run/user/$(id -u olakai)/bus ./upgrade.sh --to=X.Y.Z
```

The bundle refuses a root-run upgrade of a rootless deployment. An explicit `olakai_runtime` that contradicts the marker fails before `upgrade.sh` runs.

Ansible becomes the unprivileged service user for that one task (`become_user: <olakai_service_user>`). Three things must hold:

- **sudoers.** Ansible runs `sudo -u olakai`, so the sudoers policy of the connection user must allow that target user, not only root. `ALL=(ALL) NOPASSWD: ALL` does; a policy limited to `(root)` does not. Or connect as root.
- **The module hand-over.** Ansible writes its module to a temporary file as the connection user and hands it to the service user with `setfacl`, present on the RHEL family (package `acl`) and absent by default on Debian and Ubuntu. Pipelining sends the module over the SSH channel instead, so no file changes hands. The `ansible.cfg` in this repository sets `pipelining = True` under `[ssh_connection]` (read when you run the playbooks from the repository root; `ANSIBLE_PIPELINING=true` in the environment does the same) and needs sudoers without `requiretty`, the default on current distributions.
- **PATH.** The role passes `PATH=/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin` to `upgrade.sh` under both runtimes: `sudo` on the RHEL family resets PATH to `secure_path`, which lacks `/usr/local/bin`, where `get.sh` installed the `docker-compose` provider. `get.sh` passes PATH to `runuser` for the same reason.

## Runtime choice

`olakai_runtime` selects the container runtime.

| Value | Behaviour |
| --- | --- |
| `auto` (default) | Fresh install: `docker` on every family (Ubuntu, Debian, RHEL, Rocky, AlmaLinux, CentOS Stream, Oracle Linux, Amazon Linux). Upgrade: the runtime recorded in `<install_dir>/.olakai-runtime` (no marker means `docker`). |
| `docker` | Docker Engine 24+ with Compose v2. Installed from `get.docker.com` when missing and `olakai_auto_install_docker` is true (Amazon Linux uses the distro package; Oracle Linux is left to `get.sh`, which installs from Docker's CentOS package repository because `get.docker.com` rejects `ID=ol`). The stack runs under the Docker daemon, as root. |
| `podman-rootless` | Explicit opt-in. The role passes `--runtime=podman-rootless --service-user=<olakai_service_user>` to `get.sh`, which prepares the host and runs the bundle as an unprivileged service user. Needs `get.sh` 2026.09.03.2 or later and bundle 1.5.10 or later. |

On Oracle Linux the role does not run `get.docker.com` (it rejects `ID=ol`). It delegates the Docker CE install to `get.sh --auto-install-docker`, which installs from Docker's CentOS package repository over HTTPS. Docker CE on Oracle Linux is community-supported: Docker does not list Oracle Linux as a supported platform for Docker Engine, and the CentOS packages are used as-is. Oracle's supported runtime is Podman; see below.

### Rootless Podman

The role does not prepare the host itself. `get.sh --runtime=podman-rootless` does, as root, and the role only checks the prerequisites first (systemd, cgroup v2, unprivileged user namespaces, `runuser`) so a host that cannot run rootless Podman fails with an Ansible error before anything is downloaded. What `get.sh` changes on the host, in order:

1. Installs Podman 5 with the `podman-docker` shim from the distribution's signed repositories (`dnf`, or `apt-get` after checking that the candidate version is 5 or later; Ubuntu 24.04 ships Podman 4.9 and is refused). It refuses a host where Docker Engine is installed next to Podman.
2. Installs `docker-compose` v5.5.1 at `/usr/local/bin/docker-compose`, pinned by SHA256, and names it as the compose provider in `/etc/containers/containers.conf.d/olakai-compose.conf`.
3. Creates the service user (`useradd -r -m -s /bin/bash`, default name `olakai`, from `olakai_service_user`).
4. Assigns the service user a free 65536-wide subordinate id range in `/etc/subuid` and `/etc/subgid`, starting at 524288 or above, after checking every existing entry for overlap. Two users that share a range can read and write each other's container files, which is why `get.sh` owns this step.
5. Enables systemd linger for the service user, so its user units survive logout and reboot.
6. Writes `/etc/sysctl.d/90-olakai.conf` with `net.ipv4.ip_unprivileged_port_start=80`, so the service user can bind ports 80 and 443 for Caddy.
7. Enables the user-scope `podman.socket` and `podman-restart.service` of the service user.
8. After the license handshake and the bundle extraction: writes the `.olakai-runtime` marker, activates the Podman compose overlay, hands `<install_dir>` to the service user (owner, mode `0750`), and runs the bundle's `install.sh` through `runuser -u <user>`.

`get.sh` refuses to convert a deployment in place: an existing `.olakai-runtime` marker (or a `data/` tree from a Docker install that predates the marker) naming another runtime stops the run before the license key is consumed, and so does an install directory owned by another user than `olakai_service_user`.

Bundle floor: rootless Podman needs bundle 1.5.10 or later. 1.5.9 introduced the Podman overlay but ships a support-bundle sidecar defect under rootless Podman, and earlier bundles have no Podman support. `get.sh` checks this after the handshake, when the license key is already consumed, so the role's preflight refuses a pinned `olakai_version` older than 1.5.10 first. Leave `olakai_version` empty for the latest release.

Qualification: rootless Podman is qualified on Oracle Linux 9.8 (Podman 5.8.2). Other RHEL 9 family distributions ship the same Podman 5 and are accepted, not qualified. Debian and Ubuntu are not qualified; `get.sh` prints a warning there.

Later bundle scripts (`upgrade.sh`, `support-bundle.sh`, `scripts/restore.sh`) must run as the service user, from the install directory: `cd /opt/olakai-onprem && sudo runuser -u olakai -- ./upgrade.sh --to=X.Y.Z`. `upgrade.yml` does this for you. Files you add later (`certs/private-ca.pem`, overlays) must be readable by the service user.

Security: under rootless Podman a container escape yields the service user, not root. In-container root maps to the service user's subordinate uid range on the host. `net.ipv4.ip_unprivileged_port_start=80` is a host-wide setting: every unprivileged process on the host may then bind ports 80 to 1023, not only Caddy.

## Managed state (customer-managed PostgreSQL, Redis, object storage)

The bundled PostgreSQL, MinIO and Redis containers are convenient, not required. To run the stateless Olakai workloads only (`app`, `worker`, the one-shot `migrate` and `bootstrap` jobs, the support-bundle sidecar) against services you already operate, put the connection settings in `olakai_env_overrides`. The role writes them to a root-owned `0600` temporary file next to `get.sh` (with `no_log`), passes `--env-file=<path>`, and removes the file after the run. `get.sh` merges the keys into `.env` before the first bring-up; the bundle's `install.sh` then detects the topology, preflights the external settings, and skips the bundled containers and their `./data` directories. Needs `get.sh` 2026.09.04 or later and bundle 1.5.8 or later. Any subset works: managed PostgreSQL alone, managed object storage alone, or all three.

```yaml
# group_vars/olakai.yml (keep the credentials in Ansible Vault)
olakai_env_overrides:
  DATABASE_URL: 'postgresql://olakai:PASSWORD@db.example.com:5432/olakai?sslmode=verify-full'
  REDIS_URL: 'rediss://:PASSWORD@cache.example.com:6380'
  OBJECT_STORAGE_KIND: 's3'                       # s3 or minio only
  OBJECT_STORAGE_ENDPOINT: 'https://s3.us-east-1.amazonaws.com'
  OBJECT_STORAGE_REGION: 'us-east-1'
  OBJECT_STORAGE_BUCKET: 'customer-olakai-documents'
  OBJECT_STORAGE_ACCESS_KEY: 'AKIA...'
  OBJECT_STORAGE_SECRET_KEY: '...'
  # OBJECT_STORAGE_FORCE_PATH_STYLE: 'true'       # required for MinIO-compatible and OCI endpoints
```

Keys are upper-case identifiers. Every value is written single-quoted, one per line (`KEY='value'`, an embedded `'` as `'\''`). `.env` is read three ways: `install.sh` sources it with bash, compose interpolates it, and the bundle's env reader parses it back. Only a single-quoted value is the same literal to all three; an unquoted `pa$w0rd` would source as `pa`. So a value may contain `$`, spaces, `#` and quotes as they are. Three rules on the YAML side:

- **Quote every value in YAML.** An unquoted `true`, `1` or `0600` becomes a boolean or a number and would be written as Python spells it (`True`). The preflight refuses a value that is not a string and names the key.
- **Mark a value that contains `{{` or `{%` as `!unsafe`** (`SOME_KEY: !unsafe 'a{{b'`), so Ansible does not template it.
- **Keep the credentials in Ansible Vault.** A `!vault` value in place of the quoted string is accepted and decrypted when the file is written.

The keys `get.sh` derives from its own flags and the license handshake are refused, by the role's preflight first and by `get.sh --env-file` (which names the flag that sets each one): `OLAKAI_DOMAIN`, `AUTH_URL`, `OLAKAI_ADMIN_EMAIL`, `OLAKAI_EMAIL_RELAY_KEY`, `OLAKAI_VERSION`, and from `get.sh` 2026.09.04.1 `OLAKAI_SKIP_VERIFY` and `OLAKAI_VERIFY_IMAGES` (`olakai_skip_verify` and `olakai_verify_images` are the role's switches for those). The generated secrets are accepted: `NEXTAUTH_SECRET`, `AUTH_SECRET`, `DEFAULT_ENCRYPTION_KEY`, `NEXT_SERVER_ACTIONS_ENCRYPTION_KEY`, `ENCRYPTION_SALT`, `POSTGRES_PASSWORD`, `MINIO_ROOT_PASSWORD`, `REDIS_PASSWORD`, `OLAKAI_SUPPORT_BUNDLE_SECRET`. `get.sh` generates fresh values for them and the `--env-file` merge, which runs last, replaces them. **Supply these only when restoring a backup**: data encrypted with `DEFAULT_ENCRYPTION_KEY` and `ENCRYPTION_SALT` can only be read with the keys of the deployment that wrote it, so a restore into managed PostgreSQL needs the values from the original `.env`. On a fresh install leave them out.

Bundle floor: the state-topology detection and the private-CA overlay first shipped in bundle 1.5.8. An older bundle starts the bundled stateful containers in spite of the overrides, with no error, and `get.sh` refuses `--private-ca` only after the handshake, when the single-use license key is already consumed. So the preflight refuses a pinned `olakai_version` older than 1.5.8 when `olakai_env_overrides` or a private CA is set, under both runtimes. A pre-release is older than its base version: `v1.5.8-rc1` fails the floor, `v1.5.9-rc1` passes it. Leave `olakai_version` empty for the latest release.

Two things to get right, from the bundle README. The failure mode is misleading: `migrate` succeeds and only then the app fails with a TLS error that looks like a database problem.

- **Use DNS names, not IP addresses.** The PostgreSQL and Redis clients verify the hostname against the server certificate. A bare IP fails verification even when the CA is trusted and even when the certificate carries an IP SAN.
- **Use `sslmode=verify-full` and supply the CA when it is not a public one.** The app connects through node-postgres and the `rediss://` client, which verify against Node's built-in roots, not the VM's system trust store; `update-ca-certificates` on the host does nothing for them. This affects corporate internal CAs and provider CAs such as Amazon RDS's `rds-ca-*`. Pass the CA in PEM form as `olakai_private_ca_pem` (text) or `olakai_private_ca_file` (a path on the control node). The file must be PEM: a `-----BEGIN CERTIFICATE-----` header within the first 4096 bytes, the rule `get.sh` applies, so a text preamble such as an `openssl x509 -text` dump before the first block is fine; DER, PKCS#7 and PKCS#12 are not (`openssl x509 -inform der -in ca.crt -out ca.pem`). The role stages a copy with a trailing newline (the file lookup strips it) and passes it as `--private-ca=<path>`; `get.sh` installs it at `<install_dir>/certs/private-ca.pem` and activates the private-CA compose overlay, in that order (the overlay before the file would hand every service a directory where a CA bundle belongs). Only if you cannot supply the CA, fall back to `?uselibpqcompat=true&sslmode=require`: encrypted, but the certificate is not verified, and the nightly `pg_dump` backup rejects that URL.

```yaml
olakai_private_ca_file: files/rds-ca.pem
```

On a managed-state deployment the database, the object storage bucket, Redis, their backups, private endpoints, DNS and TLS are customer-owned. The bundle's `backup.sh` covers the bundled containers only.

## Idempotency

Run the install playbook twice and the second run reports zero changes. The guard is `<olakai_install_dir>/.env`: `get.sh` writes it on success, and the role does not download or run `get.sh` while it exists. The second run prints `.env exists: Olakai is already installed here` and touches nothing else; every task that reads the installer's output runs only when `get.sh` actually ran. The Docker install and the firewall rules are native Ansible modules and converge on their own. Under rootless Podman, `get.sh` itself is re-run safe (existing packages, user, ranges and units are detected, not recreated).

A failed install keeps its partial state in the install directory, as `get.sh` does. Fix the cause and run the playbook again. `get.sh` never overwrites an existing `.env`.

To re-install on purpose (for example after `.env` was removed), remember that the install directory is not empty: `.olakai-setup-url` and the extracted bundle are still there. `get.sh` refuses a non-empty directory unless `olakai_force_overlay: true` (`--force-overlay`), or you empty the directory first. Under rootless Podman a re-run must name the same `olakai_service_user`.

## What the playbook does not do

- It does not re-implement the installer. The license handshake, the bundle download, the SHA256 and cosign checks, the compose bring-up, and the rootless Podman host preparation stay inside `get.sh` and `upgrade.sh`.
- It does not migrate data between runtimes. Switching an existing Docker install to Podman (or back) is a backup, a fresh install on the new runtime, and a restore.
- It does not install or enable a firewall. It only adds rules to firewalld or ufw when one is already there.
- It does not manage DNS, TLS certificates, load balancers, or cloud security groups.
- It does not run backups or restores. Use the bundle's own `backup.sh` and `restore.sh`.

For the installer itself, see Olakai's on-prem install documentation.

## Security notes

- `get.sh` is fetched over HTTPS and is not itself signed. The bundle it downloads is cosign-verified (keyless, against the publisher's GitHub workflow identity), and `upgrade.sh` comes from that verified bundle. Pin `olakai_get_sh_sha256` to the checksum of the `get.sh` you reviewed; the download then fails on any mismatch.
- The license key is single-use. The relay rejects a second handshake with HTTP 409; the role reports that with a hint to contact Olakai sales.
- The license key reaches `get.sh` through the `OLAKAI_LICENSE_KEY` environment variable, never as a command-line argument, so it is not visible in `ps` or in the Ansible task line.
- The `get.sh` task runs with `no_log: true`. Ansible never records its command line or output. On failure a separate task prints the last lines of stderr with the license key and every `olakai_env_overrides` value replaced by `[redacted]` (longest value first, so a value that contains another one is covered whole), and clears that text from host vars afterwards, on the failure path as much as on success.
- `olakai_env_overrides` holds database and object storage credentials. The task that writes them runs with `no_log: true`; the file is root-owned, mode `0600`, inside the root-only temporary directory, and is removed with `get.sh` after the run. `get.sh` copies the values into `<install_dir>/.env` (mode `0600`). Keep the dict in Ansible Vault.
- The `OLAKAI_SETUP_URL` line is a one-time admin bootstrap token. The role writes it to `<olakai_install_dir>/.olakai-setup-url` (mode `0600`, owner `root`, or the service user under rootless Podman) and prints only the path. The URL is never stored in a fact: a fact cache (jsonfile, redis, AWX) stores `set_fact` values only with `cacheable: true`, which the role never sets, and the design keeps the URL out of host vars anyway so a later `cacheable`, a callback plugin or a `debug` of `hostvars` cannot leak it. It appears in the play output, and so in the job log, only with `olakai_print_setup_url: true`. Delete the file after the first login.
- `get.sh`, its input files, and the get.docker.com script are downloaded into a root-only `mkdtemp` directory, not a fixed path under `/tmp`, so another local user cannot swap a file between download and run.
- Docker auto-install runs the get.docker.com script as root. That script is not signature-verified; it extends the install's trust boundary to docker.com's HTTPS-served script. The role prints the same warning `get.sh` prints. The conservative option is `olakai_auto_install_docker: false` and Docker Engine 24+ with Compose v2 installed from Docker's signed package repositories before you run the playbook. Under rootless Podman there is no such step: `get.sh` installs Podman from the distribution's signed repositories.
- Keep `olakai_license_key` in Ansible Vault. `group_vars/olakai.yml` is git-ignored in this repository.
- `olakai_skip_verify` disables all cosign verification. Use it only for air-gapped installs where you verified the bundle out of band.
- Under rootless Podman a container escape yields the service user, not root. In-container root maps to the unprivileged service user on the host. The support-bundle sidecar's Podman socket mount grants that user's rights only, not host root. `net.ipv4.ip_unprivileged_port_start=80` is host-wide: every unprivileged process on the host may then bind ports 80 to 1023.

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

CI runs the same commands on every pull request, with pinned tool versions, on ansible-core 2.15 (the declared floor) and on the current release. The `ansible.cfg` at the repository root only enables SSH pipelining (see "Upgrades").

## License

Apache-2.0. See `LICENSE`.
