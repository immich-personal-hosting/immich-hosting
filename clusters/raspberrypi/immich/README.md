# Immich, under Flux

Everything Flux manages for Immich lives in this one directory: the `HelmRepository` +
`HelmRelease` for the chart, the `values.yaml` that feeds it (via a generated ConfigMap),
and the two raw manifests (`pvc.yaml`, `ingress.yaml`) that were never part of the chart
release. This is a different layout than `monitoring/` (chart source at the repo root,
control objects nested under `clusters/raspberrypi/monitoring/`) — deliberately: unlike
monitoring, Immich has no local umbrella chart and no manual fallback command that needs
the chart at a fixed repo-root path, so there was no reason to split them.

## Why source the chart directly from OCI, not a local wrapper chart

A local wrapper chart (`immich/Chart.yaml`, declaring a dependency on
`oci://ghcr.io/immich-app/immich-charts`) existed in this repo before this change, but was
never actually used — the real manual apply command pulled the upstream OCI chart
directly, and the wrapper had no `Chart.lock` or vendored `charts/`, so it couldn't even
build. It's been deleted. Sourcing directly from OCI:

- Needs no re-nesting of `values.yaml` under a subchart key (a local wrapper chart would
  require that, and a misplaced key there would silently drop a live override — no error,
  only a rendered-output diff would catch it).
- Has no `ChartVersion`-bump trap. `monitoring/`'s local chart only rebuilds when
  `Chart.yaml`'s `version:` is bumped — forgetting that once left a template change
  silently un-applied (see `monitoring/README.md`). Here, `helmrelease.yaml`'s
  `chart.spec.version` is pinned directly; bumping the Immich version is a one-field edit.

## `values.yaml` stays genuinely live

`kustomization.yaml`'s `configMapGenerator` turns `values.yaml` in this same directory
into a stable-named ConfigMap (`disableNameSuffixHash: true`, matching the pattern already
used by `clusters/raspberrypi/dashboards/kustomization.yaml`), which `helmrelease.yaml`
reads via `valuesFrom`. Editing `values.yaml` and merging is the whole workflow — same as
running `helm upgrade -f immich/values.yaml` before this change, just via Flux.

## Adoption note

Adopted 2026-09-23 from the existing, already-running `immich` Helm release (chart
`immich-0.13.1`, revision 16 at adoption time). `values.yaml`'s content was verified
structurally identical to `helm get values immich -n immich` (the live release's actual
user-supplied values) before being moved here — not assumed, checked field-by-field via a
YAML-aware diff, not a raw text diff (Helm re-serializes/alphabetizes on output, which
produces a large but meaningless textual diff against the hand-written file).

## Data-loss guard

`immich-nfs-pv` / `immich-nfs-pvc` in `pvc.yaml` both carry
`kustomize.toolkit.fluxcd.io/prune: disabled` — same reasoning as
`clusters/raspberrypi/postgres/`'s PV/PVC and `clusters/raspberrypi/namespaces/namespaces.yaml`:
an accidental Git-side removal must not delete a live volume, here the photo library.
