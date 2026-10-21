# HCL 문법

> Terraform 코드는 **블록**(`terraform`, `provider`, `resource`, `data`, `variable`, `output`, `locals`, `module`)으로 이루어진다. 여러 개를 만들 때는 **목록이면 `count`, 이름으로 구분되면 `for_each`**를 쓴다. 중간 원소를 지울 때 엉뚱한 리소스가 교체되는 사고는 대부분 `count`에서 생긴다. 리소스의 생성·삭제 방식은 **`lifecycle`**로 조정하고, 입력값과 결과는 **validation, precondition·postcondition, check**로 검증한다. 셋은 실패했을 때의 동작이 서로 다르다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)의 용어(provider, resource, 주소, 모듈, state)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 블록(block) | `resource "타입" "이름" { ... }`처럼 키워드와 중괄호로 묶인 설정 단위 |
| 인자(argument) | 블록 안의 `이름 = 값` |
| 표현식(expression) | 값을 계산하는 식(참조, 함수, 조건식 등) |
| 메타 인자(meta-argument) | 모든 리소스에 공통으로 쓰는 특수 인자(`count`, `for_each`, `depends_on`, `lifecycle`, `provider`) |
| 데이터 소스(data source) | 이미 있는 리소스나 정보를 **읽기만** 하는 블록 |
| 타입 제약 | 변수가 받을 값의 형태(`string`, `list(string)`, `object({...})` 등) |

## 블록 종류

