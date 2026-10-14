# 검증 도구·흔한 실패·한도

> 배포 전에 **cfn-lint**(문법·속성 검사)와 **CloudFormation Guard**(조직 규칙 검사)를 CI에서 돌리고, 계정 단위로 강제해야 하는 규칙은 **Hooks**로 막는다. 운영에서 가장 자주 만나는 실패는 다음과 같다: 비어 있지 않은 S3 버킷 때문의 `DELETE_FAILED`, 밖에서 바뀐 리소스 때문의 `UPDATE_ROLLBACK_FAILED`, 이름을 지정한 리소스의 교체 실패, import 중인 Export 변경, `CAPABILITY_*` 누락, 순환 의존성, 응답 없는 custom resource의 1시간 대기.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[05](./05-cdk-terraform.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Linter | 코드를 실행하지 않고 오류·나쁜 패턴을 찾는 검사 도구 |
| Policy as code | "버킷은 반드시 암호화" 같은 조직 규칙을 코드로 적어 자동 검사하는 방식 |
| Hook | CloudFormation이 리소스를 만들기·바꾸기·지우기 직전에 실행하는 검사 |
| Capability | IAM 리소스·매크로처럼 위험할 수 있는 작업을 "알고 한다"고 명시하는 확인 플래그 |
| Circular dependency | A가 B를, B가 A를 참조해 생성 순서를 정할 수 없는 상태 |

## 검증 도구

| 도구 | 검사 내용 | 언제 | 출처 |
| --- | --- | --- | --- |
| `aws cloudformation validate-template` | **문법만**(JSON·YAML으로 읽히는지) | 최소 확인 | [validate-template](https://docs.aws.amazon.com/cli/latest/reference/cloudformation/validate-template.html) |
| **사전 배포 검증** | 스택 생성·업데이트·change set 때 자동 실행. 속성 문법 오류, 이름 충돌, 서비스 한도 경고 | 자동(`--disable-validation`으로 끌 수 있음) | [Troubleshooting](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/troubleshooting.html) |
| **cfn-lint** | 리소스 스키마 기준의 속성 이름·값·타입, 모범 사례. E(오류)/W(경고)/I(정보) | **로컬·CI에서 항상** | [cfn-lint](https://github.com/aws-cloudformation/cfn-lint) |
| **CloudFormation Guard** | 직접 쓴 조직 규칙(policy as code). Terraform plan JSON, Kubernetes 설정도 검사 가능 | CI | [cfn-guard](https://github.com/aws-cloudformation/cloudformation-guard) |
| **Hooks** | 계정 안의 **모든** CloudFormation 작업에서 규칙을 강제(FAIL/WARN) | 계정·조직 가드레일 | [Hooks](https://docs.aws.amazon.com/cloudformation-cli/latest/hooks-userguide/what-is-cloudformation-hooks.html) |

### cfn-lint

```bash
pip install cfn-lint
cfn-lint templates/*.yaml
cfn-lint --regions ap-northeast-2 us-east-1 -- template.yaml   # 리전별 리소스 지원 여부도 검사
```

이 레포의 템플릿 예시도 cfn-lint 1.57.2로 검사했다.

### CloudFormation Guard

```
# rules/s3.guard — 모든 S3 버킷은 버전 관리를 켜야 한다
let buckets = Resources.*[ Type == 'AWS::S3::Bucket' ]

rule S3_VERSIONING_ENABLED when %buckets !empty {
    %buckets.Properties.VersioningConfiguration.Status == 'Enabled'
    <<
      Violation: S3 buckets must enable versioning
    >>
}
```

```bash
cfn-guard validate -d template.yaml -r rules/s3.guard
```

위 규칙은 Guard README의 규칙 형식을 따라 이 레포에서 작성한 예시이며, 실제로 실행하지는 않았다(확인 필요).

### Hooks

출처: [Hooks concepts](https://docs.aws.amazon.com/cloudformation-cli/latest/hooks-userguide/hooks-concepts.html)

| 항목 | 내용 |
| --- | --- |
| 구현 방식 | Guard 규칙, Lambda 함수, Control Tower proactive control, CloudFormation CLI로 직접 개발 |
| 대상 | `RESOURCE`, `STACK`, `CHANGE_SET`(생성만), `CLOUD_CONTROL` |
| 시점 | `preCreate`, `preUpdate`, `preDelete` |
| 모드 | `FAIL`(막음) / `WARN`(경고만). **AWS는 WARN으로 시작하라고 권장** |
| 제한 | 호출당 30초, 재시도 3회 |
| 요금 | AWS 관리형 hook은 무료. 직접 만든 hook은 handler 작업당 과금([01](./01-overview.md#7단계-요금)) |

Hooks는 **CloudFormation(및 Cloud Control API)을 통한 작업만** 검사한다. 콘솔·SDK·Terraform으로 직접 만든 리소스는 막지 못한다. Control Tower proactive control도 같은 한계가 있다. 모든 경로를 막으려면 SCP를 함께 쓴다([aws-organizations/03](../aws-organizations/03-scp-rcp.md)).

## 흔한 실패와 해결

출처: [Troubleshooting CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/troubleshooting.html)

| 증상 | 원인 | 해결 | 자세히 |
| --- | --- | --- | --- |
| `DELETE_FAILED` | **비어 있지 않은 S3 버킷**, 다른 곳에서 쓰는 보안 그룹, termination protection | 버킷 비우기, 의존 리소스 정리. 남겨도 되면 `--retain-resources`, 또는 `--deletion-mode FORCE_DELETE_STACK` | [03](./03-stack-operations.md#스택-삭제가-막혔을-때) |
| `UPDATE_ROLLBACK_FAILED` | 밖에서 지운·바꾼 리소스, 권한, 만료된 토큰, 한도, 안정화 실패 | 원인을 고치고 `continue-update-rollback`. 안 되면 `--resources-to-skip`(최후 수단) | [03](./03-stack-operations.md#update_rollback_failed-복구) |
| `ROLLBACK_COMPLETE`에서 업데이트 불가 | 생성이 실패한 스택 | **삭제 후 다시 생성** | [03](./03-stack-operations.md#실패와-rollback) |
| "No updates are to be performed" | `DeletionPolicy`, `UpdatePolicy`, `Conditions`, `Outputs`만 바꿈 → 업데이트로 인식 안 됨 | 리소스의 `Metadata`를 함께 바꿈 | — |
| 교체가 필요한 업데이트 실패 | **이름을 지정한 리소스**는 교체 불가(같은 이름 공존 불가) | 이름도 함께 바꾸거나, 이름 지정을 빼고 설계 | [03](./03-stack-operations.md#업데이트-동작-수정인가-교체인가) |
| Export를 바꾸거나 스택을 못 지움 | 다른 스택이 그 Export를 import 중 | `list-imports`로 확인, import하는 쪽을 먼저 바꿈 | [02](./02-templates.md#스택-사이에서-값-넘기기) |
| `InsufficientCapabilities` | IAM 리소스·매크로에 확인 플래그 없음 | IAM이면 `CAPABILITY_IAM`, **이름을 지정한 IAM**이면 `CAPABILITY_NAMED_IAM`, Transform·매크로면 `CAPABILITY_AUTO_EXPAND` | [CreateStack](https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_CreateStack.html) |
| Circular dependency | 두 리소스가 서로 참조. 대표 예: S3 버킷 알림 ↔ Lambda 권한 | 참조 하나를 끊음(이름을 문자열로 조합, 별도 리소스로 분리) | [AWS 블로그](https://aws.amazon.com/blogs/mt/resolving-circular-dependency-in-provisioning-of-amazon-s3-buckets-with-aws-lambda-event-notifications) |
| 스택이 1시간 동안 `IN_PROGRESS` | **Custom resource가 응답하지 않음** | Lambda의 모든 경로(예외 포함)에서 응답 전송, `ServiceTimeout` 단축 | [Custom resource](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-cloudformation-customresource.html) |
| 업데이트 후 앱 장애인데 스택은 성공 | 리소스는 만들어졌지만 앱이 안 뜸 | rollback trigger(CloudWatch 알람), `CreationPolicy`·`cfn-signal` | [03](./03-stack-operations.md#rollback-trigger-알람이-울리면-되돌리기) |
| StackSet 인스턴스 `OUTDATED` | 대상 계정의 SCP·한도·이름 충돌 | 원인 수정 후 재배포 | [04](./04-stacksets.md#상태-읽기) |

## 한도

출처: [CloudFormation quotas](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html), [Service endpoints and quotas](https://docs.aws.amazon.com/general/latest/gr/cfn.html)

| 항목 | 값 | 증설 |
| --- | --- | --- |
| 리전당 스택 | 2,000(`DELETE_COMPLETE`가 아닌 모든 스택) | 가능 |
| 템플릿당 리소스 | **500** | 불가 |
| 요청 본문 템플릿 / S3 템플릿 | 51,200바이트 / 1 MB | 불가 |
| Parameters / Outputs / Mappings | 각 200 | 불가 |
| 파라미터 값 | 4,096바이트 | 불가 |
| 동적 참조 | 템플릿당 60 | 불가 |
| nested stack 계층 전체 리소스 | 작업당 2,500 | 불가 |
| 스택 이름 | 128자 | 불가 |
| StackSet / StackSet당 인스턴스 | 1,000 / 100,000 | 가능 |
| Hooks | 계정당 100 | 불가 |
| Custom resource 응답 | 4,096바이트 | 불가 |

- **템플릿당 리소스 500개**에 금방 닿는다. 수명·소유자별로 스택을 나누거나 nested stack을 쓴다.
- 템플릿이 51,200바이트를 넘으면 `aws cloudformation deploy --s3-bucket`처럼 S3를 거쳐 배포한다.
- 계정당 Export 개수는 이번에 확인한 한도 문서에 없었다(확인 필요).

## 안티패턴

| 안티패턴 | 문제 | 대안 |
| --- | --- | --- |
| 콘솔에서 스택 리소스를 직접 수정 | drift, 다음 업데이트·rollback 실패 | 템플릿으로만 변경, drift 정기 점검 |
| change set 없이 운영 업데이트 | DB 교체 같은 사고를 미리 못 봄 | change set 리뷰 후 실행(CI에서 `--no-execute-changeset`) |
| 데이터 리소스에 `DeletionPolicy` 없음 | 스택 삭제·교체 시 데이터 손실 | `DeletionPolicy`와 `UpdateReplacePolicy` 함께 설정 |
| 모든 리소스에 이름 지정 | 교체가 필요한 변경 불가, 환경 복제 시 이름 충돌 | 꼭 필요한 것만 이름 지정, 나머지는 자동 이름 |
| 거대한 단일 템플릿 | 500개 한도, 한 번 실패에 전체 rollback, 리뷰 어려움 | 수명·소유자별 스택 분리 |
| Export를 남발 | 서로 묶여 스택을 바꾸거나 지우기 어려움 | 정말 공유할 값만 Export, 나머지는 Parameter Store |
| 비밀번호를 파라미터·Outputs로 | 콘솔·API에 노출 | 동적 참조(Secrets Manager), `ManageMasterUserPassword` |
| service role 없이 관리자 권한으로 배포 | 배포자에게 모든 권한 필요 | 최소 권한 service role + `iam:PassRole` 제한 |
| StackSet을 처음부터 높은 동시 수로 | 오류가 모든 계정에 동시 확산 | `MaxConcurrentCount=1`부터 점진 확대 |
| 운영 스택 termination protection 꺼 둠 | 실수 한 번에 전체 삭제 | termination protection 켜기 |

## 출처

- https://docs.aws.amazon.com/cli/latest/reference/cloudformation/validate-template.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/troubleshooting.html
- https://github.com/aws-cloudformation/cfn-lint
- https://github.com/aws-cloudformation/cloudformation-guard
- https://docs.aws.amazon.com/cloudformation-cli/latest/hooks-userguide/what-is-cloudformation-hooks.html
- https://docs.aws.amazon.com/cloudformation-cli/latest/hooks-userguide/hooks-concepts.html
- https://docs.aws.amazon.com/controltower/latest/controlreference/proactive-controls.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/APIReference/API_CreateStack.html
- https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-cloudformation-customresource.html
- https://aws.amazon.com/blogs/mt/resolving-circular-dependency-in-provisioning-of-amazon-s3-buckets-with-aws-lambda-event-notifications
- https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cloudformation-limits.html
- https://docs.aws.amazon.com/general/latest/gr/cfn.html

---
*조사 기준일: 2026-10-14. Guard 규칙 예시는 이 레포에서 작성했고 실행하지 않았다.*
