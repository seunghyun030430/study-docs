# StackSets와 멀티 계정 배포

> StackSets는 **템플릿 하나를 여러 계정 × 여러 리전에 한 번에 배포**하는 기능이다. 조직을 쓴다면 **service-managed 권한**을 고른다. Organizations와 연동해 role을 직접 만들 필요가 없고, **OU에 새 계정이 들어오면 자동 배포**된다. 다만 **management 계정에는 배포되지 않는다.** 대규모 배포에서는 동시 실행 수와 실패 허용 수(operation preferences)로 속도와 위험을 조절한다.

## 이 문서에서 새로 나오는 용어

[03-stack-operations.md](./03-stack-operations.md)와 [aws-organizations/01](../aws-organizations/01-overview.md)의 용어(OU, management 계정, 위임 관리자, trusted access)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| StackSet | 여러 계정·리전에 배포할 템플릿과 설정의 묶음(리전 리소스) |
| Administrator account | StackSet을 만들고 관리하는 계정 |
| Target account | 스택이 실제로 배포되는 계정 |
| Stack instance | "계정 하나 × 리전 하나"에 배포된 스택을 가리키는 참조 |
| Operation | StackSet에 대한 작업 한 번(생성, 업데이트, 인스턴스 추가·삭제) |
| Self-managed / service-managed 권한 | IAM role을 직접 만드는 방식 / Organizations와 연동해 AWS가 관리하는 방식 |
| Auto-deployment | 대상 OU에 계정이 들어오거나 나갈 때 자동으로 배포·삭제하는 기능 |

## 구조

출처: [StackSets concepts](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html)

```
Administrator account (management 계정 또는 위임 관리자)
└── StackSet "baseline-logging" (ap-northeast-2에 생성된 리전 리소스)
    ├── Stack instance: 계정 A × ap-northeast-2 → 스택
    ├── Stack instance: 계정 A × us-east-1      → 스택
    ├── Stack instance: 계정 B × ap-northeast-2 → 스택
    └── Stack instance: 계정 B × us-east-1      → 스택
```

- StackSet의 템플릿을 바꾸면 **모든 stack instance에 적용**된다.
- Stack instance는 스택이 없어도 존재할 수 있다(예: 생성 실패).
- StackSet은 인스턴스가 하나도 없어야 지울 수 있다. 인스턴스를 지울 때 **"스택 유지(retain stacks)"**를 고르면 스택은 계정에 남고 StackSet과의 연결만 끊긴다.

## 권한 모델

| 항목 | Self-managed | **Service-managed**(권장) |
| --- | --- | --- |
| 준비 | 관리 계정에 `AWSCloudFormationStackSetAdministrationRole`, 대상 계정마다 `AWSCloudFormationStackSetExecutionRole`을 **직접 생성** | management 계정에서 Organizations **trusted access**를 켜면 AWS가 role을 관리 |
| 대상 지정 | 계정 ID | **OU**(또는 계정) |
| 조직 밖 계정 | 가능 | 불가 |
| 새 계정 자동 배포 | 없음 | **있음** |
| 관리 주체 | 아무 계정 | management 계정 또는 **위임 관리자**(조직당 최대 5개) |
| management 계정 배포 | 가능 | **배포되지 않음** |
| 매크로 쓰는 템플릿 | 가능 | 지원 안 함 |

출처: [Self-managed](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs-self-managed.html), [Service-managed](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-associate-stackset-with-org.html), [Delegated administrator](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-delegated-admin.html), [Quotas](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html)

