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

- **Host networking**: an internal-only Linux bridge plus NAT, so VMs reach the
  internet through the host's single public IP (typical OVH dedicated setup),
  and port forwards to publish a VM's services on that same IP.
  The bridge has no physical port, so applying it never touches the public NIC.
- Proxmox native firewall managed as code via the Proxmox API (no SSH/`pvesh`
  needed): datacenter options + rules and per-node rules, using the idempotent
  `community.proxmox` modules. The role runs the API calls from the controller
  (`delegate_to: localhost`).
- Cloud-init **VM templates** built from official cloud images (`qm`), ready to
  be cloned. Idempotent: a template is built only if its VMID does not exist.
- **QEMU VMs** created by cloning a template and applying cloud-init (user, SSH
  key, IP). Idempotent: a VM is created only if its VMID does not exist.

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
| `proxmox_firewall_policy_in` | `ACCEPT` | Datacenter input policy (`ACCEPT`/`REJECT`/`DROP`) |
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

### Host networking (internal NAT bridge)

| Variable | Default | Description |
| --- | --- | --- |
| `proxmox_network_manage` | `false` | Manage host networking (bridge + NAT) — toggle |
| `proxmox_internal_bridges` | `[{iface: vmbr1, cidr: 10.0.0.1/24}]` | Internal bridges to create. Each entry: `iface`, `cidr` (host/gateway address); optional `bridge_ports`, `autostart`, `comments` |
| `proxmox_nat_enable` | `true` | Masquerade VM traffic for outbound internet |
| `proxmox_internal_subnet` | `10.0.0.0/24` | Subnet NAT'd out the public interface |
| `proxmox_public_interface` | `vmbr0` | Iface holding the public IP (NAT egress) |
| `proxmox_nat_port_forwards` | `[]` | Ports published from the public IP to a VM. Each entry: `proto` (`tcp`/`udp`), `port`, `to` (VM address); optional `comment` |

The bridge is written to `/etc/network/interfaces.d/<bridge>` (survives GUI
edits) and brought up with `ifup <bridge>` — only the internal bridge, never the
public NIC, so an SSH session can't be dropped. Point your VMs at one by setting
the template's `bridge` to a bridge's `iface` and giving each VM
`ipconfig0: "ip=10.0.0.X/24,gw=10.0.0.1"`.

> ⚠️ Find the public interface with `ip route get 1.1.1.1` on the node (the
> `dev ...` field) and set `proxmox_public_interface` to it. Changing an
> *existing* bridge's config won't re-apply live — run
> `ifdown <bridge> && ifup <bridge>` on the node (safe: no physical port).

Masquerade and port forwards both live in `proxmox-nat.service`, a `oneshot`
unit re-applied at boot and on every change. `iptables` is used directly here
rather than a module: the Proxmox firewall only filters, it does not translate
addresses, so nothing in `community.proxmox` exposes DNAT — and
`iptables-persistent` would snapshot the `PVEFW-*` chains that `pve-firewall`
rebuilds by itself. Each forward also gets a hairpin rule, so a VM reaching the
public IP on a published port still gets a correctly sourced reply.

```yaml
proxmox_nat_port_forwards:
  - { proto: tcp, port: 443, to: "10.0.0.254", comment: "HTTPS" }
  - { proto: udp, port: 3478, to: "10.0.0.254", comment: "STUN" }
```

> ⚠️ Removing an entry stops it from being re-applied, but the live rule is only
> dropped on the *next* `systemctl stop` — delete it by hand (`iptables -t nat
> -D ...`) or reboot the node to clear it immediately.

### VM templates

| Variable | Default | Description |
| --- | --- | --- |
| `proxmox_template_manage` | `false` | Build cloud-init templates (toggle) |
| `proxmox_template_image_dir` | `/var/lib/vz/template/qcow` | Where cloud images are cached on the node |
| `proxmox_templates` | Debian 12 example | List of templates to build |

Each template entry: `vmid` (required), `name` (required), `image_url`
(required), `storage` (required), plus optional `image_checksum`, `bridge`
(default `vmbr0`), `cores` (`2`), `memory` (`2048`), `disk_size` (grow the disk,
in GiB, e.g. `10`).

Templates are built with the `community.proxmox` modules (`proxmox_kvm` +
`proxmox_disk`) over the API (`delegate_to: localhost`), like the firewall — no
`qm`/SSH shell commands. Only the cloud image **download** runs on the node.

> ⚠️ **Requirements:**
> - `api_user` must be **`root@pam`**: `proxmox_disk` imports the disk from an
>   absolute path, which Proxmox only allows for root.
> - The play targets the node with `become: true` so the image can be written to
>   `proxmox_template_image_dir` on it.
> - A template is built only if its VMID does not already exist (delete the VM to
>   rebuild it).

### Virtual machines

| Variable | Default | Description |
| --- | --- | --- |
| `proxmox_vm_manage` | `false` | Create VMs (toggle) |
| `proxmox_vms` | `[]` | List of VMs to create |

Each VM entry: `vmid` (required), `name` (required), `template_vmid` (required,
the source template to clone), plus optional `storage` (target; default = the
template's), `full` (default `true`), `cores`, `memory`, `disk_size` (grow
`scsi0`, in GiB), `onboot`, `start` (default `true`), and cloud-init settings:
`ciuser`, `cipassword`, `sshkeys`, `nameservers`, `searchdomains`, and
`ipconfig` (a dict, e.g. `{ipconfig0: "ip=dhcp"}` or
`{ipconfig0: "ip=10.0.0.5/24,gw=10.0.0.1"}`).

VMs are cloned and configured with `proxmox_kvm` over the API
(`delegate_to: localhost`). A VM is created only if its VMID does not already
exist; the template referenced by `template_vmid` must exist first (build it via
`proxmox_template_manage`).

## Usage

Internal NAT bridge for the VMs (run on the node with `become`):

```yaml
- hosts: proxmox
  become: true
  roles:
    - role: mikinhas.proxmox
      vars:
        proxmox_network_manage: true
        proxmox_internal_bridges:
          - iface: vmbr1
            cidr: "10.0.0.1/24"
        proxmox_public_interface: vmbr0   # check: ip route get 1.1.1.1
```

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

```yaml
- hosts: proxmox
  become: true
  roles:
    - role: mikinhas.proxmox
      vars:
        proxmox_template_manage: true
        proxmox_api_user: "root@pam"
        proxmox_api_token_id: "root@pam!templates"
        proxmox_api_token_secret: "{{ vault_proxmox_api_token_secret }}"
        proxmox_templates:
          - vmid: 9000
            name: debian-12-cloud
            image_url: "https://cloud.debian.org/images/cloud/bookworm/latest/debian-12-genericcloud-amd64.qcow2"
            storage: local
            disk_size: 10
```

```yaml
- hosts: proxmox
  roles:
    - role: mikinhas.proxmox
      vars:
        proxmox_vm_manage: true
        proxmox_api_user: "root@pam"
        proxmox_api_token_id: "root@pam!vms"
        proxmox_api_token_secret: "{{ vault_proxmox_api_token_secret }}"
        proxmox_vms:
          - vmid: 100
            name: web-01
            template_vmid: 9000
            cores: 2
            memory: 2048
            storage: local
            disk_size: 20
            ciuser: debian
            sshkeys: "{{ vault_ssh_public_key }}"
            ipconfig:
              ipconfig0: "ip=dhcp"
            start: true
```

## License

MIT

## Author

[mikinhas](https://github.com/mikinhas)
