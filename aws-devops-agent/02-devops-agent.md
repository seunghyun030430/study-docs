# AWS DevOps Agent

> DevOps Agent는 **Agent Space**(보안 경계) 단위로 동작한다. 알림·티켓·웹훅이 들어오면 **조사(investigation)**를 시작해 연결된 관측 도구·코드 저장소·배포 기록을 스스로 뒤져 원인을 찾고, 검증·되돌리기 단계가 포함된 **완화 계획**을 낸다. 과거 장애로 **예방 개선안**도 제안한다. 기본은 **읽기 전용**이고, 리소스를 바꾸는 **directed action**은 세 단계 opt-in과 매번의 운영자 승인이 있어야 한다. 릴리스 관리 기능은 아직 미리 보기다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)의 용어(Agent Space, topology, investigation, evaluation, directed action)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 웹훅(webhook) | 이벤트가 생기면 지정한 URL로 HTTP 요청을 보내는 방식 |
| MCP(Model Context Protocol) | AI 에이전트가 외부 도구·데이터에 접근하는 표준 프로토콜 |
| A2A(Agent-to-Agent) | 에이전트끼리 작업을 주고받는 프로토콜 |
| Skill | 에이전트가 특정 작업을 하는 방법을 적어 둔 지침·절차 |
| Memory | 에이전트가 이전 조사에서 배운 내용을 저장해 다음에 쓰는 기억 |
| Custom agent | 사용자가 시스템 프롬프트·skill을 정해 만든 전용 에이전트 |
| Agent journal | 에이전트가 무엇을 했는지 남기는, 에이전트 자신은 수정할 수 없는 기록 |
| Kiro | AWS의 AI 코딩 에이전트·IDE. DevOps Agent의 지침을 받아 코드 수정 등을 수행할 수 있음 |

## 무엇을 하나

