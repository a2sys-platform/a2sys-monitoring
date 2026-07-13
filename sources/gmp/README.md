# GMP datasource for Grafana

Metrics come from **Google Managed Prometheus** (GMP), which already runs managed
collection on the `devops-dev` cluster (`gke-gmp-system`). Grafana queries it
through a small **query frontend** proxy.

## Components

- `frontend.yaml` — `gmp-frontend` Deployment + Service + ServiceAccount. The frontend
  (`prometheus-engine/frontend`, matched to the cluster's GMP version `v0.18.0-gke.2`)
  proxies PromQL to the Managed Prometheus API for project `a2sys-devops-dev`.
- Grafana datasource `GMP` (type `prometheus`, default) → `http://gmp-frontend.a2sys-monitoring.svc:9090`
  (defined in `../values-datasources.yaml`).

## Auth (Workload Identity)

The frontend authenticates to the Monitoring API via its KSA's Workload Identity
principal, which must hold `roles/monitoring.viewer` on `a2sys-devops-dev`:

```sh
gcloud projects add-iam-policy-binding a2sys-devops-dev \
  --role=roles/monitoring.viewer \
  --member="principal://iam.googleapis.com/projects/815251480853/locations/global/workloadIdentityPools/a2sys-devops-dev.svc.id.goog/subject/ns/a2sys-monitoring/sa/gmp-frontend" \
  --condition=None
```

## Deploy

```sh
kubectl apply -f sources/gmp/frontend.yaml
# grant monitoring.viewer (above), then provision the datasource:
helm upgrade a2sys-monitoring grafana/grafana \
  -n a2sys-monitoring -f grafana/values.yaml -f grafana/values-datasources.yaml
```

## Verify

```sh
kubectl -n a2sys-monitoring port-forward svc/gmp-frontend 19090:9090 &
curl -s "http://localhost:19090/api/v1/query?query=up" | head -c 200
```

In Grafana: Connections → Data sources → `GMP` → **Save & test** (should be green).
