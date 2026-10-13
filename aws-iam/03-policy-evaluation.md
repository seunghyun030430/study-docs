# 정책 평가 로직

> 기본값은 **암묵적 거부(implicit deny)**이고, 어디서든 **명시적 Deny가 하나라도 있으면 끝**이다. 그다음 RCP → SCP → resource-based → identity-based → permissions boundary → session policy 순으로 확인한다. 같은 계정에서는 identity-based와 resource-based가 **합집합**이고, boundary·SCP·RCP·session policy는 **교집합(상한선)**으로 작동한다. 교차 계정은 **양쪽 계정이 모두 Allow**해야 한다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)와 [02-policies.md](./02-policies.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Management 계정 / member 계정 | AWS Organizations에서 조직을 만든 관리 계정 / 조직에 속한 나머지 계정 |
| OU(Organizational Unit) | Organizations 안에서 계정을 묶는 폴더 |
| Trust policy | role에 붙어 "누가 이 role을 맡을 수 있는지"를 정하는 resource-based policy |
| Service-linked role | AWS 서비스가 자기 일을 하려고 내 계정에 만들어 두는 전용 role. 사용자가 권한을 마음대로 바꿀 수 없음 |
| Session principal | role을 맡은 세션처럼 임시 자격증명으로 요청하는 principal |
| Trusting / trusted 계정 | 교차 계정에서 리소스를 가진 쪽 / 접근하려는 쪽 |
| 합집합 / 교집합 | 둘 중 하나만 허용해도 됨 / 둘 다 허용해야 함 |

## 개요

AWS가 요청을 처리하는 단계는 세 가지다. ([Policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html))

1. **인증(Authentication)**: 요청을 보낸 principal을 확인한다.
2. **요청 컨텍스트 구성**: action, resource, principal, 환경 값(IP, 시간, 태그 등)을 모은다.
3. **정책 평가**: 적용되는 모든 정책을 보고 Allow / Deny를 결정한다.

> "By default, all requests are implicitly denied with the exception of the AWS account root user, which has full access." — [Explicit and implicit denies](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic_policy-eval-denyallow.html)

## 단일 계정 평가 순서

출처: [Policy evaluation logic](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html)

```
요청
 │
 ▼
① 모든 정책에 명시적 Deny가 있나? ──── 예 ──▶ DENY
 │ 아니오
 ▼
② RCP(Resource Control Policy)가 Allow하나? ── 아니오 ──▶ DENY
 │ 예 (RCP 미사용이면 기본 RCPFullAWSAccess가 항상 Allow)
 ▼
③ SCP(Service Control Policy)가 Allow하나? ── 아니오 ──▶ DENY
 │ 예
 ▼
④ resource-based policy가 Allow하나? ─── 예 ──▶ ALLOW (principal 종류에 따라, 아래 참고)
 │ 아니오
 ▼
⑤ identity-based policy가 Allow하나? ─── 아니오 ──▶ DENY (암묵적)
 │ 예
 ▼
⑥ permissions boundary가 있다면 Allow하나? ── 아니오 ──▶ DENY (암묵적)
 │ 예 / 없음
 ▼
⑦ session principal이고 session policy가 있다면 Allow하나? ── 아니오 ──▶ DENY
 │ 예 / 해당 없음
 ▼
ALLOW
```

- ①의 Deny 검사는 SCP, RCP, resource-based, identity-based, boundary, session policy **전부**를 대상으로 한다.
- RCP를 켜면 `RCPFullAWSAccess`가 자동으로 붙고 떼어 낼 수 없다. 그래서 ②는 별도 Deny RCP를 만들지 않는 한 항상 통과한다.

### 정책 조합 규칙

| 조합 | 결과 |
| --- | --- |
| identity-based + resource-based (같은 계정) | **합집합**: 둘 중 하나만 Allow해도 됨 |
| identity-based + permissions boundary | **교집합**: 둘 다 Allow해야 함 |
| identity-based + SCP + RCP | **교집합**: 셋 다 Allow해야 함 |
| role 정책 + session policy | **교집합**: session policy는 권한을 더하지 못하고 줄이기만 함 ([AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html)) |
| 어떤 정책이든 명시적 Deny | **무조건 Deny** |

## Resource-based policy의 미묘한 점

같은 계정에서는 "대부분의 리소스는 identity-based **또는** resource-based 둘 중 한쪽의 Allow면 충분"하다. 단, resource-based policy의 Principal에 **무엇을 적었는지**에 따라 boundary·session policy의 영향이 달라진다. ([Explicit and implicit denies](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic_policy-eval-denyallow.html))

| resource-based policy의 Principal | identity 정책의 암묵적 거부 | boundary의 암묵적 거부 | session policy의 암묵적 거부 |
| --- | --- | --- | --- |
| IAM **user** ARN | 영향 없음 | 영향 없음 | — |
| IAM **role** ARN (또는 `aws:PrincipalArn`) | 영향 없음 | **제한됨** | **제한됨** |
| **role session** ARN (`arn:aws:sts::…:assumed-role/<role>/<session>`) | 영향 없음 | 영향 없음 | 영향 없음 |
| federated user ARN (`GetFederationToken`) | 영향 없음 | 영향 없음 | 영향 없음 |

- 어느 경우든 **명시적 Deny**는 항상 적용된다.
- **예외**: IAM role trust policy와 **KMS key policy**는 같은 계정에서도 resource 쪽이 반드시 명시적으로 Allow해야 한다. identity 정책만으로는 접근할 수 없다. 다른 서비스 중에도 이런 리소스가 있을 수 있다. (같은 출처)
- AWS 문서의 예: Secrets Manager resource policy가 user를 허용하면 boundary에 Allow가 없어도 접근할 수 있다. "implicit denies in permissions boundaries do not limit resource-based policies." ([Permissions boundaries](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html))

## 교차 계정 평가

> "AWS evaluates the request in the trusting account and the trusted account… The request is allowed only if both evaluations return a decision of Allow." — [Cross-account policy evaluation](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic-cross-account.html)

| 계정 | 필요한 것 |
| --- | --- |
| 요청자 계정(trusted, `111122223333`) | 요청자의 identity-based policy가 해당 action·resource를 Allow (+ 자기 계정의 SCP·boundary 통과) |
| 리소스 계정(trusting, `444455556666`) | resource-based policy가 요청자를 Allow (+ 리소스 계정의 RCP 통과) |

같은 계정과 달리 **합집합이 아니다**. 한쪽만 허용하면 거부된다. resource-based policy가 없는 리소스(예: 대부분의 EC2 API)는 교차 계정으로 직접 접근할 수 없으므로, 리소스 계정의 **role을 AssumeRole**해서 접근한다(→ [04-roles-sts.md](./04-roles-sts.md)).

## SCP·RCP의 적용 범위

이 문서에서는 평가 로직에 필요한 내용만 다룬다. SCP·RCP 설계와 예시는 [aws-organizations/03-scp-rcp.md](../aws-organizations/03-scp-rcp.md)에서 다룬다.

| 항목 | SCP | RCP |
| --- | --- | --- |
| 무엇을 제한하나 | 멤버 계정의 **principal**이 할 수 있는 최대 권한 | 멤버 계정의 **리소스**에 허용될 수 있는 최대 권한 |
| 권한 부여 | 하지 않음(상한선만) | 하지 않음(상한선만) |
| management 계정 | **적용 안 됨** | **적용 안 됨** |
| service-linked role | 적용 안 됨 | 적용 안 됨(SLR 호출·SLR trust policy) |
| 멤버 계정 root user | 적용됨 | — |
| 조직 밖 principal | 영향 없음 | **영향 있음**(조직 밖에서 들어오는 접근도 제한) |
| 지원 서비스 | 전체 | S3, STS, KMS, SQS, Secrets Manager, DynamoDB, ECR 등 지원 목록에 한정 |

출처: [SCPs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html), [RCPs](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html)

## 사례별 판정 예시

| # | 상황 | 결과 | 이유 |
| --- | --- | --- | --- |
| 1 | user 정책에 `s3:GetObject` Allow, 버킷 정책 없음 | Allow | identity-based Allow |
| 2 | user 정책은 비어 있고, 같은 계정 버킷 정책이 그 **user ARN**을 Allow | Allow | 같은 계정 합집합 |
| 3 | 2와 같지만 user에 boundary가 있고 boundary에 S3가 없음 | Allow | user ARN이 Principal이면 boundary의 암묵적 거부에 제한받지 않음 |
| 4 | 버킷 정책이 **role ARN**을 Allow, role에 boundary(S3 없음) | **Deny** | role ARN이 Principal이면 boundary 암묵적 거부가 적용됨 |
| 5 | role 정책에 `kms:Decrypt` Allow, KMS key policy는 계정 루트만 허용하고 IAM 위임 문구 없음 | **Deny** | KMS key policy는 명시적 Allow가 필요한 예외 |
| 6 | 다른 계정 user 정책이 `s3:GetObject` Allow, 버킷 정책에는 그 계정 언급 없음 | **Deny** | 교차 계정은 양쪽 Allow 필요 |
| 7 | 모든 정책이 Allow, SCP에 `Deny ec2:*` (Region 조건) | **Deny** | 명시적 Deny 우선 |
| 8 | AssumeRole 시 session policy로 `s3:GetObject`만 넘김, role 정책은 `s3:*` | `GetObject`만 Allow | session policy와 교집합 |

예시 3·4·5는 AWS 문서의 규칙을 적용해 만든 사례다. 실제 동작은 아래 시뮬레이터나 테스트 계정에서 확인하는 것이 좋다.

## 사용법 / 예시: 평가 결과 확인

```bash
# 실제 요청 없이 정책 평가 결과 시뮬레이션
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::111122223333:role/AppRole \
  --action-names s3:GetObject s3:PutObject \
  --resource-arns arn:aws:s3:::my-bucket/data.txt \
  --query 'EvaluationResults[].{action:EvalActionName,decision:EvalDecision}'
```

```bash
# 지금 내가 누구로 요청하는지 확인(권한 없이도 항상 동작)
aws sts get-caller-identity
```

- `EvalDecision`은 `allowed`, `explicitDeny`, `implicitDeny` 중 하나다. SCP의 영향은 `OrganizationsDecisionDetail`에 따로 표시된다.
- `--resource-policy`로 resource-based policy를 함께 넣을 수 있지만, 이 옵션은 **IAM user에만** 적용된다. ([simulate-principal-policy](https://docs.aws.amazon.com/cli/latest/reference/iam/simulate-principal-policy.html))
- 따라서 role의 resource-based policy 효과나 RCP까지 반영한 결과는 시뮬레이터만으로 확신하기 어렵다. 실제 호출로 한 번 더 확인한다.

## 이슈 / 주의

- **"Allow를 줬는데 AccessDenied"**의 주범은 SCP/RCP, permissions boundary, session policy, KMS key policy, VPC endpoint policy다. 최근 AWS 오류 메시지는 어떤 정책 유형이 거부했는지 알려 주는 경우가 많으니 메시지를 끝까지 읽는다.
- **명시적 Deny로 길을 막으면 우회할 수 없다.** 넓은 Deny는 관리자까지 막을 수 있으니 조건(`aws:PrincipalArn` 예외 등)을 같이 설계한다.
- **resource-based policy의 Principal을 user ARN으로 주면 boundary를 우회**한다. boundary로 권한을 위임할 때는 resource-based policy 작성 권한도 함께 통제해야 한다.
- **management 계정에는 SCP가 적용되지 않는다.** 워크로드를 management 계정에 두지 않는다.

## 출처

- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic_policy-eval-denyallow.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic-cross-account.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_boundaries.html
- https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html
- https://docs.aws.amazon.com/cli/latest/reference/iam/simulate-principal-policy.html

---
*조사 기준일: 2026-10-13. 흐름도와 사례 표는 공식 문서의 규칙을 바탕으로 재구성했다.*
