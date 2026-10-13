# AWS IAM 개요

> IAM(Identity and Access Management)은 AWS 계정 안에서 **"요청한 사람이 누구인지 확인하고(인증), 그 요청을 허용할지 정하는(인가)"** 서비스다. 핵심 재료는 두 가지다. **신원**(user, role)과 신원에게 붙이는 **정책**(JSON 권한 문서)이다. AWS는 사람이든 프로그램이든 오래 쓰는 비밀번호·키 대신 **잠깐 쓰고 버리는 임시 자격증명**을 쓰라고 권장한다.

이 문서는 IAM을 처음 보는 사람을 기준으로, 용어를 한 단계씩 쌓아 가며 설명한다. 각 단계에서 처음 나오는 용어는 **굵게** 표시하고 바로 뜻을 적는다. 모든 용어는 맨 아래 [용어 정리](#용어-정리)에도 모아 두었다.

## 1단계: IAM이 푸는 문제

회사 건물에 비유하면 이렇다.

| 건물 | AWS | 설명 |
| --- | --- | --- |
| 출입증으로 "누구인지" 확인 | **인증(authentication)** | 요청이 정말 그 사람·프로그램이 보낸 것인지 확인 |
| 출입증 권한으로 "어느 방에 들어갈 수 있는지" 확인 | **인가(authorization)** | 확인된 요청자가 그 작업을 해도 되는지 판단 |
| 출입 관리 시스템 | **IAM** | 둘 다 담당 |

> IAM은 "a web service that helps you securely control access to AWS resources"이며 "who is authenticated (signed in) and authorized (has permissions)"를 제어한다. — [What is IAM?](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html)

AWS에서 일어나는 모든 작업은 **API 요청**이다. 콘솔에서 버튼을 누르든, CLI에서 `aws s3 ls`를 치든, 프로그램이 SDK로 호출하든 결국 "S3에 목록 조회 요청" 같은 API 호출이 된다. IAM은 이 요청 하나하나를 검사한다.

## 2단계: 계정과 root user

- **AWS 계정(account)**: AWS 리소스(서버, 버킷 등)와 청구가 묶이는 단위다. 12자리 숫자 ID가 붙는다(예: `111122223333`). IAM 설정도 계정마다 따로 있다.
- **Root user**: 계정을 처음 만들 때 쓴 이메일·비밀번호로 로그인하는 신원이다. 계정 안의 **모든 것을 할 수 있고**, IAM 정책으로 권한을 줄일 수도 없다.

그래서 root user는 평소에 쓰지 않는다. 처음 설정할 때와 root만 할 수 있는 작업(계정 해지 등)에만 쓰고, 일상 작업은 아래에서 설명할 다른 신원으로 한다. root 보호 방법은 [8단계](#8단계-root-user-보호)에서 다룬다.

## 3단계: 신원 — user, group, role

IAM에서 "누구"에 해당하는 것을 **신원(identity)**이라고 한다. 세 가지가 있다.

| 신원 | 무엇 | 비유 |
| --- | --- | --- |
| **IAM user** | 계정 안에 만든 사람 또는 프로그램 하나. 자기 비밀번호·키를 가짐 | 직원 개인 출입증 |
| **IAM group** | user 여러 명을 묶은 것. group에 권한을 주면 소속 user가 모두 받음 | 부서 |
| **IAM role** | 자기 비밀번호·키가 **없는** 신원. 허락된 누군가가 잠시 "맡아서(assume)" 씀 | 방문자용 임시 출입증 |

**role**이 IAM을 이해하는 핵심이다.

- role은 "이 역할을 맡으면 이런 일을 할 수 있다"는 권한 묶음이다.
- user, 다른 계정, AWS 서비스(예: EC2 서버) 등 **허락된 대상이 role을 맡으면(assume)**, 그 순간 쓸 수 있는 **임시 자격증명**을 받는다.
- 임시 자격증명은 정해진 시간(기본 1시간)이 지나면 저절로 만료된다.

user를 많이 만들어 각각 영구 키를 나눠 주는 것보다, role을 만들어 필요할 때만 맡게 하는 편이 훨씬 안전하다. 키가 새어 나가도 곧 만료되기 때문이다.

## 4단계: 자격증명 — 무엇으로 "나"를 증명하나

**자격증명(credential)**은 인증할 때 내미는 증거다.

| 자격증명 | 누가 씀 | 수명 | 설명 |
| --- | --- | --- | --- |
| 비밀번호 | root user, IAM user | 바꿀 때까지 | 콘솔(웹) 로그인용 |
| **MFA**(Multi-Factor Authentication, 다중 인증) | root user, IAM user | — | 비밀번호에 더해 휴대폰 앱 코드, 보안 키 등 "가진 것"으로 한 번 더 확인 |
| **액세스 키(access key)** | IAM user, root user | **삭제할 때까지(장기)** | CLI·SDK용. 키 ID(`AKIA…`) + 비밀 키 한 쌍 |
| **임시 자격증명(temporary credentials)** | role을 맡은 쪽 | **15분~최대 12시간** | 키 ID(`ASIA…`) + 비밀 키 + 세션 토큰. 만료되면 못 씀 |

임시 자격증명을 발급하는 AWS 서비스가 **STS**(Security Token Service)다. "role을 맡는다"는 것은 실제로는 STS의 `AssumeRole` API를 호출해 임시 자격증명을 받는 것이다. 자세한 내용은 [04-roles-sts.md](./04-roles-sts.md)에서 다룬다.

AWS는 **장기 자격증명(액세스 키)보다 임시 자격증명을 쓰라**고 권장한다. 이 권장이 IAM 설계 전체의 방향을 정한다. ([Best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html))

## 5단계: 정책 — 무엇을 할 수 있나

**정책(policy)**은 "어떤 작업을 어떤 리소스에 대해 허용(또는 거부)한다"를 적은 JSON 문서다. 신원에 정책을 붙이면 그 신원이 할 수 있는 일이 정해진다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::my-bucket/*"
    }
  ]
}
```

이 정책은 "`my-bucket` 버킷 안의 파일을 읽는 것(`s3:GetObject`)을 허용한다"는 뜻이다.

| 필드 | 뜻 |
| --- | --- |
| `Effect` | `Allow`(허용) 또는 `Deny`(거부) |
| `Action` | 어떤 작업인지. `서비스:작업` 형식(`s3:GetObject`, `ec2:StartInstances`) |
| `Resource` | 어떤 대상에 대해서인지. ARN(아래 6단계)으로 적음 |

꼭 기억할 규칙 두 가지:

1. **아무 정책도 없으면 거부다.** 이를 **암묵적 거부(implicit deny)**라고 한다. 허용하려면 누군가 명시적으로 Allow해야 한다.
2. **어디든 `Deny`가 하나라도 있으면 Allow보다 우선한다.** 이를 **명시적 거부(explicit deny)**라고 한다.

정책은 붙는 위치에 따라 크게 두 종류다.

| 종류 | 어디에 붙나 | 예 |
| --- | --- | --- |
| **Identity-based policy** | 신원(user, group, role)에 | "이 role은 S3 읽기 가능" |
| **Resource-based policy** | 리소스(S3 버킷, KMS 키 등)에 | "이 버킷은 저 계정이 읽어도 됨" |

이 밖에 권한을 주지는 않고 **상한선만 긋는** 정책들(permissions boundary, SCP, RCP 등)이 있다. 이것들은 [02-policies.md](./02-policies.md)에서 하나씩 설명한다.

## 6단계: 요청 하나가 처리되는 과정

1. **요청을 보낸 주체** — 이를 **principal**이라고 한다 — 가 서명한 API 요청을 보낸다. principal은 root user, IAM user, role을 맡은 세션 등 "요청을 보낸 쪽"을 통틀어 부르는 말이다.
2. AWS가 서명을 확인해 principal을 **인증**한다.
3. AWS가 요청 정보를 모은다. 누가(principal), 무엇을(action), 어디에(resource), 어떤 상황에서(IP, 시각, HTTPS 여부 등)인지다. 이 묶음을 **request context**라고 한다.
4. 관련된 모든 정책을 request context와 비교해 **인가** 여부를 결정한다. 정책의 `Condition` 필드는 "IP가 이 범위일 때만" 같은 상황 조건을 이 request context와 비교한다.

([How IAM works](https://docs.aws.amazon.com/IAM/latest/UserGuide/intro-structure.html))

여러 정책이 겹칠 때의 판정 순서는 [03-policy-evaluation.md](./03-policy-evaluation.md)에서 다룬다.

### Principal 종류 정리

이제 앞에서 나온 신원들을 "요청을 보내는 주체" 관점에서 다시 정리한다. ([Principal element](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_elements_principal.html))

| Principal | 설명 |
| --- | --- |
| AWS 계정 / root user | 2단계 |
| IAM user | 3단계 |
| IAM role / **role session** | role 자체, 그리고 누군가 role을 맡아서 생긴 "세션"(임시 자격증명으로 요청하는 상태) |
| **Federated user** | AWS 밖의 로그인 시스템(아래 9단계 IdP)으로 로그인한 뒤 AWS에 들어온 사용자 |
| **AWS 서비스** | `lambda.amazonaws.com`처럼 AWS 서비스 자체. "Lambda가 이 role을 맡아도 된다"처럼 씀 |
| `*` | 모든 principal(익명 포함). 매우 조심해서 써야 함 |

**IAM group은 principal이 아니다.** group은 user에게 정책을 나눠 주는 묶음일 뿐, 요청을 보내지 않는다.

## 7단계: ARN과 고유 ID

AWS의 모든 리소스와 신원에는 **ARN**(Amazon Resource Name)이라는 고유 이름이 있다. 정책의 `Resource`나 `Principal`에 이 ARN을 적는다.

```
arn:aws:iam::111122223333:role/S3Access
│   │   │   │            └─ 리소스 종류/이름
│   │   │   └─ 계정 ID
│   │   └─ 리전 (IAM은 비어 있음)
│   └─ 서비스
└─ 파티션(일반 리전은 aws)
```

| 대상 | ARN 예 |
| --- | --- |
| root | `arn:aws:iam::111122223333:root` |
| user | `arn:aws:iam::111122223333:user/Jane` |
| role | `arn:aws:iam::111122223333:role/S3Access` |
| role session | `arn:aws:sts::111122223333:assumed-role/S3Access/kenny` (STS가 만든 세션이라 `sts`) |
| 정책 | `arn:aws:iam::111122223333:policy/MyPolicy` |

([IAM identifiers](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html))

IAM은 내부적으로 이름과 별도로 **고유 ID**도 붙인다. 접두사를 보면 종류를 알 수 있다: `AIDA`(user), `AROA`(role), `AKIA`(장기 액세스 키), `ASIA`(임시 액세스 키). 로그에서 `ASIA`로 시작하는 키가 보이면 임시 자격증명이라는 뜻이다.

## 8단계: Root user 보호

[Root user best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/root-user-best-practices.html) 요약:

- 강한 비밀번호와 **MFA**를 설정한다(최대 8개 디바이스 등록 가능).
- **root 액세스 키는 만들지 않는다.**
- 개인 메일 대신 그룹 메일 주소를 쓰고, 계정 복구 수단에 접근할 수 있는 사람을 제한한다.
- root 로그인과 사용을 모니터링한다.

계정을 여러 개 묶어 관리하는 **AWS Organizations**를 쓰면, 멤버 계정의 root 자격증명을 아예 없애고 필요할 때만 중앙에서 root 작업을 하는 기능도 있다. Organizations는 다음 주제(`aws-organizations`)에서 다룬다.

## 9단계: 사람과 프로그램은 어떻게 접근해야 하나

지금까지의 재료(role, 임시 자격증명)로 AWS의 권장 방식을 이해할 수 있다.

### 사람: 페더레이션

- **IdP**(Identity Provider)는 회사가 이미 쓰는 로그인 시스템이다(Okta, Microsoft Entra ID, Google Workspace 등).
- **페더레이션(federation)**은 IdP에서 로그인한 사용자를 AWS가 믿고 받아들이는 방식이다. 사용자는 IdP로 로그인하고, AWS는 role의 임시 자격증명을 내준다. AWS 쪽에 사람마다 IAM user를 만들 필요가 없다.
- IdP와 AWS가 신원 정보를 주고받는 표준 규격으로 **SAML**과 **OIDC**(OpenID Connect)가 쓰인다.
- **IAM Identity Center**는 AWS가 제공하는 SSO(한 번 로그인으로 여러 계정 접근) 서비스다. 사람의 페더레이션 접근을 설정할 때 AWS가 권장하는 출발점이다(자세한 내용은 `aws-organizations`에서 다룬다).

> "Require human users to use federation with an identity provider to access AWS using temporary credentials" — [Best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

### 프로그램(워크로드): role

**워크로드(workload)**는 서버, 컨테이너, 함수, CI 파이프라인처럼 사람이 아니라 프로그램이 AWS를 호출하는 경우를 말한다. 이때도 액세스 키를 코드에 넣지 않고 role을 쓴다.

| 실행 위치 | 방식 | 한 줄 설명 |
| --- | --- | --- |
| EC2 서버 | **instance profile** | 서버에 role을 붙이면, 서버 안 프로그램이 임시 자격증명을 자동으로 받음 |
| Lambda 함수 | **execution role** | 함수가 실행될 때 Lambda가 role을 대신 맡아 줌 |
| ECS 컨테이너 | **task role** | 컨테이너(task)마다 role을 붙임 |
| EKS(Kubernetes) | **Pod Identity** | Pod마다 role을 붙임 |
| GitHub Actions 같은 외부 CI | **OIDC** | CI가 발급한 토큰을 AWS STS에 내고 임시 자격증명으로 교환 |
| 회사 서버(AWS 밖) | **IAM Roles Anywhere** | 회사 인증서로 신원을 증명하고 임시 자격증명을 받음 |

> "Require workloads to use temporary credentials with IAM roles to access AWS" — [Best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

각 방식의 동작과 설정은 [05-workload-identity.md](./05-workload-identity.md)에서 다룬다.

## 10단계: 최소 권한과 AWS 권장 사항 전체

**최소 권한(least privilege)**은 "필요한 작업만, 필요한 리소스에만" 허용하는 원칙이다. 처음부터 정확히 맞추기는 어려우므로 AWS는 단계적으로 좁히라고 권한다.

[Security best practices in IAM](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)의 항목을 이 문서의 용어로 풀면 다음과 같다.

| # | 권장 사항 | 쉽게 말하면 | 자세히 |
| --- | --- | --- | --- |
| 1 | 사람은 페더레이션 + 임시 자격증명 | 사람마다 IAM user·영구 키를 만들지 않는다 | 9단계 |
| 2 | 워크로드는 role의 임시 자격증명 | 코드에 액세스 키를 넣지 않는다 | [05](./05-workload-identity.md) |
| 3 | MFA 필수 | 피싱에 강한 passkey·보안 키 권장 | 4단계 |
| 4 | 장기 키가 꼭 필요하면 필요할 때 교체 | 어쩔 수 없이 쓰는 액세스 키는 관리한다 | [05](./05-workload-identity.md#iam-user-액세스-키) |
| 5 | root user 보호 | — | 8단계 |
| 6 | 최소 권한 적용 | — | [06](./06-least-privilege.md) |
| 7 | **AWS managed policy**에서 시작해 좁혀 가기 | AWS가 미리 만들어 둔 정책으로 시작한 뒤, 직접 만든 정책으로 줄여 간다 | [02](./02-policies.md) |
| 8 | **IAM Access Analyzer**로 정책 생성 | 실제 사용 기록을 보고 필요한 권한만 담은 정책 초안을 만들어 주는 AWS 도구 | [06](./06-least-privilege.md) |
| 9 | 안 쓰는 user·role·권한·키 정리 | — | [06](./06-least-privilege.md) |
| 10 | 정책에 조건(Condition) 사용 | "회사 IP에서만", "MFA를 했을 때만" 같은 조건 추가 | [02](./02-policies.md) |
| 11 | Access Analyzer로 외부 공개·교차 계정 접근 확인 | 다른 계정·인터넷에 열린 리소스를 찾아 준다 | [06](./06-least-privilege.md) |
| 12 | Access Analyzer로 정책 검증 | 정책 문법 오류·위험한 패턴을 검사 | [06](./06-least-privilege.md) |
| 13 | 여러 계정에 가드레일 설정 | Organizations의 **SCP·RCP**로 계정 전체의 상한선을 긋는다 | [03](./03-policy-evaluation.md) |
| 14 | **permissions boundary**로 권한 관리 위임 | "이 이상은 못 준다"는 상한선을 걸어 두고 팀에 role 생성을 맡긴다 | [02](./02-policies.md) |

## 11단계: 알아 두면 좋은 서비스 특성

| 특징 | 내용 | 출처 |
| --- | --- | --- |
| 요금 | IAM, IAM Identity Center, STS는 **추가 요금 없음**. IAM Access Analyzer의 일부 기능은 유료([06](./06-least-privilege.md)) | [What is IAM?](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) |
| **글로벌 서비스** | IAM 신원·정책은 리전(서울, 버지니아 등 데이터센터 지역)에 속하지 않고 계정 전체에 적용됨. 그래서 IAM ARN의 리전 칸이 비어 있음. IAM API 엔드포인트는 us-east-1 한 곳 | [Identifiers](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_identifiers.html), [Condition keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html) |
| **Eventual consistency**(최종 일관성) | IAM 변경은 전 세계 서버에 복제되는 데 시간이 걸려, 바꾼 직후에는 반영이 안 된 것처럼 보일 수 있음. 서비스 시작 경로 같은 중요한 코드에서 IAM을 바꾸지 말 것 | [What is IAM?](https://docs.aws.amazon.com/IAM/latest/UserGuide/introduction.html) |

### 주요 한도(quota)

**한도**는 계정당 만들 수 있는 개수나 크기의 상한이다. 일부는 **Service Quotas**(한도 조회·증설 요청 콘솔)에서 늘릴 수 있다. ([IAM and STS quotas](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_iam-quotas.html))

| 항목 | 기본값 | 최대(증설 시) |
| --- | --- | --- |
| 계정당 role | 1,000 | 10,000 |
| 계정당 직접 만든 정책(customer managed policy) | 1,500 | 10,000 |
| role 하나에 붙일 수 있는 managed policy | 20 | 25 |
| user 하나에 붙일 수 있는 managed policy | 10 | 20 |
| role을 누가 맡을 수 있는지 적는 정책(trust policy) 길이 | 2,048자 | 8,192자 |
| managed policy 하나의 크기 | 6,144자(고정) | — |
| role 세션 길이 | 1–12시간(기본 1시간) | — |
| STS 요청 | 계정·리전당 초당 600 | — |

정책 크기에 한도가 있으므로, 권한을 정책 하나에 몰아넣기보다 용도별로 나누는 편이 좋다.

## 용어 정리

| 용어 | 뜻 | 처음 나오는 곳 |
| --- | --- | --- |
| 인증(authentication) | 요청자가 누구인지 확인 | 1단계 |
| 인가(authorization) | 그 요청을 허용할지 판단 | 1단계 |
| AWS 계정 | 리소스·청구·IAM 설정이 묶이는 단위, 12자리 ID | 2단계 |
| Root user | 계정 생성 이메일로 로그인하는 최고 권한 신원 | 2단계 |
| IAM user / group / role | 개인 신원 / user 묶음 / 맡아서 쓰는 임시 신원 | 3단계 |
| Assume | role을 맡아 임시 자격증명을 받는 것 | 3단계 |
| 자격증명(credential) | 신원을 증명하는 비밀번호·키·토큰 | 4단계 |
| MFA | 비밀번호 외에 추가로 확인하는 다중 인증 | 4단계 |
| 액세스 키 | IAM user의 장기 프로그래밍용 키(`AKIA…`) | 4단계 |
| 임시 자격증명 | 만료되는 키·토큰(`ASIA…`) | 4단계 |
| STS | 임시 자격증명을 발급하는 서비스 | 4단계 |
| 정책(policy) | 허용·거부를 적은 JSON 문서 | 5단계 |
| 암묵적 거부 / 명시적 거부 | 허용이 없어서 거부 / `Deny`로 거부 | 5단계 |
| Identity-based / Resource-based policy | 신원에 붙는 정책 / 리소스에 붙는 정책 | 5단계 |
| Principal | 요청을 보낸 주체 | 6단계 |
| Request context | 요청의 누가·무엇·어디·상황 정보 묶음 | 6단계 |
| Role session | role을 맡아 생긴 세션 | 6단계 |
| Federated user | 외부 IdP로 로그인해 들어온 사용자 | 6단계 |
| ARN | AWS 리소스·신원의 고유 이름 | 7단계 |
| AWS Organizations | 여러 AWS 계정을 묶어 관리하는 서비스 | 8단계 |
| IdP / 페더레이션 | 외부 로그인 시스템 / 그 로그인을 AWS가 믿고 받아들이는 방식 | 9단계 |
| SAML / OIDC | IdP와 신원 정보를 주고받는 표준 | 9단계 |
| IAM Identity Center | AWS의 SSO 서비스 | 9단계 |
| 워크로드 | AWS를 호출하는 프로그램(서버·컨테이너·함수·CI) | 9단계 |
| Instance profile / execution role / task role / Pod Identity | EC2 / Lambda / ECS / EKS에서 role을 붙이는 방식 | 9단계 |
| IAM Roles Anywhere | AWS 밖 서버가 인증서로 role을 맡는 방식 | 9단계 |
| 최소 권한 | 필요한 것만 허용하는 원칙 | 10단계 |
| AWS managed / customer managed policy | AWS가 만든 정책 / 내가 만든 정책 | 10단계 |
| IAM Access Analyzer | 정책 생성·검증, 외부 공개 리소스 탐지 도구 | 10단계 |
| SCP / RCP | Organizations에서 계정의 사용자·리소스 권한 상한선을 긋는 정책 | 10단계 |
| Permissions boundary | user·role 하나의 권한 상한선 | 10단계 |
| Trust policy | role에 붙어 누가 그 role을 맡을 수 있는지 정하는 정책 | 11단계 |
| 한도(quota) / Service Quotas | 개수·크기 상한 / 한도 조회·증설 콘솔 | 11단계 |
| 글로벌 서비스 | 리전에 속하지 않고 계정 전체에 적용 | 11단계 |
| Eventual consistency | 변경이 전체에 반영되기까지 시간이 걸림 | 11단계 |

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
*조사 기준일: 2026-10-13. 건물 비유와 ARN 구조 그림은 이해를 돕기 위해 이 레포에서 만든 것이다. 한도 값은 Service Quotas 콘솔에서 다시 확인할 것.*
