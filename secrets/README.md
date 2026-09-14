# secrets/

Nothing is encrypted here yet — this directory documents the intended pattern, it isn't
populated. Building it requires an actual SOPS key, which this session doesn't have and
shouldn't fabricate.

## Intended pattern

- Per-component SOPS-encrypted values files, e.g. `secrets/postgres.enc.yaml`,
  `secrets/grafana.enc.yaml` — same shape as the plaintext `values.yaml` files elsewhere in
  this repo, just the credential-bearing keys, encrypted at rest with `sops`.
- The corresponding component's `values.yaml` references the decrypted Secret by name
  (`existingSecret: ...`) rather than embedding the value inline.
- `sops` decrypts these to a real `kubectl create secret` / a Secret manifest at apply time,
  on the machine doing the apply (a human, per the review workflow in
  `immich-infra/docs/AGENTS.md`) — never committed in plaintext, never decrypted by the
  Fixer or Tier 2 agents.

## Known gaps this is meant to close (see immich-infra/docs/CURRENT_STATE.md)

- `postgres/deployment.yaml`'s `POSTGRES_PASSWORD` / `immich/values.yaml`'s `DB_PASSWORD` are
  both currently the plaintext literal `postgres`, not Secret-backed.
- The Grafana admin Secret (`metrics-grafana` in `monitoring`) is chart-auto-generated with
  no `existingSecret` override, and its base64 value was captured in a (gitignored) audit
  dump — worth rotating regardless of this pattern being wired up.

Setting this up for real (choosing age vs. PGP, where the key lives, wiring `.sops.yaml`) is
a decision for you to make outside this session — flagging the shape, not implementing it.
