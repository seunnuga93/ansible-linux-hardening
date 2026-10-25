# ansible-linux-hardening

![ci](https://github.com/seunnuga93/ansible-linux-hardening/actions/workflows/ci.yml/badge.svg)

An Ansible role that applies a security baseline to fresh Ubuntu and Rocky Linux servers.
It is tested on every change with [Molecule](https://ansible.readthedocs.io/projects/molecule/)
against real systemd containers for both distributions, including an idempotence check.

## What it does

| Area | Changes |
| --- | --- |
| **SSH** | No root login, keys only, `MaxAuthTries 3`, no X11/agent/TCP forwarding, modern ciphers/KEX/MACs, `AllowGroups`, idle timeout, login banner. Config is checked with `sshd -t` before any restart, so a bad template can't lock you out. |
| **Users** | Creates admin users with SSH keys (`exclusive`, so stale keys are removed) and a sudoers drop-in checked with `visudo` |
| **Kernel** | `sysctl` hardening: no ICMP redirects or source routing, reverse-path filtering, SYN cookies, `kptr_restrict`, `dmesg_restrict`, full ASLR |
| **Firewall** | Default-deny inbound with `ufw` (Ubuntu) or `firewalld` (Rocky), SSH plus whatever ports you list |
| **Auditing** | auditd rules for identity, sudoers and SSH config changes and kernel module loads. fail2ban jail for SSH |
| **Updates** | `unattended-upgrades` (Ubuntu) or `dnf-automatic` security-only (Rocky) |
| **Services** | Stops and disables avahi, cups and rpcbind if present |

## Usage

```bash
ansible-galaxy collection install -r requirements.yml
cp inventory/hosts.example.yml inventory/hosts.yml   # add your servers
# edit inventory/group_vars/all.yml: admin users, extra open ports

ansible-playbook site.yml --check --diff   # dry run
ansible-playbook site.yml
```

> Run against a server where you already have key-based access as a user that will be in
> `linux_hardening_admin_users`. Password SSH logins are disabled by this role.

## Variables

All defaults are in [`roles/linux_hardening/defaults/main.yml`](roles/linux_hardening/defaults/main.yml). The common ones:

| Variable | Default | |
| --- | --- | --- |
| `linux_hardening_admin_users` | `[]` | List of `{name, ssh_key}` |
| `linux_hardening_ssh_port` | `22` | |
| `linux_hardening_ssh_allow_groups` | `[sudo, wheel]` | Only these groups may SSH in |
| `linux_hardening_firewall_allowed_tcp_ports` | `[]` | Opened in addition to SSH |
| `linux_hardening_sysctl` | see defaults | Dict of kernel parameters |
| `linux_hardening_manage_firewall` | on, off in containers | Kernel-level features are skipped automatically inside containers |

## Testing

```bash
pip install "ansible-core>=2.16,<2.19" "molecule>=24" "molecule-plugins[docker]" docker
molecule test
```

The Molecule scenario runs `dependency → syntax → create → converge → idempotence → verify` on
Ubuntu 24.04 and Rocky Linux 9. `molecule/default/verify.yml` asserts on the **effective** sshd
config (`sshd -T`), group membership, the sudoers file and that fail2ban is running. It doesn't
just check that files exist.
