# 조직 구조와 OU 설계

> OU는 **조직도(부서)가 아니라 "같은 정책을 받아야 하는 계정 묶음"** 기준으로 나눈다. AWS 백서의 기본형은 **Security, Infrastructure, Workloads(Prod / SDLC), Sandbox**에 절차용 OU(Suspended, Policy Staging 등)를 더하는 구조다. management 계정에는 아무 워크로드도 두지 않고, 서비스 관리는 **위임 관리자**에게 넘긴다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)의 용어(조직, management·member 계정, 조직 root, OU, 위임 관리자 등)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Log Archive 계정 | 조직 전체 로그(CloudTrail, Config 등)를 모아 보관하는 전용 계정 |
| Security Tooling(Audit) 계정 | GuardDuty·Security Hub 같은 보안 도구를 운영하는 계정 |
| SDLC | 개발·테스트·스테이징처럼 운영 이전 단계(Software Development Life Cycle) 환경 |
| Handshake | Organizations에서 초대·기능 전환 같은 요청을 상대 계정이 승인하는 절차 |
| Post-closure period | 계정을 해지한 뒤 되살릴 수 있는 90일 유예 기간 |
| Delegation policy | management 계정이 member 계정에 Organizations 관리 작업 일부를 허락하는 resource-based policy |

## AWS 권장 OU 구조

출처: [Recommended OUs and accounts](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/recommended-ous-and-accounts.html), [Foundational OUs](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/foundational-ous.html), [OU best practices](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_ous_best_practices.html)

| 분류 | OU | 담는 계정 | 목적 |
| --- | --- | --- | --- |
| 기반(foundational) | **Security** | Log Archive, Security Tooling | 로그 보관과 보안 도구. 로그 버킷 수정·삭제를 SCP로 막음 |
| 기반 | **Infrastructure** | Network, Identity, Shared Services, Backup, Monitoring, Operations Tooling | 여러 워크로드가 공유하는 인프라 |
| 애플리케이션 | **Workloads** → Prod / SDLC(또는 NonProd) | 서비스별 운영·개발 계정 | 실제 서비스. 환경별로 정책을 다르게 |
| 실험 | **Sandbox** | 개인·팀 실험 계정 | 자유롭게 써 보되 비용·리전 제한 |
| 절차 | **Policy Staging** | 테스트용 계정 | 새 SCP를 운영 OU에 걸기 전에 시험 |
| 절차 | **Suspended** | 해지 대기 계정 | deny-all SCP를 걸어 아무것도 못 하게 |
| 절차 | Exceptions, Transitional | 예외가 필요한 계정, 이전 중인 계정 | 일시적 수용 |
| 고급 | Individual Business Users, Deployments, Business Continuity | 업무 사용자 계정, CI/CD 배포 계정, 재해 복구 | 필요할 때 추가 |

```
Root
├── Security
│   ├── log-archive
│   └── security-tooling
├── Infrastructure
│   ├── network
│   └── shared-services
├── Workloads
│   ├── Prod
│   │   └── payments-prod
│   └── SDLC
│       └── payments-dev
├── Sandbox
├── Policy Staging
└── Suspended
```

### 설계 원칙

AWS 백서의 원칙 중 실무에 직접 영향을 주는 것들이다. ([Design principles](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/design-principles-for-your-multi-account-strategy.html))

| 원칙 | 이유 |
| --- | --- |
| OU는 **보안·운영 정책 기준**으로 나눈다(조직도 기준 X) | 팀 개편은 잦지만 "prod에 걸 규칙"은 잘 안 바뀜 |
| **management 계정에 워크로드를 두지 않는다** | SCP·RCP가 적용되지 않아 가드레일 밖에 있음 |
| **깊은 OU 계층을 피한다** | 최대 5단계까지 되지만, 깊을수록 정책 상속을 추적하기 어려움 |
| 환경(prod / non-prod)을 계정으로 분리 | 권한·한도·사고 범위를 분리 |

## Management 계정 최소화

출처: [Best practices for the management account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices_mgmt-acct.html)

