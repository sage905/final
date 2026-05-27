Sage.Final Cockpit Role
=======================

Installs the [Cockpit](https://cockpit-project.org/) web management UI on
Ubuntu, enables its socket-activated service, and (optionally) opens the
firewall.

Requirements
------------

- Target hosts running Ubuntu (tested on 22.04 Jammy and 24.04 Noble).
- `community.general` collection (for the `ufw` module) when
  `cockpit_manage_firewall` is `true`.
- Privilege escalation (`become: true`) on the target host.

Role Variables
--------------

See `defaults/main.yml` and `meta/argument_specs.yml` for the authoritative
list. Highlights:

| Variable                  | Default            | Description                                        |
|---------------------------|--------------------|----------------------------------------------------|
| `cockpit_packages`        | `[cockpit]`        | APT packages to install. Append modules as needed. |
| `cockpit_use_backports`   | `true`             | Install from the `-backports` pocket.              |
| `cockpit_service`         | `cockpit.socket`   | Systemd unit to enable.                            |
| `cockpit_manage_firewall` | `true`             | Add UFW allow rule for the Cockpit port.           |
| `cockpit_port`            | `9090`             | TCP port Cockpit listens on.                       |

Example Playbook
----------------

```yaml
- name: Install Cockpit on Ubuntu servers
  hosts: web_servers
  become: true
  roles:
    - role: sage.final.cockpit
      vars:
        cockpit_packages:
          - cockpit
          - cockpit-podman
          - cockpit-networkmanager
```

After the play completes, the UI is reachable at `https://<host>:9090/`.

License
-------

BSD-2-Clause
