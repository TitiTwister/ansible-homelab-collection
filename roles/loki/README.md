# loki

Installs and configures [Loki](https://grafana.com/oss/loki/) in single
binary mode: all targets (distributor, ingester, querier, compactor) in one
process with the tsdb schema and a filesystem object store on a dedicated
volume. Built for log aggregation from promtail shippers across the
internal network.

## Requirements

- Rocky Linux target host.
- A dedicated volume mounted at `/var/lib/loki` (format and mount it with
  the `common` role `disks` mechanism — see
  `inventories/<site>/group_vars/<loki_group>/vars_disks.yml`).
- Firewall rule for 3100 (apply via the `firewall` role) — inbound from the
  promtail pushers and the Grafana host. No authentication or TLS: keep the
  port restricted at the firewall/cloud security-group level.

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `loki_version` | `3.7.7` | Loki release (GitHub upstream binary). |
| `loki_port` | `3100` | HTTP API (push + query) listen port. |
| `loki_grpc_port` | `9096` | Internal gRPC listen port. |
| `loki_user` / `loki_group` | `loki` | System user/group. |
| `loki_config_dir` | `/etc/loki` | Configuration directory. |
| `loki_data_dir` | `/var/lib/loki` | Data directory (chunks, index, WAL, compactor). |
| `loki_retention_period` | `744h` | Compactor retention (31 days). |
| `loki_reject_old_samples_max_age` | `168h` | Reject journal entries older than this at ingestion. |
| `loki_ingestion_rate_mb` | `8` | Per-tenant ingestion rate limit (single-binary sizing). |
| `loki_ingestion_burst_size_mb` | `16` | Per-tenant ingestion burst limit. |

## Example Playbook

```yaml
- hosts: loki
  become: true
  roles:
    - role: loki
```

## Notes

- Single tenant (`auth_enabled: false`): fine for an internal,
  firewall-restricted deployment.
- `discover_service_name` is disabled: Loki would otherwise add a
  `service_name` label at push time (`unknown_service` for journald
  streams, which carry none of the mappable labels).
- The configuration is verified with `loki -verify-config` on every run.
- Retention is enforced by the compactor; the data volume must be large
  enough for `loki_retention_period` of ingested logs (plus WAL and index
  overhead).
