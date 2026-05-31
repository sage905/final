# sage.final.wireguard_client

Configures a host as a split-tunnel WireGuard client peering with a single
external gateway (typically an OPNsense WireGuard "Instance"). The role:

1. Installs `wireguard` and `wireguard-tools`.
2. Generates a Curve25519 keypair on the host (first run only).
3. Prints the public key via `debug` so you can paste it into OPNsense's
   peer config.
4. Renders `/etc/wireguard/wg0.conf` with one `[Peer]` block.
5. Enables and starts `wg-quick@wg0`.

## Required variables

| Variable                                  | Description                                                    |
|-------------------------------------------|----------------------------------------------------------------|
| `wireguard_client_address`                | This host's VPN IP in CIDR form, e.g. `10.8.0.5/24`.           |
| `wireguard_client_peer_public_key`        | OPNsense instance's public key.                                |
| `wireguard_client_peer_endpoint`          | `host:port` of the OPNsense WireGuard endpoint.                |
| `wireguard_client_peer_allowed_ips`       | List of CIDRs to route through the tunnel (split-tunnel scope).|

## Optional

| Variable                                       | Default | Notes                          |
|------------------------------------------------|---------|--------------------------------|
| `wireguard_client_interface`                   | `wg0`   |                                |
| `wireguard_client_listen_port`                 | `0`     | 0 = ephemeral.                 |
| `wireguard_client_peer_preshared_key`          | `""`    | Vault it if used.              |
| `wireguard_client_peer_persistent_keepalive`   | `25`    | NAT keepalive; 0 to disable.   |

## First-run flow

1. Set the required vars in host_vars / group_vars on the production
   inventory.
2. Run `--tags wireguard_client`. The play prints
   `<host> wireguard pubkey: <key>` for every host.
3. Add a peer for each host in OPNsense (VPN → WireGuard → Peers), pasting
   the pubkey and the matching tunnel address.
4. Re-run the play. Tunnel comes up; the play is now fully idempotent.

## Example group_vars/wireguard_clients.yml

```yaml
wireguard_client_peer_public_key: "abcd...="
wireguard_client_peer_endpoint: "vpn.example.com:51820"
wireguard_client_peer_allowed_ips:
  - 192.168.1.0/24
  - 10.0.0.0/24
```

Per-host address in host_vars:

```yaml
wireguard_client_address: "10.8.0.5/24"
```
