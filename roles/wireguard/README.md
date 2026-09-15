# wireguard

Installs and configures a WireGuard server. Generates server and client
keypairs on the server (idempotently, only when absent) and self-contained
client `.conf` files on the controller. No PKI is involved: WireGuard peers
authenticate with raw Curve25519 public keys.

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
