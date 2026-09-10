# haproxy

Installs and configures HAProxy as a hostname-based reverse proxy / load balancer for the monitoring stack, with optional TLS termination.

## Requirements

- Rocky Linux target host.
- DNS records pointing the configured hostnames to the HAProxy host.
- Network reachability from HAProxy to the backend servers.
- When TLS is enabled, certificates must be requested before this role runs (e.g. by the `certificate` role). The role expects `certificate_certs` to be defined.

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `haproxy_port` | `80` | HTTP listen port. |
| `haproxy_stats_port` | `8404` | HAProxy stats page port. |
| `haproxy_stats_user` | `admin` | Stats page auth username. |
| `haproxy_tls_enabled` | `true` | Enable HTTPS termination and HTTP-to-HTTPS redirect. |
| `haproxy_certs_dir` | `/etc/haproxy/certs` | Directory containing PEM bundles (one per certificate name). |
| `haproxy_backends` | see `defaults/main.yml` | List of HTTP backend definitions (name, hostname, group/servers, port, healthcheck, default). |
| `haproxy_tcp_backends` | `[]` | List of TCP frontend/backend definitions (port-based pass-through). |

Define `haproxy_stats_password` in inventory (vault) to enable stats page
authentication. The stats listener also exposes the native Prometheus
exporter at `http://<host>:<stats_port>/metrics`; scrape it as job `haproxy`
(with the same credentials as `basic_auth` if needed).

## Backend definition format

```yaml
haproxy_backends:
  - name: grafana
    hostname: "grafana.example.com"
    group: grafana
    port: 3000
    healthcheck: "GET /api/health"
    default: true
```

Backend servers come from the inventory `group`, or from an explicit
`servers` list of literal addresses (useful when the target is a virtual IP
such as a MetalLB LoadBalancer address rather than inventory hosts):

```yaml
haproxy_backends:
  - name: git
    hostname: "git.example.com"
    servers:
      - 192.168.1.230
    port: 80
    healthcheck: "GET /api/healthz"
```

## TCP backend definition format

```yaml
haproxy_tcp_backends:
  - name: k8s_api
    frontend:
      bind: "*:6443"
      mode: tcp
      # optional: overrides the global 50s default, e.g. for long-lived
      # sessions (git over SSH, SQL connections)
      timeout_client: 2h
    backend:
      mode: tcp
      balance: roundrobin
      servers_group: k8s_masters
      port: 6443
      # optional: health-check port when it differs from the service port
      check_port: 8008
      # optional: HTTP health check (e.g. Patroni REST API)
      httpchk: true
      httpchk_uri: /leader
      # optional: replaces servers_group with literal addresses
      # servers:
      #   - 192.168.1.230
      # optional: overrides the global 50s default
      timeout_server: 2h
```

## Example Playbook

```yaml
- hosts: haproxy
  become: true
  roles:
    - certificate
    - haproxy
```

## Notes

- When TLS is enabled, port 80 redirects all traffic to HTTPS on port 443.
- HAProxy selects the correct certificate automatically from `haproxy_certs_dir/` based on the TLS SNI / HTTP Host header.
- Backends remain on plain HTTP; TLS is terminated at HAProxy.
- Requests with an unmatched `Host` header fall back to the default backend.
