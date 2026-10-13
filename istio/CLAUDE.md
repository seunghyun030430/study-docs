# CLAUDE.md — istio

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: Istio 서비스 메시를 입문자 관점에서 정리
- 범위: 개념 / 아키텍처 / 데이터 플레인 모드(Sidecar vs Ambient) / 사용법 / 이슈·한계 / 내부 동작
- 기준 버전: Istio 1.31.x, Kubernetes Gateway API v1.6.0 (조사 기준일 2026-10-08)
- 예제: `examples/` — kind 클러스터 + `istioctl` + YAML 매니페스트

## 주제 전용 규칙

- 1차 출처는 istio.io 공식 문서·GitHub·Envoy 문서를 우선한다.
- 용어는 원문 그대로 쓴다(예: sidecar, ztunnel, waypoint, xDS).
- **버전 명시**: Istio는 릴리스마다 기능 상태(Alpha/Beta/Stable)가 바뀌므로, 버전에 따라 달라지는 주장에는 기준 버전을 적는다.
- 코드블록은 언어를 명시한다(`yaml`, `bash`).
- 예제 YAML을 바꾸면 해당 문서(특히 `04-usage.md`)의 스니펫과 일치하는지 함께 확인한다.
- `istioctl` 다운로드 디렉토리(`examples/istio-*/`)는 `.gitignore`로 제외한다.
- 커밋 scope: `docs(istio): ...`
