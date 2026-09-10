# k8s Ansible role

Deploys a high-availability Kubernetes cluster using `kubeadm` with 3 stacked control-plane masters and any number of worker nodes.

## Features

- Idempotent: re-running the playbook only applies missing changes.
- Scalable: add a worker by adding it to your workers inventory group and re-run.
- Configurable CNI: `cilium` (default) or `calico` via `k8s_cni`.
- Configurable CSI: `longhorn` (default) or `none` via `k8s_csi`.
- Optional MetalLB for bare-metal `LoadBalancer` services.
- Uses existing `haproxy` role for API-server high availability.

## Variables

See `defaults/main.yml` for all defaults.

Key tunables:

| Variable | Default | Description |
|---|---|---|
| `k8s_kubernetes_version` | `1.36.4` | Kubernetes version to install |
| `k8s_cni` | `cilium` | CNI plugin: `cilium` or `calico` |
| `k8s_csi` | `longhorn` | CSI plugin: `longhorn` or `none` |
| `k8s_longhorn_version` | `1.8.1` | Longhorn Helm chart version |
| `k8s_longhorn_ui_service_type` | `NodePort` | Longhorn UI (`longhorn-frontend`) service type |
| `k8s_longhorn_ui_nodeport` | `30800` | Pinned NodePort for the Longhorn UI |
| `k8s_cilium_hubble_ui_enabled` | `true` | Deploy the Hubble UI (requires `k8s_cni: cilium`) |
| `k8s_cilium_hubble_ui_service_type` | `NodePort` | Hubble UI service type |
| `k8s_cilium_hubble_ui_nodeport` | `31235` | Pinned NodePort for the Hubble UI |
| `k8s_metallb_enabled` | `true` | Deploy MetalLB |
| `k8s_metallb_version` | `0.14.8` | MetalLB version (upstream manifest tag) |
| `k8s_metallb_manifest_checksum` | sha256 of the pinned manifest | `sha256:<hex>` of `metallb-native.yaml` for `k8s_metallb_version`; update together with the version. Enables idempotent check mode and download verification |
| `k8s_kubestatemetrics_enabled` | `true` | Deploy kube-state-metrics |
| `k8s_kubestatemetrics_chart_version` | `8.4.2` | kube-state-metrics Helm chart version |
| `k8s_kubestatemetrics_namespace` | `kube-system` | Namespace for the kube-state-metrics release |
| `k8s_kubestatemetrics_nodeport` | `30801` | Pinned NodePort for kube-state-metrics |
| `k8s_metricsserver_enabled` | `true` | Deploy metrics-server |
| `k8s_metricsserver_chart_version` | `3.14.0` | metrics-server Helm chart version |
| `k8s_metricsserver_namespace` | `kube-system` | Namespace for the metrics-server release |
| `k8s_monitoring_rbac_enabled` | `true` | Deploy the Prometheus monitoring RBAC + token |
| `k8s_monitoring_namespace` | `monitoring` | Namespace for the monitoring ServiceAccount |
| `k8s_monitoring_serviceaccount` | `prometheus` | ServiceAccount used by the Prometheus scrapers |
| `k8s_headlamp_enabled` | `true` | Deploy Headlamp (Kubernetes web UI) |
| `k8s_headlamp_chart_version` | `0.45.0` | Headlamp Helm chart version |
| `k8s_headlamp_namespace` | `headlamp` | Namespace for the Headlamp release |
| `k8s_headlamp_serviceaccount` | `headlamp-admin` | Admin ServiceAccount created by the chart |
| `k8s_headlamp_nodeport` | `31236` | Pinned NodePort for Headlamp |
| `k8s_controller_install_k9s` | `true` | Install the k9s terminal UI on controllers |
| `k8s_k9s_version` | `0.51.0` | k9s release version |
| `k8s_forgejo_enabled` | `false` | Deploy Forgejo and its Actions runner |
| `k8s_forgejo_chart_version` | `16.0.1` | forgejo-helm chart version |
| `k8s_forgejo_namespace` | `forgejo` | Namespace for the Forgejo release |
| `k8s_forgejo_hostname` | `git.example.com` | Public hostname (web and SSH) served via your reverse proxy |
| `k8s_forgejo_ssh_hostname` | `{{ k8s_forgejo_hostname }}` | Hostname advertised in SSH clone URLs |
| `k8s_forgejo_ssh_port` | `22` | Port advertised in SSH clone URLs (e.g. reverse proxy TCP frontend port) |
| `k8s_forgejo_loadbalancer_ip` | *none* | MetalLB IP shared by the web (80) and SSH (22) services |
| `k8s_forgejo_persistence_size` | `20Gi` | Repository storage size |
| `k8s_forgejo_storage_class` | `""` | StorageClass (cluster default when empty) |
| `k8s_forgejo_db_host` | *none* | External PostgreSQL host (TLS `verify-full`) |
| `k8s_forgejo_db_port` | `5432` | External PostgreSQL port |
| `k8s_forgejo_db_name` / `k8s_forgejo_db_user` | `forgejo` | Database name and user |
| `k8s_forgejo_db_password` | *none* | Database password (vault) |
| `k8s_forgejo_db_ca_files` | `[]` | CA cert paths (on the Ansible controller): step-ca root + intermediate, trusted by all Forgejo and runner containers |
| `k8s_forgejo_admin_username` | `forgejo-admin` | Initial administrator username (must not be a Forgejo reserved name, e.g. `admin`) |
| `k8s_forgejo_admin_password` | *none* | Initial administrator password (vault, first creation only) |
| `k8s_forgejo_runner_enabled` | `true` | Deploy the Forgejo Actions runner |
| `k8s_forgejo_runner_image` | `data.forgejo.org/forgejo/runner:13` | Runner image |
| `k8s_forgejo_runner_dind_image` | `docker:28-dind` | Docker-in-Docker sidecar image |
| `k8s_forgejo_runner_scope` | `""` | Registration scope; empty = global (instance-wide) runner |
| `k8s_forgejo_runner_secret` | *none* | 40-char hex registration secret (vault) |
| `k8s_forgejo_runner_capacity` | `1` | Concurrent jobs per runner |
| `k8s_forgejo_runner_labels` | docker + ubuntu-latest | Labels advertised by the runner |
| `k8s_forgejo_runner_data_size` | `5Gi` | Runner state volume size |
| `k8s_forgejo_runner_docker_size` | `20Gi` | Docker-in-Docker image store size |
| `k8s_forgejo_backup_enabled` | `false` | Encrypted Forgejo dump backups to S3 (in-cluster CronJob) |
| `k8s_forgejo_backup_schedule` | `0 3 * * *` | Cron expression for the backup CronJob (UTC) |
| `k8s_forgejo_backup_retention` | `14` | Encrypted dumps to keep in the bucket |
| `k8s_forgejo_backup_s3_bucket` | *none* | S3 bucket for backups |
| `k8s_forgejo_backup_s3_endpoint` | *none* | S3 endpoint host (HTTPS is assumed) |
| `k8s_forgejo_backup_s3_region` | *none* | S3 region |
| `k8s_forgejo_backup_s3_prefix` | `forgejo` | Prefix (folder) inside the bucket; object keys sort chronologically |
| `k8s_forgejo_backup_s3_key` / `k8s_forgejo_backup_s3_key_secret` | *none* | S3 access key (vault) |
| `k8s_forgejo_backup_cipher_pass` | *none* | Client-side encryption passphrase (vault) |
| `k8s_forgejo_backup_pbkdf2_iterations` | `600000` | PBKDF2 iteration count for the encryption step |
| `k8s_forgejo_backup_pushgateway_urls` | `[]` | Pushgateway URLs receiving backup job metrics from the backup script; empty disables metric pushes |
| `k8s_forgejo_backup_image` | derived | Toolchain image ref: `<registry>/<owner>/forgejo-backup:<tag>` |
| `k8s_forgejo_backup_image_tag` | `<k8s version>-1` | Toolchain image tag; bump to force a rebuild |
| `k8s_forgejo_backup_kaniko_image` | `gcr.io/kaniko-project/executor:v1.23.2` | Image used by the in-cluster build Job |
| `k8s_forgejo_backup_registry_host` / `_owner` / `_user` / `_password` | derived | Forgejo container registry for the toolchain image (defaults to the admin account, vault) |
| `k8s_pod_subnet` | `10.244.0.0/16` | Pod network CIDR |
| `k8s_service_subnet` | `10.96.0.0/12` | Service network CIDR |
| `k8s_api_endpoint` | `k8s-api.example.com` | HA API endpoint |
| `k8s_controller_install_kubeadm` | `false` | Also install `kubeadm` on controller VMs |
| `k8s_controller_awscli_enabled` | `{{ k8s_forgejo_backup_enabled }}` | Install `aws-cli` on controller VMs (backup restore tooling) |
| `k8s_local_kubeconfig_dir` | (required) | Controller directory for kubeconfigs and monitoring credentials. |
| `k8s_bootstrap_host` | *none* | Hostname of the primary master that bootstraps the cluster |
| `k8s_addons_host` | *none* | Hostname of the controller used to deploy CNI, MetalLB, CSI |

