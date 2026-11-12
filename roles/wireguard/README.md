# wireguard

Installs and configures a WireGuard server. Generates server and client
keypairs on the server (idempotently, only when absent) and self-contained
client `.conf` files on the controller. No PKI is involved: WireGuard peers
authenticate with raw Curve25519 public keys. Static site-to-site peers
(routed tunnels to remote gateways) can be configured alongside the
roaming clients through `wireguard_site_peers`.

Runs happily alongside other VPN servers (e.g. the `openvpn` role) on the
same host: use a distinct UDP port and a distinct /24 subnet per VPN.

## Requirements

- RHEL-based target hosts (kernel >= 5.6 ships the WireGuard module).
- The `firewall` role to open `wireguard_port/udp`, bind the VPN subnet
  as a zone source, enable masquerade and `firewall_ip_forward: true`.

## Role Variables

### Defaults (`defaults/main.yml`)

| Variable | Default | Description |
|----------|---------|-------------|
| `wireguard_interface` | `wg0` | WireGuard interface name. |
| `wireguard_port` | `51820` | UDP listening port. |
| `wireguard_server_address` | `10.9.0.1/24` | Server interface address. /24 subnets only. |
| `wireguard_clients` | `[client1, client2]` | Client names to configure. |
| `wireguard_site_peers` | `[]` | Static site-to-site peers — see below. |
| `wireguard_dns` | `[8.8.8.8, 8.8.4.4]` | DNS servers pushed to clients. |
| `wireguard_client_allowed_ips` | `[0.0.0.0/0]` | Routes routed through the tunnel (full tunnel by default). |
| `wireguard_persistent_keepalive` | `25` | Client keepalive interval in seconds. |
| `wireguard_mtu` | `""` | Client MTU; auto-detected by wg-quick when empty. |
| `wireguard_server_public_ip` | `198.51.100.1` | Public IP/hostname clients connect to. |
| `wireguard_client_config_dir` | (required) | Controller directory where client `.conf` files are generated. |

## Example Playbook

```yaml
- hosts: vpn
  become: true
  roles:
    - role: wireguard
```

## Example Group Variables

```yaml
wireguard_clients:
  - laptop
  - phone
wireguard_server_address: "10.9.0.1/24"
wireguard_server_public_ip: "198.51.100.1"
wireguard_client_config_dir: "/path/to/files/wireguard"
```

## Site-to-Site Peers

`wireguard_site_peers` adds routed tunnels to remote gateways (another
WireGuard server, an OPNsense appliance, …) next to the roaming clients.
Each entry is rendered as one `[Peer]` in the server config with the
networks of `allowed_ips` routed to it; no keypair is generated on the
server and no `.conf` is produced on the controller — the remote side
owns its private key and configuration.

```yaml
wireguard_site_peers:
  - name: remote-gw            # comment in the server config
    ip: 10.9.0.10              # the peer's address inside the VPN /24
    public_key: "rM...="       # the peer's WireGuard public key
    allowed_ips:               # networks routed to the peer
      - "10.9.0.10/32"
      - "192.168.1.0/24"
```

`allowed_ips` defaults to `[ip/32]` (the roaming-client behaviour) when
omitted. Site peers share the VPN /24 with the roaming clients: pick a
tunnel IP beyond the client range (the server holds `.1`, clients take
`.2`, `.3`, …) so later additions to `wireguard_clients` cannot collide.

NAT is not performed for site peers: traffic from `allowed_ips` leaves
with its real source addresses, so the networks behind the peer need a
return route (or their own masquerade) on the other side.

## Notes

- The subnet must be a /24: the server takes `.1`, clients take `.2`,
  `.3`, ... in `wireguard_clients` list order. Renumbering clients means
  editing the list order and re-rolling the client configs.
- Keys are generated on the server with `wg genkey` only when the key
  files are absent; they are never regenerated on subsequent runs.
- NAT is delegated to firewalld masquerade: no `PostUp`/`PostDown` rules
  are written to the interface config.
- Client `.conf` files embed private keys: protect the controller-side
  `wireguard_client_config_dir` (e.g. ansible-vault) the same way as
  OpenVPN client keys.
- Removing a client from `wireguard_clients` removes it from the server
  config and stops generating its `.conf`, but leaves its private key on
  the server and its old `.conf` on the controller — delete both manually
  to fully revoke it.
