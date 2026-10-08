# Istio 사용법

> 순서는 **설치(`istioctl install`) → Gateway API CRD 설치 → 네임스페이스를 메시에 편입 → Gateway로 외부 노출 → 트래픽·보안 정책 적용 → 관측·디버깅**이다. 이 문서는 kind 클러스터에서 Istio 1.31.1과 Bookinfo 샘플로 두 모드를 각각 따라 하는 최소 흐름이다. 전체 실행 스크립트와 YAML은 [examples/](./examples/)에 있다.

## 준비물

| 도구 | 용도 | 비고 |
| --- | --- | --- |
| Docker | kind 노드 컨테이너 실행 | |
| [kind](https://kind.sigs.k8s.io/) | 로컬 Kubernetes 클러스터 | Istio 1.31 지원 Kubernetes 버전 범위 안의 노드 이미지를 쓴다([Supported Releases](https://istio.io/latest/docs/releases/supported-releases/)) |
| kubectl | 클러스터 조작 | |
| istioctl 1.31.1 | 설치·진단 CLI | 아래 다운로드 스크립트에 포함 |

```bash
kind create cluster --name istio-toy

curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.31.1 sh -
cd istio-1.31.1
export PATH=$PWD/bin:$PATH
istioctl version
```

- `ISTIO_VERSION` 환경변수로 버전을 고정할 수 있다. 생략하면 그 시점의 최신 버전을 받는다.
- 이후 `samples/...` 경로는 모두 `istio-1.31.1/` 디렉토리 기준이다.

## 설치 프로필

| 프로필 | 용도 |
| --- | --- |
| `default` | 운영 시작점. istiod + ingress gateway |
| `demo` | 기능 시연용("showcase Istio functionality"). istiod + ingress/egress gateway. 운영 비권장 |
| `minimal` | 컨트롤 플레인(istiod)만 |
| `ambient` | istiod + Istio CNI + ztunnel (ambient 모드) |
| `empty` / `remote` / `preview` | 커스텀 베이스 / 외부 컨트롤 플레인의 remote 클러스터 / 실험 기능 |

- 출처: [Installation Configuration Profiles](https://istio.io/latest/docs/setup/additional-setup/config-profiles/)
- 공식 Getting Started는 Gateway API로 ingress를 따로 만들기 때문에, gateway를 미리 설치하지 않는 프로필 변형을 쓴다.

## A. Sidecar 모드로 시작하기

### 1. 설치 + Gateway API CRD

```bash
istioctl install -f samples/bookinfo/demo-profile-no-gateways.yaml -y

kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.6.0" | kubectl apply -f -; }
```

### 2. 메시 편입 + 앱 배포

```bash
kubectl label namespace default istio-injection=enabled
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/curl/curl.yaml
kubectl get pods   # READY 2/2 = 앱 + istio-proxy
```

- 라벨을 붙이기 **전에** 떠 있던 Pod에는 sidecar가 없다. `kubectl rollout restart deploy -n default`로 재시작해야 한다.

### 3. Ingress (Gateway API)

```bash
kubectl apply -f samples/bookinfo/gateway-api/bookinfo-gateway.yaml
kubectl annotate gateway bookinfo-gateway networking.istio.io/service-type=ClusterIP --namespace=default
kubectl wait --for=condition=programmed gtw bookinfo-gateway
kubectl port-forward svc/bookinfo-gateway-istio 8080:80
# 브라우저: http://localhost:8080/productpage
```

- `gatewayClassName: istio`인 `Gateway`를 만들면 Istio가 `bookinfo-gateway-istio`라는 Deployment·Service·ServiceAccount를 자동으로 만든다([Gateway API task](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/)).
- kind에는 LoadBalancer가 없어서 `ClusterIP`로 바꾸고 port-forward로 접근한다.

### 4. 트래픽 관리: 카나리 + 재시도

`DestinationRule`로 버전별 subset을 정의하고, `VirtualService`로 가중치를 나눈다.

```yaml
# examples/sidecar/02-destination-rule-reviews.yaml (발췌)
apiVersion: networking.istio.io/v1
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    loadBalancer:
      simple: LEAST_REQUEST
    outlierDetection:          # 5xx가 연속되면 해당 Pod를 일시 제외(circuit breaking)
      consecutive5xxErrors: 5
      interval: 10s
      baseEjectionTime: 30s
  subsets:
  - name: v1
    labels: { version: v1 }
  - name: v2
    labels: { version: v2 }
```

```yaml
# examples/sidecar/03-virtual-service-reviews-canary.yaml
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts: [reviews]
  http:
  - route:
    - destination: { host: reviews, subset: v1 }
      weight: 90
    - destination: { host: reviews, subset: v2 }
      weight: 10
    timeout: 3s
    retries:
      attempts: 2
      perTryTimeout: 1s
      retryOn: 5xx,connect-failure,reset
```

```bash
kubectl apply -f examples/sidecar/02-destination-rule-reviews.yaml
kubectl apply -f examples/sidecar/03-virtual-service-reviews-canary.yaml
# 확인: 100번 중 약 10번 reviews-v2
kubectl exec deploy/curl -- sh -c 'for i in $(seq 1 100); do curl -s http://productpage:9080/productpage | grep -o "reviews-v[0-9]-[a-z0-9]*"; done' | sort | uniq -c
```

| 기본값 | 내용 | 출처 |
| --- | --- | --- |
| HTTP 타임아웃 | 기본 비활성 → `timeout`을 명시해야 함 | [Traffic management](https://istio.io/latest/docs/concepts/traffic-management/) |
| HTTP 재시도 | 기본 2회 재시도 | 같은 문서 |
| 로드 밸런싱 | 기본 least requests | 같은 문서 |

- 같은 기능을 Gateway API `HTTPRoute`로도 표현할 수 있다(`samples/bookinfo/gateway-api/route-reviews-90-10.yaml`). Istio는 Gateway API를 장래의 기본 API로 삼을 계획이다.

### 5. 보안: STRICT mTLS + 인가

```bash
kubectl apply -f examples/sidecar/01-peer-authentication-strict.yaml   # 평문 거부
kubectl apply -f examples/sidecar/04-authz-productpage.yaml            # gateway의 GET만 허용
```

```yaml
# examples/sidecar/04-authz-productpage.yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: productpage-allow-gateway
spec:
  selector:
    matchLabels: { app: productpage }
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-gateway-istio"]
    to:
    - operation:
        methods: ["GET"]
```

```bash
# 이제 curl Pod에서 productpage 직접 호출 → 403 (RBAC: access denied)
kubectl exec deploy/curl -- curl -s -o /dev/null -w "%{http_code}\n" http://productpage:9080/productpage
```

## B. Ambient 모드로 시작하기

새 클러스터에서 진행한다(같은 클러스터를 쓰려면 먼저 `istioctl uninstall --purge -y`로 지우고 `istio-injection` 라벨을 제거한다).

### 1. 설치 + Gateway API CRD

```bash
istioctl install --set profile=ambient --skip-confirmation
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.6.0" | kubectl apply -f -; }
kubectl get pods -n istio-system   # istiod, istio-cni-node, ztunnel
```

- 공식 ambient Getting Started는 Gateway API **experimental** 채널 CRD(`experimental-install.yaml`)를 설치한다. 이 예제의 리소스(Gateway, HTTPRoute)는 standard 채널 CRD로 충분하다고 판단했지만, experimental 전용 필드가 필요한 기능을 쓰면 experimental 채널을 설치해야 한다(**확인 필요**).

### 2. 앱 배포 + 메시 편입 (재시작 없음)

```bash
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f samples/bookinfo/platform/kube/bookinfo-versions.yaml
kubectl apply -f samples/curl/curl.yaml
kubectl apply -f samples/bookinfo/gateway-api/bookinfo-gateway.yaml
kubectl annotate gateway bookinfo-gateway networking.istio.io/service-type=ClusterIP --namespace=default

kubectl label namespace default istio.io/dataplane-mode=ambient
kubectl get pods   # READY 1/1 그대로 — sidecar 없음
istioctl ztunnel-config workloads   # PROTOCOL 열이 HBONE이면 메시 편입됨
```

### 3. L4 인가 (ztunnel)

```yaml
# examples/ambient/01-authz-l4-productpage.yaml
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: productpage-ztunnel
spec:
  selector:
    matchLabels: { app: productpage }
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/bookinfo-gateway-istio"]
```

- L4 정책에는 `from.source.principals`, 포트 같은 L4 조건만 쓴다. `methods`, `paths` 같은 L7 조건을 넣으면 ztunnel이 집행할 수 없다.

### 4. waypoint 배포 + L7 기능

```bash
istioctl waypoint apply -n default --enroll-namespace --wait
# 또는: kubectl apply -f examples/ambient/02-waypoint.yaml && kubectl label ns default istio.io/use-waypoint=waypoint
```

```yaml
# examples/ambient/03-authz-l7-productpage.yaml — selector 대신 targetRefs
apiVersion: security.istio.io/v1
kind: AuthorizationPolicy
metadata:
  name: productpage-waypoint
spec:
  targetRefs:
  - kind: Service
    group: ""
    name: productpage
  action: ALLOW
  rules:
  - from:
    - source:
        principals: ["cluster.local/ns/default/sa/curl"]
    to:
    - operation:
        methods: ["GET"]
```

```yaml
# examples/ambient/04-httproute-reviews-canary.yaml — 메시 내부 카나리(Service를 parentRef로)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: reviews
spec:
  parentRefs:
  - group: ""
    kind: Service
    name: reviews
    port: 9080
  rules:
  - backendRefs:
    - { name: reviews-v1, port: 9080, weight: 90 }
    - { name: reviews-v2, port: 9080, weight: 10 }
```

```bash
kubectl apply -f examples/ambient/03-authz-l7-productpage.yaml
kubectl apply -f examples/ambient/04-httproute-reviews-canary.yaml
kubectl exec deploy/curl -- curl -s -o /dev/null -w "%{http_code}\n" -X DELETE http://productpage:9080/productpage   # 403
```

- 이때 L4 정책(`productpage-ztunnel`)도 유지돼 있다면 ztunnel 입장에서 productpage로 들어오는 출처는 **waypoint**다. 따라서 waypoint 서비스 어카운트(`cluster.local/ns/default/sa/waypoint`)도 L4 정책에서 허용해야 한다. 공식 튜토리얼도 이 순서로 L4 정책을 갱신한다([Enforce authorization policies](https://istio.io/latest/docs/ambient/getting-started/enforce-auth-policies/)).

## 관측성

```bash
kubectl apply -f samples/addons/prometheus.yaml
kubectl apply -f samples/addons/kiali.yaml
# (선택) samples/addons/grafana.yaml, samples/addons/jaeger.yaml
istioctl dashboard kiali
```

| 도구 | 보는 것 |
| --- | --- |
| Prometheus | `istio_requests_total`, `istio_request_duration_milliseconds` 등 표준 메트릭 |
| Kiali | 서비스 그래프, mTLS 여부, 설정 검증 |
| Grafana | Istio 대시보드 |
| Jaeger | 분산 트레이스. 애플리케이션이 trace 헤더를 전파해야 이어진다 |

- `samples/addons`는 **데모용**이다. 운영에서는 별도로 설치·튜닝한 Prometheus 등을 연동한다([Integrations](https://istio.io/latest/docs/ops/integrations/)).
- Ambient에서 HTTP 메트릭과 트레이스는 waypoint가 있어야 나온다. ztunnel만 있으면 TCP 메트릭만 나온다.

## 디버깅 명령 모음

| 명령 | 용도 |
| --- | --- |
| `istioctl analyze` | 클러스터 설정 정적 분석(오타, 없는 host 참조 등) |
| `istioctl analyze --use-kube=false examples/sidecar/*.yaml` | 클러스터 없이 로컬 YAML 분석 |
| `istioctl proxy-status` | 각 Envoy의 xDS 동기화 상태 |
| `istioctl proxy-config routes\|clusters\|listeners <pod>` | 특정 sidecar·waypoint에 실제로 내려간 설정 |
| `istioctl ztunnel-config workloads` | ztunnel이 아는 워크로드와 프로토콜(HBONE/TCP) |
| `istioctl ztunnel-config certificates <ztunnel-pod>.istio-system` | ztunnel이 보유한 워크로드 인증서 |
| `istioctl x precheck` | 설치·업그레이드 전 사전 점검 |

- 출처: [istioctl analyze](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl-analyze/), [Troubleshoot ztunnel](https://istio.io/latest/docs/ambient/usage/troubleshoot-ztunnel/), [istioctl reference](https://istio.io/latest/docs/reference/commands/istioctl/)

## 정리

```bash
kind delete cluster --name istio-toy
```

## 출처

- https://istio.io/latest/docs/setup/getting-started/
- https://istio.io/latest/docs/ambient/getting-started/
- https://istio.io/latest/docs/ambient/getting-started/secure-and-visualize/
- https://istio.io/latest/docs/ambient/getting-started/enforce-auth-policies/
- https://istio.io/latest/docs/ambient/getting-started/manage-traffic/
- https://istio.io/latest/docs/ambient/usage/waypoint/
- https://istio.io/latest/docs/concepts/traffic-management/
- https://istio.io/latest/docs/reference/config/networking/destination-rule/
- https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/
- https://istio.io/latest/docs/setup/additional-setup/config-profiles/
- https://github.com/istio/istio/tree/release-1.31/samples

---
*조사 기준일: 2026-10-08. Istio 1.31.1, Gateway API v1.6.0 기준. 명령은 공식 Getting Started를 바탕으로 하며, 실행 검증 결과는 README의 검증 메모를 참고.*
