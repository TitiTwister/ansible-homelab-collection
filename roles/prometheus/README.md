# prometheus

Installs and configures a Prometheus + Alertmanager monitoring pair (HA) from
upstream binaries, with Telegram alerting. Deploy on all hosts of an inventory
group; the role wires Prometheus replicas to each other's Alertmanagers and
forms an Alertmanager gossip cluster automatically.

## Requirements

- Rocky Linux target hosts.
- A dedicated data directory (mount it on its own disk for retention safety).
- `telegram_bot_token` and `telegram_chat_id` defined in inventory (vault).
- Firewall rules for 9090 (Prometheus), 9093 (Alertmanager web), 9094
  (Alertmanager cluster gossip) and 9100 — apply via the `firewall` role.

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `prometheus_version` | `2.45.0` | Prometheus release (GitHub upstream binary). |
| `prometheus_port` | `9090` | Prometheus web port. |
| `prometheus_config_dir` | `/etc/prometheus` | Configuration and file_sd directory. |
| `prometheus_data_dir` | `/var/lib/prometheus` | TSDB data directory. |
| `prometheus_retention_time` | `15d` | TSDB retention. |
| `prometheus_alertmanager_version` | `0.27.0` | Alertmanager release. |
| `prometheus_alertmanager_port` | `9093` | Alertmanager web port. |
| `prometheus_alertmanager_cluster_port` | `9094` | Alertmanager gossip port. |
| `prometheus_alertmanager_config_dir` | `/etc/alertmanager` | Alertmanager configuration directory. |
| `prometheus_alertmanager_data_dir` | `/var/lib/alertmanager` | Alertmanager nflog/tfdata directory. |
| `prometheus_rules_dir` | `{{ prometheus_config_dir }}/rules` | Alert rule files directory. |
| `prometheus_group` | `prometheus` | Inventory group of Prometheus/Alertmanager hosts. |
| `prometheus_firewall_zone` | `public` | firewalld zone (apply rules via the `firewall` role). |

Inventory variables (not role defaults):

| Variable | Description |
|----------|-------------|
| `prometheus_targets_node` | Host list rendered into `file_sd/node.yml` (node_exporter). |
| `prometheus_targets_postgres` | Host list rendered into `file_sd/postgres.yml` (postgres_exporter). |
| `prometheus_scrape_configs` | Extra scrape jobs appended verbatim to `prometheus.yml` (e.g. patroni, haproxy, etcd, kubernetes). |
| `prometheus_k8s_credentials_src` | *undefined* | Local directory with the Kubernetes cluster CA (`ca.crt`) and monitoring ServiceAccount token (`token`); when set, deployed to `/etc/prometheus/k8s/` for the kubernetes_* scrape jobs. Populated by the k8s role (`monitoring_rbac.yml`). |
| `telegram_bot_token` / `telegram_chat_id` | Telegram receiver credentials (vault). |

## What gets deployed

- Prometheus: scrape jobs `prometheus`, `alertmanager`, plus file_sd jobs
  `node`/`postgres` and any `prometheus_scrape_configs` entries from
  inventory. Both replicas scrape everything; `external_labels.replica`
  distinguishes them.
- Alertmanager: HA cluster (each node gossips with the others on 9094),
  Telegram receiver with a custom HTML template, and inhibit rules that
  suppress warning-level alerts for an instance while its critical
  `InstanceDown` alert is firing.
- Alert rules: host, prometheus, alertmanager, HA, postgresql, patroni,
  etcd, haproxy, backup and kubernetes groups — rendered to `rules/alert_rules.yml`.
  Backup alerts consume pushgateway metrics pushed by the pgbackrest-push
  wrapper and the forgejo-backup CronJob (see the `pushgateway` role);
  kubernetes alerts consume the apiserver, kubelet, kube-state-metrics and
  longhorn scrape jobs.

## Example Playbook

```yaml
- hosts: prometheus
  become: true
  roles:
    - role: prometheus
```

## Notes

- The `hostname@ip` target format keeps hostnames as the `instance` label
  while scraping the IP.
- Prometheus/Alertmanager listen on plain HTTP; terminate TLS in front
  (e.g. HAProxy with certificates from the `certificate` role).
- After a config change, verify reload success with
  `prometheus_config_last_reload_successful == 1` and check
  `/-/targets` for scrape health.
