# Terraform 정리

Terraform을 실무 관점에서 정리한 문서 모음. 중점은 **HCL 문법·실행 흐름 / State 관리 / 모듈·프로젝트 구조 / CI/CD·테스트·운영 이슈**이고, 예제는 AWS provider 기준이다. 조사 기준일 **2026-10-21**(Terraform 1.16.6, AWS provider 6.69.1).

관련 주제: CloudFormation·CDK와 비교는 [aws-cloudformation/05](../aws-cloudformation/05-cdk-terraform.md), 배포 role·OIDC는 [aws-iam](../aws-iam/README.md), 계정 분리는 [aws-organizations](../aws-organizations/README.md).

## 한눈에 보기

| 질문 | 짧은 답 | 문서 |
| --- | --- | --- |
| Terraform이 뭔가? | 원하는 인프라를 HCL로 선언하면 provider가 API를 불러 맞춰 주는 IaC 도구. 여러 클라우드·SaaS를 한 도구로 | [01](./01-overview.md) |
| 기본 흐름은? | `init` → `plan`(변경 미리 보기) → `apply`. 운영은 **저장한 plan을 그대로 apply** | [01](./01-overview.md), [03](./03-workflow-graph.md) |
| 여러 개 만들 때 `count`와 `for_each`? | 목록이면 **`for_each`**. `count`는 중간 항목을 빼면 인덱스가 밀려 엉뚱한 리소스가 교체됨 | [02](./02-hcl.md) |
| 입력값 검사는? | variable `validation`, `precondition`/`postcondition`, `check` 블록 | [02](./02-hcl.md) |
| plan에서 무엇을 보나? | **`-/+`(교체)와 destroy 수**. `# forces replacement` 속성 확인 | [03](./03-workflow-graph.md) |
| State는 어디에? | **S3 backend + `use_lockfile`(1.11+) + 버전 관리 + 암호화**. Git 금지(비밀 평문) | [04](./04-state.md) |
| 이름 변경·가져오기·관리 해제는? | `moved`(1.1+) / `import`(1.5+) / `removed`(1.7+) 블록. CLI `state mv`·`rm`보다 안전 | [04](./04-state.md) |
| 비밀을 state에 안 남기려면? | ephemeral(1.10+), write-only `*_wo` 인자(1.11+), 서비스가 관리하는 비밀 | [04](./04-state.md) |
| 디렉터리는 어떻게? | 재사용 `modules/` + 계정·환경별 `live/` 디렉터리, **디렉터리 하나 = state 하나** | [05](./05-modules-structure.md) |
| 엉뚱한 계정 배포를 막으려면? | provider에 `allowed_account_ids`, `assume_role`, 환경별 배포 role | [05](./05-modules-structure.md) |
| CI는? | PR에서 plan, 머지 후 apply. **GitHub OIDC**, plan(읽기)·apply(쓰기) role 분리 | [06](./06-cicd-testing-issues.md) |
| 테스트·검사는? | `terraform test`(1.6+, mock 1.7+), tflint, checkov 또는 trivy(tfsec 후속) | [06](./06-cicd-testing-issues.md) |
| OpenTofu와 차이는? | MPL 2.0 포크(CNCF Sandbox). state 암호화 등 1.6 이후 기능이 갈라짐 | [01](./01-overview.md), [06](./06-cicd-testing-issues.md) |

| 기능 | 도입 버전 |
| --- | --- |
| `moved` 블록 | 1.1 |
| `import`·`check` 블록 | 1.5 |
| `terraform test`, BSL 라이선스 | 1.6 |
| `removed` 블록, test mock, import `for_each` | 1.7 |
| provider 함수 | 1.8 |
| ephemeral 값·리소스 | 1.10 |
| S3 `use_lockfile` 정식, write-only 인자 | 1.11 |
| variable·output `deprecated`, 모듈 source·version에 변수 | 1.15 |
| 모듈 안 `import` 블록 | 1.16 |

## 문서 인덱스

| 문서 | 다루는 것 |
| --- | --- |
| [01-overview.md](./01-overview.md) | 개요(단계별 입문) — 선언형 IaC, 블록, provider, state, init·plan·apply, 모듈, 라이선스·OpenTofu·HCP Terraform, CloudFormation 비교, 버전별 기능, 용어 정리 |
| [02-hcl.md](./02-hcl.md) | 블록 종류, terraform 블록, variable·locals·data·output, 표현식·함수, count vs for_each, dynamic 블록, lifecycle, validation·precondition·check, depends_on |
| [03-workflow-graph.md](./03-workflow-graph.md) | init과 lock 파일, plan 읽는 법, 저장한 plan apply, 의존성 그래프·병렬성, drift(`-refresh-only`·`-detailed-exitcode`), `-target`·`-replace`, 그 밖의 명령 |
| [04-state.md](./04-state.md) | state 내용과 비밀, S3 backend·잠금·`force-unlock`, workspace 주의, `terraform_remote_state` 대안, import·moved·removed 블록, state CLI, 복구, ephemeral·write-only |
| [05-modules-structure.md](./05-modules-structure.md) | 모듈 source·버전 고정, 좋은 모듈의 조건, AWS provider(`assume_role`·`allowed_account_ids`·`default_tags`·6.0 `region`), provider 넘기기, 환경·계정 분리 디렉터리 |
| [06-cicd-testing-issues.md](./06-cicd-testing-issues.md) | GitHub Actions OIDC plan/apply 워크플로, drift 감지 작업, `terraform test`·mock, tflint·checkov·trivy, OpenTofu 비교, 흔한 사고와 안티패턴 |

## 예제 검증

- 01·02·04·05의 HCL은 Terraform 1.16.6에서 `terraform init -backend=false` + `terraform validate`로 통과했다(2026-10-21). 문서에 없는 변수나 참조 대상은 검증할 때만 보조 정의를 붙였고, 02의 "설정 생략" 부분에는 필수 인자를 채웠다.
- 05의 Git 모듈 예시(`github.com/acme/tf-modules`)는 실제로 없는 저장소라 검증에서 뺐다.
- 06의 `terraform test` 예시는 mock provider로 실행해 통과했다. `.tflint.hcl`은 실행하지 않았다.
- 06의 GitHub Actions YAML은 문법(YAML 파싱)만 확인했고 실제 저장소에서 실행하지 않았다.
- `terraform fmt`는 일부 줄 끝 주석의 간격만 다르게 정렬한다(읽기 쉽게 그대로 둠).

## 출처 (1차)

- Terraform 문서: https://developer.hashicorp.com/terraform/docs
- Terraform 언어: https://developer.hashicorp.com/terraform/language
- Terraform CLI: https://developer.hashicorp.com/terraform/cli
- Terraform CHANGELOG: https://github.com/hashicorp/terraform/blob/main/CHANGELOG.md
- AWS provider: https://registry.terraform.io/providers/hashicorp/aws/latest/docs
- OpenTofu: https://opentofu.org/docs/

---
*이 문서는 학습·정리 목적의 요약이다. 버전·기능·요금은 자주 바뀌므로 적용 전에 공식 문서를 다시 확인할 것. `terraform destroy`, `-replace`, `state rm`, `force-unlock`은 리소스나 state를 되돌릴 수 없게 바꿀 수 있으므로 plan과 백업으로 먼저 확인할 것.*
