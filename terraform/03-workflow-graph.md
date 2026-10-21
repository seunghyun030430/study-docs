# 실행 흐름과 의존성 그래프

> `init`은 provider를 설치하고 **`.terraform.lock.hcl`**에 버전을 고정한다. 이 파일은 Git에 커밋한다. `plan`은 코드·state·실제 리소스를 비교해 변경을 보여 주고, 운영에서는 **`-out`으로 저장한 plan을 그대로 `apply`**한다. Terraform은 리소스 참조로 **의존성 그래프**를 만들어 독립된 것은 **동시에 10개씩** 처리한다. `-target`은 사고 복구용이고, `taint`·`refresh`는 deprecated다. 대신 `-replace`와 `-refresh-only`를 쓴다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)·[02](./02-hcl.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Dependency lock file | `.terraform.lock.hcl`. 선택한 provider 버전과 체크섬을 기록 |
| Saved plan | `terraform plan -out=파일`로 저장한 변경 계획 |
| Speculative plan | 저장하지 않고 보기만 하는 plan |
| 의존성 그래프 | 리소스 사이의 참조 관계를 나타낸 그래프. 실행 순서를 정함 |
| Refresh | 실제 리소스를 읽어 state를 최신으로 맞추는 단계 |
| Drift | 밖에서 바뀌어 실제가 코드·state와 달라진 상태 |

## init

출처: [terraform init](https://developer.hashicorp.com/terraform/cli/commands/init), [Dependency lock file](https://developer.hashicorp.com/terraform/language/files/dependency-lock)

| 하는 일 | 메모 |
| --- | --- |
| provider·모듈 설치 | `.terraform/` 디렉터리에 받음(Git에 커밋하지 않음) |
| backend 연결 | state 위치 설정([04](./04-state.md)) |
| **lock 파일 기록** | `.terraform.lock.hcl`에 provider 버전·체크섬 기록 → **Git에 커밋** |

```bash
terraform init                     # 처음, 또는 provider·모듈·backend가 바뀌었을 때
terraform init -upgrade            # 버전 제약 안에서 최신 provider·모듈로 올림
terraform init -lockfile=readonly  # CI: lock 파일을 바꾸지 않고 검증만
```

- lock 파일은 **provider만** 고정한다. 원격 모듈 버전은 기억하지 않으므로 모듈은 `version`을 정확히 적어 고정한다([05](./05-modules-structure.md)).
- 여러 OS에서 실행한다면 `terraform providers lock -platform=linux_amd64 -platform=darwin_arm64`로 체크섬을 함께 기록해 두면 CI에서 체크섬 오류가 나지 않는다.

```gitignore
# .gitignore
.terraform/
*.tfstate
*.tfstate.*
*.tfplan
crash.log
# .terraform.lock.hcl 은 커밋한다
```

## plan 읽는 법

출처: [terraform plan](https://developer.hashicorp.com/terraform/cli/commands/plan)

```
  # aws_db_instance.main must be replaced
-/+ resource "aws_db_instance" "main" {
      ~ engine_version = "15.7" -> "16.4"
      ~ identifier     = "myapp-db" -> "myapp-db-v2" # forces replacement
      ~ id             = "db-ABC" -> (known after apply)
        ...
    }

  # aws_s3_bucket.old will be destroyed
  # (because aws_s3_bucket.old is not in configuration)
  - resource "aws_s3_bucket" "old" { ... }

Plan: 1 to add, 1 to change, 2 to destroy.
```

| 표시 | 뜻 | 확인할 것 |
| --- | --- | --- |
| `+` | 생성 | — |
| `~` | 그 자리에서 수정 | 재시작이 필요한지 |
| `-` | 삭제 | 정말 지워도 되는지 |
| **`-/+`** | **삭제 후 생성(교체)** | **데이터 손실 여부** |
| `+/-` | 생성 후 삭제(`create_before_destroy`) | 이름 충돌 |
| `# forces replacement` | 이 속성 변경 때문에 교체됨 | 꼭 바꿔야 하는 속성인지 |
| `must be replaced` | 교체 대상 리소스 | — |
| `(known after apply)` | 실행 후에 정해지는 값 | — |
| `is not in configuration` | 코드에서 블록을 지워서 삭제됨 | 실수로 지운 블록인지 |

**마지막 줄 `Plan: N to add, N to change, N to destroy`의 destroy 숫자**를 항상 먼저 본다.

| 옵션 | 쓰임새 |
| --- | --- |
| `-out=tfplan` | plan 저장. apply 때 그대로 실행 |
| `-detailed-exitcode` | 종료 코드 0 = 변경 없음, 1 = 오류, 2 = 변경 있음(drift 감지 자동화에 씀) |
| `-destroy` | 삭제 plan 미리 보기 |
| `-refresh-only` | 실제 상태만 state에 반영하는 plan(코드 변경 없음) |
| `-refresh=false` | 실제 리소스를 읽지 않음(빠르지만 부정확할 수 있음) |
| `-var`, `-var-file` | 변수 값 |
| `-parallelism=n` | 동시 처리 수(기본 10) |

plan 파일에는 **민감 데이터가 평문으로** 들어 있으므로 아티팩트로 저장할 때 접근을 제한한다.

## apply

출처: [terraform apply](https://developer.hashicorp.com/terraform/cli/commands/apply)

```bash
# 개발: plan을 보여 주고 승인을 받음
terraform apply

# 운영: 저장한 plan을 검토한 뒤 그대로 실행 (다시 묻지 않음)
terraform plan -out=tfplan
terraform show tfplan          # 리뷰
terraform apply tfplan
```

- 저장한 plan을 apply하면 **리뷰한 내용 그대로** 실행된다. 리뷰와 실행 사이에 코드나 실제 상태가 바뀌어 plan이 맞지 않으면 실패한다.
- `-auto-approve`는 CI에서만 쓴다. AWS 문서와 마찬가지로 HashiCorp도 이때 **Terraform 밖에서 인프라를 바꾸지 못하게 하라**고 경고한다.
- **apply가 중간에 실패하면 되돌리지 않는다.** 이미 만든 리소스는 state에 남고, 원인을 고친 뒤 다시 apply한다.

> **삭제 주의**: `terraform destroy`(= `terraform apply -destroy`)는 코드가 관리하는 **모든 리소스를 삭제**한다. 먼저 `terraform plan -destroy`로 확인한다.

## 의존성 그래프와 병렬성

출처: [terraform graph](https://developer.hashicorp.com/terraform/cli/commands/graph), [depends_on](https://developer.hashicorp.com/terraform/language/meta-arguments/depends_on)

```
aws_vpc.main
   ├──▶ aws_subnet.private["a"] ──▶ aws_instance.app
   ├──▶ aws_subnet.private["b"]
   └──▶ aws_security_group.web ──▶ aws_instance.app
aws_s3_bucket.logs   (독립 → VPC와 동시에 생성)
```

- `aws_subnet.private["a"]`가 `aws_vpc.main.id`를 참조하므로 **VPC가 먼저** 만들어진다.
- 서로 참조하지 않는 리소스는 **동시에** 처리된다(기본 10개). API 호출 제한(쓰로틀링)에 걸리면 `-parallelism`을 낮춘다.
- 삭제는 반대 순서다. 의존하는 쪽(인스턴스)을 먼저 지운다.
- `terraform graph | dot -Tsvg > graph.svg`(GraphViz 필요)로 그래프를 그림으로 볼 수 있다.

## Drift: 밖에서 바뀐 것 찾기

출처: [terraform refresh](https://developer.hashicorp.com/terraform/cli/commands/refresh), [plan](https://developer.hashicorp.com/terraform/cli/commands/plan)

콘솔에서 누군가 보안 그룹 규칙을 바꿨다면, 다음 `plan`에서 Terraform은 **코드대로 되돌리는 변경**을 제안한다.

| 원하는 것 | 명령 |
| --- | --- |
| 바뀐 실제를 **코드대로 되돌림** | 그냥 `terraform plan` → `apply` |
| 바뀐 실제를 **state에만 반영**(코드는 나중에 맞춤) | `terraform apply -refresh-only` |
| drift가 있는지만 확인(자동화) | `terraform plan -detailed-exitcode` → 종료 코드 2면 변경 있음 |

- `terraform refresh`는 **deprecated**다. 검토 없이 state를 바꾸는 `apply -refresh-only -auto-approve`와 같기 때문이다. 대신 `terraform apply -refresh-only`로 검토 후 반영한다.
- CI에서 매일 `plan -detailed-exitcode`를 돌려 drift를 감지하는 패턴은 [06](./06-cicd-testing-issues.md)에서 다룬다.

## -target과 -replace

| 옵션 | 하는 일 | 주의 |
| --- | --- | --- |
| `-target=ADDRESS` | 지정한 리소스(와 그 의존 대상)만 plan·apply | HashiCorp: "exceptional circumstances, such as recovering from mistakes"에만. 평소에 쓰면 drift가 쌓임 |
| `-replace=ADDRESS` | 지정한 리소스를 강제로 교체(0.15.2+) | **데이터가 있는 리소스는 데이터 손실**. `taint`를 대신하는 방식 |

```bash
# 망가진 인스턴스 하나만 다시 만들기 (plan으로 먼저 확인)
terraform plan  -replace='aws_instance.app'
terraform apply -replace='aws_instance.app'
```

`terraform taint`는 deprecated이며 `-replace`를 쓴다. `-target`을 자주 쓰고 싶어진다면 **state가 너무 크다는 신호**다. state 분리는 [05](./05-modules-structure.md)에서 다룬다.

## 그 밖의 명령

| 명령 | 하는 일 |
| --- | --- |
| `terraform fmt -recursive` / `-check` | 코드 정렬 / 정렬이 안 맞으면 실패(CI) |
| `terraform validate` | 문법과 내부 일관성 검사. **원격 상태나 provider API는 확인하지 않음** |
| `terraform console` | 표현식을 대화형으로 계산 |
| `terraform output -json` | 출력값을 JSON으로(**sensitive 값도 평문으로 나옴**) |
| `terraform show [planfile]` | state나 plan 내용 보기 |
| `terraform providers` | 쓰는 provider 목록 |

## 이슈 / 주의

- **lock 파일을 커밋하지 않으면** 사람·CI마다 다른 provider 버전을 써서 plan이 달라진다.
- **plan 없이 `apply -auto-approve`**를 운영에서 쓰지 않는다. 저장한 plan을 리뷰하고 그대로 실행한다.
- **apply 실패 시 자동으로 되돌리지 않는다.** 실패한 상태를 그대로 두지 말고 원인을 고쳐 다시 apply한다.
- **`-target`은 습관이 되면 안 된다.**
- **plan 파일·`output -json`에는 비밀이 평문으로** 나온다.

## 출처

- https://developer.hashicorp.com/terraform/cli/commands/init
- https://developer.hashicorp.com/terraform/language/files/dependency-lock
- https://developer.hashicorp.com/terraform/cli/commands/plan
- https://developer.hashicorp.com/terraform/cli/commands/apply
- https://developer.hashicorp.com/terraform/cli/commands/destroy
- https://developer.hashicorp.com/terraform/cli/commands/refresh
- https://developer.hashicorp.com/terraform/cli/commands/taint
- https://developer.hashicorp.com/terraform/cli/commands/graph
- https://developer.hashicorp.com/terraform/cli/commands/fmt
- https://developer.hashicorp.com/terraform/cli/commands/validate
- https://developer.hashicorp.com/terraform/cli/commands/output
- https://developer.hashicorp.com/terraform/language/meta-arguments/depends_on

---
*조사 기준일: 2026-10-21. plan 출력은 공식 문서의 예시 형식으로 만든 것이다. `+/-` 기호는 공식 문서 표에는 없고 Terraform 소스 코드에서 확인했다.*
