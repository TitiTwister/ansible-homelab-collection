# docker

Installs [Docker CE](https://docs.docker.com/engine/install/) from the
upstream repository with the compose v2 plugin, and optionally
relocates the containerd image store to a data disk. Extracted from
the container workload roles (`netbox`, `openwebui`) so the runtime is
installed by one shared role instead of once per app role.

This role deliberately does **not** deploy `daemon.json`: on GPU hosts
the `nvidia_docker` role owns it (nvidia runtime) and provides the
whole runtime — do not apply both roles on the same host, or two roles
would write `/etc/containerd/config.toml`.

## Requirements

- Rocky Linux 9/10 target host (RHEL/Alma/CentOS are handled through
  the matching Docker CE repository).
- Outbound HTTPS to download.docker.com (repository metadata and
  packages).
- When `docker_containerd_root` is set: a data volume for that path,
  mounted with the `common` role `disks` mechanism (e.g.
  `inventories/<site>/group_vars/<group>/vars_disks.yml`).

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `docker_containerd_root` | `""` | containerd data root (Docker 28 image store), on the data disk. Empty skips `/etc/containerd/config.toml` entirely — containerd keeps its packaged root on the system volume. |

## Example Playbook

```yaml
- hosts: netbox
  become: true
  roles:
    - role: docker
    - role: netbox
```

## Notes

- Docker 28 uses the containerd image store: image layers live in
  containerd's data root, not in Docker's `data-root`. Set
  `docker_containerd_root` to a path on the data disk when multi-GB
  images do not fit the system volume.
- The compose plugin ships `docker compose` v2; buildx is not
  installed (compose v2 builds with buildkit).
- Handler order matters: `Restart containerd` runs before
  `Restart docker` (docker depends on containerd).
- Tag caveat: `--tags role:<app>` alone will not bootstrap Docker on a
  fresh host — run the play untagged first (same tradeoff as the
  `certificate` role in front of `haproxy`/`postgresql`).
