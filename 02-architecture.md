# Istio 아키텍처

> Istio는 **컨트롤 플레인 `istiod`** 와 **데이터 플레인 프록시**(Envoy, ambient에서는 ztunnel)로 나뉜다. 사용자가 YAML(CRD)로 의도를 선언하면 istiod가 이를 xDS 설정으로 바꿔 프록시에 내려준다. 또 워크로드마다 X.509 인증서를 발급해 mTLS의 기반을 만든다.

## 전체 구조

```
           kubectl apply (VirtualService, AuthorizationPolicy, HTTPRoute ...)
                                   │
                                   ▼
┌────────────────────────── Control plane ──────────────────────────┐
│ istiod                                                            │
│  • 서비스 디스커버리(Kubernetes Service/Endpoint 감시)            │
│  • 설정 변환: Istio/Gateway API 리소스 → xDS                      │
│  • CA: 워크로드 인증서 서명(CSR 처리)                             │
│  • sidecar injection webhook                                      │
└──────────────┬──────────────────────────────┬─────────────────────┘
               │ xDS (gRPC)                   │ 인증서 서명
               ▼                              ▼
┌────────────────────────── Data plane ─────────────────────────────┐
│  [Pod A: app + Envoy] ──mTLS──▶ [Pod B: app + Envoy]   (sidecar)  │
│  [Pod A] → ztunnel(node) ──HBONE/mTLS──▶ ztunnel(node) → [Pod B]   │
│                      (필요 시 waypoint(Envoy) 경유)       (ambient)│
└───────────────────────────────────────────────────────────────────┘
```

