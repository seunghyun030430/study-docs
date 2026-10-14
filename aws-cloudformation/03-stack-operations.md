# 스택 운영

> 템플릿을 바꾸기 전에는 **change set으로 무엇이 교체(Replacement)되는지 먼저 본다.** 교체는 새 리소스를 만들고 옛 것을 지우는 것이라 DB·볼륨이면 데이터가 사라질 수 있다. 데이터가 있는 리소스에는 **`DeletionPolicy`와 `UpdateReplacePolicy`를 `Retain`이나 `Snapshot`으로** 두고, 운영 스택에는 **termination protection**을 켠다. 실패하면 자동 rollback되지만, rollback마저 실패하는 `UPDATE_ROLLBACK_FAILED` 상태를 알아 둬야 한다. 콘솔에서 직접 바꾼 차이는 **drift detection**으로 찾는다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)와 [02-templates.md](./02-templates.md)의 용어(템플릿, 스택, 리소스, 논리 ID, change set 등)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Physical ID | 실제로 만들어진 리소스의 ID(예: `i-0123…`, 버킷 이름) |
| Replacement | 속성을 바꾸려면 리소스를 새로 만들어야 해서, 새 것을 만들고 옛 것을 지우는 업데이트 |
| Rollback | 실패한 작업을 되돌려 이전 상태로 복구하는 것 |
| Rollback trigger | 배포 중·후에 CloudWatch 알람이 울리면 자동으로 rollback하는 설정 |
| Drift | 콘솔·CLI로 직접 바꿔서 실제 리소스가 템플릿과 달라진 상태 |
| DeletionPolicy / UpdateReplacePolicy | 스택 삭제 시 / 교체 시 옛 리소스를 지울지·남길지·스냅샷할지 정하는 속성 |
| Stack policy | 업데이트 때 특정 리소스의 수정·교체·삭제를 막는 JSON 정책 |
| Service role | CloudFormation이 리소스를 만들 때 대신 쓰는 IAM role |
| Resource import | 콘솔로 이미 만든 리소스를 스택 관리 아래로 가져오는 것 |

## 업데이트 동작: 수정인가, 교체인가

출처: [Update behaviors](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-update-behaviors.html)

템플릿의 속성 하나를 바꾸면 리소스마다 세 가지 중 하나가 일어난다. 어떤 것인지는 Template Reference의 속성 설명에 있는 **"Update requires"** 줄에 적혀 있다.

| 동작 | 뜻 | 예 |
| --- | --- | --- |
| **No interruption** | 멈춤 없이 수정. physical ID 그대로 | 태그 변경 등 |
| **Some interruption** | 잠깐 멈췄다 수정(예: 재시작) | EC2 인스턴스 타입 변경 |
| **Replacement** | **새 리소스를 만들고, 참조를 옮기고, 옛 리소스를 삭제**. physical ID가 바뀜 | RDS `Port` 변경, EC2 `ImageId` 변경 |

> **데이터 손실 주의**: RDS·EBS·DynamoDB처럼 데이터를 가진 리소스가 Replacement되면 옛 리소스와 데이터가 지워진다. 배포 전에 change set으로 반드시 확인하고, `UpdateReplacePolicy`를 설정해 둔다.

- 값이 실제로 같더라도, 교체가 필요한 속성을 **추가하거나 빼기만 해도** 교체가 일어난다.
- **이름을 직접 정한 리소스는 교체할 수 없다.** 새 것과 옛 것이 같은 이름을 쓸 수 없기 때문이다. 교체가 필요하면 이름도 바꿔야 한다([Name type](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-properties-name.html)). 그래서 꼭 필요하지 않으면 이름을 지정하지 않는 편이 운영에 유리하다.

## Change set: 미리 보고 실행하기

출처: [create-change-set](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/create-change-set.html), [ResourceChange](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_ResourceChange.html), [deploy](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/deploy.html)

```bash
# 1. 변경 미리 만들기
aws cloudformation create-change-set \
  --stack-name app-prod --change-set-name upgrade-20261014 \
  --template-body file://app.yaml \
  --parameters ParameterKey=Env,ParameterValue=prod \
  --capabilities CAPABILITY_NAMED_IAM

# 2. 무엇이 바뀌는지 확인 — Replacement 열을 꼭 본다
aws cloudformation describe-change-set \
  --stack-name app-prod --change-set-name upgrade-20261014 \
  --query 'Changes[].ResourceChange.[Action,LogicalResourceId,ResourceType,Replacement]' \
  --output table

# 3. 괜찮으면 실행
aws cloudformation execute-change-set --stack-name app-prod --change-set-name upgrade-20261014
```

