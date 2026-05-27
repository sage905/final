Sage.Final Glance Role
======================

Runs [Glance](https://github.com/glanceapp/glance) as a podman-compose stack
on the shared `pterodactyl` network and registers it with the panel role's
Caddy via a `conf.d/` snippet.

Required: `glance_fqdn`. After first deploy, edit
`/srv/glance/config/glance.yml` to add pages and widgets.

```yaml
- name: Deploy Glance
  hosts: glance
  become: true
  roles:
    - role: sage.final.glance
      vars:
        glance_fqdn: home.example.org
```
