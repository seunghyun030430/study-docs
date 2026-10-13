# 정책 종류와 문법

> 권한을 **주는** 정책은 identity-based, resource-based(+ACL) 두 종류뿐이다. 나머지(permissions boundary, SCP, RCP, session policy, VPC endpoint policy)는 **상한선만 정한다**. 문법에서 가장 자주 틀리는 곳은 `Condition`이다. 여러 키는 AND, 한 키의 여러 값은 OR이고, `ForAllValues`는 키가 없으면 **참**이 된다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)의 용어(정책, identity-based, resource-based, principal 등)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Inline policy | 특정 user·role 하나에 직접 넣는 정책. 다른 곳에 재사용할 수 없음 |
| Permissions boundary | user·role 하나에 거는 **권한 상한선**. 이것만으로는 권한이 생기지 않음 |
| SCP / RCP | AWS Organizations(여러 계정 묶음)에서 계정 단위로 거는 상한선. SCP는 계정 안의 user·role, RCP는 계정 안의 리소스가 대상 |
| Session policy | role을 맡을 때 그 세션에만 추가로 거는 상한선 |
| ACL | 버킷 등에 붙는 옛 방식의 접근 목록(JSON 정책이 아님) |
| VPC / VPC endpoint | VPC는 AWS 안의 내 사설 네트워크. VPC endpoint는 인터넷을 거치지 않고 VPC에서 AWS 서비스로 가는 전용 통로 |
| AWS RAM | 리소스를 다른 계정과 공유하는 서비스(Resource Access Manager) |
| KMS / key policy | KMS는 암호화 키 관리 서비스. key policy는 키마다 붙는 resource-based policy |
| CIDR | `10.0.0.0/16`처럼 IP 주소 범위를 적는 표기 |
| 태그(tag) | 리소스·신원에 붙이는 `key=value` 꼬리표(예: `team=data`) |
| ABAC | 태그를 비교해 허용 여부를 정하는 방식(이 문서 끝에서 설명) |
| Service Authorization Reference | 서비스별 액션·리소스 타입·조건 키를 정리한 AWS 공식 목록 |
| FAS(Forward Access Sessions) | AWS 서비스가 사용자 대신 다른 서비스를 호출하는 것(예: CloudFormation이 EC2 생성) |

## 정책 종류

AWS는 현재 9가지 정책 유형을 안내한다. ([Policies and permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html))

