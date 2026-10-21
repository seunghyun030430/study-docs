# Terraform 개요

> Terraform은 인프라를 **HCL 코드로 "이렇게 되어 있어야 한다"고 선언**하면, **plan으로 차이를 보여 주고 apply로 맞춰 주는** IaC 도구다. AWS·GCP·Azure·쿠버네티스·GitHub 등은 **provider** 플러그인으로 다루고, 코드와 실제 리소스의 연결은 **state 파일**에 기록한다. CloudFormation과 달리 **state를 직접 관리해야 하고, 실패해도 자동으로 되돌리지 않는다.** 최신 안정 버전은 1.16.6(2026-10-14)이다.

이 문서는 처음 보는 사람을 기준으로 용어를 한 단계씩 쌓아 간다. 처음 나오는 용어는 **굵게** 표시하고 바로 뜻을 적는다. IaC의 필요성은 [aws-cloudformation/01](../aws-cloudformation/01-overview.md#1단계-iac가-왜-필요한가)과 같으므로 짧게 다룬다. 모든 용어는 맨 아래 [용어 정리](#용어-정리)에 모아 두었다.

## 1단계: 선언형 IaC

**IaC**(Infrastructure as Code)는 인프라를 코드로 정의해 도구가 만들게 하는 방식이다. 방식은 두 가지로 나뉜다.

| 방식 | 코드에 적는 것 | 예 |
| --- | --- | --- |
| 명령형 | "버킷을 만들어라, 그다음 태그를 붙여라" 같은 **순서** | 셸 스크립트, AWS CLI |
| **선언형** | "버킷이 이런 설정으로 **존재해야 한다**"는 **최종 상태** | Terraform, CloudFormation |

선언형은 같은 코드를 여러 번 실행해도 결과가 같다. 이미 원하는 상태면 아무것도 하지 않고, 다르면 **차이만** 고친다.

## 2단계: 코드의 기본 단위

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_s3_bucket" "logs" {
  bucket = "my-app-logs-20261021"
}

output "bucket_arn" {
  value = aws_s3_bucket.logs.arn
}
```

| 블록 | 뜻 |
| --- | --- |
| `terraform` | Terraform 자체 설정(필요한 provider와 버전 등) |
| **`provider`** | 대상 플랫폼과 연결하는 플러그인 설정. 여기서는 AWS 서울 리전 |
| **`resource`** | 만들 리소스. `"aws_s3_bucket"`은 **리소스 타입**, `"logs"`는 코드 안에서 부르는 **이름** |
| `output` | 실행 후 꺼내 볼 값 |

- 이 코드는 **HCL**(HashiCorp Configuration Language)로 쓰고, 파일 확장자는 `.tf`다.
- `aws_s3_bucket.logs`처럼 `타입.이름`으로 리소스를 가리키는 것을 **주소(address)**라고 한다.
- `aws_s3_bucket.logs.arn`처럼 다른 리소스의 속성을 참조하면, Terraform이 **의존 관계**를 알아서 파악한다.

문법은 [02-hcl.md](./02-hcl.md)에서 자세히 다룬다.

## 3단계: Provider — 무엇이든 다루는 플러그인

**Provider**는 각 플랫폼의 API를 호출하는 플러그인이다. Terraform 본체는 provider를 통해서만 실제 리소스를 만든다.

| provider | 다루는 것 |
| --- | --- |
| `hashicorp/aws` | AWS 리소스 |
| `hashicorp/awscc` | AWS(CloudFormation 스키마로 자동 생성, Cloud Control API 기반) |
| `hashicorp/kubernetes`, `hashicorp/helm` | 쿠버네티스 |
| `integrations/github` | GitHub 저장소·팀 |
| `grafana/grafana` | Grafana 대시보드·알림([aws-prometheus-grafana/04](../aws-prometheus-grafana/04-amg.md)) |

provider는 [Terraform Registry](https://registry.terraform.io/)에서 내려받는다. 이것이 Terraform이 **여러 클라우드와 SaaS를 한 도구로** 다룰 수 있는 이유다.

## 4단계: State — 코드와 실제를 잇는 기록

**State**(`terraform.tfstate`)는 "코드의 `aws_s3_bucket.logs`가 실제로는 어떤 버킷인가"를 기록한 JSON 파일이다.

```
코드(.tf)              state                     실제 AWS
aws_s3_bucket.logs ─▶ id = "my-app-logs-…"  ─▶  S3 버킷 my-app-logs-…
                      arn, tags, …(속성 전체)
