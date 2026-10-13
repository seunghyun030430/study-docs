# Istio 예제 (kind + istioctl 1.31.1)

> Bookinfo 샘플 위에서 Sidecar 모드와 Ambient 모드의 트래픽 관리·보안 정책을 각각 적용해 보는 최소 YAML 모음. 설명은 [../04-usage.md](../04-usage.md)에 있다.

## 구성

| 파일 | 모드 | 내용 |
| --- | --- | --- |
| `sidecar/01-peer-authentication-strict.yaml` | Sidecar | default 네임스페이스 STRICT mTLS |
| `sidecar/02-destination-rule-reviews.yaml` | Sidecar | reviews subset(v1–v3), LEAST_REQUEST, outlier detection |
| `sidecar/03-virtual-service-reviews-canary.yaml` | Sidecar | reviews v1 90% / v2 10%, timeout, retries |
| `sidecar/04-authz-productpage.yaml` | Sidecar | productpage: ingress gateway의 GET만 허용 |
| `ambient/01-authz-l4-productpage.yaml` | Ambient | ztunnel이 집행하는 L4 인가 |
| `ambient/02-waypoint.yaml` | Ambient | default 네임스페이스 waypoint(`istio-waypoint` GatewayClass) |
| `ambient/03-authz-l7-productpage.yaml` | Ambient | waypoint가 집행하는 L7 인가(`targetRefs`) |
| `ambient/04-httproute-reviews-canary.yaml` | Ambient | HTTPRoute(Service parentRef)로 reviews 카나리 |

## 준비 (공통)

필요 도구: Docker, kind, kubectl.

```bash
kind create cluster --name istio-toy
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.31.1 sh -
export ISTIO_DIR=$PWD/istio-1.31.1
export PATH=$ISTIO_DIR/bin:$PATH
kubectl get crd gateways.gateway.networking.k8s.io &> /dev/null || \
  { kubectl kustomize "github.com/kubernetes-sigs/gateway-api/config/crd?ref=v1.6.0" | kubectl apply -f -; }
```

명령은 이 `examples/` 디렉토리에서 실행한다고 가정한다(`istio-1.31.1/`은 `.gitignore` 처리됨).

## Sidecar 모드

```bash
istioctl install -f $ISTIO_DIR/samples/bookinfo/demo-profile-no-gateways.yaml -y
kubectl label namespace default istio-injection=enabled
kubectl apply -f $ISTIO_DIR/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f $ISTIO_DIR/samples/curl/curl.yaml
kubectl apply -f $ISTIO_DIR/samples/bookinfo/gateway-api/bookinfo-gateway.yaml
kubectl annotate gateway bookinfo-gateway networking.istio.io/service-type=ClusterIP -n default
kubectl wait --for=condition=available deploy --all -n default --timeout=300s

kubectl apply -f sidecar/02-destination-rule-reviews.yaml
sleep 5   # make-before-break: DR 먼저, VS 나중
kubectl apply -f sidecar/03-virtual-service-reviews-canary.yaml
sleep 5
# 페이지에 Pod 이름이 두 번 나오므로 요청당 첫 매치만 센다
kubectl exec deploy/curl -- sh -c 'for i in $(seq 1 200); do curl -s http://productpage:9080/productpage | grep -o "reviews-v[0-9]" | head -n1; done' | sort | uniq -c

kubectl apply -f sidecar/01-peer-authentication-strict.yaml
kubectl apply -f sidecar/04-authz-productpage.yaml
sleep 10  # 정책이 프록시에 전파될 때까지 대기(이벤추얼 컨시스턴시)
kubectl exec deploy/curl -- curl -s -o /dev/null -w "%{http_code}\n" http://productpage:9080/productpage   # 403 기대
```

## Ambient 모드 (새 클러스터에서)