출처: [About AWS DevOps Agent](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent.html), [GA blog](https://aws.amazon.com/blogs/mt/announcing-general-availability-of-aws-devops-agent/)

| 영역 | 기능 | 상태 |
| --- | --- | --- |
| **장애 조사** | 알림·티켓이 오면 텔레메트리·코드·배포 데이터를 엮어 근본 원인 분석 | 정식 |
| **완화 계획** | 검증·되돌리기 단계를 포함한 계획. 다른 에이전트(Kiro 등)가 실행할 수 있는 지침도 생성 | 정식 |
| **예방 평가** | 과거 장애를 분석해 관측성·인프라·파이프라인·복원력 개선 제안 | 정식 |
| **장애 협업** | Slack, ServiceNow, PagerDuty로 진행 상황 공유. Microsoft Teams는 단방향 알림(2026-10-16 추가) | 정식 |
| **SRE 작업 채팅** | 자연어로 "지난주 에러율 추이 보여 줘" 같은 작업 요청 | 정식 |
| **Support case** | 조사 결과로 AWS Support 케이스 생성 | 정식 |
| 릴리스 관리 | 릴리스 준비 상태 코드 리뷰, 자동 릴리스 테스트 | **미리 보기**(us-east-1) |

## Agent Space

출처: [What are Agent Spaces](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-what-are-devops-agent-spaces.html), [Supported regions](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-supported-regions.html)

```
Agent Space "payments-prod"  (ap-northeast-2에 생성 → 데이터는 서울에 저장)
├── AWS 계정: 기본 계정 + 보조 계정(payments-prod, shared-network)
│     └─ 모든 리전의 리소스를 볼 수 있음(리전마다 Space를 만들 필요 없음)
├── 연동: CloudWatch, Datadog, GitHub(payments-api), PagerDuty, Slack
├── 사용자: 결제팀 온콜 그룹(웹 앱 접근)
└── Skill·Memory·Custom agent·지침(AGENTS.md 스타일)
```

| 성질 | 내용 |
| --- | --- |
| 보안 경계 | 에이전트가 접근할 계정·연동·사용자를 Space마다 정함. 데이터도 Space별로 분리 |
| 데이터 위치 | Space를 만든 리전에 저장 |
| 감시 범위 | 연결된 계정의 **모든 리전** 리소스 |
| 콘솔 | 관리자는 AWS 콘솔에서 Space를 설정, 운영자는 Space마다 있는 **웹 앱**에서 조사를 봄 |
| 지원 리전 | 15개(서울 포함, 2026-10-20 기준) |

Space는 보통 **팀이나 서비스 단위**로 나눈다. 한 Space에 너무 많은 계정을 넣으면 접근 범위가 넓어지고, 너무 잘게 나누면 서비스 사이를 오가는 장애를 한눈에 보기 어렵다(이 레포의 정리).

## 연동

출처: [About AWS DevOps Agent](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent.html)

| 분류 | 연동 대상 |
| --- | --- |
| AWS | Amazon CloudWatch, AWS X-Ray(미리 보기 발표 기준), Amazon EKS |
| 관측 도구 | Datadog, Dynatrace, New Relic, Splunk, Grafana |
| 코드·CI/CD | GitHub, GitLab, Azure DevOps |
| 티켓·채팅 | ServiceNow, PagerDuty, Slack, Microsoft Teams(단방향) |
| 다른 클라우드 | Azure 리소스 |
| 확장 | 사설·원격 **MCP 서버**, 원격 **A2A 에이전트**, Amazon EventBridge 이벤트 |

CloudTrail은 문서에서 에이전트 자신의 API 호출을 감사하는 용도로 나온다. 조사 데이터로 CloudTrail 이벤트를 읽는지는 확인하지 못했다(확인 필요).

## 조사는 어떻게 시작되고 끝나나

출처: [Autonomous incident response](https://docs.aws.amazon.com/devopsagent/latest/userguide/production-operations-autonomous-incident-response.html)

| 시작 방법 | 예 |
| --- | --- |
| 티켓 연동 | ServiceNow에 장애 티켓이 생성되면 자동 시작 |
| **웹훅** | PagerDuty·Grafana 알림을 웹훅으로 받음(HMAC 서명 또는 bearer 토큰으로 인증) |
| 수동 | 웹 앱에서 "Latest alarm", "High CPU usage", "Error rate spike" 같은 프리셋으로 시작 |
| 일정·이벤트 | custom agent를 일정이나 웹훅 이벤트로 실행(2026-09-14 추가) |

**CloudWatch 알람이 바로 조사를 시작하는 기본 연동은 문서에 없다.** AWS 블로그는 CloudWatch 알람 → EventBridge/SNS → Lambda → 웹훅으로 연결하는 구성을 소개한다(→ [03](./03-devops-agent-setup.md#cloudwatch-알람으로-조사-시작하기)).

| 조사 중·후 | 내용 |
| --- | --- |
| 사람의 개입 | 진행 중인 조사에 채팅으로 질문, 조사를 분기(fork), 피드백, 사람 지원 요청 |
| 결과 전달 | 원래 티켓에 업데이트, Slack 장애 채널 업데이트, Teams 카드, 웹 앱의 요약·타임라인 탭, EventBridge 이벤트, Markdown 내보내기 |
| 학습 | 피드백은 그 Space 안에서만 응답 개선에 쓰임. **AWS는 고객 데이터로 모델을 학습하지 않는다고 명시** |

## Directed action: 에이전트가 리소스를 바꿀 때

출처: [Directed actions](https://docs.aws.amazon.com/devopsagent/latest/userguide/working-with-devops-agent-working-with-directed-actions.html), [Security](https://docs.aws.amazon.com/devopsagent/latest/userguide/aws-devops-agent-security.html)

기본은 **읽기 전용**이다. 2026-08-25부터 운영자가 명시적으로 요청한 변경 작업(directed action)을 실행할 수 있다.

| 조건 | 내용 |
| --- | --- |
| 기본값 | **꺼짐** |
| 켜려면(세 단계) | ① Agent Space에서 기능 활성화 ② 계정마다 권한이 높은 IAM role 등록(`AIDevOpsAgentActionsPolicy` 또는 직접 만든 정책) ③ 도구마다 분류 지정 |
| 실행 | **매번 운영자 승인** 후, 그 작업에만 범위를 좁힌 자격증명으로 실행 |
| 감사 | CloudTrail에 **승인한 운영자 이름으로** 기록 |
| 금지 목록 | 에이전트가 절대 하지 않는 작업 목록이 별도로 있음 |

보안 문서는 아직 "도구는 리소스를 바꿀 수 없다(티켓·Support 케이스 생성 제외)"고 적혀 있어 directed action 문서와 내용이 다르다. directed action 기능이 나중에 추가되면서 생긴 차이로 보인다.

**권장 사항(이 레포의 정리)**:

- 처음에는 directed action을 끄고 **조사·제안만** 쓴다.
- 켤 때는 action role을 [최소 권한](../aws-iam/06-least-privilege.md)으로 만든다(예: 특정 ASG 크기 조정, 특정 배포 롤백만).
- 승인자를 온콜 담당자로 한정한다.

## 학습과 맞춤화

| 기능 | 내용 |
| --- | --- |
| Skill / learned skill | 조사 방법을 적은 지침. 에이전트가 경험에서 skill을 배우기도 함 |
| Memory | 이전 조사 내용을 저장(Space당 메모리 저장소 50개, 저장소당 200개, 하나당 100 KB) |
| Agent 지침 | AGENTS.md 스타일로 팀의 운영 규칙·서비스 설명을 제공 |
| Custom agent | 시스템 프롬프트(최대 5만 자)와 skill로 만든 전용 에이전트. 일정·웹훅으로 실행 가능 |
| Managed agent | AWS가 제공하는 미리 만든 에이전트 |

"이 서비스는 배포 직후 캐시가 비어 지연이 늘 수 있다" 같은 팀 지식을 지침으로 넣어 두면 조사 품질이 좋아진다. 세부 동작은 이번 조사에서 깊게 확인하지 못했다.

## 이슈 / 주의

- **조사 품질은 데이터에 달려 있다.** 로그·트레이스·배포 기록이 연결되지 않으면 AI도 근거 없이 추측한다.
- **prompt injection 위험**: 공격자가 로그·태그에 넣은 문자열이 에이전트에게 지시처럼 읽힐 수 있다. AWS도 직접 만든 MCP 서버와 공격자가 통제할 수 있는 로그·태그 내용이 위험을 키운다고 경고한다(→ [03](./03-devops-agent-setup.md#데이터-처리와-보안)).
- **directed action은 신중히** 켠다. 승인 버튼을 습관적으로 누르는 순간 사람의 검토가 사라진다.
- **릴리스 관리·Sandbox 기능은 미리 보기**이고 일부 리전에서만 동작한다.

## 출처

- https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent.html
- https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-what-are-devops-agent-spaces.html
- https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-supported-regions.html
- https://docs.aws.amazon.com/devopsagent/latest/userguide/production-operations-autonomous-incident-response.html
- https://docs.aws.amazon.com/devopsagent/latest/userguide/working-with-devops-agent-working-with-directed-actions.html
- https://docs.aws.amazon.com/devopsagent/latest/userguide/aws-devops-agent-security.html
- https://docs.aws.amazon.com/devopsagent/latest/userguide/whats-new.html
- https://docs.aws.amazon.com/devopsagent/latest/userguide/quotas.html
- https://aws.amazon.com/blogs/mt/announcing-general-availability-of-aws-devops-agent/

---
*조사 기준일: 2026-10-20. 연동 목록과 기능 상태는 What's new 페이지에서 자주 바뀐다.*
