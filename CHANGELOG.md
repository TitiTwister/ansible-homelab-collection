# Changelog

## v1.5.1 (2026-11-18)

Idempotency fixes for the firewall, netbox and k8s roles.

### Fixed
- `firewall`: custom firewalld service definitions are now deployed once
  per service name — the deploy loop deduplicates the rules on the
  service, keeping the first definition. A service declared in several
  zones (e.g. `http` internally and publicly) used to get its XML file
  rewritten once per entry on every run, flip-flopping the file content
  whenever the descriptions differed and reporting perpetual changes.
- `netbox`: the local image build now runs with `--provenance=false`,
  targets the `netbox` service explicitly and only runs when a build
  input changed (or the image is missing). BuildKit attaches a fresh
  provenance attestation to every build, and `netbox-worker` inherits
  the `build` key from the compose anchor, so even a fully cached
  rebuild produced a new image ID (built once per service, with the tag
  landing on whichever export finished last) and compose replaced the
  whole stack on every run. The stack is force-recreated when the
  bind-mounted configuration changes, since those files are not part of
  the compose config hash (only the `netbox` and `netbox-worker`
  containers are recreated, not the data services).
- `k8s`: the monitoring credentials staging files are reported as `ok`
  instead of `changed` — they are created and removed inside the same
  task block, the authoritative artifacts are fetched to the control node.
- `openvpn`: the server now honors the DNS/route push variables. The
  template no longer hardcodes public resolvers (`8.8.8.8`/`8.8.4.4`) and
  an unconditional full tunnel: it renders `openvpn_push_dns` (new
  default `[]`), `openvpn_push_search_domain` (new default `""`) and
  `openvpn_redirect_gateway` (new default `false`). VPN clients that must
  resolve split-horizon names need `openvpn_push_dns` pointing at the
  internal resolvers; deployments that relied on the old unconditional
  full tunnel need `openvpn_redirect_gateway: true`.

## v1.5.0 (2026-11-18)

NetBox IPAM/DCIM source of truth, a shared Docker CE runtime role,
and a ComfyUI diffusion workbench on GPU hosts.

### Added
- `comfyui`: new role — ComfyUI node-based diffusion web UI and API
  server on NVIDIA GPU hosts: the `yanwk/comfyui-boot` image under a
  systemd unit on top of the `nvidia_docker` runtime, model files
  downloaded declaratively from HuggingFace (sha256-pinned, gated
  files through `comfyui_hf_token`), custom nodes cloned at pinned
  versions, and all persistent state under one data-disk directory.
  `comfyui_state: stopped` pauses the service and frees the GPU for
  another workload (e.g. `vllm`) without dropping management.
- `docker`: new role — Docker CE from the upstream repository with the
  compose v2 plugin, extracted from the container workload roles.
  Relocating the containerd image store (Docker 28) to a data disk is
  opt-in through `docker_containerd_root`: empty leaves containerd on
  its packaged root and skips `/etc/containerd/config.toml` entirely.
  Does not deploy `daemon.json` (on GPU hosts, `nvidia_docker` owns
  it and provides the whole runtime).
- `netbox`: new role — NetBox via the netbox-docker compose stack
  (netbox + rq worker, PostgreSQL 18, Valkey 9.1 ×2 for the task
  queue and the cache), with plugins baked into a locally built image
  (netbox-topology-views, netbox-data-flows, pinned). All persistent
  state lives on bind mounts under one data-disk root so the stack
  lifts with the volume; the initial superuser is created idempotently
  by the container entrypoint; and a host systemd timer runs the daily
  `manage.py housekeeping` that current netbox-docker releases no
  longer ship a container for.

### Changed
- `openwebui`: no longer installs Docker CE itself — apply the new
  `docker` role first (`nvidia_docker` already provides the runtime
  on GPU hosts). `openwebui_containerd_root` is gone; set
  `docker_containerd_root` instead.
- `certificate`: `certificate_renewal_threshold` raised from 10 to 30
  days of validity headroom before re-issuance, and the never-consumed
  `certificate_default_duration` / per-cert `duration` settings removed —
  certbot runs without a lifetime option, so the step-ca
  `defaultTLSCertDuration` governs how long certificates are valid.

