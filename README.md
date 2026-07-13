# a2sys-monitoring

**devops-dev** GKE 클러스터용 모니터링 스택. 현재는 **Grafana** 를 운영하며,
Kibana 등 다른 도구도 나란히 추가할 수 있도록 폴더 구조를 구성했다.

- **클러스터**: `devops-dev` (project `a2sys-devops-dev`, region `asia-northeast3`)
- **네임스페이스**: `a2sys-monitoring`

## 폴더 구조

```
a2sys-monitoring/
├── grafana/                    # Grafana (Helm 차트 grafana/grafana)
│   ├── values.yaml             #   기본: LoadBalancer, admin 시크릿, 영속성, 대시보드 provider
│   ├── values-datasources.yaml #   데이터소스 프로비저닝 (GMP + a2sys-bench)
│   └── dashboards/             #   프로비저닝 대시보드 (보드당 JSON 1개)
├── sources/                    # 도구 공통 데이터소스 백엔드 (여러 도구가 재사용)
│   ├── gmp/                    #   Google Managed Prometheus 쿼리 frontend
│   └── a2sys-bench-db/         #   a2sys-bench Cloud SQL (Private Service Connect)
└── <tool>/                     # 향후 도구(예: kibana/)는 같은 패턴으로 여기에
```

**원칙:** *도구* 는 각각 독립된 최상위 디렉토리(자체 Helm values/매니페스트/대시보드).
도구가 *연결하는 대상*(메트릭 백엔드, DB)은 여러 도구가 공유할 수 있도록 `sources/` 아래에 둔다.

## Grafana

`grafana/grafana` Helm 차트로 배포. 메트릭은 Google Managed Prometheus에서 가져오며
(self-managed Prometheus 없음), 데이터소스는:

- `GMP` (기본) — Google Managed Prometheus, `gmp-frontend` 프록시 경유 → [`sources/gmp/`](sources/gmp/)
- `a2sys-bench` — Cloud SQL (Postgres), PSC 경유 → [`sources/a2sys-bench-db/`](sources/a2sys-bench-db/)

대시보드 → [`grafana/dashboards/`](grafana/dashboards/).

> ⚠️ 공용 **GKE Autopilot** 클러스터. 컨테이너 request를 작게 유지할 것 — 안 그러면
> Autopilot이 미지정 request를 컨테이너당 500m CPU / 2Gi 메모리로 기본 할당함.

### 사전 준비

```sh
gcloud container clusters get-credentials devops-dev \
  --region asia-northeast3 --project a2sys-devops-dev
export USE_GKE_GCLOUD_AUTH_PLUGIN=True
export PATH="$(gcloud info --format='value(installation.sdk_root)')/bin:$PATH"

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update grafana
```

### 배포 / 업그레이드 (리포 루트에서 실행)

```sh
kubectl create namespace a2sys-monitoring --dry-run=client -o yaml | kubectl apply -f -

# Grafana admin 자격증명 (최초 1회 생성. 비밀번호는 리포에 저장하지 않음)
kubectl -n a2sys-monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -base64 18)"

helm upgrade --install a2sys-monitoring grafana/grafana \
  --namespace a2sys-monitoring \
  -f grafana/values.yaml -f grafana/values-datasources.yaml
```

데이터소스가 연결되려면, GMP frontend 적용 + `monitoring.viewer` 부여
([`sources/gmp/`](sources/gmp/)), 그리고 `bench-db` 시크릿 생성
([`sources/a2sys-bench-db/`](sources/a2sys-bench-db/))도 필요하다.

### 접속

```sh
kubectl -n a2sys-monitoring get svc a2sys-monitoring-grafana \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'                    # 외부 IP
kubectl -n a2sys-monitoring get secret grafana-admin \
  -o jsonpath='{.data.admin-password}' | base64 -d                      # admin 비밀번호
```

`http://<EXTERNAL-IP>` 접속 → `admin` / 조회한 비밀번호로 로그인.

### 제거

```sh
helm uninstall a2sys-monitoring -n a2sys-monitoring
```

## 새 도구 추가 (예: Kibana)

1. 최상위에 `<tool>/` 디렉토리를 만들고 해당 Helm values / 매니페스트를 둔다.
2. 새 백엔드가 필요하면 `sources/<backend>/` 에 추가하고, 기존 것은 재사용한다.
3. **도구별로 별도 Helm 릴리스**로 배포한다(같은 네임스페이스 사용).
4. 이 README와 [위키](https://github.com/a2sys-platform/a2sys-monitoring/wiki)에 문서화한다.