## Inventory

The role expects three inventory groups (names are up to you):

- Masters group — hosts that run the Kubernetes control plane.
- Workers group — hosts that run workloads.
- Controllers group — admin VMs with `kubectl` and `helm`.

You must define `k8s_bootstrap_host` (one master) and `k8s_addons_host` (one controller) in your inventory group_vars. Example:

```yaml
k8s_bootstrap_host: "{{ groups['your_k8s_masters'] | first }}"
k8s_addons_host: "{{ groups['your_k8s_controllers'] | first }}"
```

## Controller VMs

Hosts in the controllers group receive:

- `kubectl` (aligned with `k8s_kubernetes_version`).
- `helm` (aligned with `k8s_helm_version`).
- Optional `kubeadm` if `k8s_controller_install_kubeadm: true`.
- Optional `aws-cli` (appstream `awscli2`) if `k8s_controller_awscli_enabled` — the backup restore runbook uses it to fetch encrypted dumps from S3.
- The cluster admin kubeconfig, with the server rewritten to the HA API endpoint (`k8s_api_endpoint`).

Controllers skip all cluster-node setup: no `containerd`, no `kubeadm init/join`, no `kubelet`, no CNI, no MetalLB, and no Kubernetes node firewall ports.

Controllers are also used as the **addon deployer**: after the controller has a working kubeconfig, cluster addons (CNI, MetalLB, CSI) are deployed from it.

