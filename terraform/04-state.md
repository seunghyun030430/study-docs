# State 관리

> State는 **코드와 실제 리소스를 잇는 유일한 기록**이고, **비밀값이 평문으로 들어 있다.** 그래서 Git에 넣지 않는다. 대신 **S3 backend + `use_lockfile`(1.11+) + 버전 관리 + 암호화 + 접근 제한**으로 둔다. 리소스 이름을 바꿀 때는 **`moved`**, 기존 리소스를 가져올 때는 **`import`**, 지우지 않고 관리에서만 뺄 때는 **`removed`** 블록을 쓴다. CLI `state mv`/`rm`보다 이 블록들이 plan으로 검토할 수 있어 안전하다. 비밀을 state에 남기지 않으려면 **ephemeral 값과 write-only 인자**를 쓴다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)·[03](./03-workflow-graph.md)의 용어(state, backend, 잠금, plan)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Serial / lineage | state 파일의 변경 번호 / state의 고유 계보 ID. 잘못된 state로 덮어쓰는 것을 막는 데 씀 |
| Remote state | 로컬이 아닌 원격 저장소(S3 등)에 둔 state |
| Workspace(CLI) | 같은 코드·backend에서 state를 여러 개 두는 기능 |
| Import | 이미 있는 리소스를 Terraform 관리 아래로 가져오는 것 |
| Ephemeral 값 | state·plan 파일에 저장되지 않는 값(1.10+) |
| Write-only 인자 | 리소스에 전달만 하고 state에 저장하지 않는 인자(1.11+, `*_wo`) |

## State에 무엇이 들어 있나

