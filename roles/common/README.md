# common

Applies common baseline configuration to all hosts. Currently configures DNS settings via NetworkManager when `dns_nameservers` is defined.

## Requirements

- RHEL-based target hosts with NetworkManager.
- `community.general.nmcli` collection for NetworkManager management.

## Role Variables

This role has no defaults. It expects the following variables to be defined in inventory group_vars:

| Variable | Required | Description |
|----------|----------|-------------|
| `dns_nameservers` | Yes | List of DNS server IPs to configure on the active connection. |
| `dns_search_domain` | Yes | DNS search domain to configure on the active connection. |
| `disks` | No | List of data disks to partition, format as XFS, and mount. |

### Data disk variables

Each entry in `disks` supports the following keys:

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `disks[].device` | Yes | — | Block device path, e.g. `/dev/sdb`. |
| `disks[].mount_point` | Yes | — | Directory where the filesystem will be mounted. |
| `disks[].partition` | No | `true` | Whether to create a single partition on the disk first. Set to `false` to format and mount the raw block device. |
| `disks[].mount_options` | No | `defaults` | Mount options passed to `/etc/fstab`. |
| `disks[].state` | No | `mounted` | State for `ansible.posix.mount`. |
| `disks[].allow_root_disk` | No | `false` | Set to `true` to allow managing `/dev/sda`-like devices. |

## Example Playbook

```yaml
- hosts: all
  become: true
  roles:
    - role: common
```

## Example Group Variables

```yaml
dns_nameservers:
  - "192.168.1.11"
  - "192.168.1.12"
dns_search_domain: "example.com"

# Optional data disk configuration
# disks:
#   - device: /dev/sdb
#     mount_point: /data
#     mount_options: defaults
```

## Notes

- The role only runs DNS configuration when `dns_nameservers` is defined.
- Disk formatting and mounting only runs when `disks` is defined.
- Disk setup is idempotent: existing partitions and XFS filesystems are not recreated.
- By default, the role refuses to manage root-like devices (`/dev/sda`, `/dev/vda`, `/dev/xvda`, `/dev/nvme0n1`) unless `allow_root_disk: true` is set for that disk.
- `/etc/fstab` entries use the filesystem UUID for stable device naming.
