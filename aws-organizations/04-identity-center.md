# IAM Identity Center

> IAM Identity Center는 사람이 **한 번 로그인해서 여러 AWS 계정에 접근**하게 해 주는 AWS의 SSO 서비스다(무료). 관리자는 **permission set**(권한 묶음)을 만들어 "사용자·그룹 + permission set + 계정"으로 할당하고, Identity Center가 각 계정에 `AWSReservedSSO_…` role을 만들어 준다. 사용자는 그 role의 임시 자격증명으로 일한다. 운영은 management 계정이 아닌 **위임 관리자 계정**에서 한다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md) 7단계와 [aws-iam/01-overview.md](../aws-iam/01-overview.md) 9단계(페더레이션, IdP, SAML/OIDC)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Instance | Identity Center를 켠 단위. 조직용(organization instance)과 계정 하나용(account instance)이 있음 |
| Identity source | 사용자·그룹 정보를 어디서 가져오는지. Identity Center 자체 디렉터리, Active Directory, 외부 IdP 중 하나 |
| Active Directory(AD) | Microsoft의 사내 디렉터리 서비스 |
| SCIM | IdP의 사용자·그룹을 다른 시스템으로 자동 동기화하는 표준 프로토콜 |
| Permission set | 계정에 할당할 권한 묶음 템플릿(IAM 정책들의 모음) |
| Account assignment | "누구(사용자·그룹)에게 어느 계정에서 어떤 permission set"을 주는 할당 한 건 |
| Access portal | 사용자가 로그인 후 접근 가능한 계정·앱 목록을 보는 웹 페이지 |
| Provisioning | permission set을 실제 계정에 IAM role로 만들어 반영하는 것 |
| PKCE / device code | CLI가 브라우저 로그인으로 토큰을 받는 두 가지 방식 |

## 왜 Identity Center인가

| 방식 | 계정 50개, 직원 100명일 때 |
| --- | --- |
| 계정마다 IAM user | 최대 5,000개의 user·비밀번호·MFA·액세스 키 관리 |
| **Identity Center** | 사람은 회사 계정으로 한 번 로그인. 권한은 permission set 몇 개와 그룹 할당으로 관리 |

