# Immich dev instance (`dev.immich.raspberrypi`)

A second Immich server that runs a **custom build** next to the real one. It exists to use
features that are not in an upstream release yet, currently the *face graph* (Utilities →
"Review people by face similarity").

|  | Real instance | Dev instance |
|---|---|---|
| Hostname | `immich.raspberrypi` | `dev.immich.raspberrypi` |
| Managed by | Helm chart, `../immich/` | raw manifests, this directory |
| Image | `ghcr.io/immich-app/immich-server:v3.2.4` | `ghcr.io/divyakumarjain/immich-server:v3.2.4-face-graph.6` |
| Workers | API + background jobs | API only |
| Database, Valkey, machine-learning, photo library | — | **the same ones** |

Both hostnames show the same library, people and users. A change made on one (rename a
person, move faces) is visible on the other at once, because there is only one database.

## Why the two can share one database

Two Immich servers on one database are only safe while all of the following hold.

1. **Same upstream version.** The dev image is upstream `v3.2.4` plus the feature, and the
   feature adds no database migrations (`server/src/schema` is identical to `v3.2.4`). An
   image built from a newer Immich would migrate the database on start and break the real
   server.
2. **`DB_SKIP_MIGRATIONS=true`** on the dev pod, as a second guard for rule 1: it never
   changes the schema.
3. **`IMMICH_WORKERS_INCLUDE=api`** on the dev pod. It runs no background jobs. Jobs it queues
   (for example a new person thumbnail after faces are moved) go to the shared Valkey and
   are run by the real server, which has the same job code because of rule 1.
4. **Same photo library volume** (`immich-nfs-pvc` at `/data`). The server refuses to start
   if the folder markers recorded in the database are missing from its `/data`.

Checked before this was added: a local copy of the image was started as API-only with
`DB_SKIP_MIGRATIONS=true` against a database and Valkey already in use by a second server of
the same version. It started without running migrations, served the web app and the new
endpoints, and edits were visible through both servers.

## Upgrading Immich: do this in order

**Before** changing the image tag in `../immich/values.yaml`:

1. Set `replicas: 0` in `deployment.yaml` here (or delete this directory) and merge.
2. Upgrade the real instance as usual.
3. Rebuild the dev image on the new upstream tag (below), update the image here, set
   `replicas: 1` again.

If the real instance is upgraded while the dev pod still runs, the dev pod keeps serving
old code against a migrated schema: expect errors on the dev hostname and possibly jobs
queued in an old format. It will not migrate anything itself (rule 2).

## Building the image

From the fork https://github.com/divyakumarjain/immich, branch `feat/face-graph-v3.2.4`
(tag `v3.2.4` + one commit):

```sh
docker buildx build --platform linux/arm64 -f server/Dockerfile \
  --build-arg BUILD_SOURCE_REF=feat/face-graph-v3.2.4 \
  --build-arg BUILD_SOURCE_COMMIT=$(git rev-parse HEAD) \
  -t ghcr.io/divyakumarjain/immich-server:v3.2.4-face-graph.6 --push .
```

Both nodes are arm64, so one platform is enough. The package must be **public** on ghcr.io:
the cluster has no image pull secret. Use a new tag for every build (`…-face-graph.2`), the
pod uses `imagePullPolicy: IfNotPresent` and will not re-pull a tag it already has.

## Things to know

- **Nightly jobs.** The API worker that starts first holds the "nightly jobs" lock and queues
  them. Normally that is the real server. If the real server restarts while the dev pod is
  up, the dev pod takes over queuing them (they still run on the real server). If the dev pod
  is then removed, restart `immich-server` so it takes the lock back.
- **Access.** The pod is labelled `app.kubernetes.io/name: server`, which is what the
  NetworkPolicies in `../network-policies/immich.yaml` match, so it needs no policy of its
  own. It is labelled `app.kubernetes.io/instance: immich-dev`, so the real `immich-server`
  Service (selector `instance: immich`) never sends traffic to it.
- **Metrics.** Telemetry is off on the dev pod and its Service has no scrape annotations.
- **Mobile apps** should keep using `immich.raspberrypi`.
- **Removing it.** Delete this directory and merge. Flux prunes the Deployment, Service and
  Ingress. Nothing here owns data.
