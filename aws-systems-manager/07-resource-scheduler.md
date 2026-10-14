# Resource Scheduler

> Resource Scheduler는 Systems Manager **Quick Setup**의 설정 유형 하나로, **태그로 고른 EC2 인스턴스를 정해진 시간에 자동으로 켜고 끈다.** 여러 계정·리전에 한 번에 적용할 수 있고, Quick Setup 자체는 무료다. 다만 **EC2 전용**이고, 설정 하나당 리전별 5,000대까지이며, 정해진 시각에만 동작하는 이벤트 방식이라 사람이 중간에 켜고 끈 상태는 다음 예약 시각까지 그대로 둔다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)와 [05-run-command-automation.md](./05-run-command-automation.md)의 용어(Automation runbook, State Manager 등)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Quick Setup | AWS가 권장하는 Systems Manager 구성을 여러 계정·리전에 한 번에 배포하는 기능. Resource Scheduler, DHMC, Patch Policy 등 설정 유형(configuration type)이 있음 |
| Configuration manager | Quick Setup에서 만든 설정 하나. API·CLI·Terraform은 이 단위로 다룸 |
| Change Calendar | Systems Manager에서 "변경 허용(OPEN)·금지(CLOSED)" 기간을 달력으로 정의하는 기능 |
| iCalendar(iCal) | 일정을 주고받는 표준 텍스트 형식(`.ics`). Change Calendar가 이 형식을 씀 |
| Overnight schedule | 시작 시각이 종료 시각보다 늦은 일정(예: 21시 시작 → 다음 날 7시 종료) |
| OU | AWS Organizations에서 계정을 묶는 폴더([aws-organizations/01](../aws-organizations/01-overview.md)) |

## 개요

출처: [Stop and start EC2 instances automatically on a schedule using Quick Setup](https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-scheduler.html)

개발·테스트 서버처럼 업무 시간에만 쓰는 인스턴스를 밤과 주말에 꺼서 비용을 줄이는 용도다. AWS 문서의 예시는 이렇다. 주 5일·하루 10시간만 쓰는 인스턴스를 계속 켜 두는 대신 일정에 맞춰 끄면, 실행 시간이 주 168시간에서 50시간으로 줄어 **해당 인스턴스 비용이 약 70% 줄어든다.**

