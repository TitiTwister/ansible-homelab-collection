# promtail

Installs and configures [promtail](https://grafana.com/docs/loki/latest/send-data/promtail/)
to tail the local systemd journal and push every entry to Loki. Deployed
on every log source, including DMZ hosts — the push model only needs
outbound connectivity to the Loki host.

The promtail user is added to the `systemd-journal` group for read access
to all journal files. Entries get three labels: `job` (`journald`,
promtail's journald target does not derive one from `job_name`), `host`
(journal hostname) and `unit` (systemd unit); entries with journald
priority 0-3 additionally get `level="error"`. Ephemeral
`session-*.scope` and `user-*.slice` units are dropped — they would
create unbounded label cardinality (one new stream per login, cron job
or ansible task).

## Requirements

- Rocky Linux target host.
- `promtail_loki_push_url` set by the inventory (the Loki push API
  endpoint, see `group_vars/all/vars_promtail.yml`).

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `promtail_version` | `3.7.7` | Promtail release (shipped in the Loki GitHub releases). |
| `promtail_loki_push_url` | `""` | Loki push API endpoint; **required**, the role asserts it is set. |
| `promtail_user` / `promtail_group` | `promtail` | System user/group. |
| `promtail_config_dir` | `/etc/promtail` | Configuration directory. |
| `promtail_data_dir` | `/var/lib/promtail` | Data directory (positions file). |
| `promtail_http_port` | `9080` | Local status API (loopback only). |
| `promtail_journal_max_age` | `24h` | On first start only ship journal entries newer than this. |

## Example Playbook

```yaml
- hosts: all:vpn_gateway
  become: true
  roles:
    - role: promtail
```

## Notes

- The configuration is verified with `promtail -check-syntax` on every run.
- `max_age` only applies when no positions file exists yet: restarts
  resume exactly where they left off.
- Query the logs in Grafana with e.g. `{job="journald", host="..."}` or
  `{job="journald", level="error"}` for error-level entries.
