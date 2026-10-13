# SCP·RCP 설계

> SCP는 "계정 안의 **사람·role**이 할 수 있는 최대치", RCP는 "계정 안의 **리소스**에 허용될 수 있는 최대치"를 정한다. 둘 다 **권한을 주지 않는다**. 실무 기본형은 기본 허용 정책(FullAWSAccess)을 그대로 두고 **막을 것만 Deny로 추가하는 deny-list 방식**이다. 관리용 role은 `aws:PrincipalArn` 조건으로 예외 처리하고, 반드시 **테스트 OU → 하위 OU → 상위 OU** 순서로 넓혀 간다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)와 [aws-iam/03-policy-evaluation.md](../aws-iam/03-policy-evaluation.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 가드레일(guardrail) | 넘어가면 안 되는 선을 미리 긋는 통제. SCP·RCP가 대표적 |
| FullAWSAccess / RCPFullAWSAccess | 모든 조직 root·OU·계정에 기본으로 붙는 "전부 허용" SCP / RCP |
| Deny-list / allow-list | 기본은 다 허용하고 막을 것만 나열 / 기본은 다 막고 허용할 것만 나열 |
| 상속(inheritance) | 조직 root·상위 OU에 붙은 정책이 아래 OU·계정에도 적용되는 것 |
| Data perimeter(데이터 경계) | "우리 조직의 신원만, 우리 조직의 리소스에, 우리 네트워크에서만" 접근하도록 긋는 경계 |
| 글로벌 서비스 | IAM·Organizations·Route 53처럼 리전에 속하지 않고 us-east-1 등 한 곳에서 처리되는 서비스 |
| Privileged role | 가드레일 예외를 받는 관리용 role(예: 플랫폼팀 배포 role) |

## SCP 동작 방식

출처: [SCPs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html), [SCP evaluation](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_evaluation.html)

### 상속: Allow는 "모든 층", Deny는 "어느 한 층"

```
Root          [FullAWSAccess] [DenyLeaveOrg]
 └─ OU Workloads  [FullAWSAccess] [DenyRegions]
     └─ OU Prod       [FullAWSAccess]
         └─ 계정 payments-prod  [FullAWSAccess]
```

| 규칙 | 설명 |
| --- | --- |
| **Allow** | 조직 root부터 계정까지 경로의 **모든 층**에 명시적 Allow가 있어야 함. 한 층이라도 Allow가 없으면 거부 |
| **Deny** | 경로의 **어느 한 층**이라도 Deny하면 거부 |
| 기본값 | AWS가 모든 조직 root·OU·계정에 `FullAWSAccess`를 붙여 둠. 이걸 떼고 대체 정책을 안 붙이면 그 아래 **모든 작업이 실패** |

위 예에서 payments-prod 계정은 모든 층에 FullAWSAccess가 있어 기본적으로 허용되고, 조직 root의 `DenyLeaveOrg`와 Workloads OU의 `DenyRegions`가 함께 적용된다.

SCP를 끈 뒤 다시 켜면 모든 대상이 FullAWSAccess만 붙은 상태로 돌아가고, 기존에 붙여 둔 SCP는 사라진다.

### Deny-list vs allow-list

| 방식 | 방법 | 장점 | 단점 |
| --- | --- | --- | --- |
| **Deny-list**(권장 출발점) | FullAWSAccess 유지 + 막을 것을 Deny로 추가 | 새 서비스가 나와도 바로 쓸 수 있음. 실수로 다 막을 위험이 적음 | 막을 것을 빠짐없이 찾아야 함 |
| Allow-list | FullAWSAccess 제거 + 허용할 서비스만 Allow | 승인된 서비스만 쓰게 강하게 통제 | 새 서비스마다 정책 수정. 층마다 Allow가 필요해 관리가 어려움 |

AWS 공식 예시 저장소의 SCP들도 deny-list 방식으로 작성되어 있다. 그래서 그대로 쓰려면 FullAWSAccess(또는 다른 Allow 정책)가 함께 붙어 있어야 한다. ([SCP examples](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_examples.html))

### SCP 문법

2025-09-19부터 SCP는 IAM 정책 언어를 거의 전부 지원한다. Allow 문에서도 `Condition`, 개별 리소스 ARN, `NotAction`, `NotResource`를 쓸 수 있다. ([What's New](https://aws.amazon.com/about-aws/whats-new/2025/09/aws-organizations-iam-language-service-control-policies/), [SCP syntax](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_syntax.html))

- 쓸 수 없는 요소는 `Principal`과 `NotPrincipal`이다. 대상 principal을 고르려면 `aws:PrincipalArn` 같은 조건을 쓴다.
- 같은 문법 페이지에 "Allow 문의 Resource에는 `*`만 쓸 수 있다"는 옛 문장이 남아 있어 발표 내용과 충돌한다. Allow 문에 개별 ARN을 쓰는 동작은 직접 테스트해 확인할 것(확인 필요).

### SCP가 영향을 주는 것 / 주지 않는 것

| 대상 | 영향 |
| --- | --- |
| member 계정의 IAM user·role | **받음** |
| member 계정의 **root user** | **받음** |
| 위임 관리자 member 계정 | **받음** |
| management 계정의 user·role | 받지 않음 |
| service-linked role | 받지 않음 |
| 조직 밖 계정의 principal | 받지 않음 |
| resource-based policy | 직접 제한하지 않음. 예: 버킷 정책이 조직 밖 계정 B를 허용하면 SCP로는 막을 수 없음 → **RCP의 영역** |

## RCP 동작 방식

출처: [RCPs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html), [RCP syntax](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps_syntax.html)

| 항목 | 규칙 |
| --- | --- |
| Effect | 직접 만드는 RCP는 **Deny만 가능**. Allow는 기본 `RCPFullAWSAccess`에만 있음(떼어 낼 수 없음) |
| Principal | 필수이고 `"*"`만 가능. 특정 principal은 조건으로 고름 |
| 미지원 | `NotPrincipal`, `NotAction`. Action에 `"*"`만 단독으로 쓸 수 없고 서비스 접두사 필요 |
| 적용 대상 | member 계정의 리소스. **조직 밖 principal이나 root user가 접근해도 적용** |
| 적용 안 됨 | management 계정의 리소스, service-linked role, AWS managed KMS key, `kms:RetireGrant` |
| 지원 서비스 | S3, STS, KMS, SQS, Secrets Manager, DynamoDB, ECR, CloudWatch Logs, Cognito 등 약 60개(전체 목록은 RCP 문서) |

SCP와 RCP의 역할 분담:

| 막고 싶은 것 | 쓰는 정책 |
| --- | --- |
| 우리 직원·role이 금지된 일을 하는 것 | SCP |
| 조직 밖 누군가가 우리 리소스에 접근하는 것 | RCP |
| 우리 직원이 조직 밖 리소스로 데이터를 빼내는 것 | SCP(`aws:ResourceOrgID` 조건) |

## 가드레일 예시

**아래 정책을 그대로 조직 root에 붙이면 계정 전체가 막힐 수 있다.** 반드시 [안전한 테스트와 롤아웃](#안전한-테스트와-롤아웃) 절차를 따른다. `${Account}`, `[PRIVILEGED_ROLE]` 같은 자리는 실제 값으로 바꾼다.

### 1) 허용 리전 외 사용 금지 (SCP)

출처: [aws-samples/service-control-policy-examples — Region controls](https://github.com/aws-samples/service-control-policy-examples)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyOutsideAllowedRegions",
      "Effect": "Deny",
      "NotAction": [
        "a4b:*", "acm:*", "aws-marketplace-management:*", "aws-marketplace:*",
        "aws-portal:*", "budgets:*", "ce:*", "chime:*", "cloudfront:*",
        "config:*", "cur:*", "directconnect:*", "ec2:DescribeRegions",
        "ec2:DescribeTransitGateways", "ec2:DescribeVpnGateways", "fms:*",
        "globalaccelerator:*", "health:*", "iam:*", "importexport:*", "kms:*",
        "mobileanalytics:*", "networkmanager:*", "organizations:*", "pricing:*",
        "route53:*", "route53domains:*", "route53-recovery-cluster:*",
        "route53-recovery-control-config:*", "route53-recovery-readiness:*",
        "s3:GetAccountPublic*", "s3:ListAllMyBuckets",
        "s3:ListMultiRegionAccessPoints", "s3:PutAccountPublic*", "shield:*",
        "sts:*", "support:*", "trustedadvisor:*", "waf-regional:*", "waf:*",
        "wafv2:*", "wellarchitected:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": { "aws:RequestedRegion": ["ap-northeast-2", "us-east-1"] },
        "ArnNotLike": { "aws:PrincipalARN": "arn:aws:iam::*:role/[PRIVILEGED_ROLE]" }
      }
    }
  ]
}
```

- `NotAction` 목록은 **글로벌 서비스**를 예외로 둔 것이다. IAM·Organizations·Route 53 등은 us-east-1에서 처리되므로, 이걸 빼면 서울만 허용해도 IAM 작업이 막힌다.
- 허용 리전에 us-east-1을 함께 넣는 경우가 많다. CloudFront 인증서(ACM) 등 us-east-1에서만 되는 작업이 있기 때문이다.
- 원본 예시의 허용 리전은 `eu-central-1`, `eu-west-1`이고, 위에서는 서울로 바꿨다. 같은 폴더에 Bedrock 교차 리전 추론용 변형도 있다.

### 2) 조직 이탈 금지 (SCP)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Deny", "Action": "organizations:LeaveOrganization", "Resource": "*" }
  ]
}
```

2026-07-10 이후 콘솔로 만든 조직에는 이와 비슷한 SCP(`LeaveOrganization`, `account:CloseAccount` 거부)가 기본으로 붙는다(→ [02](./02-structure-ou.md#management-계정-최소화)).

### 3) 로그·보안 서비스 보호 (SCP)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ProtectOrgTrail",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:DeleteTrail", "cloudtrail:PutEventSelectors",
        "cloudtrail:StopLogging", "cloudtrail:UpdateTrail", "cloudtrail:CreateTrail"
      ],
      "Resource": "arn:aws:cloudtrail:*:*:trail/org-trail",
      "Condition": {
        "ArnNotLike": { "aws:PrincipalARN": "arn:aws:iam::*:role/[PRIVILEGED_ROLE]" }
      }
    }
  ]
}
```

같은 저장소에 AWS Config 끄기 금지(`StopConfigurationRecorder` 등 21개 액션), GuardDuty 비활성화 금지(`DeleteDetector` 등 31개 액션) 예시가 있다. 액션 목록이 길어 원본 파일에서 복사해 쓴다.

### 4) member 계정 root user 사용 금지 (SCP)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyRootUser",
      "Effect": "Deny",
      "NotAction": ["s3:GetBucketPolicy", "s3:PutBucketPolicy", "s3:DeleteBucketPolicy"],
      "Resource": "*",
      "Condition": { "ArnLike": { "aws:PrincipalArn": "arn:aws:iam::*:root" } }
    }
  ]
}
```

