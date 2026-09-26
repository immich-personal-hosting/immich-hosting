# immich-hosting

Helm charts and raw Kubernetes manifests for the Immich homelab cluster (k3s across two
nodes: `raspberrypi` and `omv`). This is the real, applied configuration — not an audit
capture. For the architecture this configuration implements (namespaces, gaps found,
rationale), see `docs/CURRENT_STATE.md` in the `immich-infra` repo.

This repo is also the branch/PR target for the **Fixer** agent described in
`immich-infra/docs/AGENTS.md` — it drafts a fix here on a `fix/<short-desc>-<issue_id>`
branch and opens a review request, but never applies anything itself. A human reviews and
runs the apply command by hand.

## Components

| Component | Type | Path | Apply command |
|---|---|---|---|
| Immich | Helm (`oci://ghcr.io/immich-app/immich-charts/immich`), Flux-managed via `clusters/raspberrypi/immich/` | `clusters/raspberrypi/immich/` | Normally applied by Flux (`flux get helmrelease immich -n immich`). Break-glass manual fallback if Flux is suspended: `helm upgrade --install immich oci://ghcr.io/immich-app/immich-charts/immich --version 0.13.1 -f clusters/raspberrypi/immich/values.yaml -n immich` (chart `immich-0.13.1`) |
| Immich ingress + middleware | raw manifest, Flux-managed via `clusters/raspberrypi/immich/` | `clusters/raspberrypi/immich/ingress.yaml` | Normally applied by Flux. Break-glass: `kubectl apply -f clusters/raspberrypi/immich/ingress.yaml` |
| Immich photo library storage (NFS) | raw manifest, Flux-managed via `clusters/raspberrypi/immich/` | `clusters/raspberrypi/immich/pvc.yaml` | Normally applied by Flux. Break-glass: `kubectl apply -f clusters/raspberrypi/immich/pvc.yaml` |
| Postgres | raw manifest (not Helm), Flux-managed via `clusters/raspberrypi/postgres/` | `postgres/deployment.yaml`, `postgres/service.yaml`, `postgres/pvc.yaml` | Normally applied by Flux (`flux get kustomization postgres -n flux-system`). Break-glass manual fallback if Flux is suspended: `kubectl apply -f postgres/pvc.yaml -f postgres/service.yaml -f postgres/deployment.yaml` (note: **not** `-f postgres/`, which now also tries to apply `postgres/kustomization.yaml` itself and fails) |
| Monitoring (Prometheus/Grafana/Loki/Promtail) | Helm (local chart, not published) | `monitoring/` | `helm dependency build monitoring && helm upgrade --install metrics monitoring -f monitoring/values.yaml -n monitoring` |
| Traefik | k3s-bundled (installed automatically by k3s itself, not by this repo) | — | — |
| Traefik customization (`HelmChartConfig/traefik`) | k3s Addon mechanism, Flux-adoption **IN PROGRESS** via `clusters/raspberrypi/traefik/` — no dedicated Kustomization (it's a stateless CR, not a stateful workload), so the root Kustomization applies it immediately on merge. **The pre-existing static manifest on the Pi's SD card must be removed BEFORE merging**, not after; see that directory's README.md for the exact steps | `clusters/raspberrypi/traefik/helmchartconfig.yaml` | Currently applied by k3s's own Addon controller from an unversioned file on the Pi (`/var/lib/rancher/k3s/server/manifests/`, inferred, root-only) — not by this repo yet |

**cert-manager was removed 2026-09-23** — it was fully idle (no `ClusterIssuer`/`Issuer`/`Certificate` anywhere, no `Ingress` referenced it). TLS is a deliberate, standing decision, not a not-done-yet gap — see `clusters/raspberrypi/immich/ingress.yaml`'s TLS note. If TLS work resumes later, reinstall the same way Immich/Postgres were brought in: a Flux `HelmRelease`, not a manual `helm upgrade`.

Every file above carries a header comment noting where it was verified from and any known
gap that applies to it — read the file, not just this table, before running its command.

## Replicating this setup elsewhere (site-specific values)

Everything below is specific to this installation. If you rebuild it on other hardware, or copy it to another site, these are the values to change. Nothing else in the repo is machine-specific.

| Value (as here) | Where it is used | How to find / what to change |
|---|---|---|
| Node names `omv`, `raspberrypi` | `postgres/deployment.yaml` (`nodeSelector`), `postgres/pvc.yaml` (PV `nodeAffinity`), and the `server:` of every NFS PV | `kubectl get nodes`. `omv` is the storage node (photos, and Postgres data); `raspberrypi` is the control-plane node. |
| **Postgres data path** `/srv/dev-disk-by-uuid-36f537fb-c606-4ed2-ae22-55ad447d5329/k8s/immich-postgres` | **`postgres/pvc.yaml` -> `spec.local.path`** (the only place it is defined; the runbook's `RAW` variable repeats the prefix) | See "Choosing the Postgres data path" below. |
| NFS server `omv` and export paths `/export/storage/Personal/Immich`, `/export/storage/metrics/grafana`, `/export/storage/metrics/prometheus` | `clusters/raspberrypi/immich/pvc.yaml`, `monitoring/templates/grafana-data-pv.yaml`, `monitoring/templates/prometheus-data-pv.yaml` | On the NFS server: `sudo exportfs -v`. Create each subdirectory first; Grafana runs as uid 472 and Prometheus as uid 65534, so the directory must be writable by them. |
| Hostnames `immich.raspberrypi`, `grafana.raspberrypi` | `clusters/raspberrypi/immich/ingress.yaml`, `monitoring/templates/grafana-ingress.yaml` (also mentioned in the NetworkPolicy comments) | Must resolve on your LAN to a node IP running Traefik. Change the `host:` lines. |
| Flux directory `clusters/raspberrypi` and the Git repository URL | `clusters/raspberrypi/flux-system/gotk-sync.yaml` (`spec.path`, and the GitRepository `spec.url` / `branch`) | Rename the directory to suit the new cluster, and re-run `flux bootstrap` pointing at your own repository and that path. |
| SOPS/age recipient `age10j22...` | `.sops.yaml` and every `*.enc.yaml` in `clusters/*/secrets/` | Generate your own key with `age-keygen`, put its **public** key in `.sops.yaml`, **re-create every secret** encrypted to it (`sops -e`), and give the cluster the private key as the `sops-age` Secret in `flux-system`. The existing `*.enc.yaml` files cannot be re-keyed (`sops updatekeys` needs to decrypt them first) and are unreadable without this installation's private key, so they must be replaced with your own values. |

### Choosing the Postgres data path

`postgres/pvc.yaml` uses a `local` PV, so it needs an absolute path **on the node named in its `nodeAffinity`**. The path here is the raw mount of the SSD array that OpenMediaVault creates (`/srv/dev-disk-by-uuid-<filesystem UUID>`); the UUID is stable if the array is renumbered. If you replicate this on other hardware:

1. Pick a directory on the node's **fast, redundant local disk**, **not the boot/SD card**, and **not under any NFS export** (this installation exports `/export/storage` to the whole LAN with `no_root_squash`, so a database under it would be readable and writable by any LAN host).
2. Create it with the ownership Postgres needs (uid/gid 999) and mode 700:
   `sudo mkdir -p "$D" && sudo chown 999:999 "$D" && sudo chmod 700 "$D"`
3. Run this preflight **on that node** (set `D` to your path); all four lines must look right before you apply anything:

   ```sh
   D=/your/path/immich-postgres
   echo "device : $(findmnt -T "$D" -no SOURCE,FSTYPE)   <- must be your data disk, NOT the root device ($(findmnt -no SOURCE /))"
   echo "free   : $(df -h --output=avail "$D" | tail -1 | tr -d ' ')"
   echo "owner  : $(stat -c '%u:%g %a' "$D")   <- must be 999:999 700"
   for e in $(sudo exportfs -v | awk '/^\//{print $1}'); do case "$D/" in "$e"/*) echo "EXPOSED under NFS export $e";; esac; done   # must print nothing
   ```
4. Put the path in `postgres/pvc.yaml` under `spec.local.path`, and make `nodeAffinity` and `postgres/deployment.yaml`'s `nodeSelector` name the same node.

Safety property to keep: the directory should live on the data disk itself (a subdirectory of that disk's mount). If the disk fails to mount at boot, the directory then does not exist and the pod refuses to start, instead of silently writing the database to the SD card.

See `postgres/MIGRATION-TO-OMV.md` for how the data was moved and how to roll back.

**Rebuilding the control plane** (the Raspberry Pi: it is the only k3s server and its state has no backup) is documented in `docs/DISASTER-RECOVERY.md`: what survives, what must be provided (the SOPS age key, a GitHub token), and the order to bring everything back from Git.

## Where this content came from

Built by comparing two independently-audited sources and confirming they matched:

1. `immich-infra/docs/audit-raw/` — a `kubectl`/`helm get` capture of live cluster state.
2. `/home/divyakumarjain/photos/immich/{k3s,metrics,cluster-issuer.yaml}` on `raspberrypi` —
   the actual chart/manifest source files used with `kubectl apply -f` / `helm upgrade`.

Every raw manifest and the monitoring chart were diffed byte-for-byte between these two
sources and found identical. Immich's Helm chart identity (`oci://...` registry path) came
from `raspberrypi`'s shell history, since it wasn't recorded in cluster state. (Immich's
chart version is now pinned explicitly in `clusters/raspberrypi/immich/helmrelease.yaml`;
an earlier, unused local wrapper chart that also pinned it, `immich/Chart.yaml`, was
deleted when Immich moved under Flux. cert-manager's chart identity was the same story,
until cert-manager itself was removed — see the Components table above.)

The Postgres `Service` (`postgres/service.yaml`) was never saved as a file on the Pi — it
was applied via an inline `kubectl apply -f - <<EOF` heredoc — and was recovered verbatim
from shell history rather than reconstructed from scratch.

## What's deliberately not fixed yet

Every "KNOWN GAP" comment in these files mirrors something already documented in
`immich-infra/docs/CURRENT_STATE.md`. This restructuring only moved and organized the
existing live configuration — it did not silently fix any of them (plaintext DB password,
missing node affinity on `immich-machine-learning`, the broken ACME URL, etc.). Those are
exactly the kind of thing the Fixer agent (or a human) should address as separate, reviewable
changes — see `immich-infra/docs/AGENTS.md`.

## `legacy/docker-swarm/`

The repo's original Docker Swarm setup (predates the k3s migration), kept for
reference/rollback. Not the current deployment — see `legacy/docker-swarm/README.md`.

## Secrets

Real secrets are SOPS/age-encrypted at `clusters/raspberrypi/secrets/*.enc.yaml`, applied
by Flux directly — see `clusters/raspberrypi/secrets/README.md` for how decryption works,
how to edit or add one, and recovery if the cluster's `sops-age` Secret is ever lost.
(`secrets/README.md` at the repo root is a superseded pre-implementation doc, kept only as
a pointer to the above.)