| 유형 | 권한 부여? | 붙는 곳 | 비고 |
| --- | --- | --- | --- |
| **Identity-based** (AWS managed / customer managed / inline) | **부여** | IAM user, group, role | 가장 기본 |
| **Resource-based** | **부여**(교차 계정 포함) | 리소스(S3 버킷 정책, KMS key policy, role trust policy 등). 항상 inline | `Principal` 필수 |
| ACL | 부여(교차 계정만) | S3, WAF, VPC 등 | JSON 아님. 레거시 |
| Permissions boundary | 제한만 | user, role(managed policy로) | resource-based로 받은 권한은 제한하지 못함(→ [03](./03-policy-evaluation.md#resource-based-policy의-미묘한-점)) |
| SCP (Organizations) | 제한만 | 조직 root, OU, 계정 | 멤버 계정의 principal 제한 |
| RCP (Organizations) | 제한만 | 조직 root, OU, 계정 | 멤버 계정의 리소스 제한. 2024-11-13 출시 |
| Session policy | 제한만 | `AssumeRole*`, `GetFederationToken` 호출 시 전달 | 그 세션에만 적용 |
| VPC endpoint policy | 제한만 | VPC endpoint | endpoint를 지나는 요청의 추가 경계 |
| RAM resource share | 공유 | 계정, OU, 조직 | 리소스 공유 |

### Identity-based 정책 세 가지

| 종류 | 관리 주체 | 재사용 | 버전 관리 | 언제 |
| --- | --- | --- | --- | --- |
| AWS managed | AWS | 여러 엔티티 | AWS가 갱신(권한이 늘어날 수 있음) | 시작점. 운영에서는 좁혀 나감 |
| Customer managed | 나 | 여러 엔티티 | 최대 5개 버전 보관(확인 필요), 기본 버전 지정 | **기본 선택** |
| Inline | 나 | 엔티티 1개에 종속 | 없음 | 그 엔티티에만 엄격히 묶어야 할 때 |

AWS managed policy의 ARN은 `arn:aws:iam::aws:policy/<이름>` 형식이다(이번 조사에서는 공식 페이지 문구를 직접 확인하지 못함, 확인 필요).

## 정책 문법

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadOwnPrefix",
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject"],
      "Resource": "arn:aws:s3:::my-bucket/home/${aws:username}/*",
      "Condition": {
        "Bool": { "aws:SecureTransport": "true" },
        "StringEquals": { "aws:RequestedRegion": "ap-northeast-2" }
      }
    }
  ]
}
```

| 요소 | 필수 | 설명 |
| --- | --- | --- |
| `Version` | 사실상 필수 | 항상 `2012-10-17`. 예전 `2008-10-17`에서는 정책 변수가 동작하지 않고 문자열 그대로 취급됨 ([Version](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_version.html)) |
| `Statement` | 필수 | 하나 이상의 statement. 여러 statement와 여러 정책은 OR로 합쳐짐 |
| `Sid` | 선택 | statement 식별자 |
| `Effect` | 필수 | `Allow` 또는 `Deny` |
| `Principal` / `NotPrincipal` | resource-based에서 필수 | identity-based에는 쓰지 않음 |
| `Action` / `NotAction` | 필수(둘 중 하나) | `서비스:액션`, 와일드카드 가능 |
| `Resource` / `NotResource` | 대부분 필수 | ARN. 정책 변수 사용 가능 |
| `Condition` | 선택 | request context 키 검사 |

### `Not*` 요소 주의

- **`NotAction` + `Allow`**: "나열한 것 빼고 전부 허용"이 된다. AWS는 의도보다 많은 권한을 줄 수 있다고 경고한다. ([NotAction](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_notaction.html)) `Deny` + `NotAction`("이것 말고는 다 거부")은 가드레일로 흔히 쓴다.
- **`NotPrincipal`**:
  - `Deny`와만 쓸 수 있다.
  - identity-based policy와 role trust policy에서는 쓸 수 없다.
  - SCP·RCP에서도 지원하지 않는다.
  - AWS는 새 정책에 쓰지 말고 `aws:PrincipalArn` + `ArnNotEquals` 조건을 쓰라고 권장한다. ([NotPrincipal](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_notprincipal.html))

## Condition

### 연산자

출처: [Condition operators](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html)

| 분류 | 연산자 | 메모 |
| --- | --- | --- |
| 문자열 | `StringEquals`, `StringNotEquals`, `StringEqualsIgnoreCase`, `StringLike`(`*`, `?`) | `StringEquals`는 대소문자 구분 |
| 숫자 | `NumericEquals`, `NumericLessThan` 등 | — |
| 날짜 | `DateGreaterThan`, `DateLessThan` 등 | ISO 8601 또는 epoch |
| 불리언 | `Bool` | `aws:SecureTransport` 등 |
| IP | `IpAddress`, `NotIpAddress` | CIDR. 접두사가 없으면 `/32` |
| ARN | `ArnEquals`, `ArnLike` | 둘은 동일하게 동작 |
| 존재 여부 | `Null` | `true` = 키가 없음, `false` = 키가 있음 |
| `…IfExists` | 예: `StringEqualsIfExists` | 키가 없으면 조건을 **참**으로 봄 |

**키가 요청에 없으면** 일반 연산자는 거짓, 부정 연산자(`StringNotEquals` 등)는 참이 된다. `…IfExists`와 `Null`은 이 규칙의 예외다.

### 여러 키·여러 값의 결합

출처: [Multiple context keys or values](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-logic-multiple-context-keys-or-values.html)

| 구조 | 결합 |
| --- | --- |
| 서로 다른 연산자 블록 | AND |
| 한 연산자 안의 여러 키 | AND |
| 한 키의 여러 값 | OR |
| 부정 연산자 + 여러 값 | NOR(어느 것과도 일치하지 않아야 함) |

```json
"Condition": {
  "StringEquals": {
    "aws:RequestedRegion": ["ap-northeast-2", "us-east-1"],
    "aws:PrincipalTag/team": "data"
  }
}
```

이 조건은 "(리전이 서울 **또는** 버지니아) **그리고** (principal 태그 team이 data)"를 뜻한다.

### `ForAllValues` / `ForAnyValue`

출처: [Single-valued vs multivalued context keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-single-vs-multi-valued-context-keys.html)

- `aws:TagKeys`, `aws:CalledVia` 같은 **다중 값 키에만** 쓴다. 단일 값 키에 쓰면 정책이 지나치게 관대해질 수 있다.
- **빈 집합 함정**: `ForAllValues`는 요청에 그 키가 **아예 없어도 참**이다. 태그 없이 요청하면 조건을 통과해 버린다. `"Null": {"aws:TagKeys": "false"}`를 함께 건다.
- `ForAnyValue`는 키가 없으면 거짓이다.

```json
{
  "Effect": "Allow",
  "Action": "ec2:CreateTags",
  "Resource": "*",
  "Condition": {
    "ForAllValues:StringEquals": { "aws:TagKeys": ["team", "project"] },
    "Null": { "aws:TagKeys": "false" }
  }
}
```

### 정책 변수

- `Resource`에서는 ARN의 다섯 번째 콜론 뒤에서만 쓸 수 있다.
- `Condition`에서는 문자열 비교에 쓸 수 있다.
- 다중 값 키는 변수로 쓸 수 없다.

([Policy variables](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_variables.html))

예: `${aws:username}`, `${aws:PrincipalTag/team}`, `${aws:userid}`.

### 자주 쓰는 글로벌 조건 키

출처: [Global condition context keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html)

| 키 | 언제 있나 | 쓰임새 |
| --- | --- | --- |
| `aws:PrincipalOrgID` | principal이 조직에 속할 때 | resource policy에서 "우리 조직만" 허용 |
| `aws:PrincipalArn` | 항상 | role이면 **role ARN**(세션 ARN 아님). 특정 role 예외 처리 |
| `aws:SourceIp` | 퍼블릭 IP 요청일 때. **VPC endpoint를 거치면 없음** | IP 제한(VPC 안에서는 `aws:VpcSourceIp` 사용) |
| `aws:SourceVpce`, `aws:SourceVpc` | VPC endpoint를 거칠 때 | 특정 endpoint로만 접근 허용 |
| `aws:SecureTransport` | 항상 | HTTPS 강제 |
| `aws:MultiFactorAuthPresent` | MFA 기반 임시 자격증명일 때. 장기 키 요청에는 없음 | MFA 강제(`…IfExists` 주의) |
| `aws:RequestedRegion` | 항상 | 리전 제한(IAM·STS 글로벌 엔드포인트는 us-east-1로 보임) |
| `aws:ResourceTag/<k>`, `aws:RequestTag/<k>`, `aws:TagKeys` | 태그가 있는 리소스·태그 요청 | ABAC |
| `aws:SourceArn`, `aws:SourceAccount` | 서비스 principal이 리소스를 대신해 호출할 때 | confused deputy 방지(→ [04](./04-roles-sts.md#confused-deputy와-external-id)) |
| `aws:ResourceOrgID` | 리소스 계정이 조직에 속할 때 | "우리 조직 리소스에만" 접근 |
| `aws:CalledVia`, `aws:ViaAWSService` | 서비스가 사용자 대신 다른 서비스를 호출할 때(FAS) | 예: CloudFormation을 통해서만 허용 |

`aws:PrincipalOrgID`, `aws:SourceAccount`, `aws:SourceVpce` 같은 **민감한 키에는 와일드카드를 쓰지 않는다.** (같은 출처)

`aws:MultiFactorAuthPresent`로 `Deny` + `Bool: false` 패턴을 쓰면, 장기 키 요청처럼 키가 아예 없는 경우를 잡지 못한다. AWS 문서는 이 패턴을 "NOT RECOMMENDED"로 표시한다. MFA 강제에는 `BoolIfExists`를 쓴다.

```json
{
  "Sid": "DenyWithoutMFA",
  "Effect": "Deny",
  "NotAction": ["iam:ChangePassword", "iam:CreateVirtualMFADevice", "iam:EnableMFADevice", "sts:GetSessionToken"],
  "Resource": "*",
  "Condition": { "BoolIfExists": { "aws:MultiFactorAuthPresent": "false" } }
}
```

## ABAC(속성 기반 접근 제어)

**principal 태그와 리소스 태그가 같을 때 허용**하는 방식이다. 새 리소스나 팀이 생겨도 정책을 고칠 필요가 없고, 정책 수도 줄어든다. 사내 디렉터리 속성을 SAML·OIDC session tag로 넘겨 쓸 수도 있다. ([ABAC](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction_attribute-based-access-control.html))

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["ec2:StartInstances", "ec2:StopInstances"],
      "Resource": "arn:aws:ec2:*:111122223333:instance/*",
      "Condition": {
        "StringEquals": { "aws:ResourceTag/team": "${aws:PrincipalTag/team}" }
      }
    }
  ]
}
```