버킷 정책 작업을 예외로 둔 이유는, 잘못된 버킷 정책으로 모두가 잠겼을 때 root만 풀 수 있기 때문이다. root 자격증명 자체를 없애는 방법은 [05](./05-other-policies.md)에서 다룬다.

### 5) IMDSv2 강제 (SCP)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RequireImdsV2",
      "Effect": "Deny",
      "Action": "ec2:RunInstances",
      "Resource": "arn:aws:ec2:*:*:instance/*",
      "Condition": { "StringNotEquals": { "ec2:MetadataHttpTokens": "required" } }
    }
  ]
}
```

출처: [EC2 example policies](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ExamplePolicies_EC2.html#iam-example-instance-metadata-requireIMDSv2). 같은 효과를 **EC2 선언형 정책**으로 기본값 자체를 바꿔서 낼 수도 있다(→ [05](./05-other-policies.md)).

### 6) HTTPS만 허용 (RCP)

출처: [aws-samples/resource-control-policy-examples](https://github.com/aws-samples/resource-control-policy-examples)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceSecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "s3:*", "sts:*", "kms:*", "sqs:*", "secretsmanager:*",
        "cognito-identity:*", "cognito-idp:*", "logs:*", "dynamodb:*", "ecr:*", "aoss:*"
      ],
      "Resource": "*",
      "Condition": { "BoolIfExists": { "aws:SecureTransport": "false" } }
    }
  ]
}
```

