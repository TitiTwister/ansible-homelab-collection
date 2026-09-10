# bind

Installs and configures BIND as an authoritative DNS server with primary/secondary setup, forward and reverse zones, and RFC2136 dynamic updates.

## Requirements

- RHEL-based target hosts (Rocky/AlmaLinux).
- Two name servers: one primary (`bind_primary_ns`) and one secondary (`bind_secondary_ns`).
- Network access between primary and secondary for zone transfers.

## Role Variables

### Defaults (`defaults/main.yml`)

| Variable | Default | Description |
|----------|---------|-------------|
| `bind_zone_name` | `home.lab` | Forward zone name. |
| `bind_zone_ttl` | `3600` | Default TTL for zone records. |
| `bind_zone_refresh` | `3600` | SOA refresh interval. |
| `bind_zone_retry` | `600` | SOA retry interval. |
| `bind_zone_expire` | `604800` | SOA expire interval. |
| `bind_zone_minimum` | `86400` | SOA minimum TTL. |
| `bind_soa_email` | `admin.home.lab` | SOA admin email. |
| `bind_soa_serial` | `2026040701` | SOA serial number. |
| `bind_primary_ns` | `ns-1` | Hostname of the primary name server. |
| `bind_secondary_ns` | `ns-2` | Hostname of the secondary name server. |
| `bind_primary_ip` | `192.168.1.1` | IP of the primary name server. |
| `bind_secondary_ip` | `192.168.1.2` | IP of the secondary name server. |
| `bind_forwarders` | `[1.1.1.1, 9.9.9.9]` | External DNS forwarders. |
| `bind_allowed_nets` | `[192.168.0.0/16, 127.0.0.0/8]` | Networks allowed to query. |
| `bind_reverse_zones` | `[]` | List of reverse zones to configure. |
| `bind_records` | `[]` | List of A/PTR host records. |
| `bind_tsig_key_name` | `certbot-updates` | TSIG key name for RFC2136 updates. |
| `bind_tsig_key_algorithm` | `hmac-sha256` | TSIG algorithm. |
| `bind_tsig_key_secret` | (empty) | TSIG secret. Override in vault. |
| `bind_config_dir` | `/etc` | BIND configuration directory. |
| `bind_zones_dir` | `/var/named` | BIND zones directory. |

### Record format

`bind_records` accepts dictionaries with `name`, `ip`, and optional aliases/CNAMEs.

## Example Playbook

```yaml
- hosts: dns
  become: true
  roles:
    - role: bind
```

## Example Group Variables

```yaml
bind_zone_name: "example.com"
bind_primary_ns: "ns-1"
bind_secondary_ns: "ns-2"
bind_primary_ip: "192.168.1.11"
bind_secondary_ip: "192.168.1.12"
bind_forwarders:
  - "1.1.1.1"
bind_allowed_nets:
  - "192.168.0.0/16"
  - "127.0.0.0/8"
bind_records:
  - name: "prometheus"
    ip: "192.168.1.10"
```

## Notes

- The TSIG secret should be stored in an Ansible Vault file.
- Dynamic updates are allowed only from `bind_primary_ip`.
- The secondary is configured via zone transfers from the primary.
