# Amazon Managed Grafana(AMG)

> AMG는 Grafana 서버(**워크스페이스**)를 AWS가 대신 운영해 주는 시각화 서비스다. 사용자는 **IAM Identity Center나 SAML로만** 로그인하고(IAM user·role로는 못 들어감), 워크스페이스는 **자기 IAM role로 AMP·CloudWatch·X-Ray 같은 데이터 소스를 읽는다.** 요금은 **로그인한 사용자 수**(편집자 $9, 뷰어 $5/월)로 매겨지고, 워크스페이스마다 편집자 1명분은 항상 나간다. CLI·IaC로 만들 때는 **customer-managed 권한 방식만** 쓸 수 있다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)의 용어(Grafana, 대시보드, 데이터 소스, 워크스페이스)와 [aws-organizations/04](../aws-organizations/04-identity-center.md)의 IAM Identity Center를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| AMG 워크스페이스 | AWS가 관리하는 Grafana 서버 하나. 리전 리소스, ID는 `g-` + 16진수 10자리 |
| SAML 2.0 | 사내 IdP(Okta, Entra ID 등)와 로그인 정보를 주고받는 표준 |
| Grafana 역할(Admin/Editor/Viewer) | 워크스페이스 안에서의 권한 등급 |
| Team | Grafana 안의 사용자 묶음. IdP 그룹과 동기화 가능 |
| Service account | 사람 대신 API·자동화가 쓰는 Grafana 계정. 토큰으로 인증 |
| 워크스페이스 IAM role | 워크스페이스가 데이터 소스(AMP, CloudWatch 등)를 읽을 때 쓰는 IAM role |
| Contact point | Grafana 알림을 보낼 대상(SNS, Slack 등) |
| Prefix list | IP 대역 목록을 묶어 두는 VPC 기능 |

## 워크스페이스와 버전

