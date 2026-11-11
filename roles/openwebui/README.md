# openwebui

Installs and runs [Open WebUI](https://docs.openwebui.com/), a
self-hosted chat interface for OpenAI-compatible backends, as the
official `ghcr.io/open-webui/open-webui` image managed by a systemd
unit. Built to sit in front of a vLLM server (see the `vllm` role)
but works with any OpenAI-compatible endpoint.

## Requirements

- Rocky Linux 9/10 target host with Docker-ready repositories.
- A data volume for the SQLite database (accounts, chat history)
  mounted with the `common` role `disks` mechanism and referenced by
  `openwebui_data_dir` (e.g.
  `inventories/<site>/group_vars/<group>/vars_disks.yml`).
- An OpenAI-compatible backend reachable from the host
  (`openwebui_openai_base_url`).
- Firewall rule for the UI port if it is exposed through a reverse
  proxy (apply via the `firewall` role).
- Outbound HTTPS to ghcr.io (image download on first run).

## Role Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `openwebui_image` | `ghcr.io/open-webui/open-webui:v0.11.3` | Container image, pinned release. |
| `openwebui_port` | `8080` | Host port publishing the container's 8080. |
| `openwebui_openai_base_url` | `http://<hostname>:8000/v1` | OpenAI-compatible backend endpoint, as reachable from inside the container (not the host loopback). |
| `openwebui_openai_api_key` | `sk-openwebui` | API key sent to the backend; vLLM ignores it but Open WebUI requires one. |
| `openwebui_data_dir` | `/srv/openwebui/data` | Persistent data (SQLite DB, uploads), on the data disk. |
| `openwebui_config_dir` | `/etc/openwebui` | Directory holding the container environment file. |
| `openwebui_containerd_root` | `/srv/containerd` | containerd data root (Docker 28 image store), on the data disk. |
| `openwebui_enable_signup` | `true` | Allow account self-registration. The first account created becomes administrator. |
| `openwebui_env` | `{}` | Extra container environment entries, appended verbatim. |
| `openwebui_container_name` | `openwebui` | Fixed container name used by the systemd unit. |

## Example Playbook

```yaml
- hosts: gpu_servers
  become: true
  roles:
    - role: openwebui
```

## Notes

- `OPENAI_API_BASE_URL` only seeds Open WebUI's connection config on
  first boot; the value is then persisted in the SQLite database and
  later changes to the environment file do **not** update it. Fix the
  backend URL in the admin panel (Settings → Connections), or start
  from a fresh `openwebui_data_dir`.
- The first registered account becomes the administrator. Disable
  self-registration afterwards, either via `openwebui_enable_signup:
  false` or the admin panel.
- `WEBUI_SECRET_KEY` is generated on first start and stored inside
  the data directory; it survives restarts and upgrades.
- This role installs Docker CE but deliberately does **not** deploy
  `daemon.json`: on GPU hosts the `nvidia_docker` role owns it (nvidia
  runtime). Both roles template `/etc/containerd/config.toml` — keep
  `openwebui_containerd_root` and `nvidia_docker_containerd_root`
  identical on shared hosts.
- The systemd unit runs `docker run` in the foreground with
  `--pull=never`: image updates go through the role (pull task), not
  through implicit downloads at service start.
- Upgrades run SQLite migrations on start; back up
  `openwebui_data_dir` before bumping `openwebui_image`.
