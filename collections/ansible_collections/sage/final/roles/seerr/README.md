Sage.Final Seerr Role
=====================

Runs [Seerr](https://seerr.dev/) as a Docker Compose stack. Seerr is the
**unified successor to Overseerr and Jellyseerr** (both now deprecated) — a
request & discovery frontend for your media library. Users browse and request
movies/TV; Seerr hands those requests to **Radarr** and **Sonarr** to fetch,
then surfaces them in **Jellyfin** (it also supports Plex and Emby).

Where it sits in the network
----------------------------

Unlike Sonarr/Radarr, Seerr is **not** routed through the VPN by default. It is a
frontend, not a downloader — it needs to be reachable, it talks to Jellyfin on
the local network, and its only outbound traffic is to the TMDB metadata API. So
by default it runs on the shared `web` network with its own address and Caddy
proxies straight to `seerr:5055`.

It reaches the other apps by their network names:

| Talks to | At |
|---|---|
| Sonarr   | `http://surfshark:8989` (Sonarr lives in the VPN container's namespace) |
| Radarr   | `http://surfshark:7878` (same) |
| Jellyfin | `http://jellyfin:8096` |

> **Optional:** set `seerr_vpn_container: surfshark` to force Seerr's traffic
> through the VPN too (same model as Sonarr/Radarr). Then add `5055` to the
> surfshark role's `surfshark_input_ports` and point Caddy at `surfshark:5055`.
> Most setups don't need this.

Requirements
------------

- Ubuntu 22.04 (Jammy) or 24.04 (Noble)
- Docker engine + Compose plugin
- The shared `web` network (`sage.final.caddy`)
- Sonarr/Radarr (`sage.final.sonarr` / `.radarr`) and Jellyfin reachable, to
  wire up in Seerr's settings after first run
- `become: true`

Role Variables
--------------

See `defaults/main.yml` and `meta/argument_specs.yml`. Common options:

| Variable               | Default            | Purpose                                       |
|------------------------|--------------------|-----------------------------------------------|
| `seerr_vpn_container`  | `""` (own network) | Set to `surfshark` to route through the VPN.   |
| `seerr_log_level`      | `info`             | `debug` / `info` / `warn` / `error`.           |
| `seerr_group_members`  | `[]`               | Admins granted SFTP access to `/srv/seerr`.     |

Accessing the UI (through Caddy)
--------------------------------

```yaml
caddy_sites:
  - { fqdn: "requests.{{ root_domain }}", upstream: "seerr:5055" }
  # If seerr_vpn_container is set, use "surfshark:5055" instead.
```

Example Playbook
----------------

```yaml
- name: Deploy Seerr
  hosts: seerr
  become: true
  roles:
    - role: sage.final.seerr
```

First-run setup
---------------

1. Run the role and open the UI. Sign in with your **Jellyfin** account.
2. Add your Jellyfin server, then add the **Radarr** (`surfshark:7878`) and
   **Sonarr** (`surfshark:8989`) services with their API keys (from each app's
   Settings → General).
3. Set quality profiles / root folders so approved requests download to the same
   library paths Sonarr/Radarr and Jellyfin use.

> **Migrating from Overseerr/Jellyseerr?** Seerr migrates an existing config
> automatically on first start — point `/srv/seerr/config` at the old config
> data before the first run. See the [migration guide](https://docs.seerr.dev/migration-guide/).

Operations
----------

- **Logs**: `docker compose -f /srv/seerr/docker-compose.yml logs -f`
- **Config on disk**: `/srv/seerr/config` (mounted at `/app/config`).

License
-------

BSD-2-Clause
