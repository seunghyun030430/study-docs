# CLAUDE.md — aws-devops-agent

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: AWS의 AI 기반 운영 도구 — **AWS DevOps Agent**(중심), **CloudWatch Investigations**, **Amazon DevOps Guru**(지원 종료 예정, 이전 관점)
- 중점: 개념과 서비스 지도 / DevOps Agent 기능·설정·보안 / Investigations·Guru와 이전 / 비용·한도·리스크
- 관련 주제: 메트릭·알람·로그는 [`aws-cloudwatch`](../aws-cloudwatch/README.md), IAM은 [`aws-iam`](../aws-iam/README.md), 계정·SSO는 [`aws-organizations`](../aws-organizations/README.md), runbook은 [`aws-systems-manager`](../aws-systems-manager/README.md)를 링크한다.
- 예제 스택: 문서 안의 AWS CLI(`aws devops-agent`, `aws devops-guru`, `aws cloudwatch`) + IAM 정책 JSON 코드블록. 별도 `examples/` 없음.

## 주제 전용 규칙

- **용어는 처음 나올 때 정의한다.** `01-overview.md`는 처음 보는 사람 기준으로 단계별로 쌓아 올리고 끝에 용어 정리를 둔다. `02` 이후 문서는 요약 바로 아래에 "이 문서에서 새로 나오는 용어" 표를 둔다.
- **변화가 매우 빠른 영역이다.** 출시·지원 종료 날짜, 지원 리전, 요금, 기능 상태(미리 보기/정식)는 반드시 날짜와 출처를 함께 적고, 문서 맨 아래 조사 기준일을 꼭 남긴다.
- 서비스 이름은 공식 이름을 쓴다(AWS 문서 안에서도 "AWS DevOps Agent"와 "Amazon DevOps Agent"가 섞여 있으면 그 사실을 적는다).
- AI가 리소스를 바꾸는 기능(directed action, runbook 실행)에는 권한·승인·감사 관점의 주의를 함께 적는다.
- 커밋 scope: `docs(aws-devops-agent): ...`
