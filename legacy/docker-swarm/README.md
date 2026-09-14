# Legacy: Docker Swarm deployment (retired)

This is the original Docker Swarm–based setup for Immich, monitoring, and
Traefik — deployed via `docker stack deploy` using the `start-*.sh` scripts in
this directory. It predates the migration to k3s and is kept here for
reference/rollback only. **It is not the current deployment.**

The live setup today is k3s (Kubernetes) on a Raspberry Pi + a second node
(`omv`), deployed via Helm charts and raw manifests in the repo root — see the
top-level `README.md` for the current component list and apply commands.

A trace of the migration: the Postgres image pinned in the current
`postgres/deployment.yaml` is deliberately the same
`ghcr.io/immich-app/postgres:14-vectorchord...` build referenced here, to keep
on-disk format compatibility with data that originated under this Swarm setup.
