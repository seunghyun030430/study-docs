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
kubectl exec deploy/curl -- sh -c 'for i in $(seq 1 50); do curl -s http://productpage:9080/productpage | grep -o "reviews-v[0-9]-[a-z0-9]*"; done' | sort | uniq -c

kubectl apply -f sidecar/01-peer-authentication-strict.yaml
kubectl apply -f sidecar/04-authz-productpage.yaml
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
kubectl exec deploy/curl -- curl -s -o /dev/null -w "%{http_code}\n" http://productpage:9080/productpage   # 연결 거부(000 등) 기대

kubectl apply -f ambient/02-waypoint.yaml
kubectl label namespace default istio.io/use-waypoint=waypoint
# waypoint를 거치면 productpage에 도달하는 출처가 waypoint가 되므로 L4 정책에 waypoint를 추가한다
kubectl patch authorizationpolicy productpage-ztunnel --type=json \
  -p='[{"op":"add","path":"/spec/rules/0/from/0/source/principals/-","value":"cluster.local/ns/default/sa/waypoint"}]'
kubectl apply -f ambient/03-authz-l7-productpage.yaml
kubectl apply -f ambient/04-httproute-reviews-canary.yaml

kubectl exec deploy/curl -- curl -s -o /dev/null -w "%{http_code}\n" http://productpage:9080/productpage             # 200
kubectl exec deploy/curl -- curl -s -o /dev/null -w "%{http_code}\n" -X DELETE http://productpage:9080/productpage   # 403
```

## 정적 검증 (클러스터 없이)

```bash
istioctl analyze --use-kube=false sidecar/*.yaml
```

## 정리

```bash
kind delete cluster --name istio-toy
```