> "The data plane is composed of a set of intelligent proxies (Envoy) deployed as sidecars." / "The control plane manages and configures the proxies to route traffic." — *[istio.io, Architecture](https://istio.io/latest/docs/ops/deployment/architecture/)*

## 컨트롤 플레인: istiod

> "Istiod provides service discovery, configuration and certificate management." — *[istio.io, Architecture](https://istio.io/latest/docs/ops/deployment/architecture/)*

| 역할 | 내용 |
| --- | --- |
| 서비스 디스커버리 | Kubernetes API를 감시해 Service·Endpoint로 내부 서비스 레지스트리를 만든다. 외부 서비스는 `ServiceEntry`로 추가한다 |
| 설정 배포 | `VirtualService`, `DestinationRule`, `AuthorizationPolicy`, Gateway API `HTTPRoute` 같은 리소스를 Envoy가 이해하는 xDS로 변환해 push한다 |
| 인증서 관리(CA) | 워크로드의 CSR을 검증·서명해 X.509 인증서를 발급한다 |
| Sidecar injection | mutating admission webhook으로 Pod에 `istio-proxy` 컨테이너를 주입한다(sidecar 모드) |

- 1.5 이전에는 Pilot(설정), Citadel(CA), Galley(설정 검증), sidecar injector가 별도 컴포넌트였다. 1.5에서 단일 바이너리 `istiod`로 통합했다([istio.io blog, 2020-03-19](https://istio.io/latest/blog/2020/istiod/)). 오래된 자료에 나오는 "Pilot", "Citadel"은 지금은 istiod의 내부 기능 이름으로 이해하면 된다.

## 데이터 플레인: Envoy / ztunnel

| 프록시 | 쓰이는 곳 | 계층 | 비고 |
| --- | --- | --- | --- |
| Envoy | sidecar, waypoint, ingress/egress gateway | L4–L7 | C++, Lyft에서 시작한 CNCF Graduated 프로젝트 |
| ztunnel | ambient 모드의 노드별 프록시 | L4 | Rust, mTLS·L4 인가·TCP 메트릭 담당 |

공식 문서에 나오는 Envoy의 주요 기능은 다음과 같다([Architecture](https://istio.io/latest/docs/ops/deployment/architecture/)).
- Dynamic service discovery, Load balancing, TLS termination, HTTP/2 and gRPC proxies
- Circuit breakers, Health checks, Staged rollouts with %-based traffic split, Fault injection, Rich metrics

두 모드의 상세 비교는 [03-dataplane-modes.md](./03-dataplane-modes.md)에 있다.

### xDS: 설정이 프록시에 전달되는 방식

Envoy는 정적 설정 파일 대신 **xDS API**(gRPC 스트림)로 설정을 동적으로 받는다. istiod가 xDS 서버 역할을 한다.

| API | 내려주는 것 | Istio 리소스와의 대응(대략) |
| --- | --- | --- |
| LDS (Listener) | 어떤 포트·프로토콜을 받을지 | `Gateway`, `Sidecar`, 포트 정보 |
| RDS (Route) | HTTP 라우팅 규칙 | `VirtualService`, `HTTPRoute` |
| CDS (Cluster) | 업스트림 클러스터와 LB·커넥션 정책 | `DestinationRule` |
| EDS (Endpoint) | 클러스터의 실제 Pod IP 목록 | Kubernetes Endpoint(Slice) |
| SDS (Secret) | TLS 인증서·키 | istio-agent가 istiod에서 받은 워크로드 인증서 |

- 출처: [Envoy xDS protocol](https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol)
- "대응" 열은 이해를 돕기 위한 근사치다. 실제로는 하나의 Istio 리소스가 여러 xDS 리소스에 영향을 준다.
- 디버깅: `istioctl proxy-status`로 각 프록시의 xDS 동기화 상태를 보고, `istioctl proxy-config routes|clusters|listeners <pod>`로 실제로 내려간 설정을 본다.

### 트래픽 가로채기 (sidecar 모드)

애플리케이션은 평소처럼 `http://reviews:9080`을 호출하지만, Pod 네트워크 네임스페이스의 iptables 규칙이 이 트래픽을 Envoy로 돌린다.

| 방식 | 동작 | 권한 |
| --- | --- | --- |
| `istio-init` init container (기본) | Pod 시작 시 iptables 리다이렉션 설정 | 각 워크로드에 `NET_ADMIN`, `NET_RAW` 필요 |
| Istio CNI node agent | 노드별 DaemonSet이 chained CNI 플러그인으로 Pod 네트워크 설정 시점에 리다이렉션 설정 | 워크로드에 특권 init container 불필요 |

> "The Istio CNI node agent is **required** in the ambient data plane mode." / "For the sidecar data plane mode, the Istio CNI node agent is optional." — *[istio.io, Install Istio with the Istio CNI node agent](https://istio.io/latest/docs/setup/additional-setup/cni/)*

- Istio CNI는 클러스터의 기본 CNI(Calico, Cilium 등)를 **대체하지 않고** 그 위에 체이닝된다.

## 보안 아키텍처: ID와 인증서

### 워크로드 ID

- Kubernetes에서는 **ServiceAccount**가 워크로드 ID다.
- 인증서에 담기는 SPIFFE ID 형식은 `spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>`이다(기본 trust domain은 `cluster.local`).

> "All registrations must follow the Istio SPIFFE ID pattern: `spiffe://<trust.domain>/ns/<namespace>/sa/<service-account>`" — *[istio.io, SPIRE integration](https://istio.io/latest/docs/ops/integrations/spire/)*

- `AuthorizationPolicy`의 `principals`에는 `spiffe://`를 뺀 `cluster.local/ns/default/sa/curl` 형태로 적는다.

### 인증서 발급 흐름

1. 워크로드가 시작되면 istio-agent(`pilot-agent`)가 개인키와 CSR을 만든다.
2. istio-agent가 자격 증명(ServiceAccount 토큰)과 함께 CSR을 istiod로 보낸다.
3. istiod CA가 자격 증명을 검증하고 CSR에 서명한다.
4. Envoy가 SDS API로 로컬 istio-agent에 인증서를 요청하고, agent가 인증서와 키를 전달한다.
5. istio-agent는 인증서 만료를 감시하다가 주기적으로 위 과정을 반복해 교체(rotation)한다.

- 출처: [istio.io, Security concepts](https://istio.io/latest/docs/concepts/security/)
- 기본 인증서 수명은 24시간이라는 설명이 많지만 공식 개념 문서에는 명시가 없다(**확인 필요**).
- Ambient 모드에서는 ztunnel이 자기 노드에 있는 워크로드의 인증서를 대신 받아 관리한다([03-dataplane-modes.md](./03-dataplane-modes.md)).

### mTLS 모드와 인가 평가 순서

| `PeerAuthentication` 모드 | 동작 |
| --- | --- |
| `PERMISSIVE` (기본) | mTLS와 평문을 모두 받는다. 마이그레이션용 |
| `STRICT` | mTLS만 받는다 |
| `DISABLE` | mTLS를 끈다(별도 보안 수단이 없으면 비권장) |

`AuthorizationPolicy`는 `CUSTOM` → `DENY` → `ALLOW` 순으로 평가한다.
- `DENY`가 `ALLOW`보다 우선한다.
- 워크로드에 `ALLOW` 정책이 하나도 없으면 모두 허용된다. 하나라도 생기면 그 정책에 맞지 않는 요청은 거부된다.

> "Istio checks for matching policies in layers, in this order: `CUSTOM`, `DENY`, and then `ALLOW`." — *[istio.io, Security concepts](https://istio.io/latest/docs/concepts/security/)*

## 설정 API: Istio API vs Kubernetes Gateway API

| 용도 | Istio 고유 API | Kubernetes Gateway API |
| --- | --- | --- |
| 진입점(Ingress) | `networking.istio.io` `Gateway` + 별도 설치한 gateway Deployment | `gateway.networking.k8s.io` `Gateway` (`gatewayClassName: istio`) → Deployment·Service 자동 생성 |
| HTTP 라우팅 | `VirtualService` | `HTTPRoute` |
| 백엔드 정책 | `DestinationRule` | (해당 없음 — Istio에서는 `DestinationRule` 병용) |
| Ambient waypoint | — | `Gateway` (`gatewayClassName: istio-waypoint`) |

> "intends to make it the default API for traffic management in the future" — *[istio.io, Kubernetes Gateway API task](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/)*

- Ambient 모드의 waypoint는 Gateway API 리소스로 배포한다. 따라서 새로 시작한다면 Gateway API를 익혀 두는 편이 유리하다.

## 출처

- https://istio.io/latest/docs/ops/deployment/architecture/
- https://istio.io/latest/docs/concepts/security/
- https://istio.io/latest/docs/concepts/traffic-management/
- https://istio.io/latest/docs/setup/additional-setup/cni/
- https://istio.io/latest/docs/ops/integrations/spire/
- https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/
- https://istio.io/latest/blog/2020/istiod/
- https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol

---
*조사 기준일: 2026-10-08. Istio 1.31 문서 기준. xDS ↔ Istio 리소스 대응표는 이해용 근사치.*
