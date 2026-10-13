# 동작 원리·아키텍처

> 노드의 **SSM Agent가 AWS 쪽으로 아웃바운드 HTTPS 연결을 맺고**, 그 채널로 명령·세션을 받는 pull 구조다. 그래서 인바운드 포트가 필요 없다. 대신 **IAM 권한**(instance profile 또는 DHMC)과 **엔드포인트 연결**(`ssm`, `ssmmessages`, 구형 리전은 `ec2messages`)이 갖춰져야 노드가 managed node로 보인다. 모든 작업은 **SSM Document**로 정의된다.

## 전체 구조

```
 사용자 (콘솔 / AWS CLI + Session Manager plugin / SDK)
        │  IAM 인증 (ssm:SendCommand, ssm:StartSession ...)
        ▼
 ┌──────────────────────── AWS Systems Manager ───────────────────────┐
 │  ssm  (서비스 API)    ssmmessages (메시지·세션 채널)    ec2messages (레거시) │
 └────────────────────────────────────────────────────────────────────┘
        ▲  아웃바운드 HTTPS 443 (agent가 먼저 연결)
        │  · 인터넷 / NAT  또는  · VPC interface endpoint
 ┌──────┴──────────────┐
 │ managed node        │  SSM Agent (root/SYSTEM 권한으로 실행)
 │  i-... / mi-...     │  자격증명: instance profile / DHMC / hybrid service role
 └─────────────────────┘
        │ 결과·로그
        ▼
 S3 / CloudWatch Logs / KMS (선택)
```

## SSM Agent

