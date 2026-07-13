# Connect Grafana → a2sys-bench Cloud SQL (Private Service Connect)

> **Status: LIVE.** PSC endpoint `cloudsql-a2sys-bench-psc` (internal IP `10.10.0.15`,
> pscConnectionStatus ACCEPTED) in `devops-dev-vpc`. Grafana datasource `a2sys-bench`
> connects directly and passes health check. This runbook is the reproduce/rollback record.

Grafana runs on `devops-dev` (project `a2sys-devops-dev`, VPC `devops-dev-vpc`).
The DB `a2sys-bench-pg-poc` is **private-IP-only** in a **different project** (`cs-poc`,
default VPC). PSC bridges the two privately: cs-poc publishes a PSC service attachment,
we create a consumer endpoint (internal IP) inside `devops-dev-vpc`, and Grafana connects
to that IP as a normal Postgres client — no Auth Proxy, no cross-project `cloudsql.client`.

## Facts

| | |
|---|---|
| Instance | `a2sys-bench-pg-poc` (POSTGRES_16) |
| Connection name | `cs-poc-6qmyyhra03nr8izqgj3wecj:asia-northeast3:a2sys-bench-pg-poc` |
| DB project | `cs-poc-6qmyyhra03nr8izqgj3wecj` (num `999254374398`) |
| Consumer project | `a2sys-devops-dev` (num `815251480853`) |
| Consumer VPC / subnet | `devops-dev-vpc` / `devops-dev-gke` (asia-northeast3, 10.10.0.0/20) |
| DB name / user | `a2sys_bench` / `a2sys-bench` |
| DB password | Secret Manager `a2sys-bench-db-password` (project cs-poc) |

---

## Step 1 — cs-poc admin (REQUIRED, cannot be done without cs-poc admin)

Enable PSC on the instance and authorize the consumer project:

```sh
gcloud sql instances patch a2sys-bench-pg-poc \
  --project cs-poc-6qmyyhra03nr8izqgj3wecj \
  --enable-private-service-connect \
  --allowed-psc-projects=a2sys-devops-dev
```

Then read the service attachment URI (needed for Step 2):

```sh
gcloud sql instances describe a2sys-bench-pg-poc \
  --project cs-poc-6qmyyhra03nr8izqgj3wecj \
  --format="value(pscServiceAttachmentLink)"
# also note the PSC dnsName:
gcloud sql instances describe a2sys-bench-pg-poc \
  --project cs-poc-6qmyyhra03nr8izqgj3wecj \
  --format="value(dnsNames)"
```

> Requires `cloudsql.instances.update` on cs-poc (roles/cloudsql.admin or editor).
> My account holds only `cloudsql.instanceUser` there, so I cannot run this step.

## Step 2 — devops-dev: create the PSC consumer endpoint

```sh
SA="<pscServiceAttachmentLink from Step 1>"

gcloud compute addresses create cloudsql-a2sys-bench-psc \
  --project a2sys-devops-dev --region asia-northeast3 \
  --subnet devops-dev-gke

gcloud compute forwarding-rules create cloudsql-a2sys-bench-psc \
  --project a2sys-devops-dev --region asia-northeast3 \
  --network devops-dev-vpc \
  --address cloudsql-a2sys-bench-psc \
  --target-service-attachment "$SA" \
  --allow-psc-global-access

# the endpoint's internal IP (== Grafana DB host):
PSC_IP=$(gcloud compute addresses describe cloudsql-a2sys-bench-psc \
  --project a2sys-devops-dev --region asia-northeast3 --format="value(address)")
echo "$PSC_IP"
```

## Step 3 — k8s secret + Grafana datasource

```sh
PW=$(gcloud secrets versions access latest \
  --secret=a2sys-bench-db-password --project cs-poc-6qmyyhra03nr8izqgj3wecj)

kubectl -n a2sys-monitoring create secret generic bench-db \
  --from-literal=host="$PSC_IP" \
  --from-literal=user=a2sys-bench \
  --from-literal=password="$PW"

cd ..   # a2sys-monitoring/
helm upgrade --install a2sys-monitoring prometheus-community/kube-prometheus-stack \
  -n a2sys-monitoring -f values.yaml -f values-db.yaml
```

## Verify

Grafana → Connections → Data sources → `a2sys-bench` → **Save & test** (should be green).
Or query in Explore: `SELECT count(*) FROM runs;`

## Rollback

```sh
helm upgrade a2sys-monitoring prometheus-community/kube-prometheus-stack -n a2sys-monitoring -f values.yaml
kubectl -n a2sys-monitoring delete secret bench-db
gcloud compute forwarding-rules delete cloudsql-a2sys-bench-psc --project a2sys-devops-dev --region asia-northeast3
gcloud compute addresses delete cloudsql-a2sys-bench-psc --project a2sys-devops-dev --region asia-northeast3
```
