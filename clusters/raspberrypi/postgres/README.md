# Postgres, under Flux

This directory holds the actual Postgres manifests (`pvc.yaml`, `service.yaml`,
`deployment.yaml`) and the migration runbook (`MIGRATION-TO-OMV.md`) — no repo-root split,
picked up directly by the root `flux-system` Kustomization the same way
`namespaces/`/`network-policies/`/`dashboards/` are, and matching Immich's layout under
`clusters/raspberrypi/immich/`.

## History: this used to be split, and staged

Originally (PR #29/#30), this directory held *only* a dedicated, `suspend`-able Flux
`Kustomization` object, while the manifests stayed at the repo root, `postgres/`. That split
existed on purpose: Postgres's `Deployment` uses `strategy: Recreate` against a node-local,
stateful volume, and the *first* Flux takeover of an already-running Deployment (a
server-side-apply field-manager handoff from `kubectl`'s client-side apply) carried a real,
if small, risk of an unwanted restart — worth staging with `suspend: true`, a dry-run diff,
then `suspend: false` as a separate step, rather than trusting the root Kustomization's
always-on, unsuspendable apply.

Unified into this single directory 2026-09-26, once that initial takeover had already run
safely for days with no incident. Doing so meant decommissioning the dedicated
`Kustomization` (`ks-postgres.yaml`) — done as two separate PRs, not one, because deleting a
Flux `Kustomization` with `prune: true` runs its garbage-collection finalizer against
everything it applied. The first PR flipped that Kustomization to `prune: false` (a
no-live-effect change, its own reviewed step); only after that merged did the second PR
delete the Kustomization and move the manifests here, so the GC finalizer had nothing to
clean up — the Deployment/Service just kept running, now tracked by the root Kustomization
instead.

## Adoption note (from the original split-layout era, still accurate)

Adopted 2026-09 from an existing, already-running Deployment (previously applied by hand
via `kubectl apply -f postgres/`). `postgres/deployment.yaml` on `main` already matched
the live Deployment's `spec.template.spec` field-for-field at adoption time — this was
confirmed by reading the live object and the checked-in file side by side, not assumed.
An unrelated, unmerged resource-limit change on another branch (`fix/a864489e-...`,
since discarded) was deliberately *not* included in this adoption, so the initial Flux
takeover was a true no-op relative to what was running.

## Data-loss guard

`pvc.yaml`'s `PersistentVolume` and `PersistentVolumeClaim` both carry
`kustomize.toolkit.fluxcd.io/prune: disabled` — the root `flux-system` Kustomization runs
with `prune: true`, and an accidental removal from Git must not delete the live database
volume. Same reasoning, same annotation, as `clusters/raspberrypi/namespaces/namespaces.yaml`.
Note the `Deployment`/`Service` do **not** carry this annotation (only the volume needs it);
that's exactly why the two-PR sequence above exists — anything without the annotation is
fair game for a Kustomization's GC finalizer.