| 항목 | 내용 | 출처 |
| --- | --- | --- |
| 구현 | 오픈소스(Go, Apache 2.0) | [GitHub](https://github.com/aws/amazon-ssm-agent) |
| 하트비트 | 5분마다 `UpdateInstanceInformation` 호출 | [Message APIs](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-setting-up-messageAPIs.html) |
| 기본 설치 AMI | Amazon Linux 2 / AL2023, ECS·EKS 최적화 AMI, Ubuntu 18.04–25.04, Windows Server 2016–2025, SLES 15.3+, macOS 13–15 등 | [Preinstalled agent](https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html) |
| 주의 | 기본 설치된 agent가 최신 버전은 아닐 수 있음 | 같은 출처 |

## 엔드포인트

출처: [Message APIs](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-setting-up-messageAPIs.html)

| 엔드포인트 | 이름 | 용도 |
| --- | --- | --- |
| `ssm` | Systems Manager API | 노드 등록·하트비트(`UpdateInstanceInformation`), association 조회 등 |
| `ssmmessages` | Amazon Message Gateway Service | Session Manager 제어·데이터 채널, Run Command 메시지. agent 3.3.40.0부터는 가능하면 ec2messages 대신 이쪽을 씀 |
| `ec2messages` | Amazon Message Delivery Service | 레거시 메시지 채널(`GetMessages`, `SendReply`). **2024년 이전에 출시된 리전에서만 지원** |

- AWS는 ec2messages를 지원하는 리전에서도 `ec2messages:*` 권한을 "can safely remove"라고 안내한다. 단, 구버전 agent는 여전히 ec2messages를 쓸 수 있으므로 agent 버전을 먼저 확인한다(확인 필요: 버전별 동작 차이).
- `ssmmessages:OpenControlChannel` 권한을 빼면 최대 1시간 안에 노드 연결이 끊긴다.

### Private 서브넷: VPC endpoint

NAT 없이 private 서브넷에서 쓰려면 interface endpoint를 만든다. ([Create VPC endpoints](https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-create-vpc.html))

| 엔드포인트 | 필요 여부 | 이유 |
| --- | --- | --- |
| `com.amazonaws.<region>.ssm` | 필수 | 서비스 API |
| `com.amazonaws.<region>.ssmmessages` | 필수 | Session Manager, Run Command |
| `com.amazonaws.<region>.ec2messages` | 구형 리전에서 필요 | 레거시 메시지 채널 |
| `com.amazonaws.<region>.s3` | 상황에 따라 | agent 업데이트, 패치, 스크립트·출력 로그 |
| `com.amazonaws.<region>.kms` | 선택 | SecureString, Session Manager KMS 암호화 |
| `com.amazonaws.<region>.logs` | 선택 | CloudWatch Logs로 로그 전송 |
| `com.amazonaws.<region>.ec2` | 선택 | VSS 스냅샷 |

- endpoint의 보안 그룹은 private 서브넷에서 들어오는 **443 인바운드**를 허용해야 한다.
- S3를 gateway endpoint로 둘지 interface endpoint로 둘지는 위 문서에 명시가 없다(확인 필요). 일반적으로는 gateway endpoint로도 충분하다.

## IAM: 노드가 SSM에 말할 권한

출처: [Instance permissions](https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-instance-permissions.html), [DHMC](https://docs.aws.amazon.com/systems-manager/latest/userguide/managed-instances-default-host-management.html)

| 방식 | 설정 단위 | 핵심 | 요구사항 |
| --- | --- | --- | --- |
| **Default Host Management Configuration (DHMC)** | 계정·리전 | 모든 EC2에 기본 역할(`AWSSystemsManagerDefaultEC2InstanceManagementRole` + `AmazonSSMManagedEC2InstanceDefaultPolicy`)을 자동 적용 | **IMDSv2 필수**(v1 미지원), SSM Agent 3.2.582.0 이상, 적용까지 최대 30분 |
| **Instance profile** | 인스턴스별 | `AmazonSSMManagedInstanceCore` 정책을 붙인 역할 | — |
| **Service role** (하이브리드) | activation별 | `ssm.amazonaws.com`을 신뢰하는 역할 | hybrid activation |

- instance profile이 `ssm:UpdateInstanceInformation`을 허용하면 DHMC 대신 instance profile이 우선 쓰인다.
- 로그를 S3/CloudWatch에 쓰거나 도메인에 조인하는 등 부가 작업에는 추가 정책이 필요하다(`CloudWatchAgentServerPolicy`, `AmazonSSMDirectoryServiceAccess`, S3 버킷 정책 등).
- 이 권한은 **노드 쪽** 권한이다. 사람이 접속하거나 명령을 보낼 권한(`ssm:StartSession`, `ssm:SendCommand`)은 사용자 IAM 정책에서 따로 준다.

## SSM Document

모든 도구는 "무엇을 할지"를 SSM Document로 정의한다. JSON 또는 YAML로 작성하고, 버전과 default version이 있으며 다른 계정과 공유할 수 있다. AWS가 제공하는 문서는 `AWS-` 접두사로 시작한다. ([Documents](https://docs.aws.amazon.com/systems-manager/latest/userguide/documents.html), [Schemas](https://docs.aws.amazon.com/systems-manager/latest/userguide/documents-schemas-features.html))

| Document type | 쓰는 도구 | schemaVersion | 예 |
| --- | --- | --- | --- |
| Command | Run Command, State Manager, Maintenance Windows | 1.2 / 2.0 / **2.2(권장)** | `AWS-RunShellScript` |
| Automation (runbook) | Automation, State Manager, Maintenance Windows | **0.3** | `AWS-RestartEC2Instance` |
| Session | Session Manager | **1.0** | `AWS-StartPortForwardingSession`, `SSM-SessionManagerRunShell` |
| Policy | Inventory | 2.0+ | `AWS-GatherSoftwareInventory` |
| Package | Distributor | — | — |
| ChangeCalendar, ApplicationConfiguration 등 | Change Calendar, AppConfig 등 | — | — |

- 문서 하나의 최대 크기는 64 KB이고, 계정당 500개, 문서당 1,000개 버전까지 만들 수 있다. ([Quotas](https://docs.aws.amazon.com/general/latest/gr/ssm.html))
- 2025-07-14부터 schema 2.2는 `interpolationType: ENV_VAR`를 지원한다. 파라미터를 문자열로 치환하지 않고 `SSM_<name>` 환경변수로 넘겨 **command injection을 막는다**. SSM Agent 3.3.2746.0 이상이 필요하다. ([Schemas](https://docs.aws.amazon.com/systems-manager/latest/userguide/documents-schemas-features.html))

## 하이브리드·멀티클라우드 노드

출처: [Hybrid activation](https://docs.aws.amazon.com/systems-manager/latest/userguide/hybrid-activation-managed-nodes.html), [Activations](https://docs.aws.amazon.com/systems-manager/latest/userguide/activations.html)

1. service role을 지정해 activation을 만들면 **Activation Code + Activation ID**가 나온다. AWS는 이를 EC2의 access key ID / secret key처럼 다루라고 안내한다.
2. 대상 서버에 SSM Agent를 설치하고 코드·ID로 등록하면 `mi-` ID를 받는다.
3. 이후에는 EC2 노드와 똑같이 Session Manager, Run Command 등을 쓸 수 있다.

```bash
aws ssm create-activation \
  --default-instance-name onprem-web \
  --iam-role service-role/AmazonEC2RunCommandRoleForManagedInstances \
  --registration-limit 10 \
  --expiration-date 2026-10-19T00:00:00Z
```

| 항목 | 값 |
| --- | --- |
| 등록 한도(registration limit) 기본값 | 1 |
| 만료 | 기본 24시간, 최대 30일. 만료되어도 이미 등록된 노드에는 영향 없음 |
| 재등록·VM 복제 | 새 코드·ID가 필요(복제한 VM은 ID가 충돌함) |
| Instance tier | 2026-06-30 advanced-instances tier 폐지 → 1,000대 제한 해제, 세션·명령 사용량 과금으로 전환 |

## 출처

- https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-setting-up-messageAPIs.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-create-vpc.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/setup-instance-permissions.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/managed-instances-default-host-management.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/ami-preinstalled-agent.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/documents.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/documents-schemas-features.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/hybrid-activation-managed-nodes.html
- https://docs.aws.amazon.com/systems-manager/latest/userguide/activations.html
- https://docs.aws.amazon.com/general/latest/gr/ssm.html
- https://github.com/aws/amazon-ssm-agent

---
*조사 기준일: 2026-10-12. 위 다이어그램은 공식 문서 설명을 바탕으로 재구성한 개념도다.*
