# AWS DevOps Agent·AI 운영 도구 정리

AWS의 AI 기반 운영 도구를 정리한 문서 모음. 중심은 **AWS DevOps Agent**이고, 같은 일을 하는 **CloudWatch Investigations**와 지원 종료 예정인 **Amazon DevOps Guru**(이전 관점)를 함께 다룬다. 조사 기준일 **2026-10-20**.

관련 주제: [aws-cloudwatch](../aws-cloudwatch/README.md)(알람·로그·이상 탐지), [aws-iam](../aws-iam/README.md)(role·최소 권한), [aws-organizations](../aws-organizations/README.md)(Identity Center), [aws-systems-manager](../aws-systems-manager/README.md)(runbook).

## 한눈에 보기

| 항목 | AWS DevOps Agent | CloudWatch Investigations | Amazon DevOps Guru |
| --- | --- | --- | --- |
| 방식 | AI 에이전트(조사·완화 계획·예방 제안) | 생성형 AI 조사(CloudWatch 기능) | ML 이상 탐지 |
| 상태 | **2026-03-31 정식 출시** | 2025-06-24 정식 출시 | **2026-10-29 신규 가입 중단, 2027-09-30 종료** |
| 서울 리전 | 지원(2026-10-09) | 정식 출시 목록에 없음(확인 필요) | 지원(종료 예정) |
| 데이터 범위 | AWS + 외부 관측 도구·코드·티켓·Azure | CloudWatch·X-Ray·CloudTrail 변경 | CloudWatch·CloudTrail |
| 시작 | 웹훅·티켓·수동·일정 | **알람 액션**·콘솔·채팅 | 항상 자동 |
| 조치 | 완화 계획 + 승인 시 directed action | runbook 제안 | 권장 사항 |
| 요금 | 에이전트 작업 **초당 $0.0083** | **무료**(일부 API만) | 리소스-시간당 |

| 질문 | 짧은 답 | 문서 |
| --- | --- | --- |
| 새로 시작한다면? | 외부 도구·서울 리전이면 DevOps Agent, CloudWatch 중심·지원 리전이면 Investigations | [01](./01-overview.md), [05](./05-cost-issues.md) |
| DevOps Agent의 기본 단위는? | **Agent Space**(접근 계정·연동·사용자를 정한 보안 경계) | [02](./02-devops-agent.md) |
| 에이전트가 리소스를 바꾸나? | 기본은 읽기 전용. **directed action**은 세 단계 opt-in + 매번 승인 | [02](./02-devops-agent.md) |
| CloudWatch 알람으로 시작하려면? | DevOps Agent: EventBridge·Lambda·웹훅으로 연결 / Investigations: 알람 액션에 바로 지정 | [03](./03-devops-agent-setup.md), [04](./04-investigations-and-guru.md) |
| 보안에서 꼭 알 것? | 서울 등은 추론이 국외로 라우팅될 수 있음, PII 필터 없음, prompt injection | [03](./03-devops-agent-setup.md) |
| DevOps Guru를 쓰고 있다면? | IaC 정리부터, 2027-06-30까지 CloudWatch·DevOps Agent로 이전 | [04](./04-investigations-and-guru.md) |

## 문서 인덱스

| 문서 | 다루는 것 |
| --- | --- |
| [01-overview.md](./01-overview.md) | 개요(단계별 입문) — 장애 대응 흐름, AIOps·AI 에이전트, 세 서비스 비교와 상태, DevOps Guru 대체 매핑, 선택 기준, 요금 개요, 용어 정리 |
| [02-devops-agent.md](./02-devops-agent.md) | 기능, Agent Space, 연동, 조사 시작·결과 전달, directed action, skill·memory·custom agent |
| [03-devops-agent-setup.md](./03-devops-agent-setup.md) | 설정 순서, IAM role, 웹 앱 로그인, CloudWatch 알람 연결 구성, 데이터 처리·보안(국외 추론, PII, prompt injection) |
| [04-investigations-and-guru.md](./04-investigations-and-guru.md) | CloudWatch Investigations(설정 방식, 알람 액션, runbook 제안, IAM·계정 간·채팅), DevOps Guru(지원 종료 일정, 기능, 이전 계획) |
| [05-cost-issues.md](./05-cost-issues.md) | 요금과 비용 시나리오, 한도, 리스크와 대응, 선택 기준, 안티패턴 |

## 출처 (1차)

- AWS DevOps Agent User Guide: https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent.html
- AWS DevOps Agent 요금: https://aws.amazon.com/devops-agent/pricing/
- AWS DevOps Agent 정식 출시: https://aws.amazon.com/about-aws/whats-new/2026/03/aws-devops-agent-generally-available/
- CloudWatch Investigations: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations.html
- DevOps Guru 지원 종료: https://docs.aws.amazon.com/devops-guru/latest/userguide/devops-guru-end-of-support.html
- DevOps Guru 이전 가이드: https://docs.aws.amazon.com/devops-guru/latest/userguide/devops-guru-transition.html

---
*이 문서는 학습·정리 목적의 요약이다. 이 영역은 기능·리전·요금이 거의 매주 바뀌므로 적용 전에 공식 문서를 다시 확인할 것.*
