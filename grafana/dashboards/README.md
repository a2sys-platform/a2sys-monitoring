# Dashboards

Provisioned Grafana dashboards (survive redeploys, unlike UI-created ones).

- `a2sys-bench.json` — a2sys-bench overview: runs/trials totals, resolve rate,
  cost, tokens, resolve-rate & tokens by model, trials over time, recent runs.
  Uses the `a2sys-bench` Postgres datasource (uid `a2sysbench`, pinned in `../values-datasources.yaml`).

## How it's wired

`../values.yaml` defines a `file` dashboard provider that reads
`/var/lib/grafana/dashboards/default`, mounted from the ConfigMap
`a2sys-bench-dashboards`. That ConfigMap is built from the JSON files here.

## Apply / update

After editing any JSON in this folder, rebuild the ConfigMap and restart Grafana
(run from repo root):

```sh
kubectl -n a2sys-monitoring create configmap a2sys-bench-dashboards \
  --from-file=grafana/dashboards/a2sys-bench.json \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n a2sys-monitoring rollout restart deploy/a2sys-monitoring-grafana
```

> Add another `--from-file=grafana/dashboards/<name>.json` for each additional dashboard.

(First-time only, also `helm upgrade` so the provider config is present — see repo README.)
