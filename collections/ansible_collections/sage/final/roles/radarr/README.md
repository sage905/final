Sage.Final Radarr Role
======================

Runs [Radarr](https://radarr.video/) (movie PVR) as a Docker Compose stack whose
**entire network is provided by the Surfshark VPN gateway**
(`sage.final.surfshark`). Radarr joins that container's network namespace with
`network_mode: "container:surfshark"`, so all of its traffic exits through the
VPN — or is dropped by the gateway's kill switch. There is no path to the
internet that bypasses the VPN.

Requirements
------------

- Ubuntu 22.04 (Jammy) or 24.04 (Noble)
- Docker engine + Compose plugin
- **`sage.final.surfshark` deployed and running first** — Radarr cannot start
  without its VPN container (the role asserts this)
- `surfshark_input_ports` on the VPN role must include `7878` (the default does)
- `become: true`

Role Variables
--------------

See `defaults/main.yml` and `meta/argument_specs.yml`. Common options:

| Variable                  | Default                        | Purpose                                      |
|---------------------------|--------------------------------|----------------------------------------------|
| `radarr_vpn_container`    | `surfshark`                    | VPN container whose namespace it joins.      |
| `radarr_puid` / `_pgid`   | `1000` / `1000`                | Owner of media/downloads (linuxserver).      |
| `radarr_movies_path`      | `/srv/bulk/media/movies`       | Movie library, mounted at `/movies`.         |
| `radarr_downloads_path`   | `/srv/bulk/downloads`          | Downloads, mounted at `/downloads`.          |
| `radarr_group_members`    | `[]`                           | Admins granted SFTP access to `/srv/radarr`. |

> **No `ports:` / `networks:` here.** A container sharing another's namespace
> can't have its own. The Radarr UI is reached through the VPN container — see
> below.

Accessing the UI (through Caddy)
--------------------------------

The web UI listens on `7878`, but inside the **VPN** container's namespace —
so Caddy proxies to `surfshark:7878`, not `radarr:7878`:

```yaml
caddy_sites:
  - { fqdn: "radarr.{{ root_domain }}", upstream: "surfshark:7878", internal_only: true }
```

Example Playbook
----------------

```yaml
- name: Deploy Radarr behind the VPN
  hosts: radarr
  become: true
  roles:
    - role: sage.final.radarr
      vars:
        radarr_puid: 1000
        radarr_pgid: 1000
```

Operations
----------

- **Confirm it's behind the VPN**:
  `docker exec radarr curl -s https://ipinfo.io/ip` should show the Surfshark
  exit IP, not your own.
- **Logs**: `docker compose -f /srv/radarr/docker-compose.yml logs -f`
- **If you recreate the VPN container**, Radarr's namespace disappears and it
  gets stuck restarting. Re-run `--tags radarr` (the bootstrap detects this and
  re-attaches it) or `cd /srv/radarr && docker compose up -d --force-recreate`.

License
-------

BSD-2-Clause
