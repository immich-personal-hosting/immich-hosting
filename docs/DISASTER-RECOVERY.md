# Rebuilding the control plane (the Raspberry Pi) after losing it

Scope: the Pi (`raspberrypi`), the only k3s server, is lost or its SD card is dead, and **`omv` survives**. The cluster's API state lives in an embedded **SQLite** database on the Pi's SD card and **has no backup** (checked 2026-09-20); this runbook rebuilds it from Git instead.

**How to read the tags.** **[V]** = verified against the live system on 2026-09-20. **[D]** = follows from k3s/Flux behaviour or documentation but **has not been tested here**: treat it as a plan, and check the result. **[?]** = unknown; the owner has to fill it in (see "Open items").

> **A drill on a scratch machine has never been run.** Every step tagged [D] is untested. If this ever matters, rehearse it once on a spare VM first.

## 0. What survives, and what does not

| Survives on `omv` | Recreated from Git | **Lost / must be provided** |
|---|---|---|
| Photo library, DB dumps (NFS export) | Everything Flux manages: monitoring release, secrets, network policies, dashboards, **Postgres** (`clusters/raspberrypi/postgres/`), **Immich** (`clusters/raspberrypi/immich/`, Helm release + raw manifests) — see `flux get all -A` for the current object count | The **SOPS age private key** (see step 0.1) |
| Grafana and Prometheus data (NFS) | Nothing is hand-applied anymore (cert-manager was removed 2026-09-23, having been fully idle — see step 6) | (nothing else: Flux needs **no GitHub credential**, see step 5) |
| Postgres data (`…/k8s/immich-postgres` on omv) | | The k3s **server token and CA** (new ones are issued) |
| Loki's old data directory (see 6.3) | | Helm release history (irrelevant on a fresh cluster) |

**The database moved to `omv` on 2026-09-20** (`clusters/raspberrypi/postgres/MIGRATION-TO-OMV.md`), so a Pi loss no longer takes it with it. (Until the old copy on the Pi is decommissioned it still exists there, but do not rely on it.) If Postgres is ever on the Pi's SD card again, a Pi loss also loses the database: restore the latest nightly dump (section 8.1), up to about 24 hours of changes.

### 0.1 Have these ready *before* you need them

- [ ] **The SOPS age private key**, backed up somewhere that is **not** the workstation and **not** the cluster (for example a password manager). Today it exists only in `~/.config/sops/age/keys.txt` on the owner's workstation and in the cluster Secret `flux-system/sops-age`. Never paste it into a chat or commit it. [V]
- [ ] **Access to the GitHub account/organisation** (including its 2FA recovery codes). Flux itself needs **no token**: the repository is public and Flux pulls it anonymously (since 2026-09-20). If the repository is ever made private, add a read-only deploy key.
- [ ] `sudo` on both nodes (the Pi needs a password; `omv` is passwordless). [V]

## 1. Preconditions for the rebuilt Pi [V unless noted]

- Debian 13 (Raspberry Pi OS "trixie"), **arm64** (`aarch64`); the original ran kernel `6.18.39+rpt-rpi-2712`.
- Hostname **`raspberrypi`**, and it must come up at **192.168.1.161** and resolve as `raspberrypi` from the other machines. Today `raspberrypi` resolves through the **router's DNS** (`raspberrypi.mynetworksettings.com`), not a hosts file. Both nodes get their address by DHCP and **both have a DHCP reservation on the router** (confirmed by the owner 2026-09-20; reservations are by MAC: `omv` eth0 `2c:cf:67:3f:b3:05`, Pi wlan0 `2c:cf:67:ec:fb:af`, Pi eth0 `2c:cf:67:ec:fb:ae`). A rebuilt Pi with a different network card has a different MAC, so update the reservation.
- **Network:** the original Pi is on **Wi-Fi (`wlan0`)** and its Ethernet port (`eth0`) has no cable. Prefer a cable on rebuild: the control plane and the pod network should not depend on wireless. The Wi-Fi name and password are not stored anywhere in Git: [?].

## 2. Install k3s on the Pi

