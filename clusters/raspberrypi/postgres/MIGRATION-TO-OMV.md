# Migrating Immich's Postgres from the Raspberry Pi to omv

**Status: EXECUTED 2026-09-20 (see "Execution record" at the end).** Postgres now runs on `omv`. The old copy on the Pi is intentionally still there as the rollback. Postgres is applied by hand, so nothing here is applied by merging.

## Why

Postgres lives on the Pi's **SD card**, and the Pi is also the only k3s control-plane node. When the Pi went down (2026-09-20) Immich was lost even though the server pod on `omv` kept running. `omv` already holds the photo library and has three 1 TB SSDs in RAID5 (`md0`). Moving the database there drops Immich's dependencies from {Pi, omv} to {omv} and moves the data from an SD card to a RAID SSD array.

Not chosen: streaming replica or cold standby (promotion needs the control plane, which is on the Pi and cannot be made HA with two nodes), Postgres on NFS (Immich advises against it), hardware changes.

## What changes / what does not

| Changes | Does not change |
|---|---|
| Postgres pod runs on `omv` (nodeSelector) | The `immich-postgres-service` Service (selector `app=immich-postgres`) |
| Data dir moves to `/srv/dev-disk-by-uuid-36f5…/k8s/immich-postgres` on `md0` | Immich's values / `DB_HOSTNAME`, the password Secret |
| New PV `postgres-omv-pv` + PVC `postgres-omv-pvc` | NetworkPolicies (same labels) |
| Resource requests/limits added (250m / 512Mi, 1.5Gi memory limit) | The image (same digest), the `vchord.so` preload |

The data directory is **deliberately not under `/export/storage`**: `omv` exports that to the whole LAN with `no_root_squash`. The raw mount of the same filesystem is not exported.

## Prerequisites

- [ ] No Immich job running (Jobs page: nothing active or waiting).
- [ ] Latest nightly dump present and recent in `omv:/export/storage/Personal/Immich/backups/`.
- [ ] k3s state backup status checked (separate task).
- [ ] Owner has approved the window.

## Sequence (Immich is down from step 3 to step 9: about 15 minutes)

Set once: `export KUBECONFIG=~/.kube/config-pi-admin`, `RAW=/srv/dev-disk-by-uuid-36f537fb-c606-4ed2-ae22-55ad447d5329`, `D=$RAW/k8s/immich-postgres`. (`RAW` and `D` are **site-specific**: they must match `spec.local.path` in `pvc.yaml` (this directory). Replicating elsewhere: see "Replicating this setup elsewhere" in the repo README.)

1. **Baseline** (save the output to compare later):
   `kubectl -n immich exec deploy/immich-postgres -- psql -U postgres -d immich -At -c "select (select count(*) from asset),(select count(*) from \"user\"),(select count(*) from album),(select count(*) from smart_search),(select count(*) from asset_face)"`
2. **Safety dump** (custom format) straight to omv:
   `kubectl -n immich exec deploy/immich-postgres -- pg_dump -U postgres -Fc immich | ssh omv "sudo tee $RAW/k8s/pre-migration-$(date +%Y%m%d-%H%M).dump.partial >/dev/null && sudo chmod 600 ...partial && sudo mv ...partial ...dump"` then check it with `pg_restore -l` (read it back over ssh into the pod). **Write it to the raw mount (root-only), not under `/export/storage`**: the export is readable by the whole LAN and the dump is your entire database.
3. **Create the target directory:** `ssh omv "sudo mkdir -p $D && sudo chown 999:999 $D && sudo chmod 700 $D"`
4. **Quiesce Immich:** `kubectl -n immich scale deploy/immich-server --replicas=0` (Valkey and ML stay up). Do **not** run `helm upgrade` until step 10.
5. **Stop Postgres cleanly:** `kubectl -n immich scale deploy/immich-postgres --replicas=0`, wait for the pod to be gone.
6. **Copy the data directory.** The Pi's data is root-only and there is no passwordless sudo on the Pi, so use a helper pod on the Pi (postgres image, runs as root, mounts the old path read-only) and stream a tar through the workstation into `omv`:
   - Helper: a pod with `nodeSelector: raspberrypi`, `securityContext.runAsUser: 0`, hostPath `/home/divyakumarjain/photos/immich/postgres` mounted read-only at `/src`, command `sleep 3600`.
   - **Confirm the source was shut down cleanly:** `kubectl exec <helper> -- pg_controldata /src | grep 'cluster state'` must say `shut down`.
   - Copy: `kubectl -n immich exec <helper> -- tar -C /src -cpf - . | ssh omv "sudo tar -C $D -xpf - --numeric-owner"`
   - **Verify** (all must match): entry count, file count, total bytes, and a hash of every file's *contents*. **Always use `LC_ALL=C` on `sort`, `find` and `xargs`**: the container (`en_US.utf8`) and `omv` (`C.UTF-8`) order files differently, so a plain `sort` gives different hashes for identical data (this happened during the execution and looked like a corrupt copy until the locale was fixed).
     - Pi (in helper): `cd /src && LC_ALL=C find . -type f | LC_ALL=C sort | LC_ALL=C xargs sha256sum | sha256sum`
     - omv: `sudo bash -c 'cd $D && LC_ALL=C find . -type f | LC_ALL=C sort | LC_ALL=C xargs sha256sum | sha256sum'`
   - **If anything differs: stop.** Delete the target contents and repeat; do not start Postgres on a partial copy.
