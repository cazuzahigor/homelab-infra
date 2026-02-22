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
| Secrets | Ansible Vault (AES256) |

---

## Project Structure

```
homelab-infra/
├── ansible.cfg                      # Ansible configuration
├── inventory/
│   └── hosts.yml                    # Host definitions (Tailscale IPs)
├── group_vars/
│   └── all/
│       ├── vars.yml                 # Global variables
│       └── vault.yml                # Encrypted secrets (AES256)
├── host_vars/
│   └── homelab/                     # Host-specific overrides
├── playbooks/
│   ├── site.yml                     # Master playbook — provisions everything
│   ├── configure_base.yml           # Hostname, /etc/hosts
│   ├── configure_shell.yml          # Zsh, Starship, Nerd Fonts
│   ├── configure_security.yml       # SSH hardening, fail2ban
│   └── configure_network.yml        # Static IP, Tailscale optimizations
└── roles/
    ├── shell/                       # Shell environment role
    │   └── tasks/
    │       └── main.yml
    ├── security/                    # Security hardening role
    │   ├── tasks/
    │   │   └── main.yml
    │   ├── templates/
    │   │   └── sshd_config.j2
    │   └── handlers/
    │       └── main.yml
    └── network/                     # Network configuration role
        ├── tasks/
        │   └── main.yml
        ├── templates/
        │   ├── 50-cloud-init.yaml.j2
        │   ├── 50-tailscale.j2
        │   └── sysctl-tailscale.conf.j2
        ├── handlers/
        │   └── main.yml
        └── defaults/
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

### `configure_security.yml` — Security Hardening

- Ensures SSH public key is present in `authorized_keys` before any lockdown
- Disables password authentication — key-based auth only
- Disables root login
- Deploys hardened `sshd_config` with pre-apply syntax validation via `sshd -t`
- Installs and configures **fail2ban** with a custom SSH jail: 3 retries, 1h ban
- Service restarts only triggered when configuration actually changes (handlers)

### `configure_network.yml` — Network Configuration

- Configures **static IP** via netplan — eliminates DHCP drift
- Deploys **IP forwarding** (`sysctl`) for Tailscale Subnet Router
- Enables **UDP GRO forwarding** for WireGuard throughput optimization
- Disables **Wi-Fi power save** to eliminate latency spikes
- Persists all optimizations across reboots via `networkd-dispatcher`
- Wi-Fi credentials managed via **Ansible Vault** (AES256) — never stored in plaintext

All tasks are **idempotent** — safe to run multiple times with no side effects.

---

## Secrets Management

Sensitive data is encrypted with Ansible Vault (AES256). The vault file lives at `group_vars/all/vault.yml` and is safe to commit — only the vault password file decrypts it.

```bash
# View encrypted secrets
ansible-vault view group_vars/all/vault.yml

# Edit encrypted secrets
ansible-vault edit group_vars/all/vault.yml

# Run playbook with vault (password file configured in ansible.cfg)
ansible-playbook playbooks/site.yml
```

The vault password file path is configured in `ansible.cfg` via `vault_password_file`. It lives outside the repository and is never committed.

---

## Usage

### Prerequisites

- Ansible Core 2.13+ (tested with 2.20)
- SSH access to the target host (via Tailscale or direct)
- Python 3.12 on the target host
- Vault password file at `~/.ansible-vault-pass`

### Running

```bash
# Dry run — see what would change without applying
ansible-playbook playbooks/site.yml --check

# Full provisioning
ansible-playbook playbooks/site.yml

# Single playbook
ansible-playbook playbooks/configure_network.yml

# Ad-hoc connectivity test
ansible all -m ping

# First-time provisioning on a new server (password auth still active)
ansible-playbook playbooks/site.yml --ask-pass
```

---

## Design Decisions

**Why `ansible-core` instead of the full `ansible` package?**
`ansible-core` is lean and explicit. Collections are installed on demand via `ansible-galaxy`, making dependencies visible and intentional rather than bundled by default.

**Why Tailscale for connectivity?**
Zero-config WireGuard mesh with device authentication. No open ports, no bastion host, no VPN server to maintain. SSH just works from anywhere, securely. Inventory uses Tailscale IPs (`100.x.x.x`) which never change, regardless of local DHCP.

**Why static IP via netplan instead of DHCP reservation?**
The ISP-provided router does not support DHCP reservations. Static IP configured directly on the host via netplan ensures the server always has a predictable local address, independent of the router's capabilities.

**Why validate `sshd_config` before deploying?**
The `validate` parameter on the template task runs `sshd -t -f %s` against a temporary copy before writing to `/etc/ssh/sshd_config`. If the configuration is invalid, the file is never replaced — preventing accidental lockout from the server.

**Why copy the SSH key before hardening?**
The `authorized_key` task runs first to guarantee key-based access is established before `PasswordAuthentication no` takes effect. This makes the role safe to run on brand-new servers without manual preparation.

**Why Ansible Vault for Wi-Fi credentials?**
Wi-Fi passwords in netplan templates would be stored in plaintext in the repository. Vault encrypts them with AES256 — the encrypted file is safe to commit, and decryption only happens at runtime on the control machine.

---

## Idempotency

Every task is designed to be run repeatedly without unintended side effects:

- Package installations use `state: present`
- Starship is only installed if `/usr/local/bin/starship` doesn't exist
- Fonts are only downloaded and extracted if `JetBrainsMonoNerdFont-Regular.ttf` is absent
- Hostname and `/etc/hosts` tasks use `lineinfile` with `regexp` to avoid duplicates
- SSH and fail2ban services are restarted only via handlers — only when their configs change
- Netplan is only applied via handler when the Wi-Fi config template actually changes

---

## Roadmap

- [ ] `configure_docker.yml` — Docker Engine setup
- [ ] `configure_java.yml` — OpenJDK installation and `JAVA_HOME` setup
- [ ] `configure_monitoring.yml` — Node Exporter + Prometheus + Grafana stack

---

## License

MIT
