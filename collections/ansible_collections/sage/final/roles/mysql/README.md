Sage.Final MySQL Role
=====================

Runs a **dedicated MariaDB (MySQL) database host** for the game-server panel as
a Docker Compose stack on the shared `web` network, with
[Adminer](https://www.adminer.org/) — the modern, single-container successor to
phpMyAdmin — as the web management UI fronted by the collection's shared Caddy.

This is the database host the **panel** (`sage.final.pelican_panel` /
`sage.final.pterodactyl_panel`) points at to provision a database per game
server. The role creates a `panel` management account with global privileges
(`GRANT OPTION`); the panel logs in as that user and creates a fresh database +
user for each game server on demand. The MySQL port is published on the host so
game servers running under Wings — including on other nodes — can connect.

Requirements
------------

- Ubuntu 22.04 (Jammy) or 24.04 (Noble)
- Docker engine + Compose plugin (installed once per host by `site.yml`'s
  docker play, or set `mysql_install_docker: true` for a standalone run)
- The shared `web` network already deployed (`sage.final.caddy`) — required
  only if you want Adminer fronted by Caddy
- `become: true`

Role Variables
--------------

See `defaults/main.yml` and `meta/argument_specs.yml` for the full list. All
have working defaults; nothing is strictly required. Common options:

| Variable                  | Default                         | Purpose                                            |
|---------------------------|---------------------------------|----------------------------------------------------|
| `mysql_image`             | `docker.io/library/mariadb:11.4`| Database image (pin a tag).                        |
| `mysql_panel_user`        | `panel`                         | Management account the panel connects as.          |
| `mysql_panel_user_host`   | `%`                             | Host the management user may connect from.         |
| `mysql_published_address` | `0.0.0.0`                       | Host IP the DB port binds to (narrow to restrict). |
| `mysql_published_port`    | `3306`                          | Published host port.                               |
| `mysql_max_connections`   | `200`                           | Raise for many concurrent game servers.            |
| `mysql_adminer_enabled`   | `true`                          | Deploy the Adminer web UI.                         |
| `mysql_group_members`     | `[]`                            | Admins granted SFTP access to `/srv/mysql`.        |

Passwords (`mysql_root_password`, `mysql_panel_password`) default to empty and
are **auto-generated once** on the target host, persisted under
`/srv/mysql/secrets/`. Supply your own (ideally vaulted) to override.

How it fits the shared-Caddy pattern
------------------------------------

The database publishes its port directly (game servers connect to it), but
Adminer does **not** — the shared Caddy fronts it. Add a site to `caddy_sites`
in inventory (lock it down, since it exposes every database):

```yaml
caddy_sites:
  - fqdn: "db.{{ root_domain }}"
    upstream: "mysql-adminer:8080"
    internal_only: true
```

Then deploy:

```bash
ansible-navigator run site.yml --tags caddy,mysql --eei ee-demo
```

Example Playbook
----------------

```yaml
- name: Deploy the panel database host
  hosts: mysql
  become: true
  roles:
    - role: sage.final.mysql
      vars:
        mysql_published_address: "10.0.0.5"   # private/WireGuard address
        mysql_group_members:
          - alice
```

Wiring the panel to it
----------------------

1. Run the role. It brings up MariaDB + Adminer and creates the `panel`
   management user.
2. In the panel, go to **Admin → Databases → Create New** (a *Database Host*):
   - **Host**: the address game servers reach this DB on
     (`mysql_published_address` / the host's IP).
   - **Port**: `mysql_published_port` (3306).
   - **Username**: `mysql_panel_user` (`panel`).
   - **Password**: the contents of `/srv/mysql/secrets/panel_password`.
3. Each game server can now request a database from the panel UI; the panel
   creates it and a scoped user on this host automatically.

Operations
----------

- **Adminer login**: server `mysql`, username `root` (or `panel`), password
  from `/srv/mysql/secrets/`.
- **Update the image**: bump `mysql_image` / `mysql_adminer_image` and re-run
  `--tags mysql`. The handler recreates the stack.
- **Logs**: `docker compose -f /srv/mysql/docker-compose.yml logs -f mysql`
- **CLI**: `docker exec -it mysql mariadb -uroot -p`
- **Data on disk**: lives under `/srv/mysql/data`, owned by the container's
  `mysql` user (UID 999). Back it up; do not edit by hand.

License
-------

BSD-2-Clause
