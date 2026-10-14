# Quick Setup

> Quick Setup은 Systems Manager와 일부 AWS 서비스의 **권장 구성을 몇 번의 선택으로 한 계정 또는 조직 전체(여러 계정·리전)에 배포**하는 기능이다. 무료이고, 내부적으로 **CloudFormation StackSets로 배포**하며 대상마다 **State Manager association**을 만들어 실행한다. 설정 유형은 11가지다. 실무에서 자주 쓰는 것은 **Host Management**(노드 기본 관리), **DHMC**, **Patch Policy**(패치), **Resource Scheduler**(EC2 자동 켜기·끄기)다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)와 [05-run-command-automation.md](./05-run-command-automation.md)의 용어(Automation runbook, State Manager 등)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 설정 유형(configuration type) | Quick Setup이 배포할 수 있는 구성 묶음의 종류(Host Management, Patch Policy, Resource Scheduler 등) |
| Configuration manager | Quick Setup에서 만든 설정 하나. 콘솔 목록·API·Terraform은 이 단위로 다룸 |
| CloudFormation StackSets | CloudFormation 템플릿 하나를 여러 계정·리전에 한꺼번에 배포하는 기능. 배포된 각 사본을 stack instance라고 함 |
| State Manager association | "이 노드들에 이 문서를 이 주기로 실행해 원하는 상태를 유지하라"는 설정 |
| Drift | 사용자가 직접 바꿔서 Quick Setup이 정한 구성과 달라진 상태 |
| Local / Organizational 배포 | 현재 계정에만 배포 / AWS Organizations의 OU·리전 여러 개에 배포 |
| OU | AWS Organizations에서 계정을 묶는 폴더([aws-organizations/01](../aws-organizations/01-overview.md)) |
| Change Calendar | 변경 허용(OPEN)·금지(CLOSED) 기간을 달력으로 정의하는 Systems Manager 기능 |
| iCalendar(iCal) | 일정을 주고받는 표준 텍스트 형식(`.ics`) |

## 개요

출처: [AWS Systems Manager Quick Setup](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-quick-setup.html)

> "Use Quick Setup to quickly configure frequently used Amazon Web Services services and features with recommended best practices." — AWS Systems Manager Quick Setup

| 항목 | 내용 |
| --- | --- |
| 하는 일 | instance profile role 생성, 정기 패치 스캔, 인벤토리 수집 같은 흔한 설정 작업을 자동화 |
| 범위 | 한 계정, 또는 Organizations와 연동해 여러 계정·리전 |
| Drift 처리 | 주기적으로 drift를 검사하고 **원래 구성으로 되돌리려 시도** |
| 비용 | Quick Setup 자체는 **무료**. 설정한 서비스(Config, Automation 등)의 사용 요금은 별도 |
| 권장 대상 | 설정할 서비스를 어느 정도 아는 사람. 모르는 서비스라면 해당 서비스 문서부터 보라고 AWS가 권함 |
| 리전 | 서울을 포함한 16개 리전에서 모든 유형을 조직·단일 계정 모두에 사용 가능. 밀라노·홍콩·바레인 등 일부 리전에서는 단일 계정의 Host Management만 가능 |

## 동작 방식

출처: [Getting started with Quick Setup](https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-getting-started.html), [Quick Setup terminology](https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-using.html)

```
Quick Setup 콘솔 / ssm-quicksetup API
        │  configuration manager 생성 (유형 + 옵션 + 대상)
        ▼
CloudFormation StackSets ──▶ 대상 계정 × 리전마다 stack instance 배포
                                    │
                                    ▼
                    State Manager association 생성·실행
                    (예: 2주마다 SSM Agent 업데이트, 30분마다 인벤토리 수집)
                                    │
                                    ▼
             상태 표시: Deployment status / Association status / Resource compliance
```

| 항목 | 내용 |
| --- | --- |
| 배포 엔진 | **CloudFormation StackSets**. 대상 계정 수 × 리전 수가 **10,000을 넘으면 배포 실패** |
| management 계정 | 조직 배포 시 **stack instance가 management 계정에는 배포되지 않음** |
| 실행 단위 | 대상마다 State Manager association을 만들어 실행. 하나라도 실패하면 association status는 `Failed` |
| 상태 | Deployment status(CloudFormation 배포 성공 여부), Association status, Setup status, Resource compliance |
| 수정 | 옵션을 추가하면 association을 새로 만들고, 빼면 지움. 조직 설정은 상태가 `Success`나 `Failed`일 때만 수정 가능 |
| 대상 변경 | OU·리전 추가·제거 가능(management 계정만). 제거하면 그 OU·리전의 association도 제거 |
| 버전 | AWS가 설정 유형의 새 버전을 내면 **Upgrade configuration**으로 올림 |
| 삭제 | "모든 OU·리전 제거 → 설정 삭제" 두 단계. 배포된 리소스와 association이 모두 지워짐 |
| 준수 현황 | Systems Manager의 Explorer나 Compliance에서 확인 |