Same version as the current cluster, default options (the running unit is the stock install-script unit: `ExecStart=/usr/local/bin/k3s server`, no flags, no `config.yaml`): [V]

```sh
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION='v1.36.2+k3s1' sh -s - server        # [D] install script not re-run here
sudo k3s kubectl get nodes                                                               # the Pi should be Ready
```

## 3. Workstation access

Get the new kubeconfig from the Pi, point it at the name and save it (the old file is invalid: the new cluster has a new CA): [D]

```sh
ssh raspberrypi 'sudo cat /etc/rancher/k3s/k3s.yaml' | sed 's#https://127.0.0.1:6443#https://raspberrypi:6443#' > ~/.kube/config-pi-admin
export KUBECONFIG=~/.kube/config-pi-admin && kubectl get nodes
```

## 4. Re-join `omv` (**do not use `k3s-agent-uninstall.sh`**)

`omv` runs `k3s agent` v1.36.2+k3s1 with `K3S_URL='https://raspberrypi:6443'` and a `K3S_TOKEN`, both in `/etc/systemd/system/k3s-agent.service.env`; there is no `config.yaml`. [V]

> **`/usr/local/bin/k3s-agent-uninstall.sh` deletes `/var/lib/rancher/k3s`** (verified by reading the script: `clean_mounted_directory ${K3S_DATA_DIR}`), which includes Loki's local-path volume, and it also removes `/etc/rancher/k3s`, `/var/lib/kubelet` and the `k3s` binary. Do not run it. [V]

Re-register with the new server's token instead: [D]

```sh
# on the Pi: the new server's join token
sudo cat /var/lib/rancher/k3s/server/node-token

# on omv:
sudo systemctl stop k3s-agent
sudo /usr/local/bin/k3s-killall.sh          # stops the OLD cluster's containers; the unit uses KillMode=process, so they keep running otherwise.
                                            # It only unmounts pod mounts and removes the CNI state: it does not touch /var/lib/rancher/k3s/storage. [V: script read]
sudo rm -rf /var/lib/rancher/k3s/agent /etc/rancher/node    # the old node identity (client certs, node password). NOT the whole data dir.
sudo nano /etc/systemd/system/k3s-agent.service.env         # set K3S_TOKEN=<the new token>; leave K3S_URL as is
sudo systemctl daemon-reload && sudo systemctl start k3s-agent
kubectl get nodes                                           # omv Ready
```

This wipes only the agent's own state, so **container images are re-pulled** the first time. Postgres data (`/srv/dev-disk-by-uuid-…/k8s/…`), the photo library and the NFS exports are outside `/var/lib/rancher/k3s` and are not touched.

## 5. Bring Flux back **from Git, not with `flux bootstrap`**

> **Do not run `flux bootstrap`.** It regenerates `clusters/raspberrypi/flux-system/gotk-sync.yaml` and pushes it, which **deletes the hand-added `decryption:` block** ("Manually added (not flux-generated)"). Flux would then be unable to decrypt every `*.enc.yaml` secret. It would also re-add a GitHub `secretRef` that is no longer needed. Apply the manifests already in Git instead. [V: the block is in `gotk-sync.yaml` today]

Create the namespaces the repo does not define (nothing in Git creates `monitoring` or
`immich`; both were made by hand — `cert-manager`'s namespace no longer exists, it was
removed 2026-09-23) [V]. If a `clusters/raspberrypi/namespaces/` directory exists in Git,
Flux creates them and you can skip this line:

```sh
kubectl create namespace monitoring; kubectl create namespace immich
```

Then (from a checkout of the repo): [D]

```sh
# 1. Flux controllers (pinned to the version the cluster runs: Flux v2.9.5)
kubectl apply -f clusters/raspberrypi/flux-system/gotk-components.yaml
kubectl -n flux-system wait --for=condition=Available deploy --all --timeout=300s

# 2. The one Secret that is deliberately NOT in Git (Flux pulls the public repo anonymously: no GitHub credential is needed)
kubectl -n flux-system create secret generic sops-age \
  --from-file=age.agekey="$HOME/.config/sops/age/keys.txt"      # from the off-machine backup

# 3. The sync objects (GitRepository + Kustomization, including the decryption block)
kubectl apply -f clusters/raspberrypi/flux-system/gotk-sync.yaml
flux get all -A                                               # wait for flux-system and metrics to become Ready
```

