# vllm

Installs and runs [vLLM](https://docs.vllm.ai/) as an
OpenAI-compatible API server on an NVIDIA GPU host: the official
`vllm/vllm-openai` image managed by a systemd unit, on top of the
`nvidia_docker` runtime role (NVIDIA driver, Docker CE with the
NVIDIA Container Toolkit).

All serving parameters (image pin, model, memory budget, context
length, serve arguments, container environment) come from a
**profile** — one file per battle-tested GPU+model combo, selected
with `vllm_profile` (see [Profiles](#profiles)).

## Requirements

- The `tititwister.homelab.nvidia_docker` role (applied automatically
  as a meta dependency).
- Rocky Linux 9 target host with a passed-through NVIDIA GPU.
- A dedicated volume for model weights, mounted with the `common` role
  `disks` mechanism and referenced by `vllm_hf_cache_dir`
  (e.g. `inventories/<site>/group_vars/<gpu_group>/vars_disks.yml`).
- Firewall rule for the API port (apply via the `firewall` role).
- Outbound HTTPS to Docker Hub and HuggingFace (image and weight
  downloads on first start).

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `vllm_profile` | `v100-16gb_qwen3-4b-instruct-2507` | Serving profile to load, from the role's `vars/profiles/` directory. See [Profiles](#profiles). |
| `vllm_image` | `vllm/vllm-openai:v0.10.2` | Container image. Fallback only — normally pinned by the profile. |
| `vllm_model` | `Qwen/Qwen3-4B-Instruct-2507` | HuggingFace model to serve. Fallback only — normally set by the profile. |
| `vllm_served_model_name` | basename of `vllm_model` | Name advertised by `/v1/models` and used in API requests. |
| `vllm_port` | `8000` | OpenAI-compatible API port. |
| `vllm_gpu_memory_utilization` | `0.90` | Fraction of GPU memory vLLM may allocate. Fallback only — normally set by the profile. |
| `vllm_max_model_len` | `32768` | Maximum context length in tokens; empty string omits `--max-model-len` (model's native context). Fallback only — normally set by the profile. |
| `vllm_extra_args` | `[]` | Extra `vllm serve` arguments appended verbatim. Fallback only — normally set by the profile. |
| `vllm_env` | `{}` | Container environment (e.g. `VLLM_USE_V1=0` for the V0 engine on Volta). Fallback only — normally set by the profile. |
| `vllm_hf_token` | `""` | HuggingFace token for gated models; provide via a vault file. |
| `vllm_hf_cache_dir` | `/srv/vllm/hf` | HuggingFace cache (model weights), on the data disk. |
| `vllm_config_dir` | `/etc/vllm` | Directory holding the container environment file. |
| `vllm_container_name` | `vllm` | Fixed container name used by the systemd unit. |
| `vllm_state` | `started` | Desired service state. `stopped` pauses vLLM and frees the GPU for other workloads: the unit is stopped and disabled, so a reboot does not re-grab the GPU. The driver, image and configuration stay managed; pending changes apply on the next start. |

## Profiles

A profile is a single YAML file in `vars/profiles/` holding every
combo-specific variable — `vllm_image`, `vllm_model`,
`vllm_gpu_memory_utilization`, `vllm_max_model_len`, `vllm_extra_args`,
`vllm_env` — together with the tuning notes that explain each value
(measured KV pool sizes, engine constraints, flags that OOM, parser
choices). Select it from the inventory:

```yaml
vllm_profile: v100-16gb_qwen3-4b-instruct-2507
```

Shipped profiles:

| Profile | Target |
|---------|--------|
| `v100-16gb_qwen3-4b-instruct-2507` | Tesla V100-PCIE-16GB (Volta, sm 7.0) + Qwen3-4B-Instruct-2507 on vLLM 0.10.2: V0 engine (`VLLM_USE_V1=0`), fp16 weights only, `max_model_len 24576`, hermes tool parser. The role's default selection. |
| `default` | Generic baseline for modern GPUs (compute capability >= 7.5) on the V1 engine. A starting template, not a tuned profile: pin the image before real use. |

**Precedence:** profiles load with `include_vars`, which sits above
`group_vars`/`host_vars` — a variable set in a profile always wins
over the same variable in inventory files. Per-combo tuning therefore
belongs in the profile file (git-tracked like everything else);
ad-hoc overrides go through `-e`. Different hosts may select
different profiles via `host_vars`.

### Adding a profile

1. Copy `vars/profiles/default.yml` to
   `<gpu>_<model>.yml` in the same directory.
2. Pin `vllm_image` to the current stable release for the GPU:
   compute capability 7.0 (Volta) is capped at the 0.10 line with the
   V0 engine (`VLLM_USE_V1: "0"`) and fp16 weights only; >= 7.5 takes
   any modern tag on the V1 engine.
3. Set `vllm_model` and check the weights fit the card at their
   dtype (2 bytes/parameter for fp16/bf16).
4. Tune `vllm_gpu_memory_utilization` and `vllm_max_model_len`
   (empty = model's native context); after each change, check the
   boot-time "Maximum concurrency for N tokens per request" line in
   `journalctl -u vllm` to see the real KV pool.
5. Select the profile with `vllm_profile` in
   `group_vars`/`host_vars` and run the play.

## Example Playbook

```yaml
- hosts: gpu_servers
  become: true
  roles:
    - role: vllm
```

## Notes

- The NVIDIA driver and the Docker/NVIDIA container runtime are
  installed by the `nvidia_docker` meta dependency (Volta driver pin,
  DKMS build, one-time reboot, containerd image store on the data
  disk) — see that role's README.
- GPU persistence mode is re-applied by `nvidia_docker` at every run
  (`nvidia-smi -pm 1`); it does not survive reboots on its own.
- The systemd unit runs `docker run` in the foreground with
  `--pull=never`: image updates go through the role (pull task), not
  through implicit downloads at service start.
- `vllm_state: stopped` pauses the service without tearing anything
  down: weights (`vllm_hf_cache_dir`), the container image and the
  NVIDIA driver stay in place, and the API verification is skipped.
  The service is also disabled, so a reboot keeps the GPU free.
- The first service start downloads the model weights before the API
  answers; the role waits up to 20 minutes for `/v1/models`.
- vLLM exposes Prometheus metrics on the API port at `/metrics`.
- Container logs reach Loki without any log-driver change: the systemd
  unit runs `docker run` in the foreground, so container stdout is
  relayed to journald under `unit="vllm.service"`. Do NOT set
  `log-driver=journald` in daemon.json — it would duplicate every
  line. `docker logs` keeps using the default json-file driver.
