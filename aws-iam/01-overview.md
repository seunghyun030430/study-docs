# AWS IAM 개요

> IAM은 AWS에서 **"누가(인증) 무엇을 할 수 있는지(인가)"**를 정하는 무료 글로벌 서비스다. 현재 AWS 권장은 분명하다: **사람은 IdP 페더레이션으로, 워크로드는 IAM role로, 둘 다 임시 자격증명**을 쓴다. IAM user의 장기 액세스 키와 root user 사용은 예외적인 경우로 줄인다.

## 개요 / 정의

> IAM은 "a web service that helps you securely control access to AWS resources"로, "who is authenticated (signed in) and authorized (has permissions)"를 제어한다. — [What is IAM?](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)

| 특징 | 내용 | 출처 |
| --- | --- | --- |
| 요금 | IAM, IAM Identity Center, STS는 추가 요금 없음. IAM Access Analyzer의 일부 기능(unused access, internal access, custom policy checks)은 유료 | [What is IAM?](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) |
| 글로벌 | IAM 리소스는 리전이 없음(ARN의 리전 칸이 비어 있음). IAM API 엔드포인트는 us-east-1 하나 | [Identifiers](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html), [Condition keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html) |
| Eventual consistency | 변경이 전 세계로 복제되므로 즉시 반영되지 않을 수 있음. 고가용성이 필요한 코드 경로에서 IAM을 변경하지 말 것 | [What is IAM?](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) |

## 핵심 개념

### Principal(요청 주체)

정책의 `Principal` 요소에 쓸 수 있는 대상이다. ([Principal element](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html))

| Principal | 설명 | 자격증명 |
| --- | --- | --- |
| AWS 계정 / root user | 계정을 만든 이메일 계정. 모든 권한 | 비밀번호(+MFA). 액세스 키는 만들지 말 것 |
| IAM user | 계정 안의 장기 신원 | 비밀번호, 장기 액세스 키 |
| IAM role | 누군가 맡는(assume) 신원. 자체 장기 자격증명 없음 | STS 임시 자격증명 |
| Role session | role을 assume한 결과(`assumed-role/…`) | 임시 자격증명 |
| Federated user | SAML·OIDC IdP 사용자, `GetFederationToken` 사용자 | 임시 자격증명 |
| AWS 서비스 | `lambda.amazonaws.com` 같은 service principal | — |
| `*` | 모든 principal(익명 포함) | — |

- **IAM group은 principal이 아니다.** user에게 정책을 묶어 주는 용도일 뿐이다.
- AWS는 role session보다 **role principal**을 정책에 쓰라고 권장한다. (같은 출처)

### 인증과 인가