Flux then creates: the `metrics` HelmRelease (Grafana, Prometheus, Loki, Promtail; the chart is built from Git), the `postgres` Kustomization and `immich` HelmRelease (see below), the Secrets (`metrics-grafana`, `immich-postgres-credentials`), the NetworkPolicies, and the dashboards. `flux reconcile source git flux-system` and `flux reconcile kustomization flux-system` speed it up.

## 6. Nothing is applied by hand anymore [D]

As of 2026-09-23, every component is Flux-managed — Postgres and Immich moved under Flux
(`clusters/raspberrypi/postgres/`, `clusters/raspberrypi/immich/`) and recreate themselves
automatically once Flux reconciles, and cert-manager (the last hand-applied piece) was
removed entirely, having been fully idle (no `ClusterIssuer`/`Issuer`/`Certificate`, no
`Ingress` referenced it — TLS is a deliberate, standing decision). No manual
`kubectl apply`/`helm upgrade` is needed for anything on a rebuild.

**Postgres-before-Immich ordering on a fresh rebuild:** Flux has no `dependsOn` linking
these two — a `Kustomization` can only depend on another `Kustomization`, and a
`HelmRelease` only on another `HelmRelease`, so there's no field that can connect
`postgres`'s `Kustomization` to `immich`'s `HelmRelease` directly. In practice this
self-heals: the Immich server's readiness/liveness probes fail until Postgres is
reachable, and it retries on its own. To be deliberate about it instead: before
reconciling on a rebuild, set `clusters/raspberrypi/immich/helmrelease.yaml`'s
`suspend: true`, wait for `flux get kustomization postgres` to report `Ready`, then flip
it back to `false` — the same staging technique used to originally adopt it.

Traefik, CoreDNS, `local-path` and metrics-server come with k3s and reappear on their own. [V: cluster inventory]

## 7. Verify

- `kubectl get nodes` (both Ready) and `flux get all -A` (everything Ready).
- `helm list -A`: `immich`, `metrics` (and the k3s-managed `traefik`, `traefik-crd`) — `immich` now shows as installed/upgraded by `helm-controller`, not by a human, but still appears here the same way. No `cert-manager` release (removed 2026-09-23).
- All pods Running; Prometheus shows its 19 targets up; the three Grafana dashboards are listed in "Immich Hosting".
- `curl -H 'Host: immich.raspberrypi' http://192.168.1.161/api/server/ping` returns `{"res":"pong"}`; the server log shows ML healthy and no database errors.
- The asset count matches what you expect (79,113-ish); the nightly dump appears the next morning.

## 8. Variants and things that differ

### 8.1 Postgres still on the Pi (before the migration)
The database is gone with the SD card. Restore the newest dump from `omv:/export/storage/Personal/Immich/backups/` into a fresh Postgres **with the `\restrict` lines stripped** (the documented `gunzip < dump | psql` fails on this image); the exact command and a measured 125 s restore time are in `clusters/raspberrypi/postgres/MIGRATION-TO-OMV.md`.

### 8.2 The age key is lost (and so is every copy)
The encrypted secrets are unreadable and must be replaced, not decrypted:
1. `age-keygen -o new.key`; put the **public** key in `.sops.yaml`; create the `sops-age` Secret from `new.key`.
2. Re-create `grafana-admin.enc.yaml` and `postgres.enc.yaml` with **new** passwords (`sops -e`), commit, let Flux apply them.
3. Grafana's database already holds the old admin password: reset it with `grafana cli admin reset-admin-password --password-from-stdin` in the pod. Postgres already holds the old role password: change it with `ALTER ROLE postgres PASSWORD …` over the local socket (trust auth) so the database matches the new Secret. Immich is Flux-managed now, so there's no manual `helm upgrade` to re-run — `flux reconcile helmrelease immich -n immich` if you don't want to wait for the next poll.

