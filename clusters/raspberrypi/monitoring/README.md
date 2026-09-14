# metrics HelmRelease (draft — not yet reconciled)

Draft manifests to bring the existing, hand-installed `metrics` Helm release
(monitoring namespace) under Flux management. **Everything here is
`suspend: true`** — nothing will actually apply until a human flips that off.

## Contents

- `helmrepositories.yaml` — `HelmRepository` sources for the two upstream
  repos the chart's dependencies come from (`prometheus-community`, `grafana`).
- `helmrelease-metrics.yaml` — the `HelmRelease` itself, sourced from the
  `flux-system` GitRepository at `./monitoring` (this repo's local umbrella
  chart), not from the `HelmRepository` objects above — see the comment block
  at the top of that file for why.
- `kustomization.yaml` — wires the two into this directory's Kustomization.

## Provenance of `spec.values`

Captured from the live cluster via:

```
helm get values metrics -n monitoring -a
```

on 2026-09-14, against release revision 14 (chart `metrics-0.1.0`, subcharts
`prometheus-27.52.0` / `grafana-10.4.0` / `loki-6.22.0` / `promtail-6.16.6` —
confirmed via `helm list -A` and the `helm.sh/chart` labels on the live
manifest, not assumed). This is the **full computed values tree** (chart
defaults + overrides), not just `monitoring/values.yaml`'s ~100 lines of
overrides — diffed against that checked-in file and confirmed there is no
drift between what's deployed and what's committed.

Two fields were flagged/redacted rather than copied verbatim — search for
`TODO(secret)`:

1. `grafana.admin.existingSecret` — empty in the live values; Grafana is
   currently auto-generating and self-managing its own admin password in the
   `metrics-grafana` Secret. There's no plaintext password in the computed
   values to redact, but flagged as something to deliberately manage via a
   Secret before/after this comes under Flux.
2. `loki.minio.rootPassword` — a **chart default placeholder** value
   (`supersecret`), not a real live secret. `loki.minio.enabled` is `false`,
   so it's inert either way, but redacted per instructions since it reads as
   a secret.

## Things that didn't map cleanly (see PR description for full list)

- The chart itself (`metrics`) isn't published anywhere — it's a local
  umbrella chart in this repo, so `chart.spec.version: 0.1.0` isn't a field
  Flux actually consults for a `GitRepository` source; the version comes from
  `monitoring/Chart.yaml` at whatever git revision is checked out.
- The `HelmRepository` objects declared here aren't referenced by this
  `HelmRelease`'s `sourceRef` — they're declared per the request, but
  source-controller resolves the chart's subchart dependencies directly from
  the URLs in `Chart.yaml`/`Chart.lock` when building from git.
- This repo has no pre-existing `clusters/` tree, and its `origin` remote
  (`divyakumarjain/immich-hosting`) differs from the repo Flux's
  `flux-system` GitRepository actually watches
  (`immich-personal-hosting/immich-hosting`, `main`, path
  `./clusters/raspberrypi`). These manifests are laid out at the requested
  path assuming it lands in the repo Flux tracks — confirm before unsuspending.
