Sage.Final Caddy Role
=====================

Runs [Caddy](https://caddyserver.com/) as a small podman-compose stack
listening on host ports 80/443 with automatic TLS. Creates the external
Podman network (`web` by default) that backend roles attach to and exposes
`/srv/caddy/conf.d/` as the drop-in directory for vhost snippets.

Required: `caddy_admin_email` (ACME registration).

Backend roles in this collection ship a small `<name>.caddy` snippet into
the shared `conf.d/`, then notify the Caddy container to reload. The Caddy
role does not know or care about backends — it just imports every
`conf.d/*.caddy` and proxies the named hosts.

Run this role **before** any backend role that depends on it. The compose
network and `conf.d/` directory must exist before backends try to attach.

```yaml
- name: Deploy Caddy
  hosts: caddy
  become: true
  roles:
    - role: sage.final.caddy
      vars:
        caddy_admin_email: ops@example.org
```

License: BSD-2-Clause.
