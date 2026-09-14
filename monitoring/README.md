# monitoring chart

Umbrella chart (Prometheus, Grafana, Loki, Promtail) bundling the four upstream subcharts
pinned in `Chart.yaml`/`Chart.lock`. Installed directly from this directory, not published
anywhere:

```
helm dependency build
helm upgrade --install metrics . -f values.yaml --namespace monitoring --create-namespace
```

Verified byte-identical to `/home/divyakumarjain/photos/immich/metrics/` on raspberrypi and
`immich-infra/docs/audit-raw/custom-metrics-chart/`.

## Known gaps (see immich-infra/docs/CURRENT_STATE.md for full detail)

- No Alertmanager / no alerting configured anywhere.
- No Grafana dashboards provisioned, only the Prometheus + Loki datasources.
- `loki-canary` / `chunks-cache` / `results-cache` are set `enabled: false` in `values.yaml`
  but the chunks-cache/results-cache pods run anyway — these keys likely don't map to this
  chart version's actual value paths. Not fixed here; flagged for a Fixer proposal.
- No Loki retention period configured — logs accumulate indefinitely on a 10Gi node-local
  volume.
- Prometheus's PV NFS path is the export root (`/export/storage`), not a scoped
  subdirectory like Grafana's — worth confirming it isn't sharing space with unrelated data.
