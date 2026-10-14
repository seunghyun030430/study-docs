# CLAUDE.md — aws-cloudwatch

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: Amazon CloudWatch를 실무 관점에서 정리
- 중점: Metrics·Alarms / Logs·Logs Insights / 비용·운영 이슈
- 개요 수준만: Application Signals·X-Ray(트레이싱), Synthetics, RUM, Container·Lambda Insights, CloudWatch agent
- 관련 주제: IAM 용어는 [`aws-iam`](../aws-iam/README.md), 여러 계정 구성은 [`aws-organizations`](../aws-organizations/README.md)를 링크하고 반복 설명하지 않는다.
- 예제 스택: 문서 안의 AWS CLI(`aws cloudwatch`, `aws logs`) + Logs Insights 쿼리 + JSON 코드블록. 별도 `examples/` 없음.

## 주제 전용 규칙

- **용어는 처음 나올 때 정의한다.** `01-overview.md`는 처음 보는 사람 기준으로 단계별로 쌓아 올리고 끝에 용어 정리를 둔다. `02` 이후 문서는 요약 바로 아래에 "이 문서에서 새로 나오는 용어" 표를 둔다.
- 1차 출처는 CloudWatch User Guide, CloudWatch Logs User Guide, API·CLI 레퍼런스, 요금 페이지를 우선한다.
- **요금은 리전마다 다르다.** 금액은 기준 리전(기본 us-east-1)을 명시하고, 서울 리전 값이 확인되지 않으면 "서울 리전은 요금 페이지에서 확인"이라고 적는다.
- Logs Insights 쿼리는 코드펜스 언어를 비워 두거나 `sql` 대신 일반 텍스트로 쓴다(전용 문법이므로).
- 커밋 scope: `docs(aws-cloudwatch): ...`
