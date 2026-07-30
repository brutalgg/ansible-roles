# ansible-roles

Shared, environment-agnostic Ansible role library. All environment-specific values
(hostnames, VIPs, domains, secrets, CA certificates, user lists) live in each
consuming control repo's `group_vars`/`host_vars`/`vault`/`files`, **not** here.

Self-authored roles are **not** published to Ansible Galaxy — they are consumed
directly from Git. Public Galaxy roles/collections may still be pulled normally.

## Consuming this library

Each control repo pulls roles at runtime into a gitignored directory via its own
`requirements.yml`:

```yaml
# requirements.yml (in a control repo)
roles:
  - src: https://<git-host>/ansible-roles.git
    scm: git
    version: main          # pin a tag/sha for reproducible runs
```

```bash
ansible-galaxy install -r requirements.yml -p roles/   # roles/ is gitignored
```

Collections baseline (standardized on the newer set — see migration notes):

```yaml
collections:
  - name: community.docker      # >=5.0.0,<6.0.0
  - name: community.general     # >=12.0.0,<13.0.0
  - name: ansible.posix         # >=2.0.0,<3.0.0
```

## Standards & conventions

Role code in this library follows two short docs:

- [`docs/CONVENTIONS.md`](docs/CONVENTIONS.md) — mechanical line-by-line rules
  (FQCN, lowercase task names, `true`/`false`, quoted file modes, `loop` over
  `with_items`, `role__` registered-var prefixes, documented defaults).
- [`docs/STANDARDS.md`](docs/STANDARDS.md) — role structure (namespacing, variable
  ownership / interface vars, role siloing, lifecycle, multi-distro).

In short: provisioning is idempotent and never upgrades in place (upgrades live in
out-of-band `update.yml` files reached via a control-repo operations playbook), and
every value a role reads has a documented default in its `defaults/main.yml`. The
whole library passes `ansible-lint` at the **production** profile.

---

## Baseline roles

These provide the shared host baseline; control repos typically run
`base` → `custom_ca` (gated) → `accounts` → `docker` (on container hosts) ahead of
service roles.

### `base` — OS baseline
Apt sources (multi-distro, DietPi-aware, deb822 on Debian 13+), sshd hardening
(ed25519 host key, no password auth, no root login), timezone, hostname, and
`/etc/hosts`. Cache-only — never upgrades in place.

| Variable | Default | Notes |
|---|---|---|
| `base_extra_packages` | `[]` | Packages to install on every host; callers append. |
| `base_timezone` | `America/New_York` | |

### `accounts` — login accounts, sudo, keys, root
Unified list-driven account model (a single primary user is a list of one).
Manages users, the passwordless-sudo admin group, authorized keys, `.hushlogin`,
and the root account.

| Variable | Default | Notes |
|---|---|---|
| `accounts_users` | `[]` | List of `{name, shell?, sudo?, ssh_keys?, hushlogin?}`. |
| `accounts_default_shell` | `/bin/bash` | Applied when an entry omits `shell`. |
| `accounts_admin_group` | `admin` | Created only if some user has `sudo: true`. |
| `accounts_root` | `""` | `""` leaves root untouched; `"lock"` locks it; any other value is set as root's crypted password (supply from vault). |

Example:
```yaml
accounts_users:
  - name: ash
    sudo: true
    ssh_keys: ["{{ vault_ash_ssh_key }}"]
    hushlogin: true
accounts_root: "{{ vault_root_password }}"   # or "lock"
```

### `custom_ca` — custom CA trust
Installs a custom CA certificate into the system trust store. The cert is
environment-specific and **lives in the control repo**, not here — supply it as
inline PEM (`custom_ca_cert_content`) or a file path (`custom_ca_cert_src`);
content wins when both are set.

| Variable | Default | Notes |
|---|---|---|
| `custom_ca_enabled` | `true` | Set `false` for hosts that should not trust the CA; the role then no-ops. |
| `custom_ca_cert_content` | `""` | Inline PEM (e.g. `"{{ vault_root_ca }}"`). Takes precedence over `_src`. |
| `custom_ca_cert_src` | `""` | Path to the control repo's CA cert (e.g. `files/custom-ca.crt`). |
| `custom_ca_cert_name` | `custom-ca.crt` | Filename under `/usr/local/share/ca-certificates/`. |

Provide the cert one way or the other; the role asserts at least one is set when
enabled.