### Fixed
- `openvpn`: the CA-chain move/remove pair in `fetch_certs` ran on
  every run instead of only after the bootstrap extraction — once
  `ca.crt` was in place, every subsequent run failed with
  `Source /etc/openvpn/ca-chain.crt not found`, which also silently
  skipped the `wireguard` role on the same host in full site runs.
  Both tasks now live inside the bootstrap block.
- `haproxy`: PEM bundle assembly used `creates:` on the destination
  file, so renewed certificates never reached existing bundles —
  certificates were refreshed on disk while HAProxy kept serving the
  stale PEMs until they expired. Bundles are now reassembled on every
  run and only swapped (and the reload notified) when the content
  changes; PEM files left behind by removed `certificate_certs`
  entries are cleaned up instead of being served indefinitely by the
  `crt` directory bind.
- `postgresql`: the PGDG repository definition points baseurls at
  `rhel-$releasever_major.$releasever_minor`, but PGDG does not publish
  every OS minor release — Rocky 10.0 hosts resolve to `rhel-10.0`, which
  does not exist — so every dnf transaction failed with "All mirrors were
  tried". The role now deploys a managed
  `/etc/yum.repos.d/pgdg-redhat-all.repo` pinned to the evergreen
  `rhel-<major>-$basearch` layout and enabling only `pgdg-common` plus
  the `postgresql_version` major repository (opt out with
  `postgresql_pgdg_manage_repo: false`); the repo RPM install itself
  runs with `disablerepo: pgdg*` so it succeeds even while the broken
  definition is still in place.
- `certificate`: the validity check treated an `openssl -checkend` rc=1
  (certificate inside the renewal window) as a task failure and aborted
  the play instead of renewing, so in-place renewal never worked —
  deleting the certificate files to force re-issuance was the only
  refresh path. rc=1 now flows into the existing renewal path.

## v1.4.0 (2026-11-11)

Site-to-site WireGuard peers and the firewalld transit policy they need.

### Added
- `wireguard`: `wireguard_site_peers` — static routed peers alongside
  the roaming clients. Each entry brings its own public key, an explicit
  tunnel IP in the server /24 and optional `allowed_ips` networks routed
  to it; rendered into the server config with no server-side key
  generation and no client `.conf` on the controller.
- `firewall`: `firewall_zones` and `firewall_policies` — custom zone
  creation with optional interface binding, and firewalld policies for
  forwarding between zone pairs that do not share a zone (ingress/egress
  zones, rich rules and optional per-policy masquerade). Drives the
  WireGuard site-to-site transit on home-eu-ovpn: a `tunnel` zone for
  wg0 plus the two scoped policies (`public-to-tunnel` unmasqueraded so
  VPC-to-home traffic keeps its real source, `tunnel-to-public`
  masqueraded to the staged-NAT model), so transit works in both
  directions instead of being rejected by firewalld's zone-pair
  dispatch.

### Fixed
- `firewall`: `firewall_zones` creation goes through
  `ansible.posix.firewalld` (`state: present`, the module's zone-level
  operation) instead of raw `firewall-cmd --new-zone` — idempotent,
  check-mode aware, one task less.
- `firewall`: the read-only `firewall-cmd` queries behind
  `firewall_policies` now run with `check_mode: false` — command tasks
  are skipped in `--check` runs by default, which left the rc-based
  apply conditions undefined and broke `--check`.

## v1.3.0 (2026-09-19)

AI/LLM serving stack on GPU hosts.

### Added
- `nvidia_docker`: new role — GPU host runtime shared by the GPU
  workload roles: NVIDIA driver from the CUDA repository (DKMS),
  Docker CE with the NVIDIA Container Toolkit, containerd image store
  on the data disk.
- `vllm`: new role — vLLM OpenAI-compatible API server (official
  image, systemd unit), with serving parameters grouped in
  battle-tested GPU+model profiles selected by `vllm_profile`.
- `openwebui`: new role — Open WebUI self-hosted chat interface for
  OpenAI-compatible backends, in front of a vLLM server.
- `dcgm_exporter`: new role — NVIDIA DCGM exporter exposing GPU
  metrics for Prometheus (utilization, memory, temperature, power,
  clocks, ECC/XID errors).

## v1.2.1 (2026-09-15)

