Sage.Final Sonarr Role
======================

Runs [Sonarr](https://sonarr.tv/) (TV series PVR) as a Docker Compose stack
whose **entire network is provided by the Surfshark VPN gateway**
(`sage.final.surfshark`). Sonarr joins that container's network namespace with
`network_mode: "container:surfshark"`, so all of its traffic exits through the
VPN — or is dropped by the gateway's kill switch. There is no path to the
internet that bypasses the VPN.

Requirements
------------

- Ubuntu 22.04 (Jammy) or 24.04 (Noble)
- Docker engine + Compose plugin
- **`sage.final.surfshark` deployed and running first** — Sonarr cannot start
  without its VPN container (the role asserts this)
- `surfshark_input_ports` on the VPN role must include `8989` (the default does)
- `become: true`

Role Variables
--------------

See `defaults/main.yml` and `meta/argument_specs.yml`. Common options:

| Variable                | Default                          | Purpose                                   |
|-------------------------|----------------------------------|-------------------------------------------|
| `sonarr_vpn_container`  | `surfshark`                      | VPN container whose namespace it joins.   |
| `sonarr_puid` / `_pgid` | `1000` / `1000`                  | Owner of media/downloads (linuxserver).   |
| `sonarr_tv_path`        | `/srv/bulk/media/tv`             | TV library, mounted at `/tv`.             |
| `sonarr_downloads_path` | `/srv/bulk/downloads`            | Downloads, mounted at `/downloads`.       |
| `sonarr_group_members`  | `[]`                             | Admins granted SFTP access to `/srv/sonarr`. |

> **No `ports:` / `networks:` here.** A container sharing another's namespace
> can't have its own. The Sonarr UI is reached through the VPN container — see
> below.

Accessing the UI (through Caddy)
--------------------------------

The web UI listens on `8989`, but inside the **VPN** container's namespace —
so Caddy proxies to `surfshark:8989`, not `sonarr:8989`:

```yaml
caddy_sites:
  - { fqdn: "sonarr.{{ root_domain }}", upstream: "surfshark:8989", internal_only: true }
```

Example Playbook
----------------

```yaml
- name: Deploy Sonarr behind the VPN
  hosts: sonarr
  become: true
  roles:
    - role: sage.final.sonarr
      vars:
        sonarr_puid: 1000
        sonarr_pgid: 1000
```

Operations
----------

- **Confirm it's behind the VPN**: in Sonarr, a quick check is that its
  namespace IP matches the gateway — `docker exec sonarr curl -s https://ipinfo.io/ip`
  should show the Surfshark exit IP.
- **Logs**: `docker compose -f /srv/sonarr/docker-compose.yml logs -f`
- **If you recreate the VPN container**, Sonarr's namespace disappears and it
  gets stuck restarting. Re-run `--tags sonarr` (the bootstrap detects this and
  re-attaches it) or `cd /srv/sonarr && docker compose up -d --force-recreate`.

License
-------

BSD-2-Clause