출처: [What is AMG](https://docs.aws.amazon.com/grafana/latest/userguide/what-is-Amazon-Managed-Service-Grafana.html), [Create a workspace](https://docs.aws.amazon.com/grafana/latest/userguide/AMG-create-workspace.html), [Version differences](https://docs.aws.amazon.com/grafana/latest/userguide/version-differences.html), [Update version](https://docs.aws.amazon.com/grafana/latest/userguide/AMG-workspace-version-update.html)

| 항목 | 내용 |
| --- | --- |
| 단위 | 워크스페이스 = 독립된 Grafana 서버. 리전당 최대 5개(증설 가능) |
| 리전 | 서울 포함 |
| Grafana 버전 | **9, 10, 12, 13** 중 선택. 13은 대시보드를 GitHub로 관리하는 Git Sync 추가 |
| 버전 업그레이드 | **올리기만 가능, 되돌릴 수 없음.** 최대 약 10분, 그동안 읽기 전용. 플러그인은 자동으로 올라가지 않음 |
| 암호화 | 고객 관리형 KMS 키로 저장 데이터 암호화 가능 |
| IPv6 | 버전 10 이상에서 dual-stack |

## 로그인과 사용자

출처: [Authentication](https://docs.aws.amazon.com/grafana/latest/userguide/authentication-in-AMG.html), [IAM Identity Center](https://docs.aws.amazon.com/grafana/latest/userguide/authentication-in-AMG-SSO.html), [User roles](https://docs.aws.amazon.com/grafana/latest/userguide/Grafana-user-roles.html)

> "Amazon Managed Grafana does not support the use of IAM users and roles to assign permissions within an Amazon Managed Grafana workspace."

| 방식 | 사용자·역할을 정하는 곳 | 메모 |
| --- | --- | --- |
| **IAM Identity Center** | AMG 콘솔·API에서 사용자·그룹에 역할 할당 | AWS Organizations가 켜져 있어야 함(없으면 AMG가 켜 줌) |
| **SAML 2.0** | IdP가 보내는 SAML 속성(role 속성 값)으로 결정 | Okta, Entra ID 등을 직접 연결 |
| 둘 다 | 둘 다 사용 가능, 나중에 바꿀 수 있음 | — |

| Grafana 역할 | 할 수 있는 것 | 요금 |
| --- | --- | --- |
| Admin | 데이터 소스·사용자·설정 관리 | 편집자 요금 |
| Editor | 대시보드·알림 만들기·수정(데이터 소스 관리는 불가) | 편집자 요금 |
| Viewer | 대시보드 보기만(Explore 불가) | 뷰어 요금 |

- 사용자를 **Grafana 화면 안에서는 추가할 수 없다.** Identity Center라면 AMG 콘솔·API로, SAML이라면 IdP 쪽에서 정한다.
- IdP 그룹을 Grafana Team과 동기화할 수 있다.

```bash
# Identity Center 그룹을 편집자로 추가
aws grafana update-permissions --workspace-id g-0123456789 \
  --update-instruction-batch '[{"action":"ADD","role":"EDITOR","users":[{"id":"<identity-center-group-id>","type":"SSO_GROUP"}]}]'
```

### 자동화용: service account

출처: [API keys](https://docs.aws.amazon.com/grafana/latest/userguide/v10-using-api-keys.html), [Authenticating Grafana APIs](https://docs.aws.amazon.com/grafana/latest/userguide/v13-authenticating-grafana-apis.html)

- **API key는 deprecated이고 버전 12에서 제거되었다.** service account와 토큰을 쓴다.
- 토큰 수명은 **최대 30일**이므로 자동 갱신이 필요하다.
- **service account도 요금상 사용자 1명**으로 센다(Admin·Editor 역할이면 $9).

```bash
SA_ID=$(aws grafana create-workspace-service-account --workspace-id g-0123456789 \
  --name terraform --grafana-role ADMIN --query id --output text)
aws grafana create-workspace-service-account-token --workspace-id g-0123456789 \
  --service-account-id "$SA_ID" --name tf-token --seconds-to-live 2592000
```

## 데이터 소스와 권한

출처: [Manage permissions](https://docs.aws.amazon.com/grafana/latest/userguide/AMG-manage-permissions.html), [Built-in data sources](https://docs.aws.amazon.com/grafana/latest/userguide/AMG-data-sources-builtin.html), [AMG and Organizations](https://docs.aws.amazon.com/grafana/latest/userguide/AMG-and-Organizations.html)

워크스페이스는 **자기 IAM role**로 AWS 데이터 소스를 읽는다. 이 role을 누가 관리하느냐에 따라 세 가지 방식이 있다.

| 권한 방식 | role | 만들 수 있는 곳 |
| --- | --- | --- |
| Service-managed(현재 계정) | AMG가 `AmazonGrafanaServiceRole-<id>`를 만들고 관리형 정책을 붙임 | **콘솔만** |
| Service-managed(조직) | 선택한 OU의 각 계정에 `AmazonGrafanaOrgMemberRole-<id>`를 StackSets로 배포 | 콘솔만, management 계정 또는 위임 관리자 |
| **Customer-managed** | 내가 role을 만들고 ARN을 지정 | 콘솔·**CLI·API·CloudFormation** |

> API 레퍼런스: "Use only the CUSTOMER_MANAGED permission type when creating a workspace with the API, CLI or AWS CloudFormation."

- 콘솔에서 데이터 소스를 고르면 **IAM 권한만** 만들어진다. 데이터 소스 자체는 Grafana 안에서 따로 추가한다.
- SNS 알림 정책은 이름이 `grafana`로 시작하는 토픽에만 게시를 허용한다.
- 조직 방식이 StackSets로 role을 배포하는 원리는 [aws-cloudformation/04](../aws-cloudformation/04-stacksets.md)와 같다.
- member 계정 워크스페이스가 다른 계정 데이터를 볼 때의 규칙은 AWS 문서 두 곳이 다르게 말한다. 하나는 위임 관리자면 service-managed가 된다고 하고, 다른 하나는 customer-managed를 써야 한다고 한다(확인 필요). AWS는 워크스페이스를 management 계정에 두지 말라고 권한다.

| 주요 내장 데이터 소스 | 용도 |
| --- | --- |
| **Prometheus(AMP)** | 메트릭(PromQL) |
| **CloudWatch** | AWS 메트릭·로그 |
| **X-Ray**, Tempo | 트레이스 |
| OpenSearch, Loki | 로그 검색 |
| Athena, Redshift, Timestream, PostgreSQL, MySQL | SQL 데이터 |
| Alertmanager | Prometheus 계열 알림 상태 |
| Enterprise 플러그인(Datadog, Splunk, ServiceNow 등) | **사용자당 월 $45 추가**, Grafana Labs 토큰 연결 필요 |

### Customer-managed role 예시

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "grafana.amazonaws.com" },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": { "aws:SourceAccount": "111122223333" },
        "StringLike": { "aws:SourceArn": "arn:aws:grafana:ap-northeast-2:111122223333:/workspaces/*" }
      }
    }
  ]
}
```

위 trust policy의 형식과 `aws:SourceArn` 값의 정확한 모양은 이번 조사에서 공식 예시와 직접 대조하지 못했다(확인 필요). 권한 정책으로는 AMP 조회용 `AmazonPrometheusQueryAccess`, CloudWatch용 `AmazonGrafanaCloudWatchAccess` 등을 붙인다.

## 네트워크

출처: [VPC configuration](https://docs.aws.amazon.com/grafana/latest/userguide/AMG-configure-vpc.html), [Network access control](https://docs.aws.amazon.com/grafana/latest/userguide/AMG-configure-nac.html)

| 방향 | 기본 동작 | 바꾸는 방법 |
| --- | --- | --- |
| **나가는 쪽**(워크스페이스 → 데이터 소스) | 인터넷으로 나감. 단, 같은 리전 CloudWatch·AMP·X-Ray는 PrivateLink | **VPC 설정**: 서브넷마다 ENI를 만들어 모든 트래픽(알림 포함)을 VPC로 보냄. 이때 인터넷·다른 네트워크로 가려면 직접 경로를 만들어야 함 |
| **들어오는 쪽**(사용자 → 워크스페이스) | **모든 네트워크에서 접근 가능** | **Network access control**: prefix list(공인 IP)나 VPC endpoint(`com.amazonaws.<region>.grafana-workspace`)로 제한 |

private 서브넷의 RDS·자체 Prometheus를 데이터 소스로 쓰려면 VPC 설정이 필요하다. 사내망에서만 접속하게 하려면 network access control을 건다.

## 알림

출처: [Alerting](https://docs.aws.amazon.com/grafana/latest/userguide/v13-alerts.html), [Configure workspace](https://docs.aws.amazon.com/grafana/latest/userguide/AMG-configure-workspace.html), [External Alertmanager](https://docs.aws.amazon.com/grafana/latest/userguide/v13-alerting-setup-alertmanager.html)

| 항목 | 내용 |
| --- | --- |
| Grafana alerting | 규칙, 알림 정책, 묵음(silence), mute timing. Grafana 규칙과 Prometheus 계열 규칙을 한 화면에서 봄 |
| 평가 제한 | 규칙 평가 타임아웃 30초 고정. 워크스페이스당 알림 규칙 인스턴스 100개 |
| 버전 | 버전 12에서 classic alert 제거. **버전 9는 알림이 중복으로 가는 문제**가 있어 AWS는 10.4 이상을 권장 |
| SNS | contact point 타입 `sns`. 워크스페이스 생성 때 SNS를 고르면 IAM 정책만 생기고, contact point는 Grafana 안에서 만든다 |
| AMP Alertmanager | Alertmanager 데이터 소스로 상태를 볼 수 있지만, **AMP를 Grafana의 외부 Alertmanager로 쓸 수는 없음** |

### 알림을 어디서 할까

| 방식 | 장점 | 단점 |
| --- | --- | --- |
| **AMP 규칙 + Alertmanager**([02](./02-amp.md)) | Prometheus 표준, 규칙을 코드로 관리, Grafana가 꺼져도 동작 | 알림 경로가 SNS 중심 |
| **Grafana alerting** | AMP·CloudWatch 등 여러 데이터 소스를 섞은 규칙, UI로 관리 | 워크스페이스당 100개 한도, 30초 평가 제한 |

AWS가 둘 중 무엇을 권장하는지는 문서에서 찾지 못했다. 메트릭 알림은 AMP 규칙으로 코드화하고, 여러 소스를 섞는 알림만 Grafana로 하는 분리도 흔하다(이 레포의 정리).

## 사용법 / 예시

출처: [create-workspace](https://docs.aws.amazon.com/cli/latest/reference/grafana/create-workspace.html), [CreateWorkspace API](https://docs.aws.amazon.com/grafana/latest/APIReference/API_CreateWorkspace.html)

```bash
aws grafana create-workspace \
  --workspace-name observability \
  --account-access-type CURRENT_ACCOUNT \
  --authentication-providers AWS_SSO \
  --permission-type CUSTOMER_MANAGED \
  --workspace-role-arn arn:aws:iam::111122223333:role/amg-workspace-role \
  --workspace-notification-destinations SNS \
  --grafana-version 12.4

