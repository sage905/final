# Changelog

All notable changes to the `sage.final` collection and its deployment playbooks.
Format loosely follows [Keep a Changelog](https://keepachangelog.com/). This
project is not yet versioned, so entries are grouped under **Unreleased**.

## [Unreleased]

### Changed
- **Migrated the entire estate from rootful Podman to rootful Docker.** All app
  roles now use `docker compose`, `docker network`, and
  `community.docker.docker_container_exec` in place of their Podman equivalents.
  A new shared `sage.final.docker` role (wrapping `geerlingguy.docker`) installs
  the engine, Compose v2 plugin, and Docker SDK for Python once per host from a
  dedicated play at the top of `site.yml`. Each app role keeps a gated
  `<role>_install_docker` flag (renamed from `<role>_install_podman`, now
  defaulting `false`). Portainer mounts `/var/run/docker.sock`; Wings units
  depend on `docker.service` and drop the Podman `DOCKER_HOST` override; Cockpit
  drops the `cockpit-podman` plugin. App data is bind-mounted, so the runtime
  swap moves no data. (`4329dd5`, `5ff9f24`)
- **Game-server paths** moved to Pelican-native locations
  (`/etc/pelican`, `/var/log/pelican`, `/srv/gameservers`,
  `/srv/bulk/gamebackups`). (`e50ac90`)
- **Documentation** updated for the Podman→Docker switch and the
  Pterodactyl→Pelican migration. (`3e1bdb0`)

### Added
- **Media stack behind a VPN** — three new roles wired into `site.yml`:
  - `sage.final.surfshark` runs a **Surfshark VPN gateway** (gluetun configured
    for the Surfshark provider) with a kill switch, DNS-over-TLS, and a
    healthcheck. It joins the shared `web` network and opens the *arr UI ports
    on its firewall (`surfshark_input_ports`).
  - `sage.final.sonarr` and `sage.final.radarr` (linuxserver.io images) run with
    `network_mode: "container:surfshark"`, so **all their traffic exits only
    through the VPN** — no leak path. Their bootstrap asserts the VPN container
    is up first and self-heals the namespace coupling if the VPN container was
    recreated. Caddy fronts their UIs via the VPN container
    (`surfshark:8989` / `surfshark:7878`).
- **MySQL database-host role** (`sage.final.mysql`) — runs a dedicated MariaDB
  container as a Docker Compose stack to serve as the panel's "Database Host",
  provisioning a database per game server. Creates a global-privilege `panel`
  management user (auto-generated, persisted secrets), publishes 3306 on the
  host so remote Wings nodes/game servers can connect, and ships **Adminer** as
  the modern phpMyAdmin replacement fronted by the shared Caddy. Wired into
  `site.yml` under the `mysql` tag/host group.
- **Pelican Wings role** (`sage.final.pelican_wings`) — installs the upstream
  Wings binary and systemd unit as the Pelican-native game-server daemon,
  alongside the existing Pelican Panel role. (`03bc360`, `e50ac90`)
- **Custom fail2ban filters/jails** in the `hardening` role via
  `hardening_fail2ban_filters` + `hardening_fail2ban_extra_jails`, with a worked
  example banning Pelican Wings SFTP brute-force on port 2022 (journald-backed,
  `wings.service`). (`6e20120`)
- **Migration playbook** `migrate_podman_to_docker.yml` plus docs
  `docs/Podman-to-Docker-Migration.md` and `docs/Wings-Mono-Segfault-Fix.md`.
  (`4329dd5`)
- **Day-to-day container management guide** `docs/Container-Management.md`.
  (`d6221f2`)

### Fixed
- Pelican Panel container networking and `docker-compose.yml.j2` template.
  (`f9476dd`, `f145abc`)
- Pelican Panel user/UID handling so the `www-data` (UID 82) container can write
  its SQLite DB, `.env`, and logs. (`67624a9`)
- Variable name collision in the `docker` role. (`5ff9f24`)
