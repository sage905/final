Sage.Final Dashy Role
=====================

Runs [Dashy](https://dashy.to/) as a podman-compose stack on the shared
`pterodactyl` network and registers it with the panel role's Caddy via a
`conf.d/` snippet.

Required: `dashy_fqdn`. After first deploy, edit
`/srv/dashy/user-data/conf.yml` to define your sections and items.

```yaml
- name: Deploy Dashy
  hosts: dashy
  become: true
  roles:
    - role: sage.final.dashy
      vars:
        dashy_fqdn: dash.example.org
```