### 처음 쓸 때 만들어지는 IAM role

콘솔로 처음 들어가면 Quick Setup이 아래 role을 대신 만든다(온보딩). API·CLI만으로 쓰려면 이 과정을 직접 해야 한다.

| Role | 용도 |
| --- | --- |
| `AWS-QuickSetup-LocalAdministrationRole` | CloudFormation이 실행 role을 맡게 함 |
| `AWS-QuickSetup-LocalExecutionRole` | CloudFormation이 템플릿(Patch Policy 제외)으로 리소스를 만듦 |
| `AWS-QuickSetup-PatchPolicy-LocalAdministrationRole` / `-LocalExecutionRole` | Patch Policy 전용 |
| `AWS-QuickSetup-SSM-RoleForEnablingExplorer` | management 계정 온보딩 시. Explorer 활성화 runbook(`AWS-EnableExplorer`)용 |

management 계정을 온보딩하면 Organizations와 CloudFormation 사이의 **trusted access**도 켠다. 이때 관리자 권한이 필요하지만, 온보딩이 끝나면 더는 필요하지 않다.

API로 직접 온보딩하는 순서(모두 management 계정에서 실행):

1. CloudFormation `ActivateOrganizationsAccess`
2. Organizations `EnableAWSServiceAccess`(`ssm.amazonaws.com`)
3. Explorer용 role 생성(`AWSSystemsManagerEnableExplorerExecutionPolicy`)
4. Quick Setup `UpdateServiceSettings`로 그 role 지정
5. StackSets용 administration·execution role 생성(`AWSQuickSetupDeploymentRolePolicy`)

## 설정 유형

