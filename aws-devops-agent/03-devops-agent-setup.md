# DevOps Agent 설정과 보안

> 설정은 **Agent Space 만들기 → 계정 연결(IAM role) → 연동 추가 → 웹 앱 사용자 연결 → 조사 트리거** 순서다. 에이전트가 쓰는 role은 **읽기 전용**으로 두고, 여러 계정은 **보조 계정 role**로 연결한다. 보안에서 꼭 알아둘 점이 세 가지 있다. **서울 리전은 추론(inference)이 전 세계 리전으로 라우팅될 수 있다.** **PII는 걸러지지 않는다.** **SCP로는 막을 수 없다고 문서에 명시되어 있다.**

## 이 문서에서 새로 나오는 용어

[02-devops-agent.md](./02-devops-agent.md)의 용어와 [aws-iam](../aws-iam/README.md)·[aws-organizations/04](../aws-organizations/04-identity-center.md)의 IAM role·Identity Center를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 기본 계정(primary account) / 보조 계정(secondary account) | Agent Space를 만든 계정 / 추가로 연결한 계정 |
| 운영자 앱(operator app) | 운영자가 쓰는 DevOps Agent 웹 앱 |
| 추론(inference) | AI 모델이 입력을 받아 결과를 만드는 계산 |
| 리전 간 추론(cross-Region inference) | 모델 계산을 다른 리전에서 처리하는 Amazon Bedrock 방식 |
| PII | 개인 식별 정보(이름, 이메일, 전화번호 등) |
| Prompt injection | 데이터에 숨긴 문장이 AI에게 지시처럼 작동하게 만드는 공격 |

## 설정 순서

