# CLAUDE.md — aws-kms

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: AWS KMS(Key Management Service)를 실무 관점에서 정리
- 중점: 암호화 원리·키 종류 / 권한(키 정책·IAM·grant) / 서비스 연동·운영(교체·삭제·감사) / 비용·한도·이슈
- 관련 주제: IAM 정책 평가는 [`aws-iam`](../aws-iam/README.md), SCP·RCP는 [`aws-organizations`](../aws-organizations/README.md), 서비스별 암호화는 [`aws-ebs`](../aws-ebs/README.md)·[`aws-cloudwatch`](../aws-cloudwatch/README.md)·[`aws-systems-manager`](../aws-systems-manager/README.md)를 링크하고 반복 설명하지 않는다.
- 예제 스택: 문서 안의 AWS CLI(`aws kms`) + 키 정책·IAM 정책 JSON 코드블록. 별도 `examples/` 없음.

## 주제 전용 규칙

- **용어는 처음 나올 때 정의한다.** `01-overview.md`는 처음 보는 사람 기준으로 단계별로 쌓아 올리고 끝에 용어 정리를 둔다. `02` 이후 문서는 요약 바로 아래에 "이 문서에서 새로 나오는 용어" 표를 둔다.
- 1차 출처는 AWS KMS Developer Guide, KMS API Reference, AWS CLI 레퍼런스, 요금 페이지, KMS Cryptographic Details를 우선한다.
- **키 삭제·키 정책 변경은 되돌릴 수 없는 사고로 이어질 수 있다.** 해당 명령·정책 예시에는 경고와 안전한 대안(비활성화 먼저, lockout safety check)을 함께 적는다.
- 정책 JSON은 ` ```json ` 코드펜스, 계정 ID는 `111122223333`(키 소유)·`444455556666`(다른 계정), 키 ID는 `1234abcd-12ab-34cd-56ef-1234567890ab` 같은 예시 값을 쓴다.
- 커밋 scope: `docs(aws-kms): ...`
