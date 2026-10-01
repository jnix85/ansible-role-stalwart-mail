# Project Context: ansible-role-stalwart-mail

**Type:** Ansible Role / Infrastructure Automation  
**Target:** Debian 12 (Bookworm), Debian 13 (Trixie), Ubuntu  
**Service:** [Stalwart Mail Server](https://stalw.art) 0.16+ (SMTP, IMAP, JMAP, CalDAV)

---

## Overview

Installs and configures Stalwart 0.16+ using the official release binaries under systemd.
Stalwart 0.16 uses a minimal `config.json` descriptor pointing to the datastore, with all domain,
listener, route, and account configuration applied as NDJSON plans via `stalwart-cli apply`.

---

## Repository Layout

```
.
├── AGENTS.md                         # AI source of truth
├── CLAUDE.md -> AGENTS.md            # Symlink for Claude Code
├── README.md                         # Comprehensive documentation
├── HANDOFF.md                        # Session state (0.16 port details)
├── ansible.cfg                       # Inventory and SSH settings (wires shared-inventory)
├── requirements.yml                  # Collections
├── inventory/
│   ├── hosts.yaml                    # mailservers, mailbox_servers, mx_edges
│   └── group_vars/                   # vars.yml per group
├── playbooks/                        # mailserver.yml, mailbox_servers.yml, mx_edges.yml
└── roles/
    └── stalwart/                     # Tasks, defaults, templates (plan.ndjson.j2, config.json.j2)
```

---

## Common Commands

```bash
# Syntax check
ansible-playbook -i inventory/hosts.yaml site.yml --syntax-check

# Dry-run
ansible-playbook -i inventory/hosts.yaml site.yml --check --diff
```
