# secrets/ (superseded — see clusters/raspberrypi/secrets/README.md)

This directory is a leftover pre-implementation planning doc. Everything it described as
"intended" has since been built, for real, at a different path:
**`clusters/raspberrypi/secrets/`** — SOPS/age-encrypted Secret manifests, applied directly
by the `flux-system` Kustomization's `decryption` block. That directory's `README.md` is
the current, accurate documentation: how decryption works, how to edit or add an encrypted
file, and how to recover if the `sops-age` cluster Secret is ever lost.

This directory itself holds no files and isn't referenced by any Flux Kustomization or
`.sops.yaml` `path_regex` — it exists only as this pointer, kept so the old
`secrets/README.md` link in the top-level `README.md` still resolves to something correct
rather than a stale, wrong description.

For the record, this doc's original "known gaps" are closed:
- `clusters/raspberrypi/postgres/deployment.yaml`'s `POSTGRES_PASSWORD` and
  `clusters/raspberrypi/immich/values.yaml`'s `DB_PASSWORD` both reference the
  SOPS-managed `immich-postgres-credentials` Secret via `secretKeyRef` — no plaintext
  literal.
- The Grafana admin Secret is SOPS-managed (`clusters/raspberrypi/secrets/grafana-admin.enc.yaml`,
  referenced via `existingSecret` in the `metrics` HelmRelease), not chart-auto-generated.
