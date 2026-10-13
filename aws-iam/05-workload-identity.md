# 워크로드 자격증명

> 워크로드에는 **장기 액세스 키를 주지 않는다.** AWS 안에서는 컴퓨팅 서비스가 role을 대신 assume해 임시 자격증명을 자동으로 공급하고(EC2 instance profile, Lambda/ECS role, EKS Pod Identity), AWS 밖에서는 **OIDC 토큰**(GitHub Actions, IRSA)이나 **X.509 인증서**(IAM Roles Anywhere)를 STS 임시 자격증명으로 바꾼다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md) 9단계(사람과 프로그램의 접근)와 [04-roles-sts.md](./04-roles-sts.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| IMDS | EC2 인스턴스 안에서만 접근되는 메타데이터 서비스(주소 `169.254.169.254`). role의 임시 자격증명도 여기서 받음 |
| IMDSv2 | 토큰을 먼저 받아야 조회할 수 있게 강화한 IMDS |
| SSRF | 서버를 속여 공격자가 원하는 내부 주소로 요청을 보내게 하는 공격. IMDS 자격증명 탈취에 악용됨 |
| Hop limit | 패킷이 거칠 수 있는 네트워크 단계 수 |
| ECS task | ECS에서 함께 실행되는 컨테이너 묶음 |
| Pod / service account / namespace | Kubernetes의 실행 단위 / Pod에 부여하는 신원 / 리소스를 나누는 구역 |
| DaemonSet | Kubernetes에서 모든 노드에 하나씩 실행되는 Pod |
| OIDC provider | IAM에 등록한 "이 토큰 발급자를 믿는다"는 설정 |
| `sub` / `aud` | OIDC 토큰 안의 값. `sub`는 토큰 주인(예: 어느 저장소·브랜치), `aud`는 토큰을 받을 대상 |
| X.509 인증서 / CA | 신원을 증명하는 디지털 인증서 / 인증서를 발급하는 기관 |
| CRL / OCSP | 폐기된 인증서 목록 / 인증서 폐기 여부를 실시간으로 묻는 방식 |

## 한눈에 보기

| 실행 환경 | 방식 | trust policy의 Principal | STS 호출 |
| --- | --- | --- | --- |
| EC2 | Instance profile + IMDS | `ec2.amazonaws.com` | EC2가 대신 |
| Lambda | Execution role | `lambda.amazonaws.com` | Lambda가 대신 |
| ECS | Task role(앱) / Task execution role(에이전트) | `ecs-tasks.amazonaws.com` | ECS가 대신 |
| EKS | **Pod Identity**(권장) | `pods.eks.amazonaws.com` | EKS Auth가 대신 |
| EKS 등 Kubernetes | IRSA | 클러스터 OIDC provider (`Federated`) | SDK가 `AssumeRoleWithWebIdentity` |
| GitHub Actions | OIDC | `token.actions.githubusercontent.com` (`Federated`) | `AssumeRoleWithWebIdentity` |
| 온프레미스·타 클라우드 | IAM Roles Anywhere | `rolesanywhere.amazonaws.com` | Roles Anywhere가 대신 |
| (최후 수단) | IAM user 액세스 키 | — | — |

## EC2: instance profile과 IMDS

출처: [IAM roles for EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html), [Instance metadata security credentials](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-metadata-security-credentials.html)

- instance profile은 role 하나를 담는 컨테이너다.
  - 콘솔에서 role을 만들면 같은 이름의 instance profile이 자동으로 생긴다.
  - CLI·API로 만들 때는 둘을 따로 만들어야 한다.
- 자격증명은 IMDS 경로 `iam/security-credentials/<role-name>`으로 제공된다.
  - AWS가 자동으로 교체하며, 만료 최소 5분 전에 새 자격증명을 내놓는다.
  - SDK는 이 경로를 자동으로 읽는다.
- EC2 role 자격증명에는 role의 max session duration 설정이 적용되지 않는다.
- 인스턴스에 붙은 role을 코드에서 다시 `sts:AssumeRole`하지 않는다. 꼭 그래야 한다면 그 role의 trust policy가 자기 자신을 명시적으로 허용해야 한다.
- 실행 중인 인스턴스의 profile에서 role을 빼는 방식은 반영에 최대 1시간이 걸릴 수 있다. AWS는 instance profile 자체를 교체하라고 권장한다.

### IMDSv2

출처: [Configure IMDS](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html), [Retrieve instance metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html)

```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

| 항목 | 내용 |
| --- | --- |
| 토큰 | `PUT`으로 발급, TTL 1초–6시간. 발급한 인스턴스에서만 유효 |
| 강제 | IMDSv2 필수로 설정하면 토큰 없는 요청은 `401` |
| SSRF 방어 | `X-Forwarded-For` 헤더가 있는 `PUT`은 거부 |
| hop limit | 기본 1. **컨테이너 안에서는 한 홉이 더 늘어** 응답을 못 받을 수 있으므로 AWS는 2로 올리라고 안내 |

```bash
aws ec2 modify-instance-metadata-options \
  --instance-id i-0123456789abcdef0 \
  --http-tokens required \
  --http-put-response-hop-limit 2
```

hop limit을 2로 올리면 같은 호스트의 컨테이너가 **인스턴스 role**을 쓸 수 있게 된다. ECS·EKS에서는 task role·Pod Identity로 분리하고, 컨테이너의 IMDS 접근은 막는 편이 안전하다(아래 ECS 참고).

### 자격증명 유출 탐지용 조건 키

EC2 role 자격증명으로 서명한 요청에는 `aws:Ec2InstanceSourceVpc`, `aws:Ec2InstanceSourcePrivateIPv4`가 붙는다. 이 값을 `aws:SourceVpc`와 비교하면 **자격증명이 원래 VPC 밖에서 쓰였는지** 탐지·차단할 수 있다. `aws:Ec2InstanceSourcePrivateIPv4`는 단독으로 Allow 조건에 쓰지 말라고 AWS가 명시한다. ([Condition keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html))

## Lambda

- execution role의 trust policy는 `lambda.amazonaws.com`을 신뢰한다. Lambda가 호출마다 자동으로 assume하므로 직접 `AssumeRole`하지 않는다.
- 콘솔 기본 role에는 CloudWatch Logs 쓰기용 `AWSLambdaBasicExecutionRole`이 붙는다.
- AWS는 Access Analyzer 정책 생성으로 이 role을 좁히라고 권장한다.

([Lambda execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html))

## ECS: task role vs task execution role

| 구분 | Task role | Task execution role |
| --- | --- | --- |
| 쓰는 주체 | **컨테이너 안의 앱 코드** | ECS·Fargate **에이전트** |
| 용도 | 앱이 S3·DynamoDB 등 호출 | ECR 이미지 pull, `awslogs` 로그 전송, Secrets Manager/Parameter Store 시크릿 주입 |
| 컨테이너에서 접근 | 가능(엔드포인트 169.254.170.2) | **불가** |
| 관리형 정책 | — | `AmazonECSTaskExecutionRolePolicy` |

출처: [Task IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html), [Task execution IAM role](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html)

- task role trust policy에는 confused deputy 방지용 `aws:SourceAccount`·`aws:SourceArn` 조건을 권장한다. 특정 클러스터 하나로 SourceArn을 고정하는 방식은 지원하지 않으므로 와일드카드를 쓴다.
- **EC2 기반 ECS**에서는 task 간 격리가 없어 컨테이너가 IMDS로 인스턴스 role에 닿을 수 있다. `ECS_AWSVPC_BLOCK_IMDS` 설정이나 169.254.169.254로 가는 트래픽을 막는 iptables 규칙으로 차단한다.

## EKS: Pod Identity vs IRSA

AWS는 가능하면 **Pod Identity를 쓰라고** 권장한다. ([Service accounts](https://docs.aws.amazon.com/eks/latest/userguide/service-accounts.html))

| 항목 | EKS Pod Identity | IRSA |
| --- | --- | --- |
| 동작 | 노드의 Pod Identity Agent(DaemonSet)가 EKS Auth API로 자격증명을 받아 Pod에 제공 | 클러스터 OIDC provider + Pod의 projected service account 토큰 → SDK가 `AssumeRoleWithWebIdentity` |
| trust policy | `pods.eks.amazonaws.com` 한 번만 신뢰 → **클러스터가 늘어도 수정 불필요** | 클러스터마다 OIDC provider 조건 추가 필요 |
| 한도 | 클러스터당 association 5,000개 | 계정당 OIDC provider 기본 100개, trust policy 2,048자라 role 하나에 약 4개 클러스터 |
| STS 호출 | EKS가 처리 | Pod마다 SDK가 호출 → 쓰로틀링 가능 |
| Session tags | 클러스터·namespace·service account 태그 자동 → ABAC 가능 | 없음 |
| 지원 환경 | Amazon EKS의 **Linux EC2 노드만**(Fargate, Windows 노드, Outposts, EKS Anywhere 미지원) | EKS, EKS Anywhere, ROSA, EC2 위 자체 Kubernetes |

출처: [Pod Identity](https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html), [IRSA](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html)

Pod Identity role의 trust policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "pods.eks.amazonaws.com" },
      "Action": ["sts:AssumeRole", "sts:TagSession"],
      "Condition": {
        "StringEquals": {
          "aws:RequestTag/kubernetes-namespace": "payments",
          "aws:RequestTag/kubernetes-service-account": "payments-api"
        }
      }
    }
  ]
}
```

([Pod Identity role](https://docs.aws.amazon.com/eks/latest/userguide/pod-id-role.html))

```bash
aws eks create-pod-identity-association \
  --cluster-name prod \
  --namespace payments \
  --service-account payments-api \
  --role-arn arn:aws:iam::111122223333:role/PaymentsApiRole
```

- Fargate에서 IRSA가 동작하는지는 이번 조사에서 1차 출처로 확인하지 못했다(확인 필요).
- `create-pod-identity-association` 명령의 옵션은 CLI 레퍼런스에서 다시 대조하지 않았다(확인 필요).

## GitHub Actions OIDC

출처: [GitHub Docs: OIDC in AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws), [Create a role for OIDC](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html)

1. IAM에 OIDC provider를 만든다. URL은 `https://token.actions.githubusercontent.com`, audience는 `sts.amazonaws.com`이다.
2. role의 trust policy에서 **`sub`로 저장소·브랜치(또는 environment)를 제한**한다.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::111122223333:oidc-provider/token.actions.githubusercontent.com"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
          "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:ref:refs/heads/main"
        }
      }
    }
  ]
}
```

3. 워크플로에서 `id-token: write` 권한을 주고 `aws-actions/configure-aws-credentials`를 쓴다.

```yaml
permissions:
  id-token: write
  contents: read
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4   # GitHub 문서는 커밋 SHA로 고정하는 예시를 씀
        with:
          role-to-assume: arn:aws:iam::111122223333:role/GitHubDeployRole
          aws-region: ap-northeast-2
      - run: aws sts get-caller-identity