### `docker` — Docker engine
Installs Docker CE from Docker's apt repo (deb822 `.sources` + `/etc/apt/keyrings`),
manages `daemon.json`, the data-root, the service, and docker-group membership.

| Variable | Default | Notes |
|---|---|---|
| `docker_users` | `[]` | Users added to the `docker` group. |
| `docker_data_folder` | `/var/lib/docker/` | |
| `docker_data_folder_mode` | `0710` | Mode for the data-root dir. Matches the `0710` dockerd sets on startup, so the dir stays idempotent. |
| `docker_apt_release_channel` | `stable` | |
| `docker_packages` | docker-ce set | |
| `docker_service_state` / `docker_service_enabled` | `started` / `true` | |

### `firewall` — host ufw manager
Host-level ufw facility for internet-facing hosts. The host declares the complete
set of ports it exposes in a single `firewall_rules` list (one source, no cross-scope
merging); the role allows SSH, applies the rules, and default-denies incoming.
Application roles never touch ufw — they just need their ports in the list.

| Variable | Default | Notes |
|---|---|---|
| `firewall_manage` | `true` | Master switch; `false` no-ops the role. |
| `firewall_allow_ssh` | `true` | Allow inbound SSH (OpenSSH profile) before enabling, so a run can't strand SSH. |
| `firewall_rules` | `[]` | `[{port, proto?, from?, comment?, rule?}]` — `proto` defaults `tcp`, `from` any, `rule` allow. |

Example:
```yaml
firewall_rules:
  - { port: "443", proto: tcp, comment: "https" }
  - { port: "51820", proto: udp, comment: "wireguard" }
  - { port: "2376", proto: tcp, from: "10.9.0.0/24", comment: "docker api (tunnel only)" }
```

### `fastfetch` — system info on ssh login
Installs [fastfetch](https://github.com/fastfetch-cli/fastfetch) (distro package
where one exists, GitHub releases `.deb` otherwise) and shows it on interactive
SSH logins via `/etc/profile.d` — chosen over motd because `~/.hushlogin` (see
`accounts`) silences pam_motd but not profile.d. zsh login shells don't read
`/etc/profile.d`, so a managed block in `/etc/zsh/zprofile` sources the same
snippet where zsh is installed. Provisioning installs once and never upgrades;
the deliberate upgrade is `tasks_from: update.yml` (honors the shared
`github_api_*` globals for the release lookup).

| Variable | Default | Notes |
|---|---|---|
| `fastfetch_install_method` | `auto` | `auto` (apt if available, else GitHub `.deb`), `apt`, or `github`. |
| `fastfetch_github_version` | `latest` | Release for the github path; pin like `"2.53.0"` (no `v` prefix). |
| `fastfetch_login_banner` | `true` | Manage `/etc/profile.d/fastfetch.sh`; `false` removes it. |
| `fastfetch_login_zsh` | `true` | Also hook `/etc/zsh/zprofile` where zsh is present. |
| `fastfetch_login_command` | `fastfetch` | Command the login hook runs (add flags/config here). |
| `fastfetch_config` | `""` | Inline jsonc → `/etc/fastfetch/config.jsonc`; empty = unmanaged. |

---

## Service roles

> Variable tables below list the **key / required** inputs; see each role's
> `defaults/main.yml` for the full surface. Vars marked **vault** must come from
> the control repo's encrypted vault.

### `act_runner` — Gitea CI runner
Installs the runner **native** (systemd daemon from the pinned binary, the
default) or as a **container** (`act_runner_install_mode: container` —
`gitea/act_runner` via docker compose, registered through the image's env-based
auto-registration); the same config interface drives both. Independently,
`act_runner_config_type` (`host`|`container`) selects where CI **jobs** run.
Container install mode needs the `docker` role first. Switch an already-provisioned
host between modes by running the out-of-band `tasks_from: teardown.yml` for the
old mode first (`act_runner_teardown_purge` for a full wipe), then provisioning
the new mode.
Required: `act_runner_gitea_instance_url`, `act_runner_token` (**vault**).
Key: `act_runner_version`, `act_runner_install_mode` (`native`|`container`),
`act_runner_config_type` (`host`|`container`), `act_runner_image`,
`act_runner_compose_dir`.

### `autorestic` — Restic backups
Restic + autorestic with systemd timers and NFS handling (managed on VMs,
assumed-present on LXC). Renders per-host config from `autorestic_services`.
Required: `autorestic_services` (host_vars), `autorestic_restic_password` (**vault**).
Key: `autorestic_nfs_managed`, `autorestic_dest_dir`, `autorestic_schedule`.

