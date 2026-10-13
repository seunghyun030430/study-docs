# 권한 점검과 최소 권한

> 최소 권한은 한 번에 설계하지 않고 **넓게 시작 → 사용 기록으로 좁히기 → 배포 전 자동 검증**의 반복으로 만든다. 도구는 IAM Access Analyzer(외부·내부·미사용 접근, 정책 검증, custom policy checks, 정책 생성)와 마지막 사용 정보(last accessed)다. 가장 위험한 안티패턴은 `iam:PassRole`·정책 수정 권한을 통한 **권한 상승**과 조건 없는 와일드카드 principal이다.

## 최소 권한 절차

AWS 모범 사례(6–12번 항목)를 작업 순서로 정리하면 다음과 같다. ([Best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html))

| 단계 | 할 일 | 도구 |
| --- | --- | --- |
| 1. 시작 | AWS managed policy나 넓은 customer managed policy로 동작부터 확인 | — |
| 2. 관찰 | 실제로 쓰는 서비스·액션 기록 수집 | CloudTrail, last accessed |
| 3. 좁히기 | 사용 기록으로 정책 생성, Resource·Condition 추가 | Access Analyzer **policy generation** |
| 4. 검증 | 문법·보안 경고 확인, "새 권한이 늘었는지" 자동 검사 | **validate-policy**, **custom policy checks**(CI에 연결) |
| 5. 감시 | 외부 공개·교차 계정 접근, 미사용 role·키·권한 탐지 | **external / unused access analyzer** |
| 6. 정리 | 미사용 user·role·키·권한 제거 | last accessed, unused access findings |

## IAM Access Analyzer