### 7) 조직 밖 접근 차단 — identity perimeter (RCP)

출처: [aws-samples/data-perimeter-policy-examples — identity_perimeter_rcp.json](https://github.com/aws-samples/data-perimeter-policy-examples) (원본에서 S3 부분만 발췌·단순화)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceOrgIdentities",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {
        "StringNotEqualsIfExists": { "aws:PrincipalOrgID": "o-exampleorgid" },
        "BoolIfExists": { "aws:PrincipalIsAWSService": "false" }
      }
    },
    {
      "Sid": "EnforceConfusedDeputyProtection",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": "*",
      "Condition": {
        "StringNotEqualsIfExists": { "aws:SourceOrgID": "o-exampleorgid" },
        "Null": { "aws:SourceAccount": "false" },
        "Bool": { "aws:PrincipalIsAWSService": "true" }
      }
    }
  ]
}
```

| 문(statement) | 뜻 |
| --- | --- |
| EnforceOrgIdentities | 요청자가 우리 조직 소속이 아니면 거부. 단, **AWS 서비스 principal**(CloudTrail 등)은 이 규칙에서 제외 |
| EnforceConfusedDeputyProtection | AWS 서비스가 대신 접근할 때는 그 호출이 **우리 조직의 리소스를 위한 것**(`aws:SourceOrgID`)일 때만 허용([aws-iam/04 confused deputy](../aws-iam/04-roles-sts.md#confused-deputy와-external-id)) |

- 원본은 STS·KMS·SQS 등 여러 서비스를 함께 다루고, 외부 파트너 계정과 `dp:exclude:identity` 태그로 예외를 두는 조건도 포함한다. 실제 적용 시 원본 전체를 기준으로 한다.
- 외부 파트너·SaaS가 우리 버킷을 쓰고 있다면 이 RCP가 그 접근을 끊는다. 적용 전에 **IAM Access Analyzer의 외부 접근 결과**로 현재 외부 접근을 먼저 파악한다.

## 안전한 테스트와 롤아웃

출처: [SCPs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html), [RCPs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html), [RCP examples](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps_examples.html)

| 단계 | 할 일 |
| --- | --- |
| 1. 문법 검사 | `aws accessanalyzer validate-policy --policy-type SERVICE_CONTROL_POLICY`(RCP는 `RESOURCE_CONTROL_POLICY`) |
| 2. 영향 조사 | 막으려는 서비스를 실제로 누가 쓰는지 확인(Organizations 접근 보고서, CloudTrail). RCP는 Access Analyzer 외부 접근 결과도 확인 |
| 3. 테스트 OU | 테스트 OU(백서의 Policy Staging OU)에 붙이고, 계정을 **하나씩** 옮겨 시험 |
| 4. 점진 확대 | 하위 OU → 상위 OU 순으로 넓힘. 조직 root에 바로 붙이지 않음 |
| 5. 모니터링 | CloudTrail에서 SCP·RCP 때문에 생긴 AccessDenied를 찾음 |

```bash
# 1. 정책 문법·보안 경고 검사
aws accessanalyzer validate-policy \
  --policy-type SERVICE_CONTROL_POLICY \
  --policy-document file://deny-regions.json

