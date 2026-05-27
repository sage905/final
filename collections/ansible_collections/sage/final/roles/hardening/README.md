Sage.Final Hardening Role
=========================

Reasonable hardening for an Ubuntu server sitting behind a NAT gateway and
exposing the usual web + game-server ports. Five independent sections you
can toggle off in inventory:

| Section            | Toggle var                                  | What it does                                                                      |
|--------------------|---------------------------------------------|-----------------------------------------------------------------------------------|
| SSH                | `hardening_ssh_enabled`                     | Writes `/etc/ssh/sshd_config.d/99-hardening.conf` and reloads sshd.               |
| UFW firewall       | `hardening_firewall_enabled`                | Installs UFW, sets default-deny-incoming, allows the configured port list.       |
| fail2ban           | `hardening_fail2ban_enabled`                | Installs fail2ban with the sshd jail enabled (systemd backend).                  |
| Unattended upgrades| `hardening_unattended_upgrades_enabled`     | Installs `unattended-upgrades` and enables the security-updates timer.           |
| Sysctl             | `hardening_sysctl_enabled`                  | Drops `/etc/sysctl.d/99-hardening.conf` with standard kernel hardening knobs.    |

SSH defaults assume **key-based root login** because that's what Ansible
uses in this project. If your bastion connects as a non-root user with
sudo, tighten `hardening_ssh_permit_root_login` to `no`.

Default open ports (TCP unless noted):

| Port  | Service                       |
|-------|-------------------------------|
| 22    | SSH (`limit` rule by default) |
| 80    | HTTP (Caddy)                  |
| 443   | HTTPS (Caddy)                 |
| 2022  | Pterodactyl Wings SFTP        |
| 9090  | Cockpit                       |
| 25565 | Minecraft Java                |
| 19132 | Minecraft Bedrock (UDP)       |

Add more via `hardening_firewall_extra_tcp` / `_extra_udp` — each entry
is a `{port: <n>, comment: "<label>"}` dict.

Example
-------

```yaml
- name: Harden the host
  hosts: all
  become: true
  roles:
    - role: sage.final.hardening
      vars:
        hardening_firewall_extra_tcp:
          - {port: 7777, comment: Terraria}
          - {port: 27015, comment: Source Engine}
        hardening_firewall_extra_udp:
          - {port: 27015, comment: Source Engine}
        hardening_fail2ban_extra_ignoreip:
          - 203.0.113.7      # admin bastion
```

License: BSD-2-Clause.