출처: [Supported configuration types](https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-config-types.html), [ConfigurationDefinitionInput](https://docs.aws.amazon.com/quick-setup/latest/APIReference/API_ConfigurationDefinitionInput.html)

"범위"는 API 필수 파라미터로 판단했다. `TargetOrganizationalUnits`가 필수인 유형은 조직 배포 전용이다.

| 설정 유형 | API `Type` | 하는 일 | 범위 |
| --- | --- | --- | --- |
| **Host Management** | `AWSQuickSetupType-SSMHostMgmt` | EC2를 managed node로 만들고 기본 관리 작업 자동화 | 계정 / 조직 |
| **Default Host Management Configuration** | `AWSQuickSetupType-DHMC` | 조직 전체에 DHMC 적용([02](./02-architecture.md#iam-노드가-ssm에-말할-권한)) | **조직 전용** |
| **Patch Policy** | `AWSQuickSetupType-PatchPolicy` | 패치 스캔·설치 일정과 baseline을 정책으로 배포 | 계정 / 조직 |
| **Resource Scheduler** | `AWSQuickSetupType-Scheduler` | 태그로 고른 EC2를 일정에 맞춰 켜고 끔([아래](#resource-scheduler)) | 계정 / 조직 |
| Distributor | `AWSQuickSetupType-Distributor` | 노드에 패키지 설치(EFS 도구, CloudWatch agent, EC2Launch agent) | 계정 / 조직 |
| AWS Config 기록 | `AWSQuickSetupType-CFGRecording` | AWS Config configuration recorder 생성 | 계정 / 조직 |
| AWS Config conformance pack | `AWSQuickSetupType-CFGCPacks` | Config 규칙 묶음(conformance pack) 배포 | 계정 / 조직 |
| DevOps Guru | `AWSQuickSetupType-DevOpsGuru` | DevOps Guru(ML 기반 운영 이상 탐지) 활성화 | 계정 / 조직 |
| OpsCenter | `AWSQuickSetupType-SSMOpsCenter` | 위임 관리자 계정에서 조직의 OpsItem을 모아 관리 | **조직 전용** |
| Resource Explorer | `AWSQuickSetupType-ResourceExplorer` | 계정 간 리소스 검색용 aggregator index 구성 | **조직 전용** |
| Change Manager | `AWSQuickSetupType-SSMChangeMgr` | 변경 요청·승인 조직 설정 | **조직 전용**. Change Manager는 2025-11-07부터 신규 고객 가입 중단([01](./01-overview.md)) |

### Host Management

출처: [Set up Amazon EC2 host management](https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-host-management.html), ConfigurationDefinitionInput

인스턴스가 managed node가 되지 않을 때 AWS가 가장 먼저 안내하는 설정이다. 아래 옵션은 API 기본값이다.

| 옵션 | 기본값 | 내용 |
| --- | --- | --- |
| `UpdateSsmAgent` | 켜짐 | **2주마다** SSM Agent 업데이트 |
| `CollectInventory` | 켜짐 | **30분마다** 인벤토리(설치 소프트웨어 등) 수집 |
| `ScanInstances` | 켜짐 | **매일** 누락 패치 스캔(설치는 안 함) |
| `InstallCloudWatchAgent` / `UpdateCloudWatchAgent` | 꺼짐 | CloudWatch agent 설치 / 매월 업데이트 |
| `UpdateEc2LaunchAgent` | 꺼짐 | 매월 EC2Launch agent 업데이트 |
| `IsPolicyAttachAllowed` | 꺼짐 | 이미 instance profile이 붙은 인스턴스에 필요한 정책을 추가로 붙일지 |
| `TargetType` | — | `*`(전체), `InstanceIds`, `Tags`, `ResourceGroups` 중 대상 선택 방식 |

`IsPolicyAttachAllowed`가 꺼져 있으면, 기존 instance profile에 SSM 권한이 없는 인스턴스는 managed node가 되지 않을 수 있다. 이 동작은 옵션 설명에서 추론한 것이다(확인 필요).

### Default Host Management Configuration(DHMC)

[02-architecture.md](./02-architecture.md#iam-노드가-ssm에-말할-권한)에서 다룬 DHMC를 조직의 OU·리전 전체에 켜는 설정이다. `UpdateSsmAgent`(기본 켜짐, 2주마다)도 함께 설정할 수 있다. IMDSv2와 SSM Agent 3.2.582.0 이상이 필요하다는 DHMC 조건은 그대로 적용된다.

### Patch Policy

출처: [Configure patching using a patch policy](https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-patch-manager.html)

| 옵션 | 내용 |
| --- | --- |
| `ConfigurationOptionsPatchOperation` | `Scan`(기본, 스캔만) 또는 `ScanAndInstall` |
| `ConfigurationOptionsScanValue` / `InstallValue` | 스캔·설치 일정(cron 식) |
| `SelectedPatchBaselines` | OS별로 쓸 patch baseline(AWS 기본 또는 직접 만든 것) |
| `RebootOption` | `RebootIfNeeded` 또는 `NoReboot` |
| `OutputLogEnableS3` | 실행 로그를 S3에 저장 |

- Patch Policy는 **Quick Setup 위임 관리자로 관리할 수 없다.** 조직용 patch policy는 management 계정에서 만들고 유지해야 한다.
- 전용 role(`AWS-QuickSetup-PatchPolicy-*`)을 쓴다.

### 그 밖의 유형

| 유형 | 주요 옵션(API) |
| --- | --- |
| Distributor | `PackagesToInstall`: `AWSEFSTools`, `AWSCWAgent`, `AWSEC2LaunchAgent`. `RemediationSchedule` 기본 `rate(30 days)` |
| AWS Config 기록 | 모든 리소스 기록 여부(기본 켜짐), 전역 리소스 기록, 전달 S3 버킷, SNS 알림 |
| Conformance pack | `CPackNames`(배포할 pack 목록), 조직 배포 시 위임 관리자 계정 |
| DevOps Guru | 분석 대상, SNS 알림(기본 켜짐), OpsItem 생성(기본 켜짐) |
| OpsCenter | 위임 관리자 계정 ID(필수) |
| Resource Explorer | aggregator index를 둘 리전, 기존 aggregator 교체 여부 |

`RemediationSchedule`(drift 복구 주기)은 `rate(30 days)` 같은 값이나 `none`으로 설정한다. 유형마다 허용 값과 기본값이 다르다.

## Resource Scheduler

출처: [Stop and start EC2 instances automatically on a schedule using Quick Setup](https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-scheduler.html)

개발·테스트 서버처럼 업무 시간에만 쓰는 EC2를 밤과 주말에 꺼서 비용을 줄이는 설정이다. AWS 문서의 예시는 이렇다. 주 5일·하루 10시간만 쓰는 인스턴스를 일정에 맞춰 끄면, 실행 시간이 주 168시간에서 50시간으로 줄어 **해당 인스턴스 비용이 약 70% 줄어든다.**

| 항목 | 내용 |
| --- | --- |
| 대상 | **EC2 인스턴스만**. RDS, Auto Scaling 그룹 등은 대상이 아님 |
| 고르는 방법 | **태그 키·값 하나**(예: `Schedule=office-hours`) |
| 범위 | 현재 계정(현재 리전 또는 여러 리전), 또는 OU 여러 개 × 리전 여러 개 |
| 일정 | 시간대, 요일, 시작·종료 시각 |
| 한도 | 설정 하나당 **리전별 최대 5,000대**. 넘으면 서로 다른 태그 값으로 설정을 나눔 |

### 동작 규칙

| 규칙 | 설명 |
| --- | --- |
| **상태가 맞을 때만 동작** | 시작 시각에는 `Stopped`인 인스턴스만 켜고, 종료 시각에는 `running`인 인스턴스만 끈다 |
| **이벤트 방식** | 정해진 시각에만 동작한다. 9시에 켠 뒤 누군가 수동으로 끄면 다시 켜지 않는다. 예약 종료 후 수동으로 켠 인스턴스도 다음 예약 시각까지 그대로 둔다 |
| **Overnight 일정** | 시작 시각이 종료 시각보다 늦으면 밤새 실행으로 본다. 21시 시작·7시 종료면 선택한 요일 21시에 켜고 **다음 날** 7시에 끈다 |
| **생성 즉시 상태 맞춤** | 설정을 만드는 순간 현재 시각이 실행 시간대 밖이면 끄고, 안이면 켠다 |
| 삭제하면 | 더 이상 켜고 끄지 않는다(인스턴스는 마지막 상태로 남음) |
| 실패 가능성 | 드물게 API 실패로 켜기·끄기가 안 될 수 있다고 AWS 문서에 명시 |

인스턴스에 붙은 **EBS 볼륨이 KMS로 암호화되어 있으면**, Resource Scheduler가 쓰는 IAM role에 그 KMS 키 권한을 추가해야 인스턴스를 켤 수 있다.

### 내부적으로 쓰는 것

Resource Scheduler용 AWS 관리형 정책에서 확인한 구성 요소다. ([AWSQuickSetupStartStopInstancesExecutionPolicy](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AWSQuickSetupStartStopInstancesExecutionPolicy.html), [AWS managed policies for Systems Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/security-iam-awsmanpol.html))

| 구성 요소 | 정책에서 보이는 이름 | 역할(정책 내용으로 본 추정) |
| --- | --- | --- |
| Change Calendar 문서 | `AWSQuickSetup-ChangeCalendar*` | 실행 시간대를 달력(OPEN/CLOSED)으로 표현 |
| Automation runbook | `AWSQuickSetupType-Scheduler-ChangeCalendarState` | 달력 상태 변화 관리 |
| Automation runbook | `AWSQuickSetupType-Scheduler-ApplyInstanceState` | 태그가 맞는 인스턴스를 실제로 켜고 끔 |
| State Manager association | `ssm:StartAssociationsOnce` 권한 | runbook 실행 연결·즉시 실행 |
| IAM role | `AWS-QuickSetup*` | `ec2:StartInstances`·`StopInstances` 권한 보유 |

달력 상태 변화 → association 실행 → runbook이 인스턴스 상태를 적용하는 흐름은 정책 내용으로 추정한 것이며, AWS 문서에 명시된 흐름은 아니다(확인 필요).

### 설정 방법

콘솔:

1. 대상 인스턴스에 태그를 붙인다.

   ```bash
   aws ec2 create-tags \
     --resources i-0123456789abcdef0 i-0fedcba9876543210 \
     --tags Key=Schedule,Value=office-hours
   ```

2. Systems Manager 콘솔 → **Quick Setup** → **Resource Scheduler** 카드에서 **Create**를 누른다.
3. **Instance tag**에 태그 키·값을 넣는다.
4. **Schedule options**에서 시간대(`Asia/Seoul`), 요일, 시작·종료 시각을 고른다.
5. **Targets**에서 **Current account** 또는 **Custom**(OU·리전)을 고르고 **Create**를 누른다.

API 파라미터:

| 파라미터 | 필수 | 뜻 |
| --- | --- | --- |
| `TargetTagKey` / `TargetTagValue` | 필수 | 대상 인스턴스 태그 |
| `ICalendarString` | 필수 | 일정(iCalendar 형식 문자열) |
| `TargetRegions` | 필수 | 적용 리전(쉼표 구분) |
| `TargetAccounts` / `TargetOrganizationalUnits` | 둘 중 하나 | 현재 계정 / OU |

`ICalendarString`의 정확한 형식은 확인하지 못했다(확인 필요). 콘솔로 만든 설정을 `get-configuration-manager`로 조회해 생성된 값을 재사용하는 방법을 고려할 수 있지만, 이것도 검증하지 않았다.

## 위임 관리자

출처: [Using a delegated administrator for Quick Setup](https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-delegated-administrator.html)

- management 계정은 member 계정 **하나**를 Quick Setup 위임 관리자로 지정할 수 있다.
- 위임 관리자는 OU를 대상으로 하는 configuration manager를 만들고, 수정·조회·삭제할 수 있다. management 계정이 예전에 만든 설정도 관리할 수 있다.
- Quick Setup 위임 관리자로 등록하면 **CloudFormation StackSets와 Systems Manager Explorer의 위임 관리자도 함께** 된다. 배포와 모니터링에 둘 다 필요하기 때문이다.
- **Patch Policy는 위임 관리자로 관리할 수 없다.** management 계정에서 관리한다.

management 계정 사용을 최소화하라는 권장([aws-organizations/02](../aws-organizations/02-structure-ou.md#management-계정-최소화))에 맞춰, Quick Setup도 위임 관리자 계정(보통 Infrastructure OU의 운영 도구 계정)에서 운영하는 것이 좋다.

## CLI·API·Terraform

Quick Setup은 `ssm-quicksetup` API로 다룬다. 모든 설정 유형이 같은 형식(`Type` + `Parameters`)을 쓴다. ([create-configuration-manager](https://docs.aws.amazon.com/cli/latest/reference/ssm-quicksetup/create-configuration-manager.html), [Quick Setup API](https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-api.html))

```bash
# 단일 계정에 Host Management 배포 (기본 옵션 + CloudWatch agent 설치)
aws ssm-quicksetup create-configuration-manager \
  --name host-mgmt-seoul \
  --configuration-definitions '[
    {
      "Type": "AWSQuickSetupType-SSMHostMgmt",
      "LocalDeploymentAdministrationRoleArn": "arn:aws:iam::111122223333:role/AWS-QuickSetup-LocalAdministrationRole",
      "LocalDeploymentExecutionRoleName": "AWS-QuickSetup-LocalExecutionRole",
      "Parameters": {
        "UpdateSsmAgent": "true",
        "CollectInventory": "true",
        "ScanInstances": "true",
        "InstallCloudWatchAgent": "true",
        "TargetType": "*",
        "TargetAccounts": "111122223333",
        "TargetRegions": "ap-northeast-2"
      }
    }
  ]'

aws ssm-quicksetup list-configuration-managers
aws ssm-quicksetup get-configuration-manager --manager-arn <manager-arn>
aws ssm-quicksetup delete-configuration-manager --manager-arn <manager-arn>
```

- 단일 계정 배포에는 `LocalDeploymentAdministrationRoleArn`과 `LocalDeploymentExecutionRoleName`을 반드시 넣는다. 조직 배포는 Patch Policy를 빼면 생략할 수 있다(API 레퍼런스).
- 읽기 전용 사용자에게는 `ssm-quicksetup:List*`, `ssm-quicksetup:Get*`만 준다.
- CLI 레퍼런스 페이지는 유형 이름을 `Amazon Web ServicesQuickSetupType-…`로 보여 준다. 문서 렌더링 문제로 보이며, API 레퍼런스의 `AWSQuickSetupType-…`가 맞는 값이다(확인 필요).
- 위 명령은 API 파라미터로 조합한 예시이며 실제 계정에서 실행하지 않았다.

Terraform은 공식 AWS 프로바이더의 `aws_ssmquicksetup_configuration_manager` 리소스를 쓴다. ([Terraform 문서 원본](https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/ssmquicksetup_configuration_manager.html.markdown))

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

- Terraform 문서에는 Patch Policy 예시만 있다. 위 Scheduler 예시는 API 파라미터로 조합한 것이며 실행해 보지 않았다.
- Terraform 문서 안에서 role 속성 이름이 예제(`local_deployment_administration_role_arn`)와 인자 설명(`local_deployment_administrator_role_arn`)에서 다르다. `terraform providers schema`로 확인한다.

## 대안 비교: EC2 켜기·끄기

| 방법 | 대상 | 장점 | 단점 |
| --- | --- | --- | --- |
| **Resource Scheduler**(Quick Setup) | EC2 | 콘솔 몇 번으로 여러 계정·리전 적용, 별도 인프라 없음 | EC2 전용, 태그 하나, 설정당 리전별 5,000대, iCal 문자열 관리 |
| [Instance Scheduler on AWS](https://docs.aws.amazon.com/solutions/instance-scheduler-on-aws/) | EC2, Auto Scaling 그룹, RDS | 여러 일정·복잡한 규칙 | DynamoDB·Lambda·EventBridge를 직접 배포·운영 |
| EventBridge Scheduler → EC2 API | AWS API로 가능한 모든 것 | 단순, IaC 친화 | 계정·리전마다 일정을 따로 만들고, 대상 선택 로직을 직접 구현 |
| Maintenance Window + `AWS-StopEC2Instance` 등 | EC2 | Systems Manager 안에서 해결 | 시작·종료 창을 따로 관리 |

## 이슈 / 주의

- **Quick Setup이 만든 리소스를 직접 고치지 않는다.** association, Change Calendar 문서, `AWS-QuickSetup-*` role을 직접 바꾸면 drift가 생긴다. Quick Setup이 되돌리거나 설정이 깨질 수 있으니, 변경은 Quick Setup 설정을 통해 한다.
- **StackSets 대상 수 한도**: 계정 수 × 리전 수가 10,000을 넘으면 배포가 실패한다. 큰 조직은 설정을 OU 단위로 나눈다.
- **management 계정에는 stack instance가 배포되지 않는다.** 조직 배포를 했는데 management 계정 인스턴스가 관리되지 않는 이유가 이것이다. management 계정에는 워크로드를 두지 않는 것이 원칙이다([aws-organizations/02](../aws-organizations/02-structure-ou.md)).
- **"무료"는 Quick Setup 자체만이다.** AWS Config 기록, Automation step 요금(현재 free tier 없음), CloudWatch agent가 보내는 메트릭·로그는 각각 과금된다. Resource Scheduler가 한 번 켜고 끌 때 몇 Automation step이 과금되는지는 확인하지 못했다(확인 필요).
- **Resource Scheduler 태그 충돌**: 운영 인스턴스에 같은 태그가 붙어 있으면 설정을 만드는 즉시 꺼질 수 있다. 적용 전에 대상을 조회한다.

  ```bash
  aws ec2 describe-instances \
    --filters Name=tag:Schedule,Values=office-hours \
    --query 'Reservations[].Instances[].[InstanceId,State.Name]' \
    --output table
  ```

- **Resource Scheduler와 암호화된 EBS**: KMS 키 권한이 없으면 끄기만 되고 켜기는 실패한다.
- **인스턴스를 꺼도** EBS 볼륨, 스냅샷 요금은 계속 나간다. 70% 절감은 인스턴스 시간 요금 기준이다.
- **Auto Scaling 그룹 소속 인스턴스**를 Resource Scheduler로 끄면 Auto Scaling이 새 인스턴스를 띄울 수 있다. ASG는 예약된 조정이나 Instance Scheduler on AWS를 쓴다. 이 동작은 ASG의 일반 동작에서 추론한 것이다(확인 필요).

## 출처

- https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-quick-setup.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-getting-started.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-using.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-config-types.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-host-management.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-patch-manager.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-scheduler.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-delegated-administrator.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/quick-setup-api.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/security-iam-awsmanpol.html
- https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AWSQuickSetupStartStopInstancesExecutionPolicy.html
- https://docs.aws.amazon.com/quick-setup/latest/APIReference/API_ConfigurationDefinitionInput.html
- https://docs.aws.amazon.com/cli/latest/reference/ssm-quicksetup/create-configuration-manager.html
- https://github.com/hashicorp/terraform-provider-aws/blob/main/website/docs/r/ssmquicksetup_configuration_manager.html.markdown
- https://docs.aws.amazon.com/solutions/instance-scheduler-on-aws/

---
*조사 기준일: 2026-10-14. 설정 유형별 옵션과 기본값은 Quick Setup API 레퍼런스 기준이다. CLI·Terraform 예시는 실제 계정에서 실행하지 않았다.*