요청 하나는 action, resource, principal, 환경 데이터(IP, 시각, TLS 여부 등), 리소스 데이터로 구성된다. AWS는 이를 **request context**로 모아 정책과 비교한다. ([How IAM works](https://docs.aws.amazon.com/IAM/latest/UserGuide/intro-structure.html)) 정책의 `Condition`은 이 request context의 키를 검사한다. 평가 순서는 [03-policy-evaluation.md](./03-policy-evaluation.md)에서 다룬다.

### ARN과 고유 ID

출처: [IAM identifiers](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html)

| 대상 | ARN 형식 |
| --- | --- |
| root | `arn:aws:iam::111122223333:root` |
| user | `arn:aws:iam::111122223333:user/division_abc/Jane` |
| role | `arn:aws:iam::111122223333:role/S3Access` |
| service-linked role | `arn:aws:iam::111122223333:role/aws-service-role/access-analyzer.amazonaws.com/AWSServiceRoleForAccessAnalyzer` |
| role session | `arn:aws:sts::111122223333:assumed-role/S3Access/kenny` |
| customer managed policy | `arn:aws:iam::111122223333:policy/MyPolicy` |
| federated user | `arn:aws:sts::111122223333:federated-user/Paulo` |

고유 ID 접두사로 무엇인지 구분할 수 있다: `AIDA`(user), `AROA`(role), `ANPA`(managed policy), `AKIA`(장기 액세스 키), `ASIA`(임시 액세스 키).

- 사용자 이름은 로그인할 때는 대소문자를 구분하지 않지만, 정책과 ARN 안에서는 구분한다.

## AWS가 권장하는 구성

[Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)의 현재 항목 순서(요약):

| # | 권장 사항 | 이 레포 문서 |
| --- | --- | --- |
| 1 | 사람은 IdP 페더레이션 + 임시 자격증명으로 접근 | `aws-organizations`(예정, IAM Identity Center) |
| 2 | 워크로드는 IAM role의 임시 자격증명 사용 | [05-workload-identity.md](./05-workload-identity.md) |
| 3 | MFA 필수(피싱 저항형 passkey·보안 키 권장) | — |
| 4 | 장기 자격증명이 꼭 필요하면 액세스 키를 필요할 때 교체 | [05](./05-workload-identity.md#iam-user-액세스-키) |
| 5 | root user 자격증명 보호 | 아래 |
| 6–9 | 최소 권한, AWS managed policy에서 시작해 좁히기, Access Analyzer로 정책 생성, 미사용 신원·권한 정리 | [06-least-privilege.md](./06-least-privilege.md) |
| 10 | 조건(Condition)으로 추가 제한 | [02-policies.md](./02-policies.md) |
| 11–12 | Access Analyzer로 공개·교차 계정 접근 검증, 정책 검증 | [06](./06-least-privilege.md) |
| 13 | 여러 계정에 SCP·RCP 가드레일 | [03](./03-policy-evaluation.md#scprcp의-적용-범위) |
| 14 | permissions boundary로 권한 관리 위임 | [02](./02-policies.md) |

### 사람 vs 워크로드

| 구분 | 권장 방식 | 피할 방식 |
| --- | --- | --- |
| 사람(콘솔·CLI) | IAM Identity Center 또는 외부 IdP(SAML/OIDC) 페더레이션 → role 임시 자격증명 | 사람마다 IAM user + 장기 액세스 키 |
| AWS 안 워크로드 | EC2 instance profile, Lambda execution role, ECS task role, EKS Pod Identity | 코드·환경변수에 액세스 키 |
| AWS 밖 워크로드 | IAM Roles Anywhere(X.509), OIDC(`AssumeRoleWithWebIdentity`, 예: GitHub Actions), SAML | CI 시크릿에 장기 액세스 키 |

### Root user

[Root user best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html) 요약:

- 강한 비밀번호와 **MFA**(최대 8개 디바이스 등록 가능)를 쓴다.
- **root 액세스 키를 만들지 않는다.**
- 그룹 메일 주소를 쓰고, 계정 복구 수단에 대한 접근을 제한한다.
- Organizations를 쓰면 멤버 계정의 root 자격증명을 중앙에서 제거하고, 필요할 때만 `sts:AssumeRoot`로 root 작업을 한다.
- root 로그인과 사용을 모니터링한다.

root user에는 identity-based policy나 permissions boundary를 붙일 수 없다. 하지만 멤버 계정의 root는 SCP·RCP의 영향을 받는다. ([Policies and permissions](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html))

## 주요 한도

출처: [IAM and STS quotas](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html)

| 항목 | 기본값 | 최대(증설 시) |
| --- | --- | --- |
| 계정당 role | 1,000 | 10,000 |
| 계정당 customer managed policy | 1,500 | 10,000 |
| role당 managed policy 연결 | 20 | 25 |
| user당 managed policy 연결 | 10 | 20 |
| role trust policy 길이 | 2,048자 | 8,192자 |
| 계정당 OIDC provider | 100 | 700 |
| managed policy 크기 | 6,144자(고정) | — |
| role inline policy 합계 | 10,240자(고정, 공백 제외) | — |
| role 세션 길이 | 1–12시간(기본 1시간) | — |
| STS 요청 | 계정·리전당 초당 600 | — |

정책 크기 제한 때문에 "정책을 하나로 몰아넣기"는 금방 한계에 부딪힌다. 정책을 역할별로 나누고, 와일드카드보다는 조건과 태그(ABAC)로 범위를 표현한다.

## 출처

- https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/intro-structure.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html

---
*조사 기준일: 2026-10-13. 한도 값은 증설 가능 여부와 함께 자주 바뀌므로 Service Quotas 콘솔에서 다시 확인할 것.*
