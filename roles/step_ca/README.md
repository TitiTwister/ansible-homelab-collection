# step_ca Role

Ansible role to install step-ca and configure a Root CA and Intermediate CA on Rocky Linux 10.

## Requirements

- Rocky Linux 10 (or other RHEL-based distribution)
- Internet access to download step-ca and step-cli from Smallstep

## Role Variables

See `defaults/main.yml` for default values.

| Variable | Description | Default |
|----------|-------------|---------|
| `step_ca_root_ca_name` | Subject name for the Root CA | "Example Root CA" |
| `step_ca_intermediate_ca_name` | Subject name for the Intermediate CA | "Example Intermediate CA" |
| `step_ca_install_path` | Directory for CA files | "/etc/step-ca" |
| `step_ca_key_type` | Key type (EC or RSA) | "EC" |
| `step_ca_key_size` | Key size | 256 (for EC P-256) |
| `step_ca_root_validity` | Root CA validity period | "87600h" (10 years) |
| `step_ca_intermediate_validity` | Intermediate CA validity period | "43800h" (5 years) |
| `step_ca_controller_files_dir` | Controller directory where the PKI material is stored; files may be ansible-vault encrypted (read via lookup) | (required) |

## Generated Files

The role creates the following files in `/etc/step-ca/`:

- `root_ca.crt` - Root CA certificate (public)
- `root_ca.key` - Root CA private key (protected, mode 0600)
- `intermediate_ca.crt` - Intermediate CA certificate (public)
- `intermediate_ca.key` - Intermediate CA private key (protected, mode 0600)

## Example Playbook

```yaml
---
- name: Configure step-ca PKI
  hosts: vault
  become: true
  roles:
    - role: step_ca
```

## Running the Playbook

```bash
ansible-playbook playbooks/step_ca.yml
```

## Notes

- This role installs step-ca binaries to `/usr/local/bin/`
- No systemd service is configured (step-ca does not run as a service)
- The role is idempotent - running it multiple times will not recreate existing CA certificates
