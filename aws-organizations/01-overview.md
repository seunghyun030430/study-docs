# AWS Organizations 개요

> AWS Organizations는 **여러 AWS 계정을 하나의 조직으로 묶어** 계정 생성, 계정 분류(OU), 정책 적용, 결제 통합을 한곳에서 하는 무료 서비스다. 핵심은 세 가지다. **계정을 용도별로 나누고**, **OU라는 폴더로 묶고**, **폴더 단위로 정책(SCP·RCP 등)을 걸어** 모든 계정에 일관된 규칙을 적용한다.

이 문서는 Organizations를 처음 보는 사람을 기준으로 용어를 한 단계씩 쌓아 간다. 처음 나오는 용어는 **굵게** 표시하고 바로 뜻을 적는다. IAM 기본 용어(role, 정책, principal 등)는 [aws-iam/01-overview.md](../aws-iam/01-overview.md)를 참고한다. 모든 용어는 맨 아래 [용어 정리](#용어-정리)에 모아 두었다.

## 1단계: 왜 계정을 여러 개로 나누나

[aws-iam 2단계](../aws-iam/01-overview.md#2단계-계정과-root-user)에서 본 것처럼 **AWS 계정**은 리소스·청구·IAM 설정이 묶이는 단위다. 계정 하나에 모든 것을 넣을 수도 있지만, AWS는 **여러 계정으로 나누라**고 권장한다. 이유는 AWS 멀티 계정 백서에 정리되어 있다. ([Benefits of using multiple AWS accounts](https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/benefits-of-using-multiple-aws-accounts.html))

| 이유 | 쉽게 말하면 |
| --- | --- |
| 업무·소유자별로 묶기 | 팀·서비스마다 자기 계정을 가짐 |
| 환경별 보안 통제 | 운영(prod)과 개발(dev)에 다른 규칙 적용 |
| 민감 데이터 접근 제한 | 사고가 나도 피해가 그 계정 안에 머묾(**blast radius**, 피해 반경 축소) |
| 실험과 혁신 | 마음대로 써 볼 수 있는 sandbox 계정 |
| 사고 영향 범위 제한 | 한 계정의 실수·장애가 다른 계정으로 번지지 않음 |
| 비용 관리 | 계정이 비용을 나누는 기본 단위 |
| 한도 분산 | 서비스 한도(quota)와 API 호출 한도가 **계정마다** 따로 있음 |

## 2단계: 계정이 많아지면 생기는 문제

계정이 10개, 100개가 되면 이런 문제가 생긴다.

- 계정마다 따로 결제해야 한다.
- 계정마다 같은 보안 규칙(예: "서울 리전만 써라")을 하나하나 설정해야 하고, 누군가 바꿔도 알기 어렵다.
- 새 계정을 만들 때마다 로그 수집·보안 도구 설정을 반복해야 한다.
- 사람마다 계정별 로그인 정보를 따로 관리해야 한다.

**AWS Organizations**는 이 문제를 풀기 위해 계정들을 하나의 **조직(organization)**으로 묶는 서비스다. **추가 요금은 없다.** 각 계정이 쓴 리소스 요금만 낸다. ([Pricing](https://docs.aws.amazon.com/organizations/latest/userguide/pricing.html))

## 3단계: 조직의 구조

출처: [Terminology and concepts](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_getting-started_concepts.html)

```
Organization (o-exampleorgid)
└── Root (r-examplerootid111)            ← 조직의 최상위 컨테이너
    ├── Management account (111122223333) ← 조직을 만든 관리 계정
    ├── OU: Security
    │   ├── Log Archive 계정
    │   └── Security Tooling 계정
    ├── OU: Workloads
    │   ├── OU: Prod
    │   │   └── payments-prod 계정
    │   └── OU: SDLC
    │       └── payments-dev 계정
    └── OU: Sandbox
        └── kenny-sandbox 계정
```

| 용어 | 뜻 |
| --- | --- |
| **Organization** | 계정들을 묶은 조직 전체. 트리 구조 |
| **Management account** | 조직을 만든 계정. 계정 생성·초대·제거, 정책 연결, 서비스 연동을 하고 **조직 전체 요금을 냄**. 나중에 다른 계정으로 바꿀 수 없음. 예전 이름은 master account |
| **Member account** | management 계정을 뺀 나머지 계정. 한 번에 한 조직에만 속함 |
| **Root** | 조직 트리의 맨 위 컨테이너. 조직마다 하나 |
| **OU**(Organizational Unit) | 계정이나 다른 OU를 담는 폴더. 최대 5단계까지 중첩 가능 |

**주의: 여기서 말하는 "root"는 [root user](../aws-iam/01-overview.md#2단계-계정과-root-user)와 다르다.** 조직의 root는 트리의 맨 위 폴더이고, root user는 계정 하나의 최고 권한 로그인이다. 이 문서에서는 헷갈리지 않도록 조직의 것은 "조직 root"라고 쓴다.

OU를 쓰는 이유는 **정책 상속** 때문이다. OU에 정책을 걸면 그 아래 모든 OU와 계정이 그 정책을 물려받는다. 그래서 계정을 하나씩 설정하지 않고 "Prod OU에는 이 규칙"처럼 폴더 단위로 관리할 수 있다.

## 4단계: 기능 세트와 통합 결제

조직을 만들 때 두 가지 기능 세트 중 하나를 쓴다. ([Concepts](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_getting-started_concepts.html))

| 기능 세트 | 포함 기능 | 비고 |
| --- | --- | --- |
| **All features**(권장, 기본값) | 통합 결제 + 정책 + 서비스 연동 + 권한 위임 | 이 레포의 나머지 내용은 모두 이것을 전제로 함 |
| Consolidated billing features | 통합 결제만 | 정책을 쓸 수 없음 |

**통합 결제(consolidated billing)**는 모든 계정의 요금을 management 계정이 한 번에 내는 것이다. 장점은 다음과 같다. ([Consolidated billing](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/consolidated-billing.html))

- 청구서가 하나로 모인다.
- 계정별 요금을 한곳에서 볼 수 있다.
- 여러 계정의 사용량을 합쳐 **볼륨 할인**을 받는다. **Reserved Instance**와 **Savings Plans**(약정으로 받는 할인)도 계정 사이에 나눠 쓴다.

통합 결제만 쓰다가 all features로 바꾸려면, 초대받아 들어온 모든 member 계정이 전환 요청을 승인해야 한다.

## 5단계: 조직 정책 — 폴더 단위로 규칙 걸기

**조직 정책(organization policy)**은 조직 root, OU, 계정에 붙여 그 아래 모든 계정에 적용하는 정책이다. 현재 문서는 두 범주로 나눈다. ([Managing organization policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies.html))

### 권한 정책(authorization policies)

IAM 권한의 **상한선**을 긋는다. 권한을 직접 주지는 않는다. ([aws-iam/03](../aws-iam/03-policy-evaluation.md)에서 평가 순서를 다뤘다.)

| 정책 | 무엇의 상한선 | 예 |
| --- | --- | --- |
| **SCP**(Service Control Policy) | 계정 안의 **IAM user·role**이 할 수 있는 최대 범위 | "이 OU의 누구도 서울 외 리전을 못 씀" |
| **RCP**(Resource Control Policy) | 계정 안의 **리소스**에 허용될 수 있는 최대 범위 | "이 OU의 S3 버킷은 조직 밖에서 접근 불가" |

**SCP·RCP는 management 계정에는 적용되지 않는다.** 그래서 management 계정에는 워크로드를 두지 않는다(→ [02](./02-structure-ou.md)). 자세한 설계는 [03-scp-rcp.md](./03-scp-rcp.md)에서 다룬다.

### 선언형 정책(declarative policies)

서비스 설정의 **기본값이나 표준**을 조직 전체에 강제한다. 권한이 아니라 "설정 상태"를 정한다.

| 정책 | 무엇을 정하나 |
| --- | --- |
| Tag policies | 태그 키·값의 표준(대소문자, 허용 값) |
| Backup policies | AWS Backup 백업 계획 |
| AI services opt-out policies | AWS AI 서비스가 내 데이터를 서비스 개선에 쓰는 것을 거부 |
| Amazon EC2 policies | EC2 기본 설정(예: IMDSv2 강제, 공개 스냅샷 차단) |
| Chat applications policies | Slack·Teams 연동 앱 제어 |
| Security Hub / Inspector / GuardDuty / Bedrock / S3 policies 등 | 각 서비스의 조직 전체 설정 |

선언형 정책은 SCP와 달리 **management 계정에도 적용**된다. 자세한 내용은 [05-other-policies.md](./05-other-policies.md)에서 다룬다.

- 공식 문서 안에서도 정책 목록이 페이지마다 조금 다르다(예: Shield Network Security Director 정책은 개념 페이지에만 있음). 정확한 목록은 [Managing organization policies](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies.html)에서 확인한다.

## 6단계: 다른 AWS 서비스와 연동 — trusted access와 위임 관리자

GuardDuty(위협 탐지), Security Hub(보안 현황), Config(설정 기록), CloudTrail(API 기록) 같은 서비스는 조직 전체에 한 번에 켜는 기능이 있다. 이를 위한 두 가지 개념이 있다. ([Integrated services](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_integrate_services.html), [Delegated administrator](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_integrate_delegated_admin.html))

| 개념 | 뜻 |
| --- | --- |
| **Trusted access**(신뢰할 수 있는 접근) | 특정 AWS 서비스가 조직 안의 모든 계정에서 일할 수 있게 허락하는 것. 그 서비스는 각 계정에 **service-linked role**(서비스 전용 role)을 만들어 작업함 |
| **위임 관리자(delegated administrator)** | 그 서비스의 조직 전체 관리 권한을 management 계정 대신 **특정 member 계정에 맡기는 것**. 예: GuardDuty 관리를 Security Tooling 계정에 위임 |

위임 관리자를 쓰면 보안팀이 management 계정에 들어가지 않고도 조직 전체 보안 도구를 운영할 수 있다. 설정 방법은 [02](./02-structure-ou.md#위임-관리자)에서 다룬다.

## 7단계: 사람은 어떻게 여러 계정에 로그인하나

계정이 수십 개면 사람마다 계정별 IAM user를 만들 수 없다. AWS 권장 방식은 **IAM Identity Center**다.

- 사람은 한 번 로그인(**SSO**, Single Sign-On)한 뒤 접근 가능한 계정 목록에서 계정을 고른다.
- 관리자는 **permission set**(권한 묶음 템플릿, 예: "ReadOnly", "Admin")을 만들어 "누구에게, 어느 계정에, 어떤 permission set"을 할당한다.
- 그러면 Identity Center가 각 계정에 role을 만들고, 사용자는 그 role의 임시 자격증명으로 일한다.

자세한 내용은 [04-identity-center.md](./04-identity-center.md)에서 다룬다.

## 8단계: 계정 만들기와 관리 접근

조직에 계정을 넣는 방법은 두 가지다. ([Create an account](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_create.html))

| 방법 | 관리용 role |
| --- | --- |
| 조직에서 새로 **생성** | `OrganizationAccountAccessRole`이 자동 생성됨 → management 계정이 이 role을 맡아 새 계정을 관리자 권한으로 다룰 수 있음 |
| 기존 계정을 **초대** | 같은 이름의 role을 **직접 만들어야** 함 |

계정 생성·이동·해지와 OU 설계는 [02-structure-ou.md](./02-structure-ou.md)에서 다룬다.

## 9단계: Control Tower와의 관계

**AWS Control Tower**는 Organizations 위에 얹는 서비스로, AWS 권장 멀티 계정 환경(**landing zone**)을 자동으로 만들어 준다. 예를 들어 로그 보관·감사용 계정을 만들고, 권장 SCP와 설정 점검(**control**)을 미리 걸어 둔다(만드는 OU 구성은 landing zone 버전에 따라 다르다). Organizations를 직접 다루는 대신 Control Tower로 시작하는 경우가 많다. 자세한 관계는 [05-other-policies.md](./05-other-policies.md)에서 다룬다.

## 용어 정리

| 용어 | 뜻 | 처음 나오는 곳 |
| --- | --- | --- |
| Blast radius | 사고가 났을 때 피해가 미치는 범위 | 1단계 |
| AWS Organizations | 여러 계정을 조직으로 묶어 관리하는 무료 서비스 | 2단계 |
| Organization | 계정들을 묶은 조직 전체 | 2단계 |
| Management account | 조직을 만든 관리·결제 계정. SCP·RCP가 적용되지 않음 | 3단계 |
| Member account | 조직에 속한 나머지 계정 | 3단계 |
| 조직 root | 조직 트리의 최상위 컨테이너(root user와 다름) | 3단계 |
| OU | 계정·OU를 담는 폴더. 정책이 상속됨 | 3단계 |
| All features / Consolidated billing features | 전체 기능 / 통합 결제만 | 4단계 |
| 통합 결제 | management 계정이 모든 계정 요금을 한 번에 냄 | 4단계 |
| Reserved Instance / Savings Plans | 사용 약정으로 받는 할인 | 4단계 |
| 조직 정책 | 조직 root·OU·계정에 붙여 아래로 상속되는 정책 | 5단계 |
| SCP / RCP | IAM user·role / 리소스의 권한 상한선 | 5단계 |
| 선언형 정책 | 서비스 설정 기본값·표준을 강제하는 정책 | 5단계 |
| Trusted access | AWS 서비스가 조직 전체 계정에서 일하도록 허락 | 6단계 |
| Service-linked role | AWS 서비스가 계정에 만드는 전용 role | 6단계 |
| 위임 관리자 | 서비스 관리 권한을 특정 member 계정에 맡기는 것 | 6단계 |
| IAM Identity Center / SSO | AWS의 SSO 서비스 / 한 번 로그인으로 여러 시스템 사용 | 7단계 |
| Permission set | Identity Center에서 계정에 할당하는 권한 묶음 템플릿 | 7단계 |
| OrganizationAccountAccessRole | 조직에서 만든 계정에 자동 생성되는 관리용 role | 8단계 |
| Control | Control Tower가 거는 통제 규칙(SCP·Config 규칙 등) | 9단계 |
| AWS Control Tower / landing zone | 권장 멀티 계정 환경을 자동 구성하는 서비스 / 그렇게 만든 환경 | 9단계 |

## 출처

- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_introduction.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_getting-started_concepts.html
- https://docs.aws.amazon.com/organizations/latest/userguide/pricing.html
- https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/consolidated-billing.html
- https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/benefits-of-using-multiple-aws-accounts.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_integrate_services.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_integrate_delegated_admin.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_accounts_create.html

---
*조사 기준일: 2026-10-13. 트리 그림과 계정 이름은 이해를 돕기 위한 예시다.*
