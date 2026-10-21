# 모듈과 프로젝트 구조

> 모듈은 **입력(variable)과 출력(output)이 분명한 작은 단위**로 만들고, 트리는 **한 단계로 얕게** 유지한다(HashiCorp 권장). 공유 모듈 안에는 **`provider` 블록을 두지 않고**, 호출하는 쪽에서 provider를 넘긴다. 버전은 루트에서 **`~>`로 위아래를 함께** 고정한다. 환경은 **계정·환경별 디렉터리와 backend**로 나누고, 하나의 거대한 state 대신 **수명·소유자별로 state를 쪼갠다.** AWS provider는 `assume_role`, `default_tags`, `allowed_account_ids`로 "엉뚱한 계정에 배포"를 막는다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[04](./04-state.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 루트 모듈(root module) | `terraform`을 실행하는 디렉터리의 코드. backend와 provider 설정이 여기 있음 |
| 자식 모듈(child module) | 루트에서 `module` 블록으로 부르는 모듈 |
| Provider alias | 같은 provider를 다른 설정(리전·계정)으로 여러 개 쓰기 위한 별칭 |
| Dependency inversion | 모듈 안에서 만들지 않고 밖에서 만든 것(VPC ID 등)을 입력으로 받는 설계 |
| Blast radius | 실수 하나가 영향을 주는 범위 |

## 모듈 기본

출처: [Module sources](https://developer.hashicorp.com/terraform/language/modules/sources), [Standard structure](https://developer.hashicorp.com/terraform/language/modules/develop/structure), [Version constraints](https://developer.hashicorp.com/terraform/language/expressions/version-constraints)

```
modules/
└── web-service/
    ├── main.tf        # 리소스
    ├── variables.tf   # 입력
    ├── outputs.tf     # 출력
    ├── versions.tf    # required_providers (커뮤니티 관례)
    └── README.md
```

```hcl
module "api" {
  source = "./modules/web-service"     # 로컬

  name       = "api"
  vpc_id     = data.aws_ssm_parameter.vpc_id.value
  subnet_ids = local.private_subnet_ids
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"   # Registry
  version = "~> 6.7"
  # ...
}

module "shared" {
  source = "git::https://github.com/acme/tf-modules.git//shared?ref=v1.4.0"   # Git(태그 고정)
}
```

| 출처 | 버전 고정 |
| --- | --- |
| 로컬 경로 | 같은 저장소라 필요 없음 |
| Registry | `version = "~> 6.7"` |
| Git | `?ref=v1.4.0`(태그·SHA). `version` 인자는 Registry에서만 동작 |

- `.terraform.lock.hcl`은 provider만 고정한다. **모듈은 `version`이나 `ref`로 직접 고정**한다([03](./03-workflow-graph.md#init)).
- HashiCorp 권장: 재사용 모듈은 **최소 버전만**(`>= 6.0`), 루트 모듈은 **`~>`로 위아래 모두** 정한다.
- `terraform-aws-modules`는 많이 쓰이는 **커뮤니티 모듈**이다(Registry에서 HashiCorp verified 표시는 없음). 쓰기 전에 코드를 읽어 보고 버전을 고정한다.

### 좋은 모듈의 조건

출처: [Module composition](https://developer.hashicorp.com/terraform/language/modules/develop/composition), [Providers within modules](https://developer.hashicorp.com/terraform/language/modules/develop/providers)

| 원칙 | 이유 |
| --- | --- |
| **트리를 얕게**(자식 모듈 한 단계) | HashiCorp: "strongly recommend keeping the module tree flat" |
| **필요한 것을 입력으로 받기**(dependency inversion) | VPC를 모듈 안에서 만들지 않고 `vpc_id`를 받으면 여러 곳에서 재사용 가능 |
| **공유 모듈에 `provider` 블록 금지** | provider 블록이 있는 모듈은 `count`·`for_each`·`depends_on`을 못 씀 |
| 출력은 필요한 것만 | 다른 모듈이 의존하는 계약이 됨 |
| 변수에 타입·설명·validation | 잘못된 입력을 plan에서 막음 |
| 리소스 하나를 감싸기만 하는 모듈은 피하기 | 이득 없이 한 단계만 늘어남(이 레포의 정리) |

## AWS provider 설정

출처: [AWS provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs), [Enhanced region support](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/enhanced-region-support), [Version 6 upgrade](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/version-6-upgrade)

최신 AWS provider는 6.69.1(2026-10-15)이고, 6.0.0은 2025-06-18에 나왔다.

```hcl
provider "aws" {
  region              = "ap-northeast-2"
  allowed_account_ids = ["111122223333"]       # 다른 계정 자격증명이면 실패 → 엉뚱한 계정 배포 방지

  assume_role {
    role_arn     = "arn:aws:iam::111122223333:role/terraform-deploy"
    session_name = "terraform"
  }

  default_tags {
    tags = {
      ManagedBy = "terraform"
      Repo      = "acme/infra"
    }
  }
}

# 다른 리전(us-east-1)이 필요한 리소스용 별칭
provider "aws" {
  alias  = "use1"
  region = "us-east-1"
  assume_role {
    role_arn = "arn:aws:iam::111122223333:role/terraform-deploy"
  }
}

resource "aws_acm_certificate" "cdn" {
  provider          = aws.use1               # CloudFront 인증서는 us-east-1
  domain_name       = "example.com"
  validation_method = "DNS"
}
```

| 설정 | 하는 일 | 주의 |
| --- | --- | --- |
| `assume_role` | 실행 자격증명으로 지정한 role을 맡아 배포([aws-iam/04](../aws-iam/04-roles-sts.md)) | 여러 블록이면 role chaining(1시간 제한) |
| `allowed_account_ids` | 지정한 계정이 아니면 실행 거부 | `forbidden_account_ids`와 함께 못 씀 |
| `default_tags` | 모든 리소스에 기본 태그 | 리소스 `tags`로 덮어쓸 수는 있지만 뺄 수는 없음. `aws_autoscaling_group`은 미지원 |
| `alias` | 다른 리전·계정용 provider | 리소스에 `provider = aws.use1`로 지정 |

### 6.0의 리소스별 region 인자

AWS provider 6.0부터 대부분의 리소스와 데이터 소스에 **`region` 인자**가 생겼다. 리전마다 provider 별칭을 만들지 않아도 된다.

```hcl
resource "aws_s3_bucket" "replica" {
  region = "ap-northeast-1"     # provider 기본 리전과 다른 곳에 생성
  bucket = "acme-replica-111122223333"
}
```

- `region`을 바꾸면 **리소스가 교체**된다(삭제하는 것은 교체 아님).
- IAM 같은 **글로벌 리소스**에는 없다.
- 기존 리전별 provider 방식도 그대로 유효하고 deprecated가 아니다.
- 5.x에서 올릴 때는 **최신 5.x로 먼저 올린 뒤** `~> 6.0`으로 바꾼다.

### 모듈에 provider 넘기기

```hcl
# 루트
module "cdn" {
  source = "./modules/cdn"
  providers = {
    aws      = aws
    aws.use1 = aws.use1
  }
}

# modules/cdn/versions.tf
terraform {
  required_providers {
    aws = {
      source                = "hashicorp/aws"
      version               = ">= 6.0"
      configuration_aliases = [aws.use1]
    }
  }
}
```

별칭이 붙은 provider는 **자동으로 상속되지 않는다.** `providers`로 명시적으로 넘기고, 모듈은 `configuration_aliases`로 받는다.

## 환경·계정 분리

[aws-organizations/02](../aws-organizations/02-structure-ou.md)처럼 환경을 계정으로 나눴다면, Terraform도 **계정·환경 단위로 디렉터리와 state를 나누는** 구성이 자연스럽다.

```
infra/
├── modules/                    # 재사용 모듈
│   ├── network/
│   └── web-service/
├── bootstrap/                  # state 버킷·배포 role (처음 한 번)
└── live/
    ├── shared/
    │   └── network/            # backend key: shared/network
    ├── prod/
    │   ├── network/            # backend key: prod/network
    │   └── app/                # backend key: prod/app
    └── dev/
        ├── network/
        └── app/
```

| 원칙 | 이유 |
| --- | --- |
| **디렉터리 = state 하나** | plan 범위가 작아져 빠르고, 실수의 영향이 그 디렉터리로 한정됨 |
| **수명·소유자별로 나누기** | 자주 안 바뀌는 네트워크와 매일 바뀌는 앱을 분리 |
| **환경마다 다른 backend key, 다른 배포 role** | prod 권한으로 dev를, dev 권한으로 prod를 건드리지 못함 |
| 스택 사이 값은 Parameter Store로 | `terraform_remote_state`의 state 전체 접근을 피함([04](./04-state.md#다른-state의-값-쓰기)) |
| 환경 차이는 변수로 | 같은 모듈 + `prod.tfvars`/`dev.tfvars` |

- **bootstrap 문제**: state 버킷 자체는 Terraform으로 만들기 전에 존재해야 한다. 보통 별도의 작은 구성(로컬 state)으로 만든 뒤 S3로 옮기거나, CloudFormation으로 만든다.
- 디렉터리마다 반복되는 backend·provider 설정을 줄여 주는 도구(Terragrunt 등)도 있다. 이번 조사에서 1차 출처로 비교하지 않았다.
- HCP Terraform(원격 실행·state·정책)을 쓰면 workspace 단위로 이 분리를 관리한다. 요금은 리소스당 월 과금(Essentials $0.10, Standard $0.47, Premium $0.99)이다. ([HashiCorp pricing](https://www.hashicorp.com/en/pricing))

## 이슈 / 주의

- **하나의 거대한 state**: plan이 느려지고, 한 번의 실수가 모든 것에 영향을 준다. `-target`을 자주 쓰게 되면 분리할 때다.
- **모듈 버전 미고정**: 모듈이 바뀌면 같은 코드로도 다른 plan이 나온다.
- **공유 모듈 안의 provider 블록**: `for_each`를 쓸 수 없고, 모듈을 지우기도 어려워진다.
- **깊은 모듈 중첩**: 어디서 무엇이 만들어지는지 추적하기 어렵다.
- **`allowed_account_ids` 없이 여러 계정 운영**: 자격증명을 잘못 잡으면 다른 계정에 배포된다.
- **`default_tags`와 리소스 `tags` 충돌**: 같은 키를 양쪽에 두면 plan이 계속 바뀌는 문제가 생길 수 있으니 키를 겹치지 않게 한다(이 레포의 정리, 확인 필요).

## 출처

- https://developer.hashicorp.com/terraform/language/modules/sources
- https://developer.hashicorp.com/terraform/language/modules/develop/structure
- https://developer.hashicorp.com/terraform/language/modules/develop/composition
- https://developer.hashicorp.com/terraform/language/modules/develop/providers
- https://developer.hashicorp.com/terraform/language/expressions/version-constraints
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/enhanced-region-support
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/guides/version-6-upgrade
- https://registry.terraform.io/v1/modules/terraform-aws-modules/vpc/aws
- https://github.com/hashicorp/terraform-provider-aws/releases
- https://www.hashicorp.com/en/pricing

---
*조사 기준일: 2026-10-21. HCL 예시는 Terraform 1.16.6 `terraform validate`로 검사했다(결과는 README 참고). 디렉터리 구성은 이 레포의 정리이며 HashiCorp 공식 규칙은 아니다.*
