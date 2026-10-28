# CLAUDE.md — aws-ses

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: Amazon SES(Simple Email Service)를 실무 관점에서 정리. **트랜잭셔널 발송(가입 인증, 비밀번호 재설정, 알림 메일) 중심**이고, 마케팅 대량 발송과 수신(inbound)은 개요 수준
- 중점: identity·인증(DKIM·SPF·DMARC) / 발송 방식·이벤트 / 샌드박스·한도·평판·바운스 처리 / 비용·IP·운영 이슈
- 관련 주제: 알람·대시보드는 [`aws-cloudwatch`](../aws-cloudwatch/README.md), IAM 정책·조건 키는 [`aws-iam`](../aws-iam/README.md), Terraform 문법은 [`terraform`](../terraform/README.md)을 링크하고 반복 설명하지 않는다.
- 예제 스택: 문서 안의 AWS CLI(`aws sesv2`) + Terraform HCL(hashicorp/aws) + DNS 레코드 예시 코드블록. 별도 `examples/` 없음.

## 주제 전용 규칙

- **용어는 처음 나올 때 정의한다.** `01-overview.md`는 처음 보는 사람 기준으로 단계별로 쌓아 올리고 끝에 용어 정리를 둔다. `02` 이후 문서는 요약 바로 아래에 "이 문서에서 새로 나오는 용어" 표를 둔다.
- 1차 출처는 Amazon SES Developer Guide, SES API v2 레퍼런스, AWS CLI `sesv2` 레퍼런스, 요금 페이지를 우선한다. 메일 인증 표준은 RFC, 메일함 사업자 요구사항은 Google·Yahoo 공식 발송자 가이드를 출처로 쓴다.
- **API는 v2(`sesv2`)를 기준**으로 쓰고, v1(`ses`)과 다른 점이 있으면 밝힌다.
- **한도·평판 임계치·요금은 자주 바뀐다.** 조사 기준일과 출처를 남기고, 금액은 us-east-1과 서울 리전을 구분한다.
- 메일 주소·도메인은 `example.com`, 계정 ID는 `111122223333` 같은 예시 값만 쓴다.
- 실제로 메일을 보내는 명령에는 샌드박스·평판 영향(바운스가 쌓이면 계정이 정지될 수 있음)을 함께 적는다.
- 커밋 scope: `docs(aws-ses): ...`
