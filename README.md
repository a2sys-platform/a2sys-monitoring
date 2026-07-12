# a2sys-monitoring

Grafana + Prometheus (kube-prometheus-stack) on the existing **devops-dev** GKE cluster.

- **Cluster**: `devops-dev` (project `a2sys-devops-dev`, region `asia-northeast3`)
- **Namespace**: `a2sys-monitoring`
- **Helm release**: `a2sys-monitoring`
- **Chart**: `prometheus-community/kube-prometheus-stack`

> ⚠️ Shared **GKE Autopilot** cluster. Autopilot forbids node-exporter (hostPath/hostNetwork) and
> patching the managed `kube-system` namespace, so `values.yaml` disables node-exporter and the
> control-plane scrapers (kube-controller-manager/scheduler/proxy/etcd, coreDns). Node/system
> metrics come from Google Managed Prometheus (`gmp-system`, already on this cluster); kubelet
> (cAdvisor) scraping stays enabled for pod/container metrics. This release runs its own
> self-managed Prometheus + kube-state-metrics alongside GMP.

## Prerequisites

```sh
gcloud container clusters get-credentials devops-dev \
  --region asia-northeast3 --project a2sys-devops-dev
# kubectl needs the auth plugin on PATH:
export USE_GKE_GCLOUD_AUTH_PLUGIN=True
export PATH="$(gcloud info --format='value(installation.sdk_root)')/bin:$PATH"

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update prometheus-community
```

## Deploy / upgrade

```sh
kubectl create namespace a2sys-monitoring --dry-run=client -o yaml | kubectl apply -f -

# Grafana admin credentials (create once; password is NOT stored in this repo)
kubectl -n a2sys-monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 18)"

helm upgrade --install a2sys-monitoring prometheus-community/kube-prometheus-stack \
  --namespace a2sys-monitoring \
  -f values.yaml
```

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