- 위임 관리자는 배포 권한을 모두 가지며, management 계정이 이를 제한할 수 없다. CLI에서는 `--call-as DELEGATED_ADMIN`을 붙인다.
- Systems Manager Quick Setup의 위임 관리자로 등록하면 StackSets 위임 관리자도 함께 된다([aws-systems-manager/06](../aws-systems-manager/06-quick-setup.md#위임-관리자)).

## Auto-deployment

출처: [Auto-deployment](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-manage-auto-deployment.html), [AutoDeployment API](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_AutoDeployment.html)

| 설정 | 동작 |
| --- | --- |
| `Enabled=true` | 대상 OU에 계정이 **추가되면 배포**, **제거되면 삭제** |
| `RetainStacksOnAccountRemoval=true` | 계정이 OU를 떠나도 스택(리소스)은 남김 |
| `RetainStacksOnAccountRemoval=false` | 계정이 떠나면 스택과 리소스를 삭제 |

- 계정을 다른 OU로 **옮기면** "삭제 후 생성"으로 처리된다.
- auto-deployment는 StackSet 전체에 적용되며 일부 OU·리전만 고를 수 없다.
- 특정 계정에 덮어쓴(override) 파라미터는 나중에 들어온 계정에는 적용되지 않는다.

> **삭제 주의**: `RetainStacksOnAccountRemoval=false`면 계정이 OU를 떠날 때 배포했던 리소스가 지워진다. 로그 버킷처럼 데이터가 있는 리소스를 배포하는 StackSet은 `true`로 두거나 템플릿에 `DeletionPolicy: Retain`을 둔다.

## 배포 속도와 실패 제어

출처: [StackSetOperationPreferences](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_StackSetOperationPreferences.html), [Concurrency mode](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/concurrency-mode.html), [ManagedExecution](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_ManagedExecution.html)

| 설정 | 뜻 | 기본값 |
| --- | --- | --- |
| `MaxConcurrentCount` / `MaxConcurrentPercentage` | 한 리전에서 동시에 배포할 계정 수(비율은 내림) | 1 |
| `FailureToleranceCount` / `Percentage` | 리전마다 허용할 실패 수. 넘으면 작업을 멈추고 남은 리전은 취소 | 0 |
| `RegionConcurrencyType` | 리전을 `SEQUENTIAL`(차례로) 또는 `PARALLEL`(동시에) | SEQUENTIAL |
| `RegionOrder` | 리전 배포 순서 | — |
| `ConcurrencyMode` | `STRICT_FAILURE_TOLERANCE`: 실제 동시 수가 최대 "실패 허용 + 1"로 줄어듦 / `SOFT_FAILURE_TOLERANCE`: 실패와 상관없이 최대 동시 수 유지 | STRICT |
| `ManagedExecution` | 겹치지 않는 작업을 동시에 실행하고 겹치는 작업은 대기열에 넣음 | 꺼짐(한 번에 하나) |

```
예: MaxConcurrentCount=10, FailureToleranceCount=5, STRICT 모드
→ 실제 동시 배포 수는 최대 6 (실패 허용 5 + 1)
```

처음 배포할 때는 `MaxConcurrentCount=1, FailureToleranceCount=0`으로 한 계정에서 문제가 없는지 확인한 뒤 늘린다. [aws-organizations/03](../aws-organizations/03-scp-rcp.md)의 "테스트 OU부터 점진적으로" 원칙과 같다.

## 사용법 / 예시

출처: [Create a service-managed StackSet](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-associate-stackset-with-org.html)

조직의 모든 계정에 "로그 보존 기간 30일인 log group"을 배포하는 예다.

```yaml
AWSTemplateFormatVersion: "2010-09-09"
Description: Baseline log group for every account
Resources:
  BaselineLogGroup:
    Type: AWS::Logs::LogGroup
    DeletionPolicy: Retain
    UpdateReplacePolicy: Retain
    Properties:
      LogGroupName: /org/baseline
      RetentionInDays: 30
```

```bash
# 1. StackSet 생성 (service-managed, 자동 배포)
aws cloudformation create-stack-set --stack-set-name baseline-logging \
  --template-body file://baseline.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=true

# 2. OU × 리전에 인스턴스 배포 — 처음엔 천천히
aws cloudformation create-stack-instances --stack-set-name baseline-logging \
  --deployment-targets OrganizationalUnitIds=ou-exam-ple12345 \
  --regions ap-northeast-2 us-east-1 \
  --operation-preferences MaxConcurrentCount=1,FailureToleranceCount=0,RegionConcurrencyType=SEQUENTIAL

# 3. 진행 상황
aws cloudformation list-stack-set-operations --stack-set-name baseline-logging
aws cloudformation list-stack-instances --stack-set-name baseline-logging \
  --query 'Summaries[].[Account,Region,StackInstanceStatus.DetailedStatus]' --output table
```

위임 관리자 계정에서 실행하면 각 명령에 `--call-as DELEGATED_ADMIN`을 붙인다.

## 상태 읽기

출처: [StackSets concepts](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html), [StackInstanceComprehensiveStatus](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_StackInstanceComprehensiveStatus.html)

| Stack instance 상태 | 뜻 | 대응 |
| --- | --- | --- |
| `CURRENT` | 최신 템플릿으로 배포됨 | — |
| `OUTDATED` | 생성·업데이트가 실패했거나 멈춤 | 원인(대상 계정의 SCP, 리소스 한도, 이름 충돌 등)을 고치고 재배포 |
| `INOPERABLE` | 인스턴스 삭제가 실패 | `RetainStacks=true`로 인스턴스를 지운 뒤, 대상 계정의 스택을 직접 정리 |

세부 상태에는 `PENDING`, `RUNNING`, `SUCCEEDED`, `FAILED`, `CANCELLED`, `SKIPPED_SUSPENDED_ACCOUNT` 등이 있다.

### StackSet drift

```bash
aws cloudformation detect-stack-set-drift --stack-set-name baseline-logging
```

인스턴스의 스택이 drift되면 그 인스턴스가, 인스턴스 하나라도 drift되면 StackSet 전체가 drift 상태가 된다. 대상 계정에서 스택에 직접 drift 검사를 하면 그 결과는 StackSet에서 보이지 않는다. ([StackSet drift](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-drift.html))

## 한도

출처: [CloudFormation quotas](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html)

| 항목 | 값 |
| --- | --- |
| 관리 계정당 StackSet | 1,000 |
| StackSet당 stack instance | 100,000 |
| 관리 계정·리전당 동시 stack instance 작업 | 10,000 |
| StackSet당 대기 작업 | 10,000 |

Quick Setup이 "대상 계정 수 × 리전 수가 10,000을 넘으면 실패"하는 것도 StackSets를 쓰기 때문이다([aws-systems-manager/06](../aws-systems-manager/06-quick-setup.md)).

## 이슈 / 주의

- **management 계정에는 service-managed StackSet이 배포되지 않는다.** management 계정에 필요한 리소스는 일반 스택으로 따로 만든다.
- **대상 계정의 SCP**가 StackSet이 만들려는 리소스를 막으면 그 인스턴스는 `OUTDATED`가 된다. 가드레일 SCP를 만들 때 StackSets 실행 role을 예외 처리할지 검토한다.
- **처음부터 높은 동시 수로 배포하지 않는다.** 템플릿 오류가 수백 개 계정에 동시에 퍼진다.
- **auto-deployment의 삭제 동작**을 이해하고 켠다. 계정 이동도 삭제 후 생성이다.
- **Control Tower 환경**에서는 Control Tower가 관리하는 StackSet(이름이 `AWSControlTower`로 시작하는 것 등)을 직접 바꾸지 않는다([aws-organizations/05](../aws-organizations/05-other-policies.md)). 정확한 이름 규칙은 확인하지 못했다(확인 필요).

## 출처

- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-concepts.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-prereqs-self-managed.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-associate-stackset-with-org.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-delegated-admin.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-orgs-manage-auto-deployment.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_AutoDeployment.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_StackSetOperationPreferences.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/concurrency-mode.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_ManagedExecution.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_StackInstanceComprehensiveStatus.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stacksets-drift.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html

---
*조사 기준일: 2026-10-14. 템플릿은 cfn-lint 1.57.2로 검사해 통과했다. CLI 예시는 공식 예시를 바탕으로 했고 실제 조직에서 실행하지 않았다.*
