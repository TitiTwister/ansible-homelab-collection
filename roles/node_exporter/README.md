# node_exporter

Installs and configures Prometheus Node Exporter as a systemd service.

## Requirements

- RHEL-based target hosts.
- Internet access to download the Node Exporter binary from GitHub releases.

## Role Variables

### Defaults (`defaults/main.yml`)

| Variable | Default | Description |
|----------|---------|-------------|
| `node_exporter_version` | `1.8.2` | Node Exporter version to install. |
| `node_exporter_port` | `9100` | TCP port Node Exporter listens on. |
| `node_exporter_user` | `node_exporter` | System user for Node Exporter. |
| `node_exporter_group` | `node_exporter` | System group for Node Exporter. |
| `node_exporter_firewall_zone` | `public` | Default firewalld zone for the Node Exporter port. |

## Dependencies

This role does not install a firewall rule. Use the `firewall` role to open port `node_exporter_port`.

## Example Playbook

```yaml
- hosts: prometheus
  become: true
  roles:
    - role: node_exporter
```

## Example Firewall Rule

```yaml
firewall_rules:
  - service: node_exporter
    port: "9100/tcp"
    zone: public
```

## Notes

- The binary is installed to `/usr/local/bin/node_exporter`.
- The systemd service file is deployed to `/etc/systemd/system/node_exporter.service`.
- Node Exporter is enabled and started automatically.