- management 계정은 **꼭 그 계정에서만 할 수 있는 작업**에만 쓴다(조직 정책 관리, 계정 생성, 결제 등).
- 서비스 관리는 위임 관리자 계정으로 넘긴다.
- management 계정에 접근하는 사람과 role을 최소화하고 MFA를 강제한다.
- 2026-07-10 이후 콘솔에서 만든 조직에는 조직 root에 `organizations:LeaveOrganization`과 `account:CloseAccount`를 거부하는 SCP가 기본으로 붙는다. 계정이 조직을 멋대로 떠나거나 해지되는 것을 막기 위해서다. CLI·SDK·CloudFormation으로 만들었거나 그 전에 만든 조직에는 붙지 않으므로 직접 만들어 붙여야 한다.

## 계정 수명 주기

### 생성

```bash
aws organizations create-account \
  --email payments-prod@example.com \
  --account-name "payments-prod"
# → CreateAccountStatus.Id = car-... , State = IN_PROGRESS

aws organizations describe-create-account-status \
  --create-account-request-id car-examplecreateaccountrequestid111
# → State: IN_PROGRESS | SUCCEEDED | FAILED
```

- 새 계정은 **조직 root 바로 아래**에 만들어지므로, 만든 뒤 원하는 OU로 옮긴다.
- `OrganizationAccountAccessRole`이 자동으로 생기고, management 계정이 관리자 권한으로 이 role을 맡을 수 있다(`--role-name`으로 이름 변경 가능). 이 role에도 SCP가 적용된다.
- 이메일은 다른 AWS 계정에서 쓰지 않은 주소여야 한다.
- 동시에 5개까지 만들 수 있다.

([Create an account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_create.html))

### 초대

기존 계정을 조직에 초대하면 그 계정이 handshake를 승인해야 들어온다.

- 초대는 15일 뒤 만료된다.
- 초대한 계정에는 `OrganizationAccountAccessRole`이 생기지 않는다. `AdministratorAccess` 정책을 붙이고 management 계정을 신뢰하는 같은 이름의 role을 **직접 만들어야** 한다. ([Create the role in an invited account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_create-cross-account-role.html))

### OU 만들기와 계정 이동

```bash
aws organizations list-roots --query 'Roots[0].Id' --output text   # r-examplerootid111

aws organizations create-organizational-unit \
  --parent-id r-examplerootid111 --name Workloads

aws organizations move-account \
  --account-id 333333333333 \
  --source-parent-id r-examplerootid111 \
  --destination-parent-id ou-exam-ple12345
```