| `Replacement` 값 | 뜻 |
| --- | --- |
| `True` | 반드시 교체 |
| `Conditional` | 다른 리소스의 값에 따라 교체될 수 있음(실행해 봐야 앎) |
| `False` | 교체 없음 |

- `aws cloudformation deploy`는 내부적으로 change set을 만들고 바로 실행한다. `--no-execute-changeset`을 주면 만들기만 하고 멈추므로, CI에서 "리뷰 후 실행" 흐름에 쓴다.
- 새 스택에도 change set을 쓸 수 있다(`--change-set-type CREATE`). 실행 전까지 스택은 `REVIEW_IN_PROGRESS` 상태다.
- nested stack이 있으면 부모 출력값이 바뀐 자식 리소스가 `Conditional`로 표시될 수 있다.

### Drift-aware change set

`--deployment-mode REVERT_DRIFT`로 만들면, 템플릿과 **실제 리소스 상태**를 함께 비교한다. 그래서 템플릿을 바꾸지 않은 리소스라도 누군가 콘솔에서 바꾼 부분(drift)을 템플릿대로 되돌린다. ([Drift-aware change sets](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/drift-aware-change-sets.html))

- Auto Scaling이 바꾼 용량처럼 AWS가 관리하는 값은 템플릿에서 바꾸지 않는 한 실제 값을 유지한다.
- 템플릿에 없는 태그는 지우지 않는다. 교체가 필요한 속성의 drift는 되돌리지 않는다.
- 이 기능의 출시일은 확인하지 못했다.

## 실패와 rollback

출처: [Stack failure options](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stack-failure-options.html), [create-stack](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/create-stack.html)

| 상황 | 기본 동작 | 결과 상태 |
| --- | --- | --- |
| 생성 실패 | 만든 것을 모두 지움 | `ROLLBACK_COMPLETE` → **삭제만 가능**. 같은 이름으로 다시 만들려면 먼저 지워야 함 |
| 업데이트 실패 | 이전 상태로 되돌림 | `UPDATE_ROLLBACK_COMPLETE` → 정상(다시 업데이트 가능) |
| 업데이트 rollback마저 실패 | 멈춤 | **`UPDATE_ROLLBACK_FAILED`** → 아래 복구 절차 필요 |

| 옵션 | 동작 |
| --- | --- |
| `--on-failure ROLLBACK`(생성 기본값) / `DELETE` / `DO_NOTHING` | 생성 실패 시 처리 방식 |
| `--disable-rollback` | **성공한 리소스는 남겨 두기**. `CREATE_FAILED`·`UPDATE_FAILED`에서 멈추고, 원인을 고친 뒤 재시도·업데이트·`rollback-stack` 중 선택. 개발 중 디버깅에 편함 |

새로 생긴 **Express 모드**(`--deployment-config '{"mode":"EXPRESS"}'`)는 리소스가 준비될 때까지 기다리지 않고 설정만 적용하면 완료로 처리한다. 이 모드는 rollback이 기본으로 꺼져 있다. 출시일과 세부 조건은 확인하지 못했다(확인 필요).

### Rollback trigger: 알람이 울리면 되돌리기

출처: [Rollback triggers](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-rollback-triggers.html)

```bash
aws cloudformation update-stack --stack-name app-prod \
  --template-body file://app.yaml --capabilities CAPABILITY_IAM \
  --rollback-configuration \
    "RollbackTriggers=[{Arn=arn:aws:cloudwatch:ap-northeast-2:111122223333:alarm:app-5xx,Type=AWS::CloudWatch::Alarm}],MonitoringTimeInMinutes=10"
```

- CloudWatch 알람(composite 포함) 최대 5개를 지정하고, 배포 후 0–180분 동안 감시한다.
- 알람이 `ALARM`이 되면 rollback한다. `INSUFFICIENT_DATA`로는 rollback하지 않으므로, 알람의 missing data 처리를 맞춰 둔다([aws-cloudwatch/03](../aws-cloudwatch/03-alarms.md)).
- 지정한 알람이 존재하지 않으면 작업 전체가 실패한다.

### UPDATE_ROLLBACK_FAILED 복구

출처: [Continue rolling back an update](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-continueupdaterollback.html)

흔한 원인은 **누군가 CloudFormation 밖에서 리소스를 지우거나 바꿔서**, 되돌릴 대상이 없는 경우다(예: DB를 콘솔에서 삭제).

```bash
# 원인을 고칠 수 있으면 고친 뒤
aws cloudformation continue-update-rollback --stack-name app-prod

# 고칠 수 없는 리소스는 건너뛰기 (최후 수단)
aws cloudformation continue-update-rollback --stack-name app-prod \
  --resources-to-skip AppDatabase
```

