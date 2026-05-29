# sage.final.borg

Nightly [borgmatic](https://torsion.org/borgmatic/) backups of `/srv/`
to a Hetzner Storage Box subaccount. Excludes `/srv/bulk`. Retains
30 daily / 12 monthly / 10 yearly archives. Pings Healthchecks on every
run when `borg_healthchecks_url` is set.

Uses the borgmatic package's systemd service + timer with a drop-in
that overrides `OnCalendar` to the configured schedule (default:
nightly at 03:00 with a 30-minute randomized smear).

## Required variables

| Variable | Description |
|---|---|
| `borg_ssh_host`   | Hetzner host, e.g. `u123456.your-storagebox.de` |
| `borg_ssh_user`   | Subaccount username, e.g. `u123456-sub1` |
| `borg_passphrase` | Repokey passphrase. **Vault it.** Lose it and the repo is unreadable. |

## Recommended optional variables

| Variable | Description |
|---|---|
| `borg_healthchecks_url` | Healthchecks ping URL. Get from your Healthchecks instance (`New Check → Ping URL`). Pings on start, finish, fail; uploads the log as ping body. |

## First-deploy flow

1. **Create the Hetzner Storage Box subaccount** in the Hetzner Robot UI.
   Enable SSH support on it (subaccounts have it off by default).

2. **Deploy Healthchecks first** if you want monitoring:
   `ansible-navigator run site.yml --tags healthchecks --eei ee-demo`

   In Healthchecks: New project → New check → name it (e.g. `borg-cotest`)
   → copy the **Ping URL**.

3. **Set inventory vars** including `borg_healthchecks_url`.

4. **Run the role.** It generates an ed25519 keypair on the host and
   prints the public key:

   ```
   ansible-navigator run site.yml --tags borg --eei ee-demo
   ```

   The init task at the end will *fail* with "Permission denied" on the
   very first run — that's expected, because Hetzner doesn't have the
   key yet.

5. **Paste the printed public key** into the subaccount's SSH Keys
   panel in the Hetzner Robot UI.

6. **Re-run** `--tags borg`. The init probe now succeeds, the role
   initializes the repo (`borgmatic init --encryption repokey`), and the
   systemd timer is left enabled.

7. **Trigger a test run** immediately instead of waiting for 03:00:

   ```
   ssh root@<host> 'systemctl start borgmatic.service && journalctl -u borgmatic.service -f'
   ```

   Healthchecks should flip green within a few seconds of `start` and
   stay green after `finish`.

## Operational commands (run on the target host)

```bash
# Status / next scheduled run
systemctl list-timers borgmatic.timer

# Last run's output
journalctl -u borgmatic.service -e

# Manual on-demand backup
systemctl start borgmatic.service

# Ad-hoc borgmatic — finds /etc/borgmatic/config.yaml automatically
sudo borgmatic --verbosity 1 create prune

# List archives
sudo borgmatic list

# Show repo stats
sudo borgmatic info

# Mount the latest archive (browse it like a filesystem)
sudo mkdir -p /mnt/borg && sudo borgmatic mount --archive latest --mount-point /mnt/borg
# ...browse /mnt/borg, then:
sudo borgmatic umount --mount-point /mnt/borg

# Restore a specific path from the latest archive
sudo borgmatic extract --archive latest --path srv/wordpress/wordpress/wp-content
```

## Disaster recovery

What you need off-host to restore:

1. The `borg_passphrase`. **There is no recovery if this is lost.**
2. SSH access to the Hetzner subaccount (a copy of the private key, or
   the ability to register a new one in the UI).
3. The repo URL — derivable from `borg_ssh_user@borg_ssh_host:port/./borg-<host>`.

Encryption mode is `repokey` by default, which embeds the keyfile inside
the repo. The encrypted keyfile travels with the backup; only the
passphrase needs to be kept out-of-band.

## A nicer restore UI

For browsing archives and clicking-to-extract from a desktop:
[Vorta](https://vorta.borgbase.com/) — Qt app for Linux/macOS. Add the
Hetzner repo, paste passphrase, point at the SSH key. No web UI for borg
restore exists that's actively maintained; Vorta is the best option.

## Important caveats

- `keep_daily/monthly/yearly` are applied **per host prefix**
  (`{hostname}-*`), so multiple hosts backing up to the same Hetzner
  subaccount under different repo names won't interfere with each
  other's retention. If you change `inventory_hostname`,
  previously-named archives will no longer be matched by prune and will
  accumulate. Use `borg_repo_name` to override the repo path explicitly
  if you rename the host.
- The role assumes Hetzner has SSH borg support on the chosen
  subaccount. Some older Storage Box plans don't — check the Hetzner
  feature matrix if the init step fails with "command not found".
- First backup of a populated `/srv/` is slow (no dedup history yet);
  budget for it. Subsequent runs are fast — borg dedups at the chunk
  level across archives.
- Upstream borgmatic.service has `ProtectHome=read-only` and
  `ReadOnlyPaths=/root /home/`. Since this role backs up `/srv/` only,
  those don't matter. If you change `borg_source_paths` to include
  `/home`, add an override drop-in that loosens those protections.
