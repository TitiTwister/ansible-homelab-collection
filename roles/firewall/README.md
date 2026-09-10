# firewall

Installs and configures `firewalld` rules. Supports ports, predefined services, custom services, source networks, masquerade, and IP forwarding.

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
