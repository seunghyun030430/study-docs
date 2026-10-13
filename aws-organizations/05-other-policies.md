# 선언형 정책, root 중앙 관리, Control Tower

> SCP·RCP가 "API 호출을 막는" 정책이라면, **선언형 정책(declarative policies)**은 태그 표준, 백업 계획, EC2 기본 설정 같은 **서비스 설정 상태**를 조직 전체에 강제한다. management 계정에도 적용된다. 여기에 **root 자격증명 중앙 관리**(member 계정 root 비밀번호·키 제거, 필요 시 `sts:AssumeRoot`로 15분짜리 작업)를 더하면 계정 수준 위험이 크게 줄어든다. **Control Tower**는 이 모든 것을 권장 구성으로 묶어 자동화하는 상위 서비스다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md) 5·9단계와 [03-scp-rcp.md](./03-scp-rcp.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Control plane에서 강제 | 서비스가 설정을 바꾸는 API 단계에서 직접 규칙을 적용함(권한 검사와 별개) |
| Effective policy | 상속된 정책과 직접 붙인 정책을 합쳐 계정에 실제로 적용되는 최종 정책 |
| 상속 연산자 | 선언형 정책에서 상위 값을 덮어쓰거나(`@@assign`) 더하거나(`@@append`) 빼는(`@@remove`) 키워드 |
| Task policy | `sts:AssumeRoot`로 root 작업을 할 때 허용 범위를 정하는 AWS 제공 정책 |
| Account Factory | Control Tower에서 표준 설정이 적용된 새 계정을 찍어 내는 기능 |
| Control | Control Tower의 통제 규칙. 예방형(preventive), 탐지형(detective), 사전 검사형(proactive) |
| Drift | Control Tower가 만든 설정이 밖에서 바뀌어 기대 상태와 달라진 것 |

## 선언형 정책

### SCP·RCP와 무엇이 다른가

출처: [Managing organization policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies.html), [Declarative policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_declarative.html)

| 항목 | SCP·RCP(권한 정책) | 선언형 정책 |
| --- | --- | --- |
| 무엇을 하나 | API 호출 허용 여부의 상한선 | 서비스 설정 상태·기본값을 강제 |
| 어디서 적용 | IAM 권한 평가 | 각 서비스의 control plane |
| management 계정 | **적용 안 됨** | **적용됨** |
| service-linked role | 적용 안 됨 | 적용됨(서비스 설정이므로) |
| 상속 방식 | Allow는 모든 층, Deny는 한 층 | 상속 연산자로 병합 |
| Effective policy 조회 | 불가 | `describe-effective-policy`로 가능 |
| 정책을 떼면 | 제한이 풀림 | 설정이 **이전 상태로 되돌아감** |

### 종류

현재 공식 문서는 예전 "management policies"를 모두 선언형 정책으로 분류한다.

| 정책 | 무엇을 강제하나 |
| --- | --- |
| **Tag policies** | 태그 키의 대소문자·허용 값, 특정 리소스에 태그 표준 강제 |
| **Backup policies** | AWS Backup 백업 계획을 조직 전체에 배포 |
| **AI services opt-out policies** | AWS AI 서비스가 콘텐츠를 서비스 개선에 쓰는 것을 거부 |
| **Amazon EC2 policies** | EC2·VPC·EBS 보안 기본값(아래 표) |
| Chat applications policies | Slack·Teams 연동(예전 이름 chatbot) 제어 |
| Security Hub / Amazon Inspector / Amazon GuardDuty policies | 각 보안 서비스의 조직 전체 설정 |
| Amazon Bedrock policies | Bedrock 추론에 Guardrails 강제 |
| Amazon S3 policies | S3 조직 수준 설정 |
| Upgrade rollout policies | 업그레이드 배포 순서 설정 |

AWS Shield Network Security Director 정책은 일부 페이지에만 나온다. 최신 목록은 정책 문서에서 확인한다.

### EC2 정책으로 설정할 수 있는 것

출처: [Amazon EC2 policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_ec2.html)

| 속성 | 효과 |
| --- | --- |
| Instance Metadata Defaults | 새 인스턴스의 IMDS 기본값(예: IMDSv2 필수) |
| EBS Snapshot Block Public Access | EBS 스냅샷 공개 공유 차단 |
| Image Block Public Access | AMI 공개 공유 차단 |
| Allowed Images Settings | 허용한 AMI만 사용 |
| Serial Console Access | EC2 시리얼 콘솔 사용 여부 |
| VPC Block Public Access | VPC의 인터넷 게이트웨이 경로 차단 |
| VPC Encryption Controls | VPC 내부 트래픽 암호화 통제 |

[03](./03-scp-rcp.md#5-imdsv2-강제-scp)의 IMDSv2 SCP는 "IMDSv2를 안 쓰는 RunInstances 요청을 거부"하는 방식이다. EC2 정책은 **기본값 자체를 바꾸므로** 사용자가 옵션을 빠뜨려도 실패하지 않는다. 사용자 경험은 선언형 정책 쪽이 낫다.

### 상속 연산자

출처: [Policy operators](https://docs.aws.amazon.com/organizations/latest/userguide/policy-operators.html)

| 연산자 | 동작 | 적용 대상 |
| --- | --- | --- |
| `@@assign` | 상속된 값을 덮어씀 | 모든 설정 |
| `@@append` | 상속된 값에 추가 | 여러 값을 갖는 설정 |
| `@@remove` | 상속된 값에서 제거 | 여러 값을 갖는 설정 |
| `@@operators_allowed_for_child_policies` | 하위 정책이 쓸 수 있는 연산자 제한. `["@@none"]`이면 값을 잠금 | — |

콘솔의 시각 편집기는 `@@assign`만 지원한다.

### Tag policy 예시

출처: [Tag policy examples](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_example-tag-policies.html)

```json
{
  "tags": {
    "costcenter": {
      "tag_key": { "@@assign": "CostCenter" },
      "tag_value": { "@@assign": ["100", "200", "300*"] },
      "enforced_for": { "@@assign": ["ec2:instance", "ec2:volume"] }
    }
  }
}
```

- `tag_key`: 키의 대소문자 표준(`CostCenter`). 정의하지 않으면 소문자가 기준이다.
- `tag_value`: 허용 값. `*` 와일드카드를 쓸 수 있다.
- `enforced_for`: 이 리소스에서 규칙에 맞지 않는 태그 작업을 **거부**한다. 넣지 않으면 보고만 한다.
- **태그가 없는 리소스는 규정 준수 평가 대상이 아니다.** "태그 필수"를 강제하려면 SCP의 `aws:RequestTag` 조건이나 IaC 검사를 함께 쓴다.

```bash
aws organizations enable-policy-type --root-id r-examplerootid111 --policy-type TAG_POLICY
aws organizations describe-effective-policy --policy-type TAG_POLICY --target-id 333333333333
```

## Root 자격증명 중앙 관리

출처: [Centralized root access](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html), [Enable](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-enable-root-access.html), [Privileged tasks](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user-privileged-task.html)

member 계정마다 root 비밀번호·MFA를 관리하는 대신, **member 계정의 root 자격증명을 아예 없애고** root가 꼭 필요한 작업만 management 계정(또는 IAM 위임 관리자)에서 짧게 수행하는 기능이다.

- 켜면 member 계정의 root 비밀번호, 액세스 키, 서명 인증서, MFA를 제거할 수 있다.
- 조직에서 새로 만드는 계정은 처음부터 root 자격증명이 없다.

```bash
# management 계정에서
aws organizations enable-aws-service-access --service-principal iam.amazonaws.com
aws iam enable-organizations-root-credentials-management
aws iam enable-organizations-root-sessions
# (선택) IAM 위임 관리자 등록
aws organizations register-delegated-administrator \
  --service-principal iam.amazonaws.com --account-id 444455556666
```

### `sts:AssumeRoot`로 root 작업 하기

| 항목 | 내용 |
| --- | --- |
| 호출 가능 | management 계정 또는 IAM 위임 관리자 계정 |
| 엔드포인트 | **리전 STS 엔드포인트만** 가능(글로벌 엔드포인트 불가) |
| 세션 | 최대 15분(900초, 기본값도 900초) |
| 범위 | task policy 하나로 제한. `sts:TaskPolicyArn` 조건으로 허용할 task를 고를 수 있음 |

| Task policy | 용도 |
| --- | --- |
| `IAMAuditRootUserCredentials` | root 자격증명 현황 점검 |
| `IAMCreateRootUserPassword` | root 비밀번호 복구 허용 |
| `IAMDeleteRootUserCredentials` | root 자격증명 삭제 |
| `S3UnlockBucketPolicy` | 잘못된 버킷 정책으로 잠긴 S3 버킷 풀기 |
| `SQSUnlockQueuePolicy` | 잘못된 정책으로 잠긴 SQS 큐 풀기 |

```bash
aws sts assume-root \
  --region ap-northeast-2 \
  --target-principal 333333333333 \
  --task-policy-arn arn=arn:aws:iam::aws:policy/root-task/S3UnlockBucketPolicy \
  --duration-seconds 900
```

([AssumeRoot API](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoot.html))

이 기능을 쓰면 [03의 root 사용 금지 SCP](./03-scp-rcp.md#4-member-계정-root-user-사용-금지-scp)에 있던 "버킷 정책 예외"가 필요 없어질 수 있다. 다만 두 장치를 함께 둘지는 조직 정책에 따라 판단한다.

## AWS Control Tower와의 관계

출처: [What is Control Tower?](https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html), [How it works](https://docs.aws.amazon.com/controltower/latest/userguide/how-control-tower-works.html), [Controls](https://docs.aws.amazon.com/controltower/latest/userguide/how-controls-work.html), [Pricing](https://aws.amazon.com/controltower/pricing/)

Control Tower는 Organizations, IAM Identity Center, Service Catalog를 **조율(orchestrate)**해 권장 멀티 계정 환경을 만들고 유지하는 서비스다.

| Control Tower 기능 | 내부적으로 쓰는 것 |
| --- | --- |
| Landing zone | Organizations(OU·계정), 로그 보관·감사 계정, IAM Identity Center |
| Account Factory | Service Catalog로 표준 계정 생성 |
| 예방형 control | **SCP·RCP**, 선언형 정책 |
| 탐지형 control | AWS Config 규칙 |
| 사전 검사형 control | CloudFormation hook(배포 전 검사) |
| 대시보드 | 계정·control 준수 현황 |

- control은 필수(mandatory, 변경 불가), 강력 권장, 선택으로 나뉜다.
- 예방형 control은 SCP 기반이므로 **management 계정에는 적용되지 않는다.**
- Control Tower 자체는 무료다. Config, CloudTrail, S3, Service Catalog 등 내부 서비스 요금이 발생한다.
- landing zone 3.3 이하는 Log Archive·Audit 계정을 담은 Security OU를 만들었다. **4.0부터는 Security OU가 필수가 아니다.**

### Organizations를 직접 다룰 때 주의: drift

Control Tower가 관리하는 것을 Organizations 콘솔·CLI로 직접 바꾸면 **drift**가 생긴다. ([Drift](https://docs.aws.amazon.com/controltower/latest/userguide/drift.html))

- drift가 생기는 예:
  - Control Tower 관리 SCP(이름이 `aws-guardrails`로 시작)를 수정하거나 떼는 것
  - 계정을 OU 사이로 옮기거나 조직에서 제거하는 것
  - Security OU를 삭제하는 것(landing zone 재설정 필요)
  - `AWSControlTowerAdmin` 등 필수 role을 삭제하는 것
- Control Tower는 관리 SCP·RCP·선언형 정책을 매일 검사한다.
- landing zone이 drift 상태면 "Enroll account"가 동작하지 않는다.

Control Tower를 쓰는 조직에서는 **Control Tower 콘솔·API를 통해 변경**하고, 직접 만든 SCP는 Control Tower 관리 정책과 따로 둔다.

## 이슈 / 주의

- **선언형 정책은 management 계정에도 적용된다.** SCP처럼 "management는 예외"라고 생각하고 조직 root에 붙이면 management 계정 설정도 바뀐다.
- **선언형 정책을 떼면 설정이 되돌아간다.** "한 번 적용하고 뗀다"는 방식으로 쓸 수 없다.
- **tag policy만으로는 "태그 필수"가 되지 않는다.** 태그가 없는 리소스는 평가하지 않는다.
- **AssumeRoot는 리전 STS 엔드포인트가 필요하다.** `--region`을 빼고 글로벌 엔드포인트로 호출하면 실패한다.
- Backup·AI opt-out·Chat applications 정책의 세부 문법은 이번에 1차 출처로 확인하지 못했다(확인 필요).
- 중앙 root 관리의 출시일(2024-11-15)은 AWS 블로그 검색 요약으로만 확인했다(확인 필요).

## 출처

- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_declarative.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_ec2.html
- https://docs.aws.amazon.com/organizations/latest/userguide/policy-operators.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_tag-policies.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_example-tag-policies.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-enable-root-access.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_root-user-privileged-task.html
- https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoot.html
- https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html
- https://docs.aws.amazon.com/controltower/latest/userguide/how-control-tower-works.html
- https://docs.aws.amazon.com/controltower/latest/userguide/how-controls-work.html
- https://docs.aws.amazon.com/controltower/latest/userguide/drift.html
- https://aws.amazon.com/controltower/pricing/

---
*조사 기준일: 2026-10-13. tag policy JSON은 공식 예시 형식을 따라 이 레포에서 작성했다.*