IAM Identity Center는 추가 요금이 없다. 고객 관리형 KMS 키를 쓰면 KMS 요금만 발생한다. ([FAQ](https://aws.amazon.com/iam/identity-center/faqs/)) 예전 이름은 AWS Single Sign-On(2022-07-26 개명)이며, 그래서 CLI 명령 이름에 `sso`, `sso-admin`이 남아 있다. ([What is IAM Identity Center?](https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html))

## 구성 요소

### Instance: 조직용 vs 계정용

출처: [Identity Center instances](https://docs.aws.amazon.com/singlesignon/latest/userguide/identity-center-instances.html)

| 항목 | Organization instance(권장) | Account instance |
| --- | --- | --- |
| 어디서 켜나 | management 계정 | member 또는 단독 계정 |
| **여러 AWS 계정 접근** | 가능 | **불가** |
| 위임 관리자 | 가능 | 불가 |
| 다중 리전 복제 | 가능 | 불가 |
| 용도 | 사람의 AWS 계정 접근 전반 | 일부 AWS 관리형 앱의 독립 배포 |

**AWS 계정 접근을 관리하려면 organization instance여야 한다.**

### Identity source

조직당 하나만 고를 수 있다. ([Manage your identity source](https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source.html))

| Identity source | 언제 | 비고 |
| --- | --- | --- |
| Identity Center directory(기본값) | 별도 IdP가 없는 작은 조직 | 사용자·그룹을 Identity Center에서 직접 관리 |
| Active Directory | 사내 AD 사용 | 온프레미스 AD 또는 AWS Managed Microsoft AD. Simple AD는 미지원 |
| **외부 IdP**(Okta, Microsoft Entra ID 등) | 회사 IdP가 있을 때 | SAML 2.0으로 로그인, **SCIM**으로 사용자·그룹 자동 동기화 |

SCIM 설정 시 주의점([Automatic provisioning](https://docs.aws.amazon.com/singlesignon/latest/userguide/provision-automatically.html)):

- SCIM 토큰은 **1년 뒤 만료**된다. 만료 90일 전부터 알림이 온다. 만료되면 동기화가 멈추므로 갱신 일정을 관리한다.
- SCIM을 켜면 Identity Center 콘솔에서 사용자를 직접 추가·수정할 수 없다. IdP 쪽에서 관리한다.

### Permission set과 할당

출처: [Permission sets](https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html)

permission set에 넣을 수 있는 것:

| 구성 | 설명 |
| --- | --- |
| AWS managed policy | 예: `ReadOnlyAccess`, 직무별 정책(job function) |
| Customer managed policy 참조 | **각 계정에 같은 이름의 정책이 미리 있어야** 함 |
| Inline policy | permission set 안에 직접 작성 |
| Permissions boundary | AWS managed 또는 customer managed policy로 상한선 지정 |
| Session duration | role 세션 길이. 기본 1시간, 최대 12시간 |

**할당(account assignment)** = 사용자 또는 그룹 + permission set + 대상 계정.

할당하면 대상 계정에 IAM role이 생긴다. ([Referencing permission sets](https://docs.aws.amazon.com/singlesignon/latest/userguide/referencingpermissionsets.html))

```
arn:aws:iam::333333333333:role/aws-reserved/sso.amazonaws.com/ap-northeast-2/AWSReservedSSO_ReadOnly_a1b2c3d4e5f6
```

- 이름 형식은 `AWSReservedSSO_<permission set 이름>_<임의 접미사>`다.
- 그 계정의 할당을 모두 지우면 role도 삭제되고, 다시 할당하면 **접미사가 바뀐 새 role**이 생긴다.
- 그래서 버킷 정책 등에서 이 role을 가리킬 때는 `ArnLike`와 와일드카드(`AWSReservedSSO_ReadOnly_*`)를 쓰라고 AWS가 권장한다.
- 이 role은 Identity Center의 service-linked role(`AWSServiceRoleForSSO`)이 관리한다. IAM에서 직접 수정하면 보호된 role이라 거부된다(이번에는 문제 해결 페이지의 검색 요약으로만 확인, 확인 필요).

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "*" },
  "Action": "s3:GetObject",
  "Resource": "arn:aws:s3:::team-reports/*",
  "Condition": {
    "ArnLike": {
      "aws:PrincipalArn": "arn:aws:iam::333333333333:role/aws-reserved/sso.amazonaws.com/*/AWSReservedSSO_ReadOnly_*"
    }
  }
}
```

permission set으로 표현할 수 없는 것(직접 정한 trust policy, role 태그, role 경로)이 필요하면, IAM의 새 기능인 **account access manager**로 기존 IAM role을 Identity Center 사용자·그룹에 할당할 수 있다. (같은 출처)

### 위임 관리자

출처: [Delegated administration](https://docs.aws.amazon.com/singlesignon/latest/userguide/delegated-admin.html)

- organization instance는 **항상 management 계정에 있다**. 하지만 관리 작업은 member 계정(보통 Infrastructure OU의 Identity 계정)에 위임할 수 있고, AWS도 이를 권장한다.
- 위임 관리자는 **management 계정에 프로비저닝된 permission set을 바꿀 수 없다.** management 계정 접근은 management 계정에서 따로 관리한다.
- AWS 권장 사항:
  - management 계정 permission set은 그룹이 아니라 **사용자에게 직접** 할당한다. 그룹에 할당하면 그룹 멤버십을 바꿀 수 있는 사람이 management 계정 접근을 얻게 된다.
  - SCP로 `identitystore:CreateGroupMembership`, `sso-directory:CreateBearerToken`을 제한한다.

### ABAC

Identity source의 사용자 속성(부서, 팀 등)을 **session tag**로 넘겨, 정책에서 `aws:PrincipalTag/<key>`로 쓸 수 있다. 외부 IdP는 SAML 속성 `https://aws.amazon.com/SAML/Attributes/AccessControl:<TagKey>`로 보낼 수도 있다. Identity Center에서 설정한 매핑이 SAML 값보다 우선한다. ([Attributes for access control](https://docs.aws.amazon.com/singlesignon/latest/userguide/attributesforaccesscontrol.html))

permission set 하나로 "자기 팀 태그가 붙은 리소스만" 다루게 할 수 있다. 정책 예시는 [aws-iam/02 ABAC](../aws-iam/02-policies.md#abac속성-기반-접근-제어)를 참고한다.

### MFA와 리전

| 항목 | 내용 | 출처 |
| --- | --- | --- |
| MFA | 기본으로 켜져 있음. 기본 모드는 로그인할 때마다. 사용자당 인증 앱 2개, FIDO 인증기 6개까지 | [Enable MFA](https://docs.aws.amazon.com/singlesignon/latest/userguide/enable-mfa.html) |
| 외부 IdP 사용 시 MFA | Identity Center MFA는 **지원 안 됨**. IdP에서 MFA를 강제해야 함 | 같은 출처 |
| 리전 | 기본 리전 하나에서 동작. 2026-02-03부터 **추가 리전으로 복제** 가능(organization instance, multi-Region KMS 키 필요, AD 미지원) | [Multi-Region](https://docs.aws.amazon.com/singlesignon/latest/userguide/multi-region-iam-identity-center.html) |
| 세션 | access portal 세션 기본 8시간(최대 90일), 계정 role 세션은 permission set 설정(최대 12시간) | [Permission sets](https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html) |

## 사용법 / 예시

### 사용자: CLI로 SSO 로그인

출처: [Configure IAM Identity Center authentication with the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html)

```bash
aws configure sso
# SSO session name: my-sso
# SSO start URL: https://my-sso-portal.awsapps.com/start
# SSO region: ap-northeast-2
# SSO registration scopes: sso:account:access
# → 브라우저 로그인 → 계정과 role(permission set) 선택 → 프로필 이름 입력
```

생성되는 `~/.aws/config`:

```ini
[sso-session my-sso]
sso_start_url = https://my-sso-portal.awsapps.com/start
sso_region = ap-northeast-2
sso_registration_scopes = sso:account:access

[profile payments-prod-readonly]
sso_session = my-sso
sso_account_id = 333333333333
sso_role_name = ReadOnly
region = ap-northeast-2
```

```bash
aws sso login --sso-session my-sso            # 토큰은 ~/.aws/sso/cache 에 저장
aws s3 ls --profile payments-prod-readonly    # 토큰이 유효한 동안 자격증명 자동 갱신
aws sso logout
```

- `sso_role_name`은 permission set 이름이다.
- `sso-session` 하나를 여러 프로필이 같이 쓴다. 계정마다 프로필만 추가하면 된다.
- AWS CLI v2.22.0부터 브라우저 로그인 기본 방식이 PKCE로 바뀌었다. 브라우저가 없는 환경에서는 `--use-device-code`를 쓴다.

### 관리자: permission set 만들고 할당

```bash
INSTANCE_ARN=$(aws sso-admin list-instances --query 'Instances[0].InstanceArn' --output text)

PS_ARN=$(aws sso-admin create-permission-set \
  --instance-arn "$INSTANCE_ARN" \
  --name ReadOnly \
  --session-duration PT8H \
  --query PermissionSet.PermissionSetArn --output text)

aws sso-admin attach-managed-policy-to-permission-set \
  --instance-arn "$INSTANCE_ARN" \
  --permission-set-arn "$PS_ARN" \
  --managed-policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# 그룹 ID는 identitystore에서 조회 (list-groups 등)
aws sso-admin create-account-assignment \
  --instance-arn "$INSTANCE_ARN" \
  --target-id 333333333333 --target-type AWS_ACCOUNT \
  --permission-set-arn "$PS_ARN" \
  --principal-type GROUP --principal-id <group-id>
```

- `create-account-assignment`는 비동기다. `describe-account-assignment-creation-status`로 결과를 확인한다.
- 이미 할당된 permission set의 정책을 바꿨다면 `provision-permission-set`으로 계정에 다시 반영해야 한다.
- 위 명령들은 CLI 레퍼런스의 옵션으로 조합한 것이다(레퍼런스에 예시가 없음). 실제 조직에서 실행하지는 않았다.

출처: [create-permission-set](https://docs.aws.amazon.com/cli/latest/reference/sso-admin/create-permission-set.html), [attach-managed-policy-to-permission-set](https://docs.aws.amazon.com/cli/latest/reference/sso-admin/attach-managed-policy-to-permission-set.html), [create-account-assignment](https://docs.aws.amazon.com/cli/latest/reference/sso-admin/create-account-assignment.html), [provision-permission-set](https://docs.aws.amazon.com/cli/latest/reference/sso-admin/provision-permission-set.html)

## 이슈 / 주의

- **customer managed policy 참조는 계정마다 그 정책이 있어야 한다.** 없으면 프로비저닝이 실패한다. StackSets 등으로 미리 배포한다.
- **SCIM 토큰 만료(1년)**를 놓치면 퇴사자가 IdP에서 지워져도 AWS에는 남는다.
- **`AWSReservedSSO_` role ARN을 정책에 그대로 박지 않는다.** 재할당하면 접미사가 바뀐다.
- **management 계정 접근은 그룹이 아니라 개인 할당으로** 최소화한다.
- **외부 IdP를 쓰면 MFA는 IdP의 책임**이다. Identity Center 쪽 MFA 설정은 적용되지 않는다.

## 출처

- https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/identity-center-instances.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/manage-your-identity-source.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/provision-automatically.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/permissionsetsconcept.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/referencingpermissionsets.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/delegated-admin.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/attributesforaccesscontrol.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/enable-mfa.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/multi-region-iam-identity-center.html
- https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-sso.html
- https://docs.aws.amazon.com/cli/latest/reference/sso-admin/create-account-assignment.html
- https://aws.amazon.com/iam/identity-center/faqs/

---
*조사 기준일: 2026-10-13. 다중 리전 복제와 account access manager는 2026년에 나온 기능이라 세부 조건이 바뀔 수 있다.*
