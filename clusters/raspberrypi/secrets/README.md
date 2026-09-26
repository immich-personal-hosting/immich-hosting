# clusters/raspberrypi/secrets/

SOPS/age-encrypted Kubernetes Secret manifests that Flux applies directly (via the
`flux-system` Kustomization's `spec.decryption` block in `../flux-system/gotk-sync.yaml`).

- `grafana-admin.enc.yaml` — the `metrics-grafana` Secret (admin-user/admin-password) used
  by the `metrics` Helm release's Grafana subchart. Only `stringData` is encrypted; `kind`,
  `metadata`, etc. stay plaintext so diffs are readable.
- `postgres.enc.yaml` — the `immich-postgres-credentials` Secret (`postgres-password`,
  namespace `immich`), referenced by `clusters/raspberrypi/postgres/deployment.yaml`'s `POSTGRES_PASSWORD` and
  `clusters/raspberrypi/immich/values.yaml`'s `DB_PASSWORD`, both via `secretKeyRef` — no
  plaintext password in either. Same encryption shape as above.

## How decryption works here

- The age **private** key lives only on the machine(s) that manage this cluster
  (`~/.config/sops/age/keys.txt`) and in a Kubernetes Secret named `sops-age` in the
  `flux-system` namespace — never in git.
- The age **public** key (safe to share) is in `.sops.yaml` at the repo root.
- kustomize-controller decrypts files matching `.sops.yaml`'s `path_regex` in-memory at
  apply time; the encrypted form is what's committed.

## Editing an existing encrypted file

```
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
sops clusters/raspberrypi/secrets/grafana-admin.enc.yaml   # opens decrypted in $EDITOR, re-encrypts on save
```

## Adding a new one

Write a plain `apiVersion: v1 / kind: Secret` manifest at this path ending in `.enc.yaml`,
then:

```
export SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt
sops --encrypt --in-place clusters/raspberrypi/secrets/<name>.enc.yaml
```

...and add it to `kustomization.yaml`'s `resources:` list.

## If the `sops-age` cluster Secret is ever lost

Flux will fail to decrypt (Kustomization goes `False`/`DecryptionError`) until it's
recreated from the same age private key:

```
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=$HOME/.config/sops/age/keys.txt
```

This is why the private key file also needs a backup outside this one machine — it
currently only exists locally at `~/.config/sops/age/keys.txt`; back that up somewhere
durable (password manager, offline copy) so a disk failure on this machine doesn't also
take out the ability to decrypt these secrets.
