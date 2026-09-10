# certificate

Requests TLS certificates from an internal `step-ca` ACME server using certbot with DNS-01 validation (RFC2136).

Certificates are stored in `/etc/ssl/certs/` and private keys in `/etc/ssl/private/`. The role also builds a certificate chain file including the root CA.

## Requirements

- Target hosts must reach the `step-ca` ACME directory.
- A BIND DNS server configured with RFC2136 dynamic updates and a TSIG key.
- The `bind` role (or equivalent) must provide these variables:
  - `bind_tsig_key_name`
  - `bind_tsig_key_secret`
  - `bind_tsig_key_algorithm`
- `dns_nameservers` should contain the primary DNS server for RFC2136 updates.

## Role Variables

### Defaults (`defaults/main.yml`)

| Variable | Default | Description |
|----------|---------|-------------|
| `certificate_step_ca_url` | `ca-1.example.com:443` | Address of the step-ca server. |
| `certificate_certbot_email` | `admin@example.com` | Email used for ACME registration. |
| `certificate_certbot_server` | `https://{{ certificate_step_ca_url }}/acme/acme/directory` | ACME directory URL. |
| `certificate_certbot_dns_plugin` | `dns-rfc2136` | Certbot DNS plugin. |
| `certificate_certbot_credentials_path` | `/etc/letsencrypt/rfc2136.ini` | Path to RFC2136 credentials file. |
| `certificate_certbot_propagation_seconds` | `10` | Seconds to wait after DNS update before validation. |
| `certificate_dns_server` | `{{ dns_nameservers[0] }}` | DNS server used for RFC2136 updates. |
| `certificate_default_duration` | `2160h` | Default certificate lifetime. |
| `certificate_key_size` | `2048` | Private key size. |
| `certificate_root_ca_url` | `https://{{ certificate_step_ca_url }}/roots.pem` | URL to download root CA certificate. |
| `certificate_root_ca_file` | `root_ca.crt` | Filename of the root CA certificate. |
| `certificate_certs_path` | `/etc/ssl/certs` | Where to install certificates. |
| `certificate_private_path` | `/etc/ssl/private` | Where to install private keys. |
| `certificate_renewal_threshold` | `864000` | If existing cert is valid for longer than this (seconds), skip re-issue. Default 10 days. |
| `certificate_certs` | `[]` | List of certificates to request. |

### Certificate list format

Each item in `certificate_certs` supports:

```yaml
certificate_certs:
  - name: prometheus
    common_name: "prometheus.example.com"
    san:
      - "prom-1.example.com"
      - "prom-2.example.com"
    duration: "8760h"
```

- `name` (required): Short name used for filenames and certbot `--cert-name`.
- `common_name` (required): Primary CN of the certificate.
- `san` (optional): List of Subject Alternative Names.
- `duration` (optional): Certificate lifetime. Falls back to `certificate_default_duration` if not set.

## Dependencies

- `certbot`
- `python3-certbot-dns-rfc2136`

These are installed by the role.

## Example Playbook

```yaml
- hosts: prometheus
  become: true
  roles:
    - role: certificate
```

## Example Group Variables

```yaml
# inventories/<site>/group_vars/prometheus/vars_certs.yml
certificate_certs:
  - name: prometheus
    common_name: "prometheus.example.com"
    san:
      - "prom-1.example.com"
      - "prom-2.example.com"
    duration: "8760h"

  - name: alertmanager
    common_name: "alertmanager.example.com"
    duration: "720h"
```

## Idempotency

The role checks whether `/etc/ssl/certs/{{ cert.name }}.crt` exists and is still valid for at least `certificate_renewal_threshold` seconds (default 10 days). If so, certbot is skipped.

If the certificate is missing or near expiry, the role removes the stale certbot state for that cert name and requests a new certificate.

## Forcing Re-issuance

To force a new certificate (for example after changing SANs), delete the final certificate files on the target host and re-run the playbook:

```bash
rm -f /etc/ssl/certs/prometheus.crt /etc/ssl/certs/prometheus-chain.crt /etc/ssl/private/prometheus.key
```

The role will then clean up certbot state and request a fresh certificate.

## Notes

- certbot stores its working files under `/etc/letsencrypt/`.
- The role copies the leaf certificate and private key from certbot to `/etc/ssl/certs/` and `/etc/ssl/private/`.
- A chain file named `{{ cert.name }}-chain.crt` is created by appending the root CA to the leaf certificate.
