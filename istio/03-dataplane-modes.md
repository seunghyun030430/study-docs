# 데이터 플레인 모드: Sidecar vs Ambient

> **Sidecar**는 Pod마다 Envoy를 붙여 모든 Pod에 L7 기능을 준다. 오래 검증된 방식이지만 리소스와 운영 비용이 크다. **Ambient**는 노드마다 하나씩 도는 L4 프록시 `ztunnel`로 mTLS를 기본 제공하고, L7 기능이 필요한 곳에만 `waypoint`(Envoy)를 붙인다. 1.24(2024-11)부터 Stable이다. 새 클러스터라면 ambient로 L4 zero-trust부터 시작하고, L7은 필요한 서비스에만 더하는 접근이 공식 권장 방향에 가깝다.

## 한눈 비교

| 항목 | Sidecar | Ambient |
| --- | --- | --- |
| 프록시 배치 | Pod마다 Envoy 컨테이너(`istio-proxy`) | 노드마다 ztunnel(DaemonSet) + 필요 시 waypoint(Deployment) |
| 기본 제공 기능 | L4 + L7 전부 | L4: mTLS, L4 인가, TCP 메트릭 |
| L7 기능(HTTP 라우팅·재시도·L7 인가·요청 메트릭) | 기본 제공 | waypoint를 배포해야 가능 |
| 메시 편입 방법 | 네임스페이스 라벨 `istio-injection=enabled` + **Pod 재시작** | 네임스페이스 라벨 `istio.io/dataplane-mode=ambient` (재시작 불필요) |
| 리소스 비용 | 큼(Pod 수에 비례) | 작음(노드 수 + waypoint 수에 비례) |
| L7 처리 홉 | 2회(출발지·목적지 sidecar) | 1회(목적지 waypoint) |
| 프록시 수명주기 | 앱 Pod와 함께(앱 개발자 영역) | 앱과 독립(플랫폼 관리자 영역) |
| Kubernetes Job | sidecar가 종료되지 않아 번거로움 | 투명하게 동작 |
| server-first 프로토콜(예: MySQL) | 추가 설정 필요 | 지원 |
| `EnvoyFilter` | 지원 | 미지원(waypoint의 Wasm 플러그인은 지원) |
| 키 보관 | 워크로드마다 자기 키 보유. 앱 Pod가 탈취되면 키도 노출 | 노드 ztunnel이 그 노드 워크로드의 키만 보유. 앱 Pod가 탈취돼도 메시 키에 접근 불가 |
| 플랫폼 | Kubernetes, VM | Kubernetes만 |
| 성숙도 | Stable(멀티클러스터 포함) | 단일 클러스터 Stable. multi-network 멀티클러스터는 1.29부터 Beta |
| Istio CNI node agent | 선택 | **필수** |

