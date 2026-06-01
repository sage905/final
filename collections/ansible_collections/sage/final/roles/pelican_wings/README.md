Sage.Final Pelican Wings Role
=============================

Installs the [Pelican Wings](https://pelican.dev/docs/wings/install) daemon —
the Pelican project's fork of Pterodactyl Wings — on an Ubuntu game-server node.
Wings runs the actual game-server containers on behalf of the Pelican Panel.

This is the Pelican counterpart to `sage.final.pterodactyl_wings`. It installs
the same way (a single binary + a systemd unit) but uses Pelican's paths and
release source.

Requirements
------------

- Ubuntu 22.04 (Jammy) or 24.04 (Noble)
- A running Pelican Panel this node will register against
- `become: true`

Role Variables
--------------

See `defaults/main.yml` and `meta/argument_specs.yml`. Highlights:

| Variable | Default | Description |
|---|---|---|
| `pelican_wings_version` | `latest` | Release tag (e.g. `1.0.0-beta25`) or `latest`. |
| `pelican_wings_config_yaml` | `""` | Contents of `config.yml`; paste from the Panel if empty. |
| `pelican_wings_config_dir` | `/etc/pelican` | Where `config.yml` lives. |
| `pelican_wings_enable_swap_accounting` | `true` | Adds `swapaccount=1` to GRUB (reboot required). |
| `pelican_wings_install_podman` | `true` | Install Podman and expose its Docker-compatible socket. |

Bootstrapping a node
--------------------

1. Run the role. It installs Podman (Wings is pointed at Podman's
   Docker-compatible socket via `DOCKER_HOST`), downloads the Wings binary,
   and installs the `wings` systemd unit. The service is enabled but stays
   stopped while `config.yml` is missing.
2. In the Pelican Panel UI, **Admin → Nodes → Create Node**, fill in this
   host's FQDN and ports. Set the **Daemon Server File Directory** to
   `/srv/gameservers/volumes` so server files land under the role-managed
   `pelican_wings_data_dir` (`/srv/gameservers`). If you leave the panel
   default, Wings writes to that default instead and ignores `/srv/gameservers`.
3. After copying `config.yml` to the host, set the backup path in it so
   server backups go to the role-managed `pelican_wings_backup_dir`:

   ```yaml
   system:
     backup_directory: /srv/bulk/gamebackups
   ```

   `/srv/bulk` is excluded from the offsite borg backup, so game-server
   backups aren't re-uploaded. The panel UI doesn't expose this field — edit
   `config.yml` directly (or bake it into `pelican_wings_config_yaml`).
3. Open the new node's **Configuration** tab and copy the generated
   `config.yml` to `/etc/pelican/config.yml` (or pass it as
   `pelican_wings_config_yaml`, vaulted, on a re-run).
4. `systemctl start wings`.

Switching a host from Pterodactyl Wings to Pelican Wings
--------------------------------------------------------

Pelican Wings and Pterodactyl Wings use the **same** binary path
(`/usr/local/bin/wings`) and the **same** systemd service name (`wings`), so a
host runs one or the other — not both. Deploying this role on a node that
previously ran `sage.final.pterodactyl_wings` **replaces** the binary and unit
in place; the only functional change is the config directory
(`/etc/pterodactyl` → `/etc/pelican`).

To switch:

1. Move the node from the `pterodactyl_wings` inventory group into a
   `pelican_wings` group.
2. **Pin `pelican_wings_version` to a release tag** (e.g. `1.0.0-beta25`) for
   the switch. The binary download only overwrites an existing
   `/usr/local/bin/wings` when a version is pinned (`force`); with the default
   `latest` and a Pterodactyl binary already in place, the download is skipped
   and the old binary would remain. (Re-runs with the same pin are idempotent —
   `get_url` compares content and won't restart Wings unless the binary
   actually changed.)
3. Run `--tags pelican_wings`. The role replaces the binary and the `wings`
   unit and leaves the service **stopped** (the old `/etc/pterodactyl/config.yml`
   is not reused, so the previous Pterodactyl Wings stops here — game servers it
   managed go offline until the new node config is in place).
4. Re-create the node in the **Pelican** Panel and drop its `config.yml` into
   `/etc/pelican/config.yml`, then `systemctl start wings`.
5. **Remove the leftover Pterodactyl Wings network.** Pterodactyl Wings creates
   a `pterodactyl_nw` bridge on `172.18.0.0/16`; Pelican Wings wants the same
   default subnet for its own `pelican0` network and will FATAL with
   `subnet 172.18.0.0/16 is already used` until the old one is gone. Once no
   game servers depend on it:

   ```bash
   sudo podman network rm pterodactyl_nw
   sudo systemctl restart wings
   ```

6. Once the new node is confirmed working, the now-unused
   `/etc/pterodactyl/config.yml` can be removed.

> Game servers do not migrate automatically — they're registered against a
> specific panel. Recreate your servers in the Pelican Panel after switching.

> The `pelican` system user collision and the `pterodactyl_nw` subnet clash
> above both stem from a host having previously run the Pterodactyl stack. The
> role pre-creates the `pelican` user to avoid the first; the network must be
> removed by hand (it may still hold servers, so the role won't delete it).

Operations
----------

- **Status / logs**: `systemctl status wings`,
  `journalctl -u wings -f`, or tail `/var/log/pelican/`.
- **Update the binary**: bump `pelican_wings_version` (or re-run with `latest`)
  and re-run the role; the handler restarts Wings.

License
-------

BSD-2-Clause