| 항목 | 내용 |
| --- | --- |
| 대상 | **EC2 인스턴스만**. RDS, Auto Scaling 그룹 등은 대상이 아님 |
| 고르는 방법 | **태그 키·값 하나**(예: `Schedule=office-hours`). 일치하는 인스턴스만 켜고 끔 |
| 범위 | 현재 계정(현재 리전 또는 여러 리전), 또는 Organizations의 OU 여러 개 × 리전 여러 개 |
| 일정 | 시간대, 요일, 시작·종료 시각 |
| 한도 | 설정 하나당 **리전별 최대 5,000대**. 넘으면 서로 다른 태그 값으로 설정을 나눔 |
| 비용 | Quick Setup 자체는 무료. 설정이 만드는 리소스와 사용하는 서비스 요금은 별도(아래 [주의](#이슈--주의) 참고) |

## 동작 규칙

출처: 위와 같음

| 규칙 | 설명 |
| --- | --- |
| **상태가 맞을 때만 동작** | 시작 시각에는 `Stopped`인 인스턴스만 켜고, 종료 시각에는 `running`인 인스턴스만 끈다 |
| **이벤트 방식** | 정해진 시각에만 동작한다. 9시에 켠 뒤 누군가 10시에 수동으로 끄면 다시 켜지 않는다. 반대로 예약 종료 후 수동으로 켠 인스턴스도 다시 끄지 않는다(다음 예약 시각까지) |
| **Overnight 일정** | 시작 시각이 종료 시각보다 늦으면 밤새 실행으로 본다. 21시 시작·7시 종료면 선택한 요일 21시에 켜고 **다음 날** 7시에 끈다 |
| **생성 즉시 상태 맞춤** | 설정을 만드는 순간 현재 시각이 실행 시간대 밖이면 인스턴스를 끄고, 안이면 켠다. 운영 중인 인스턴스에 실수로 같은 태그가 붙어 있으면 바로 꺼질 수 있다 |
| 삭제하면 | 설정을 지우면 더 이상 켜고 끄지 않는다(인스턴스는 마지막 상태로 남음) |
| 실패 가능성 | AWS 문서도 드물게 API 실패로 켜기·끄기가 안 될 수 있다고 적고 있다 |

- 인스턴스에 붙은 **EBS 볼륨이 KMS로 암호화되어 있으면**, Resource Scheduler가 쓰는 IAM role에 그 KMS 키 권한을 추가해야 인스턴스를 켤 수 있다. 권한이 없으면 끄기는 되는데 켜기가 실패한다.

## 내부적으로 쓰는 것

Quick Setup이 Resource Scheduler용으로 쓰는 AWS 관리형 정책에서 확인한 구성 요소다. ([AWSQuickSetupStartStopInstancesExecutionPolicy](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AWSQuickSetupStartStopInstancesExecutionPolicy.html), [AWS managed policies for Systems Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/security-iam-awsmanpol.html))

| 구성 요소 | 정책에서 보이는 이름 | 역할(정책 내용으로 본 추정) |
| --- | --- | --- |
| Change Calendar 문서 | `AWSQuickSetup-ChangeCalendar*` | 실행 시간대(OPEN/CLOSED)를 달력으로 표현 |
| Automation runbook | `AWSQuickSetupType-Scheduler-ChangeCalendarState` | 달력 상태 변화를 관리 |
| Automation runbook | `AWSQuickSetupType-Scheduler-ApplyInstanceState` | 태그가 맞는 인스턴스를 실제로 켜고 끔 |
| State Manager association | `ssm:StartAssociationsOnce` 권한 | runbook 실행을 연결·즉시 실행 |
| IAM role | `AWS-QuickSetup*` | 위 작업을 실행하는 role. `ec2:StartInstances`·`StopInstances` 권한 보유 |

달력 상태 변화 → association 실행 → runbook이 인스턴스 상태 적용이라는 흐름은 정책 내용으로 추정한 것이다. AWS 문서에 명시된 흐름은 아니다(확인 필요). 정책 자체는 2025-08-08에 만들어졌고 2026-02-12에 마지막으로 수정되었다.

Quick Setup이 만든 Change Calendar 문서와 role을 직접 고치면 설정이 깨질 수 있다. 변경은 Quick Setup 설정을 통해 한다.

## 사용법 / 예시

### 콘솔

출처: [Quick Setup scheduler](https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-scheduler.html)

1. 스케줄 대상 인스턴스에 태그를 붙인다.

   ```bash
   aws ec2 create-tags \
     --resources i-0123456789abcdef0 i-0fedcba9876543210 \
     --tags Key=Schedule,Value=office-hours
   ```

2. Systems Manager 콘솔 → **Quick Setup** → **Resource Scheduler** 카드에서 **Create**를 누른다.
3. **Instance tag**에 태그 키·값(`Schedule` / `office-hours`)을 넣는다.
4. **Schedule options**에서 시간대(`Asia/Seoul`), 요일(월–금), 시작·종료 시각(09:00 / 19:00)을 고른다.
5. **Targets**에서 범위를 고른다.
   - **Current account**: 현재 리전 또는 고른 리전들
   - **Custom**: 대상 OU와 리전들(Organizations 필요)
6. **Summary**를 확인하고 **Create**를 누른다.

### CLI·API

Quick Setup은 `ssm-quicksetup` API로도 만들 수 있다. Resource Scheduler의 설정 유형 이름은 `AWSQuickSetupType-Scheduler`다. ([ConfigurationDefinitionInput](https://docs.aws.amazon.com/quick-setup/latest/APIReference/API_ConfigurationDefinitionInput.html), [create-configuration-manager](https://docs.aws.amazon.com/cli/latest/reference/ssm-quicksetup/create-configuration-manager.html))

| 파라미터 | 필수 | 뜻 |
| --- | --- | --- |
| `TargetTagKey` / `TargetTagValue` | 필수 | 대상 인스턴스 태그 |
| `ICalendarString` | 필수 | 일정(iCalendar 형식 문자열) |
| `TargetRegions` | 필수 | 적용 리전(쉼표 구분) |
| `TargetAccounts` / `TargetOrganizationalUnits` | 둘 중 하나 | 현재 계정 단독 배포 / OU 대상 배포 |

단일 계정 배포에는 `LocalDeploymentAdministrationRoleArn`과 `LocalDeploymentExecutionRoleName`도 지정해야 한다(API 레퍼런스).

```bash
aws ssm-quicksetup create-configuration-manager \
  --name dev-office-hours \
  --configuration-definitions '[
    {
      "Type": "AWSQuickSetupType-Scheduler",
      "LocalDeploymentAdministrationRoleArn": "arn:aws:iam::111122223333:role/AWS-QuickSetup-LocalAdministrationRole",
      "LocalDeploymentExecutionRoleName": "AWS-QuickSetup-LocalExecutionRole",
      "Parameters": {
        "TargetTagKey": "Schedule",
        "TargetTagValue": "office-hours",
        "ICalendarString": "<iCalendar 문자열>",
        "TargetAccounts": "111122223333",
        "TargetRegions": "ap-northeast-2"
      }
    }
  ]'

aws ssm-quicksetup list-configuration-managers
aws ssm-quicksetup get-configuration-manager --manager-arn <manager-arn>
```

- **`ICalendarString`을 직접 만들기가 가장 까다롭다.** 콘솔은 요일·시각만 고르면 되지만, API에는 iCalendar 문자열을 넣어야 한다. Quick Setup이 기대하는 정확한 iCal 형식은 이번 조사에서 확인하지 못했다(확인 필요). 콘솔로 한 번 만든 뒤 `get-configuration-manager`로 생성된 값을 확인해 재사용하는 방법을 고려할 수 있지만, 이 방법도 검증하지 않았다.
- role 이름(`AWS-QuickSetup-Local…`)은 예시다. 실제로 쓸 role 이름은 Quick Setup 문서에서 확인한다.
- CLI 레퍼런스 페이지는 유형 이름을 `Amazon Web ServicesQuickSetupType-Scheduler`로 보여 준다. 이는 문서 렌더링 문제로 보이고, API 레퍼런스의 `AWSQuickSetupType-Scheduler`가 맞는 값이다(확인 필요).

### Terraform

공식 AWS 프로바이더(`hashicorp/aws`)의 `aws_ssmquicksetup_configuration_manager` 리소스로 만든다. ([Terraform registry 문서 원본](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/ssmquicksetup_configuration_manager.html.markdown))

```hcl
resource "aws_ssmquicksetup_configuration_manager" "scheduler" {
  name = "dev-office-hours"

  configuration_definition {
    type = "AWSQuickSetupType-Scheduler"
    # 단일 계정 배포라면 local_deployment_* 역할 2개도 지정
    parameters = {
      TargetTagKey    = "Schedule"
      TargetTagValue  = "office-hours"
      ICalendarString = file("${path.module}/office-hours.ics")
      TargetAccounts  = data.aws_caller_identity.current.account_id
      TargetRegions   = "ap-northeast-2"
    }
  }
}
```

- Terraform 문서에는 Patch Policy 예시만 있고 Scheduler 예시는 없다. 위 코드는 API 파라미터로 이 레포에서 조합한 것이며 실행해 보지 않았다.
- Terraform 문서 안에서 role 속성 이름이 예제(`local_deployment_administration_role_arn`)와 인자 설명(`local_deployment_administrator_role_arn`)에서 다르다. `terraform providers schema`로 실제 이름을 확인한다.

## 다른 방법과 비교

| 방법 | 대상 | 장점 | 단점 |
| --- | --- | --- | --- |
| **Resource Scheduler**(Quick Setup) | EC2 | 콘솔 몇 번 클릭으로 여러 계정·리전에 적용, 별도 인프라 없음 | EC2 전용, 태그 키·값 하나, 설정당 리전별 5,000대, iCal 문자열 관리 |
| [Instance Scheduler on AWS](https://docs.aws.amazon.com/solutions/instance-scheduler-on-aws/) | EC2, Auto Scaling 그룹, RDS | 여러 일정과 복잡한 규칙 | DynamoDB·Lambda·EventBridge를 직접 배포·운영(CloudFormation 솔루션) |
| EventBridge Scheduler → EC2 API 직접 호출 | AWS API로 할 수 있는 모든 것 | 단순하고 IaC로 관리하기 쉬움 | 계정·리전마다 일정을 따로 만들어야 함, 대상 선택 로직을 직접 구현 |
| Maintenance Window + `AWS-StopEC2Instance` 등 runbook | EC2 | Systems Manager 안에서 해결 | 시작·종료 창을 따로 관리 |

간단히 정리하면 이렇다. **EC2만 업무 시간에 켜고 끄면 되면 Resource Scheduler**, **RDS까지 포함하거나 일정이 복잡하면 Instance Scheduler on AWS**, **계정 하나에서 IaC로 단순하게 관리하고 싶으면 EventBridge Scheduler**를 쓴다.

## 이슈 / 주의

- **태그 충돌**: 운영 인스턴스에 같은 태그 키·값이 붙어 있으면 설정을 만드는 즉시, 또는 다음 종료 시각에 꺼진다. 스케줄 전용 태그 키(예: `Schedule`)를 따로 쓰고, 적용 전에 대상을 조회한다.

  ```bash
  aws ec2 describe-instances \
    --filters Name=tag:Schedule,Values=office-hours \
    --query 'Reservations[].Instances[].[InstanceId,State.Name,Tags[?Key==`Name`]|[0].Value]' \
    --output table
  ```

- **암호화된 EBS**: KMS 키 권한이 없으면 끄기만 되고 켜기는 실패한다. 아침에 서버가 안 켜져 있다면 가장 먼저 확인한다.
- **수동 조작은 다음 예약 시각까지 유지**된다. 야근하느라 수동으로 켠 인스턴스는 다음 종료 시각에 꺼진다. 반대로 예약 종료 후 켠 인스턴스는 그날 밤 내내 켜져 있다.
- **비용 효과의 한계**: 인스턴스를 꺼도 **EBS 볼륨, Elastic IP(연결 해제 시), 스냅샷 요금은 계속 나간다.** 70% 절감은 인스턴스 시간 요금 기준이다.
- **Automation 요금**: 스케줄 동작에 Automation runbook이 쓰인다. Automation은 step당 과금되고 현재 free tier가 없다([01-overview.md](./01-overview.md#요금-개요)). 실제로 한 번 켜고 끌 때 몇 step이 과금되는지는 확인하지 못했다(확인 필요). 인스턴스가 많고 여러 리전·계정이면 Cost Explorer에서 Systems Manager Automation 요금을 확인한다.
- **Auto Scaling 그룹 소속 인스턴스**: Resource Scheduler가 끄면 Auto Scaling이 비정상으로 보고 새 인스턴스를 띄울 수 있다. ASG는 desired capacity를 조정하는 다른 방법(예약된 조정, Instance Scheduler on AWS)을 쓴다. 이 동작은 ASG의 일반 동작에서 추론한 것이다(확인 필요).

## 출처

- https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-scheduler.html
- https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AWSQuickSetupStartStopInstancesExecutionPolicy.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/security-iam-awsmanpol.html
- https://docs.aws.amazon.com/quick-setup/latest/APIReference/API_ConfigurationDefinitionInput.html
- https://docs.aws.amazon.com/cli/latest/reference/ssm-quicksetup/create-configuration-manager.html
- https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/ssmquicksetup_configuration_manager.html.markdown
- https://docs.aws.amazon.com/solutions/instance-scheduler-on-aws/

---
*조사 기준일: 2026-10-14. CLI·Terraform 예시는 API 레퍼런스 파라미터로 조합했고 실제 계정에서 실행하지 않았다.*
