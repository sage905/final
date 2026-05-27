Sage.Final WordPress Role
=========================

Deploys WordPress as a small podman-compose stack (wordpress + mariadb)
attached to the shared `pterodactyl` Podman network. Drops a Caddy vhost
snippet into the panel role's `conf.d/` and reloads Caddy, so the panel
role's existing Caddy fronts WordPress under `wordpress_fqdn`.

Requirements
------------

- The host already runs `sage.final.caddy` (or whatever role
  owns the shared Caddy + Podman network); the role expects to attach to an
  external network named `web` and write into
  `/srv/caddy/conf.d/`. Both are overridable.
- `containers.podman` collection for the Caddy reload handler.
- `become: true`.

Variables
---------

See `defaults/main.yml` and `meta/argument_specs.yml`. Required:

| Variable          | Description                                            |
|-------------------|--------------------------------------------------------|
| `wordpress_fqdn`  | Hostname Caddy serves WordPress on (apex or subdomain).|

Common overrides:

| Variable                       | Default                          |
|--------------------------------|----------------------------------|
| `wordpress_data_dir`           | `/srv/wordpress`                 |
| `wordpress_caddy_conf_d_dir`   | `/srv/caddy/conf.d`  |
| `wordpress_caddy_container`    | `caddy`              |
| `wordpress_shared_network`     | `pterodactyl`                    |

Example
-------

```yaml
- name: Deploy WordPress
  hosts: wordpress
  become: true
  roles:
    - role: sage.final.wordpress
      vars:
        wordpress_fqdn: blog.example.org
```

License
-------

BSD-2-Clause
