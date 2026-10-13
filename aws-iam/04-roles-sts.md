# Role·STS·교차 계정

> Role은 **"누가 맡을 수 있나"(trust policy) + "맡으면 무엇을 할 수 있나"(permissions policy)**의 묶음이고, STS `AssumeRole`로 **임시 자격증명**(기본 1시간, 최대 12시간)을 받아 쓴다. 교차 계정 접근의 표준 방법이며, 서드파티에 role을 열어 줄 때는 **external ID**로 confused deputy를 막는다.

## 개요 / 정의

> "When you create a role, you create two policies: a role trust policy that specifies who can assume the role, and a permissions policy that specifies what can be done with the role." — [AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html)

| 구성 요소 | 역할 | 예 |
| --- | --- | --- |
| Trust policy | role을 assume할 수 있는 principal(사람·계정·서비스·IdP) 지정. role에 붙는 **resource-based policy** | `"Principal": {"AWS": "arn:aws:iam::444455556666:root"}` |
| Permissions policy | role session이 할 수 있는 일 | `s3:GetObject` on `my-bucket/*` |
| Permissions boundary (선택) | role 권한의 상한선 | — |
| Max session duration | 이 role로 받을 수 있는 세션 최대 길이 | 1시간(기본) ~ 12시간 |

assume하면 세션 ARN은 `arn:aws:sts::<계정>:assumed-role/<role 이름>/<session 이름>` 형식이 된다.

## STS API 비교

출처: [Compare STS credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_sts-comparison.html)

| API | 호출 주체 | 세션 길이(최소 / 최대 / 기본) | MFA | Session policy | 쓰임새 |
| --- | --- | --- | --- | --- | --- |
| `AssumeRole` | 자격증명이 있는 IAM user·role | 15분 / role 최대값 / 1시간 | 가능 | 가능 | 교차 계정, 권한 상승 분리 |
| `AssumeRoleWithWebIdentity` | OIDC JWT를 가진 누구나 | 15분 / role 최대값 / 1시간 | 불가 | 가능 | GitHub Actions, EKS IRSA, 모바일 |
| `AssumeRoleWithSAML` | SAML 응답을 가진 누구나 | 15분 / role 최대값 / 1시간 | 불가 | 가능 | 사내 IdP SSO |
| `GetSessionToken` | IAM user·root | user: 15분 / 36시간 / 12시간 | 가능 | 불가 | IAM user에 MFA 세션 부여 |
| `GetFederationToken` | IAM user·root | user: 15분 / 36시간 / 12시간 | 불가 | 가능 | 레거시 프록시형 페더레이션 |

- `AssumeRoleWithWebIdentity`와 `AssumeRoleWithSAML`은 AWS 자격증명으로 서명하지 않는다. 토큰 자체가 증명이다. ([Requesting temporary credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_request.html))
- `AssumeRole` 계열로 받은 자격증명으로는 `GetSessionToken`·`GetFederationToken`을 호출할 수 없다.

## 세션 길이와 role chaining

| 항목 | 값 | 출처 |
| --- | --- | --- |
| `DurationSeconds` | 900–43200초, 기본 3600. role의 max session duration을 넘을 수 없음 | [AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) |
| Role max session duration | 1–12시간, 기본 1시간. AWS 서비스가 assume하는 세션에는 적용 안 됨 | [Update role settings](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_update-role-settings.html) |
| **Role chaining** | role로 다른 role을 assume하면 **최대 1시간**. 1시간 초과를 요청하면 실패 | [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html) |
| Session policy | inline 1개 + managed ARN 최대 10개, 합쳐서 평문 2,048자 이내 | [AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html) |

```bash
aws iam update-role --role-name AppRole --max-session-duration 43200
```

## 교차 계정 접근

### 1) 리소스 계정(444455556666)에 role 만들기 — trust policy

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::111122223333:role/DeployRole" },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Principal 작성 방식에 따른 차이([Principal element](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html)):

| Principal | 의미 | 주의 |
| --- | --- | --- |
| `arn:aws:iam::111122223333:root` (또는 `"111122223333"`) | 계정에 **권한을 위임**. root user만 뜻하는 게 아님 | 111122223333 쪽에서 누구에게 `sts:AssumeRole`을 줄지 관리자가 정함 |
| `arn:aws:iam::111122223333:role/DeployRole` | 특정 role만 | 저장 시 **고유 principal ID로 변환**됨. role을 삭제 후 재생성하면 trust가 깨짐 |
| `"*"` + `aws:PrincipalArn`·`aws:PrincipalOrgID` 조건 | 조건으로 범위 지정 | 조건 없는 `"*"`는 누구나 assume 가능 → 절대 금지 |

