Sage.Final Pterodactyl Panel Role
=================================

Runs the [Pterodactyl Panel](https://pterodactyl.io/) as a Docker Compose stack
on Ubuntu. The stack is composed of four services: `panel`, `database`
(MariaDB), `cache` (Redis), and `caddy` (auto-TLS reverse proxy).

Requirements
------------

- Ubuntu 22.04 (Jammy) or 24.04 (Noble) on the panel host
- `community.docker` collection
- A DNS record for `ptero_panel_fqdn` pointing at the panel host (required for
  Caddy's HTTP-01 ACME challenge)
- Ports 80/tcp and 443/tcp reachable from the public internet
- `become: true`

Role Variables
--------------

See `defaults/main.yml` and `meta/argument_specs.yml` for the full list.
Required variables that must be set per-host:

| Variable             | Description                                     |
|----------------------|-------------------------------------------------|
| `ptero_panel_fqdn`   | Public hostname for the panel.                  |
| `ptero_admin_email`  | ACME contact and default service author.        |

The role generates and persists `APP_KEY`, the database user password, and the
MariaDB root password under `{{ ptero_data_dir }}/secrets/` on first run. Pass
your own values via `ptero_db_password` / `ptero_db_root_password` to override.

Example Playbook
----------------

```yaml
- name: Deploy Pterodactyl Panel
  hosts: pterodactyl_panel
  become: true
  roles:
    - role: sage.final.pterodactyl_panel
      vars:
        ptero_panel_fqdn: panel.example.org
        ptero_admin_email: ops@example.org
        ptero_admin_username: rootadmin
        ptero_admin_password: "{{ vault_ptero_admin_password }}"
        ptero_mail_driver: smtp
        ptero_mail_host: smtp.sendgrid.net
        ptero_mail_username: apikey
        ptero_mail_password: "{{ vault_sendgrid_api_key }}"
```

Operations
----------

- **Update the panel image**: bump `ptero_panel_image` and re-run the role.
  The compose handler recreates affected services.
- **Manual artisan commands**:
  `docker compose -f /srv/pterodactyl/docker-compose.yml exec panel php artisan <cmd>`
- **Logs**: `docker compose -f /srv/pterodactyl/docker-compose.yml logs -f panel`

This role only deploys the *Panel*. Game-server nodes still need Pterodactyl
Wings installed and registered through the panel UI.

License
-------

BSD-2-Clause