aws grafana describe-workspace --workspace-id g-0123456789 \
  --query 'workspace.[status,endpoint,grafanaVersion]'
aws grafana list-versions
```

- `--workspace-data-sources`는 API 문서에 "for internal use only, and should not be used"라고 적혀 있다. 쓰지 않는다.
- `--grafana-version`에 넣을 정확한 값은 `list-versions`로 확인한다. 위의 `12.4`는 예시이며 실제 값은 확인 필요다.
- 대시보드·알림을 코드로 관리하려면 Terraform `grafana/grafana` provider를 service account 토큰으로 연결한다. AWS 문서는 provider 1.27.0 이상으로 알림 리소스를 만드는 예시를 든다. ([Provision alerting with Terraform](https://docs.aws.amazon.com/grafana/latest/userguide/v13-alerting-setup-provision-terraform.html))

## 요금과 한도

출처: [AMG pricing](https://aws.amazon.com/grafana/pricing/), [Quotas](https://docs.aws.amazon.com/grafana/latest/userguide/AMG_quotas.html)

| 항목 | 요금 |
| --- | --- |
| Editor·Admin 사용자 | 워크스페이스당 활성 사용자 1명에 월 $9 |
| Viewer 사용자 | 워크스페이스당 활성 사용자 1명에 월 $5 |
| 최소 요금 | 워크스페이스마다 **편집자 1명분은 로그인이 없어도** 과금 |
| 활성 사용자 | 그달에 한 번이라도 로그인하거나 API를 호출한 사용자 |
| Service account | 역할에 따라 $9 또는 $5 |
| Enterprise 플러그인 | 활성 사용자당 월 $45 추가 |
| 무료 체험 | 90일, 계정당 사용자 5명 |

```
예: 편집자 20명 + 뷰어 30명 = 20 × $9 + 30 × $5 = $330/월 (요금 페이지 예시)
```

| 한도(워크스페이스당) | 값 | 증설 |
| --- | --- | --- |
| 워크스페이스 | 리전당 5개 | **가능(유일하게)** |
| 사용자 | 등록 10,000명, 동시 500명 | 불가 |
| 대시보드 / 데이터 소스 | 각 2,000 | 불가 |
| 알림 규칙 인스턴스 | 100 | 불가 |
| Service account / 토큰 | 각 100(만료된 토큰도 셈) | 불가 |
| VPC endpoint / prefix list | 각 5 | 불가 |

요금 페이지에 리전별 금액이 따로 없어 서울도 같은 단가로 보이지만, 직접 확인하지는 못했다.

## 이슈 / 주의

- **IAM user·role로는 로그인할 수 없다.** Identity Center나 SAML을 먼저 준비한다.
- **CLI·IaC는 customer-managed 권한만** 쓸 수 있다. 콘솔에서 만든 service-managed 워크스페이스와 설정이 다르니 처음부터 방식을 정한다.
- **버전 업그레이드는 되돌릴 수 없다.** 플러그인 호환성을 먼저 확인한다.
- **기본은 인터넷 어디서나 접속 가능**이다. 운영 대시보드는 network access control로 막는다.
- **service account도 과금 사용자**다. 자동화용 계정을 무분별하게 만들지 않는다. 만료된 토큰도 100개 한도에 들어가므로 정리한다.
- **알림 규칙 100개 한도**가 작다. 대량의 메트릭 알림은 AMP 규칙으로 둔다.

## 출처

- https://docs.aws.amazon.com/grafana/latest/userguide/what-is-Amazon-Managed-Service-Grafana.html
- https://docs.aws.amazon.com/grafana/latest/userguide/AMG-create-workspace.html
- https://docs.aws.amazon.com/grafana/latest/userguide/version-differences.html
- https://docs.aws.amazon.com/grafana/latest/userguide/AMG-workspace-version-update.html
- https://docs.aws.amazon.com/grafana/latest/userguide/authentication-in-AMG.html
- https://docs.aws.amazon.com/grafana/latest/userguide/authentication-in-AMG-SSO.html
- https://docs.aws.amazon.com/grafana/latest/userguide/Grafana-user-roles.html
- https://docs.aws.amazon.com/grafana/latest/userguide/v10-using-api-keys.html
- https://docs.aws.amazon.com/grafana/latest/userguide/v13-authenticating-grafana-apis.html
- https://docs.aws.amazon.com/grafana/latest/userguide/AMG-manage-permissions.html
- https://docs.aws.amazon.com/grafana/latest/userguide/AMG-data-sources-builtin.html
- https://docs.aws.amazon.com/grafana/latest/userguide/AMG-and-Organizations.html
- https://docs.aws.amazon.com/grafana/latest/userguide/AMG-configure-vpc.html
- https://docs.aws.amazon.com/grafana/latest/userguide/AMG-configure-nac.html
- https://docs.aws.amazon.com/grafana/latest/userguide/v13-alerts.html
- https://docs.aws.amazon.com/grafana/latest/userguide/AMG-configure-workspace.html
- https://docs.aws.amazon.com/grafana/latest/userguide/v13-alerting-setup-alertmanager.html
- https://docs.aws.amazon.com/grafana/latest/userguide/v13-alerting-setup-provision-terraform.html
- https://docs.aws.amazon.com/grafana/latest/userguide/AMG_quotas.html
- https://docs.aws.amazon.com/grafana/latest/APIReference/API_CreateWorkspace.html
- https://docs.aws.amazon.com/cli/latest/reference/grafana/create-workspace.html
- https://docs.aws.amazon.com/cli/latest/reference/grafana/update-permissions.html
- https://aws.amazon.com/grafana/pricing/

---
*조사 기준일: 2026-10-20. CLI 예시는 레퍼런스의 옵션으로 조합했고 실제 계정에서 실행하지 않았다.*
