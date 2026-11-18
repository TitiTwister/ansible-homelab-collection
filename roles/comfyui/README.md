# comfyui

Installs and runs [ComfyUI](https://github.com/comfyanonymous/ComfyUI)
as a node-based diffusion web UI and API server on an NVIDIA GPU host:
the `yanwk/comfyui-boot` image managed by a systemd unit, on top of
the `nvidia_docker` runtime role (NVIDIA driver, Docker CE with the
NVIDIA Container Toolkit), with model files downloaded declaratively
from HuggingFace.

Built for Volta-class GPUs (Tesla V100, compute capability 7.0,
16 GB, fp16 compute only): the image is pinned to the cu126 line —
CUDA 12.6 is the last CUDA release supporting Volta — and full fp16
diffusion transformers (e.g. FLUX.1 at ~24 GB) do not fit, so plan
for quantized weights (GGUF). On newer GPUs use a newer image tag and
full-precision weights.

## Requirements

- The `tititwister.homelab.nvidia_docker` role (applied automatically
  as a meta dependency).
- Rocky Linux 9 target host with a passed-through NVIDIA GPU.
- A dedicated volume for model files and outputs, mounted with the
  `common` role `disks` mechanism and referenced by
  `comfyui_data_dir`
  (e.g. `inventories/<site>/group_vars/<gpu_group>/vars_disks.yml`).
- Firewall rule for the web UI port (apply via the `firewall` role).
- Outbound HTTPS to Docker Hub, HuggingFace and GitHub (image, model
  files, custom nodes).
- The GPU cannot be shared with other GPU workloads (vLLM grabs
  ~90% of VRAM at startup): manage exclusivity with the `*_state`
  toggles of both roles.

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `comfyui_image` | `yanwk/comfyui-boot:cu126-slim-20260914` | Container image. Pinned to a dated cu126 build: last CUDA line supporting Volta (sm_70). |
| `comfyui_port` | `8188` | Web UI / API port (published by the container). |
| `comfyui_state` | `started` | Desired service state. `stopped` pauses ComfyUI and frees the GPU for other workloads: the unit is stopped and disabled, so a reboot does not re-grab the GPU. Everything else stays managed; pending changes apply on the next start. |
| `comfyui_data_dir` | `/srv/comfyui` | Persistent data directory on the data disk: models, custom nodes, inputs, outputs, user profile, pip caches. |
| `comfyui_hf_token` | `""` | HuggingFace token for gated model files; provide via a vault file. |
| `comfyui_models` | `[]` | Model files to place under `models/`. Entries: `subdir` (unet, clip, vae, loras...), `repo`, `file`, optional `sha256`. |
| `comfyui_custom_nodes` | `[]` | Custom nodes cloned into `custom_nodes/`. Entries: `name`, `repo`, `version` (tag or commit). |
| `comfyui_extra_args` | `[]` | Extra ComfyUI CLI arguments appended verbatim. |
| `comfyui_container_name` | `comfyui` | Fixed container name used by the systemd unit. |

## Example Playbook

```yaml
- hosts: gpu_servers
  become: true
  roles:
    - role: comfyui
```

## Notes

- The image entrypoint copies the ComfyUI tree bundled in the image
  into the persistent storage on first start, then runs it with
  `--listen` and the port from `CLI_ARGS`. ComfyUI's own version is
  therefore pinned by the image, not by a git checkout.
- The `custom_nodes` volume shadows the ComfyUI-Manager copy bundled
  in the image: clone it explicitly (pinned) when wanted. Node Python
  dependencies must ship in the image (the cu126-slim line bundles
  the common ones, including `gguf` for ComfyUI-GGUF); ComfyUI-Manager
  installs land on the persistent `/root/.local` volume.
- `get_url` has no partial resume: a failed multi-GB model download
  restarts from zero (bounded by `retries`). On a flaky link a manual
  `curl -C -` into the models directory is the escape hatch - the
  next run verifies the checksum and leaves the file untouched.
- Model file sha256 pins are the HuggingFace LFS oids: visible on the
  file's web page or via the repo tree API. Gated repositories need
  `comfyui_hf_token` (and an accepted license) for downloads, even
  when the pins are known.
- FLUX.1-dev on a 16 GB V100: use the GGUF quants from
  `city96/FLUX.1-dev-gguf` (Q4_K_S ≈ 6.8 GB) with
  `t5xxl_fp8_e4m3fn.safetensors` (4.9 GB), `clip_l.safetensors` and
  `ae.safetensors` — ~12.3 GB of weights, loaded sequentially with
  ComfyUI's memory management.
- ComfyUI exposes no Prometheus metrics; container stdout reaches
  Loki through journald under `unit="comfyui.service"` like the other
  systemd-managed containers.
