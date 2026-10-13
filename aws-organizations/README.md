# AWS Organizations 정리

AWS Organizations로 여러 AWS 계정을 묶어 관리하는 방법을 실무 관점에서 정리한 문서 모음. 중점은 **조직 구조·OU 설계 / SCP·RCP 설계 / IAM Identity Center / 선언형 정책·root 중앙 관리·Control Tower**. 조사 기준일 **2026-10-13**.

선행 주제: [aws-iam](../aws-iam/README.md). principal, role, 정책 평가 로직 같은 IAM 기본 개념은 그쪽에서 다룬다.

## 한눈에 보기

| 질문 | 짧은 답 | 문서 |
| --- | --- | --- |
| 왜 계정을 나누나? | 보안 경계, 피해 반경(blast radius), 비용 분리, 계정별 한도 분산 | [01](./01-overview.md) |
| 조직의 뼈대는? | 조직 root → OU(폴더, 최대 5단계) → 계정. 정책은 아래로 상속 | [01](./01-overview.md) |
| OU는 어떻게 나누나? | 조직도가 아니라 **같은 정책을 받을 묶음** 기준. Security / Infrastructure / Workloads(Prod·SDLC) / Sandbox + Policy Staging·Suspended | [02](./02-structure-ou.md) |
| management 계정은? | 조직 관리·결제만. **SCP·RCP가 적용되지 않으므로** 워크로드 금지, 관리는 위임 관리자에게 | [02](./02-structure-ou.md) |
| SCP vs RCP? | SCP = 우리 user·role의 상한선, RCP = 우리 리소스의 상한선(조직 밖 접근 차단). 둘 다 권한을 주지 않음 | [03](./03-scp-rcp.md) |
| SCP 운영 기본형? | FullAWSAccess 유지 + Deny 추가(deny-list), 관리 role 예외, 테스트 OU부터 점진 적용 | [03](./03-scp-rcp.md) |
| 사람의 로그인은? | IAM Identity Center: 외부 IdP + SCIM, permission set을 그룹·계정에 할당 | [04](./04-identity-center.md) |
| 설정 표준(태그, IMDSv2 기본값 등)은? | 선언형 정책. **management 계정에도 적용** | [05](./05-other-policies.md) |
| member 계정 root는? | root 중앙 관리로 자격증명 제거, 필요 시 `sts:AssumeRoot`(15분) | [05](./05-other-policies.md) |
| 처음부터 다 만들어야 하나? | Control Tower가 landing zone·control을 자동 구성. 단 직접 수정하면 drift | [05](./05-other-policies.md) |

| 정책 | 대상 | management 계정 | 권한 부여 |
| --- | --- | --- | --- |
| SCP | member 계정의 IAM user·role(root 포함) | 적용 안 됨 | 안 함(상한선) |
| RCP | member 계정의 리소스(조직 밖 접근 포함) | 적용 안 됨 | 안 함(Deny만) |
| 선언형 정책 | 서비스 설정 상태 | **적용됨** | 해당 없음 |

## 문서 인덱스

| 문서 | 다루는 것 |
| --- | --- |
| [01-overview.md](./01-overview.md) | 개요(단계별 입문) — 계정 분리 이유, 조직·management·member·조직 root·OU, 기능 세트·통합 결제, 정책 지도, trusted access·위임 관리자, Identity Center·Control Tower 소개, 용어 정리 |
| [02-structure-ou.md](./02-structure-ou.md) | AWS 권장 OU 구조와 설계 원칙, management 계정 최소화, 계정 생성·초대·이동·제거·해지, 위임 관리자와 정책 관리 위임 |
| [03-scp-rcp.md](./03-scp-rcp.md) | SCP 상속·deny-list vs allow-list·문법, RCP 규칙, 가드레일 예시 7종(리전 제한, 이탈 금지, 로그 보호, root 금지, IMDSv2, HTTPS, identity perimeter), 테스트·롤아웃, 거부 메시지, 한도 |
| [04-identity-center.md](./04-identity-center.md) | instance 종류, identity source·SCIM, permission set·할당·`AWSReservedSSO_` role, 위임 관리자, ABAC, MFA·다중 리전, CLI SSO 로그인과 sso-admin |
| [05-other-policies.md](./05-other-policies.md) | 선언형 정책(종류, EC2 정책, 상속 연산자, tag policy), root 자격증명 중앙 관리와 `sts:AssumeRoot`, Control Tower 구성과 drift |
| [06-issues.md](./06-issues.md) | 주요 한도, 흔한 장애·함정 표, 안티패턴 |

## 출처 (1차)

- Organizations User Guide: https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html
- 멀티 계정 백서: https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/organizing-your-aws-environment.html
- SCP: https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html
- RCP: https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_rcps.html
- IAM Identity Center User Guide: https://docs.aws.amazon.com/singlesignon/latest/userguide/what-is.html
- Control Tower User Guide: https://docs.aws.amazon.com/controltower/latest/userguide/what-is-control-tower.html
- 한도: https://docs.aws.amazon.com/organizations/latest/userguide/orgs_reference_limits.html
- 예시 저장소: https://github.com/aws-samples/service-control-policy-examples, https://github.com/aws-samples/data-perimeter-policy-examples

---
*이 문서는 학습·정리 목적의 요약이다. SCP·RCP는 잘못 붙이면 조직 전체가 멈출 수 있으므로, 예시를 적용하기 전에 반드시 테스트 OU에서 확인할 것.*
