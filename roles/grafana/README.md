# grafana

Deploys Grafana on a dedicated host and provisions Prometheus data sources and dashboards.

## Requirements

- Rocky Linux target host.
- Network reachability to the Prometheus servers defined in `grafana_prometheus_group`.

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `grafana_port` | `3000` | Grafana listen port. |
| `grafana_admin_user` | `admin` | Default admin username. |
| `grafana_data_dir` | `/var/lib/grafana` | Grafana data directory. |
| `grafana_config_dir` | `/etc/grafana` | Grafana configuration directory. |
| `grafana_prometheus_group` | `prometheus` | Inventory group containing Prometheus hosts. |
| `grafana_prometheus_port` | `9090` | Prometheus HTTP port. |
| `grafana_dashboards_src` | (required to enable dashboards) | Control-node path with dashboard JSON files; dashboards are skipped when unset. |
| `grafana_firewall_zone` | `public` | firewalld zone for the Grafana service. |

## Dependencies

- Firewall rules are applied via the `firewall` role using `grafana_firewall_zone`.

## Example Playbook

```yaml
- hosts: grafana
  become: true
  roles:
    - grafana
```

## Dashboards

Set `grafana_dashboards_src` to a control-node directory containing dashboard JSON files. They will be copied to `/var/lib/grafana/dashboards/` and provisioned automatically. Dashboards are skipped when `grafana_dashboards_src` is unset.
