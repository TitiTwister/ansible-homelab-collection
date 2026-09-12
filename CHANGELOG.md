# Changelog

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
