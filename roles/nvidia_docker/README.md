# nvidia_docker

GPU host runtime shared by the GPU workload roles (`vllm`,
`comfyui`): the NVIDIA driver from the CUDA repository, Docker CE
with the NVIDIA Container Toolkit (nvidia runtime), and the
containerd image store relocated to the data disk.

Built for Volta-class GPUs (Tesla V100, compute capability 7.0):
the driver is pinned to the 580 legacy branch, the last one
supporting them, and the proprietary DKMS kmod is used (the open
variant does not support Volta). On newer GPUs raise
`nvidia_docker_driver_version` accordingly.

## Requirements

- Rocky/Alma/RHEL 9 or 10 target host with a passed-through NVIDIA GPU.
- A data disk mounted with the `common` role `disks` mechanism and
  referenced by `nvidia_docker_containerd_root`
  (e.g. `inventories/<site>/group_vars/<gpu_group>/vars_disks.yml`).
- Outbound HTTPS to NVIDIA, Docker and EPEL repositories.

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `nvidia_docker_driver_version` | `580.178.04` | NVIDIA driver release, pinned to the 580 legacy branch (last one supporting Volta — see notes). |
| `nvidia_docker_driver_allow_reboot` | `true` | Reboot the host once when a freshly installed driver needs a boot to load. |
| `nvidia_docker_containerd_root` | `/srv/containerd` | containerd data root (Docker 28 image store), on the data disk. |

## Example Playbook

```yaml
- hosts: gpu_servers
  become: true
  roles:
    - role: nvidia_docker
```

Usually consumed as a meta dependency of the GPU workload roles
(`vllm`, `comfyui`) rather than applied directly.

## Notes

- The driver comes from NVIDIA's CUDA repository (EPEL 10 no longer
  ships the akmod-nvidia packages used on EL9). The kmod is the
  proprietary DKMS variant — the open-modules kmod, which the unpinned
  `nvidia-driver-cuda` meta tracks, does not support Volta (V100).
  `nvidia_docker_driver_version` is pinned to the **580 legacy
  branch**, the last one supporting Volta: newer branches install but
  ignore the GPU at runtime with an NVRM "supported through the NVIDIA
  580.xx Legacy drivers" message, even though their README still lists
  the V100. Do not bump it on Volta hardware.
- DKMS builds the module against the running kernel at install time
  (`kernel-devel` is resolved from a live `uname -r`, not from possibly
  stale cached facts) and rebuilds automatically on kernel updates.
  A fresh install blacklists nouveau, rebuilds the initramfs and
  triggers exactly one reboot, gated by
  `nvidia_docker_driver_allow_reboot`.
- GPU persistence mode is re-applied at every run (`nvidia-smi -pm 1`);
  it does not survive reboots on its own. It keeps the GPU initialized
  between GPU workload restarts without blocking other CUDA processes.
- Services consuming the nvidia runtime are not restarted by this
  role's handlers: they run `docker run` in the foreground under
  systemd units with `Restart=always` and `Requires=docker.service`,
  so systemd brings them back after a docker restart.
- Do NOT set `log-driver=journald` in daemon.json: GPU service units
  run `docker run` in the foreground, so container stdout is relayed
  to journald under the unit name (and from there in Loki). A journald
  log driver would duplicate every line.