## MetalLB

MetalLB (L2 mode) is installed from the upstream `metallb-native.yaml`
manifest, pinned by `k8s_metallb_version`. The manifest is applied only when
the deployed controller image differs from that version — installing,
healing a missing deployment, or upgrading — because re-applying on every
run would revert the webhook `caBundle` that MetalLB's controller rotates in
the live CRDs at runtime. The controller Deployment gets a control-plane
toleration patch so it can also schedule on control-plane nodes; the speaker
DaemonSet needs none (the upstream manifest already tolerates the standard
control-plane taint — revisit only if custom node taints are introduced).
The `IPAddressPool`/`L2Advertisement` config is rendered from
`metallb-config.yaml.j2` (pool range plus the node interface holding
`ansible_host`) and is idempotent on re-runs.

## Persistent Storage (CSI)

The role supports a swappable CSI layer, installed and managed from the controller VM.

| `k8s_csi` | Behavior |
|---|---|
| `longhorn` | Install `iscsi-initiator-utils` and `nfs-utils` on nodes; deploy Longhorn via Helm from the controller. |
| `none` | Skip all CSI tasks. No `StorageClass` is installed. |

### Longhorn requirements

- Each worker must have a dedicated data disk mounted at `/var/lib/longhorn`.
- Nodes need `iscsi-initiator-utils` and `nfs-utils` (installed by the role).
- The firewall must allow Longhorn node-to-node ports.
- Longhorn is installed with `persistence.defaultClass=true`, so it becomes the cluster default `StorageClass`.

### Adding a new CSI

To add another CSI, create `roles/k8s/tasks/csi_<name>.yml`, add its branch in `roles/k8s/tasks/csi.yml`, and add any node prerequisites in `roles/k8s/tasks/csi_prerequisites.yml`.

## GUI exposure (NodePort)

The Longhorn UI and the Hubble UI are exposed via NodePort services with
pinned ports, so an external load balancer (e.g. the `haproxy` role) can route
to them deterministically:

