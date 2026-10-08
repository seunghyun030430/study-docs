# Istio 정리

Istio 서비스 메시를 입문자 관점에서 정리한 문서 모음이다. 개념, 아키텍처, Sidecar vs Ambient, 사용법, 이슈를 다룬다. 기준 버전은 **Istio 1.31.x**, Gateway API **v1.6.0**이고, 조사 기준일은 **2026-10-08**이다.

## 한눈에 보기

| 항목 | 내용 |
| --- | --- |
| 정체 | 코드 수정 없이 서비스 간 통신에 트래픽 제어·mTLS·관측성을 더하는 오픈소스 서비스 메시 |
| 기원 / 거버넌스 | Google·IBM·Lyft가 2017-05-24 공개, CNCF Graduated(2023-07-12) |
| 컨트롤 플레인 | `istiod` — 서비스 디스커버리, xDS 설정 배포, CA(인증서 발급), sidecar injection |
| 데이터 플레인 | Sidecar 모드: Pod마다 Envoy / Ambient 모드: 노드별 ztunnel(L4) + 선택적 waypoint(Envoy, L7) |
| 주요 API | 트래픽: `VirtualService`, `DestinationRule`, Gateway API `Gateway`/`HTTPRoute` · 보안: `PeerAuthentication`, `AuthorizationPolicy` · 관측: `Telemetry` |
| 최신 지원 버전 | 1.31(2026-08), 1.30, 1.29(EOL 2026-10-12 예정). 마이너 버전당 약 6개월 지원 |
| 가장 큰 비용 | 프록시 리소스, 반기 단위 업그레이드, 정책 기본 동작에서 오는 함정 |

### Sidecar vs Ambient 요약

| 항목 | Sidecar | Ambient |
| --- | --- | --- |
| 프록시 | Pod마다 Envoy | 노드별 ztunnel + 필요 시 waypoint |
| 기본 기능 | L4 + L7 | L4(mTLS, L4 인가, TCP 메트릭) |
| 편입 | 라벨 + Pod 재시작 | 라벨만 |
| 리소스 | 큼 | 작음 |
| 성숙도 | Stable(멀티클러스터·VM 포함) | 단일 클러스터 Stable(1.24~), multi-network 멀티클러스터 Beta |

## 문서 인덱스

| 문서 | 다루는 것 |
| --- | --- |
| [01-overview.md](./01-overview.md) | 개요: 서비스 메시와 Istio 정의, 핵심 기능, 역사, 지원 버전, 대안 메시 |
| [02-architecture.md](./02-architecture.md) | 아키텍처: istiod, Envoy/ztunnel, xDS, 트래픽 가로채기, ID·인증서, 설정 API |
| [03-dataplane-modes.md](./03-dataplane-modes.md) | Sidecar vs Ambient: ztunnel, HBONE, waypoint, 모드 선택 기준 |
| [04-usage.md](./04-usage.md) | 사용법: kind 설치, 두 모드 실습, 카나리, mTLS·인가, 관측성, 디버깅 명령 |
| [05-issues.md](./05-issues.md) | 이슈·한계: 운영 비용, 업그레이드, 트래픽·보안 함정, 앱 요구사항, 안티패턴 |
| [examples/](./examples/README.md) | 실습 YAML(sidecar/, ambient/)과 실행 순서 |

## 출처 (1차)

- Istio 공식 문서: https://istio.io/latest/docs/
- 데이터 플레인 모드 비교: https://istio.io/latest/docs/overview/dataplane-modes/
- 지원 릴리스: https://istio.io/latest/docs/releases/supported-releases/
- Istio 1.31 발표: https://istio.io/latest/news/releases/1.31.x/announcing-1.31/
- Istio GitHub(샘플 포함): https://github.com/istio/istio
- Envoy xDS 프로토콜: https://www.envoyproxy.io/docs/envoy/latest/api-docs/xds_protocol
- Kubernetes Gateway API: https://gateway-api.sigs.k8s.io/

---
*이 문서는 학습·정리 목적의 요약이다. 정확한 동작과 옵션은 항상 위 공식 출처를 확인할 것.*
