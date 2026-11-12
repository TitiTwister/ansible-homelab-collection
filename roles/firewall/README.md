# firewall

Installs and configures `firewalld`. Supports ports, predefined services, custom services, source networks, masquerade, IP forwarding, custom zones with interface binding, and firewalld policies for forwarding between zones.

## Requirements

- RHEL-based target hosts.
- `ansible.posix` collection for `firewalld` and `sysctl` modules.

## Role Variables

### Defaults (`defaults/main.yml`)

| Variable | Default | Description |
|----------|---------|-------------|
| `firewall_rules` | `[]` | List of firewall rules. Defined per host/group. |
| `firewall_default_zone` | `public` | Default zone for rules that do not specify one. |
| `firewall_ip_forward` | `false` | Enable IP forwarding (required for VPN routing). |
| `firewall_zones` | `[]` | Custom zones to create, with optional interface binding. |
| `firewall_policies` | `[]` | Firewalld policies forwarding between zone pairs. |

### Rule format

Each item in `firewall_rules` can define one of the following:

#### Port rule

```yaml
firewall_rules:
  - port: "22/tcp"
```

#### Service rule

```yaml
firewall_rules:
  - service: ssh
```

#### Custom service rule

```yaml
firewall_rules:
  - service: prometheus
    port: "9090/tcp"
```

#### Source network rule

```yaml
firewall_rules:
  - source: "192.168.0.0/16"
```

#### Masquerade rule

```yaml
firewall_rules:
  - masquerade: true
    zone: public
```

All rules support optional `zone`, `state` (`enabled` or `disabled`), and `immediate` settings.

### Custom zones (`firewall_zones`)

Zones that do not ship with firewalld, each with an optional interface binding:

```yaml
firewall_zones:
  - name: tunnel
    interface: wg0
```

Zones are created through `ansible.posix.firewalld`: `state: present`
with no parameter besides the zone is the module's zone-level operation —
idempotent and check-mode aware. Zone operations are permanent-only by
firewalld design.

### Policies (`firewall_policies`)

Firewalld policies forward between zone pairs that do not share a zone,
with optional rich rules and per-policy masquerade (SNAT to the egress
interface):

```yaml
firewall_policies:
  - name: public-to-tunnel
    ingress_zones: [public]
    egress_zones: [tunnel]
    rich_rules:
      - 'rule family=ipv4 source address=10.111.100.200 port port=80 protocol=tcp accept'
  - name: tunnel-to-public
    ingress_zones: [tunnel]
    egress_zones: [public]
    masquerade: true
    rich_rules:
      - 'rule family=ipv4 source address=192.168.93.0/24 accept'
```

`ansible.posix.firewalld` has no policy support, so policies are managed
with `firewall-cmd --permanent` behind query-then-apply tasks (a query
rc of 0 means the setting is present). The read-only queries run with
`check_mode: false` so `--check` runs stay usable.

## Example Playbook

```yaml
- hosts: prometheus
  become: true
  roles:
    - role: firewall
```

## Example Group Variables

```yaml
firewall_rules:
  - service: ssh
  - port: "9090/tcp"
  - service: prometheus
    port: "9090/tcp"
    zone: public
  - source: "192.168.0.0/16"
    zone: trusted

firewall_ip_forward: false
```

## Notes

- Custom service definitions are deployed to `/etc/firewalld/services/`.
- `firewalld` is reloaded after custom service definitions are created.
- IP forwarding is enabled when `firewall_ip_forward` is `true`.
- Zones and policies are written to the permanent configuration only;
  the `Reload firewalld` handler activates them at the end of the play.
- Zones and policies are presence-only: removing an entry does not
  delete the zone or policy from the host.
