# Istio 이슈·한계·안티패턴

> Istio의 비용은 기능이 아니라 **운영**에서 나온다. 프록시 리소스, 업그레이드 주기(약 6개월 지원), 이벤추얼 컨시스턴시 때문에 생기는 503, 포트·프로토콜 규칙, 정책 기본 동작(PERMISSIVE, allow-by-default) 같은 함정을 알고 들어가야 한다. 많은 함정이 "설정했는데 적용이 안 됨" 형태로 나타나므로 `istioctl analyze`와 `proxy-config`를 습관적으로 쓰는 것이 가장 큰 예방책이다.

## 운영·비용

| 이슈 | 내용 | 대응 |
| --- | --- | --- |
| 프록시 리소스 | sidecar는 Pod마다 Envoy가 붙어 CPU·메모리가 Pod 수에 비례해 늘어난다 | `Sidecar` 리소스로 설정 범위 축소, 또는 ambient 검토([03](./03-dataplane-modes.md)) |
| 설정 크기 | 기본적으로 모든 프록시가 메시 전체 서비스 설정을 받는다. 메시가 크면 메모리와 xDS push 비용이 커진다 | `Sidecar` 리소스의 `egress.hosts`, `exportTo`로 가시성 제한 |
| 지연 | sidecar는 홉당 서브 밀리초~1ms 수준의 지연이 붙고, L7 처리는 L4보다 비싸다 | L7이 필요 없는 곳은 ambient의 ztunnel만 사용 |
| 업그레이드 주기 | 마이너 버전 하나의 지원 기간이 약 6개월이라 연 2회 이상 업그레이드가 사실상 필수다 | revision 기반 canary 업그레이드를 표준 절차로 정착 |
| 학습 곡선 | CRD 종류가 많고 Istio API와 Gateway API가 공존해 같은 일을 두 방식으로 할 수 있다 | 신규 구성은 Gateway API 우선. 팀 내 표준을 하나로 정한다 |

