# a2sys-monitoring

Monitoring stack for the **devops-dev** GKE cluster. Today it runs **Grafana**;
the layout is organized so more tools (e.g. Kibana) can be added alongside it.

- **Cluster**: `devops-dev` (project `a2sys-devops-dev`, region `asia-northeast3`)
- **Namespace**: `a2sys-monitoring`

## Layout

```
a2sys-monitoring/
├── grafana/                    # Grafana (Helm chart grafana/grafana)
│   ├── values.yaml             #   base: LoadBalancer, admin secret, persistence, dashboard provider
│   ├── values-datasources.yaml #   datasource provisioning (GMP + a2sys-bench)
│   └── dashboards/             #   provisioned dashboards (one JSON per board)
├── sources/                    # shared data-source backends (reused by any tool)
│   ├── gmp/                    #   Google Managed Prometheus query frontend
│   └── a2sys-bench-db/         #   a2sys-bench Cloud SQL over Private Service Connect
└── <tool>/                     # future tools (e.g. kibana/) go here, same pattern
```

**Principle:** each *tool* is a self-contained top-level directory (its own Helm
values / manifests / dashboards). Anything a tool *connects to* — metrics backends,
databases — lives under `sources/` so multiple tools can share it.

## Grafana

Deployed with the `grafana/grafana` Helm chart. Metrics come from Google Managed
Prometheus (no self-managed Prometheus). Datasources:

- `GMP` (default) — Google Managed Prometheus via the `gmp-frontend` proxy → [`sources/gmp/`](sources/gmp/)
- `a2sys-bench` — Cloud SQL (Postgres) over PSC → [`sources/a2sys-bench-db/`](sources/a2sys-bench-db/)

Dashboards → [`grafana/dashboards/`](grafana/dashboards/).

> ⚠️ Shared **GKE Autopilot** cluster. Keep container requests small — Autopilot
> otherwise defaults unset requests to 500m CPU / 2Gi memory.

### Prerequisites

```sh
gcloud container clusters get-credentials devops-dev \
  --region asia-northeast3 --project a2sys-devops-dev
export USE_GKE_GCLOUD_AUTH_PLUGIN=True
export PATH="$(gcloud info --format='value(installation.sdk_root)')/bin:$PATH"

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update grafana
```

### Deploy / upgrade (run from repo root)

```sh
kubectl create namespace a2sys-monitoring --dry-run=client -o yaml | kubectl apply -f -

# Grafana admin credentials (create once; password is NOT stored in this repo)
kubectl -n a2sys-monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 18)"

helm upgrade --install a2sys-monitoring grafana/grafana \
  --namespace a2sys-monitoring \
  -f grafana/values.yaml -f grafana/values-datasources.yaml
```

For the datasources to resolve, also apply the GMP frontend + grant `monitoring.viewer`
([`sources/gmp/`](sources/gmp/)) and create the `bench-db` secret
([`sources/a2sys-bench-db/`](sources/a2sys-bench-db/)).

### Access

```sh
kubectl -n a2sys-monitoring get svc a2sys-monitoring-grafana \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'                    # external IP
kubectl -n a2sys-monitoring get secret grafana-admin \
  -o jsonpath='{.data.admin-password}' | base64 -d                      # admin password
```

Open `http://<EXTERNAL-IP>` → login `admin` / retrieved password.

### Uninstall

```sh
helm uninstall a2sys-monitoring -n a2sys-monitoring
```

## Adding a new tool (e.g. Kibana)

1. Create a top-level directory `<tool>/` with its Helm values / manifests.
2. If it needs a new backend, add it under `sources/<backend>/`; reuse existing ones otherwise.
3. Deploy it as its **own** Helm release (keep releases per-tool), in this same namespace.
4. Document it in this README and the [wiki](https://github.com/a2sys-platform/a2sys-monitoring/wiki).
