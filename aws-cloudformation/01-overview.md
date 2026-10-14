# AWS CloudFormation 개요

> CloudFormation은 AWS 리소스를 **템플릿(YAML/JSON 파일)으로 적어 두면 AWS가 그대로 만들고, 고치고, 지워 주는** AWS 기본 IaC 서비스다. 리소스는 **스택**이라는 묶음 단위로 관리되고, 실패하면 **자동으로 되돌린다(rollback).** AWS 리소스(`AWS::*`)를 다루는 데는 추가 요금이 없다. CDK는 코드로 이 템플릿을 만들어 주는 도구이고, Terraform은 같은 일을 하는 다른 회사의 도구다.

이 문서는 CloudFormation을 처음 보는 사람을 기준으로 용어를 한 단계씩 쌓아 간다. 처음 나오는 용어는 **굵게** 표시하고 바로 뜻을 적는다. 모든 용어는 맨 아래 [용어 정리](#용어-정리)에 모아 두었다.

## 1단계: IaC가 왜 필요한가

AWS 리소스를 콘솔에서 클릭으로 만들면 이런 문제가 생긴다.

- 개발·스테이징·운영 환경을 **똑같이** 만들기 어렵다. 어디선가 설정 하나가 다르다.
- 누가 언제 무엇을 바꿨는지 **기록이 남지 않는다.**
- 지울 때 무엇을 만들었는지 몰라 **찌꺼기 리소스**가 남는다.
- 다른 계정·리전에 같은 구성을 만들려면 처음부터 다시 클릭해야 한다.

**IaC**(Infrastructure as Code, 코드형 인프라)는 인프라를 **코드 파일로 정의**하고, 도구가 그 파일대로 만들게 하는 방식이다. 코드이므로 Git으로 버전을 관리하고, 리뷰하고, 같은 파일로 여러 환경을 찍어 낼 수 있다.

> "CloudFormation is a service that helps you model and set up your AWS resources so that you can spend less time managing those resources and more time focusing on your applications that run in AWS." — [What is CloudFormation?](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html)

## 2단계: 템플릿 — 무엇을 만들지 적은 파일

**템플릿(template)**은 만들 AWS 리소스를 적은 YAML 또는 JSON 파일이다.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Log bucket and log group for one environment
Parameters:
  Env:
    Type: String
    AllowedValues: [dev, prod]
Resources:
  LogBucket:
    Type: AWS::S3::Bucket
  AppLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub "/app/${Env}"
      RetentionInDays: 30
Outputs:
  BucketName:
    Value: !Ref LogBucket
```

| 부분 | 뜻 |
| --- | --- |
| `Resources` | **만들 리소스 목록**(유일한 필수 섹션). `LogBucket`, `AppLogGroup` 같은 이름을 **논리 ID(logical ID)**라고 함 |
| `Type` | 리소스 종류. `AWS::서비스::리소스` 형식(예: `AWS::S3::Bucket`) |
| `Properties` | 그 리소스의 설정 |
| `Parameters` | 배포할 때 바꿔 넣는 입력값(예: `Env=dev`) |
| `Outputs` | 배포 후 꺼내 볼 값(예: 만들어진 버킷 이름) |
| `!Ref`, `!Sub` | **내장 함수(intrinsic function)**. 다른 리소스·파라미터 값을 참조하거나 문자열을 조합 |

템플릿 문법은 [02-templates.md](./02-templates.md)에서 자세히 다룬다.

## 3단계: 스택 — 함께 관리되는 리소스 묶음

템플릿 하나를 배포하면 **스택(stack)**이 생긴다. 스택은 그 템플릿으로 만든 리소스들을 **하나의 단위로** 관리한다.

```
템플릿 app.yaml ──배포──▶ 스택 "app-dev"
                           ├─ LogBucket   → 실제 S3 버킷 (app-dev-logbucket-1a2b3c)
                           └─ AppLogGroup → 실제 log group (/app/dev)
```

- 같은 템플릿으로 파라미터만 바꿔 `app-dev`, `app-prod` 스택을 따로 만들 수 있다.
- 템플릿을 고쳐 다시 배포하면 스택이 **업데이트**된다. CloudFormation이 달라진 부분만 고친다.
- 스택을 지우면 그 스택이 만든 리소스가 **모두 삭제**된다(보존 설정을 한 것은 제외).
- 실제 리소스의 ID(버킷 이름, 인스턴스 ID 등)를 **물리 ID(physical ID)**라고 한다.

## 4단계: 배포 흐름

출처: [How CloudFormation works](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-overview.html)

```
템플릿 작성 → (선택) change set으로 변경 미리 보기 → 배포
                                                  │
              ┌───────────────────────────────────┤
              ▼                                   ▼
   서로 의존하지 않는 리소스는 병렬로 생성      실패하면 만든 것을 되돌림(rollback)
              │
              ▼
   CREATE_COMPLETE / UPDATE_COMPLETE
```

| 개념 | 뜻 |
| --- | --- |
| **Change set** | 업데이트 전에 "무엇이 추가·수정·**교체**·삭제될지" 미리 보여 주는 변경 요약 |
| **Rollback** | 배포 중 실패하면 이전 상태로 자동 복구 |
| **스택 상태** | `CREATE_COMPLETE`, `UPDATE_ROLLBACK_COMPLETE`, `ROLLBACK_COMPLETE` 등. 상태를 보면 지금 무엇을 할 수 있는지 알 수 있음 |
| **Drift** | 누군가 콘솔에서 직접 바꿔 실제 리소스가 템플릿과 달라진 상태 |

- 로컬 템플릿 파일로 배포하면 CloudFormation이 리전마다 S3 버킷을 만들어 템플릿을 올린다. 요청 본문으로 보낼 수 있는 템플릿은 51,200바이트까지이고, S3에 올린 템플릿은 1 MB까지다.
- 리소스는 의존 관계가 없으면 **병렬로** 만들어진다.

자주 마주치는 상태:

| 상태 | 뜻 |
| --- | --- |
| `CREATE_COMPLETE` / `UPDATE_COMPLETE` | 정상 |
| `ROLLBACK_COMPLETE` | **생성이 실패해 되돌린 상태. 삭제만 가능** |
| `UPDATE_ROLLBACK_COMPLETE` | 업데이트가 실패해 이전 상태로 돌아옴(다시 업데이트 가능) |
| `UPDATE_ROLLBACK_FAILED` | 되돌리기마저 실패. 복구 절차 필요 |
| `DELETE_FAILED` | 삭제 실패(예: 비어 있지 않은 S3 버킷) |

스택 운영은 [03-stack-operations.md](./03-stack-operations.md)에서 다룬다.

## 5단계: 여러 계정·리전에 배포 — StackSets

조직의 모든 계정에 같은 설정(로그 수집, 보안 role 등)을 깔고 싶다면 **StackSets**를 쓴다. 템플릿 하나를 **여러 계정 × 여러 리전**에 한 번에 배포한다. Organizations와 연동하면 새 계정이 들어올 때 자동으로 배포된다.

지금까지 본 AWS 서비스 중에도 StackSets를 쓰는 것이 있다.

- Systems Manager **Quick Setup**([aws-systems-manager/06](../aws-systems-manager/06-quick-setup.md))
- **Control Tower**([aws-organizations/05](../aws-organizations/05-other-policies.md))

자세한 내용은 [04-stacksets.md](./04-stacksets.md)에서 다룬다.

## 6단계: 템플릿을 쓰는 다른 방법들

| 도구 | 무엇 | 관계 |
| --- | --- | --- |
| **AWS CDK** | TypeScript·Python 등 **프로그래밍 언어로 인프라를 쓰면** CloudFormation 템플릿을 만들어 주는 도구 | 결국 CloudFormation으로 배포 |
| **AWS SAM** | 서버리스(Lambda, API Gateway)용 축약 문법 | 템플릿 안의 `Transform`으로 CloudFormation이 펼쳐서 배포 |
| **Template studio** | CloudFormation 콘솔 안의 시각 편집기 | 콘솔용. 예전 Infrastructure Composer를 대체 |
| **IaC generator** | 이미 콘솔로 만든 리소스를 스캔해 템플릿을 만들어 줌 | 기존 환경을 IaC로 옮길 때 |
| **Terraform** | HashiCorp의 IaC 도구. 여러 클라우드 지원 | CloudFormation과 별개(경쟁 도구) |

Infrastructure Composer는 2026-10-29부터 신규 고객을 받지 않고 2026-12-07에 지원이 끝난다. ([End of support](https://docs.aws.amazon.com/infrastructure-composer/latest/dg/infrastructure-composer-end-of-support.html))

CDK와 Terraform 비교는 [05-cdk-terraform.md](./05-cdk-terraform.md)에서 다룬다.

## 7단계: 요금

출처: [CloudFormation pricing](https://aws.amazon.com/cloudformation/pricing/)

| 대상 | 요금 |
| --- | --- |
| `AWS::*`, `Alexa::*` 리소스 | **추가 요금 없음**. 만든 리소스(EC2, S3 등)의 요금만 냄 |
| AWS 관리형 Hooks | 추가 요금 없음 |
| 서드파티 리소스 타입, 직접 만든 Hooks | handler 작업 1회당 $0.0009, 30초 초과 시 초당 $0.00008. 월 1,000회 무료 |
| Transform(SAM 등) | 추가 요금 없음 |

즉 일반적인 AWS 리소스만 다룬다면 CloudFormation 자체 비용은 0이다. **비용은 만들어진 리소스에서 나온다.** 스택을 지울 때 보존(`Retain`)으로 남긴 리소스도 계속 과금된다.

## 용어 정리

| 용어 | 뜻 | 처음 나오는 곳 |
| --- | --- | --- |
| IaC | 인프라를 코드 파일로 정의해 도구가 만들게 하는 방식 | 1단계 |
| 템플릿 | 만들 리소스를 적은 YAML/JSON 파일 | 2단계 |
| 논리 ID / 물리 ID | 템플릿 안의 리소스 이름 / 실제 만들어진 리소스 ID | 2·3단계 |
| Parameters / Outputs | 배포 시 입력값 / 배포 후 꺼내는 값 | 2단계 |
| 내장 함수 | `!Ref`, `!Sub` 등 템플릿 안에서 값을 계산·참조하는 함수 | 2단계 |
| 스택 | 템플릿 하나로 만든 리소스 묶음 | 3단계 |
| Change set | 업데이트 전 변경 미리 보기 | 4단계 |
| Rollback | 실패 시 이전 상태로 자동 복구 | 4단계 |
| Drift | 템플릿 밖에서 바뀌어 실제와 템플릿이 달라진 상태 | 4단계 |
| StackSets | 템플릿을 여러 계정·리전에 한 번에 배포 | 5단계 |
| CDK / SAM | 프로그래밍 언어로 템플릿 생성 / 서버리스용 축약 문법 | 6단계 |
| IaC generator | 기존 리소스로 템플릿을 만들어 주는 기능 | 6단계 |
| Terraform | HashiCorp의 다중 클라우드 IaC 도구 | 6단계 |
| Hooks | 리소스를 만들기 전에 규칙을 검사하는 기능 | 7단계 |

## 출처

- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/Welcome.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-overview.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-anatomy.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/view-stack-events.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/what-is-cfnstacksets.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-studio.html
- https://docs.aws.amazon.com/infrastructure-composer/latest/dg/infrastructure-composer-end-of-support.html
- https://aws.amazon.com/cloudformation/pricing/

---
*조사 기준일: 2026-10-14. 템플릿 예시는 cfn-lint 1.57.2로 검사해 통과했다. 다이어그램은 이해를 돕기 위한 예시다.*