- Longhorn UI: `longhorn-frontend` in `longhorn-system` on `k8s_longhorn_ui_nodeport`.
- Hubble UI: `hubble-ui` in `kube-system` on `k8s_cilium_hubble_ui_nodeport` (Cilium only).
- Headlamp: `headlamp` in `k8s_headlamp_namespace` on `k8s_headlamp_nodeport`.

NodePorts must be unique cluster-wide and inside the API server node-port
range (30000-32767 by default); Helm fails the upgrade if the port is already
taken. The Hubble UI has no built-in authentication: keep it behind a
TLS-terminating proxy with access control.

## Monitoring addons

- **kube-state-metrics** (`k8s_kubestatemetrics_enabled`): cluster state
  metrics (nodes, pods, deployments, PVCs) for Prometheus. Deployed from the
  prometheus-community Helm chart as a single replica (every instance exposes
  the full cluster state) behind a pinned NodePort. Scrape it through a
  single logical target — e.g. the API server service proxy
  (`/api/v1/namespaces/<ns>/services/http:<svc>:<port>/proxy/metrics`,
  RBAC `services/proxy` included) — not per-node NodePorts, or every series
  is duplicated.
- **metrics-server** (`k8s_metricsserver_enabled`): the in-cluster Metrics API
  for `kubectl top` and HPA. Configured with `--kubelet-insecure-tls` because
  kubeadm kubelets serve self-signed certificates. No external exposure.
- **Monitoring RBAC** (`k8s_monitoring_rbac_enabled`): a read-only
  ServiceAccount + ClusterRole for the site Prometheus (service discovery and
  kubelet/cAdvisor/API-server metrics) plus a long-lived token Secret. The
  token and the cluster CA are fetched to `files/k8s/monitoring/` on the
  control node; the `prometheus` role deploys them to its hosts. The token
  grants cluster-wide metrics read only, but treat the fetched files as
  credentials.
- **Headlamp** (`k8s_headlamp_enabled`): Kubernetes web UI from the
  kubernetes-sigs Helm chart. The chart creates the ServiceAccount and a
  cluster-admin ClusterRoleBinding. Log in by pasting a token created with
  `kubectl -n {{ k8s_headlamp_namespace }} create token {{ k8s_headlamp_serviceaccount }}`
  — that admin token is never stored in the repository.

## Forgejo (optional)