- 출처: [istio.io, Data plane modes](https://istio.io/latest/docs/overview/dataplane-modes/), [Ambient reaches GA (2024-11-07)](https://istio.io/latest/blog/2024/ambient-reaches-ga/), [Ambient multi-network multicluster Beta](https://istio.io/latest/blog/2026/ambient-multinetwork-multicluster-beta/), [Feature Status](https://istio.io/latest/docs/releases/feature-stages/)

### 지연 시간 (공식 문서 수치)

| 경로 | p90/p99 추가 지연 |
| --- | --- |
| Sidecar | 약 0.63–0.88 ms |
| Ambient (ztunnel만) | 약 0.16–0.20 ms |
| Ambient + waypoint | 약 0.40–0.50 ms |

- 출처: [Data plane modes](https://istio.io/latest/docs/overview/dataplane-modes/). 측정 조건(부하, 페이로드 크기, 버전)에 따라 달라지므로 절대값보다 상대 비교로 보는 것이 좋다.

> "The overhead for processing protocols at Layer 7 is substantially higher than processing network packets at Layer 4." — *[istio.io, Data plane modes](https://istio.io/latest/docs/overview/dataplane-modes/)*

## Sidecar 모드

```
┌──────── Pod A ────────┐            ┌──────── Pod B ────────┐
│ app ─▶ istio-proxy ───┼── mTLS ───▶┼─ istio-proxy ─▶ app   │
│        (Envoy, L4–L7) │            │ (Envoy, L4–L7)        │
└───────────────────────┘            └───────────────────────┘
```

- 네임스페이스에 `istio-injection=enabled`를 붙이면, 이후 생성되는 Pod에 istiod의 webhook이 `istio-proxy` 컨테이너를 주입한다. **이미 떠 있는 Pod에는 주입되지 않으므로** `kubectl rollout restart`가 필요하다.
- 모든 Pod가 Envoy를 가지므로 출발지 쪽 기능(클라이언트 측 재시도, 로컬 LB)과 목적지 쪽 기능을 모두 쓸 수 있다.
- 메시가 커지면 모든 sidecar가 전체 서비스 설정을 받아 메모리를 많이 쓴다. `Sidecar` 리소스로 범위를 줄이는 튜닝이 사실상 필요하다([Traffic management concepts](https://istio.io/latest/docs/concepts/traffic-management/)).

## Ambient 모드

```
             Node 1                                    Node 2
┌──────────────────────────────┐        ┌──────────────────────────────┐
│ Pod A ─▶ ztunnel ────────────┼─HBONE─▶┼─ ztunnel ─▶ Pod B            │
│          (L4, Rust)          │ :15008 │                              │
└──────────────────────────────┘        └──────────────────────────────┘
                L7이 필요할 때:  ztunnel ─HBONE─▶ waypoint(Envoy) ─HBONE─▶ ztunnel ─▶ Pod B
```

### ztunnel (L4)

> "Ztunnel is responsible for securely connecting and authenticating workloads within the mesh." — *[istio.io, Ambient overview](https://istio.io/latest/docs/ambient/overview/)*

- 노드마다 하나씩 도는 Rust 프록시다. mTLS, 인증, L4 인가, TCP 텔레메트리를 담당한다.
- HTTP를 종료하거나 헤더를 파싱하지 **않는다**. 그래서 HTTP 기반 라우팅이나 인가는 할 수 없다.

### HBONE

- **H**TTP-**B**ased **O**verlay **N**etwork **E**nvironment의 약자. HTTP CONNECT로 터널을 열고, mTLS로 암호화하며, HTTP/2로 여러 연결을 다중화한다.
- 관례상 TCP **15008** 포트를 쓴다([HBONE](https://istio.io/latest/docs/ambient/architecture/hbone/)).

### 트래픽 리다이렉션 (in-pod)

- Pod나 네임스페이스에 ambient 라벨이 붙으면 Istio CNI node agent가 **Pod의 네트워크 네임스페이스 안에** 리다이렉션 규칙을 만든다. 그다음 ztunnel이 그 네임스페이스 안에 리스닝 소켓(15008, 15006, 15001)을 연다([Traffic redirection](https://istio.io/latest/docs/ambient/architecture/traffic-redirection/)).
- 그래서 이미 실행 중인 Pod도 재시작 없이 메시에 편입된다. 기본 CNI가 무엇이든 동작한다.

### waypoint (L7)

> "The waypoint proxy is a deployment of the Envoy proxy." — *[istio.io, Ambient overview](https://istio.io/latest/docs/ambient/overview/)*

- 다음 중 하나라도 필요하면 waypoint를 쓴다([Configure waypoint proxies](https://istio.io/latest/docs/ambient/usage/waypoint/)).
  - 트래픽 관리: HTTP 라우팅, 로드 밸런싱, 재시도, 타임아웃
  - 보안: 요청 메서드·경로·헤더 기반 인가
  - 관측성: HTTP 메트릭, 액세스 로그, 트레이싱
- Gateway API의 `Gateway` 리소스(`gatewayClassName: istio-waypoint`)로 배포한다. 보통 네임스페이스 단위로 하나를 둔다.
- `istio.io/waypoint-for` 라벨로 처리 대상을 정한다.

| 값 | 처리 대상 |
| --- | --- |
| `service` (기본) | Kubernetes Service로 향하는 트래픽 |
| `workload` | Pod/VM IP로 직접 향하는 트래픽 |
| `all` | 둘 다 |
| `none` | 처리하지 않음(테스트용) |

- 네임스페이스, Service, Pod에 `istio.io/use-waypoint=<waypoint 이름>` 라벨을 붙여 waypoint를 쓰게 한다. `istioctl waypoint apply --enroll-namespace`를 쓰면 배포와 라벨링을 한 번에 한다.
- L7 정책은 `selector` 대신 `targetRefs`로 Service나 Gateway(waypoint)를 가리킨다. 예시는 [04-usage.md](./04-usage.md)에 있다.
- 1.31 신기능으로 primary/canary waypoint를 가중치로 나누는 "weighted waypoint canaries"(`istio.io/use-waypoint-canary` 라벨)가 추가되었다([Announcing 1.31](https://istio.io/latest/news/releases/1.31.x/announcing-1.31/)).

> 실제 iptables/nftables 규칙, ztunnel이 Pod netns에 여는 소켓, HBONE 로그, UID 1337 우회 실험은 [06-internals.md](./06-internals.md)에 있다.

## 어떤 모드를 고를까

| 상황 | 추천 |
| --- | --- |
| 신규 도입, 우선 mTLS·L4 zero-trust가 목표 | Ambient (L7은 필요한 네임스페이스에만 waypoint) |
| 기존 sidecar 메시가 안정적으로 운영 중 | 당장 바꿀 필요는 없음. 마이그레이션 가이드를 검토한 뒤 점진 전환 |
| VM 워크로드 포함 | Sidecar (ambient는 VM 미지원) |
| `EnvoyFilter`로 세밀한 Envoy 커스터마이징에 의존 | Sidecar |
| 멀티클러스터(특히 single-network) 운영 | Sidecar가 더 성숙. ambient multi-network는 Beta, single-network는 alpha 수준(**확인 필요**) |
| Job·CronJob이 많음 | Ambient |

> 공식 문서는 zero-trust 보안부터 시작하고 L7 기능은 선택적으로 추가하라고 권한다. ambient는 L7이 필요 없는 곳에서 L7 처리 비용을 아예 내지 않게 해 준다([Data plane modes](https://istio.io/latest/docs/overview/dataplane-modes/)).

- 같은 메시 안에서 sidecar 워크로드와 ambient 워크로드가 공존할 수 있다([Ambient overview](https://istio.io/latest/docs/ambient/overview/)).

## 출처

- https://istio.io/latest/docs/overview/dataplane-modes/
- https://istio.io/latest/docs/ambient/overview/
- https://istio.io/latest/docs/ambient/architecture/hbone/
- https://istio.io/latest/docs/ambient/architecture/traffic-redirection/
- https://istio.io/latest/docs/ambient/usage/waypoint/
- https://istio.io/latest/blog/2024/ambient-reaches-ga/
- https://istio.io/latest/blog/2026/ambient-multinetwork-multicluster-beta/
- https://istio.io/latest/docs/releases/feature-stages/
- https://istio.io/latest/news/releases/1.31.x/announcing-1.31/

---
*조사 기준일: 2026-10-08. Istio 1.31 기준. 멀티클러스터 성숙도는 블로그와 설치 문서의 표현이 달라 보수적으로 기재.*