건너뛴 리소스는 실제 상태와 상관없이 `UPDATE_COMPLETE`로 표시된다. 다음 업데이트 전에 템플릿과 실제 상태를 맞추지 않으면 스택을 복구할 수 없게 될 수 있다.

## Drift detection

출처: [Detect drift](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/detect-drift-stack.html), [Stack drift](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.html)

```bash
ID=$(aws cloudformation detect-stack-drift --stack-name app-prod --query StackDriftDetectionId --output text)
aws cloudformation describe-stack-drift-detection-status --stack-drift-detection-id "$ID"
aws cloudformation describe-stack-resource-drifts --stack-name app-prod \
  --stack-resource-drift-status-filters MODIFIED DELETED
```

| 리소스 상태 | 뜻 |
| --- | --- |
| `IN_SYNC` | 템플릿과 같음 |
| `MODIFIED` | 속성이 바뀜 |
| `DELETED` | 밖에서 삭제됨 |
| `NOT_CHECKED` | drift 검사를 지원하지 않는 타입 |

drift를 고치는 방법은 세 가지다. ([Resolve drift](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-resolve-drift.html))

1. 실제 상태가 맞으면 **템플릿을 실제에 맞게** 고쳐 업데이트한다(교체 속성이면 리소스가 다시 만들어지니 주의).
2. 템플릿이 맞으면 **drift-aware change set**으로 실제를 템플릿대로 되돌린다.
3. `Retain`으로 스택에서 떼어 낸 뒤 실제 상태로 다시 **import**한다.

배열 기본값 때문에 실제로는 같은데 drift로 표시되는 경우도 있다.

## 지우거나 망가뜨리지 않게 보호하기

### DeletionPolicy / UpdateReplacePolicy

출처: [DeletionPolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-deletionpolicy.html), [UpdateReplacePolicy](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-updatereplacepolicy.html)

```yaml
Resources:
  AppDatabase:
    Type: AWS::RDS::DBInstance
    DeletionPolicy: Snapshot          # 스택 삭제 시 스냅샷을 남기고 삭제
    UpdateReplacePolicy: Snapshot     # 교체 시 옛 DB의 스냅샷을 남김
    Properties:
      Engine: postgres
      DBInstanceClass: db.t4g.micro
      AllocatedStorage: "20"
      MasterUsername: appadmin
      ManageMasterUserPassword: true
  ArchiveBucket:
    Type: AWS::S3::Bucket
    DeletionPolicy: Retain            # 스택을 지워도 버킷은 남김
    UpdateReplacePolicy: Retain
```

| 값 | 동작 |
| --- | --- |
| `Delete` | 지움(**대부분 리소스의 기본값**) |
| `Retain` | 남김(스택에서만 빠짐). **남은 리소스는 계속 과금** |
| `RetainExceptOnCreate` | 생성 중 rollback될 때는 지우고, 그 외에는 남김 |
| `Snapshot` | 스냅샷을 남기고 지움(EBS 볼륨, RDS 등 지원 타입만) |

- `AWS::RDS::DBCluster`와 클러스터에 속하지 않은 `AWS::RDS::DBInstance`는 기본값이 `Snapshot`이다.
- `DeletionPolicy`는 **업데이트 중 교체에는 적용되지 않는다.** 교체에는 `UpdateReplacePolicy`가 적용되므로 둘을 함께 쓴다.
- S3 버킷은 비어 있어야 지울 수 있다. 비어 있지 않으면 스택 삭제가 `DELETE_FAILED`로 멈춘다(→ [06](./06-issues.md)).

### 스택 단위 보호

| 기능 | 막는 것 | 메모 |
| --- | --- | --- |
| **Termination protection** | 스택 삭제 | 기본은 꺼짐. nested stack에 자동 적용 |
| **Stack policy** | 업데이트 중 특정 리소스의 `Update:Modify`/`Replace`/`Delete` | 설정하면 명시적으로 허용하지 않은 리소스는 모두 보호. **삭제는 못 하고** 전부 허용하는 정책으로 바꿔야 함. IAM 권한과는 별개 |

```bash
aws cloudformation update-termination-protection --stack-name app-prod --enable-termination-protection
```

```json
{
  "Statement": [
    { "Effect": "Allow", "Action": "Update:*", "Principal": "*", "Resource": "*" },
    { "Effect": "Deny", "Action": ["Update:Replace", "Update:Delete"], "Principal": "*",
      "Resource": "LogicalResourceId/AppDatabase" }
  ]
}
```

출처: [Termination protection](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-protect-stacks.html), [Stack policy](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/protect-stack-resources.html)

## 기존 리소스를 스택으로 가져오기

