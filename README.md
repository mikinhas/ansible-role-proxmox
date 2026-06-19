# ansible-role-proxmox

[![Lint](https://github.com/mikinhas/ansible-role-proxmox/actions/workflows/lint.yml/badge.svg)](https://github.com/mikinhas/ansible-role-proxmox/actions/workflows/lint.yml)

Ansible role for Proxmox VE host configuration.

## Installation

```bash
ansible-galaxy role install mikinhas.proxmox
```

This role depends on the `community.proxmox` collection (>= 2.0.0):

```bash
ansible-galaxy collection install -r requirements.yml
```

## Features

- Proxmox native firewall managed as code via the Proxmox API (no SSH/`pvesh`
  needed): datacenter options + rules and per-node rules, using the idempotent
  `community.proxmox` modules. The role runs the API calls from the controller
  (`delegate_to: localhost`).

## Supported Platforms

- Debian 12 (Bookworm) / Proxmox VE 8
- Debian 13 (Trixie) / Proxmox VE 9

## Requirements

API credentials for the target node. Prefer an **API token** of a dedicated user
over the root password, stored in Ansible Vault. The token needs permissions on
`/cluster/firewall` and `/nodes/<node>/firewall`.

## Role Variables

| Variable | Default | Description |
| --- | --- | --- |
| `proxmox_firewall_manage` | `true` | Master toggle for the whole role |
| `proxmox_firewall_apply` | `true` | Apply against the API. Set `false` to build rules without any API call (CI) |
| `proxmox_api_host` | `"{{ ansible_host \| default(inventory_hostname) }}"` | Proxmox API host |
| `proxmox_api_user` | `"root@pam"` | API user |
| `proxmox_api_password` | `""` | API password (use a token instead when possible) |
| `proxmox_api_token_id` | `""` | API token id, e.g. `ansible@pve!fw` |
| `proxmox_api_token_secret` | `""` | API token secret (store in Vault) |
| `proxmox_api_validate_certs` | `false` | Verify TLS cert of the API (set `true` in prod) |
| `proxmox_node` | `"{{ ansible_hostname }}"` | Node name (under `/nodes/<node>`) |
| `proxmox_firewall_enable` | `true` | Master switch (datacenter level) |
| `proxmox_firewall_policy_in` | `DROP` | Datacenter input policy (`ACCEPT`/`REJECT`/`DROP`) |
| `proxmox_firewall_policy_out` | `ACCEPT` | Datacenter output policy (`ACCEPT`/`REJECT`/`DROP`) |
| `proxmox_firewall_policy_forward` | `ACCEPT` | Datacenter forward policy (`ACCEPT`/`DROP` only) |
| `proxmox_firewall_ebtables` | `true` | Enable ebtables filtering |
| `proxmox_firewall_cluster_rules` | `[]` | Datacenter-wide rules |
| `proxmox_firewall_host_rules` | SSH + web UI | Per-node rules |

Each rule is a mapping: `action` (required, e.g. `ACCEPT`/`DROP`), plus optional
`type` (default `in`; one of `in`/`out`/`forward`/`group`), `proto`, `dport`
(string, e.g. `"22"` or a range `"8000:8100"`), `source`, `dest`, `iface`,
`comment`, `enable`. Rule positions (`pos`) are assigned automatically in list order.

> ⚠️ **Anti-lockout:** with `policy_in: DROP`, always keep an `ACCEPT` rule for SSH (22)
> and the web UI (8006) — otherwise enabling the firewall cuts your access.

## Usage

```yaml
- hosts: proxmox
  roles:
    - role: mikinhas.proxmox
      vars:
        proxmox_api_user: "ansible@pve"
        proxmox_api_token_id: "ansible@pve!fw"
        proxmox_api_token_secret: "{{ vault_proxmox_api_token_secret }}"
        proxmox_firewall_host_rules:
          - { action: ACCEPT, proto: tcp, dport: "22", comment: SSH }
          - { action: ACCEPT, proto: tcp, dport: "8006", comment: "Proxmox web UI" }
```

## License

MIT

## Author

[mikinhas](https://github.com/mikinhas)
