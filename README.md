# Ansible Role: Stalwart Mail Server

[![CircleCI](https://dl.circleci.com/status-badge/img/gh/jnix85/ansible-role-stalwart-mail-server/tree/main.svg?style=svg)](https://dl.circleci.com/status-badge/redirect/gh/jnix85/ansible-role-stalwart-mail-server/tree/main)

Installs and configures [Stalwart](https://stalw.art) **0.16+** — an
all-in-one mail & collaboration server (SMTP, IMAP, JMAP, POP3,
CalDAV/CardDAV) — from the official release binaries, running under systemd
as a dedicated unprivileged user.

Stalwart 0.16 changed its configuration model: the only file on disk is a
tiny `config.json` that names the datastore; every other setting (listeners,
TLS, logging, routing, accounts, ...) lives *in* the datastore and is managed
through the JMAP management API. This role follows that model: it writes
`config.json`, renders a declarative **plan** (NDJSON) from your variables and
applies it with [`stalwart-cli apply`](https://github.com/stalwartlabs/cli),
which idempotently creates, updates and removes the objects the role owns.
Servers running 0.15 or older need the previous major release of this role.

## Features

- **Debian 13 and Ubuntu 24.04/26.04** support (the primary, CI-tested
  targets; RHEL-family code paths exist but are currently untested)
- **Native binary install** to `/opt/stalwart`, version-pinned and upgradeable
  by bumping `stalwart_version`
- **Declarative configuration** applied through `stalwart-cli`; re-applied
  only when the rendered plan changes (or on demand)
- **Configurable TLS**: Stalwart's built-in ACME (Let's Encrypt), existing
  certificate files, or a generated self-signed cert for labs
- **Outbound relay / MX-edge mode**: accept mail for your domains and forward
  it to a mailbox server or smarthost
- **Configurable storage**: RocksDB (default, zero dependencies), SQLite, or
  an external PostgreSQL
- **Per-protocol listener toggles**: SMTP, submission (STARTTLS + implicit
  TLS), IMAP/IMAPS, POP3, ManageSieve, HTTPS (JMAP/API/web admin)
- **Hardened systemd unit** (non-root, `ProtectSystem=strict`,
  `CAP_NET_BIND_SERVICE` only)
- **Optional firewall management** (ufw/firewalld)
- Admin password is required up front (Vault-friendly) and stored on the host
  only as a SHA512-crypt hash — as the service's fallback admin
  (`STALWART_RECOVERY_ADMIN`) and as a permanent `admin` account

## Requirements

- Ansible **2.15+** on the controller
- A systemd-based target on `x86_64` or `aarch64`
- Collections (see `requirements.yml`): `community.crypto` (only for
  `stalwart_tls_mode: selfsigned`), `community.general` and `ansible.posix`
  (only when `stalwart_manage_firewall: true`)
- Outbound HTTPS from the target to `github.com` for the server and
  `stalwart-cli` release tarballs (or override `stalwart_download_url` /
  `stalwart_cli_download_url` to point at an internal mirror)

## Quick start

This repository is a ready-to-run Ansible project. The role itself lives in
[`roles/stalwart/`](roles/stalwart/); the playbooks, inventory and
group variables around it are a working starting point:

```
.
├── ansible.cfg                       # inventory + roles path
├── requirements.yml                  # collections
├── site.yml                          # entry point (runs playbooks/mailserver.yml)
├── playbooks/
│   ├── mailserver.yml                # every host in `mailservers`
│   ├── mx_edges.yml                  # inbound-only MX relays
│   ├── mailbox_servers.yml           # the full mailbox server(s)
│   └── smoke.yml                     # re-run only the health checks
├── inventory/
│   ├── hosts.yaml                    # mailservers → mx_edges + mailbox_servers
│   └── group_vars/
│       ├── all/vars.yml              # SSH connection settings
│       ├── mailservers/vars.yml      # shared Stalwart settings
│       ├── mailservers/vault.yml.example
│       ├── mx_edges/vars.yml         # port 25 + relay clawduino.com to the mailbox server
│       └── mailbox_servers/vars.yml  # full listener set
└── roles/stalwart/              # the role
```

```sh
# 1. Collections the role depends on
ansible-galaxy collection install -r requirements.yml

# 2. Point the inventory at your hosts and set your domain
$EDITOR inventory/hosts.yaml
$EDITOR inventory/group_vars/mailservers/vars.yml   # mail_domain, mailbox_server_fqdn, TLS

# 3. Create the vault with the admin password
cd inventory/group_vars/mailservers
cp vault.yml.example vault.yml && $EDITOR vault.yml && ansible-vault encrypt vault.yml
cd -

# 4. Deploy
ansible-playbook site.yml --ask-vault-pass
```

A green run means a working server: the role's built-in smoke test verifies
the service, listeners and health endpoint. Then open `https://<host>` and
log in as `admin` to add domains and accounts; Stalwart shows the exact
SPF/DKIM/DMARC records to publish.

For a lab box without public DNS, set `stalwart_tls_mode: selfsigned` in
`inventory/group_vars/mailservers/vars.yml`.

### Using the role on its own

Install it from git into any other project (it is published under the
Galaxy name `jnix85.stalwart_mail_server`):

```sh
ansible-galaxy role install \
  git+https://github.com/jnix85/ansible-role-stalwart-mail-server.git,main,jnix85.stalwart_mail_server
```

Minimal playbook:

```yaml
- hosts: mailservers
  become: true
  roles:
    - role: jnix85.stalwart_mail_server
      vars:
        stalwart_hostname: mail.example.com
        stalwart_admin_password: "{{ vault_stalwart_admin_password }}"
        stalwart_tls_mode: acme
        stalwart_acme_contact: postmaster@example.com
        stalwart_acme_domains:
          - mail.example.com
```

After the first run, open `https://mail.example.com/admin` and log in as
`admin` with the password you supplied. Mail domains, accounts, DKIM signing
and everything else not modelled by the role are managed there (or with
`stalwart-cli`). The role owns: the datastore, the server hostname and its
Domain object, listeners, TLS (ACME provider or certificate), logging, the
optional relay route, the admin account, and anything you add through
`stalwart_extra_plan_ops`.

### How the plan is applied

1. `config.json`, the service environment file and the plan are rendered
   under `/opt/stalwart/etc/`.
2. The service is started. On an empty datastore Stalwart inserts its safe
   defaults (all listeners on `[::]`, a `/var/log/stalwart` tracer, ...).
3. When the rendered plan changed (or `stalwart_plan_always_apply` /
   `-e stalwart_plan_force_apply=true`), the role waits for the loopback
   management listener (`http://127.0.0.1:8080` by default), runs
   `stalwart-cli apply`, and restarts the service so listener changes take
   effect. The plan replaces the default listener/tracer set with yours.

Because the plan is only re-applied when it changes, edits you make in the web
admin to role-owned objects persist until the next plan change. Set
`stalwart_plan_always_apply: true` to enforce the plan on every run.

## Role variables

Defaults live in [`roles/stalwart/defaults/main.yml`](roles/stalwart/defaults/main.yml). The important ones:

### Version & install

| Variable | Default | Description |
| --- | --- | --- |
| `stalwart_version` | `"0.16.9"` | Release to install (tag without `v`, must be >= 0.16). Changing it upgrades in place. |
| `stalwart_cli_version` | `"1.0.12"` | `stalwart-cli` release (separate project, own version line). Required. |
| `stalwart_download_urls` | GitHub release URLs | Candidate URLs probed in order. |
| `stalwart_download_url` / `stalwart_cli_download_url` | `""` | Explicit overrides (internal mirror); skip the candidate probing. |
| `stalwart_download_checksum` | `""` | Optional, e.g. `sha256:abc...`. |
| `stalwart_libc` | `gnu` | `gnu` or `musl` release flavour. |
| `stalwart_install_dir` | `/opt/stalwart` | Install prefix (`bin/`, `etc/`, `data/`, `logs/`). |
| `stalwart_user` / `stalwart_group` | `stalwart` | Dedicated system account the daemon runs as. |

### Identity & admin

| Variable | Default | Description |
| --- | --- | --- |
| `stalwart_hostname` | `ansible_fqdn` | FQDN the server identifies as. Also created as a Domain object (carries the server certificate, home of the admin account). Set this explicitly in production. |
| `stalwart_admin_user` | `admin` | Administrator name: the service's fallback admin and the permanent account `admin@<hostname>`. |
| `stalwart_admin_password` | — | **Required**, min 12 chars. Supply via Ansible Vault. |
| `stalwart_admin_create_account` | `true` | Also create the permanent Admin account (so an administrator exists if the fallback env var is removed). |

### TLS (`stalwart_tls_mode`)

| Mode | Description |
| --- | --- |
| `acme` | Stalwart obtains/renews certificates itself: the role creates an ACME provider and enables automatic certificate management on the hostname's Domain. Configure `stalwart_acme_contact` (Let's Encrypt rejects `example.com` addresses), `stalwart_acme_domains`, `stalwart_acme_challenge` (`tls-alpn-01` default — requires port 443 reachable from the internet; `http-01` needs port 80; `dns-01` needs no inbound port: set `stalwart_acme_dns_provider: cloudflare` and `stalwart_acme_dns_cloudflare_api_token`, a token with Zone:DNS:Edit — Stalwart publishes the `_acme-challenge` TXT record plus the types in `stalwart_acme_dns_publish_records`, CAA by default). Issuance happens in the background after the first start. |
| `files` | Use existing certs: set `stalwart_tls_cert_file` and `stalwart_tls_key_file` (must be readable by the `stalwart` user). The Certificate object references the files by path; restart Stalwart when they renew. |
| `selfsigned` | **Default.** Role generates a self-signed cert under `etc/certs/`. Labs and testing only. |

### Storage (`stalwart_storage_backend`)

| Backend | Description |
| --- | --- |
| `rocksdb` | **Default.** Embedded store under `stalwart_data_dir`. Best single-node choice. |
| `sqlite` | Embedded SQL store, fine for small deployments. |
| `postgresql` | External PostgreSQL — set `stalwart_postgresql_host/port/database/user/password` (and `stalwart_postgresql_use_tls`). The role does **not** install PostgreSQL. |

The datastore is the only thing in `config.json`. Changing the backend on an
existing host starts from an empty database — migrate data first.

### Listeners

`stalwart_listen_address` defaults to `auto`: dual-stack `[::]` when the host
has IPv6, `0.0.0.0` when IPv6 is disabled (e.g. booted with `ipv6.disable=1`).
Set it explicitly to pin a specific bind address — and set
`stalwart_smoke_test_host` to match if it isn't a wildcard.

Each protocol has an `_enabled` toggle and a `_port` variable:

| Toggle | Default | Port(s) |
| --- | --- | --- |
| `stalwart_smtp_enabled` | `true` | 25 |
| `stalwart_submission_enabled` | `true` | 587 (STARTTLS) |
| `stalwart_submissions_enabled` | `true` | 465 (implicit TLS) |
| `stalwart_imap_enabled` | `true` | 143 (STARTTLS) |
| `stalwart_imaps_enabled` | `true` | 993 (implicit TLS) |
| `stalwart_pop3_enabled` | `false` | 110 / 995 |
| `stalwart_managesieve_enabled` | `false` | 4190 |
| `stalwart_https_enabled` | `true` | 443 (JMAP, REST API, web admin) |
| `stalwart_http_enabled` | `false` | 8080 (plaintext, for reverse proxies / `http-01`) |

The plaintext `http` listener always exists because the role manages the
server through it: with `stalwart_http_enabled: false` it is bound to
`stalwart_management_bind_address` (`127.0.0.1`) only; with `true` it is bound
on `stalwart_listen_address` like the others.

The toggles feed a single `stalwart_listeners` list that the plan, firewall
rules and smoke test all iterate — override that list directly to add custom
listeners (fields: `name`, `protocol` [`smtp`, `lmtp`, `imap`, `pop3`,
`managesieve`, `http`], `enabled`, `port`, optional `tls_implicit`). Keep the
listener named `http`.

### Outbound relay (MX edges, smarthosts)

Set `stalwart_relay_host` (plus `stalwart_relay_port`,
`stalwart_relay_tls_implicit`, `stalwart_relay_allow_invalid_certs`,
`stalwart_relay_auth_username`/`_password`) to route all mail for non-local
domains to that host instead of MX delivery. List the domains an inbound-only
edge should accept in `stalwart_relay_domains`; they are created as Domain
objects with relaying allowed and routed to the relay ahead of local delivery.
See `inventory/group_vars/mx_edges/vars.yml`.

### Firewall

`stalwart_manage_firewall: false` by default. When enabled, the role opens the
enabled listener ports — and removes rules for listeners you've toggled off —
via **ufw** (Debian family) or **firewalld** (RedHat family). It deliberately
never activates a firewall itself (no `ufw enable`, no starting/enabling
firewalld): switching on a default-deny firewall could cut off your SSH
session. If firewalld isn't running, rules are written permanent-only and take
effect if you start it.

### Everything else

- `stalwart_smoke_test` (default `true`): after deployment the role verifies
  the service is active, enabled listeners accept connections, and
  `/healthz/live` returns 200 — so a green play means a working server. Set
  `stalwart_smoke_test_host` if `stalwart_listen_address` binds a specific
  address instead of a wildcard.
- `stalwart_plan_always_apply` (default `false`) re-applies the plan on every
  run; `-e stalwart_plan_force_apply=true` does it once.
- `stalwart_extra_plan_ops` appends raw `stalwart-cli apply` operations (one
  dict per item) to the role's plan for any setting the role doesn't model,
  e.g. `{"@type": "update", "object": "Jmap", "value": {"maxSizeUpload": 104857600}}`.
  Run `stalwart-cli describe <Object>` against a server for the schema.

## Example: existing certs + PostgreSQL

```yaml
- hosts: mailservers
  become: true
  roles:
    - role: jnix85.stalwart_mail_server
      vars:
        stalwart_hostname: mail.example.com
        stalwart_admin_password: "{{ vault_stalwart_admin_password }}"
        stalwart_tls_mode: files
        stalwart_tls_cert_file: /etc/letsencrypt/live/mail.example.com/fullchain.pem
        stalwart_tls_key_file: /etc/letsencrypt/live/mail.example.com/privkey.pem
        stalwart_storage_backend: postgresql
        stalwart_postgresql_host: db.internal
        stalwart_postgresql_password: "{{ vault_stalwart_pg_password }}"
        stalwart_manage_firewall: true
```

## After installation: DNS checklist

The role sets up the server; deliverability needs DNS:

1. **A/AAAA** record for `stalwart_hostname`
2. **MX** record for each mail domain pointing at `stalwart_hostname`
3. **PTR** (reverse DNS) on the server IP matching `stalwart_hostname`
4. **SPF**, **DKIM** and **DMARC** — Stalwart generates DKIM keys when you add
   a domain in the web admin and shows you the exact records to publish

## Testing

```sh
pip install ansible-core molecule molecule-plugins[docker] docker ansible-lint yamllint
ansible-galaxy collection install -r requirements.yml
molecule test                       # Debian 13 (default)
MOLECULE_DISTRO=ubuntu2404 molecule test
```

CI runs on [CircleCI](https://circleci.com): a lint job (yamllint +
ansible-lint) gates a Molecule matrix across Debian 13, Ubuntu 24.04 and
Ubuntu 26.04.

## License

MIT

## Author

jnix85