# 2. 계정(또는 OU)이 실제로 쓰는 서비스 조회 — management 계정에서 실행
JOB=$(aws iam generate-organizations-access-report \
  --entity-path o-exampleorgid/r-examplerootid111/ou-exam-ple12345/333333333333 \
  --query JobId --output text)
aws iam get-organizations-access-report --job-id "$JOB"
```

접근 보고서에는 실패한 시도도 포함된다. AWS는 CloudTrail을 기준 자료로 보라고 안내한다. ([generate-organizations-access-report](https://docs.aws.amazon.com/cli/latest/reference/iam/generate-organizations-access-report.html))

### 거부 메시지 읽기

AccessDenied 메시지에 어떤 정책 유형이 막았는지가 나온다. ([Troubleshoot access denied](https://docs.aws.amazon.com/IAM/latest/UserGuide/troubleshoot_access-denied.html))

| 메시지 일부 | 의미 |
| --- | --- |
| `because no service control policy allows the <action> action` | 경로 어딘가에 SCP Allow가 없음(FullAWSAccess를 뗀 경우 등) |
| `with an explicit deny in a service control policy` | SCP의 Deny에 걸림 |
| `with an explicit deny in a resource control policy` | RCP의 Deny에 걸림 |

## 사용법 / 예시: CLI

```bash
# 정책 유형 켜기 (조직 root ID 필요)
aws organizations enable-policy-type --root-id r-examplerootid111 --policy-type SERVICE_CONTROL_POLICY
aws organizations enable-policy-type --root-id r-examplerootid111 --policy-type RESOURCE_CONTROL_POLICY