출처: [State](https://developer.hashicorp.com/terraform/language/state), [Sensitive data in state](https://developer.hashicorp.com/terraform/language/state/sensitive-data)

```json
{
  "version": 4,
  "serial": 42,
  "lineage": "3f1c...",
  "resources": [
    {
      "type": "aws_db_instance",
      "name": "main",
      "instances": [
        { "attributes": { "id": "myapp-db", "password": "평문으로 저장될 수 있음", "...": "..." } }
      ]
    }
  ]
}
```

- 리소스의 **모든 속성**이 저장된다. 코드에 쓴 비밀번호, 생성된 키 같은 값도 **평문**이다.
- `sensitive = true`는 **화면 출력만** 가린다. state와 plan 파일에는 그대로 남는다.
- **직접 고치지 않는다.** 외부 도구는 `terraform output -json`이나 `terraform show -json`으로 읽는다.
- `serial`과 `lineage` 덕분에 오래되거나 다른 state로 덮어쓰려 하면 `state push`가 거부한다.

## 원격 state: S3 backend

출처: [S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3), [State locking](https://developer.hashicorp.com/terraform/language/state/locking)

HashiCorp는 state를 Git이나 잠금·접근 통제가 없는 저장소에 두면 **데이터 손실과 비밀 노출** 위험이 있다고 경고한다.

```hcl
terraform {
  backend "s3" {
    bucket       = "acme-tf-state-111122223333"
    key          = "network/prod/terraform.tfstate"
    region       = "ap-northeast-2"
    encrypt      = true
    use_lockfile = true      # 1.10에서 추가, 1.11 정식. <key>.tflock 객체로 잠금
  }
}
```

| 설정·조치 | 이유 |
| --- | --- |
| `use_lockfile = true` | 두 사람이 동시에 apply해 state가 깨지는 것을 막음. S3 객체 `<key>.tflock`으로 잠금 |
| `dynamodb_table` | **deprecated**. 이전 중에는 둘을 함께 켤 수 있고, 이후 제거 |
| `encrypt = true` | state·잠금 파일을 서버 측 암호화(`kms_key_id`로 고객 관리형 KMS 키 지정 가능) |
| **S3 버킷 버전 관리** | 문서가 "highly recommended". state가 깨지면 이전 버전으로 복구 |
| 버킷 정책·퍼블릭 차단 | state를 읽을 수 있는 사람을 최소화 |
| state 버킷을 별도 계정에 | 워크로드 계정 권한으로 state를 지우지 못하게(→ [05](./05-modules-structure.md)) |

잠금 파일에는 `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` 권한이 필요하다.

### 잠금이 남아 있을 때

```bash
terraform force-unlock <LOCK_ID>
```

> "Be very careful with this command." 다른 사람이 정말 실행 중이 아닌지 확인한 뒤, **자동 해제가 실패한 자기 잠금**만 푼다. `-lock=false`는 권장하지 않는다.

## 환경을 나누는 방법: workspace를 조심해서

출처: [Workspaces](https://developer.hashicorp.com/terraform/cli/workspaces)

```bash
terraform workspace new stg
terraform workspace select stg     # 같은 코드, 다른 state
```

HashiCorp 문서는 CLI workspace가 맞지 않는 경우를 명시한다.

- 환경마다 **다른 자격증명·접근 통제**가 필요할 때
- 개발 단계나 팀을 **분리**해야 할 때(모든 workspace가 같은 backend를 공유)
- "alone are not a suitable tool for system decomposition"

그래서 **dev·prod를 다른 계정에 두는 AWS 환경이라면 workspace보다 환경별 디렉터리 + 환경별 backend**가 일반적이다(→ [05](./05-modules-structure.md#환경계정-분리)). workspace는 같은 구성을 잠깐 여러 벌 띄우는 용도(기능별 테스트 환경 등)에 맞는다.

## 다른 state의 값 쓰기

출처: [terraform_remote_state](https://developer.hashicorp.com/terraform/language/state/remote-state-data)

`terraform_remote_state` 데이터 소스로 다른 state의 output을 읽을 수 있다. 하지만 읽는 쪽이 **state 전체에 접근 권한**이 있어야 하고, 그 state에는 민감 정보가 들어 있는 경우가 많다. HashiCorp는 대안을 권한다.

```hcl
# network 스택: 값을 Parameter Store에 게시
resource "aws_ssm_parameter" "vpc_id" {
  name  = "/network/prod/vpc_id"
  type  = "String"
  value = aws_vpc.main.id
}

# app 스택: 필요한 값만 읽음
data "aws_ssm_parameter" "vpc_id" {
  name = "/network/prod/vpc_id"
}
```

Parameter Store는 [aws-systems-manager/04](../aws-systems-manager/04-parameter-store.md)를 참고한다. CloudFormation의 Export/ImportValue와 비슷한 역할이지만 **강한 연결이 없어** 한쪽을 바꾸기 쉽다.

## 코드와 state를 맞추는 블록

### 가져오기: import 블록(1.5+)

출처: [Generating configuration](https://developer.hashicorp.com/terraform/language/import/generating-configuration), [import block](https://developer.hashicorp.com/terraform/language/block/import)

콘솔로 만든 리소스를 Terraform 관리 아래로 가져온다.

```hcl
import {
  to = aws_s3_bucket.legacy
  id = "legacy-bucket-name"
}
```

```bash
terraform plan -generate-config-out=generated.tf   # 리소스 설정 초안 생성(새 파일이어야 함)
# generated.tf를 검토·정리한 뒤
terraform apply
```

- 설정 생성 기능은 **experimental**이고 결과는 "템플릿"이다. 그대로 쓰지 말고 다듬는다.
- 1.7부터 `for_each`로 여러 개를 한 번에 가져올 수 있고, 1.16부터 모듈 안에서도 import 블록을 쓸 수 있다.
- import가 끝난 뒤에는 블록을 지워도 된다.

### 리팩터링: moved 블록

출처: [Refactoring](https://developer.hashicorp.com/terraform/language/modules/develop/refactoring)

Terraform 1.1부터 쓸 수 있다. 리소스 이름을 바꾸거나 모듈로 옮기면 Terraform은 "옛 것 삭제 + 새 것 생성"으로 본다. `moved`로 **같은 리소스**라고 알려 준다.

```hcl
# aws_instance.web → aws_instance.api 로 이름 변경
moved {
  from = aws_instance.web
  to   = aws_instance.api
}

# count → for_each 전환 (02의 count 사고 방지)
moved {
  from = aws_iam_user.by_count[0]
  to   = aws_iam_user.by_key["alice"]
}

# 루트의 리소스를 모듈 안으로
moved {
  from = aws_vpc.main
  to   = module.network.aws_vpc.this
}
```

공유 모듈에서 `moved` 블록을 지우면, 아직 옮기지 않은 사용자에게는 **breaking change**가 된다. 한동안 남겨 둔다.

### 관리에서 빼기: removed 블록(1.7+)

출처: [removed block](https://developer.hashicorp.com/terraform/language/block/removed)

```hcl
removed {
  from = aws_instance.legacy
  lifecycle {
    destroy = false     # 실제 리소스는 지우지 않고 state에서만 제거
  }
}
```

다른 state로 옮기거나 수동 관리로 넘길 때 쓴다. `destroy = true`(또는 lifecycle 생략)면 실제로 삭제된다.

### CLI state 명령

```bash
terraform state list                             # 관리 중인 리소스 주소 목록
terraform state show aws_s3_bucket.logs          # 한 리소스의 속성
terraform state mv aws_instance.web aws_instance.api   # 주소 변경 (moved 블록 권장)
terraform state rm aws_instance.legacy           # state에서 제거, 실제 리소스는 남음 (removed 블록 권장)
terraform state pull > backup.tfstate            # 현재 state 백업
```

> **주의**: `state mv`·`rm`은 plan 없이 **바로 state를 바꾼다.** 코드 리뷰를 거칠 수 있는 `moved`·`removed` 블록을 먼저 고려하고, CLI를 쓸 때는 `state pull`로 백업부터 한다. `state push -force`는 serial·lineage 검사를 건너뛰므로 쓰지 않는다.

### State가 깨졌을 때

1. 다른 사람의 실행을 멈추고 잠금 상태를 확인한다.
2. S3 버전 관리에서 **이전 버전의 state 객체**를 확인한다.
3. 이전 버전을 복원한 뒤 `terraform plan`으로 실제와의 차이를 본다.
4. 차이를 `import`·`removed`·`apply -refresh-only`로 맞춘다.

## State에 비밀이 남지 않게

출처: [Sensitive data](https://developer.hashicorp.com/terraform/language/state/sensitive-data), [Write-only arguments](https://developer.hashicorp.com/terraform/language/manage-sensitive-data/write-only)

| 방법 | 버전 | 효과 |
| --- | --- | --- |
| `sensitive = true` | — | 화면에서만 가림. **state에는 남음** |
| `ephemeral = true` 변수·output, ephemeral 리소스 | 1.10+ | 값이 **state·plan에 저장되지 않음** |
| **write-only 인자**(`*_wo` + `*_wo_version`) | 1.11+ | 리소스에 전달만 하고 state에 저장하지 않음. 값을 바꾸려면 버전 숫자를 올림 |
| 비밀을 Terraform 밖에서 관리 | — | RDS `manage_master_user_password`처럼 서비스가 Secrets Manager로 직접 관리 |

```hcl
variable "db_password" {
  type      = string
  ephemeral = true
}

resource "aws_db_instance" "main" {
  identifier          = "myapp-db"
  engine              = "postgres"
  instance_class      = "db.t4g.micro"
  allocated_storage   = 20
  username            = "appadmin"
  password_wo         = var.db_password
  password_wo_version = 1          # 비밀번호를 바꿀 때 2, 3으로 올림
  skip_final_snapshot = true
}
```

`password_wo`는 AWS provider의 `aws_db_instance`가 제공하는 write-only 인자다. 어떤 리소스가 write-only 인자를 지원하는지는 provider 문서에서 확인한다.

## 이슈 / 주의

- **state를 Git에 커밋하지 않는다.** 비밀이 평문으로 들어 있다.
- **잠금 없이 여러 명이 apply하면 state가 깨진다.** `use_lockfile`을 켠다.
- **버전 관리 없는 state 버킷**은 복구 수단이 없다.
- **workspace로 dev·prod를 나누면** 같은 backend·권한을 공유한다. 계정이 다르면 디렉터리를 나눈다.
- **`state rm`·`force-unlock`·`state push -force`**는 사고의 지름길이다. 블록 방식과 백업을 먼저 쓴다.
- **`terraform_remote_state`는 state 전체 접근**이 필요하다. Parameter Store 같은 대안을 쓴다.

## 출처

- https://developer.hashicorp.com/terraform/language/state
- https://developer.hashicorp.com/terraform/language/state/sensitive-data
- https://developer.hashicorp.com/terraform/language/backend/s3
- https://developer.hashicorp.com/terraform/language/state/locking
- https://developer.hashicorp.com/terraform/cli/workspaces
- https://developer.hashicorp.com/terraform/language/state/remote-state-data
- https://developer.hashicorp.com/terraform/language/import/generating-configuration
- https://developer.hashicorp.com/terraform/language/block/import
- https://developer.hashicorp.com/terraform/language/modules/develop/refactoring
- https://developer.hashicorp.com/terraform/language/block/removed
- https://developer.hashicorp.com/terraform/cli/commands/state/push
- https://developer.hashicorp.com/terraform/language/manage-sensitive-data/write-only
- https://raw.githubusercontent.com/hashicorp/terraform/v1.11/CHANGELOG.md

---
*조사 기준일: 2026-10-21. HCL 예시는 Terraform 1.16.6 `terraform validate`로 검사했다(결과는 README 참고). state JSON은 구조를 보여 주기 위한 예시다.*
