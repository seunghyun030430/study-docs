# CI/CD·테스트·운영 이슈

> 운영 Terraform은 **사람의 PC가 아니라 CI에서** 실행한다. **PR에서 plan을 보여 주고, 머지 후 그 plan을 apply**하는 흐름이다. AWS 자격증명은 **GitHub Actions OIDC**로 받고(액세스 키 없음), plan용 읽기 role과 apply용 쓰기 role을 나눈다. 배포 전에는 `fmt`·`validate`·**tflint**·**checkov/trivy**·**`terraform test`**로 검사한다. 가장 흔한 사고는 비밀이 든 state 노출, 콘솔 수정으로 인한 drift, `count` 인덱스 밀림, 잠금 없는 동시 실행, 거대한 state다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[05](./05-modules-structure.md)의 용어와 [aws-iam/05](../aws-iam/05-workload-identity.md#github-actions-oidc)의 GitHub Actions OIDC를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| CI/CD | 코드 변경마다 자동으로 검사·배포하는 파이프라인 |
| OIDC | CI가 발급한 토큰을 AWS 임시 자격증명으로 바꾸는 방식 |
| 정적 분석 | 코드를 실행하지 않고 오류·보안 문제를 찾는 검사 |
| Mock provider | 실제 API를 부르지 않고 가짜 응답을 주는 테스트용 provider |
| Drift 감지 작업 | 정기적으로 plan을 돌려 밖에서 바뀐 것을 찾는 CI 작업 |

## CI/CD 흐름

```
PR 생성 ──▶ fmt -check · validate · tflint · checkov/trivy · terraform test
        └─▶ plan (읽기 role) ──▶ PR 코멘트·요약에 결과 표시 ──▶ 리뷰·승인
머지    ──▶ plan -out (쓰기 role) ──▶ 환경 승인(prod) ──▶ apply tfplan
매일    ──▶ plan -detailed-exitcode ──▶ 종료 코드 2면 drift 알림
```

PR에서 plan, 머지 후 apply, plan·apply role 분리는 널리 쓰이는 패턴이지만, HashiCorp가 하나의 공식 규칙으로 정한 문서는 찾지 못했다(이 레포의 정리).

### GitHub Actions 예시

출처: [GitHub OIDC in AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws), [configure-aws-credentials](https://github.com/aws-actions/configure-aws-credentials), [setup-terraform](https://github.com/hashicorp/setup-terraform)

```yaml
name: terraform
on:
  pull_request:
    paths: ["live/prod/app/**"]
  push:
    branches: [main]
    paths: ["live/prod/app/**"]

permissions:
  id-token: write      # OIDC 토큰 발급
  contents: read
  pull-requests: write # plan 결과 코멘트

jobs:
  plan:
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: live/prod/app
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::111122223333:role/tf-plan-readonly
          aws-region: ap-northeast-2
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: 1.16.6
      - run: terraform fmt -check -recursive
      - run: terraform init -lockfile=readonly
      - run: terraform validate
      - run: terraform plan -no-color -lock=false -out=tfplan
      - run: terraform show -no-color tfplan >> "$GITHUB_STEP_SUMMARY"

  apply:
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    environment: prod          # GitHub 환경 보호 규칙으로 승인자 지정
    defaults:
      run:
        working-directory: live/prod/app
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::111122223333:role/tf-apply
          aws-region: ap-northeast-2
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: 1.16.6
      - run: terraform init -lockfile=readonly
      - run: terraform plan -out=tfplan
      - run: terraform apply -auto-approve tfplan
```

| 포인트 | 이유 |
| --- | --- |
| `id-token: write` + `configure-aws-credentials` | 액세스 키 없이 임시 자격증명. trust policy의 `sub`를 저장소·브랜치·environment로 제한([aws-iam/05](../aws-iam/05-workload-identity.md#github-actions-oidc)) |
| **plan role은 읽기 전용, apply role은 쓰기** | PR 코드(외부 기여자 포함)가 쓰기 권한을 얻지 못하게 |
| apply role의 `sub`는 `environment:prod`로 | 승인을 거친 job만 apply role을 맡을 수 있음 |
| `-lockfile=readonly` | CI에서 provider 버전이 바뀌지 않게 |
| plan 요약은 `$GITHUB_STEP_SUMMARY` | PR 코멘트는 65,535자 제한 |
| PR plan에 `-lock=false` | 읽기 role이 잠금 파일을 쓸 수 없을 때. 대신 PR plan을 apply에 쓰지 않음 |

- GitHub는 2026-07-15 이후 만든(또는 opt-in한) 저장소의 OIDC `sub`에 **바뀌지 않는 owner·repo ID**를 넣는다. trust policy를 이름 기반 `sub`로 만들었다면 `AssumeRoleWithWebIdentity`가 실패할 수 있으니 형식을 확인한다.
- 위 워크플로는 공식 문서·README의 구성 요소로 이 레포에서 조합한 예시다. 실제 저장소에서 실행하지 않았다. 액션은 공급망 공격을 막기 위해 **커밋 SHA로 고정**하는 편이 더 안전하다.

### Drift 감지 작업

```yaml
on:
  schedule:
    - cron: "0 0 * * *"     # 매일 09:00 KST
jobs:
  drift:
    runs-on: ubuntu-latest
    steps:
      # ... checkout, 자격증명(읽기 role), setup-terraform, init
      - id: plan
        run: terraform plan -detailed-exitcode -lock=false
        continue-on-error: true
      - if: steps.plan.outputs.exitcode == '2'
        run: echo "drift detected" && exit 1     # 알림 연동
```

`-detailed-exitcode`: 0 = 변경 없음, 1 = 오류, 2 = 변경 있음([03](./03-workflow-graph.md#drift-밖에서-바뀐-것-찾기)). `steps.<id>.outputs.exitcode`는 setup-terraform의 wrapper(기본 켜짐)가 제공하는 출력이다.

## 테스트: terraform test(1.6+)

출처: [Tests](https://developer.hashicorp.com/terraform/language/tests), [Mocking](https://developer.hashicorp.com/terraform/language/tests/mocking)

```hcl
# tests/bucket.tftest.hcl
mock_provider "aws" {}            # 1.7+. 실제 AWS를 부르지 않음

variables {
  bucket_name = "acme-test-bucket"
}

run "name_is_set" {
  command = plan                  # 기본값은 apply

  assert {
    condition     = aws_s3_bucket.this.bucket == "acme-test-bucket"
    error_message = "버킷 이름이 입력값과 다릅니다."
  }
}
```

```bash
terraform test
```

| 항목 | 내용 |
| --- | --- |
| 파일 | `*.tftest.hcl`(루트나 `tests/` 디렉터리) |
| `run` 블록 | 차례로 실행. `command = plan`이면 리소스를 만들지 않는 단위 테스트, `apply`(기본)면 **실제로 만들고 끝나면 삭제** |
| `assert` | 조건이 거짓이면 실패 |
| `mock_provider` | 1.7+. 가짜 provider로 빠르게 테스트 |
| 변수 validation 실패 테스트 | `expect_failures`로 "이 입력은 거부되어야 한다"를 검사 |

`command = apply` 테스트는 **실제 리소스를 만들고 비용이 든다.** 별도 테스트 계정에서 돌린다.

## 정적 분석

| 도구 | 하는 일 | 명령 | 출처 |
| --- | --- | --- | --- |
| `terraform fmt -check` | 코드 정렬 | `terraform fmt -check -recursive` | [fmt](https://developer.hashicorp.com/terraform/cli/commands/fmt) |
| `terraform validate` | 문법·일관성(API 확인 안 함) | `terraform validate` | [validate](https://developer.hashicorp.com/terraform/cli/commands/validate) |
| **tflint** + AWS ruleset | 존재하지 않는 인스턴스 타입 같은 provider별 오류, 관례 위반 | `tflint --init && tflint` | [tflint-ruleset-aws](https://github.com/terraform-linters/tflint-ruleset-aws) |
| **checkov** | 보안·규정 정책(암호화 안 된 버킷 등) | `checkov -d .` 또는 `-f tfplan.json --framework terraform_plan` | [checkov CLI](https://www.checkov.io/2.Basics/CLI%20Command%20Reference.html) |
| **trivy** | 보안 설정 검사(tfsec 후속) | `trivy config .` | [trivy terraform](https://trivy.dev/latest/docs/coverage/iac/terraform/) |

```hcl
# .tflint.hcl
plugin "aws" {
  enabled = true
  version = "0.49.0"
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}
```

- **tfsec는 trivy로 옮겨 가는 중**이다. tfsec README가 trivy로 전환을 권하고, 마지막 릴리스는 2025-05-02다. 새로 시작한다면 trivy를 쓴다.
- checkov는 `#checkov:skip=CKV_AWS_20:이유` 주석으로 특정 검사를 이유와 함께 건너뛸 수 있다.
- 조직 규칙을 강제하는 다른 방법으로 CloudFormation Guard도 Terraform plan JSON을 검사할 수 있다([aws-cloudformation/06](../aws-cloudformation/06-issues.md)).

## OpenTofu와의 차이

출처: [OpenTofu FAQ](https://opentofu.org/faq/), [State encryption](https://opentofu.org/docs/language/state/encryption/), [CNCF OpenTofu](https://www.cncf.io/projects/opentofu/)

| 항목 | Terraform | OpenTofu |
| --- | --- | --- |
| 라이선스 | BSL(1.6.0+) | **MPL 2.0** |
| 운영 주체 | HashiCorp(IBM) | Linux Foundation 프로젝트, **CNCF Sandbox**(2025-04-23) |
| 최신 버전 | 1.16.6 | 1.13.1(2026-10-01) |
| 명령 | `terraform` | `tofu` |
| state 호환 | — | Terraform **1.5.x까지** 만든 state를 그대로 읽음 |
| **state 암호화** | 없음(backend 암호화에 의존) | **클라이언트 측 암호화**(1.7+, AWS KMS 등) |
| 모듈 source·backend에 변수 | 1.15+ | 1.8+(early evaluation) |
| provider 블록 `for_each` | — | 1.9+ |
| provider | 같은 provider 사용(레지스트리는 별도) | — |

두 도구는 1.6 이후 각자 기능이 추가되어 **완전히 같지 않다.** 한쪽에서 쓴 새 기능(Terraform의 ephemeral·write-only, OpenTofu의 state 암호화 등)은 다른 쪽에서 동작하지 않을 수 있다. 전환할 때는 사용하는 기능 목록부터 비교한다.

## 흔한 사고와 안티패턴

| 안티패턴 | 결과 | 대안 | 근거 |
| --- | --- | --- | --- |
| state를 Git에 커밋 | **비밀 노출**, 충돌로 state 손상 | S3 backend + 잠금 + 암호화 | [04](./04-state.md) |
| 비밀을 변수·리소스 인자로 그대로 | state에 평문 저장 | ephemeral, write-only, Secrets Manager | [04](./04-state.md#state에-비밀이-남지-않게) |
| 콘솔에서 직접 수정 | drift, 다음 apply가 되돌려 버림 | 코드로만 변경, drift 감지 작업 | [03](./03-workflow-graph.md) |
| `-target` 습관 | 숨은 drift 누적 | state 분리 | [03](./03-workflow-graph.md#-target과--replace) |
| 거대한 단일 state | 느린 plan, 큰 blast radius | 수명·소유자별 분리 | [05](./05-modules-structure.md#환경계정-분리) |
| 목록 + `count` | 중간 삭제 시 엉뚱한 리소스 교체 | `for_each`, `moved` | [02](./02-hcl.md#count에서-생기는-사고) |
| provider·모듈 버전 미고정 | 같은 코드로 다른 결과 | `~>` 제약, lock 파일 커밋 | [03](./03-workflow-graph.md#init) |
| 잠금 없이 동시 apply, 무분별한 `force-unlock` | state 손상 | `use_lockfile`, CI 동시 실행 제한 | [04](./04-state.md#잠금이-남아-있을-때) |
| PC에서 운영 apply | 이력 없음, 권한 과다 | CI + OIDC + plan 리뷰 | 이 문서 |
| plan을 읽지 않고 승인 | DB 교체 같은 사고 | `-/+`·destroy 수 확인 | [03](./03-workflow-graph.md#plan-읽는-법) |
| `prevent_destroy`만 믿기 | 블록을 지우면 삭제됨 | 서비스 자체 삭제 보호 병행 | [02](./02-hcl.md#lifecycle) |

## 출처

- https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws
- https://github.com/aws-actions/configure-aws-credentials
- https://github.com/hashicorp/setup-terraform
- https://developer.hashicorp.com/terraform/cli/commands/plan
- https://developer.hashicorp.com/terraform/language/tests
- https://developer.hashicorp.com/terraform/language/tests/mocking
- https://github.com/terraform-linters/tflint-ruleset-aws
- https://www.checkov.io/2.Basics/CLI%20Command%20Reference.html
- https://trivy.dev/latest/docs/coverage/iac/terraform/
- https://github.com/aquasecurity/tfsec
- https://opentofu.org/faq/
- https://opentofu.org/docs/language/state/encryption/
- https://www.cncf.io/projects/opentofu/

---
*조사 기준일: 2026-10-21. 워크플로 YAML은 문서 구성 요소로 이 레포에서 조합했고 실행하지 않았다. 테스트 HCL은 Terraform 1.16.6으로 검사했다(결과는 README 참고).*
