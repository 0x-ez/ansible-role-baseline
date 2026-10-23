# baseline

An Ansible role that applies a "good enough" security baseline to a fresh
Ubuntu 22.04 (jammy) or 24.04 (noble) server. It follows a personal checklist
and is not a CIS benchmark.

- Ansible core 2.15+, collections `community.general` and `ansible.posix`
- Idempotent (checked by Molecule on both releases)
- Passes `ansible-lint` with the `production` profile

## Contents

- [Quick start](#quick-start)
- [What each task does](#what-each-task-does)
- [Variables](#variables)
- [Example inventory and group_vars](#example-inventory-and-group_vars)
- [First run vs later runs](#first-run-vs-later-runs)
- [Safety: root lockout](#safety-root-lockout)
- [Safety: Docker bypasses UFW](#safety-docker-bypasses-ufw)
- [Running a single tag](#running-a-single-tag)
- [Testing](#testing)

## Quick start

```bash
ansible-galaxy collection install -r requirements.yml
cp inventory.example.ini inventory.ini   # edit hosts
ansible-playbook site.yml -u root        # first run, see below
```

`ansible.cfg` points at `./inventory.ini` and `./roles`.

## What each task does

Tasks run in this order. Each file has a tag with the same name, and each
feature has a `baseline_<feature>_enabled` toggle.

| Tag | What it does |
|---|---|
| `updates` | Writes `/etc/needrestart/conf.d/90-baseline.conf` (`$nrconf{restart} = 'a'`, kernel hints off) **before** upgrading so apt never stops at a prompt. Then runs `apt update` + `dist-upgrade` (with `cache_valid_time`), installs `vim curl htop` + `baseline_extra_packages`, and reboots if `/var/run/reboot-required` exists and `baseline_reboot_if_required: true`. |
| `docker` | Downloads Docker's key to `/etc/apt/keyrings/docker.asc`, adds a deb822 source (`/etc/apt/sources.list.d/docker.sources`) for the host's architecture and codename, installs `docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin`, and enables the service. |
| `sysctl` | Writes `baseline_sysctl_settings` (default `vm.overcommit_memory = 1`) to `/etc/sysctl.d/99-baseline.conf` and applies them. |
| `users` | Creates `baseline_admin_user` with a locked password and bash, in `sudo` (and `docker` if Docker is enabled). Sudo rights go in `/etc/sudoers.d/90-baseline-<user>` (0440, checked with `visudo -cf`). `/etc/sudoers` itself is never edited. SSH keys come from `baseline_admin_ssh_keys`, or are copied from root's `authorized_keys` on the host if that list is empty. The role stops if it finds no key at all. |
| `ssh` | Writes `/etc/ssh/sshd_config.d/00-baseline.conf` (see below), checked with `sshd -t -f`. Checks the merged config with `sshd -t`, reloads sshd **right away** (handlers are flushed), then reads `sshd -T` to confirm the settings took effect. Handles port changes, including `ssh.socket` on 24.04. |
| `firewall` | UFW: allows SSH **first**, then `baseline_ufw_allowed`, sets the default policies (deny incoming, allow outgoing) and enables UFW last. Can also add the Docker/UFW fix (off by default). |
| `fail2ban` | Installs fail2ban and `python3-systemd`, and writes `/etc/fail2ban/jail.d/baseline.local` (sshd jail, `backend = systemd`, port = `baseline_ssh_port`). |
| `shm` | **Off by default.** Mounts `/dev/shm` as tmpfs with `defaults,noexec,nosuid,nodev` (`ansible.posix.mount`, `state: mounted`). |
| `tailscale` | **Off by default.** Adds the official Tailscale repo, installs it, enables `tailscaled`, and runs `tailscale up` if an auth key is set and the node is not already `Running`. Can also limit SSH to `tailscale0`. |

### The SSH drop-in

Ubuntu's `sshd_config` starts with `Include /etc/ssh/sshd_config.d/*.conf`, and
sshd keeps the **first** value it reads for each keyword. cloud-init ships
`50-cloud-init.conf` with `PasswordAuthentication yes`, so the file is named
`00-baseline.conf` to be read first. It sets:

```
Port <baseline_ssh_port>        # only when it is not 22
PermitRootLogin <baseline_ssh_permit_root_login>
PasswordAuthentication no
PubkeyAuthentication yes
PermitEmptyPasswords no
KbdInteractiveAuthentication no
```

You can add more directives with `baseline_sshd_extra_settings`. After the
reload, the role checks the effective config with `sshd -T` and fails if
another file or a `Match` block overrides these settings.

### Changing the SSH port

You can change `baseline_ssh_port` on a host that is already hardened. The role:

1. Opens the new port in UFW **before** touching sshd, so a host with UFW
   already enabled does not drop the reconnect.
2. Writes the drop-in, then applies the change:
   - **24.04 (socket activation):** `ssh.socket` owns the port.
     `sshd-socket-generator` reads `Port` from the sshd config on
     `systemctl daemon-reload`. The role runs a daemon-reload, restarts
     `ssh.socket`, then restarts `ssh.service`.
   - **22.04:** restarts `ssh.service`.
   Existing sessions survive both (`KillMode=process`).
3. Points the Ansible connection at the new port and runs
   `wait_for_connection`.
4. Updates the fail2ban jail and UFW rule to the new port.

The old port's UFW rule is **not** removed. Once the new port works, delete it
with `sudo ufw delete allow 22/tcp`. Also set `ansible_port` in your inventory.

## Variables

All variables are defined in [`roles/baseline/defaults/main.yml`](roles/baseline/defaults/main.yml).

### Toggles

| Variable | Default | Description |
|---|---|---|
| `baseline_updates_enabled` | `true` | Run the `updates` tasks |
| `baseline_docker_enabled` | `true` | Install Docker CE (also adds the admin user to `docker`) |
| `baseline_sysctl_enabled` | `true` | Apply sysctl settings |
| `baseline_users_enabled` | `true` | Create the admin user |
| `baseline_ssh_enabled` | `true` | Harden sshd |
| `baseline_firewall_enabled` | `true` | Configure and enable UFW |
| `baseline_fail2ban_enabled` | `true` | Install and configure fail2ban |
| `baseline_secure_shm_enabled` | `false` | Mount shared memory as noexec (can break software) |
| `baseline_tailscale_enabled` | `false` | Install and join Tailscale |
| `baseline_in_container` | auto-detected | Skips the docker, sysctl, firewall, shm and reboot steps. Detected from the `virtualization_*` facts |

### updates

| Variable | Default | Description |
|---|---|---|
| `baseline_apt_cache_valid_time` | `3600` | Seconds before the apt cache is refreshed again |
| `baseline_base_packages` | `[vim, curl, htop]` | Always installed |
| `baseline_extra_packages` | `[]` | Extra packages to install |
| `baseline_reboot_if_required` | `false` | Reboot if `/var/run/reboot-required` exists |
| `baseline_reboot_timeout` | `600` | Seconds to wait for the host to come back |

### docker

| Variable | Default | Description |
|---|---|---|
| `baseline_docker_packages` | docker-ce, docker-ce-cli, containerd.io, docker-buildx-plugin, docker-compose-plugin | Packages to install |
| `baseline_docker_ufw_fix_enabled` | `false` | Add the ufw-docker `DOCKER-USER` rules (see the safety section) |
| `baseline_docker_ufw_routes` | `[]` | Published container ports to allow when the fix is on (`ufw route allow`). Same item format as `baseline_ufw_allowed` |

### sysctl

| Variable | Default | Description |
|---|---|---|
| `baseline_sysctl_file` | `/etc/sysctl.d/99-baseline.conf` | Target file |
| `baseline_sysctl_settings` | `{vm.overcommit_memory: 1}` | Dict of key: value pairs |

### users

| Variable | Default | Description |
|---|---|---|
| `baseline_admin_user` | `nonroot` | Admin login name (must not be `root`) |
| `baseline_admin_shell` | `/bin/bash` | Login shell |
| `baseline_admin_nopasswd` | `true` | Passwordless sudo. Keep `true` unless you set a password yourself: the account password is locked |
| `baseline_admin_ssh_keys` | `[]` | Public keys. If empty, root's `authorized_keys` on the host is copied |
| `baseline_admin_ssh_keys_exclusive` | `false` | Remove keys that are not in the list |

### ssh

| Variable | Default | Description |
|---|---|---|
| `baseline_ssh_port` | `22` | sshd port, also used by UFW and fail2ban |
| `baseline_ssh_permit_root_login` | `prohibit-password` | `prohibit-password`, `no`, `yes` or `forced-commands-only` |
| `baseline_admin_login_verified` | `false` | Must be `true` before `PermitRootLogin no` is allowed |
| `baseline_ssh_verify_admin_from_controller` | `true` | With `PermitRootLogin no`, also test `ssh -o BatchMode=yes <admin>@<host>` from the controller |
| `baseline_sshd_extra_settings` | `{}` | Extra sshd directives, e.g. `{MaxAuthTries: 3}` |

### firewall

| Variable | Default | Description |
|---|---|---|
| `baseline_ufw_default_incoming` | `deny` | Default incoming policy |
| `baseline_ufw_default_outgoing` | `allow` | Default outgoing policy |
| `baseline_ufw_logging` | `low` | UFW logging level |
| `baseline_ufw_allowed` | 80/tcp, 443/tcp | Items: `{port, proto, from?, interface?, comment?}`. Commented redis/postgres examples are in defaults |

### fail2ban

| Variable | Default | Description |
|---|---|---|
| `baseline_fail2ban_maxretry` | `5` | Failures before a ban |
| `baseline_fail2ban_findtime` | `600` | Window in seconds |
| `baseline_fail2ban_bantime` | `600` | Ban length in seconds |
| `baseline_fail2ban_ignoreip` | `[]` | Added to `127.0.0.1/8 ::1` |

### shm

| Variable | Default | Description |
|---|---|---|
| `baseline_shm_path` | `/dev/shm` | Mount point (see the note below) |
| `baseline_shm_opts` | `defaults,noexec,nosuid,nodev` | Mount options |

`noexec` on shared memory can break software that runs code from it, such as
some JITs, Chromium/Electron, certain database or Java setups, and some
installers. Test before you enable it in production.

### tailscale

| Variable | Default | Description |
|---|---|---|
| `baseline_tailscale_authkey` | `""` | Auth key. Load it from Vault or the environment; it is never logged |
| `baseline_tailscale_args` | `[]` | Extra `tailscale up` args, e.g. `["--ssh", "--advertise-tags=tag:server"]` |
| `baseline_ssh_tailscale_only` | `false` | Allow SSH in UFW only on `tailscale0` |

**Lockout risk with `baseline_ssh_tailscale_only`.** Once the tunnel is up,
SSH from the public internet is blocked. If `tailscaled` stops, the node is
logged out, its key expires, or the tailnet ACLs change, you can only get back
in through your provider's console. To reduce the risk:

- The public SSH rule is removed only after `tailscale status` reports `Running`.
  Until then SSH stays reachable from anywhere.
- Turn off key expiry for the node in the Tailscale admin console.
- Before you enable this, make sure you have out-of-band access (VNC or serial
  console), and test `ssh <admin>@<tailscale-ip>`.

## Example inventory and group_vars

`inventory.ini`:

```ini
[servers]
web1 ansible_host=203.0.113.10
web2 ansible_host=203.0.113.11

[servers:vars]
ansible_python_interpreter=/usr/bin/python3
```

`group_vars/servers/main.yml`:

```yaml
baseline_admin_user: deploy
baseline_admin_ssh_keys:
  - "ssh-ed25519 AAAA... me@laptop"
baseline_extra_packages: [git, jq, unattended-upgrades]
baseline_ufw_allowed:
  - { port: "80", proto: tcp, comment: http }
  - { port: "443", proto: tcp, comment: https }
  - { port: "5432", proto: tcp, from: "10.0.0.0/8", comment: postgres }
baseline_fail2ban_ignoreip: ["198.51.100.7"]

baseline_tailscale_enabled: true
baseline_tailscale_authkey: "{{ vault_tailscale_authkey }}"
```

`group_vars/servers/vault.yml` (encrypt it with `ansible-vault encrypt`):

```yaml
vault_tailscale_authkey: tskey-auth-...
```

Or read it from the environment:
`baseline_tailscale_authkey: "{{ lookup('ansible.builtin.env', 'TS_AUTHKEY') }}"`.

## First run vs later runs

A fresh server usually has only root with your key. Use this sequence:

```bash
# 1. First run as root. PermitRootLogin stays at prohibit-password,
#    so root key login keeps working.
ansible-playbook site.yml -u root

# 2. Check the admin user by hand.
ssh nonroot@203.0.113.10 sudo -v

# 3. From now on, connect as the admin user.
ansible-playbook site.yml -u nonroot

# 4. Optional: disable root SSH entirely.
ansible-playbook site.yml -u nonroot \
  -e baseline_ssh_permit_root_login=no -e baseline_admin_login_verified=true
```

To keep step 4 permanent, set both variables in `group_vars` and
`ansible_user: nonroot` in your inventory.

## Safety: root lockout

`baseline_ssh_permit_root_login` defaults to `prohibit-password`. That blocks
password logins for root but keeps key logins, so the first run cannot lock
you out even if the admin user turns out to be broken.

Before the role writes `PermitRootLogin no`, it runs two checks. Both run
**before** the sshd drop-in is written, so if either fails, sshd is not touched:

1. **An explicit flag.** `baseline_admin_login_verified: true` must be set, or
   the play fails with instructions. The flag means "I logged in as the admin
   user myself". Only you can confirm that, because only you know which key
   and network path you will use later.
2. **An automated check from the controller.** Unless you set
   `baseline_ssh_verify_admin_from_controller: false`, the role runs
   `ssh -o BatchMode=yes -p <current port> <admin>@<host> sudo -n true` on the
   controller (delegated to localhost, using `ansible_ssh_private_key_file`
   when it is set). If key login or passwordless sudo fails, the play stops.

Why both: `wait_for_connection` reuses Ansible's existing (often root,
ControlPersist) connection, so it would not prove that the admin login works.
A controller-side `ssh` check proves it from one machine but can give false
failures behind bastions or with a custom `~/.ssh/config`, which is why you
can turn it off. The flag alone relies on trust. Together they are the most
robust option that still works everywhere.

The role adds more protection:

- The play stops if the admin user would have no SSH key.
- `sshd -t -f` checks the drop-in, then `sshd -t` checks the full merged
  config, before any reload.
- sshd handlers are **flushed at the end of the `ssh` tasks**, before UFW is
  enabled. A bad sshd change and a newly enabled firewall can never land
  together after the play has stopped.
- UFW allows SSH before `ufw enable`. Enabling UFW keeps established
  connections.
- `sshd -T` is read after the reload to confirm that the settings took effect.

## Safety: Docker bypasses UFW

> **Ports you publish with Docker (`-p 8080:80`, `ports:` in compose) are reachable
> from the internet even when UFW denies them.**

Docker writes its own iptables rules in the `nat` and `FORWARD` chains. Traffic
to a published port is DNATed and forwarded, so it never passes through UFW's
`INPUT` rules. `ufw status` says "deny" and the port is still open.

The role **does not change this by default**. You have these options:

1. **Bind containers to localhost (recommended).** Publish to `127.0.0.1` and
   put a reverse proxy (Caddy, nginx, Traefik) in front on 80/443:
   ```yaml
   ports:
     - "127.0.0.1:8080:80"
   ```
   Bind to a private or Tailscale IP for internal services.
2. **Don't publish ports at all.** Use a shared Docker network between the
   proxy and the apps.
3. **Opt-in fix: `baseline_docker_ufw_fix_enabled: true`.** This adds the
   [ufw-docker](https://github.com/chaifeng/ufw-docker) `DOCKER-USER` rules to
   `/etc/ufw/after.rules` (in a marked block) and reloads UFW. Then:
   - Published ports are **blocked from public networks** until you allow them
     with `ufw route allow`. List them in `baseline_docker_ufw_routes`:
     ```yaml
     baseline_docker_ufw_routes:
       - { port: "80", proto: tcp, comment: "web container" }
     ```
     Note that the **container** port goes here, not the host port, because
     the rule matches after DNAT.
   - Traffic from the private ranges (10/8, 172.16/12, 192.168/16) is still
     allowed.
   - Test it before relying on it, especially with custom Docker networks or
     an nftables backend.

## Running a single tag

```bash
ansible-playbook site.yml --tags ssh
ansible-playbook site.yml --tags firewall,fail2ban
ansible-playbook site.yml --skip-tags updates
ansible-playbook site.yml --tags tailscale -e baseline_tailscale_enabled=true
```

The platform assert and container detection are tagged `always`. The `ssh` tag
installs `ufw` (without enabling it) so it can pre-open a new port.

## Testing

```bash
pip install ansible-core ansible-lint molecule "molecule-plugins[docker]"
ansible-lint                         # production profile, see .ansible-lint
ansible-playbook --syntax-check site.yml -i inventory.example.ini
molecule test                        # Ubuntu 24.04 + 22.04, systemd, privileged
```

The Molecule scenario (`molecule/default`) uses the docker driver with the
systemd-enabled `geerlingguy/docker-ubuntu2404-ansible` and
`docker-ubuntu2204-ansible` images. It runs converge, an idempotence check and
`verify.yml`, which checks the effective sshd config, the admin user, sudo,
the fail2ban jail and the needrestart drop-in.

Inside a container `baseline_in_container` is true, and these steps are
**skipped** because they need a real kernel or host:

| Skipped | Why |
|---|---|
| `docker` | Docker-in-Docker |
| `sysctl` | Kernel parameters belong to the host |
| `firewall` | UFW/iptables in a container does not show real behaviour |
| `shm` | Mounts |
| reboot / controller ssh check / port reconnect | No real ssh connection |

These still run: `updates`, `users`, `ssh` (including the `sshd -T` checks and
the 24.04 `ssh.socket` port-change handlers) and `fail2ban`.
