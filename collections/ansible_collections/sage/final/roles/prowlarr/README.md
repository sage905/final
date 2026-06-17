# sage.final.prowlarr

Run **Prowlarr** (the indexer manager for the *arr stack) routed through the
Surfshark VPN gateway container, so all indexer/tracker queries leave only
through the tunnel (kill switch).

## Why behind the VPN

Prowlarr is where the *sensitive* external traffic happens — it queries
indexers and trackers. Sonarr/Radarr can run on the normal network and still
keep those queries tunneled, because Prowlarr syncs its indexers into them as
proxy URLs pointing back at Prowlarr: a search goes Sonarr → Prowlarr (in the
VPN namespace) → tracker (through the tunnel).

## How it works

- Prowlarr joins the VPN container's network namespace
  (`network_mode: "container:{{ prowlarr_vpn_container }}"`). Its Web UI
  (`prowlarr_webui_port`, default `9696`) is reached through the VPN container's
  name — add that port to the surfshark role's `surfshark_input_ports`, and
  point Caddy at `{{ prowlarr_vpn_container }}:{{ prowlarr_webui_port }}`.
- It reaches Sonarr/Radarr (on the shared network) by name; gluetun's
  `FIREWALL_OUTBOUND_SUBNETS` already allows the docker subnets.
- Set `prowlarr_vpn_container: ""` to run it on the shared `web` network instead.

## Wiring it up

In Prowlarr → Settings → Apps, add Sonarr (`http://sonarr:8989`) and Radarr
(`http://radarr:7878`); set the Prowlarr server URL the apps call back on to
`http://{{ prowlarr_vpn_container }}:{{ prowlarr_webui_port }}`.

## Requirements

- `sage.final.surfshark` deployed first (the VPN gateway must be running).

See `defaults/main.yml` for all options.