# 정책 만들고 테스트 OU에 붙이기
POLICY_ID=$(aws organizations create-policy \
  --name DenyOutsideAllowedRegions \
  --description "Allow only ap-northeast-2 and us-east-1" \
  --type SERVICE_CONTROL_POLICY \
  --content file://deny-regions.json \
  --query Policy.PolicySummary.Id --output text)
aws organizations attach-policy --policy-id "$POLICY_ID" --target-id ou-exam-ple12345

# 대상에 붙은 SCP 확인
aws organizations list-policies-for-target \
  --target-id ou-exam-ple12345 --filter SERVICE_CONTROL_POLICY
```

- `create-policy`, `attach-policy`, `list-policies-for-target`의 옵션은 Organizations CLI의 일반 형식에 따라 썼고, 각 레퍼런스 페이지와 직접 대조하지는 않았다(확인 필요).
- `describe-effective-policy`는 SCP·RCP에는 쓸 수 없다. 선언형 정책 전용이다. ([describe-effective-policy](https://docs.aws.amazon.com/cli/latest/reference/organizations/describe-effective-policy.html))

## 한도

출처: [Quotas](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_reference_limits.html)

| 항목 | SCP | RCP |
| --- | --- | --- |
| 정책 하나의 최대 크기 | 10,240자 | 5,120자 |
| 조직 root·OU·계정 하나에 직접 붙이는 개수 | 최대 10개(최소 1개는 붙어 있어야 함) | 최대 5개(RCPFullAWSAccess 포함) |
| 조직당 정책 수 | 10,000 | 2,000 |

- 상속받은 정책은 붙이는 개수 한도에 포함되지 않는다. 이 한도들은 늘릴 수 없다.
- CLI·SDK로 올리면 **공백도 글자 수에 포함**된다. 콘솔은 공백을 제거한다. 긴 정책은 JSON을 압축(minify)해서 올린다.

## 이슈 / 주의

- **FullAWSAccess를 떼는 순간 그 아래가 전부 막힌다.** allow-list로 바꿀 때는 대체 Allow 정책을 먼저 붙이고 나서 뗀다.
- **예외 role 없이 Deny를 걸면 복구도 막힌다.** 관리 role 예외(`aws:PrincipalArn`)를 기본으로 넣는다.
- **management 계정은 SCP·RCP 밖이다.** 가드레일이 필요한 워크로드를 management 계정에 두지 않는다.
- **SCP는 resource-based policy를 막지 못한다.** "외부에서 들어오는 접근"은 RCP로 막는다.

## 출처

- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_evaluation.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_syntax.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps_examples.html
- https://aws.amazon.com/about-aws/whats-new/2025/09/aws-organizations-iam-language-service-control-policies/
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps_syntax.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps_examples.html
- https://github.com/aws-samples/service-control-policy-examples
- https://github.com/aws-samples/resource-control-policy-examples
- https://github.com/aws-samples/data-perimeter-policy-examples
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ExamplePolicies_EC2.html
- https://docs.aws.amazon.com/cli/latest/reference/iam/generate-organizations-access-report.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/troubleshoot_access-denied.html
- https://docs.aws.amazon.com/cli/latest/reference/organizations/describe-effective-policy.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_reference_limits.html

---
*조사 기준일: 2026-10-13. 예시 정책은 AWS 공식 문서·aws-samples 저장소를 바탕으로 했고, 일부는 단순화했다. 실제 조직에서 실행하지는 않았다.*