ARN 중간에 부분 와일드카드(`role/Deploy*`)는 Principal에 쓸 수 없다. 이런 경우 `"*"`와 `aws:PrincipalArn` 조건(`ArnLike`)을 조합한다.

### 2) 요청자 계정(111122223333) — identity policy

교차 계정은 양쪽이 모두 허용해야 하므로(→ [03-policy-evaluation.md](./03-policy-evaluation.md#교차-계정-평가)), 요청자 쪽에도 `sts:AssumeRole` 권한이 필요하다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "arn:aws:iam::444455556666:role/CrossAccountReadOnly"
    }
  ]
}
```

### 3) assume해서 쓰기

```bash
# 직접 호출
aws sts assume-role \
  --role-arn arn:aws:iam::444455556666:role/CrossAccountReadOnly \
  --role-session-name kenny-readonly \
  --duration-seconds 3600

# 결과: Credentials(AccessKeyId, SecretAccessKey, SessionToken, Expiration)
#      AssumedRoleUser.Arn = arn:aws:sts::444455556666:assumed-role/CrossAccountReadOnly/kenny-readonly
```

매번 직접 부르기보다 CLI 프로필에 등록해 두는 편이 편하다. ([Use an IAM role in the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html))

```ini
# ~/.aws/config
[profile prod-readonly]
role_arn = arn:aws:iam::444455556666:role/CrossAccountReadOnly
source_profile = default
role_session_name = kenny-readonly
# mfa_serial = arn:aws:iam::111122223333:mfa/kenny   # MFA 필수 role일 때
# external_id = ...                                  # external ID가 필요할 때
```

```bash
aws s3 ls --profile prod-readonly
aws sts get-caller-identity --profile prod-readonly
```

- `source_profile` 대신 `credential_source = Ec2InstanceMetadata | EcsContainer | Environment`를 쓸 수 있다.
- 받은 자격증명은 `~/.aws/cli/cache`에 캐시된다.

## Confused deputy와 external ID

> confused deputy: "an entity that doesn't have permission to perform an action can coerce a more-privileged entity to perform the action." — [The confused deputy problem](https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html)

### 교차 계정(서드파티 SaaS에 role을 열어 줄 때)

서드파티(예: 모니터링 SaaS)가 여러 고객 계정의 role을 assume한다. 공격자가 내 role ARN을 알아내 자기 설정에 넣으면, SaaS가 공격자 대신 내 계정에 접근하게 된다. 이를 막기 위해 **서드파티가 고객마다 발급·관리하는 external ID**를 trust policy 조건으로 건다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "AWS": "arn:aws:iam::999988887777:root" },
      "Action": "sts:AssumeRole",
      "Condition": { "StringEquals": { "sts:ExternalId": "customer-7f3a9c" } }
    }
  ]
}
```

- external ID는 고객이 아니라 **서드파티가 고유하게 생성·통제**해야 한다. (같은 출처)
- 길이 2–1224자, 허용 문자 `[\w+=,.@:\/-]`. ([AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html))
- external ID는 비밀번호가 아니다. 목적은 "다른 고객의 role을 대신 assume하는 것"을 막는 데 있다.

### 교차 서비스(AWS 서비스 principal에 권한을 줄 때)

S3 버킷 정책 등에서 `cloudtrail.amazonaws.com` 같은 **서비스 principal**에 권한을 줄 때는 `aws:SourceArn`, `aws:SourceAccount`(또는 `aws:SourceOrgID`)로 "어느 계정·리소스를 위한 호출인지"를 제한한다.

```json
{
  "Effect": "Allow",
  "Principal": { "Service": "cloudtrail.amazonaws.com" },
  "Action": "s3:PutObject",
  "Resource": "arn:aws:s3:::my-trail-bucket/AWSLogs/111122223333/*",
  "Condition": { "StringEquals": { "aws:SourceAccount": "111122223333" } }
}
```

## Session tags와 source identity

| 기능 | 무엇 | 필요한 권한 | 핵심 규칙 |
| --- | --- | --- | --- |
| **Session tags** | assume할 때 세션에 key-value 태그를 붙임 → 정책에서 `aws:PrincipalTag/<key>`로 참조(ABAC) | `sts:TagSession` | 최대 50개(key 128자, value 256자). 기본적으로 다음 role chaining으로 전달되지 않음. `TransitiveTagKeys`로 지정한 것만 전달. 콘솔 Switch Role로는 못 넘김 |
| **Source identity** | 세션을 시작한 "원래 사람"을 기록 → CloudTrail에 남고 chaining해도 유지 | `sts:SetSourceIdentity` | 세션 중 변경 불가. 조건 키 `sts:SourceIdentity`(설정 요청), `aws:SourceIdentity`(이후 요청) |

출처: [Session tags](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_session-tags.html), [Monitor and control actions with assumed roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_control-access_monitor.html)

