# Changelog

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