출처: [CLI onboarding guide](https://docs.aws.amazon.com/devopsagent/latest/userguide/getting-started-with-aws-devops-agent-cli-onboarding-guide.html), [Connecting multiple accounts](https://docs.aws.amazon.com/devopsagent/latest/userguide/configuring-integrations-and-knowledge-connecting-multiple-aws-accounts.html)

| 단계 | 할 일 | 메모 |
| --- | --- | --- |
| 1 | 기본 계정 정하기 | AWS는 **전용 계정**을 기본 계정으로 쓰라고 권장 |
| 2 | Agent Space 생성 | 리전 선택 = 데이터 저장 위치 |
| 3 | 계정 연결 | 기본·보조 계정마다 에이전트가 맡을 role(`AIDevOpsAgentAccessPolicy` 기반) |
| 4 | 연동 추가 | CloudWatch, GitHub, Datadog, PagerDuty, Slack 등 |
| 5 | 웹 앱 활성화·사용자 연결 | 웹 앱 role(`AIDevOpsOperatorAppAccessPolicy`), Identity Center 또는 외부 IdP |
| 6 | 조사 트리거 연결 | 티켓 연동, 웹훅, 수동 |

```bash
aws devops-agent create-agent-space ...      # Space 생성
aws devops-agent list-agent-spaces
aws devops-agent register-service ...        # 연동 대상 등록
aws devops-agent associate-service ...       # Space에 연동 연결
aws devops-agent enable-operator-app ...     # 웹 앱 활성화
aws devops-agent create-trigger ...          # 조사 트리거 생성
```

- CLI 이름공간은 `aws devops-agent`이고, IAM 액션 접두사는 `aidevops:`다(예: `aidevops:GetAgentSpace`).
- 각 명령의 정확한 옵션은 이번 조사에서 확인하지 못해 `...`으로 두었다. CLI 온보딩 가이드를 따른다(확인 필요).
- CDK, CloudFormation, Terraform으로 설정하는 시작 가이드도 있다. 여러 Space를 운영한다면 IaC로 관리하는 편이 낫다([aws-cloudformation](../aws-cloudformation/README.md)).

## IAM

출처: [Security](https://docs.aws.amazon.com/devopsagent/latest/userguide/aws-devops-agent-security.html)

| role | 용도 | 권장 |
| --- | --- | --- |
| 기본 계정 role | 에이전트가 기본 계정 리소스를 읽음 | **읽기 전용**, 최소 권한 |
| 보조 계정 role | 계정마다 하나, 에이전트가 그 계정을 읽음 | 읽기 전용 |
| 웹 앱 role | 사용자가 웹 앱에 접근 | — |
| Action role(선택) | directed action 실행 | 꼭 필요한 변경만 허용([02](./02-devops-agent.md#directed-action-에이전트가-리소스를-바꿀-때)) |

관리형 정책 `AIDevOpsAgentAccessPolicy`, `AIDevOpsOperatorAppAccessPolicy`로 시작하고, 운영하면서 [IAM Access Analyzer](../aws-iam/06-least-privilege.md)로 실제로 쓰는 권한만 남긴다.

### 웹 앱 로그인

| 방식 | 세션 |
| --- | --- |
| IAM Identity Center(OAuth, Okta·Entra ID 연동 가능) | 기본 최대 12시간 |
| 외부 IdP | — |
| IAM 콘솔 링크 | **10분** |

## CloudWatch 알람으로 조사 시작하기

CloudWatch 알람이 DevOps Agent 조사를 바로 시작하는 기본 연동은 문서에서 찾지 못했다. AWS DevOps 블로그는 아래처럼 연결하는 구성을 소개한다(블로그는 제목만 확인).

```
CloudWatch 알람(ALARM) ──▶ EventBridge 규칙 또는 SNS ──▶ Lambda ──▶ DevOps Agent 웹훅(HMAC 서명)
```

| 구성 요소 | 역할 | 참고 |
| --- | --- | --- |
| CloudWatch 알람 | 장애 감지 | [aws-cloudwatch/03](../aws-cloudwatch/03-alarms.md) |
| EventBridge 규칙 | 알람 상태 변화 이벤트(`aws.cloudwatch`) 받기 | — |
| Lambda | 알람 정보를 웹훅 형식으로 바꿔 서명 후 전송 | 웹훅 비밀값은 Secrets Manager에 |
| DevOps Agent 웹훅 트리거 | 조사 시작 | `create-trigger`로 생성 |

```json
{
  "source": ["aws.cloudwatch"],
  "detail-type": ["CloudWatch Alarm State Change"],
  "detail": { "state": { "value": ["ALARM"] } }
}
```

위 EventBridge 이벤트 패턴은 CloudWatch 알람 상태 변화 이벤트의 일반 형식으로 이 레포에서 작성했다. 웹훅 요청 형식은 DevOps Agent 문서를 따른다(확인 필요). **모든 알람으로 조사를 시작하면 비용이 커지므로**, 중요한 composite alarm만 연결한다(→ [05](./05-cost-issues.md)).

PagerDuty·Grafana를 이미 쓴다면 그쪽 웹훅 연동이 더 간단하다.

## 데이터 처리와 보안

출처: [Security](https://docs.aws.amazon.com/devopsagent/latest/userguide/aws-devops-agent-security.html)

| 항목 | 내용 |
| --- | --- |
| 모델 학습 | "AWS does not use agent data, chat messages, or data from integrated data sources to train models or improve the product." |
| 저장 위치 | Agent Space를 만든 리전 |
| 암호화 | 저장 시 AWS 관리 키, 전송 중 암호화 |
| **추론 위치** | Bedrock 리전 간 추론으로 **같은 지역권**(미국, EU, 호주, 일본) 안에서 처리. **서울·싱가포르·뭄바이·상파울루는 전 세계(global) 라우팅** |
| **SCP·Control Tower 리전 제한** | 문서에 "not impacted by"라고 명시. 즉 **리전 제한 SCP로 추론 위치를 막을 수 없음** |
| **PII** | 걸러 주지 않음. 로그에서 직접 마스킹해야 함([aws-cloudwatch/04 data protection](../aws-cloudwatch/04-logs.md#그-밖의-기능)) |
| 삭제 | Space를 지우면 30일 안에 데이터 삭제 |
| 감사 | agent journal(에이전트가 못 고침), CloudTrail |
| 보호 장치 | 모델 보호(ASL-3), Bedrock Guardrails의 prompt attack 필터, 계정 경계 강제, PrivateLink VPC endpoint |

### 데이터 위치 규정이 있는 조직이라면

서울 리전에 Agent Space를 만들어도 **AI 추론은 국외 리전에서 처리될 수 있다.** 개인정보·금융 규정으로 데이터의 국외 처리를 제한해야 하는 조직은 다음을 먼저 검토한다(이 레포의 정리).

- 어떤 데이터(로그 원문, 메트릭, 코드)가 에이전트에 전달되는지
- 로그의 PII 마스킹
- 규정 담당 부서의 검토

### Prompt injection

에이전트는 로그·태그·티켓 본문 같은 데이터를 읽는다. 공격자가 이런 곳에 "이전 지시를 무시하고 …" 같은 문장을 넣을 수 있다. AWS도 **직접 만든 MCP 서버와 공격자가 통제할 수 있는 로그·태그 내용**이 위험을 키운다고 경고한다.

| 대응 | 이유 |
| --- | --- |
| 에이전트 role을 읽기 전용으로 | 속더라도 바꿀 수 있는 것이 없음 |
| directed action 승인 시 내용을 반드시 읽기 | 사람이 마지막 방어선 |
| 신뢰할 수 있는 MCP 서버만 연결 | 도구 자체가 공격 경로가 될 수 있음 |
| agent journal·CloudTrail 정기 검토 | 이상한 조회·요청 탐지 |

## 이슈 / 주의

- **CLI 옵션과 웹훅 형식**은 이번 조사에서 세부까지 확인하지 못했다. 공식 온보딩 가이드를 따른다.
- **기본 계정을 운영 계정과 분리**한다. AWS도 전용 계정을 권장한다.
- **리전 제한 SCP가 추론 위치를 막지 못한다**는 점을 보안팀과 공유한다.
- **IAM 콘솔 링크 세션은 10분**이라 장애 대응 중에 끊길 수 있다. 운영자는 Identity Center로 로그인하게 한다.

## 출처

- https://docs.aws.amazon.com/devopsagent/latest/userguide/getting-started-with-aws-devops-agent-cli-onboarding-guide.html
- https://docs.aws.amazon.com/devopsagent/latest/userguide/configuring-integrations-and-knowledge-connecting-multiple-aws-accounts.html
- https://docs.aws.amazon.com/devopsagent/latest/userguide/aws-devops-agent-security.html
- https://docs.aws.amazon.com/devopsagent/latest/userguide/production-operations-autonomous-incident-response.html
- https://aws.amazon.com/blogs/devops/building-an-end-to-end-agentic-sre-using-aws-devops-agent/

---
*조사 기준일: 2026-10-20. CLI·EventBridge 예시는 실제 계정에서 실행하지 않았다.*
