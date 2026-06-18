# sage.final.flaresolverr

Run **FlareSolverr** — a headless-browser proxy that solves Cloudflare's
"checking your browser" challenge — alongside Prowlarr, so the *arr stack can
reach indexers/trackers protected by Cloudflare.

## Why it must share Prowlarr's network

When Cloudflare clears a challenge it issues a `cf_clearance` cookie **bound to
the requesting IP**. FlareSolverr therefore has to solve the challenge from the
**same exit IP Prowlarr queries the tracker from**, or the cookie it hands back
is rejected.

On this host Prowlarr runs on the shared `web` network (only qBittorrent is
tunnelled through the VPN), exiting via the host's real IP — so FlareSolverr
defaults to the shared `web` network too, and both exit via the same IP.

> If you instead route Prowlarr through the VPN
> (`prowlarr_vpn_container: surfshark`), set `flaresolverr_vpn_container:
> surfshark` as well so they share the tunnel's exit IP — see *Behind the VPN*
> below.

## How it works

- FlareSolverr runs on the shared `web` network with its own address. It keeps
  no persistent state (a throwaway browser per request) and no UI of its own.
- Prowlarr reaches it by name at
  `http://{{ flaresolverr_container_name }}:{{ flaresolverr_port }}`.
- qBittorrent does **not** use it — it speaks the BitTorrent protocol, not the
  indexer website. Sonarr/Radarr don't need it directly either: they query
  indexers *through* Prowlarr (`prowlarr:9696`), so only Prowlarr makes the
  Cloudflare-facing request.

## Wiring it up in Prowlarr

1. Prowlarr → **Settings → Indexers → Add (＋) → FlareSolverr**.
2. Host (Tags): `http://{{ flaresolverr_container_name }}:{{ flaresolverr_port }}`,
   give it a tag, e.g. `flaresolverr`.
3. Apply that tag to each Cloudflare-protected indexer.

## LAN access for troubleshooting (on by default)

So you can browse FlareSolverr's index page or POST test requests to `/v1` from
a workstation, its API is published on the host LAN by default via
`flaresolverr_published_ports`:

```yaml
flaresolverr_published_ports:
  - "{{ '{{' }} ansible_default_ipv4.address {{ '}}' }}:8191:8191"
```

Bound to the host's **primary LAN IP, not `0.0.0.0`**, because Docker's
published ports bypass the host firewall — this keeps the solver off any WAN
interface. Set `flaresolverr_published_ports: []` to disable. Test with:

```
curl -L -X POST http://<host>:8191/v1 \
  -H 'Content-Type: application/json' \
  -d '{"cmd":"request.get","url":"https://www.google.com","maxTimeout":60000}'
```

## Behind the VPN (alternative)

If Prowlarr is tunnelled, set `flaresolverr_vpn_container: surfshark`.
FlareSolverr then joins the VPN namespace and is reached over loopback at
`http://localhost:{{ flaresolverr_port }}`. In this mode it owns no ports of its
own, so `flaresolverr_published_ports` **must be empty** (the role asserts
otherwise); publish LAN access via the surfshark role instead — add
`{{ flaresolverr_port }}` to `surfshark_input_ports` and a LAN-bound mapping to
`surfshark_published_ports`.

## Requirements

- Prowlarr deployed (`sage.final.prowlarr`); the shared `web` network from
  `sage.final.caddy`.
- In VPN mode only: `sage.final.surfshark` running first.

See `defaults/main.yml` for all options.
