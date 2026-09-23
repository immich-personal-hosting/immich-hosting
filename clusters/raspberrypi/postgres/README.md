# Postgres, under Flux

This directory holds only the Flux `Kustomization` object (`ks-postgres.yaml`) that
brings the Postgres manifests under GitOps. The manifests themselves are **not** here —
they stay at the repo root, `postgres/` (`pvc.yaml`, `service.yaml`, `deployment.yaml`),
unmoved, so every existing reference to those paths (`README.md`,
`docs/DISASTER-RECOVERY.md`, `postgres/MIGRATION-TO-OMV.md`) keeps working.

## Why a separate `Kustomization` instead of nesting `postgres/` under `clusters/raspberrypi/`

Every other raw-manifest directory in this repo (`namespaces/`, `secrets/`,
`network-policies/`, `dashboards/`) is nested directly under `clusters/raspberrypi/` and
picked up implicitly by the root `flux-system` Kustomization — which has no `suspend`
mechanism at the subdirectory level. Postgres's `Deployment` uses `strategy: Recreate`
against a node-local, stateful volume (`postgres-omv-pv`): on first Flux takeover, any
field mismatch between what's in Git and what's live — including a server-side-apply
field-manager handoff from `kubectl`'s client-side apply — could trigger an actual pod
restart. A dedicated `Kustomization` object lets that transition be staged: merged with
`suspend: true` (changes nothing live), verified with a dry-run diff, then flipped to
`suspend: false` as a separate, explicit step. See `ks-postgres.yaml`'s own comments for
the exact verification command.

## Adoption note

Adopted 2026-09 from an existing, already-running Deployment (previously applied by hand
via `kubectl apply -f postgres/`). `postgres/deployment.yaml` on `main` already matched
the live Deployment's `spec.template.spec` field-for-field at adoption time — this was
confirmed by reading the live object and the checked-in file side by side, not assumed.
An unrelated, unmerged resource-limit change on another branch (`fix/a864489e-...`) was
deliberately *not* included in this adoption, so the initial Flux takeover is a true
no-op relative to what's running.

## Data-loss guard

`postgres/pvc.yaml`'s `PersistentVolume` and `PersistentVolumeClaim` both carry
`kustomize.toolkit.fluxcd.io/prune: disabled` — the root `flux-system` Kustomization (and
this one) both run with `prune: true`, and an accidental removal from Git must not delete
the live database volume. Same reasoning, same annotation, as
`clusters/raspberrypi/namespaces/namespaces.yaml`.