```

- **`sub` 조건이 없거나 와일드카드뿐이면 IAM이 trust policy 저장을 거부한다.** AWS 경고: 조건이 없으면 "GitHub Actions from organizations or repositories outside of your control are able to assume roles." (같은 AWS 출처)
- `sub` 형식 예: `repo:ORG/REPO:ref:refs/heads/BRANCH`, `repo:ORG/REPO:environment:NAME`. 운영 배포 role은 **environment 단위**로 묶으면 GitHub의 환경 보호 규칙(승인자 등)과 함께 쓸 수 있다.
- **thumbprint**: AWS는 IdP 인증서를 신뢰된 루트 CA 목록으로 검증한다. thumbprint는 그 검증을 못 할 때만 쓰는 대비책이고, CLI·API에서 생략할 수 있다. ([Obtain the thumbprint](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc_verify-thumbprint.html))

## IAM Roles Anywhere

AWS 밖(온프레미스 서버, 타 클라우드)의 워크로드가 **자체 CA가 발급한 X.509 인증서**로 임시 자격증명을 받는 방식이다. ([Roles Anywhere](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/introduction.html), [Trust model](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/trust-model.html))

| 구성 요소 | 설명 |
| --- | --- |
| Trust anchor | 신뢰할 CA(AWS Private CA 또는 외부 CA 인증서) |
| Profile | assume할 수 있는 role 목록 + 선택적 session policy |
| Role trust policy | `rolesanywhere.amazonaws.com`에 `sts:AssumeRole`, `sts:TagSession`, `sts:SetSourceIdentity` 허용. `aws:SourceArn`을 trust anchor ARN으로 제한 권장 |
| 인증서 속성 | Subject·Issuer·SAN이 `aws:PrincipalTag/x509...` 태그가 됨 → 조건으로 특정 인증서만 허용 |
| 폐기 | 가져온 CRL로만 가능(OCSP 미지원) |

credential helper(`aws_signing_helper`)를 `~/.aws/config`의 `credential_process`로 연결하면 SDK가 그대로 쓴다. ([Credential helper](https://docs.aws.amazon.com/rolesanywhere/latest/userguide/credential-helper.html))

```ini
[profile onprem]
credential_process = aws_signing_helper credential-process \
  --certificate /etc/pki/app.crt --private-key /etc/pki/app.key \
  --trust-anchor-arn arn:aws:rolesanywhere:ap-northeast-2:111122223333:trust-anchor/TA_ID \
  --profile-arn arn:aws:rolesanywhere:ap-northeast-2:111122223333:profile/PROFILE_ID \
  --role-arn arn:aws:iam::111122223333:role/OnPremAppRole
