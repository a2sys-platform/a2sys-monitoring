# a2sys-monitoring

Grafana on the existing **devops-dev** GKE cluster. Metrics come from Google
Managed Prometheus; there is no self-managed Prometheus.

- **Cluster**: `devops-dev` (project `a2sys-devops-dev`, region `asia-northeast3`)
- **Namespace**: `a2sys-monitoring`
- **Helm release**: `a2sys-monitoring`
- **Chart**: `grafana/grafana`

**Datasources**
- `GMP` (default) — Google Managed Prometheus, via the `gmp-frontend` proxy → see [`gmp/`](gmp/)
- `a2sys-bench` — a2sys-bench Cloud SQL (Postgres) over PSC → see [`db-connection/`](db-connection/)

> ⚠️ Shared **GKE Autopilot** cluster. Metrics are served by the cluster's Google
> Managed Prometheus, so this release runs only Grafana. Container requests are kept
> small — Autopilot otherwise defaults unset requests to 500m CPU / 2Gi memory.

## Files

| Path | Purpose |
|---|---|
| `values.yaml` | Grafana base (LoadBalancer, admin secret, persistence, requests) |
| `values-db.yaml` | Datasources overlay (GMP + a2sys-bench) |
| `gmp/` | Google Managed Prometheus query frontend + datasource |
| `db-connection/` | PSC path + Grafana → Cloud SQL datasource |

## Prerequisites

```sh
gcloud container clusters get-credentials devops-dev \
  --region asia-northeast3 --project a2sys-devops-dev
# kubectl needs the auth plugin on PATH:
export USE_GKE_GCLOUD_AUTH_PLUGIN=True
export PATH="$(gcloud info --format='value(installation.sdk_root)')/bin:$PATH"

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update grafana
```

## Deploy / upgrade

```sh
kubectl create namespace a2sys-monitoring --dry-run=client -o yaml | kubectl apply -f -

# Grafana admin credentials (create once; password is NOT stored in this repo)
kubectl -n a2sys-monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 18)"

helm upgrade --install a2sys-monitoring grafana/grafana \
  --namespace a2sys-monitoring \
  -f values.yaml -f values-db.yaml
```

For the datasources to resolve, also apply the GMP frontend + grant `monitoring.viewer`
(see [`gmp/`](gmp/)) and create the `bench-db` secret (see [`db-connection/`](db-connection/)).

## Access Grafana

```sh
# External IP (LoadBalancer)
kubectl -n a2sys-monitoring get svc a2sys-monitoring-grafana \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# Admin password
kubectl -n a2sys-monitoring get secret grafana-admin \
  -o jsonpath='{.data.admin-password}' | base64 -d
```

Open `http://<EXTERNAL-IP>` → login `admin` / retrieved password.

## Uninstall

```sh
helm uninstall a2sys-monitoring -n a2sys-monitoring
kubectl delete namespace a2sys-monitoring   # also removes PVCs
```