### 8.3 Loki's logs
Loki's volume is a `local-path` directory on `omv` under `/var/lib/rancher/k3s/storage/pvc-…_monitoring_storage-metrics-loki-0`. The rebuilt cluster creates a **new** volume, so the old logs (30 days at most) are orphaned but still on disk. Recover them only if they matter, by creating a `local` PV that points at the old directory before Loki starts; otherwise delete the old directory later. [V: path from the live PV]

### 8.4 `omv` is lost too
Out of scope: the photos, database dumps and Grafana/Prometheus data exist only there, with **no off-box copy yet** (planned as part of the backup gap). Recovery would depend on whatever exists outside these machines.

## 9. Things that will bite (all found the hard way in this repo)

- **`flux bootstrap`** removes the SOPS decryption block; use step 5. **`k3s-agent-uninstall.sh`** deletes Loki's data; use step 4.
- **Template changes need a chart version bump** (`monitoring/Chart.yaml`), otherwise Flux keeps deploying its old cached chart. (Changes to the HelmRelease's inlined `spec.values` do not.)
- **Postgres and Immich are Flux-managed** (as of 2026-09-23; `clusters/raspberrypi/postgres/`, `clusters/raspberrypi/immich/`). Postgres's *initial* takeover was staged through a dedicated, temporary `Kustomization` (`suspend: true`, verified with a server-side dry-run diff, then flipped) because its `Deployment` uses `Recreate` strategy against a stateful, node-local volume; that dedicated `Kustomization` was decommissioned 2026-09-26 once the takeover had run safely for days, unifying it into this directory the same way Immich already was — deleting a Flux `Kustomization` with `prune: true` runs its GC finalizer against what it applied, so that decommission was itself a two-PR sequence (`prune: false` first, then delete) — see `clusters/raspberrypi/postgres/README.md`.
- **cert-manager was removed** (2026-09-23), not just left idle: `helm uninstall`, then its CRDs by hand (`helm.sh/resource-policy: keep` means `helm uninstall` leaves them behind on purpose), then `kubectl delete namespace cert-manager` (its namespace entry in Git carried `prune: disabled`, so removing it from Git alone would never have deleted the live namespace — same as every other namespace in `namespaces/namespaces.yaml`).
- `flux suspend` is not reliable here: the Kustomization re-applies `suspend: false` from Git within minutes.
- **Removing a field from Git does not always remove it from the cluster.** Flux applies with server-side apply, and a field owned by *two* managers survives when one of them stops applying it. This bit us when dropping Flux's GitHub token (2026-09-20): `spec.secretRef` on the GitRepository was co-owned by `kustomize-controller` and the `flux` bootstrap CLI, so merging the change did nothing until the field was removed by hand (`kubectl patch ... --type=json -p '[{"op":"remove","path":"/spec/secretRef"}]'`). Check who owns a field with `kubectl get <kind> <name> -o json --show-managed-fields` (kubectl hides them by default), and preview what Flux would do with `kubectl diff --server-side --field-manager=kustomize-controller -f <file>`: plain `kubectl diff` uses client-side logic and shows nothing for such a removal.
- Rate windows in Grafana must stay at a fixed 5m (Prometheus scrapes about once a minute).

## 10. Open items (owner to fill in) [?]

- [x] DHCP reservations exist for both nodes (owner, 2026-09-20).
- [x] **The age key is backed up** (owner, 2026-09-20): a copy in the password manager, and a passphrase-encrypted file `immich-hosting-age-key.txt.age` (its passphrase is in the password manager). Both were verified: the decrypted copy hashes to `12ca947af7b6` and decrypts the repo's secrets (`DECRYPT OK`). **Still to do:** the `.age` file is currently in the owner's home folder, so move it to an offline USB drive kept apart from these machines. [name of the password-manager entry and the USB's location: to be filled in, names only, never the key]
- [x] No GitHub token is needed by Flux (dropped 2026-09-20; the repository is public). The old personal access token should be revoked on GitHub.
- [ ] Will the rebuilt Pi use Ethernet? (Its Wi-Fi credentials are not in Git.)
- [ ] Has a drill ever been run? (No, as of 2026-09-20.)