When `k8s_forgejo_enabled: true`, the role deploys
[Forgejo](https://forgejo.org/) from the `forgejo-helm` OCI chart plus a
Forgejo Actions runner. The deployment expects:

- **MetalLB** with a free IP in the pool for `k8s_forgejo_loadbalancer_ip`.
  The IP is shared between the `forgejo-http` (port 80) and `forgejo-ssh`
  (port 22) LoadBalancer services via `metallb.universe.tf/allow-shared-ip`
  (MetalLB v0.14 API; MetalLB >= 0.15 renames it to `metallb.io/share-ipv4`).
- **An external PostgreSQL database** (`k8s_forgejo_db_host`), reached with
  `SSL_MODE: verify-full`. The database and user must exist (see the
  `postgresql_databases`/`postgresql_users` variables of the `postgresql`
  role); the password is injected from the `forgejo-db-secret` Secret and
  never written to Helm values.
- **DNS and reverse proxy**: the hostname (e.g. `git.example.com`) must
  resolve to your reverse proxy, which forwards web traffic to the MetalLB IP
  on port 80 and git SSH to the MetalLB IP on port 22 (TCP frontend listening
  on `k8s_forgejo_ssh_port`, e.g. 2222). Ports 80 and 22 must be open on the
  k8s nodes.
- **A CA bundle**: `k8s_forgejo_db_ca_files` (paths on the Ansible
  controller, typically the step-ca root and intermediate) are concatenated
  into the `forgejo-ca-bundle` Secret. It is mounted into every Forgejo and
  runner container and scanned as an extra Go trust directory
  (`SSL_CERT_DIR=/forgejo-ca:/etc/ssl/certs`), so the database and instance
  certificates are trusted — Forgejo's pgx driver has no libpq-style
  `~/.postgresql/root.crt` fallback. Public roots come from each container
  image's own `/etc/ssl/certs`.

Secrets come from inventory vault variables (`k8s_forgejo_db_password`,
`k8s_forgejo_admin_password`, `k8s_forgejo_runner_secret`); the tasks assert
they are set and not placeholders before deploying.

### Git over SSH

Both the web UI and git SSH enter through the reverse proxy: HTTPS on the
standard port and SSH as a TCP pass-through on `k8s_forgejo_ssh_port`
(`server.SSH_PORT` only controls the port advertised in clone URLs — the
container still listens on its own port, 2222 for the rootless image).
Clone URLs therefore read
`ssh://git@git.example.com:2222/owner/repo.git`. To keep scp-style URLs
(`git@git.example.com:owner/repo.git`) working, add to `~/.ssh/config`:

```
Host git.example.com
    Port 2222
```

### Actions runner

The runner registers offline: `forgejo-cli actions register` runs inside the
Forgejo pod with the shared 40-character hex secret and is idempotent (the
runner identifier is derived from the secret's first 16 characters). The
runner Deployment pairs the `forgejo-runner` container with a privileged
Docker-in-Docker sidecar (`DOCKER_HOST=tcp://127.0.0.1:2375`, plain HTTP on
the pod loopback — the dind image entrypoint is bypassed with
`command: dockerd ...` because it would otherwise enable TLS and override
`--tls=false`); jobs run as
containers created by that daemon. Job containers receive git env config
(`GIT_CONFIG_KEY_1=http.https://git.example.com/.sslCAInfo`, requires
git >= 2.31) pointing at the CA bundle so `git clone` against the instance
works, while clones of public action hosts use the job image's own CA
bundle. The runner cache is disabled (its server is not reachable from job
containers behind Docker-in-Docker without a stable address).

### Backup (encrypted dump to S3)

When `k8s_forgejo_backup_enabled: true`, an in-cluster CronJob
(`forgejo-backup`, schedule `k8s_forgejo_backup_schedule` in UTC, never
overlapping runs, one-hour deadline) execs `forgejo dump` inside the live
Forgejo pod over kubectl and streams the zip out. The zip contains the git
repositories, LFS objects, attachments, avatars, packages, `app.ini`/custom
and a native database dump (`forgejo-db.sql`) — Forgejo dumps PostgreSQL
in-process, so no `pg_dump` binary is needed in the container. The zip is
staged, together with its encrypted copy, in a memory-backed `emptyDir`
(`/work`) — dumps never land on the nodes' root disks and no plaintext
copy persists at rest. The zip is integrity-checked (`python3 -m zipfile
-t`), encrypted client-side (salted
AES-256-CBC with PBKDF2, the same cipher family as the pgBackRest
repository) and uploaded to the configured S3 bucket; the stored object
size is verified and backups beyond `k8s_forgejo_backup_retention` are
pruned (object keys sort chronologically). Completed-job history is
capped at one successful job (writable layers of completed pods are
pinned on the node that ran them). `forgejo dump` fatals on a
fresh instance until the repository root exists, so the script bootstraps
`/data/git/gitea-repositories` with an idempotent `mkdir -p`.

The S3 bucket must already exist, and the access key needs put/get/list/
delete rights on it (upload, size verification and pruning).

The backup script runs in a purpose-built toolchain image (bash, kubectl,
openssl, python3, aws-cli on Alpine, pinned to GNU coreutils) that a kaniko
Job builds inside the cluster from a Containerfile shipped in a ConfigMap
and pushes to the Forgejo container registry
(`k8s_forgejo_backup_image`). The CronJob pulls it with registry
credentials from the `forgejo-backup-registry` Secret, so the k8s nodes
must trust the CA that signed the registry certificate — apply the
`certificate` role to the node plays in your playbook (as
`playbooks/site.yml` does). The tasks recreate the build Job whenever it
is absent, failed, or its kaniko image/destination no longer match; to
force a rebuild after changing the Containerfile, bump
`k8s_forgejo_backup_image_tag` (or delete the
`forgejo-backup-image-build` Job) and re-run the playbook. Only this
build step pushes to the registry — nightly backups never touch it. The
registry credentials default to the Forgejo admin account, provisioned
from `vault_forgejo_admin_password` at first install
(`initialOnlyRequireReset`): if you ever reset that password in the UI,
keep the vault in sync or rebuilds will fail authentication.

RBAC is least-privilege: the `forgejo-backup` ServiceAccount may get/list
pods, create pods/exec and get deployments in the Forgejo namespace only,
and kubectl authenticates with that ServiceAccount (no kubeconfig, no
cluster-admin). Secrets come from inventory vault variables; the tasks
assert they are set. The S3 credentials are injected as environment
variables and the cipher passphrase as a 0400-mounted file — neither
appears in the CronJob spec or the script.

Missed schedules are not caught up: while the cluster is down no backups
run, and the next schedule slot after recovery fires normally. Watch runs
with `kubectl -n forgejo get jobs`, inspect one with
`kubectl -n forgejo logs job/<job-name>`, and trigger a manual run with:

```bash
kubectl -n forgejo create job forgejo-backup-manual --from=cronjob/forgejo-backup
```

**The passphrase is irreplaceable**: like the pgBackRest repository cipher,
losing `k8s_forgejo_backup_cipher_pass` makes every dump in the bucket
unrecoverable.

#### Restoring a backup

A dump zip contains the whole instance in four parts: `repos/` (the git
repositories, including wikis — they map to `/data/git/gitea-repositories/`),
`data/` (the APP_DATA_PATH tree that lands in `/data/`: attachments, avatars,
LFS objects, package blobs, Actions artifacts/logs, the internal SSH server
host key in `data/ssh`, the OAuth jwt key), `app.ini` (a reference copy —
never restore it, the chart regenerates the config on every deploy) and
`forgejo-db.sql` (a full database dump). Under `data/`, skip `gitea`
(chart-owned), `tmp` and `lost+found` (root-owned, would abort the copy) and
`queues`/`indexers` (stale or regenerable runtime state).

What to restore depends on what was lost:

- **Database intact** (k8s cluster lost, external PostgreSQL untouched — the
  common case): restore the **files only**. Do not import `forgejo-db.sql`:
  the live database is newer than the snapshot and the redeployed Forgejo
  reconnects to it on its own (same database host, name, user and password
  from the same vault variables; the same forgejo version means no
  migrations; the chart only creates the admin account when it is missing).
- **Database also lost**: restore the files as below, then either import
  `forgejo-db.sql` (single transaction, TLS settings per your
  forgejo-db-secret: `psql "host=<db-host> dbname=<db-name> user=<db-user>
  sslmode=verify-full" -1 -f forgejo-db.sql`) or, for full cluster fidelity,
  recover the PostgreSQL cluster via pgBackRest point-in-time recovery
  instead.

The procedure runs entirely on the controller — it has `aws-cli`
(`k8s_controller_awscli_enabled`), `openssl` and `kubectl` with the admin
kubeconfig. S3 credentials are not deployed anywhere: export the vault
values (access key, secret, region) at restore time. If you ever *upload*
to an S3-compatible endpoint that lacks trailing checksum support (some
providers) with aws-cli >= 2.23, also export
`AWS_REQUEST_CHECKSUM_CALCULATION=when_required` — downloads are unaffected:

```bash
# 1. Fetch, decrypt and sanity-check (bucket credentials and cipher
#    passphrase from the vault; the iteration count must match
#    k8s_forgejo_backup_pbkdf2_iterations):
aws s3 cp s3://<bucket>/<prefix>/forgejo-dump-<timestamp>.zip.enc . \
    --endpoint-url https://<endpoint>
openssl enc -d -aes-256-cbc -pbkdf2 -iter 600000 \
    -in forgejo-dump-<timestamp>.zip.enc -out forgejo-dump.zip \
    -pass file:<passphrase-file>
python3 -m zipfile -t forgejo-dump.zip

# 2. Inspect the archive: repos/, data/, app.ini, forgejo-db.sql.
unzip -l forgejo-dump.zip

# 3. Copy the dump into the pod:
POD=$(kubectl -n forgejo get pod -l app.kubernetes.io/name=forgejo \
    -o jsonpath='{.items[0].metadata.name}')
kubectl -n forgejo cp forgejo-dump.zip "$POD":/tmp/forgejo-dump.zip

# 4. Restore the files: repositories, then data/ minus the chart-owned and
#    regenerable directories. The copies are merge-only: files added after
#    the dump (e.g. registry images pushed to the fresh instance) survive.
kubectl -n forgejo exec deployment/forgejo -- sh -c '
  set -eu
  cd /tmp && unzip -q forgejo-dump.zip
  mkdir -p /data/git/gitea-repositories
  cp -a repos/. /data/git/gitea-repositories/
  cd /tmp/data
  for entry in *; do
    [ -e "$entry" ] || continue
    case "$entry" in gitea|tmp|lost+found|queues|indexers) continue ;; esac
    cp -a "$entry" /data/
  done
  cd /tmp && rm -rf repos data app.ini forgejo-db.sql forgejo-dump.zip
'

# 5. Restart and wait for the deployment:
kubectl -n forgejo rollout restart deployment/forgejo
kubectl -n forgejo rollout status deployment/forgejo
```

Notes:

- Do not use the instance while the restore runs.
- Restoring `data/ssh` keeps git SSH clients' `known_hosts` valid.
- Repositories created *after* the dump was taken will have database
  records but no files on disk.
- The Actions runner re-registers on the fresh cluster by itself (same
  registration secret derives the same UUID).
- Step 4's restore set assumes the chart's default `APP_DATA_PATH = /data`
  (visible in the dump's `app.ini`); adjust if your values differ.

## Usage

```bash
ansible-playbook -i inventories/<site> playbooks/site.yml --limit your_k8s_group
```

## Before first run

1. Ensure `k8s_api_endpoint` resolves to the HAProxy frontend.
2. The HAProxy backend for port `6443` must target the masters group.
3. Nodes need outbound internet access for container images and CNI manifests.
4. All nodes must have a unique hostname and a static IP matching the inventory.

## Deployment order

The `k8s` role is designed to run in the following order:

1. **Control plane** — bootstrap the first master, then join the remaining masters.
2. **Controller** — install `kubectl`/`helm` and fetch the admin kubeconfig.
3. **Addons** — deploy CNI, MetalLB (if enabled), CSI, and Forgejo (if enabled) from the controller.
4. **Workers** — join worker nodes. CNI is already in place, so workers become `Ready` immediately.

This ordering ensures the cluster is fully functional before workers join.

## Bootstrap flow

1. The `k8s_bootstrap_host` bootstraps the cluster using its local API server.
   This avoids a chicken-and-egg problem when the HAProxy frontend has no healthy backends yet.
2. `kube-proxy` is temporarily patched on the bootstrap master to use the local API server until CNI is ready.
3. The remaining control-plane nodes join via the HAProxy endpoint (`k8s_api_endpoint`).
4. The controller VM is configured with `kubectl`, `helm`, and the admin kubeconfig.
5. Cluster addons (CNI, MetalLB, CSI) are deployed from the controller.
6. Worker nodes join via the HAProxy endpoint.

## Firewall requirements

The `k8s` role does not manage host firewall rules. Make sure your firewall role/group_vars allow at least:

- `6443/tcp` — Kubernetes API server (HAProxy backend).
- `10250/tcp` — Kubelet.
- `2379-2380/tcp` — etcd client and peer (control-plane nodes).
- `10259/tcp` — kube-scheduler.
- `10257/tcp` — kube-controller-manager.
- `7946/tcp` — MetalLB speaker memberlist (when MetalLB is enabled).
- `7946/udp` — MetalLB speaker memberlist (when MetalLB is enabled).
- `30000-32767/tcp` — NodePort services (Longhorn UI, Hubble UI, and any other NodePort exposure).

For Cilium:
- VXLAN overlay traffic (UDP 8472 by default) must be allowed between nodes.
- Cilium health probes use TCP 4240 between nodes.
- ICMP between nodes is required for Cilium health checks.

For MetalLB L2 mode:
- The role pins L2Advertisement to the network interface carrying the node's primary IP (`ansible_host`).
- Ensure the MetalLB pool range is reachable from clients on the same L2 segment as the nodes.

For Longhorn:
- `9500/tcp` — Longhorn engine monitor.
- `9501/tcp` — Longhorn engine.
- `9502/tcp` — Longhorn replica.
- `9503/tcp` — Longhorn backing image manager.
- `8500-10000/tcp` — Longhorn instance manager.
- `2049/tcp` — NFS, only if using RWX volumes.
- `3260/tcp` — iSCSI target, only if exposing iSCSI.