PostgreSQL Extensions support.

### Added
- `postgresql`: `postgresql_databases[].extensions` handles extensions 
  to create in target database.Extensions whose files are not shipped in
  `postgresql-contrib` (e.g. pgvector)

## v1.2.0 (2026-09-15)

WireGuard VPN server support.

### Added
- `wireguard`: new role — installs and configures a WireGuard server
  (UDP, `wg-quick@wg0`), generates server and client Curve25519 keypairs
  idempotently on the server, and writes self-contained client `.conf`
  files on the controller. NAT is delegated to firewalld masquerade; no
  PKI, no `PostUp`/`PostDown`. Can run alongside the `openvpn` role on the
  same host when given a distinct port and /24 subnet.

## v1.1.1 (2026-09-13)

Fix a Sunday false positive in the pgBackRest staleness alert.

### Fixed
- `prometheus`: `PgBackRestBackupStale` fired every Sunday: the diff
  staleness threshold (26h) was shorter than the weekend gap between
  successful differential backups (48h — Sunday's slot runs a full
  backup, and the diff success timestamp is only refreshed Mon–Sat).
  The threshold is now 52h (the weekend gap plus a buffer, mirroring
  the full branch's one-day margin over its weekly cadence), the alert
  gained a 30m `for` clause to dampen flapping, and the description
  documents the weekend gap.

## v1.1.0 (2026-09-12)

Flux CD GitOps support.

### Added
- `k8s`: Flux CD bootstrap (`k8s_flux_enabled`). A restricted machine
  user owns a private fleet repository on the git server (created via the
  admin API, idempotent); its SSH key is Flux's identity, and a human
  collaborator keeps write access for web UI edits. `flux bootstrap` is
  the reconciling operation — only the initial installation reports a
  change, and bumping `k8s_flux_version` upgrades Flux on the next run.
- `k8s`: the apps root kustomization (`<sync path>/apps.yaml`, watching
  `apps/`) is seeded once into the fleet repository. Application
  manifests are maintained in git only: pushing `apps/<name>/` is the
  whole app lifecycle — the role never writes app manifests.
- `k8s`: `flux` CLI and `git` installed on controller VMs when Flux is
  enabled (`k8s_controller_install_flux`, `k8s_controller_install_git`).
- `haproxy`: backends accept an optional `healthcheck_host` — the HTTP
  health check is sent with that `Host` header, for applications that
  validate it (e.g. the Homepage dashboard).

### Fixed
- `k8s`: the Forgejo admin account could not use the API: the helm chart
  created it with must-change-password set
  (`passwordMode: initialOnlyRequireReset`), and Forgejo answers 403 to
  every API request while that flag is set — nobody signs in through the
  web UI to complete the reset on an API-only account. The chart value is
  now `keepUpdated`: the vault password is re-applied with the flag
  cleared on every start, and the role verifies the account over the API
  after each deploy.
- `k8s`: the Flux machine user creation failure is no longer a censored
  retry loop — the 401/403 responses are reported with an actionable
  message (vault mismatch, must-change-password state, or validation
  errors for the `k8s_flux_bot_*` fields).

## v1.0.1 (2026-09-11)

Check-mode correctness and MetalLB manifest idempotency.

### Fixed
- `common`: disk discovery (parted, blkid TYPE/UUID) runs in check mode —
  the mount task no longer reports spurious changes on an empty UUID.
- `certificate`: root CA fetch (uri) runs in check mode — fixes check-mode
  crash on missing response content.
- `postgresql`: Patroni health and cluster-state probes run in check mode.
- `step_ca`: fingerprint read runs in check mode; CA cert/key deploys are
  byte-true (`rstrip=False`) — no perpetual change from a trailing newline.
- `k8s`: wait/render tasks guarded in check mode (Forgejo backup build,
  etcd metrics, monitoring token, runner manifest); MetalLB interface
  detection runs in check mode; monitoring RBAC manifest no longer deleted
  with credential staging files so the render converges.

### Added
- `k8s`: `k8s_metallb_manifest_checksum` pins the sha256 of the upstream
  MetalLB manifest — check mode verifies the download instead of always
  predicting one, and the manifest is integrity-checked before apply.

## v1.0.0 (2026-09-10)

Initial release.