출처: [terraform block](https://developer.hashicorp.com/terraform/language/block/terraform), [variable](https://developer.hashicorp.com/terraform/language/block/variable), [output](https://developer.hashicorp.com/terraform/language/block/output)

| 블록 | 하는 일 | 예 |
| --- | --- | --- |
| `terraform` | Terraform 버전, 필요한 provider, backend | 아래 예시 |
| `provider` | provider 설정(리전, 자격증명, 기본 태그) | `provider "aws" { region = "ap-northeast-2" }` |
| `resource` | **만들고 관리할** 리소스 | `resource "aws_vpc" "main" { ... }` |
| `data` | **읽기만** 할 정보 | 최신 AMI ID, 기존 VPC |
| `variable` | 입력값 | `var.env` |
| `output` | 출력값(모듈의 반환값) | `output "vpc_id" { ... }` |
| `locals` | 코드 안에서 쓰는 이름 붙은 값 | `local.common_tags` |
| `module` | 다른 모듈 호출 | [05](./05-modules-structure.md) |

### terraform 블록

```hcl
terraform {
  required_version = ">= 1.11, < 2.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }

  backend "s3" {
    bucket       = "my-tf-state"
    key          = "network/prod.tfstate"
    region       = "ap-northeast-2"
    use_lockfile = true
  }
}
```

| 버전 제약 | 뜻 |
| --- | --- |
| `= 1.2.0` | 정확히 그 버전 |
| `>= 1.2, < 2.0` | 범위 |
| `~> 6.0` | 6.x(7.0 미만) |
| `~> 6.3.1` | 6.3.x(6.4 미만) |

`backend`와 HCP Terraform용 `cloud` 블록은 함께 쓸 수 없다. backend는 [04](./04-state.md)에서 다룬다.

### variable, locals, output, data

```hcl
variable "env" {
  type        = string
  description = "배포 환경"
  validation {
    condition     = contains(["dev", "stg", "prod"], var.env)
    error_message = "env는 dev, stg, prod 중 하나여야 합니다."
  }
}

variable "db_password" {
  type      = string
  sensitive = true
}

variable "subnets" {
  type = map(object({
    cidr = string
    az   = string
  }))
}

locals {
  name_prefix = "myapp-${var.env}"
  common_tags = {
    Project = "myapp"
    Env     = var.env
  }
}

data "aws_ami" "al2023" {
  most_recent = true
  owners      = ["amazon"]
  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

output "ami_id" {
  value       = data.aws_ami.al2023.id
  description = "최신 Amazon Linux 2023 AMI"
}
```

| 인자 | 메모 |
| --- | --- |
| `type` | 생략하면 어떤 타입이든 받음. 타입을 적어 두는 편이 실수를 줄임 |
| `nullable` | 기본 `true`. `false`면 null을 받지 않음 |
| `sensitive` | 화면 출력에서 값을 가림. **state에는 평문으로 저장됨**(→ [04](./04-state.md#state에-비밀이-남지-않게)) |
| `ephemeral` | 1.10+. 값을 state·plan 파일에 남기지 않음 |
| `deprecated` | 1.15+. 이 변수가 곧 없어진다는 경고 |

변수 값은 `terraform.tfvars`, `*.auto.tfvars`, `-var`, `-var-file`, 환경변수 `TF_VAR_이름`으로 넣는다.

## 표현식과 함수

출처: [Conditional](https://developer.hashicorp.com/terraform/language/expressions/conditionals), [For](https://developer.hashicorp.com/terraform/language/expressions/for), [Splat](https://developer.hashicorp.com/terraform/language/expressions/splat), [Functions](https://developer.hashicorp.com/terraform/language/functions)

```hcl
locals {
  instance_type = var.env == "prod" ? "m7i.large" : "t3.small"      # 조건식
  upper_names   = [for n in var.names : upper(n)]                    # list for
  name_to_len   = { for n in var.names : n => length(n) }            # map for
  subnet_ids    = aws_subnet.private[*].id                           # splat
  tags          = merge(local.common_tags, { Role = "api" })         # 맵 합치기
  port          = try(var.config.port, 8080)                         # 없으면 기본값
  az_cidrs      = [for i in range(3) : cidrsubnet("10.0.0.0/16", 8, i)]  # 10.0.0.0/24, 10.0.1.0/24, 10.0.2.0/24
}
```

| 함수 | 하는 일 |
| --- | --- |
| `merge(a, b)` | 맵 합치기(키가 겹치면 뒤가 이김) |
| `try(expr, fallback)` | 처음으로 오류 없이 평가되는 값 |
| `can(expr)` | 오류 없이 평가되면 true(대부분 `try`가 더 적합) |
| `coalesce(a, b)` | null·빈 문자열이 아닌 첫 값 |
| `lookup(map, key, default)` | 맵에서 키 찾기 |
| `cidrsubnet(prefix, newbits, netnum)` | CIDR 나누기 |
| `jsonencode(value)` | IAM 정책 등 JSON 만들기 |
| `templatefile(path, vars)` | 템플릿 파일 렌더링(UserData 등) |
| `provider::<이름>::<함수>` | 1.8+. provider가 제공하는 함수 |

IAM 정책은 문자열보다 `jsonencode`나 `aws_iam_policy_document` 데이터 소스로 만드는 편이 오타를 줄인다.

```hcl
resource "aws_iam_role" "app" {
  name = "${local.name_prefix}-app"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "ec2.amazonaws.com" }
      Action    = "sts:AssumeRole"
    }]
  })
}
```

## 여러 개 만들기: count vs for_each

출처: [count](https://developer.hashicorp.com/terraform/language/meta-arguments/count), [for_each](https://developer.hashicorp.com/terraform/language/meta-arguments/for_each)

> "Use the count argument when you want to create nearly identical instances. Use for_each when some instance arguments must have distinct values that can't be directly derived from an integer index."

```hcl
# count: 숫자 인덱스로 주소가 매겨짐
resource "aws_iam_user" "by_count" {
  count = length(var.user_names)
  name  = var.user_names[count.index]
}
# 주소: aws_iam_user.by_count[0], [1], [2]

# for_each: 키로 주소가 매겨짐
resource "aws_iam_user" "by_key" {
  for_each = toset(var.user_names)
  name     = each.value
}
# 주소: aws_iam_user.by_key["alice"], ["bob"], ["carol"]
```

### count에서 생기는 사고

`user_names = ["alice", "bob", "carol"]`에서 **가운데 `bob`을 지우면** 이렇게 된다.

| 방식 | 결과 |
| --- | --- |
| `count` | `[1]`이 bob → carol로 **바뀌고**, `[2]`(carol)는 **삭제**. bob 하나만 지우려 했는데 carol이 수정·재생성되는 plan이 나옴 |
| `for_each` | `["bob"]`만 삭제. 나머지는 그대로 |

공식 문서에 이 시나리오를 그대로 설명한 문구는 찾지 못했지만, 숫자 인덱스로 주소를 매기는 방식에서 바로 따라 나오는 동작이다. **목록에서 중간이 빠질 수 있으면 `for_each`**를 쓴다. 이미 `count`로 만든 것을 옮길 때는 `moved` 블록을 쓴다([04](./04-state.md#리팩터링-moved-블록)).

- `count`와 `for_each`는 한 블록에 함께 쓸 수 없다.
- 둘 다 **plan 전에 값을 알아야** 한다. 다른 리소스를 만들어야 알 수 있는 ID로는 쓸 수 없다.

## dynamic 블록

출처: [Dynamic blocks](https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks)

리소스 안의 중첩 블록(보안 그룹 규칙 등)을 반복으로 만든다.

```hcl
resource "aws_security_group" "web" {
  name   = "${local.name_prefix}-web"
  vpc_id = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_ports
    content {
      from_port   = ingress.value
      to_port     = ingress.value
      protocol    = "tcp"
      cidr_blocks = ["10.0.0.0/8"]
    }
  }
}
```

> "Overuse of dynamic blocks can make configuration hard to read… Always write nested blocks out literally where possible."

## lifecycle

출처: [lifecycle](https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle)

```hcl
resource "aws_db_instance" "main" {
  # ... 설정 생략
  lifecycle {
    prevent_destroy = true
    ignore_changes  = [engine_version]   # 자동 마이너 업그레이드로 바뀐 값 무시
  }
}

resource "aws_launch_template" "app" {
  # ...
  lifecycle {
    create_before_destroy = true
  }
}
```

| 옵션 | 하는 일 | 주의 |
| --- | --- | --- |
| `create_before_destroy` | 교체 시 **새 것을 먼저 만들고** 옛 것을 지움(기본은 반대) | 이름 같은 고유값이 겹치면 실패. 의존 리소스에도 전파됨 |
| `prevent_destroy` | 이 리소스를 지우는 plan을 거부 | **리소스 블록 자체를 코드에서 지우면 막지 못함**(lifecycle 설정이 state에 남지 않음). 문서도 "sparingly" 사용 권장 |
| `ignore_changes` | 지정한 속성의 변경을 무시(`all` 가능) | 밖에서 바뀌는 값(오토스케일링 용량, 자동 태그)에 씀. 남용하면 drift를 숨김 |
| `replace_triggered_by` | 참조한 리소스·속성이 바뀌면 이 리소스를 교체 | managed resource만 참조 가능 |
| `precondition` / `postcondition` | 생성 전·후 조건 검사 | 아래 검증 표 |

`lifecycle` 안에는 **리터럴 값만** 쓸 수 있다. 변수나 표현식은 쓸 수 없다.

## 검증: validation, precondition·postcondition, check

출처: [Validate your configuration](https://developer.hashicorp.com/terraform/language/validate)

| 수단 | 버전 | 언제 | 실패하면 |
| --- | --- | --- | --- |
| variable `validation` | 0.13+ | plan | **오류로 중단** |
| `precondition` / `postcondition` | 1.2+ | 리소스 생성 전 / 생성 후 | **오류로 중단** |
| `check` 블록 | 1.5+ | plan·apply | **경고만 내고 계속 진행** |

```hcl
resource "aws_instance" "app" {
  ami           = data.aws_ami.al2023.id
  instance_type = local.instance_type

  lifecycle {
    precondition {
      condition     = data.aws_ami.al2023.architecture == "x86_64"
      error_message = "x86_64 AMI가 필요합니다."
    }
  }
}

check "site_up" {
  data "http" "home" {
    url = "https://example.com"
  }
  assert {
    condition     = data.http.home.status_code == 200
    error_message = "사이트가 200을 반환하지 않습니다."
  }
}
```

입력값 오류는 `validation`, 리소스 사이의 조건은 `precondition`, 배포 후 서비스 상태처럼 "막을 필요는 없지만 알고 싶은 것"은 `check`로 둔다.

## depends_on

> `depends_on`은 "last resort"로만 쓴다. "can cause Terraform to create more conservative plans that replace more resources than necessary." — [depends_on](https://developer.hashicorp.com/terraform/language/meta-arguments/depends_on)

참조(`aws_vpc.main.id`)로 의존성이 생기면 `depends_on`은 필요 없다. 참조로 드러나지 않는 의존(예: IAM 정책이 붙은 뒤에야 동작하는 Lambda)에만 쓴다.

## 이슈 / 주의

- **목록 + `count`**는 중간 원소 삭제 때 사고를 낸다. 이름으로 구분되면 `for_each`.
- **`prevent_destroy`는 블록을 지우면 무력**하다. 중요한 리소스는 별도 state로 분리하거나 provider·리소스 자체의 삭제 보호(RDS `deletion_protection` 등)를 함께 켠다.
- **`ignore_changes`는 drift를 숨긴다.** 정말 밖에서 바뀌어야 하는 속성에만 쓴다.
- **`sensitive`는 state를 보호하지 않는다.**
- **`dynamic` 남용**은 읽기 어려운 코드를 만든다.

## 출처

- https://developer.hashicorp.com/terraform/language/block/terraform
- https://developer.hashicorp.com/terraform/language/block/variable
- https://developer.hashicorp.com/terraform/language/block/output
- https://developer.hashicorp.com/terraform/language/expressions/conditionals
- https://developer.hashicorp.com/terraform/language/expressions/for
- https://developer.hashicorp.com/terraform/language/expressions/splat
- https://developer.hashicorp.com/terraform/language/expressions/dynamic-blocks
- https://developer.hashicorp.com/terraform/language/functions
- https://developer.hashicorp.com/terraform/language/meta-arguments/count
- https://developer.hashicorp.com/terraform/language/meta-arguments/for_each
- https://developer.hashicorp.com/terraform/language/meta-arguments/lifecycle
- https://developer.hashicorp.com/terraform/language/meta-arguments/depends_on
- https://developer.hashicorp.com/terraform/language/validate

---
*조사 기준일: 2026-10-21. HCL 예시는 Terraform 1.16.6 `terraform validate`로 검사했다(결과는 README 참고).*
