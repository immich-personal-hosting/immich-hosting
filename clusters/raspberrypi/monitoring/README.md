# metrics HelmRelease

Brings the existing, hand-installed `metrics` Helm release (monitoring
namespace) under Flux management. Landed as `suspend: true` in PR #1 while
under review, then flipped to `suspend: false` in PR #4 once the SOPS secret
wiring (PR #3) was in place and a `helm template` dry-run against the exact
chart+values confirmed the only diff was the old chart-managed Grafana
Secret dropping out (expected) and one resulting Grafana pod restart.

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

Two fields needed special handling rather than copying verbatim:

1. `grafana.admin.existingSecret` — was empty in the live values (Grafana was
   auto-generating and self-managing its own admin password). Now repointed
   at `metrics-grafana`, the SOPS-managed Secret added in PR #3
   (`clusters/raspberrypi/secrets/grafana-admin.enc.yaml`), so Flux uses that
   instead of the chart's auto-generation-with-lookup behavior.
2. `loki.minio.rootPassword` — a **chart default placeholder** value
   (`supersecret`), not a real live secret. `loki.minio.enabled` is `false`,
   so it's inert either way, but redacted per instructions since it reads as
   a secret — search this file for `TODO(secret)`.

## Things that didn't map cleanly (see PR description for full list)

- The chart itself (`metrics`) isn't published anywhere — it's a local
  umbrella chart in this repo, so `chart.spec.version: 0.1.0` isn't a field
  Flux actually consults for a `GitRepository` source; the version comes from
  `monitoring/Chart.yaml` at whatever git revision is checked out.
- The `HelmRepository` objects declared here aren't referenced by this
  `HelmRelease`'s `sourceRef` — they're declared per the request, but
  source-controller resolves the chart's subchart dependencies directly from
  the URLs in `Chart.yaml`/`Chart.lock` when building from git.
- `metrics.grafana.admin.existingSecret: metrics-grafana` depended on PR #3
  (SOPS/age secret management) being merged, and on its manual follow-up step
  (creating the `sops-age` Secret in `flux-system`) — both done before this
  was unsuspended in PR #4.