ABAC의 함정: 사용자가 **리소스나 자기 자신의 태그를 바꿀 수 있으면** 경계가 무너진다. `ec2:CreateTags`, `iam:TagRole`, `iam:TagUser` 같은 태그 변경 권한을 함께 통제한다.

## 이슈 / 주의

- **AWS managed policy는 AWS가 바꾼다.** 새 액션이 추가되면 권한이 자동으로 늘어날 수 있다. 운영 권한은 customer managed로 고정한다.
- **`Resource: "*"`이 불가피한 액션이 있다.** 리소스 수준 권한을 지원하지 않는 액션(예: 일부 `Describe*`)은 `*`만 받는다. Service Authorization Reference에서 액션별 리소스 타입을 확인한다.
- **조건 키 존재 여부**를 먼저 확인한다. 키가 없는 요청에서 조건이 어떻게 평가되는지(거짓 / 부정 연산자는 참 / `IfExists`는 참)가 보안 구멍의 대부분을 만든다.
- **inline 정책은 추적이 어렵다.** 같은 권한을 여러 곳에 inline으로 넣지 말고 customer managed로 공유한다.

## 출처

- https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_version.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_notaction.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_notprincipal.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_condition_operators.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-logic-multiple-context-keys-or-values.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-single-vs-multi-valued-context-keys.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_variables.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction_attribute-based-access-control.html
- https://aws.amazon.com/about-aws/whats-new/2024/11/resource-control-policies-restrict-access-aws-resources/

---
*조사 기준일: 2026-10-13. customer managed policy 버전 보관 개수(5개)와 AWS managed policy ARN 형식은 이번에 1차 출처로 다시 확인하지 못했다.*
