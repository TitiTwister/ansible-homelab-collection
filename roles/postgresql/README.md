# postgresql

Ansible role to deploy a highly available PostgreSQL cluster using **Patroni**,
**etcd** and **HAProxy**, with optional **pgBackRest** backups to S3-compatible
object storage.

## Architecture

- 3 PostgreSQL nodes managed by Patroni
- etcd running collocated on the same 3 nodes for DCS/quorum
- HAProxy TCP passthrough routes port `5432` to the current Patroni leader
- Each node uses a dedicated data disk mounted at `/var/lib/pgsql`
- Backups, when enabled (`postgresql_backup_enabled: true`):
  - pgBackRest installed on every node, WAL archived continuously by the
    leader (`archive_mode: on`; pgBackRest disallows `always`)
  - scheduled backups run only on the current Patroni leader: the systemd
    templated unit `pgbackrest@{full,diff}.service` uses `ExecCondition`
    against the Patroni REST API `/primary` endpoint; standbys skip cleanly
    and failovers shift backups automatically
  - single S3 repository, optional AES-256-CBC client-side encryption
  - the PITR window equals the retention of full backups

## Variables

See `defaults/main.yml` for all options. The most important ones to override are:

```yaml
postgresql_cluster_name: "postgresql"
postgresql_group: "postgresql"

postgresql_superuser_username: "postgres"
postgresql_superuser_password: "CHANGE_ME_IN_VAULT"
postgresql_replication_password: "CHANGE_ME_IN_VAULT"

postgresql_databases:
  - name: myapp
    owner: myapp

postgresql_users:
  - name: myapp
    password: "CHANGE_ME_IN_VAULT"
```

Note: the rendered `pg_hba.conf` always includes a
`local all postgres peer` rule so local tools (pgBackRest) can connect to
PostgreSQL over the unix socket.

### Networking

| Variable | Default | Description |
|---|---|---|
| `postgresql_hba_network` | `10.0.0.0/8` | Network allowed to reach PostgreSQL without TLS (pg_hba when TLS is disabled). |

### Backups (optional)

Set `postgresql_backup_enabled: true` plus the repository settings. The
`postgresql_archive_*` variables are derived automatically from the toggle.
Required:

| Variable | Description |
|---|---|
| `postgresql_backup_s3_bucket` | S3 bucket name |
| `postgresql_backup_s3_endpoint` | S3 endpoint, e.g. `s3.example.com` |
| `postgresql_backup_s3_region` | S3 region, e.g. `eu-west-2` |
| `postgresql_backup_s3_key` | Access key (vault) |
| `postgresql_backup_s3_key_secret` | Secret key (vault) |
| `postgresql_backup_cipher_pass` | AES-256-CBC passphrase (vault, required when `postgresql_backup_cipher_enabled`) |

Optional:

| Variable | Default | Description |
|---|---|---|
| `postgresql_backup_stanza` | `main` | Stanza name |
| `postgresql_backup_s3_prefix` | `/pgbackrest` | Repository prefix inside the bucket |
| `postgresql_backup_cipher_enabled` | `true` | Client-side repository encryption |
| `postgresql_backup_retention_full` | `4` | Full backups to retain (PITR window) |
| `postgresql_backup_full_calendar` | `Sun *-*-* 01:00:00` | Full backup schedule (systemd `OnCalendar`) |
| `postgresql_backup_diff_calendar` | `Mon..Sat *-*-* 01:00:00` | Differential backup schedule |
| `postgresql_backup_check_calendar` | `*-*-* 05:00:00` | Repository check schedule |
| `postgresql_backup_pg1_path` | `postgresql_data_dir` | PGDATA backing the stanza |
| `postgresql_backup_user` | `postgres` | User running backups and checks |
| `postgresql_backup_seed_first_backup` | `true` | Run a first full backup on first apply |
| `postgresql_backup_packages` | `[pgbackrest, curl]` | Packages installed for backups |
| `postgresql_backup_pushgateway_urls` | `[]` | Pushgateway URLs receiving backup job metrics from the `pgbackrest-push` wrapper; empty disables metric pushes |

Backup tasks are tagged `backup`: use `--tags backup` to target them or
`--skip-tags backup` to skip them.

## Requirements

- Rocky Linux / RHEL / AlmaLinux with the PGDG repository (installed by the role)
- A dedicated data disk per VM, configured via the `common` role's `disks` variable
- The `firewall` role opens the required ports
- For backups: an S3-compatible bucket and a dedicated access key

## Usage

The role is applied by `playbooks/site.yml` to the `postgresql` inventory group.

**First enablement of backups**: run the full role (not `--tags backup`
alone) — changing `archive_mode` restarts PostgreSQL on every node (brief
failovers expected), and the first full backup is seeded automatically.

## Backup operations

```bash
# List backups (any node)
sudo -u postgres pgbackrest --stanza=main info

# Verify repository and WAL archiving round trip
sudo -u postgres pgbackrest --stanza=main check

# Trigger a backup manually on the leader
sudo systemctl start pgbackrest@full.service

# Inspect scheduled timers
systemctl list-timers 'pgbackrest@*' pgbackrest-check.timer
```

## Restore runbook

### Point-in-time recovery of the whole cluster

Prerequisites: the cluster is down or being rebuilt, and the bucket
credentials are configured on the target node (`/etc/pgbackrest/pgbackrest.conf`).

1. Stop Patroni on all nodes:

   ```bash
   systemctl stop patroni
   ```

2. Remove the old cluster state from the DCS so Patroni re-bootstraps from
   the restored data directory (run on one node):

   ```bash
   patronictl -c /etc/patroni/patroni.yml remove <cluster_name>
   ```

3. Restore on the node that will become the new primary, as the postgres user.
   `--delta` reuses unchanged blocks; `--type=time --target` performs PITR:

   ```bash
   sudo -u postgres pgbackrest --stanza=main --delta \
     --type=time '--target=2026-09-04 12:00:00+02' restore
   ```

   Without `--type`/`--target` this restores the latest backup.

4. Start Patroni on that node; it adopts the restored data directory as the
   leader of a fresh cluster:

   ```bash
   systemctl start patroni
   ```

5. On the remaining nodes, wipe their stale data directory contents, then
   start Patroni; they clone from the restored leader:

   ```bash
   rm -rf /var/lib/pgsql/17/data/*
   systemctl start patroni
   ```

### Single node rebuild

No pgBackRest involved: Patroni reclones a lost node from the running leader
via streaming replication automatically.
