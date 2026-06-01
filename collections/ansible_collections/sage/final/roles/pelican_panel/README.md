Sage.Final Pelican Panel Role
=============================

Runs the [Pelican Panel](https://pelican.dev/) — the modern, Laravel-based
successor to Pterodactyl — as a single-container podman-compose stack on the
shared `web` network, fronted by the collection's shared Caddy.

This role is a **drop-in alternative to `sage.final.pterodactyl_panel`**. Where
the Pterodactyl role runs a four-service stack (panel + MariaDB + Redis), Pelican
ships as one container that defaults to an embedded **SQLite** database and
generates its own `APP_KEY` on first boot — so there are no database secrets to
manage for a standard install.

Requirements
------------

- Ubuntu 22.04 (Jammy) or 24.04 (Noble)
- The shared Caddy + `web` network already deployed (`sage.final.caddy`)
- A DNS record for `pelican_panel_fqdn` pointing at the host
- `containers.podman` collection
- `become: true`

Role Variables
--------------

See `defaults/main.yml` and `meta/argument_specs.yml` for the full list.
Required per-host:

| Variable                    | Description                                  |
|-----------------------------|----------------------------------------------|
| `pelican_panel_fqdn`        | Public hostname for the panel (→ `APP_URL`). |
| `pelican_panel_admin_email` | Pre-seeds the first-run installer's email.   |

Common options: `pelican_panel_image` (pin a release tag),
`pelican_panel_timezone`, `pelican_panel_mail_driver`,
`pelican_panel_group_members` (admins granted SFTP access to `/srv/pelican`).

How it fits the shared-Caddy pattern
------------------------------------

The container does **not** publish 80/443 on the host — the shared Caddy owns
those and reverse-proxies to it. Add the site to `caddy_sites` in inventory:

```yaml
caddy_sites:
  - fqdn: "panel.{{ root_domain }}"
    upstream: "pelican-panel:80"
```

Then deploy Caddy and Pelican:

```bash
ansible-navigator run site.yml --tags caddy,pelican --eei ee-demo
```

Example Playbook
----------------

```yaml
- name: Deploy Pelican Panel
  hosts: pelican_panel
  become: true
  roles:
    - role: sage.final.pelican_panel
      vars:
        pelican_panel_fqdn: panel.example.org
        pelican_panel_admin_email: ops@example.org
```

First-run setup
---------------

1. Run the role. It brings the container up; first boot generates `APP_KEY`
   and runs database migrations (the panel is briefly unavailable — the role
   polls until it answers).
2. Open **`https://<pelican_panel_fqdn>/installer`** in a browser to create the
   first admin user and confirm the database (SQLite by default). The email
   field is pre-filled from `pelican_panel_admin_email`.
3. Configure mail, OAuth, CAPTCHA, etc. from the panel's **Settings** UI.

Using Pelican *instead of* Pterodactyl
--------------------------------------

This collection's `site.yml` includes both a Pterodactyl Panel play and a
Pelican Panel play. To switch a host from Pterodactyl to Pelican:

- Put the host in the `pelican_panel` inventory group **instead of**
  `pterodactyl_panel`.
- Point the panel's `caddy_sites` upstream at `pelican-panel:80` (was
  `pterodactyl-panel:80`).
- Game-server nodes: Pelican has its own [Wings fork](https://github.com/pelican-dev/wings);
  this role deploys only the panel. Node setup is done from the panel UI.

> The two panels keep their data in different places (`/srv/pterodactyl` vs
> `/srv/pelican`) and use different databases, so this is a migration, not a
> live swap. Export/recreate servers through the new panel; there is no
> in-place data conversion in this role.

Operations
----------

- **Update the image**: bump `pelican_panel_image` and re-run `--tags pelican`.
  The handler recreates the container.
- **Logs**:
  `podman-compose -f /srv/pelican/docker-compose.yml logs -f panel`
- **Artisan commands**:
  `podman exec pelican-panel php artisan <cmd>`
- **Data on disk**: SQLite DB, key, and settings live under
  `/srv/pelican/data` (owned by the container's web user after first boot —
  manage through the UI, not by editing files).

License
-------

BSD-2-Clause
