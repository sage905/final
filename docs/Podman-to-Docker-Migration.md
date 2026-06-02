# Migrating the estate from rootful Podman to rootful Docker

**Applies to:** the `sage.final` collection on Ubuntu 24.04.
**Why:** Pelican/Pterodactyl Wings target the Docker Engine API; running them on
Podman's Docker-compatible socket leaks quirks (e.g. tmpfs `tmpcopyup`
behaviour) that don't exist on Docker. Everything else in the collection is
plain `docker-compose.yml` over the same socket, so consolidating on one
runtime — Docker — removes a two-runtime host and the compat-shim risk.

The estate is **rootful** Podman, so Podman's headline rootless-security
advantage was already not in play; the move is a near-even trade that buys
first-class runtime support.

## What changed in the collection

- **New shared role `sage.final.docker`** — installs the rootful Docker Engine +
  Compose v2 plugin (via `geerlingguy.docker`, added to `requirements.yml`) plus
  the Docker SDK for Python. It runs **once per host** from a dedicated play at
  the top of `site.yml`, so the application plays don't each re-install Docker.
  Each container role also keeps a gated `include_role` of it, but defaults
  `<role>_install_docker: false` (renamed from `<role>_install_podman`) — flip a
  role's flag to `true` only to install Docker when running that role standalone.
- **App roles** (caddy, dashy, glance, bracket, jellyfin, healthchecks,
  portainer, pelican_panel, pterodactyl_panel, wordpress):
  - `podman-compose …` → `docker compose …`
  - `podman network create` → `docker network create`
  - `containers.podman.podman_container_exec` → `community.docker.docker_container_exec`
    (note: the parameter is `container:`, not `name:`)
  - Portainer now mounts `/var/run/docker.sock` instead of `/run/podman/podman.sock`.
- **Wings roles** (pelican_wings, pterodactyl_wings):
  - unit file depends on `docker.service` (was `podman.socket`) and drops the
    `DOCKER_HOST=unix:///run/podman/podman.sock` override — Wings uses the
    default `/var/run/docker.sock`.
- **Cockpit** drops the `cockpit-podman` plugin (no maintained Docker
  equivalent); use Portainer for container management.

### No data migration

Every compose volume in the collection is a **host bind mount**
(`{{ <role>_data_dir }}/…`). Nothing lives in Podman's named-volume store, so
the runtime swap moves no data. Docker re-pulls images on first start.

## Running the migration

```bash
ansible-navigator run migrate_podman_to_docker.yml --eei ee-demo \
  -i inventory/hosts.yml
```

The playbook:

1. **Tears down Podman** on every host that has it — stops Wings, stops and
   removes all Podman containers (including game-server containers), then stops
   and **disables** `podman.socket` and the `podman` service. Packages and
   `/var/lib/containers` are **left in place** so the change is reversible.
2. **Imports `site.yml`**, which installs Docker and brings every stack back up.

Skip the apt dist-upgrade with `--skip-tags update`. To converge the Docker side
separately, run the teardown alone with `--tags migrate_podman_to_docker`, then
`ansible-navigator run site.yml`.

> First, install the role dependency: `ansible-galaxy install -r requirements.yml`
> (lands in `./roles`, per `roles_path` in `ansible.cfg`). For AAP, point the
> project at `requirements.yml`.

## Rollback

Podman is only stopped/disabled, not removed. To revert a host:

```bash
docker compose -p <stack> down      # in each {{ *_data_dir }}
systemctl enable --now podman.socket
# then re-run the pre-migration (Podman) revision of the collection
```

## Note: the Wings `no-new-privileges` patch still applies

Switching to Docker does **not** affect the Mono/Unity segfault fix in
[Wings-Mono-Segfault-Fix.md](Wings-Mono-Segfault-Fix.md): `SecurityOpt:
no-new-privileges` is hardcoded in Wings' `container.go` and Docker honours it
too. The patched Wings binary is still required for Rust/Unity game servers.
