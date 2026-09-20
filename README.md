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
| Immich | Helm (`oci://ghcr.io/immich-app/immich-charts/immich`) | `immich/` | `helm upgrade --install immich oci://ghcr.io/immich-app/immich-charts/immich -f immich/values.yaml -n immich` — **version unconfirmed, see `immich/Chart.yaml`** |
| Immich ingress + middleware | raw manifest | `immich/ingress.yaml` | `kubectl apply -f immich/ingress.yaml` |
| Immich photo library storage (NFS) | raw manifest | `immich/pvc.yaml` | `kubectl apply -f immich/pvc.yaml` |
| Postgres | raw manifest (not Helm) | `postgres/deployment.yaml`, `postgres/service.yaml`, `postgres/pvc.yaml` | `kubectl apply -f postgres/` |
| Monitoring (Prometheus/Grafana/Loki/Promtail) | Helm (local chart, not published) | `monitoring/` | `helm dependency build monitoring && helm upgrade --install metrics monitoring -f monitoring/values.yaml -n monitoring` |
| cert-manager | Helm (`oci://quay.io/jetstack/charts/cert-manager`) | `cert-manager/values.yaml` | `helm upgrade --install cert-manager oci://quay.io/jetstack/charts/cert-manager -f cert-manager/values.yaml -n cert-manager` |
| Traefik | k3s-bundled (installed automatically by k3s itself, not by this repo) | — | — |

Every file above carries a header comment noting where it was verified from and any known
gap that applies to it — read the file, not just this table, before running its command.

## Where this content came from

Built by comparing two independently-audited sources and confirming they matched:

1. `immich-infra/docs/audit-raw/` — a `kubectl`/`helm get` capture of live cluster state.
2. `/home/divyakumarjain/photos/immich/{k3s,metrics,cluster-issuer.yaml}` on `raspberrypi` —
   the actual chart/manifest source files used with `kubectl apply -f` / `helm upgrade`.

Every raw manifest and the monitoring chart were diffed byte-for-byte between these two
sources and found identical. The Immich and cert-manager Helm chart identities (`oci://...`
registry paths) came from `raspberrypi`'s shell history, since neither is recorded in
cluster state or in a committed `Chart.yaml` — see the version caveat in `immich/Chart.yaml`.

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

Nothing in this repo is a real secret — see `secrets/README.md` for the intended
SOPS-based pattern, which isn't wired up yet.
