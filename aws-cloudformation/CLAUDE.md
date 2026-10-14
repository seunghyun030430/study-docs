# CLAUDE.md — aws-cloudformation

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: AWS CloudFormation을 실무 관점에서 정리
- 중점: 템플릿 문법 / 스택 운영(change set·rollback·drift·보호) / StackSets·멀티 계정 / CDK·Terraform 비교 / 검증 도구·흔한 실패·한도
- 관련 주제: 계정·OU는 [`aws-organizations`](../aws-organizations/README.md), IAM은 [`aws-iam`](../aws-iam/README.md), Quick Setup의 StackSets 사용은 [`aws-systems-manager/06`](../aws-systems-manager/06-quick-setup.md)를 링크한다.
- 예제 스택: 문서 안의 **YAML 템플릿** + AWS CLI(`aws cloudformation`) 코드블록. 별도 `examples/` 없음. CDK·Terraform은 비교용 짧은 코드만.

## 주제 전용 규칙

- **용어는 처음 나올 때 정의한다.** `01-overview.md`는 처음 보는 사람 기준으로 단계별로 쌓아 올리고 끝에 용어 정리를 둔다. `02` 이후 문서는 요약 바로 아래에 "이 문서에서 새로 나오는 용어" 표를 둔다.
- 1차 출처는 CloudFormation User Guide, Template Reference, AWS CLI 레퍼런스, CDK Developer Guide, HashiCorp 공식 문서를 우선한다.
- 템플릿은 ` ```yaml ` 코드펜스로 쓰고, 가능하면 `cfn-lint`로 문법을 검사한다. 검사하지 못한 템플릿은 "확인 필요"로 표시한다.
- 리소스를 지우거나 교체하는 동작(stack 삭제, Replacement)에는 데이터 손실 경고를 함께 적는다.
- 커밋 scope: `docs(aws-cloudformation): ...`
