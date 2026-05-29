# sage.final.healthchecks

Self-hosted [Healthchecks](https://healthchecks.io/docs/self_hosted/) —
cron-job monitoring. Each scheduled job pings a unique URL; if the ping
doesn't arrive within `period + grace`, Healthchecks notifies you.

Pairs with `sage.final.borg` (borgmatic's `monitoring.healthchecks`
config block pings on every backup run) and works for any other cron /
systemd-timer job on the network.

## Required variables

| Variable | Description |
|---|---|
| `healthchecks_site_root` | Public URL, e.g. `https://healthchecks.example.com`. Baked into ping URLs and emails. |
| `healthchecks_secret_key` | Django SECRET_KEY. **Vault it.** Generate with `python3 -c 'import secrets; print(secrets.token_urlsafe(50))'`. |

## Recommended setup

```yaml
healthchecks_site_root: "https://healthchecks.{{ root_domain }}"
healthchecks_secret_key: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  <ciphertext>
healthchecks_superuser_email: admin@example.com
healthchecks_superuser_password: !vault |
  $ANSIBLE_VAULT;1.1;AES256
  <ciphertext>
healthchecks_registration_open: false  # only the superuser can sign in
```

Add the site to Caddy as internal-only (Healthchecks's ping URLs are
unguessable but the admin UI shouldn't be public):

```yaml
caddy_sites:
  - fqdn: "healthchecks.{{ root_domain }}"
    upstream: "healthchecks:8000"
    internal_only: true
```

## First-deploy flow

1. Deploy via `--tags healthchecks`. Wait for the container to settle (~30s).
2. Browse to `healthchecks_site_root` from inside your network.
3. Log in with the superuser credentials you set.
4. **Projects → New project** → name it something like "Backups". Open it.
5. **Add Check** → name it (e.g. `borg-cotest`), schedule (`Cron` or `Simple`), grace period.
6. Copy the ping URL from the check's detail page — it looks like
   `https://healthchecks.example.com/ping/<uuid>`. Drop that into the
   borg role's `borg_healthchecks_url` variable, then re-run
   `--tags borg`. Borgmatic will start pinging on every backup.

## Notification channels

Set up in **Integrations** (per project). Free-tier options:

- **Email** — works out of the box if you configured `healthchecks_email_*`.
- **Slack / Discord / ntfy / Pushover / Telegram / Webhooks** — paste the
  channel URL/token, assign to checks. No additional config in Ansible.

## Upgrade

`:latest` tracks Healthchecks's master branch. Re-run `--tags healthchecks`
to pull the latest image. Django migrations run automatically on start.
For a pinned version, override `healthchecks_image` to a tagged release.

## Disaster recovery

Healthchecks state lives in `/srv/healthchecks/data/hc.sqlite`. Back up
that file (the borg role catches it as part of `/srv`) and you can
restore by dropping it back into a fresh container.
