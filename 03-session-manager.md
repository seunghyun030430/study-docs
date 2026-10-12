# Session Manager

> 인바운드 포트·bastion·SSH 키 없이 IAM 권한만으로 노드에 셸/포트포워딩 접속을 제공한다. 핵심은 **IAM으로 "누가 어느 노드에 어떤 문서로" 접속하는지 제어**하고 **세션 로그를 S3/CloudWatch Logs로 남기는 것**이다. 단, 포트포워딩·SSH 세션은 로그가 남지 않는다.

## 개요 / 정의

Session Manager는 Systems Manager의 노드 접속 기능이다. 노드의 SSM Agent가 AWS 쪽으로 아웃바운드 연결(`ssmmessages`)을 맺고, 사용자는 콘솔이나 AWS CLI로 그 채널에 붙는다. 그래서 보안 그룹에 22/3389 인바운드를 열 필요가 없다.

> "secure node management without the need to open inbound ports, maintain bastion hosts, or manage SSH keys" — [AWS Docs: Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)

| 항목 | 내용 |
| --- | --- |
| 접근 제어 | IAM 정책(`ssm:StartSession` 등). 리소스 태그·문서 ARN으로 세분화 가능 |
| 감사 | CloudTrail에 API 호출 기록. 세션 로그는 S3 / CloudWatch Logs로 전송(KMS 암호화 선택) |
| 전송 보안 | 클라이언트↔노드 TLS 1.2, 요청은 SigV4 서명. KMS 키로 추가 암호화 가능 |
| 이벤트 | EventBridge·SNS로 세션 시작/종료 알림 가능 |
| 지원 형태 | 대화형 셸, 포트포워딩(노드 자신 / 원격 호스트), SSH 터널, VPC endpoint(PrivateLink) |
| 비용 | EC2 대상은 추가 요금 없음. 하이브리드·멀티클라우드 노드는 2026-09-30부터 세션당 $0.05 ([Pricing](https://aws.amazon.com/systems-manager/pricing/)) |

> 참고: AWS는 기존(legacy) Session Manager 콘솔 페이지를 2026-12-31에 지원 종료하고 통합 콘솔(unified console)로 대체한다고 공지했다. — [AWS Docs: Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)

## 사전 요구사항

| 대상 | 요구사항 |
| --- | --- |
| 노드 | SSM Agent 실행 중 + managed node로 등록(IAM·네트워크 조건은 [02-architecture.md](./02-architecture.md) 참고) |
| 로컬 PC | AWS CLI 1.16.12 이상 + **Session Manager plugin** 설치 ([설치 문서](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html)) |
| 플러그인 버전 | AWS는 1.2.764.0 이상으로 업데이트할 것을 안내(구버전 지원 종료 예정) |
| 포트포워딩 | SSM Agent 2.3.672.0 이상(노드 자신), 3.1.1374.0 이상(원격 호스트) |

```bash
# 플러그인 설치 확인
session-manager-plugin --version
```

## 사용법 / 예시

출처: [Start a session](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html)

### 셸 접속

```bash
aws ssm start-session --target i-0123456789abcdef0
```

기본적으로 Linux에서는 시스템이 만든 `ssm-user` 계정으로 접속하며, 이 계정은 sudoers에 추가된다. ([Run As](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-preferences-run-as.html))

### 포트포워딩 — 노드 자신의 포트

```bash
# 노드의 80 포트를 로컬 56789로
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --document-name AWS-StartPortForwardingSession \
  --parameters '{"portNumber":["80"],"localPortNumber":["56789"]}'
```

- `portNumber`: 노드 쪽 포트(생략 시 80)
- `localPortNumber`: 로컬 PC 쪽 포트

### 포트포워딩 — 원격 호스트(예: private RDS)

노드를 점프 호스트로 써서 VPC 내부의 다른 호스트에 접속한다. 원격 호스트는 managed node일 필요가 없다.

```bash
aws ssm start-session \
  --target i-0123456789abcdef0 \
  --document-name AWS-StartPortForwardingSessionToRemoteHost \
  --parameters '{"host":["mydb.xxxx.ap-northeast-2.rds.amazonaws.com"],"portNumber":["3306"],"localPortNumber":["3306"]}'
# 이후 로컬에서: mysql -h 127.0.0.1 -P 3306 -u admin -p
```

같은 문서로 ECS 태스크(`ecs:<cluster>_<task-id>_<runtime-id>`)도 대상으로 지정할 수 있다.

### SSH over Session Manager

SSH 클라이언트·scp 등을 그대로 쓰고 싶을 때 쓴다. 이때는 `ssm-user`가 아니라 **키를 소유한 OS 사용자**(예: `ec2-user`)로 접속한다. 플러그인 1.1.23.0 이상이 필요하다. ([SSH 연결 설정](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-enable-ssh-connections.html))

```
# ~/.ssh/config
Host i-* mi-*
    ProxyCommand sh -c "aws ssm start-session --target %h --document-name AWS-StartSSHSession --parameters 'portNumber=%p'"
    User ec2-user
```

```bash
ssh -i key.pem ec2-user@i-0123456789abcdef0
```

IAM에서 `ssm:StartSession`을 인스턴스 ARN **과** `arn:aws:ssm:*:*:document/AWS-StartSSHSession` 양쪽에 허용해야 한다. 반대로 SSH를 막으려면 이 문서 ARN에 Deny를 건다.

## 세션 설정(Session preferences)

계정·리전 단위 설정은 `SSM-SessionManagerRunShell` 문서(`documentType: Session`, `sessionType: Standard_Stream`)에 저장된다. ([CLI로 설정 만들기](https://docs.aws.amazon.com/systems-manager/latest/userguide/getting-started-create-preferences-cli.html), [스키마](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-schema.html))

| 입력 | 의미 | 기본/범위 |
| --- | --- | --- |
| `s3BucketName`, `s3KeyPrefix`, `s3EncryptionEnabled` | 세션 로그 S3 저장 | — |
| `cloudWatchLogGroupName`, `cloudWatchEncryptionEnabled` | 세션 로그 CloudWatch Logs 저장 | — |
| `cloudWatchStreamingEnabled` | `true`: 실시간 스트리밍 / `false`: 세션 종료 시 업로드 | — |
| `kmsKeyId` | 클라이언트↔노드 데이터 KMS 암호화 | — |
| `runAsEnabled`, `runAsDefaultUser` | 접속 OS 사용자 지정(Linux/macOS만, `root` 불가) | 기본 `ssm-user` |
| `idleSessionTimeout` | 유휴 세션 종료(분) | 기본 20, 1–60 ([Quotas](https://docs.aws.amazon.com/general/latest/gr/ssm.html#session-manager)) |
| `maxSessionDuration` | 최대 세션 시간(분) | 기본 제한 없음, 1–1440 ([문서](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-preferences-max-timeout.html)) |
| `shellProfile` | 세션 시작 시 실행할 명령(`linux`/`windows` 키) | — |

예시 설정 파일:

```json
{
  "schemaVersion": "1.0",
  "description": "Session Manager preferences",
  "sessionType": "Standard_Stream",
  "inputs": {
    "s3BucketName": "my-session-logs",
    "s3KeyPrefix": "ssm/",
    "s3EncryptionEnabled": true,
    "cloudWatchLogGroupName": "/ssm/sessions",
    "cloudWatchEncryptionEnabled": true,
    "cloudWatchStreamingEnabled": true,
    "kmsKeyId": "alias/session-manager",
    "runAsEnabled": false,
    "runAsDefaultUser": "",
    "idleSessionTimeout": "20",
    "maxSessionDuration": "120",
    "shellProfile": { "linux": "cd ~ && bash", "windows": "" }
  }
}
```

```bash
# 처음 만들 때(이미 있으면 update-document --document-version '$LATEST')
aws ssm create-document \
  --name SSM-SessionManagerRunShell \
  --content file://SessionManagerRunShell.json \
  --document-type Session \
  --document-format JSON
```

Run As 사용자 결정 순서: IAM 사용자/역할의 `SSMSessionRunAs` 태그 → 세션 설정의 `runAsDefaultUser`. 지정한 OS 사용자로 접속에 실패해도 `ssm-user`로 대체되지 않는다. ([Run As](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-preferences-run-as.html))

## IAM 권한 제어 예시

출처: [Restrict access examples](https://docs.aws.amazon.com/systems-manager/latest/userguide/getting-started-restrict-access-examples.html)

### 태그가 붙은 인스턴스에만 접속 허용

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "ssm:StartSession",
      "Resource": "arn:aws:ec2:ap-northeast-2:111122223333:instance/*",
      "Condition": {
        "StringLike": { "ssm:resourceTag/Team": ["web"] }
      }
    },
    {
      "Effect": "Allow",
      "Action": "ssm:StartSession",
      "Resource": "arn:aws:ssm:ap-northeast-2:111122223333:document/SSM-SessionManagerRunShell"
    },
    {
      "Effect": "Allow",
      "Action": ["ssm:TerminateSession", "ssm:ResumeSession"],
      "Resource": "arn:aws:ssm:*:*:session/${aws:userid}-*"
    }
  ]
}
```

- 자기 세션만 종료하게 하려면 세션 ARN을 `${aws:username}-*`로 제한한다. 단, AWS 문서는 **federated ID에서는 이 방식이 동작하지 않는다**고 명시한다. 이 경우 AWS가 붙이는 세션 태그 `ssm:resourceTag/aws:ssmmessages:session-id`에 `${aws:userid}*` 조건을 쓴다.
- 위 예시는 AWS 문서 예시를 단순화한 것이다. 원문 예시에는 `ssmmessages:OpenDataChannel` 문도 포함되어 있으므로 실제 적용 시 원문과 대조할 것.

## 이슈 / 안티패턴 / 주의

- **포트포워딩·SSH 세션은 로그가 남지 않는다.**
  > "Logging isn't available for Session Manager sessions that connect through port forwarding or SSH." — [Session logging](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-logging.html)

  감사가 필요한 환경이면 `AWS-StartSSHSession` 문서를 IAM으로 Deny하고 셸 세션만 허용하는 방식을 고려한다.
- Linux 노드에서 세션 로깅을 쓸 때 `screen` 패키지가 없으면 로그가 잘릴 수 있다. (같은 문서)
- 로그에는 화면에 출력된 내용이 그대로 남는다. 비밀번호 입력은 `stty -echo; read passwd; stty echo;` 방식이 권장된다. (같은 문서)
- `ssm:StartSession`을 `Resource: "*"`로 주면 계정 내 모든 managed node에 접속할 수 있다. 태그 조건으로 좁힌다.
- 기본 `ssm-user`는 sudo 권한이 있다. 최소 권한이 필요하면 Run As로 일반 OS 사용자를 지정한다.
- API 한도: `StartSession` 3 TPS, `TerminateSession`/`ResumeSession`/`DescribeSessions` 각 6 TPS. 세션 히스토리는 30일 보관. ([Quotas](https://docs.aws.amazon.com/general/latest/gr/ssm.html#session-manager))

## 출처

- https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-sessions-start.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started-enable-ssh-connections.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/getting-started-create-preferences-cli.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-schema.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/session-preferences-run-as.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/session-preferences-max-timeout.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/getting-started-restrict-access-examples.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-logging.html
- https://docs.aws.amazon.com/general/latest/gr/ssm.html#session-manager
- https://aws.amazon.com/systems-manager/pricing/

---
*조사 기준일: 2026-10-12. 모든 값은 위 AWS 공식 문서 기준이다. 하이브리드 노드 세션 과금(2026-09-30 시작)은 최근 변경 사항이므로 적용 전에 요금 페이지를 다시 확인할 것.*
