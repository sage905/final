# sage.final.portainer

Run Portainer CE as a containerized admin UI for the host's rootless
podman stacks. Lives behind the shared Caddy. Marked `internal_only` by
default in the example inventory because Portainer is effectively
root-equivalent (it controls the podman socket).

## Required variables

None — the role's defaults render a working compose file. To expose
Portainer through Caddy, add a `caddy_sites` entry in your Caddy
inventory:

```yaml
caddy_sites:
  - fqdn: "portainer.{{ root_domain }}"
    upstream: "portainer:9000"
    internal_only: true
```

## Migration from Ansible-deployed stacks

The stacks the other roles in this collection deploy
(`pterodactyl_panel`, `wordpress`, `dashy`, `glance`, `bracket`,
`jellyfin`, `caddy`) are created by `podman-compose`. Their containers
carry the `com.docker.compose.project=<name>` label that Portainer
recognizes.

After Portainer first-run setup (create the local admin user, accept the
"local environment" prompt — Podman socket is at the bind-mounted
location `/var/run/docker.sock`):

1. Open **Stacks** in the Portainer UI.
2. Each Ansible-deployed stack appears as **Limited** (orange icon).
   Limited means Portainer didn't deploy the stack and doesn't have the
   compose source, so it can manage individual containers but not
   `up`/`down` the whole stack as a unit.
3. **Recommended workflow:** leave them Limited and keep Ansible as the
   source of truth for compose files. Use Portainer for day-to-day
   container ops (restart, logs, exec, stats). When the compose changes,
   re-run the Ansible role — the role's handler recreates the stack and
   Portainer picks up the new containers.

If you want a stack to be **fully** Portainer-managed (orange → green,
Editor tab usable, recreate button enabled), the steps are:

1. Copy the rendered compose from `/srv/<service>/docker-compose.yml`.
2. In Portainer: **Stacks → Add stack**, name it identically to the
   existing stack (e.g. `wordpress`), paste the compose content, Deploy.
3. Portainer detects the matching container labels, claims them as
   members of the new stack, and the previous Limited stack disappears.

**Caveat:** once a stack is Portainer-managed, re-running the matching
Ansible role will cause Ansible to recreate containers under
podman-compose's project label. Portainer will see the stack revert to
Limited until you re-import. **Pick one source of truth per stack and
stick to it.**

## RBAC

Portainer CE supports **teams**: create a team per service group
(e.g. `wordpress-admins`), assign humans to it, grant team-level access
to specific stacks/containers. This implements the `service_admins`
intent at the application layer (Portainer login) rather than at the
unix layer (SSH/SFTP).

Set this up by hand in the Portainer UI after first login; there's no
Ansible-driven way to provision it that's cheaper than just clicking
through the first time.

## Update

Re-run with `--tags portainer` to bump the image (defaults to
`portainer-ce:lts`, follows the LTS channel).