출처: [What is IAM Access Analyzer?](https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html), [Pricing](https://aws.amazon.com/iam/access-analyzer/pricing/)

| 기능 | 무엇을 찾나 | 범위 | 요금 |
| --- | --- | --- | --- |
| External access | 신뢰 영역(계정·조직) 밖에 공유된 리소스(S3, KMS, role trust 등) | **리전별** analyzer. 정책 변경 후 약 30분 안에 분석 | 무료 |
| Internal access | 조직·계정 **안**에서 중요 리소스(S3, RDS 스냅샷, DynamoDB)에 닿을 수 있는 principal | 지정한 리소스 | 리소스·리전당 월 $9.00 |
| Unused access | 미사용 role, 액세스 키, 비밀번호, 미사용 서비스·액션 권한 | 리전과 무관 | role·user당 월 $0.20(service-linked role 제외, analyzer마다 과금) |
| Policy validation | 100개 이상의 문법·모범 사례 검사(`ERROR`, `SECURITY_WARNING`, `WARNING`, `SUGGESTION`) | 정책 문서 | 무료 |
| Custom policy checks | `CheckNoNewAccess`, `CheckAccessNotGranted`, `CheckNoPublicAccess` | 정책 문서 | API 호출당 $0.0020 |
| Policy generation | CloudTrail 기록으로 정책 초안 생성 | 지정 기간 | 무료 |

### 정책 검증과 custom policy checks

```bash
# 문법·보안 경고
aws accessanalyzer validate-policy \
  --policy-type IDENTITY_POLICY \
  --policy-document file://policy.json \
  --query 'findings[].{type:findingType,issue:issueCode}'

# PR에서 "기존보다 권한이 늘었나?" 검사 → PASS / FAIL
aws accessanalyzer check-no-new-access \
  --policy-type IDENTITY_POLICY \
  --existing-policy-document file://policy.main.json \
  --new-policy-document file://policy.pr.json

# "이 정책이 iam:PassRole·s3:DeleteBucket을 허용하나?"
aws accessanalyzer check-access-not-granted \
  --policy-type IDENTITY_POLICY \
  --policy-document file://policy.pr.json \
  --access '[{"actions":["iam:PassRole","s3:DeleteBucket"]}]'

# 버킷 정책이 공개 접근을 허용하나?
aws accessanalyzer check-no-public-access \
  --resource-type AWS::S3::Bucket \
  --policy-document file://bucket-policy.json
```

출처: [validate-policy](https://docs.aws.amazon.com/cli/latest/reference/accessanalyzer/validate-policy.html), [check-no-new-access](https://docs.aws.amazon.com/cli/latest/reference/accessanalyzer/check-no-new-access.html), [Custom policy checks](https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-custom-policy-checks.html). `check-access-not-granted`와 `check-no-public-access`의 옵션 형식은 CLI 레퍼런스와 직접 대조하지 않았다(확인 필요).

`validate-policy`의 `--policy-type`에는 `SERVICE_CONTROL_POLICY`, `RESOURCE_CONTROL_POLICY`도 쓸 수 있다.

## 마지막 사용 정보(last accessed)

출처: [Refine permissions using last accessed information](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_last-accessed.html)

| 항목 | 내용 |
| --- | --- |
| 서비스 단위 추적 기간 | 최소 400일. 최근 활동은 4시간 안에 반영 |
| 액션 단위 추적 | **관리(control plane) 액션만**. S3는 2020-04-12, EC2·IAM·Lambda는 2021-04-07, 그 외 서비스는 2023-05-23부터 |
| 미추적 | `iam:PassRole` |
| 집계 기준 | 거부된 시도도 "접근"으로 기록됨 |

```bash
JOB=$(aws iam generate-service-last-accessed-details \
  --arn arn:aws:iam::111122223333:role/AppRole --query JobId --output text)
aws iam get-service-last-accessed-details --job-id "$JOB" \
  --query 'ServicesLastAccessed[?TotalAuthenticatedEntities==`0`].ServiceNamespace'
```

위 명령의 `--query` 필터는 "한 번도 쓰지 않은 서비스"를 뽑는 예시다. 정확한 응답 필드 이름은 CLI 레퍼런스로 다시 확인할 것(확인 필요).

- **데이터 플레인 액션(예: `s3:GetObject`)은 액션 단위로 추적되지 않는다.** 서비스 단위로는 보이지만, 액션까지 좁히려면 CloudTrail 데이터 이벤트나 Access Analyzer 정책 생성이 필요하다.

## 안티패턴

| 안티패턴 | 왜 위험한가 | 대안 | 근거 |
| --- | --- | --- | --- |
| `iam:PassRole` on `"*"` | 자기보다 강한 role을 EC2·Lambda 등에 넘겨 그 권한으로 실행 → 권한 상승 | 넘길 role ARN 명시 + `iam:PassedToService` 조건 | Access Analyzer `PASS_ROLE_WITH_STAR_IN_RESOURCE` ([Policy checks](https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-reference-policy-checks.html), [PassRole](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_passrole.html)) |
| `iam:CreateRole` + `iam:AttachRolePolicy`/`PutRolePolicy` 무제한 | 관리자 정책을 붙인 role을 만들어 스스로 권한 상승 | permissions boundary 강제 조건(`iam:PermissionsBoundary`) | [Create role for service](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html) |
| `Allow` + `NotAction` | "이것 빼고 전부" → 새 서비스까지 자동 허용 | 허용할 액션 열거. `NotAction`은 `Deny` 가드레일에만 | [NotAction](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_notaction.html) |
| `Allow` + `NotPrincipal` | 의도치 않은 principal 허용 | `aws:PrincipalArn` 조건 | `ALLOW_WITH_NOT_PRINCIPAL` |
| OIDC principal 조건 없음 | 다른 조직의 GitHub 워크플로 등이 assume | `sub`·`aud` 조건 | `MISSING_CONDITION_KEY_FOR_OIDC_PRINCIPAL`, `MISSING_GITHUB_REPO_CONDITION_KEY` |
| `Action: "*"`, `Resource: "*"` | 최소 권한 위반, 사고 반경 최대 | 액션·리소스 명시, 조건 추가 | 최소 권한 원칙(특정 검사 이름은 확인 필요) |
| IAM user 장기 키, root 액세스 키 | 유출 시 만료 없이 사용 가능 | role·페더레이션·Roles Anywhere | [Best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) |
| 같은 권한을 여러 inline 정책으로 복사 | 감사·수정 누락 | customer managed policy 공유 | — |

### 권한 위임할 때 boundary로 상승 막기

개발자에게 "role은 직접 만들되, 반드시 정해진 boundary를 붙여야 한다"는 규칙을 거는 패턴이다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CreateRolesOnlyWithBoundary",
      "Effect": "Allow",
      "Action": ["iam:CreateRole", "iam:AttachRolePolicy", "iam:PutRolePolicy"],
      "Resource": "arn:aws:iam::111122223333:role/app/*",
      "Condition": {
        "StringEquals": {
          "iam:PermissionsBoundary": "arn:aws:iam::111122223333:policy/DevBoundary"
        }
      }
    },
    {
      "Sid": "DenyBoundaryTampering",
      "Effect": "Deny",
      "Action": [
        "iam:CreatePolicyVersion",
        "iam:DeletePolicy",
        "iam:DeletePolicyVersion",
        "iam:SetDefaultPolicyVersion",
        "iam:DeleteRolePermissionsBoundary"
      ],
      "Resource": [
        "arn:aws:iam::111122223333:policy/DevBoundary",
        "arn:aws:iam::111122223333:role/app/*"
      ]
    },
    {
      "Sid": "PassOnlyAppRoles",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::111122223333:role/app/*",
      "Condition": { "StringEquals": { "iam:PassedToService": "lambda.amazonaws.com" } }
    }
  ]
}
```

- boundary 정책 자체를 고치거나 떼어 내지 못하게 막아야 위임이 의미가 있다. AWS의 [permissions boundary 문서](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html) 예시도 같은 취지로 Deny를 건다. 이번 조사에서는 이 예시를 검색 요약으로만 확인했다(확인 필요).
- 위 정책은 공식 예시를 바탕으로 이 레포에서 재구성한 것이다. 실제 적용 전에 `validate-policy`와 테스트 계정에서 확인한다.

## 흔한 함정

- **unused access analyzer는 리전별 analyzer마다 과금된다.** 결과는 리전과 무관하므로 한 리전에만 만든다.
- **last accessed는 거부된 시도도 "사용"으로 센다.** 실제로는 실패한 권한이 남아 있을 수 있다.
- **정책 생성 결과를 그대로 쓰지 않는다.** CloudTrail에 남은 기간 동안의 활동만 반영하므로, 분기·연 단위 배치 작업의 권한이 빠질 수 있다.
- **"Allow를 줬는데 거부"**는 [03-policy-evaluation.md](./03-policy-evaluation.md)의 순서대로 SCP·RCP → boundary → session policy → resource policy(KMS key policy 등)를 확인한다.

## 출처

- https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html
- https://aws.amazon.com/iam/access-analyzer/pricing/
- https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-custom-policy-checks.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/access-analyzer-reference-policy-checks.html
- https://docs.aws.amazon.com/cli/latest/reference/accessanalyzer/validate-policy.html
- https://docs.aws.amazon.com/cli/latest/reference/accessanalyzer/check-no-new-access.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_last-accessed.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_passrole.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-service.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_notaction.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html

---
*조사 기준일: 2026-10-13. Access Analyzer 요금은 기준일 시점 값이다. 권한 상승 경로를 한곳에 정리한 AWS 공식 페이지는 찾지 못했다.*