- 지원 기간: [Supported Releases](https://istio.io/latest/docs/releases/supported-releases/)

### 업그레이드

- 전략은 세 가지다: **canary(revision) 업그레이드**, in-place 업그레이드, Helm 업그레이드([Upgrade](https://istio.io/latest/docs/setup/upgrade/)).
- 한 번에 마이너 버전 두 개를 넘게 건너뛰는 업그레이드는 "not officially tested or recommended"다. 예: 1.29 → 1.31은 가능, 1.28 → 1.31은 비권장.
- Sidecar 모드에서는 컨트롤 플레인을 올린 뒤 **워크로드를 재시작해야** 새 버전 sidecar가 주입된다. 재시작하지 않으면 구버전 프록시가 계속 남는다.
- 1.31부터 `gcr.io/istio-release`, `registry.istio.io`, `istio-release.storage.googleapis.com`에 아티팩트를 더 이상 게시하지 않는다. 이미지는 Docker Hub, Helm 차트 등은 `blob.istio.io`를 쓴다. 2026년 중 해당 경로를 일시적으로 끄는 "scream test" 일정이 공지되었다(2026-09-15, 10-13, 11-17, 12-08~09). 기존 경로를 쓰는 CI·미러 설정을 점검해야 한다([Announcing 1.31](https://istio.io/latest/news/releases/1.31.x/announcing-1.31/)).

## 트래픽 관리의 함정

| 함정 | 증상 | 대응 | 출처 |
| --- | --- | --- | --- |
| DestinationRule·VirtualService 동시 변경 | subset이 아직 없는 상태에서 라우팅돼 일시적인 503 | make-before-break: subset 추가 시 DR 먼저 → 몇 초 뒤 VS, 삭제 시 반대 순서 | [Traffic mgmt best practices](https://istio.io/latest/docs/ops/best-practices/traffic-management/) |
| 기본 라우트 없음 | 새 버전 Pod가 뜨자마자 트래픽을 받음 | 처음부터 서비스마다 기본 VS 라우트를 둔다 | 같은 문서 |
| DR 위치 | 다른 네임스페이스에 둔 DR이 적용되지 않음 | DR은 서비스와 같은 네임스페이스에 둔다(조회 경로: 클라이언트 ns → 서비스 ns → root ns) | 같은 문서 |
| 리소스 기본 공개 범위 | 트래픽 관리 리소스가 기본적으로 모든 네임스페이스에 export됨 | `exportTo`로 범위 제한 | 같은 문서 |
| 큰 VS 분할 | 여러 VS로 나눴더니 규칙 순서가 의도와 다름 | 리소스 간 규칙 순서는 정의되지 않음. catch-all 규칙은 하나만 둔다 | 같은 문서 |
| 타임아웃 기본값 | HTTP 타임아웃이 기본 비활성이라 느린 업스트림에 무한정 대기 | VS에 `timeout` 명시 | [Traffic management](https://istio.io/latest/docs/concepts/traffic-management/) |
| 재시도 폭증 | 기본 2회 재시도가 장애 시 부하를 증폭 | 비멱등 요청은 `retryOn`과 횟수를 조정 | 같은 문서 |
| fault injection 병용 | 같은 VS에서 fault injection과 retry·timeout을 함께 쓸 수 없음 | 테스트용 VS를 분리 | 같은 문서 |
| Pod IP 직접 호출 | Service를 거치지 않으면 mTLS·라우팅·텔레메트리가 빠짐 | 항상 Service 이름으로 호출 | [Application requirements](https://istio.io/latest/docs/ops/deployment/application-requirements/) |

## 애플리케이션 요구사항

- **포트 이름·프로토콜**: Service 포트 이름(`http-`, `grpc-`, `tcp-` 접두사)이나 `appProtocol`로 프로토콜을 명시한다. 명시하지 않으면 자동 감지에 의존하게 된다([Protocol Selection](https://istio.io/latest/docs/ops/configuration/traffic-management/protocol-selection/)).
- **server-first 프로토콜**(서버가 먼저 바이트를 보내는 MySQL 등): PERMISSIVE mTLS와 자동 프로토콜 감지가 첫 바이트를 기다리므로 충돌한다. 25, 53, 3306, 27017 포트는 자동으로 TCP로 간주된다. 그 밖의 포트는 TCP로 명시해야 한다.
- **예약 포트**: Envoy가 15000, 15001, 15002, 15004, 15006, 15008, 15020, 15021, 15053, 15090을 쓰므로 애플리케이션은 이 포트를 쓰면 안 된다.
- **UID 1337**: sidecar 프록시가 쓰는 UID이므로 앱 컨테이너를 UID 1337로 실행하지 않는다.
- **Pod Security**: `istio-init`은 `NET_ADMIN`, `NET_RAW`가 필요하다. `baseline`·`restricted` Pod Security 수준은 이를 막는다. 이 경우 Istio CNI를 쓴다.
- **라벨**: 텔레메트리를 위해 `app`/`version`(또는 `app.kubernetes.io/name`/`version`) 라벨을 붙인다.

> "Ensure your pods do **not** run applications as a user with the user ID (UID) value of `1337`" — *[istio.io, Application Requirements](https://istio.io/latest/docs/ops/deployment/application-requirements/)*

### Sidecar 시작·종료 순서

- 과거에는 앱이 Envoy보다 먼저 떠서 시작 직후 네트워크 호출이 실패하거나, Job이 끝나도 sidecar가 남아 Pod가 종료되지 않는 문제가 흔했다. 대응책으로 `holdApplicationUntilProxyStarts` 옵션이 쓰였다.
- Istio 1.27부터 `ENABLE_NATIVE_SIDECARS`가 기본 `true`다. 자격이 되는 Pod에는 Kubernetes **native sidecar**(`restartPolicy: Always`인 initContainer)로 주입된다([1.27 change notes](https://istio.io/latest/news/releases/1.27.x/announcing-1.27/change-notes/), [Native sidecars blog](https://istio.io/latest/blog/2023/native-sidecars/)). native sidecar는 프록시가 준비된 뒤 앱 컨테이너를 시작하고, 프록시가 실행 중이어도 Pod 종료를 막지 않는다. 그래서 위 두 문제가 크게 줄었다.
- 워크로드별 opt-out 어노테이션 이름이 문서마다 다르게 적혀 있다(change notes: `sidecar.istio.io/native-side`, annotation 레퍼런스: `sidecar.istio.io/nativeSidecar`). **확인 필요**: [Resource Annotations](https://istio.io/latest/docs/reference/config/annotations/).

## 보안 정책의 함정

| 함정 | 설명 |
| --- | --- |
| PERMISSIVE가 기본 | 설치만 해도 mTLS가 "가능"해질 뿐, 평문도 계속 받는다. zero-trust가 목표라면 `PeerAuthentication` STRICT를 명시해야 한다 |
| ALLOW 정책 하나가 기본 동작을 뒤집음 | ALLOW 정책이 하나도 없으면 전부 허용되지만, 하나라도 생기면 매칭되지 않는 요청은 전부 거부된다. 헬스체크·모니터링 경로가 막히는 사고가 흔하다 |
| DENY 우선 | 평가 순서는 `CUSTOM` → `DENY` → `ALLOW`. 넓은 DENY 하나가 모든 ALLOW를 무력화한다 |
| Ambient L4/L7 혼동 | ztunnel은 HTTP를 파싱하지 않는다. `selector` 기반 정책에 `methods`·`paths` 같은 L7 조건을 넣으면 ztunnel이 집행할 수 없다. L7 정책은 waypoint에 `targetRefs`로 붙인다 |
| Ambient waypoint 경유 시 출처 변경 | waypoint를 거치면 목적지 ztunnel이 보는 출처가 waypoint ID가 된다. 기존 L4 정책에 waypoint 서비스 어카운트를 추가하지 않으면 트래픽이 막힌다([Enforce authorization policies](https://istio.io/latest/docs/ambient/getting-started/enforce-auth-policies/)) |
| egress 기본 허용 | 기본 outbound 정책(`ALLOW_ANY`)에서는 메시 밖으로 나가는 트래픽을 막지 않는다. 통제가 필요하면 `REGISTRY_ONLY`와 `ServiceEntry`, egress gateway를 조합한다([Accessing External Services](https://istio.io/latest/docs/tasks/traffic-management/egress/egress-control/)) |

- 출처: [Security concepts](https://istio.io/latest/docs/concepts/security/)

## 관측성의 한계

- 분산 트레이싱은 프록시만으로 완성되지 않는다. 애플리케이션이 받은 요청의 trace 헤더(`traceparent`, `x-b3-*` 등)를 다음 요청에 **직접 전파**해야 span이 이어진다([Distributed Tracing FAQ](https://istio.io/latest/about/faq/distributed-tracing/)).
- `samples/addons`의 Prometheus·Grafana·Kiali·Jaeger는 데모용이다(영속성·보안·스케일 미고려).
- 메트릭 레이블이 많으면 Prometheus 카디널리티가 폭증할 수 있다. `Telemetry` API로 불필요한 메트릭과 레이블을 줄인다(**확인 필요**: [Telemetry API](https://istio.io/latest/docs/tasks/observability/telemetry/)).

## 안티패턴

| 안티패턴 | 왜 나쁜가 | 대안 |
| --- | --- | --- |
| `EnvoyFilter` 남용 | Envoy 내부 구조에 직접 의존해 업그레이드 때 쉽게 깨진다. ambient에서는 지원되지 않는다 | 가능한 한 정식 API(`Telemetry`, `WasmPlugin`, `AuthorizationPolicy` CUSTOM 등) 사용 |
| 운영에 `demo` 프로필 | 시연용 설정 | `default` 또는 `ambient` + 필요 옵션을 명시적으로 설정 |
| 메시 전체 일괄 적용 | 문제 발생 시 영향 범위가 메시 전체가 된다 | 네임스페이스 단위로 점진 편입. PERMISSIVE → 관측 → STRICT 순 |
| 모든 서비스에 waypoint (ambient) | L7 비용을 불필요하게 지불한다 | L7이 필요한 네임스페이스·서비스에만 배포 |
| 업그레이드 미루기 | 지원 기간(약 6개월)을 넘기면 CVE 패치를 받지 못하고, 건너뛸 수 있는 버전 폭(2개)을 넘으면 단계적 업그레이드가 필요해진다 | 분기마다 업그레이드 리허설 |
| 설정 적용 후 검증 생략 | 오타나 존재하지 않는 host 참조도 적용 자체는 성공한다 | CI에서 `istioctl analyze`, 적용 후 `istioctl proxy-config`로 확인 |

## 출처

- https://istio.io/latest/docs/ops/deployment/application-requirements/
- https://istio.io/latest/docs/ops/best-practices/traffic-management/
- https://istio.io/latest/docs/concepts/traffic-management/
- https://istio.io/latest/docs/concepts/security/
- https://istio.io/latest/docs/setup/upgrade/
- https://istio.io/latest/docs/releases/supported-releases/
- https://istio.io/latest/news/releases/1.31.x/announcing-1.31/
- https://istio.io/latest/news/releases/1.27.x/announcing-1.27/change-notes/
- https://istio.io/latest/blog/2023/native-sidecars/
- https://istio.io/latest/docs/ambient/getting-started/enforce-auth-policies/
- https://istio.io/latest/docs/overview/dataplane-modes/

---
*조사 기준일: 2026-10-08. Istio 1.31 기준. "확인 필요" 표시 항목은 공식 문서 간 표기가 다르거나 세부를 직접 검증하지 못한 것.*