```

- Terraform은 plan 때 **코드, state, 실제 리소스**를 비교해 무엇을 바꿀지 정한다.
- state에는 리소스 속성이 **평문으로** 저장된다. DB 비밀번호 같은 값도 들어갈 수 있다.
- 팀이 함께 쓰려면 state를 S3 같은 **원격 저장소(backend)**에 두고 **잠금(lock)**으로 동시 실행을 막아야 한다.

CloudFormation은 이 기록을 AWS가 스택으로 대신 관리하지만, Terraform은 **사용자가 직접 관리**한다. 자세한 내용은 [04-state.md](./04-state.md)에서 다룬다.

## 5단계: 실행 흐름 — init, plan, apply

출처: [terraform init](https://developer.hashicorp.com/terraform/cli/commands/init), [plan](https://developer.hashicorp.com/terraform/cli/commands/plan), [apply](https://developer.hashicorp.com/terraform/cli/commands/apply)

```bash
terraform init      # provider·모듈 설치, backend 연결, .terraform.lock.hcl 기록
terraform plan      # 코드 vs state vs 실제를 비교해 변경 계획 출력
terraform apply     # 계획을 보여 주고 승인하면 실행
terraform destroy   # 코드가 관리하는 리소스 모두 삭제 (주의!)
```

```
Terraform will perform the following actions:

  # aws_s3_bucket.logs will be created
  + resource "aws_s3_bucket" "logs" {
      + arn    = (known after apply)
      + bucket = "my-app-logs-20261021"
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

| 기호 | 뜻 |
| --- | --- |
| `+` | 생성 |
| `~` | 그 자리에서 수정 |
| `-` | 삭제 |
| `-/+` | **삭제 후 다시 생성(교체)**. 데이터가 있는 리소스면 데이터가 사라질 수 있음 |
| `(known after apply)` | 만들어 봐야 알 수 있는 값(ARN, ID 등) |

**plan을 읽는 습관이 Terraform 사고를 막는 가장 큰 장치다.** 특히 `-/+`와 `must be replaced`를 놓치지 않는다. 자세한 내용은 [03-workflow-graph.md](./03-workflow-graph.md)에서 다룬다.

## 6단계: 코드를 나누고 재사용하기 — 모듈

**모듈(module)**은 `.tf` 파일을 묶은 디렉터리다. "VPC 하나 만들기"처럼 반복되는 구성을 모듈로 만들어 입력값만 바꿔 재사용한다.

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"

  name = "prod"
  cidr = "10.0.0.0/16"
}
```

Registry에 공개 모듈이 많다(예: `terraform-aws-modules`). 모듈 설계와 환경 분리는 [05-modules-structure.md](./05-modules-structure.md)에서 다룬다. 위 예의 모듈 버전 `~> 6.0`은 예시이며 실제 최신 버전은 Registry에서 확인한다.

## 7단계: 라이선스, OpenTofu, HCP Terraform

| 항목 | 내용 | 출처 |
| --- | --- | --- |
| 라이선스 | 2023-08-10 발표로 이후 버전(1.6.0 이상)은 **BSL**(Business Source License). 그 전에는 MPL 2.0 | [HashiCorp 발표](https://www.hashicorp.com/blog/hashicorp-adopts-business-source-license), [LICENSE](https://github.com/hashicorp/terraform/blob/main/LICENSE) |
| **OpenTofu** | Terraform을 포크한 오픈 소스 도구(Linux Foundation 프로젝트). 명령은 `tofu` | [OpenTofu](https://opentofu.org/docs/intro/) |
| HCP Terraform | HashiCorp의 관리형 서비스(예전 Terraform Cloud). 원격 실행·state 관리·정책 | — |

BSL은 HashiCorp와 경쟁하는 상용 제품에 쓰는 것을 제한하는 라이선스다. 일반 회사가 자기 인프라를 관리하는 데 쓰는 것은 대부분 해당하지 않지만, 정확한 조건은 라이선스 원문으로 확인한다. OpenTofu와의 차이는 [06](./06-cicd-testing-issues.md)에서 다룬다.

## 8단계: CloudFormation과 무엇이 다른가

| 항목 | Terraform | CloudFormation |
| --- | --- | --- |
| 대상 | 여러 클라우드·SaaS | AWS 중심 |
| 상태 관리 | **state 파일을 직접 관리**(backend·잠금·보안) | AWS가 스택으로 관리 |
| 미리 보기 | `terraform plan` | change set |
| 실패 시 | **자동으로 되돌리지 않음**("does not automatically roll back a partially-completed apply") | 자동 rollback |
| 언어 | HCL | YAML/JSON, CDK |

자세한 비교와 선택 기준은 [aws-cloudformation/05](../aws-cloudformation/05-cdk-terraform.md)에 정리해 두었다.

## 9단계: 버전마다 달라지는 기능

Terraform은 버전에 따라 쓸 수 있는 기능이 다르다. 이 레포의 다른 문서에 나오는 주요 기능의 도입 버전이다. 출처: [CHANGELOG](https://github.com/hashicorp/terraform/releases)

| 기능 | 도입 버전 | 문서 |
| --- | --- | --- |
| `import` 블록, `check` 블록 | 1.5 | [04](./04-state.md), [02](./02-hcl.md) |
| `terraform test` | 1.6 | [06](./06-cicd-testing-issues.md) |
| `removed` 블록, 테스트 mock | 1.7 | [04](./04-state.md), [06](./06-cicd-testing-issues.md) |
| provider 정의 함수 | 1.8 | [02](./02-hcl.md) |
| ephemeral 리소스·변수 | 1.10 | [04](./04-state.md) |
| S3 자체 잠금(`use_lockfile`) 정식, write-only 인자 | 1.11 | [04](./04-state.md) |
| `terraform query`, `action` 블록 | 1.14 | — |
| 모듈 `source`·`version`에 변수 사용 | 1.15 | [05](./05-modules-structure.md) |

`required_version`으로 팀 전체가 같은 범위의 버전을 쓰게 고정한다([02](./02-hcl.md#terraform-블록)).

## 용어 정리

| 용어 | 뜻 | 처음 나오는 곳 |
| --- | --- | --- |
| IaC / 선언형 | 인프라를 코드로 / 최종 상태를 적는 방식 | 1단계 |
| HCL / `.tf` | Terraform 설정 언어 / 그 파일 | 2단계 |
| provider | 플랫폼 API를 호출하는 플러그인 | 2·3단계 |
| resource / 리소스 타입 / 이름 | 만들 대상 / `aws_s3_bucket` 같은 종류 / 코드 안의 이름 | 2단계 |
| 주소(address) | `aws_s3_bucket.logs` 같은 리소스 식별자 | 2단계 |
| Registry | provider·모듈을 배포하는 공식 저장소 | 3단계 |
| state / backend / 잠금 | 코드와 실제의 연결 기록 / state 원격 저장소 / 동시 실행 방지 | 4단계 |
| init / plan / apply / destroy | 준비 / 변경 계획 / 실행 / 전체 삭제 | 5단계 |
| `-/+`(교체) | 삭제 후 다시 생성 | 5단계 |
| 모듈 | 재사용 가능한 `.tf` 묶음 | 6단계 |
| BSL / OpenTofu / HCP Terraform | 현재 라이선스 / 오픈 소스 포크 / HashiCorp 관리형 서비스 | 7단계 |

## 출처

- https://developer.hashicorp.com/terraform/intro
- https://developer.hashicorp.com/terraform/cli/commands/init
- https://developer.hashicorp.com/terraform/cli/commands/plan
- https://developer.hashicorp.com/terraform/cli/commands/apply
- https://developer.hashicorp.com/terraform/language/block/terraform
- https://registry.terraform.io/
- https://github.com/hashicorp/terraform/releases
- https://checkpoint-api.hashicorp.com/v1/check/terraform
- https://www.hashicorp.com/blog/hashicorp-adopts-business-source-license
- https://github.com/hashicorp/terraform/blob/main/LICENSE
- https://opentofu.org/docs/intro/

---
*조사 기준일: 2026-10-21. HCL 예시는 Terraform 1.16.6 `terraform validate`로 검사했다(결과는 README 참고). plan 출력은 문서 형식을 바탕으로 만든 예시다.*
