# openvpn

Installs and configures an OpenVPN server. Pulls certificates and keys from the PKI host and generates client configuration files.

## Requirements

- RHEL-based target hosts.
- A PKI host that has already generated OpenVPN certificates. Use the `step_ca` role with `step_ca_pki_mode: true`.
- The controller must be able to SSH to the PKI host as `openvpn_pki_user` using `openvpn_pki_ssh_key`, or the files must already be present in `files/openvpn/`.

## Role Variables

### Defaults (`defaults/main.yml`)

| Variable | Default | Description |
|----------|---------|-------------|
| `openvpn_port` | `1194` | OpenVPN listener port. |
| `openvpn_protocol` | `udp` | Transport protocol (`udp` or `tcp`). |
| `openvpn_dev` | `tun` | Device type. |
| `openvpn_subnet` | `10.8.0.0` | VPN subnet. |
| `openvpn_netmask` | `255.255.255.0` | VPN netmask. |
| `openvpn_cidr` | `10.8.0.0/24` | VPN CIDR. |
| `openvpn_cert_dir` | `/etc/openvpn/certs` | Where certs/keys are stored on the server. |
| `openvpn_pki_host` | `pki.example.com` | Hostname of the PKI host. |
| `openvpn_pki_user` | `pki` | SSH user on the PKI host. |
| `openvpn_pki_ssh_key` | (required) | Path to the SSH private key used to fetch certs from the PKI host. |
| `openvpn_pki_cert_path` | `/etc/step-ca/certs` | Path to certificates on the PKI host. |
| `openvpn_pki_ca_path` | `/etc/step-ca` | Path to CA chain on the PKI host. |
| `openvpn_clients` | `[client1, client2]` | List of client names to generate configs for. |
| `openvpn_client_config_dir` | (required) | Controller directory where client certs/keys/.ovpn files are generated. |
| `openvpn_keepalive` | `10 120` | Keepalive ping settings. |
| `openvpn_cipher` | `AES-256-GCM` | Data channel cipher. |
| `openvpn_tls_version_min` | `1.2` | Minimum TLS version. |
| `openvpn_push_dns` | `[]` | DNS resolvers pushed to clients (`dhcp-option DNS`). Empty leaves the client resolvers untouched. |
| `openvpn_push_search_domain` | `""` | DNS suffix pushed to clients (`dhcp-option DOMAIN`). Empty pushes nothing. |
| `openvpn_redirect_gateway` | `false` | Push `redirect-gateway def1 bypass-dhcp` (full tunnel). When `false` only the VPN subnet is routed. |
| `openvpn_server_public_ip` | `198.51.100.1` | Public IP clients connect to. |
| `openvpn_server_private_ip` | `10.8.0.1` | Private IP of the OpenVPN server. |

## Example Playbook

```yaml
- hosts: vpn
  become: true
  roles:
    - role: openvpn
```

## Example Group Variables

```yaml
openvpn_server_public_ip: "198.51.100.1"
openvpn_server_private_ip: "10.8.0.1"
openvpn_subnet: "10.8.0.0"
openvpn_netmask: "255.255.255.0"
openvpn_cidr: "10.8.0.0/24"
openvpn_clients:
  - client1
  - client2
# Hand the internal resolvers to the clients and route everything through
# the tunnel. Without these, clients keep their own DNS and only internal
# names served by the lab (split-horizon) become unreachable.
openvpn_push_dns:
  - "192.0.2.53"
  - "192.0.2.54"
openvpn_push_search_domain: "lab.example.com"
openvpn_redirect_gateway: true
```

## Notes

- The role fetches certificates from `openvpn_pki_host` as `openvpn_pki_user` using `openvpn_pki_ssh_key`.
- Client certs, keys and `.ovpn` files are generated on the controller into `openvpn_client_config_dir`.
- Use the `firewall` role to open `openvpn_port/udp` and enable masquerade for `openvpn_cidr`.
- Enable `firewall_ip_forward: true` when routing VPN traffic to other networks.
- `openvpn_push_dns` / `openvpn_push_search_domain` / `openvpn_redirect_gateway` are server-side pushes evaluated at connect time: re-run the role and reconnect the client, no need to regenerate the `.ovpn` files.
- Pushed options are only applied by clients that honor DHCP options (Tunnelblick and the OpenVPN GUI do; a bare `openvpn` CLI needs an `up`/`down` script). If internal names still resolve to their public addresses, check the client's resolvers first.
