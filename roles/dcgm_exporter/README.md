# dcgm_exporter

Runs [NVIDIA DCGM exporter](https://github.com/NVIDIA/dcgm-exporter)
as the official `nvcr.io/nvidia/k8s/dcgm-exporter` image managed by a
systemd unit, exposing GPU metrics for Prometheus: utilization,
memory, temperature, power draw, clocks, ECC/XID errors.

Deploy it on GPU hosts alongside the `vllm` role; the scrape job
pattern lives in the play project (target the GPU host group, port
`dcgm_exporter_port`).

## Requirements

- Rocky Linux 9/10 target host with a passed-through NVIDIA datacenter
  GPU (Volta or newer).
- The NVIDIA runtime registered in Docker's `daemon.json` — provided
  by the `nvidia_docker` role on GPU hosts. This role does not manage
  `daemon.json`.
- A data volume for the container image store, referenced by
  `dcgm_exporter_containerd_root`.
- Firewall rule for the metrics port (apply via the `firewall` role).

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `dcgm_exporter_image` | `nvcr.io/nvidia/k8s/dcgm-exporter:4.6.0-4.8.3-distroless` | Container image, pinned release. |
| `dcgm_exporter_port` | `9400` | Host port publishing the exporter's 9400. |
| `dcgm_exporter_containerd_root` | `/srv/containerd` | containerd data root (Docker 28 image store), on the data disk. |
| `dcgm_exporter_container_name` | `dcgm-exporter` | Fixed container name used by the systemd unit. |

## Example Playbook

```yaml
- hosts: gpu_servers
  become: true
  roles:
    - role: dcgm_exporter
```

## Notes

- DCGM supports Volta and newer datacenter GPUs; on pre-Ampere GPUs
  (e.g. V100) profiling metrics come from the proprietary DCGM
  package included in the container.
- The systemd unit runs `docker run` in the foreground with
  `--pull=never` and `--cap-add SYS_ADMIN` (required by the DCGM host
  engine): image updates go through the role (pull task), not through
  implicit downloads at service start.
- The default collectors CSV ships in the image; override it with a
  custom volume + `-f` in the unit if you need extra fields (see the
  upstream README).