7. **Apply and start on omv:** `kubectl apply -f postgres/pvc.yaml` then `kubectl -n immich apply -f postgres/deployment.yaml`; wait for the rollout.
8. **Verify Postgres** before touching Immich:
   - Log shows `database system was shut down at …` then `ready to accept connections`, and **no** `not properly shut down; automatic recovery`.
   - `SHOW shared_preload_libraries` is `vchord.so`; `\dx` lists `vchord 0.4.3`, `vector 0.8.0`.
   - The step 1 counts match exactly.
   - A similarity query works (needs `SET vchordrq.probes = 1;` first): `select "assetId" from smart_search order by embedding <=> (select embedding from smart_search limit 1) limit 5;`
9. **Bring Immich back:** `kubectl -n immich scale deploy/immich-server --replicas=1`, then check `/api/server/ping` through the ingress, no auth/connection errors in the server log, the server still sees ML healthy, all four pods `1/1`. Delete the helper pod.
10. **Afterwards:** confirm the *next nightly dump* is written by the server against the new Postgres; the Nodes and Immich dashboards should show Postgres on `omv`.

## Rollback

- **Before step 9** (nothing has written to the new database): `kubectl -n immich scale deploy/immich-postgres --replicas=0`, then point the Deployment back at the Pi (the old PV/PVC objects still exist, `Retain`):
  `kubectl -n immich patch deploy immich-postgres --type=json -p '[{"op":"replace","path":"/spec/template/spec/nodeSelector/kubernetes.io~1hostname","value":"raspberrypi"},{"op":"replace","path":"/spec/template/spec/volumes/0/persistentVolumeClaim/claimName","value":"postgres-local-pvc"}]'`
  and scale Postgres then the server back up. The Pi's data was only ever read.
- **After step 9:** writes now land on `omv`. Rolling back means taking a fresh `pg_dump -Fc` from the new database and restoring it into the old location with `pg_restore`, losing nothing but time.
- **Last resort:** the step 2 safety dump and the nightly dumps. **Restore gotcha:** the documented `gunzip < dump.sql.gz | psql` fails on this image (`invalid command \restrict`, because the dump is written by pg_dump 14.23 and the image's psql is 14.18). Strip those lines: `gunzip -c dump.sql.gz | sed -E '/^\\(un)?restrict( |$)/d' | psql -U postgres -d immich -v ON_ERROR_STOP=1`. A restore of a nightly dump took 125 s in the 2026-09-20 rehearsal.

## Decommissioning the old copy (later, with the owner's approval)

Keep the Pi's data directory and the old PV/PVC **untouched for several days** while the new setup runs. Then delete `postgres/pvc-raspberrypi-rollback.yaml`, the live `postgres-local-pv` / `postgres-local-pvc`, and the Pi's data directory together.

## Things to remember

- uid/gid 999 (Postgres) is the `dnsmasq` user on `omv`'s own OS. The directory is mode 700; it is only a name collision.
- Postgres is now pinned to `omv` (data gravity). If `omv` is down, Immich is down: it needs `omv` for the photos anyway.
- Still to decide separately: a second CoreDNS replica and a second A record (or VIP) for `immich.raspberrypi`, so a Pi outage does not take the name or DNS with it.

## Execution record (2026-09-20)

| | |
|---|---|
| Immich downtime | **4 min 23 s** (server scaled to 0 at 20:24:00 UTC, back at 20:28:23 UTC) |
| Safety dump | 622 MB custom-format, root-only, 182 TOC entries, taken before any downtime |
| Source state | `pg_controldata`: `Database cluster state: shut down`, final checkpoint 20:24:01 UTC, no `postmaster.pid` |
| Copy | 1,820 entries / 1,789 files / 2,024,636,521 bytes in **65 s** via a read-only root helper pod on the Pi |
| Verification | entries, files, bytes equal; SHA-256 over the contents of **all 1,789 files identical** on both sides |
| First start on omv | `database system was shut down at 20:24:01 UTC` then `ready to accept connections`: **no crash recovery** |
| Data | counts identical to the pre-migration baseline (79,117 assets / 2 users / 9 albums / 69,927 embeddings / 150,739 faces / 79,114 exif); schema 66 / 474 / 231 / 523; extensions identical; 0 invalid indexes; both vector indexes used |
| After restart | ingress `pong` via both nodes, no server errors, ML healthy, 6 server connections to the new Postgres, resources applied (250m / 512Mi, 1.5Gi limit) |
| NetworkPolicy | unchanged and correct after the move (see below) |

**What was learned**
- The manifest hash first differed because of the locale (see step 6); the data was identical.
- A **brand-new pod's first connection attempt** after creation can be refused for a few seconds by the NetworkPolicy while the rules pick up its IP (first of eight attempts blocked, the rest reached). A rescheduled server pod may therefore retry briefly; Immich did so without errors.
- The runbook originally put the safety dump under the exported `backups/` folder; it now goes to the raw mount, root-only.

**Still in place, on purpose (do not delete yet):** the old PV/PVC objects (`postgres-local-pv`, `postgres-local-pvc`), the Pi's data directory `/home/divyakumarjain/photos/immich/postgres` (untouched, cleanly shut down: it is the rollback), and the safety dump `$RAW/k8s/pre-migration-20260920-1621.dump`. Decommission them together, with the owner's approval, after the new setup has run for several days and at least one nightly dump has been written against the new Postgres.
