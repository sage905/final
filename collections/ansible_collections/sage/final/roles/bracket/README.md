Sage.Final Bracket Role
=======================

Runs [Bracket](https://github.com/evroon/bracket) (tournament system) as a
podman-compose stack (bracket + postgres) on the shared `pterodactyl`
network and registers it with the panel role's Caddy via a `conf.d/`
snippet.

Required: `bracket_fqdn`. DB password is auto-generated and persisted to
`/srv/bracket/secrets/db_password`.

First-run default admin credentials are shipped by the image — see the
Bracket docs and change them immediately after first login.

```yaml
- name: Deploy Bracket
  hosts: bracket
  become: true
  roles:
    - role: sage.final.bracket
      vars:
        bracket_fqdn: tournaments.example.org
```