```bash
istioctl install --set profile=ambient --skip-confirmation
kubectl apply -f $ISTIO_DIR/samples/bookinfo/platform/kube/bookinfo.yaml
kubectl apply -f $ISTIO_DIR/samples/bookinfo/platform/kube/bookinfo-versions.yaml
kubectl apply -f $ISTIO_DIR/samples/curl/curl.yaml
kubectl apply -f $ISTIO_DIR/samples/bookinfo/gateway-api/bookinfo-gateway.yaml
kubectl annotate gateway bookinfo-gateway networking.istio.io/service-type=ClusterIP -n default
kubectl label namespace default istio.io/dataplane-mode=ambient
istioctl ztunnel-config workloads

kubectl apply -f ambient/01-authz-l4-productpage.yaml
sleep 10
kubectl exec deploy/curl -- curl -s -o /dev/null -w "%{http_code}\n" http://productpage:9080/productpage   # 000, exit 56 (L4에서 연결 리셋)
kubectl exec deploy/curl -- curl -s -o /dev/null -w "%{http_code}\n" http://bookinfo-gateway-istio/productpage   # 200 (gateway SA는 허용)

kubectl apply -f ambient/02-waypoint.yaml
kubectl wait --for=condition=programmed gtw/waypoint --timeout=180s
kubectl label namespace default istio.io/use-waypoint=waypoint
# waypoint를 거치면 productpage에 도달하는 출처가 waypoint가 되므로 L4 정책에 waypoint를 추가한다
# (추가하지 않으면 curl -> productpage 가 503)
kubectl patch authorizationpolicy productpage-ztunnel --type=json \
  -p='[{"op":"add","path":"/spec/rules/0/from/0/source/principals/-","value":"cluster.local/ns/default/sa/waypoint"}]'
kubectl apply -f ambient/03-authz-l7-productpage.yaml
kubectl apply -f ambient/04-httproute-reviews-canary.yaml
sleep 10

kubectl exec deploy/curl -- curl -s -o /dev/null -w "%{http_code}\n" http://productpage:9080/productpage             # 200
kubectl exec deploy/curl -- curl -s -o /dev/null -w "%{http_code}\n" -X DELETE http://productpage:9080/productpage   # 403
kubectl exec deploy/curl -- sh -c 'for i in $(seq 1 200); do curl -s http://productpage:9080/productpage | grep -o "reviews-v[0-9]" | head -n1; done' | sort | uniq -c
# 주의: ingress gateway에서 들어온 트래픽은 기본적으로 waypoint를 거치지 않는다.
# 그래서 gateway 경유 GET은 L7 정책(curl SA만 허용)과 무관하게 200이다.
```

## 정적 검증 (클러스터 없이)

```bash
istioctl analyze --use-kube=false sidecar/*.yaml
```

## 검증 기록 (2026-10-08)

- 환경: kind v0.33.0(노드 Kubernetes v1.37.0), Docker 27.4.0, istioctl 1.31.1, Gateway API v1.6.0
- 정적 검증: `istioctl validate -f sidecar/ -f ambient/`와 Bookinfo 샘플을 포함한 `istioctl analyze --use-kube=false` 모두 통과

| 모드 | 확인 항목 | 결과 |
| --- | --- | --- |
| Sidecar | 주입 | 앱 Pod READY 2/2. `istio-proxy`가 `restartPolicy: Always`인 initContainer(native sidecar)로 주입됨 |
| Sidecar | VS 적용 전 분산 | v1/v2/v3에 고르게 분산 |
| Sidecar | 카나리 90:10 | 200회 중 v1 179 / v2 21 |
| Sidecar | Envoy 라우트 | `proxy-config routes`에서 timeout 3s, 재시도 2회, `5xx,connect-failure,reset`, 가중치 90/10 확인 |
| Sidecar | STRICT mTLS | 메시 밖 Pod(평문) → details: 연결 실패(exit 56) |
| Sidecar | 인가 | curl → productpage 403, gateway GET 200, gateway POST 403. 적용 직후 약 5초 동안은 200(전파 지연) |
| Ambient | 편입 | 라벨만 붙였을 때 재시작 없이 READY 1/1, `ztunnel-config workloads`의 PROTOCOL이 HBONE |
| Ambient | L4 인가 | curl → productpage 연결 리셋(000, exit 56), gateway 경유 200 |
| Ambient | waypoint + L4 정책 | L4 정책에 waypoint SA가 없으면 503, 추가하면 200 |
| Ambient | L7 인가 | curl GET 200, DELETE 403 |
| Ambient | HTTPRoute 카나리 | 200회 중 v1 179 / v2 21 |
| Ambient | ingress → waypoint | gateway 경유 요청은 waypoint를 우회해 L7 정책이 적용되지 않음(200). 공식 문서상 기본 동작이다([Configure waypoint proxies](https://istio.io/latest/docs/ambient/usage/waypoint/)) |

- `istioctl analyze`(클러스터 모드)는 `networking.istio.io/service-type` 어노테이션이 자동 생성된 Deployment·Service·Pod로 전파된 것에 대해 IST0107(Misplaced annotation) 경고를 낸다. 공식 Getting Started 절차에서도 똑같이 생기는 경고이고, 동작에는 영향이 없었다.

## 정리

```bash
kind delete cluster --name istio-toy
```
