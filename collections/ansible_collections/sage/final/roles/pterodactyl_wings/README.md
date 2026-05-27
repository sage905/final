Sage.Final Pterodactyl Wings Role
=================================

Installs the [Pterodactyl Wings](https://pterodactyl.io/wings/) daemon on
Ubuntu game-server nodes. Wings runs game-server containers under Docker on
behalf of the Pterodactyl Panel.

Requirements
------------

- Ubuntu 22.04 (Jammy) or 24.04 (Noble)
- `community.general` collection (for the Docker repo bits, no extra
  collection beyond what `pterodactyl_panel` already pulls in)
- A running Pterodactyl Panel that this node will register against
- `become: true`

Role Variables
--------------

See `defaults/main.yml` and `meta/argument_specs.yml`. Highlights:

| Variable                       | Default            | Description                                             |
|--------------------------------|--------------------|---------------------------------------------------------|
| `pterodactyl_wings_version`                | `latest`           | Release tag (e.g. `1.11.13`) or `latest`.               |
| `pterodactyl_wings_config_yaml`            | `""`               | Contents of `config.yml`; paste from Panel UI if empty. |
| `pterodactyl_wings_enable_swap_accounting` | `true`             | Adds `swapaccount=1` to GRUB (reboot required).         |
| `pterodactyl_wings_install_podman`         | `true`             | Install Docker CE from the upstream apt repo.           |

Bootstrapping a node
--------------------

1. Run the role to install Docker, the Wings binary, and the systemd unit.
   The service is enabled but not started while `config.yml` is missing.
2. In the Panel UI, **Admin → Nodes → Create Node** and fill in the FQDN and
   ports for this host.
3. Copy the auto-generated `config.yml` from the Panel into
   `/etc/pterodactyl/config.yml` on the node (or pass it as
   `pterodactyl_wings_config_yaml` on a re-run via vault).
4. `systemctl start wings`.

If `pterodactyl_wings_enable_swap_accounting` made GRUB changes, reboot before relying on
per-container memory+swap limits.

Example Playbook
----------------

```yaml
- name: Deploy Pterodactyl Wings
  hosts: pterodactyl_wings
  become: true
  roles:
    - role: sage.final.pterodactyl_wings
      vars:
        pterodactyl_wings_version: "1.11.13"
        pterodactyl_wings_config_yaml: "{{ lookup('ansible.builtin.file', 'wings-config.yml') }}"
```

License
-------

BSD-2-Clause