- source identity 최대 길이는 IAM 가이드에 256자, AssumeRole API 레퍼런스에 64자로 서로 다르게 적혀 있다. 실제로는 API 제한(64자)을 기준으로 잡는 것이 안전하다(확인 필요).

```bash
aws sts assume-role \
  --role-arn arn:aws:iam::111122223333:role/DataRole \
  --role-session-name kenny \
  --tags Key=team,Value=data Key=project,Value=alpha \
  --transitive-tag-keys team \
  --source-identity kenny@example.com
```

이때 trust policy에 `sts:TagSession`, `sts:SetSourceIdentity`도 함께 허용해야 한다.

```json
{
  "Effect": "Allow",
  "Principal": { "AWS": "arn:aws:iam::111122223333:root" },
  "Action": ["sts:AssumeRole", "sts:TagSession", "sts:SetSourceIdentity"]
}
```

## STS 엔드포인트

출처: [Manage STS in an AWS Region](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_region-endpoints.html), [STS Regionalized endpoints](https://docs.aws.amazon.com/sdkref/latest/guide/feature-sts-regionalized-endpoints.html)

- AWS는 **리전 엔드포인트**(`sts.ap-northeast-2.amazonaws.com`) 사용을 권장한다.
- 글로벌 엔드포인트(`sts.amazonaws.com`)로 보낸 요청은 기본 활성 리전에서는 요청이 출발한 리전에서 처리된다. 단, opt-in 리전에서 보냈거나 Amazon DNS가 아닌 리졸버를 쓰면 us-east-1로 간다. 이 경우 CloudTrail 기록도 us-east-1에 남는다.
- SDK 설정은 `sts_regional_endpoints`(환경변수 `AWS_STS_REGIONAL_ENDPOINTS`)로 바꾼다. 2022년 7월 이후 나온 SDK 메이저 버전은 기본값이 `regional`이다.

| 기본값 | SDK·도구 |
| --- | --- |
| `regional` | AWS CLI v2, Java 2.x, JS v3, Go v2 |
| `legacy` (글로벌) | AWS CLI v1, Java 1.x, JS v2, Go v1 |

구버전 SDK와 private 서브넷(VPC endpoint)을 함께 쓸 때는 `regional`로 명시해 두는 것이 안전하다.

## 탈취된 세션 무효화

role 세션은 발급 후 만료 전까지 유효하다. 탈취가 의심되면 **그 시각 이전에 발급된 세션을 모두 거부**한다. ([Revoke role session credentials](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_revoke-sessions.html))

- 콘솔의 "Revoke active sessions"는 role에 `AWSRevokeOlderSessions` inline 정책을 붙인다. 클릭 시각에 약 30초를 더한 시점 이전의 세션을 거부한다. `iam:PutRolePolicy` 권한이 필요하다.
- 같은 내용을 직접 넣을 수도 있다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Action": "*",
      "Resource": "*",
      "Condition": { "DateLessThan": { "aws:TokenIssueTime": "2026-10-13T09:00:00Z" } }
    }
  ]
}
```

- `aws:TokenIssueTime`은 임시 자격증명 요청에만 있으므로 IAM user 장기 액세스 키에는 영향이 없다. 장기 키는 비활성화하거나 삭제해야 한다.
- service-linked role의 세션은 이 방식으로 무효화할 수 없다.

## 이슈 / 주의

- **trust policy가 role ARN을 가리키면 role 재생성에 약하다.** 고유 ID로 저장되므로, 같은 이름으로 다시 만들어도 trust가 깨진다. 이 경우 정책에 알 수 없는 ID(`AROA…`)가 보인다.
- **Role chaining은 1시간 제한**이다. CI에서 role → role로 넘어가는 구조라면 긴 작업이 만료로 실패할 수 있다.
- **`Principal: "*"` + 조건 없음**은 누구나 assume할 수 있다는 뜻이다. IAM Access Analyzer가 외부 접근으로 잡아 준다(→ [06-least-privilege.md](./06-least-privilege.md)).
- **콘솔 Switch Role 제약**: session tag를 넘길 수 없고, source identity가 필요한 role로는 전환할 수 없다.

## 출처

- https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_sts-comparison.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_request.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_update-role-settings.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/confused-deputy.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_session-tags.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_control-access_monitor.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp_region-endpoints.html
- https://docs.aws.amazon.com/sdkref/latest/guide/feature-sts-regionalized-endpoints.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_revoke-sessions.html
- https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-role.html
- https://docs.aws.amazon.com/cli/latest/reference/sts/assume-role.html

---
*조사 기준일: 2026-10-13. CLI 예시는 공식 레퍼런스의 옵션으로 조합했고, 실제 계정에서 실행하지는 않았다.*
