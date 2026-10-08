# Istio 개요

> Istio는 애플리케이션 코드를 고치지 않고 서비스 간 통신에 **트래픽 제어·mTLS 보안·관측성**을 덧입히는 오픈소스 서비스 메시다. 2023-07-12에 CNCF Graduated 프로젝트가 되었고, 2026-10-08 기준 최신 릴리스는 1.31.x다.

## 개요 / 정의

### 서비스 메시(Service Mesh)란

마이크로서비스가 늘어나면 서비스 간 네트워크 호출에 재시도, 타임아웃, 암호화, 인증, 메트릭 수집 같은 공통 관심사가 생긴다. 이것을 서비스마다 라이브러리로 구현하지 않고 **인프라 계층(프록시)으로 빼낸 것**이 서비스 메시다.

> "an infrastructure layer that gives applications capabilities like zero-trust security" — *[istio.io, What is a service mesh?](https://istio.io/latest/about/service-mesh/)*

### Istio란

> "Istio is an open source service mesh that layers transparently onto existing distributed applications." — *[istio.io, What is Istio?](https://istio.io/latest/docs/overview/what-is-istio/)*

- 핵심 아이디어: 서비스 앞뒤에 프록시(Envoy 또는 ztunnel)를 두고, 모든 트래픽이 프록시를 지나게 한다. 컨트롤 플레인(`istiod`)이 프록시 설정을 중앙에서 내려준다.
- 대부분 Kubernetes 위에서 쓴다. Sidecar 모드는 VM도 지원한다([Data plane modes](https://istio.io/latest/docs/overview/dataplane-modes/)).

## 핵심 기능

공식 문서의 기능 목록을 네 범주로 묶었다(범주 이름은 정리 편의를 위한 것).

| 범주 | 제공 기능 | 대표 리소스(CRD) |
| --- | --- | --- |
| 트래픽 관리 | 라우팅 규칙, 재시도, 페일오버, fault injection, HTTP/gRPC/WebSocket/TCP 부하 분산, 카나리(가중치 분할) | `VirtualService`, `DestinationRule`, `Gateway`, Kubernetes Gateway API `HTTPRoute` |
| 보안 | 자동 mTLS, 워크로드 ID 기반 인증·인가 | `PeerAuthentication`, `RequestAuthentication`, `AuthorizationPolicy` |
| 관측성 | 클러스터 내 모든 트래픽의 메트릭·로그·트레이스 자동 수집 | `Telemetry` (+ Prometheus, Grafana, Kiali, Jaeger 등 애드온) |
| 확장성 | 접근 제어·rate limit 등 플러그형 정책 계층, WebAssembly 플러그인 | `WasmPlugin`, `EnvoyFilter`(sidecar 전용) |

> "Automatic metrics, logs, and traces for all traffic within a cluster" — *[istio.io, What is Istio?](https://istio.io/latest/docs/overview/what-is-istio/)*

## 왜 쓰는가 / 언제 쓰지 않는가

| 쓰면 좋은 경우 | 굳이 쓰지 않아도 되는 경우 |
| --- | --- |
| 서비스 수가 많고 언어가 섞여 있어 공통 네트워크 기능을 라이브러리로 맞추기 어렵다 | 서비스가 몇 개뿐이고 단일 언어 프레임워크(예: Spring Cloud)로 이미 해결된다 |
| 서비스 간 mTLS·zero-trust 인가가 필요하다(규제·보안 요구) | 클러스터 경계(Ingress)에서의 TLS로 충분하다 |
| 카나리·트래픽 미러링·fault injection으로 배포 위험을 줄이고 싶다 | 운영 인력이 메시 자체를 운영·업그레이드할 여력이 없다 |
| 코드 수정 없이 서비스 간 골든 시그널(지연·트래픽·에러)을 보고 싶다 | 프록시가 추가하는 지연(서브 밀리초~밀리초)조차 허용할 수 없다 |

판단 근거와 운영 부담은 [05-issues.md](./05-issues.md)에 정리했다.

## 역사

| 시점 | 사건 | 출처 |
| --- | --- | --- |
| 2017-05-24 | Istio 0.1 공개. Google, IBM, Lyft가 공개했고 Lyft의 Envoy 프록시를 기반으로 함 | [Istio 0.x releases](https://istio.io/latest/news/releases/0.x/) |
| 2018-07-31 | Istio 1.0 발표(production ready) | [HPCwire 2018-07-31](https://www.hpcwire.com/aiwire/2018/07/31/istio-service-mesh-advances-to-production/) (2차 출처, 공식 발표문은 확인 필요) |
| 2020-03 | Istio 1.5에서 Pilot·Galley·Citadel·sidecar injector를 단일 바이너리 `istiod`로 통합 | [istio.io blog, 2020-03-19](https://istio.io/latest/blog/2020/istiod/) |
| 2022-04 | CNCF Incubating 합류 | [CNCF 발표](https://www.cncf.io/announcements/2023/07/12/cloud-native-computing-foundation-reaffirms-istio-maturity-with-project-graduation/) |
| 2023-07-12 | CNCF Graduated | [istio.io blog, 2023-07-12](https://istio.io/latest/blog/2023/istio-graduates-within-cncf/) |
| 2024-11-07 | Istio 1.24에서 Ambient 모드(ztunnel, waypoint, API) Stable(GA) | [istio.io blog, 2024-11-07](https://istio.io/latest/blog/2024/ambient-reaches-ga/) |
| 2026-08 | Istio 1.31.0 릴리스 | [Announcing 1.31](https://istio.io/latest/news/releases/1.31.x/announcing-1.31/) |

> 참고: 공식 "What is a service mesh" 페이지는 Istio를 "Founded by Google, IBM and Lyft in 2016"이라고 적는다. 내부 개발 시작은 2016년, 공개(0.1)는 2017-05-24로 보는 것이 맞다.

## 지원 버전 (2026-10-08 기준)

| Istio | 릴리스 | 예상 EOL | 지원 Kubernetes |
| --- | --- | --- | --- |
| 1.31 | 2026-08 (아래 메모 참고) | ~2027-02 | 1.32 – 1.37 |
| 1.30 | 2026-05-14 | ~2026-12 | 1.32 – 1.36 |
| 1.29 | 2026-02-16 | 2026-10-12 | 1.31 – 1.35 |

- 출처: [Supported Releases](https://istio.io/latest/docs/releases/supported-releases/)
- 메모: 1.31의 릴리스일이 공식 지원 표에는 2026-08-27로, 발표문에는 2026-08-31로 적혀 있어 서로 다르다. 1.31의 지원 Kubernetes 범위도 지원 표(1.32–1.37)와 발표문(1.32–1.36)이 다르다. **둘 다 확인 필요**.
- Istio는 대략 분기마다 마이너 버전을 내고, 마이너 버전 하나를 약 6개월 지원한다(위 표의 릴리스일과 EOL 간격에서 추정).

## 대안 메시 (간단 비교)

| 항목 | Istio | Linkerd | Cilium Service Mesh |
| --- | --- | --- | --- |
| 데이터 플레인 | Envoy (sidecar/waypoint) + ztunnel (ambient, Rust) | 자체 Rust 프록시 `linkerd2-proxy` (sidecar) | eBPF + 노드 단위 Envoy |
| CNCF 상태 | Graduated | Graduated | Cilium 프로젝트 Graduated |
| 특징 | 기능 범위가 가장 넓음, 모드 두 가지 | 단순함·경량을 지향 | CNI와 메시를 통합 |

- 출처: [linkerd.io](https://linkerd.io/), [cilium.io/use-cases/service-mesh](https://cilium.io/use-cases/service-mesh/), [CNCF projects](https://www.cncf.io/projects/)
- 성능 우열은 측정 환경에 따라 결론이 달라지므로 이 문서에서는 판단하지 않는다(**확인 필요**).

## 출처

- https://istio.io/latest/docs/overview/what-is-istio/
- https://istio.io/latest/about/service-mesh/
- https://istio.io/latest/docs/overview/dataplane-modes/
- https://istio.io/latest/docs/releases/supported-releases/
- https://istio.io/latest/news/releases/1.31.x/announcing-1.31/
- https://istio.io/latest/news/releases/0.x/
- https://istio.io/latest/blog/2020/istiod/
- https://istio.io/latest/blog/2023/istio-graduates-within-cncf/
- https://istio.io/latest/blog/2024/ambient-reaches-ga/

---
*조사 기준일: 2026-10-08. 버전·지원 기간은 공식 지원 표 기준. 1.0 릴리스일은 2차 출처 기반.*
