# CLAUDE.md — terraform

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: HashiCorp Terraform을 실무 관점에서 정리(예시는 AWS provider 중심)
- 중점: HCL 문법 / 실행 흐름(init·plan·apply·그래프) / State 관리 / 모듈·프로젝트 구조 / CI/CD·테스트·운영 이슈
- 관련 주제: CloudFormation·CDK와의 비교는 [`aws-cloudformation/05`](../aws-cloudformation/05-cdk-terraform.md), IAM·OIDC는 [`aws-iam`](../aws-iam/README.md), 멀티 계정은 [`aws-organizations`](../aws-organizations/README.md)를 링크한다.
- 예제 스택: 문서 안의 HCL + CLI 코드블록. 별도 `examples/` 없음.

## 주제 전용 규칙

- **용어는 처음 나올 때 정의한다.** `01-overview.md`는 처음 보는 사람 기준으로 단계별로 쌓아 올리고 끝에 용어 정리를 둔다. `02` 이후 문서는 요약 바로 아래에 "이 문서에서 새로 나오는 용어" 표를 둔다.
- 1차 출처는 developer.hashicorp.com Terraform 문서, Terraform Registry(provider 문서), HashiCorp GitHub CHANGELOG를 우선한다.
- **기능마다 도입 버전을 적는다**(예: `import` 블록은 1.5+). 버전에 따라 동작이 다르기 때문이다.
- HCL은 ` ```hcl ` 코드펜스로 쓰고, 가능하면 `terraform validate`로 검사한다. 로컬 Terraform 버전보다 새 기능을 쓰는 예시는 검사하지 못했다고 적는다.
- `destroy`, `state rm`, `-replace`처럼 리소스·state를 지우거나 바꾸는 명령에는 경고를 함께 적는다.
- 커밋 scope: `docs(terraform): ...`
