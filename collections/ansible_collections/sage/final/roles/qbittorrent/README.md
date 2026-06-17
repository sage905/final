# sage.final.qbittorrent

Run **qBittorrent** (the download client) routed through the Surfshark VPN
gateway container, so every peer and tracker connection leaves only through the
tunnel (kill switch). It shares the same PUID/PGID and the same `/downloads`
path as Sonarr/Radarr, so completed downloads import as instant **hardlinks**
rather than copies.

## How it works

- qBittorrent joins the VPN container's network namespace
  (`network_mode: "container:{{ qbittorrent_vpn_container }}"`), so it has no
  network of its own. Its Web UI (`qbittorrent_webui_port`, default `8080`) is
  reached through the VPN container's name — add that port to the surfshark
  role's `surfshark_input_ports`, and point Caddy at
  `{{ qbittorrent_vpn_container }}:{{ qbittorrent_webui_port }}`.
- Set `qbittorrent_vpn_container: ""` to run it on the shared `web` network
  instead (not recommended for a torrent client).
- The `/downloads` mount (`qbittorrent_downloads_path`) **must** equal
  `sonarr_downloads_path` / `radarr_downloads_path`. The tree itself is owned by
  `sage.final.disk_pool`.

## First login

The linuxserver image generates a temporary Web UI password on first start —
read it from the container logs and change it in the UI:

```bash
sudo docker logs qbittorrent | grep -i password
```

## Requirements

- `sage.final.surfshark` deployed first (the VPN gateway must be running).
- Surfshark does not offer port forwarding, so incoming peer connections are
  limited; outgoing connectivity is unaffected.

See `defaults/main.yml` for all options.