### `caddy` — reverse proxy / TLS
Caddy, disabled by default. Installs **native** (systemd binary built with xcaddy,
the default) or as a **container** (`caddy_install_mode: container` — the caddy
image via docker compose); the same `caddy_sites` interface drives both, so a
consumer switches modes with one var. Renders sites from `caddy_sites`: each site
reverse-proxies an `upstream` **or** serves static files from a `root`
(+ optional `encode`/`headers`), and does ACME (dns-01/http-01), `tls internal`,
or operator-supplied certs via per-site `tls_cert`/`tls_key` (under `caddy_ssl_dir`).
Container mode needs the `docker` role first and, for the dns-01 challenge, a
`caddy_image` prebuilt with the dns plugin(s) (`caddy_image_has_dns_plugins`).
Switching an already-provisioned host between modes: run the out-of-band
`tasks_from: teardown.yml` for the old mode first (it frees `:80`/`:443`;
`caddy_teardown_purge` for a full wipe), then provision the new mode.
Key: `caddy_enabled`, `caddy_install_mode`, `caddy_sites`
(domain / upstream|root / tls per entry), `caddy_le_staging`, `caddy_resolvers`,
`caddy_image`, `caddy_compose_dir`.

### `gitea` — self-hosted git host
Compose stack (gitea + its own postgres). **Standalone by default**: reachable
directly on the host, optionally terminating its own TLS (`gitea_tls_enabled`),
no proxy or runner assumed. Front it with the `caddy` role and add CI with the
`act_runner` role by composing them in a **playbook** — the role stays
proxy-agnostic. Self-contained "seed": depends only on `docker`. Config is mostly
first-class `gitea_*` vars (deployments differ freely), with `gitea_extra_config`
(`{section: {KEY: value}}`) as an escape hatch for anything not exposed.
Required (**vault**): `gitea_postgres_password`, `gitea_secret_key`,
`gitea_internal_token`, `gitea_jwt_secret`, `gitea_lfs_jwt_secret`.
Key: `gitea_domain`, `gitea_version`, `gitea_uid`/`gitea_gid`, `gitea_http_bind`,
`gitea_tls_enabled` (+ `gitea_tls_cert`/`_key`), `gitea_postgres_data_dir`
(pg18 → `/var/lib/postgresql`), `gitea_data_bind`/`_volume_opts` (host path or NFS),
`gitea_extra_config`.

### `openvpn` — OpenVPN server
Routed (`tun`) OpenVPN **server** with a self-contained easy-rsa PKI: issues a CA,
a server cert, and one cert per client, renders the server config, enables
forwarding + NAT, and (with `openvpn_sync_certs`) writes inline `.ovpn` profiles and
fetches them back to the controller. PKI material is generated when absent and
reused when present — supply your own CA/keys via the `openvpn_*_content` vars
(**vault**) to sign under an existing CA. A drop-in for the common
`ansible-role-openvpn` interface: `openvpn_clients` defaults to the unprefixed
`clients` var, so a `clients: [...]` playbook works unchanged.
Key: `openvpn_server_hostname` (address clients dial), `clients` /
`openvpn_clients`, `openvpn_proto`/`openvpn_port`, `openvpn_server_network`/`_netmask`
(+ `openvpn_server_subnet` for the NAT rule), `openvpn_redirect_gateway`,
`openvpn_compression`, `openvpn_sync_certs` (+ `openvpn_fetch_client_configs_dir`),
`openvpn_revoke_these_certs`. NAT masquerade is applied but not reboot-persisted —
pair with `firewall`/netfilter-persistent if it must survive a reboot.

### `wireguard` — VPN overlay
`wg-quick` mesh between the play's hosts: every host peers with every other play
host; hosts outside Ansible join via `wireguard_unmanaged_peers`. Auto-generates
a keypair if absent and reuses the deployed key thereafter (key handling `no_log`).
Required: `wireguard_addresses` (per host, host_vars).
Key: `wireguard_interface` (`wg0`), `wireguard_port` (`51820`),
`wireguard_private_key` (optional; supply from **vault** or let it generate),
`wireguard_endpoint` (`""` = client-only host), `wireguard_service_enabled`/`_state`.

---

## License

[MIT](LICENSE).
