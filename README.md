# tititwister.homelab

Ansible roles for running a homelab: PKI, DNS, load balancing, monitoring,
logging, a PostgreSQL HA cluster and Kubernetes. Targets EL-family systems
(Rocky/Alma/RHEL 9).

Every role is environment-agnostic: no inventory groups, hostnames or
networks are baked in. Site-specific values are injected from the inventory
of the play project that consumes this collection. For a complete working
example, see [ansible-play-homelab](https://github.com/TitiTwister/ansible-play-homelab).

## Roles

| Role | Description |
|---|---|
| `tititwister.homelab.bind` | BIND authoritative DNS (primary/secondary) with RFC2136 dynamic updates |
| `tititwister.homelab.certificate` | Certbot certificates against a step-ca ACME server |
| `tititwister.homelab.common` | Base host configuration: DNS resolvers, data disks |
| `tititwister.homelab.firewall` | firewalld rules from inventory data |
| `tititwister.homelab.grafana` | Grafana with provisioned datasources and dashboards |
| `tititwister.homelab.haproxy` | HAProxy HTTP/TCP load balancer with TLS |
| `tititwister.homelab.k8s` | Kubernetes cluster (kubeadm): control plane, workers, addons |
| `tititwister.homelab.loki` | Loki log aggregation, single binary with filesystem storage |
| `tititwister.homelab.node_exporter` | Prometheus node_exporter |
| `tititwister.homelab.openvpn` | OpenVPN server with per-client configurations |
| `tititwister.homelab.postgres_exporter` | Prometheus postgres_exporter |
| `tititwister.homelab.postgresql` | PostgreSQL HA cluster (Patroni + etcd) with TLS |
| `tititwister.homelab.prometheus` | Prometheus + Alertmanager with Telegram alerting |
| `tititwister.homelab.promtail` | Promtail journald shipper for Loki |
| `tititwister.homelab.pushgateway` | Prometheus Pushgateway for batch job metrics |
| `tititwister.homelab.step_ca` | step-ca PKI: offline root/intermediate CA and online CA server |

## Requirements

- ansible-core >= 2.18
- Collections: `ansible.posix`, `community.general`, `community.postgresql`

## Installation

Add to your play project's `requirements.yml`:

```yaml
---
collections:
  - name: https://github.com/TitiTwister/ansible-homelab-collection.git
    type: git
    version: v1.0.0
```

Then:

```bash
ansible-galaxy collection install -r requirements.yml
```

Roles are referenced by FQCN in plays:

```yaml
---
- name: Configure Prometheus
  hosts: prometheus
  become: true
  roles:
    - tititwister.homelab.prometheus
```

Each role documents its variables in its own `README.md`.

## License

[Beerware](LICENSE) — do whatever you want with this stuff; if we meet
some day and you think it's worth it, you can buy me a beer.
