Sage.Final Jellyfin Role
========================

Runs [Jellyfin](https://jellyfin.org/) as a podman-compose stack on the
shared `pterodactyl` network and registers it with the panel role's Caddy
via a `conf.d/` snippet.

Required: `jellyfin_fqdn`. Override `jellyfin_media_path` to point at your
existing media library (typically a NAS bind-mount); it defaults to
`/srv/jellyfin/media`.

DLNA discovery (UDP 7359) is **off** by default since Caddy fronts the web
UI. Set `jellyfin_publish_dlna_port: true` to publish the port on the host.

Hardware transcoding requires device passthrough (`/dev/dri`, `/dev/nvidia*`)
which this role does not configure — add via `vars` overrides or a custom
compose if needed.

```yaml
- name: Deploy Jellyfin
  hosts: jellyfin
  become: true
  roles:
    - role: sage.final.jellyfin
      vars:
        jellyfin_fqdn: media.example.org
        jellyfin_media_path: /mnt/nas/media
```
