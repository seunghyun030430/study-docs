# AWS IAM 정리

AWS IAM을 실무 관점에서 정리한 문서 모음. 중점은 **정책 평가 로직 / Role·STS·교차 계정 / 워크로드 자격증명 / 권한 점검·최소 권한**. 조사 기준일 **2026-10-13**.

## 한눈에 보기

| 질문 | 짧은 답 | 문서 |
| --- | --- | --- |
| 기본값은? | 암묵적 거부. 명시적 Deny가 하나라도 있으면 끝 | [03](./03-policy-evaluation.md) |
| 권한을 "주는" 정책은? | identity-based, resource-based(+ACL)뿐. boundary·SCP·RCP·session policy는 상한선만 | [02](./02-policies.md) |
| 같은 계정에서 identity와 resource policy는? | **합집합**(한쪽만 Allow해도 됨). 단 role trust policy·KMS key policy는 resource 쪽 Allow 필수 | [03](./03-policy-evaluation.md) |
| 교차 계정은? | **양쪽 계정 모두** Allow 필요 → 보통 리소스 계정의 role을 AssumeRole | [03](./03-policy-evaluation.md), [04](./04-roles-sts.md) |
| 세션 길이? | 기본 1시간, role 설정으로 최대 12시간, **role chaining은 1시간** | [04](./04-roles-sts.md) |
| 서드파티에 role 열어 줄 때? | trust policy에 `sts:ExternalId` 조건(confused deputy 방지) | [04](./04-roles-sts.md) |
| 사람의 접근은? | IdP 페더레이션 + 임시 자격증명(IAM Identity Center) | [01](./01-overview.md) |
| 워크로드의 접근은? | role 임시 자격증명. EKS는 Pod Identity, CI는 OIDC, AWS 밖은 Roles Anywhere | [05](./05-workload-identity.md) |
| 최소 권한은 어떻게? | 넓게 시작 → last accessed·정책 생성으로 좁히기 → custom policy checks로 CI 검증 | [06](./06-least-privilege.md) |

| 워크로드 | 권장 방식 | trust principal |
| --- | --- | --- |
| EC2 | instance profile + IMDSv2 | `ec2.amazonaws.com` |
| Lambda / ECS | execution role / task role | `lambda.amazonaws.com` / `ecs-tasks.amazonaws.com` |
| EKS | Pod Identity(Linux EC2 노드) | `pods.eks.amazonaws.com` |
| GitHub Actions | OIDC(`sub`로 repo·브랜치 제한) | `token.actions.githubusercontent.com` |
| 온프레미스 | IAM Roles Anywhere(X.509) | `rolesanywhere.amazonaws.com` |

## 문서 인덱스

| 문서 | 다루는 것 |
| --- | --- |
| [01-overview.md](./01-overview.md) | 개요 — principal, 인증·인가, ARN·고유 ID, AWS 권장 구성(사람 vs 워크로드), root user, 한도 |
| [02-policies.md](./02-policies.md) | 정책 종류 9가지 비교, 정책 문법, Not* 요소, Condition 연산자·결합 규칙·ForAllValues 함정, 글로벌 조건 키, ABAC |
| [03-policy-evaluation.md](./03-policy-evaluation.md) | 평가 순서 흐름도, 합집합·교집합 규칙, resource-based의 미묘한 점, 교차 계정, SCP·RCP 적용 범위, 사례별 판정 |
| [04-roles-sts.md](./04-roles-sts.md) | role 구조, STS API 비교, 세션 길이·role chaining, 교차 계정 설정, external ID·confused deputy, session tags·source identity, STS 엔드포인트, 세션 무효화 |
| [05-workload-identity.md](./05-workload-identity.md) | EC2 instance profile·IMDSv2, Lambda, ECS task/execution role, EKS Pod Identity vs IRSA, GitHub Actions OIDC, Roles Anywhere, 액세스 키 교체 |
| [06-least-privilege.md](./06-least-privilege.md) | 최소 권한 절차, IAM Access Analyzer 기능·요금, custom policy checks, last accessed, 권한 상승 안티패턴, boundary 위임 패턴 |

다음 주제: [aws-organizations](../aws-organizations/README.md) — OU 설계, SCP·RCP, IAM Identity Center.

## 출처 (1차)

- IAM User Guide: https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html
- 정책 평가 로직: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_evaluation-logic.html
- 보안 모범 사례: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- STS API Reference: https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html
- 글로벌 조건 키: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html
- Service Authorization Reference: https://docs.aws.amazon.com/service-authorization/latest/reference/reference.html
- IAM Access Analyzer: https://docs.aws.amazon.com/IAM/latest/UserGuide/what-is-access-analyzer.html
- IAM·STS 한도: https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html

---
*이 문서는 학습·정리 목적의 요약이다. 정책 예시는 실제 계정에서 실행하지 않았으므로, 적용 전에 `aws accessanalyzer validate-policy`와 테스트 계정에서 확인할 것.*