| 방법 | 쓰임새 | 출처 |
| --- | --- | --- |
| **Resource import**(`--change-set-type IMPORT`) | 콘솔로 만든 리소스 몇 개를 스택에 넣음. 리소스마다 식별자와 `DeletionPolicy`가 필요 | [Resource import](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import.html) |
| **IaC generator** | 계정·리전 전체를 스캔해 기존 리소스로 템플릿을 생성. 스택으로 import하거나 CDK로 옮김 | [IaC generator](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/generate-IaC.html) |
| **Stack refactoring** | 리소스를 다시 만들지 않고 **다른 스택으로 옮기거나 논리 ID 이름을 바꿈**(최대 5개 스택) | [Stack refactoring](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stack-refactoring.html) |

```bash
# IaC generator: 스캔 → 템플릿 생성
aws cloudformation start-resource-scan
aws cloudformation create-generated-template --generated-template-name legacy-vpc \
  --resources ResourceType=AWS::EC2::VPC,ResourceIdentifier={VpcId=vpc-0123456789abcdef0}
```

- IaC generator 스캔은 리전 단위이고 30일 뒤 만료된다. 템플릿당 리소스 500개까지이며, Cloud Control API를 지원하는 타입만 대상이다.
- `create-generated-template`의 `--resources` 형식은 연구 노트 기준이며, CLI 레퍼런스와 하나씩 대조하지는 않았다(확인 필요).
- Stack refactoring은 리소스를 만들거나 지우거나 수정하지 않는다. stack policy가 붙은 스택은 지원하지 않고, 원본 스택을 빈 상태로 남길 수 없다.

## Service role과 권한

출처: [Service role](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-iam-servicerole.html)

```bash
aws cloudformation create-stack --stack-name app-prod --template-body file://app.yaml \
  --role-arn arn:aws:iam::111122223333:role/cfn-deploy-app
```

- service role이 없으면 CloudFormation은 **호출한 사용자의 권한**으로 리소스를 만든다. 그러면 배포하는 사람이 모든 리소스 권한을 가져야 한다.
- service role을 쓰면 사용자에게는 CloudFormation 권한과 그 role에 대한 `iam:PassRole`만 주면 된다([aws-iam/06 PassRole](../aws-iam/06-least-privilege.md)).
- 한 번 service role을 지정한 스택은 **이후 모든 작업에 그 role을 쓰고, role을 떼어 낼 수 없다.**
- 스택 권한이 있는 다른 사용자는 `PassRole` 없이도 그 role로 작업할 수 있다. 그래서 service role 자체를 최소 권한으로 만든다.

### 스택 삭제가 막혔을 때

```bash
aws cloudformation delete-stack --stack-name app-dev --retain-resources NonEmptyBucket   # DELETE_FAILED 상태에서만
aws cloudformation delete-stack --stack-name app-dev --deletion-mode FORCE_DELETE_STACK   # DELETE_FAILED 강제 정리
```

`--retain-resources`나 강제 삭제로 남은 리소스는 스택 밖에서 계속 과금되니 따로 정리한다.

## 이슈 / 주의

- **change set 없이 운영 스택을 직접 업데이트하지 않는다.** 교체 여부를 모른 채 실행하면 DB가 새로 만들어질 수 있다.
- **데이터 리소스에는 `DeletionPolicy`·`UpdateReplacePolicy`를 함께** 둔다. 하나만 있으면 다른 상황에서 지워진다.
- **콘솔에서 직접 고치지 않는다.** drift가 생기고, 다음 업데이트나 rollback이 실패하는 원인(`UPDATE_ROLLBACK_FAILED`)이 된다.
- **`--resources-to-skip`은 최후 수단**이다. 스택과 실제 상태가 어긋난 채로 남는다.
- **이름을 지정한 리소스**는 교체가 필요한 변경을 할 수 없다.

## 출처

- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-update-behaviors.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-properties-name.html
- https://docs.aws.amazon.com/cli/latest/reference/cloudformation/create-change-set.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_ResourceChange.html
- https://docs.aws.amazon.com/cli/latest/reference/cloudformation/deploy.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/drift-aware-change-sets.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stack-failure-options.html
- https://docs.aws.amazon.com/cli/latest/reference/cloudformation/create-stack.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-rollback-triggers.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-updating-stacks-continueupdaterollback.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/detect-drift-stack.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-stack-drift.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import-resolve-drift.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-deletionpolicy.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-attribute-updatereplacepolicy.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-cfn-protect-stacks.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/protect-stack-resources.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/resource-import.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/generate-IaC.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/stack-refactoring.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/using-iam-servicerole.html
- https://docs.aws.amazon.com/cli/latest/reference/cloudformation/delete-stack.html

---
*조사 기준일: 2026-10-14. 템플릿 조각은 cfn-lint 1.57.2로 검사해 통과했다. CLI 예시는 실제 계정에서 실행하지 않았다.*