```

위 ini 설정은 줄바꿈(`\`) 없이 한 줄로 써야 한다. 가독성을 위해 나눠 적었다.

## IAM user 액세스 키

> "Where possible, we recommend relying on temporary credentials instead of creating long-term credentials such as access keys." — [Best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)

꼭 써야 한다면 무중단으로 교체한다. ([Update access keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/id-credentials-access-keys-update.html))

```bash
aws iam create-access-key --user-name legacy-app            # 1. 두 번째 키 생성 → 앱에 배포
aws iam get-access-key-last-used --access-key-id AKIA...OLD # 2. 옛 키가 더 이상 안 쓰이는지 확인
aws iam update-access-key --user-name legacy-app \
  --access-key-id AKIA...OLD --status Inactive              # 3. 바로 지우지 말고 비활성화
aws iam delete-access-key --user-name legacy-app \
  --access-key-id AKIA...OLD                                # 4. 문제없으면 삭제
```

root user에는 액세스 키를 만들지 않는다.

## 이슈 / 주의

- **컨테이너와 IMDS**: hop limit을 2로 올리는 것은 "컨테이너가 인스턴스 role을 쓰게 하는 것"이다. 인스턴스 role이 넓다면 모든 Pod·task가 그 권한을 갖게 된다.
- **IRSA trust policy 크기**: 클러스터가 늘 때마다 OIDC 조건이 쌓여 2,048자 한도에 부딪힌다. 새로 시작한다면 Pod Identity를 쓴다.
- **GitHub `sub`를 org 단위(`repo:my-org/*`)로 열지 않는다.** 조직 안의 아무 저장소, 아무 브랜치(PR 포함)가 운영 role을 assume할 수 있다.
- **ECS execution role과 task role을 섞지 않는다.** 앱에 execution role 권한(모든 시크릿 읽기 등)을 줄 필요가 없다.

## 출처

- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-metadata-security-credentials.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/configuring-instance-metadata-service.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html
- https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_execution_IAM_role.html
- https://docs.aws.amazon.com/eks/latest/userguide/service-accounts.html
- https://docs.aws.amazon.com/eks/latest/userguide/pod-identities.html
- https://docs.aws.amazon.com/eks/latest/userguide/pod-id-role.html
- https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html
- https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_create_for-idp_oidc.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_create_oidc_verify-thumbprint.html
- https://docs.aws.amazon.com/rolesanywhere/latest/userguide/introduction.html
- https://docs.aws.amazon.com/rolesanywhere/latest/userguide/trust-model.html
- https://docs.aws.amazon.com/rolesanywhere/latest/userguide/credential-helper.html
- https://docs.aws.amazon.com/IAM/latest/UserGuide/id-credentials-access-keys-update.html

---
*조사 기준일: 2026-10-13. `configure-aws-credentials@v4` 버전 태그는 예시이며, GitHub 문서처럼 커밋 SHA로 고정하는 것이 더 안전하다.*
