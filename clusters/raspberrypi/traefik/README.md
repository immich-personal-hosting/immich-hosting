# Traefik customization, under Flux

Traefik itself stays k3s-bundled (its `HelmChart` object is created and applied by k3s's own
Addon controller, not by this repo — see the top-level README's Components table). This
directory brings only its **customization** — a `HelmChartConfig` object, k3s's mechanism for
overriding a bundled component's chart values — under Flux, since there's no chart source
for Traefik itself in this repo to attach it to. Unlike `postgres/`/`immich/`, there's no
repo-root split here: `helmchartconfig.yaml` is the only file this component needs, so it
lives directly here, picked up by the root `flux-system` Kustomization like `namespaces/`,
`network-policies/`, and `dashboards/` already are.

## Why this needs care that those other directories don't

Postgres and Immich were previously applied by **a human running a command once** — nothing
kept re-enforcing them afterward, so adopting them under Flux was a clean handoff. Traefik's
`HelmChartConfig` is different: it's continuously managed by **k3s's own built-in Addon
controller**, reading a static manifest file from the Pi's local disk
(`/var/lib/rancher/k3s/server/manifests/`, inferred — not directly confirmed, that directory
is root-only and there's no passwordless `sudo` on the Pi). That controller will keep
re-applying its version of this object for as long as that file exists, the same way
`kustomize-controller` keeps re-applying what's in Git.

**Because this file lives directly under `clusters/raspberrypi/` (no dedicated, suspendable
Kustomization — see the repo's top-level notes on why Postgres/Immich's raw manifests don't
get one either once unified this way), merging this PR applies it on Flux's very next
reconcile.** There is no staging window. The static file on the Pi **must be removed before
this PR is merged**, not after:

1. On the Pi (needs root): confirm the file and that its content matches
   `helmchartconfig.yaml`'s `spec`:
   ```
   sudo cat /var/lib/rancher/k3s/server/manifests/traefik-logs-config.yaml
   ```
2. Remove it:
   ```
   sudo rm /var/lib/rancher/k3s/server/manifests/traefik-logs-config.yaml
   ```
   (k3s's Addon controller does not delete the live object when its source file is removed —
   per k3s's own Addon semantics, the object is simply orphaned, left as-is, ready for a
   different manager to take it over.)
3. Confirm no other manager is still asserting it:
   `kubectl get helmchartconfig traefik -n kube-system -o json --show-managed-fields`
4. Only then merge this PR. `kubectl diff --server-side --field-manager=kustomize-controller
   -f clusters/raspberrypi/traefik/helmchartconfig.yaml` was empty against live before this
   was written — a true no-op adoption once the static file is gone.

## Why this is worth doing at all

Until now, this customization existed **only** as an unversioned file on the Pi's SD card —
outside Git, outside any backup, and (per `docs/DISASTER-RECOVERY.md`) exactly the kind of
thing a Pi rebuild would silently lose, since a fresh k3s install starts with no such file
and vanilla Traefik defaults. Once this is genuinely Flux-managed, a rebuild restores it
automatically along with everything else Flux manages.