계정을 옮기면 **새 OU의 정책이 즉시 적용**된다. 운영 계정을 옮기기 전에 새 OU의 SCP가 그 계정의 작업을 막지 않는지 먼저 확인한다(→ [03](./03-scp-rcp.md#안전한-테스트와-롤아웃)).

### 조직에서 제거

출처: [Remove a member account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_remove.html)

- 단독 계정으로 운영할 정보(지원 플랜, 연락처, 결제 수단)가 있어야 한다. 없으면 `ConstraintViolationException`이 난다.
- 조직에서 **만든** 계정은 생성 후 4일이 지나야 제거할 수 있다. 초대한 계정은 기다릴 필요가 없다.
- 위임 관리자로 등록된 계정은 먼저 등록을 해제해야 한다.
- 제거하면 계정 태그는 삭제되지만, `OrganizationAccountAccessRole`은 자동으로 지워지지 않는다.

```bash
aws organizations remove-account-from-organization --account-id 333333333333   # management 계정에서
aws organizations leave-organization                                          # member 계정에서
```

### 해지(close)

```bash
aws organizations close-account --account-id 333333333333
```

- 해지한 계정은 최대 90일간 `CLOSED` 상태로 보이고, 이 기간(post-closure period)에는 다시 열 수 있다. 90일이 지나면 영구 해지되며 계정 ID는 재사용되지 않는다.
- 영구 해지 전까지는 조직의 계정 수 한도에 포함된다.
- 해지 한도: 30일 동안 member 계정의 20% 또는 250개 중 큰 값(최대 1,000개). 동시 진행은 3개까지다. 이 한도는 늘릴 수 없다.

([Close an account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_close.html), [Quotas](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_reference_limits.html))

해지 대기 중인 계정은 Suspended OU로 옮겨 deny-all SCP를 걸어 두는 것이 백서의 권장 패턴이다.

## 위임 관리자

### AWS 서비스 관리 위임

출처: [Delegated administrator](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_integrate_delegated_admin.html), [register-delegated-administrator](https://docs.aws.amazon.com/cli/latest/reference/organizations/register-delegated-administrator.html)

1. 해당 서비스의 trusted access를 켠다. AWS는 Organizations CLI보다 **그 서비스 자체의 콘솔·API로 켜라**고 강하게 권장한다. 서비스가 필요한 초기 설정과 정리를 함께 하기 때문이다.
2. management 계정에서 위임 관리자를 등록한다.

```bash
aws organizations register-delegated-administrator \
  --account-id 444455556666 \
  --service-principal config.amazonaws.com

aws organizations list-delegated-administrators
```

| 서비스(예) | 보통 위임하는 계정 |
| --- | --- |
| GuardDuty, Security Hub, Macie, Inspector, Detective, Config, Firewall Manager | Security Tooling |
| CloudTrail(조직 trail) | Security Tooling 또는 Log Archive |
| IAM Identity Center | Identity 계정(Infrastructure OU) |
| AWS Backup | Backup 계정 |
| CloudFormation StackSets | Deployments·Shared Services 계정 |

([Foundational OUs](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/foundational-ous.html)) 위 표의 계정 배치는 백서 예시를 바탕으로 정리한 것이다. 위임을 지원하는 전체 서비스 목록은 [Services that work with Organizations](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_integrate_services_list.html)에서 확인한다.

위임 관리자 계정은 그 서비스의 관리 권한과 함께, Organizations의 읽기 전용 작업(`Describe*`, `List*`) 일부를 받는다.

### Organizations 정책 관리 위임

management 계정이 **delegation policy**(resource-based policy)를 걸어, 특정 member 계정이 조직 정책을 만들고 붙이게 할 수 있다. ([Delegate policy management](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_delegate_policies.html))

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DelegateBackupPolicies",
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::444455556666:root" },
      "Action": [
        "organizations:DescribeOrganization",
        "organizations:ListRoots",
        "organizations:ListOrganizationalUnitsForParent",
        "organizations:ListAccountsForParent",
        "organizations:CreatePolicy",
        "organizations:UpdatePolicy",
        "organizations:DeletePolicy",
        "organizations:AttachPolicy",
        "organizations:DetachPolicy"
      ],
      "Resource": "*",
      "Condition": {
        "StringLikeIfExists": { "organizations:PolicyType": "BACKUP_POLICY" }
      }
    }
  ]
}
```

```bash
aws organizations put-resource-policy --content file://delegation-policy.json
```

- `organizations:PolicyType` 조건으로 위임할 정책 종류를 제한한다.
- 2026-06-30부터 delegation policy에서 `NotAction`·`NotResource`는 거부된다. 이미 저장된 정책은 계속 동작한다.
- delegation policy는 최대 40,000자다.
- 위 JSON은 AWS 예시의 구조를 따라 이 레포에서 줄여 쓴 것이다. 실제로 위임할 액션 목록과 조건 연산자는 원문과 대조할 것(확인 필요).
- SCP처럼 강한 정책의 관리를 위임하면 위임받은 계정이 조직 전체를 막을 수 있다. SCP 위임은 신중히 한다.

## 이슈 / 주의

- **새 조직의 계정 수 한도는 기본 10개**다. 새 조직은 이보다 낮을 수도 있으며, 초대 대기 중인 계정도 한도에 포함된다. 계정을 많이 만들 계획이면 미리 증설을 요청한다. ([Quotas](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_reference_limits.html))
- **management 계정은 바꿀 수 없다.** 처음 조직을 만드는 계정은 워크로드가 없는 새 계정으로 하는 것이 좋다.
- **OU 이동 = 정책 변경**이다. 계정을 옮기는 것 자체가 그 계정의 권한 상한을 바꾸는 작업이다.
- 조직에서 만든 계정의 root user는 비밀번호가 없는 상태로 시작한다. root 관리 방식은 [05](./05-other-policies.md)의 root 중앙 관리를 참고한다.

## 출처

- https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/recommended-ous-and-accounts.html
- https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/foundational-ous.html
- https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/advanced-ous.html
- https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/design-principles-for-your-multi-account-strategy.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_ous_best_practices.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices_mgmt-acct.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_create.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_create-cross-account-role.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_remove.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_close.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_integrate_delegated_admin.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_integrate_services_list.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_delegate_policies.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_reference_limits.html
- https://docs.aws.amazon.com/cli/latest/reference/organizations/register-delegated-administrator.html

---
*조사 기준일: 2026-10-13. CLI 예시는 공식 레퍼런스 예시를 바탕으로 했고 실제 조직에서 실행하지는 않았다.*
