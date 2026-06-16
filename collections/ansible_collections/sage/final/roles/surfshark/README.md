Sage.Final Surfshark Role
=========================

Runs a **Surfshark VPN gateway** as a Docker Compose stack, so other containers
can send all their internet traffic through the VPN. Under the hood it uses
[gluetun](https://github.com/qdm12/gluetun) configured for the Surfshark
provider — gluetun adds a **kill switch** (its firewall drops any traffic that
can't go through the tunnel), DNS-over-TLS, and a healthcheck.

Other containers attach with `network_mode: "container:surfshark"` and inherit
this container's network namespace — they have **no** route to the internet
except the VPN. In this collection, `sage.final.sonarr` and `sage.final.radarr`
do exactly that.

How the namespace sharing works
-------------------------------

A container that joins another's namespace **cannot** declare its own
`networks:` or `ports:`. So:

- This `surfshark` container joins the shared `web` network, and **its** firewall
  opens the apps' listening ports (`surfshark_input_ports`, default
  `8989,7878`).
- The shared Caddy reverse-proxies to the apps **by this container's name** — the
  Sonarr UI is at `surfshark:8989`, Radarr at `surfshark:7878` (the app
  containers themselves are not on any network).
- `surfshark_firewall_outbound_subnets` must include the Docker subnet (it
  defaults to all RFC1918 ranges) so Caddy's request gets a reply and LAN/media
  traffic stays off the VPN.

Requirements
------------

- Ubuntu 22.04 (Jammy) or 24.04 (Noble)
- Docker engine + Compose plugin (installed by `site.yml`'s docker play, or set
  `surfshark_install_docker: true` for a standalone run)
- The shared `web` network (`sage.final.caddy`) if you want the app UIs fronted
- A Surfshark subscription and its **service credentials** (Surfshark dashboard
  → VPN → Manual setup → Credentials — these are *not* your account login)
- `become: true`

Role Variables
--------------

See `defaults/main.yml` and `meta/argument_specs.yml`. Required (set vaulted in
inventory):

| Variable                     | Description                                   |
|------------------------------|-----------------------------------------------|
| `surfshark_openvpn_user`     | Surfshark service username (OpenVPN default). |
| `surfshark_openvpn_password` | Surfshark service password.                   |

(For `surfshark_vpn_type: wireguard`, set `surfshark_wireguard_private_key` and
`surfshark_wireguard_addresses` instead.)

Common options: `surfshark_server_countries` (e.g. `"Netherlands,Germany"`),
`surfshark_input_ports` (open a port here for every app you route through),
`surfshark_published_ports` (host access), `surfshark_group_members`.

Example
-------

```yaml
- name: Deploy the Surfshark VPN gateway
  hosts: surfshark
  become: true
  roles:
    - role: sage.final.surfshark
      vars:
        surfshark_openvpn_user: "{{ vault_surfshark_user }}"
        surfshark_openvpn_password: "{{ vault_surfshark_password }}"
        surfshark_server_countries: "Netherlands"
```

Then front the apps in `caddy_sites` (pointing at the **VPN** container):

```yaml
caddy_sites:
  - { fqdn: "sonarr.{{ root_domain }}", upstream: "surfshark:8989", internal_only: true }
  - { fqdn: "radarr.{{ root_domain }}", upstream: "surfshark:7878", internal_only: true }
```

Operations
----------

- **Confirm traffic is going through the VPN**:
  `docker exec surfshark wget -qO- https://ipinfo.io/ip` — should print a
  Surfshark exit IP, not your own.
- **Logs**: `docker compose -f /srv/surfshark/docker-compose.yml logs -f`
- **Health**: `docker inspect -f '{{ "{{" }}.State.Health.Status{{ "}}" }}' surfshark`
- **Recreating this container** changes its namespace, which breaks the apps
  attached to it — re-run `--tags sonarr,radarr` (or restart those stacks)
  afterwards. See those roles; their bootstrap self-heals this on the next run.

License
-------

BSD-2-Clause
