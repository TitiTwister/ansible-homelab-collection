# netbox

Installs [NetBox](https://docs.netbox.dev/) — the IPAM/DCIM source of
truth — as the [netbox-docker](https://github.com/netbox-community/netbox-docker)
compose stack: NetBox (granian) + RQ worker, PostgreSQL and two Valkey
instances (task queue + cache), all colocated on one host. The app
image is built locally with plugins baked in (default:
[netbox-topology-views](https://github.com/maveteodoro/netbox-topology-views)
and [netbox-data-flows](https://github.com/aabanis/netbox-data-flows));
everything persistent lives on one data disk so the stack lifts with
the volume.

## Requirements

- Rocky Linux 9/10 target host with the container runtime already
  applied: the `docker` role, with `docker_containerd_root` on the
  same data disk (the locally built image is multi-GB).
- A data volume mounted at `netbox_root` (default `/var/lib/netbox`)
  with the `common` role `disks` mechanism (e.g.
  `inventories/<site>/group_vars/<group>/vars_disks.yml`).
- Outbound HTTPS to docker.io and pypi.org (base image, plugin
  packages).
- Firewall rule for the UI port when fronted by a reverse proxy
  (apply via the `firewall` role).

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `netbox_image_tag` | `v4.7-5.1.2` | Upstream base image line: `v<netbox>-<netbox-docker release>`. Both halves matter — the image support files must match the compose layout this role deploys. |
| `netbox_image_name` | `local/netbox` | Name of the locally built image carrying the plugins. |
| `netbox_postgres_version` | `18` | PostgreSQL image major version. |
| `netbox_valkey_version` | `9.1` | Valkey image version (task queue + cache). |
| `netbox_plugins` | topology-views + data-flows, pinned | Plugins baked into the image: each entry is a `pip` requirement line and the Django `module` name. |
| `netbox_plugins_config` | coordinate saving for topology-views | Rendered into `PLUGINS_CONFIG` in `configuration/plugins.py`. |
| `netbox_listen_port` | `8000` | Host port publishing the container's 8080. |
| `netbox_root` | `/var/lib/netbox` | Deployment root on the data disk: compose project, PostgreSQL, media, Valkey. |
| `netbox_compose_dir` | `{{ netbox_root }}/docker` | Compose project directory (also the docker build context). |
| `netbox_allowed_hosts` | `[]` (required) | `ALLOWED_HOSTS`: load-balancer hostname(s) plus anything reaching NetBox directly. |
| `netbox_csrf_trusted_origins` | `[]` (required) | `CSRF_TRUSTED_ORIGINS`, with the `https://` scheme. |
| `netbox_superuser_name` | `admin` | Initial superuser, created on first start (idempotent). |
| `netbox_superuser_email` | `admin@example.com` | Initial superuser email. |
| `netbox_secret_key` | `""` (required) | Django `SECRET_KEY`, 50+ random characters. |
| `netbox_postgres_password` | `""` (required) | PostgreSQL `netbox` user password. |
| `netbox_redis_password` | `""` (required) | Valkey auth password (both instances). |
| `netbox_superuser_password` | `""` (required) | Initial superuser password. |
| `netbox_housekeeping_on_calendar` | `*-*-* 04:00:00` | systemd timer calendar for the daily housekeeping run. |

## Example Playbook

```yaml
- hosts: netbox
  become: true
  roles:
    - role: netbox
```

## Notes

- This role does not install the container runtime: apply the
  `docker` role first, with `docker_containerd_root` on the data
  disk (Docker 28 image store — the locally built image is
  multi-GB and does not fit the system volume).
- The stack layout (compose file, env-driven `configuration.py`,
  entrypoint behavior, health checks) mirrors the upstream
  netbox-docker release branch pinned by `netbox_image_tag`. When
  bumping the tag, diff the upstream `configuration/` and
  `docker-compose.yml` against the role templates and keep them in
  sync.
- The first deployment builds the plugin image and runs the database
  migrations inside the container: expect several minutes before the
  login page answers. The verify phase waits for it.
- Persistent state is a set of bind mounts under `netbox_root`
  (PostgreSQL, media/reports/scripts, Valkey AOF), pre-owned to match
  the container users (`netbox` uid 999 gid 0, postgres uid 70,
  valkey uid 999 gid 1000) and SELinux-labeled via `:z`.
- Housekeeping (changelog/session pruning, cache cleanup) has no
  container in current netbox-docker releases; this role runs
  `manage.py housekeeping` daily through a host systemd timer.
- Upgrades: bump `netbox_image_tag` (and plugin pins if needed), rerun
  the role — the image is rebuilt, `docker compose up` recreates the
  containers and the entrypoint applies the database migrations.
  Back up `netbox_root` (at least `postgres/` and `media/`) first.
