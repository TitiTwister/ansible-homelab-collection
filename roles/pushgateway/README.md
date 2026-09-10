# pushgateway

Installs and configures the Prometheus Pushgateway — the metrics cache that
bridges short-lived batch jobs (backups, cron jobs) to Prometheus. Jobs push
their result via HTTP; Prometheus scrapes the gateway with
`honor_labels: true` so pushed labels win.

Deploy on all Prometheus hosts; batch jobs push to every instance (the
gateways are independent, no clustering) and each Prometheus scrapes only
its local instance to avoid duplicate series.

## Requirements

- Rocky Linux target hosts.
- Firewall rule for 9091 (apply via the `firewall` role) — inbound from the
  hosts that push (backup jobs) and loopback/local scraping.

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `pushgateway_version` | `1.11.3` | Pushgateway release (GitHub upstream binary). |
| `pushgateway_port` | `9091` | Listen port. |
| `pushgateway_user` / `pushgateway_group` | `pushgateway` | System user/group. |
| `pushgateway_config_dir` | `/etc/pushgateway` | Configuration directory. |
| `pushgateway_data_dir` | `/var/lib/pushgateway` | Data directory (persistence file). |
| `pushgateway_persistence_file` | `{{ pushgateway_data_dir }}/pushgateway.dat` | Pushed metrics survive restarts. |
| `pushgateway_persistence_interval` | `5m` | Persistence flush interval. |
| `pushgateway_firewall_zone` | `public` | firewalld zone (apply rules via the `firewall` role). |

## Metrics convention

Pushing to `http://<gateway>:9091/metrics/job/<group>/instance/<name>`
stores all metrics of the push under that group; a new push to the same
group replaces it. For backup-style jobs use two groups:

- a per-run group (`<job>_run`) replaced on every run, success or failure:
  `<job>_last_run_success`, `<job>_last_run_duration_seconds`
- a success group (`<job>`) replaced only on success, so the
  `<job>_last_success_timestamp_seconds` staleness marker survives
  failing runs.

## Example Playbook

```yaml
- hosts: prometheus
  become: true
  roles:
    - role: pushgateway
```

## Notes

- No authentication or TLS; keep the port restricted at the firewall/cloud
  security-group level to known push sources and the local Prometheus.
- Pushgateway is not a durable database: enable the persistence file
  (default) so a service restart does not lose job metrics.
