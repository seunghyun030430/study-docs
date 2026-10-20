# CLAUDE.md — aws-prometheus-grafana

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: Amazon Managed Service for Prometheus(AMP)와 Amazon Managed Grafana(AMG)로 만드는 관측 파이프라인
- 중점: Prometheus 모델 기초 / AMP(저장·조회·알림) / 수집 방식(관리형 수집기·ADOT·Prometheus agent) / AMG(인증·데이터 소스·대시보드·알림) / 비용·이슈
- 관련 주제: CloudWatch는 [`aws-cloudwatch`](../aws-cloudwatch/README.md), IAM(SigV4·IRSA·Pod Identity)은 [`aws-iam`](../aws-iam/README.md), IAM Identity Center는 [`aws-organizations`](../aws-organizations/README.md), Istio 메트릭은 [`istio`](../istio/README.md)를 링크한다.
- 예제 스택: 문서 안의 AWS CLI(`aws amp`, `aws grafana`) + Prometheus·ADOT 설정 YAML + PromQL 코드블록. 별도 `examples/` 없음.

## 주제 전용 규칙

- **용어는 처음 나올 때 정의한다.** `01-overview.md`는 처음 보는 사람 기준으로 단계별로 쌓아 올리고 끝에 용어 정리를 둔다. `02` 이후 문서는 요약 바로 아래에 "이 문서에서 새로 나오는 용어" 표를 둔다.
- 1차 출처는 AMP·AMG User Guide, API·CLI 레퍼런스, 요금 페이지, prometheus.io·grafana.com 공식 문서를 우선한다.
- 서비스 이름은 처음에 정식 이름을 쓰고, 이후 AMP·AMG로 줄인다.
- PromQL은 코드펜스 언어를 `promql`로 쓴다.
- 커밋 scope: `docs(aws-prometheus-grafana): ...`
