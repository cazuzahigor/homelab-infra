# 🏠 homelab-infra

> Infrastructure as Code for a production-grade homelab — automated, idempotent, and built to be rebuilt from scratch in minutes.

---

## Overview

This repository contains the full Ansible automation for my personal homelab, running on a dedicated Ubuntu 24.04 server connected via [Tailscale](https://tailscale.com/) mesh VPN with zero-trust SSH access.

The philosophy here is simple: **if it's not in code, it doesn't exist**. Every configuration, every package, every system state is declared, versioned, and reproducible.

---

## Stack

| Layer | Technology |
|---|---|
| OS | Ubuntu 24.04 LTS |
| Automation | Ansible Core 2.20 |
| Connectivity | Tailscale (WireGuard-based mesh VPN) |
| Python | 3.12 |
| Shell | Zsh + Starship + JetBrainsMono Nerd Font |

---

## Project Structure

```
homelab-infra/
├── ansible.cfg                  # Ansible configuration
├── inventory/
│   └── hosts.yml                # Host definitions
├── group_vars/
│   └── all/
│       └── vars.yml             # Global variables
├── host_vars/
│   └── homelab/                 # Host-specific overrides
├── playbooks/
│   ├── site.yml                 # Master playbook — provisions everything
│   ├── configure_base.yml       # Hostname, /etc/hosts
│   └── configure_shell.yml      # Zsh, Starship, Nerd Fonts
└── roles/
    └── shell/                   # Shell environment role
        └── tasks/
            └── main.yml
```

---

## Playbooks

### `site.yml` — Full Provisioning

Runs all playbooks in order. Use this to provision a server from scratch.

```bash
ansible-playbook playbooks/site.yml
```

### `configure_base.yml` — Base System

- Ensures correct hostname via `ansible.builtin.hostname`
- Keeps `/etc/hosts` in sync with `inventory_hostname`

### `configure_shell.yml` — Shell Environment

- Installs: `zsh`, `curl`, `git`, `btop`, `unzip`, `fontconfig`
- Sets `zsh` as the default shell for the provisioned user
- Installs [Starship](https://starship.rs/) prompt via official installer
- Downloads and installs **JetBrainsMono Nerd Font v3.3.0**
- Rebuilds the font cache with `fc-cache`

All tasks are **idempotent** — safe to run multiple times with no side effects.

---

## Usage

### Prerequisites

- Ansible Core 2.13+ (tested with 2.20)
- SSH access to the target host (via Tailscale or direct)
- Python 3.12 on the target host

### Running

```bash
# Dry run — see what would change without applying
ansible-playbook playbooks/site.yml --check

# Full provisioning
ansible-playbook playbooks/site.yml

# Single playbook
ansible-playbook playbooks/configure_shell.yml

# Ad-hoc connectivity test
ansible all -m ping
```

---

## Design Decisions

**Why `ansible-core` instead of the full `ansible` package?**
`ansible-core` is lean and explicit. Collections are installed on demand via `ansible-galaxy`, making dependencies visible and intentional rather than bundled by default.

**Why Tailscale for connectivity?**
Zero-config WireGuard mesh with device authentication. No open ports, no bastion host, no VPN server to maintain. SSH just works from anywhere, securely.

**Why Starship + JetBrainsMono Nerd Font in a server playbook?**
The server is accessed daily via terminal. A well-configured shell reduces friction, displays meaningful context (git branch, Java version, command duration), and signals that the environment was set up with intention — not improvisation.

---

## Idempotency

Every task is designed to be run repeatedly without unintended side effects:

- Package installations use `state: present`
- Starship is only installed if `/usr/local/bin/starship` doesn't exist
- Fonts are only downloaded and extracted if `JetBrainsMonoNerdFont-Regular.ttf` is absent
- Hostname and `/etc/hosts` tasks use `lineinfile` with `regexp` to avoid duplicates

---

## Roadmap

- [ ] `configure_docker.yml` — Docker Engine setup
- [ ] `configure_java.yml` — OpenJDK installation and `JAVA_HOME` setup
- [ ] `configure_security.yml` — SSH hardening, UFW firewall rules
- [ ] `configure_monitoring.yml` — Node Exporter + Prometheus + Grafana stack
- [ ] Ansible Vault for secrets management

---

## License

MIT
