# CDK와 Terraform

> **CDK**는 TypeScript·Python 같은 언어로 인프라를 쓰면 **CloudFormation 템플릿을 만들어 주는 도구**다. 배포·rollback·drift는 그대로 CloudFormation이 맡는다. **Terraform**은 CloudFormation과 별개인 HashiCorp의 IaC 도구로, 여러 클라우드를 다루고 상태(state)를 직접 관리하며 **실패해도 자동으로 되돌리지 않는다.** AWS만 쓰고 AWS 관리형 기능(StackSets, Control Tower, Service Catalog)과 엮는다면 CloudFormation·CDK가, 여러 클라우드·SaaS를 한 도구로 다룬다면 Terraform이 자연스럽다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[04](./04-stacksets.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Construct | CDK에서 리소스(들)를 나타내는 기본 블록. 클래스로 표현 |
| Synth(합성) | CDK 코드를 실행해 CloudFormation 템플릿을 만들어 내는 단계 |
| Cloud assembly | synth 결과물(템플릿, 에셋)이 담긴 `cdk.out` 디렉터리 |
| Bootstrap | CDK가 배포에 쓸 S3 버킷·ECR·IAM role을 계정·리전에 미리 만드는 작업 |
| HCL | Terraform 설정 언어(HashiCorp Configuration Language) |
| Provider | Terraform이 각 플랫폼 API(AWS, GCP, GitHub 등)와 통신하는 플러그인 |
| State | Terraform이 "설정의 어떤 리소스가 실제 어떤 리소스인지" 기록한 파일 |
| Plan / apply | Terraform의 변경 미리 보기 / 실행 |

## AWS CDK

출처: [What is the AWS CDK?](https://docs.aws.amazon.com/cdk/v2/guide/home.html), [Constructs](https://docs.aws.amazon.com/cdk/v2/guide/constructs.html)

> CDK는 "an open-source software development framework for defining cloud infrastructure in code and provisioning it through AWS CloudFormation"이다.

- 지원 언어: TypeScript, JavaScript, Python, Java, C#/.NET, Go
- 현재 버전은 v2다. v1은 2023-06-01에 지원이 끝났다.

### Construct 수준

| 수준 | 무엇 | 예 |
| --- | --- | --- |
| **L1** | CloudFormation 리소스와 1:1. 이름이 `Cfn`으로 시작. 새 리소스·속성이 반영되기까지 최대 약 1주 | `CfnBucket` = `AWS::S3::Bucket` |
| **L2** | 좋은 기본값, 권한 연결 메서드 등이 들어간 고수준 API | `s3.Bucket`, `bucket.grantRead(role)` |
| **L3**(패턴) | 여러 리소스를 묶은 구성 | `ApplicationLoadBalancedFargateService` |

```ts
import { App, Stack, StackProps } from 'aws-cdk-lib';
import * as s3 from 'aws-cdk-lib/aws-s3';

class HelloCdkStack extends Stack {
  constructor(scope: App, id: string, props?: StackProps) {
    super(scope, id, props);

    new s3.Bucket(this, 'MyFirstBucket', {
      versioned: true
    });
  }
}

const app = new App();
new HelloCdkStack(app, "HelloCdkStack");
```

위 코드는 CDK 공식 문서의 예시다. `App` → `Stack` → construct 순으로 트리를 이루고, Stack 하나가 CloudFormation 스택 하나가 된다.

### 주요 명령

출처: [CDK CLI reference](https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd.html), [Bootstrapping](https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping-env.html)

```bash
cdk init app --language typescript   # 프로젝트 생성
cdk bootstrap                         # 계정·리전마다 한 번: CDKToolkit 스택 생성
cdk synth                             # 템플릿 생성 → cdk.out/
cdk diff                              # 배포된 스택과 비교(교체 여부 포함)
cdk deploy                            # CloudFormation으로 배포
cdk drift                             # CloudFormation drift detection 실행
cdk destroy                           # 스택 삭제
```

| 명령 | 메모 |
| --- | --- |
| `cdk bootstrap` | `CDKToolkit` 스택에 에셋용 S3 버킷, ECR 저장소, 배포용 IAM role 5개, 버전 SSM 파라미터를 만든다. AWS는 이 스택에 termination protection을 권장 |
| `cdk diff` | 정확한 교체 정보를 위해 읽기 전용 change set을 만든다. `--method=template`이면 템플릿만 비교(빠름) |
| `cdk drift` | `cdk diff`는 템플릿끼리 비교하고, `cdk drift`는 실제 리소스와 비교한다. `--fail`이면 drift가 있을 때 종료 코드 1 |
| `cdk refactor` | construct 이름을 바꾸거나 스택 사이로 옮겨도 리소스를 다시 만들지 않게 한다(CloudFormation stack refactoring 사용). **미리 보기 기능**이라 `--unstable=refactor` 필요 |
| `cdk migrate` | 기존 템플릿이나 IaC generator 결과를 CDK 코드로 변환 |

([cdk drift](https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd-drift.html), [cdk refactor](https://docs.aws.amazon.com/cdk/v2/guide/refactor.html))

### CDK를 쓸 때 알아 둘 점

- **결국 CloudFormation이다.** 템플릿 한도(리소스 500개 등), 교체 동작, rollback, `UPDATE_ROLLBACK_FAILED`가 그대로 적용된다([03](./03-stack-operations.md)).
- **논리 ID는 construct 경로로 만들어진다.** construct ID를 바꾸거나 다른 곳으로 옮기면 논리 ID가 바뀌어 **리소스가 교체**될 수 있다. `cdk diff`로 확인하고, 필요하면 `cdk refactor`를 쓴다.
- **L2 construct는 기본값을 대신 정한다.** 편하지만 어떤 리소스가 만들어지는지 `cdk synth` 결과를 한 번은 읽어 본다.
- `cdk-nag`(cdklabs 오픈 소스)로 보안·규정 준수 규칙을 검사할 수 있다. AWS 공식 서비스는 아니다. ([cdk-nag](https://github.com/cdklabs/cdk-nag))

## Terraform

출처: [What is Terraform?](https://developer.hashicorp.com/terraform/intro), [terraform apply](https://developer.hashicorp.com/terraform/cli/commands/apply), [S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3)

```hcl
terraform {
  backend "s3" {
    bucket       = "my-tf-state"
    key          = "app/prod.tfstate"
    region       = "ap-northeast-2"
    use_lockfile = true      # S3 native 잠금 (DynamoDB 잠금은 deprecated)
  }
}

provider "aws" {
  region = "ap-northeast-2"
}

resource "aws_cloudwatch_log_group" "app" {
  name              = "/app/prod"
  retention_in_days = 30
}
```

```bash
terraform init
terraform plan     # 변경 미리 보기
terraform apply    # 실행
```

| 개념 | 내용 |
| --- | --- |
| 흐름 | Write → Plan → Apply |
| Provider | AWS용은 `hashicorp/aws`. CloudFormation 스키마로 자동 생성되는 `hashicorp/awscc`(Cloud Control API 기반, 매주 릴리스)도 있음 |
| **State** | 설정과 실제 리소스를 잇는 파일. **팀이 함께 쓰려면 원격 저장(S3 등)과 잠금이 필요** |
| S3 잠금 | `use_lockfile = true`로 S3 자체 잠금(`.tflock` 파일). v1.11에서 정식 지원, DynamoDB 잠금은 deprecated |
| **실패 시** | "Terraform does not automatically roll back a partially-completed apply." 일부만 적용된 상태로 남음 |
| 라이선스 | 2023-08-10 발표로 이후 버전은 BSL(Business Source License). 오픈 소스 포크로 OpenTofu(Linux Foundation 프로젝트)가 있음 |

([HashiCorp BSL 발표](https://www.hashicorp.com/blog/hashicorp-adopts-business-source-license), [terraform-provider-awscc](https://github.com/hashicorp/terraform-provider-awscc), [OpenTofu](https://opentofu.org/docs/intro/))

## 비교

| 항목 | CloudFormation(+CDK) | Terraform |
| --- | --- | --- |
| 만든 곳 | AWS 관리형 서비스 | HashiCorp(IBM) CLI 도구 |
| 대상 | AWS(+ 레지스트리의 서드파티 타입) | AWS·GCP·Azure·Kubernetes·SaaS 등 다수 provider |
| 상태 관리 | **AWS가 스택으로 관리** | **state 파일을 직접 관리**(백엔드·잠금·보안) |
| 미리 보기 | change set, `cdk diff` | `terraform plan` |
| 실패 시 | **자동 rollback** | 자동 rollback 없음(부분 적용 상태) |
| Drift | drift detection API, `cdk drift`, drift-aware change set | `plan`이 실제 상태를 읽어 차이를 보여 줌 |
| 재사용 | nested stack, 모듈, CDK construct 라이브러리 | 모듈, Terraform Registry |
| 여러 계정·리전 | **StackSets**(Organizations 자동 배포) | provider 별칭·워크스페이스·외부 도구로 구성 |
| AWS 관리형 기능 연계 | Control Tower, Service Catalog, Quick Setup이 CloudFormation 기반 | 별도 |
| 새 AWS 기능 | 대부분 빠르게 지원(CDK L1은 최대 약 1주) | `aws` provider 업데이트 시점에 따름(`awscc`는 스키마에서 매주 생성) |
| 언어 | YAML/JSON, CDK는 범용 언어 | HCL |
| 비용 | `AWS::*` 리소스는 무료 | CLI는 무료, HCP Terraform 등은 유료 플랜 |

"Terraform `plan`이 drift를 보여 준다"는 행의 정확한 문구는 1차 출처에서 직접 확인하지 못했다(확인 필요).

### 어떻게 고를까

| 상황 | 추천 |
| --- | --- |
| AWS만 쓰고, 조직 전체 기본 설정을 StackSets·Control Tower로 배포 | **CloudFormation**(템플릿) |
| AWS만 쓰고, 개발자가 코드로 인프라를 관리하고 싶음 | **CDK** |
| 여러 클라우드·SaaS(Datadog, GitHub, Cloudflare 등)를 한 도구로 | **Terraform** |
| 이미 Terraform을 쓰는 조직 | Terraform 유지 + 조직 기본 설정만 StackSets로 하는 혼합도 흔함 |

어느 쪽을 쓰든 **콘솔에서 직접 고치지 않는 것**이 가장 중요하다. 두 도구 모두 직접 고친 부분은 다음 배포 때 덮어쓰거나 충돌한다.

## 출처

- https://docs.aws.amazon.com/cdk/v2/guide/home.html
- https://docs.aws.amazon.com/cdk/v2/guide/constructs.html
- https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd.html
- https://docs.aws.amazon.com/cdk/v2/guide/hello-world.html
- https://docs.aws.amazon.com/cdk/v2/guide/bootstrapping-env.html
- https://docs.aws.amazon.com/cdk/v2/guide/ref-cli-cmd-drift.html
- https://docs.aws.amazon.com/cdk/v2/guide/refactor.html
- https://github.com/cdklabs/cdk-nag
- https://developer.hashicorp.com/terraform/intro
- https://developer.hashicorp.com/terraform/cli/commands/apply
- https://developer.hashicorp.com/terraform/language/backend/s3
- https://developer.hashicorp.com/terraform/language/v1.11.x/upgrade-guides
- https://github.com/hashicorp/terraform-provider-awscc
- https://www.hashicorp.com/blog/hashicorp-adopts-business-source-license
- https://opentofu.org/docs/intro/

---
*조사 기준일: 2026-10-14. CDK 예시는 공식 문서 코드이고, Terraform 예시는 이 레포에서 작성했고 `terraform validate`(Terraform 1.5.7, AWS provider 6.68.0)로 리소스 부분을 검사해 통과했다. backend 블록은 `-backend=false`로 검사에서 뺐으며, `use_lockfile`은 Terraform 1.11 이상이 필요하다.*
