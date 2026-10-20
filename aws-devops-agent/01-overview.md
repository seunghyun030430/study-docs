# AWS AI 운영 도구 개요

> AWS의 AI 기반 운영 도구는 지금 세 가지가 공존한다. **AWS DevOps Agent**는 장애가 나면 **AI 에이전트가 스스로 메트릭·로그·배포 기록·코드를 뒤져 원인을 찾고 완화 계획을 내놓는** 서비스다(2026-03-31 정식 출시, 사용한 초 단위로 과금). **CloudWatch Investigations**는 같은 종류의 조사를 CloudWatch 콘솔 안에서 무료로 제공한다. **Amazon DevOps Guru**는 ML 이상 탐지 서비스지만 **2026-10-29부터 신규 가입을 받지 않고 2027-09-30에 지원이 끝난다.** 새로 시작한다면 DevOps Agent 또는 CloudWatch Investigations를 본다.

이 문서는 처음 보는 사람을 기준으로 용어를 한 단계씩 쌓아 간다. 처음 나오는 용어는 **굵게** 표시하고 바로 뜻을 적는다. 메트릭·알람·로그는 [aws-cloudwatch/01](../aws-cloudwatch/01-overview.md)을 참고한다. 모든 용어는 맨 아래 [용어 정리](#용어-정리)에 모아 두었다.

## 1단계: 장애 대응은 어떻게 흘러가나

```
감지 ──▶ 알림 ──▶ 조사 ──▶ 완화 ──▶ 사후 분석·예방
(알람)   (온콜)   (원인 찾기) (되돌리기·확장) (재발 방지)
```

| 단계 | 사람이 하던 일 | 오래 걸리는 이유 |
| --- | --- | --- |
| 감지 | 알람이 울림 | — |
| **조사** | 대시보드·로그·최근 배포·변경 이력을 하나씩 열어 봄 | 데이터가 여러 도구에 흩어져 있고, 무엇부터 볼지 경험에 의존 |
| 완화 | 배포 되돌리기, 인스턴스 늘리기 | 원인을 모르면 무엇을 할지 모름 |
| 예방 | 사후 분석(postmortem)으로 개선 항목 정리 | 바빠서 미뤄짐 |

장애 대응 시간의 대부분은 **조사**에 든다. AI 운영 도구가 가장 먼저 겨냥하는 곳도 여기다.

## 2단계: AIOps와 AI 에이전트

- **AIOps**: 운영 데이터(메트릭, 로그, 이벤트)에 머신러닝·AI를 써서 이상을 찾고 원인을 좁히는 방식을 넓게 부르는 말이다.
- **이상 탐지(anomaly detection)**: 과거 패턴을 학습해 "평소와 다른" 값을 찾는다. DevOps Guru와 CloudWatch anomaly detection 알람이 이 방식이다([aws-cloudwatch/03](../aws-cloudwatch/03-alarms.md#anomaly-detection-알람)).
- **생성형 AI 조사**: 대형 언어 모델(**LLM**)이 여러 데이터를 읽고 "이 배포 이후 이 서비스의 에러가 늘었다" 같은 **가설(hypothesis)**을 문장으로 만든다. CloudWatch Investigations가 이 방식이다.
- **AI 에이전트(agent)**: LLM이 **도구(tool)**를 스스로 골라 쓰며 여러 단계를 진행하는 프로그램이다. 예를 들어 로그를 조회하고, 결과를 보고 다시 배포 기록을 확인하는 식이다. AWS DevOps Agent가 이 방식이고, AWS는 이런 서비스를 "frontier agent"라고 부른다.

| 방식 | 하는 일 | 사람의 역할 |
| --- | --- | --- |
| 이상 탐지 | "이 지표가 이상하다"를 알려 줌 | 원인 조사는 사람이 함 |
| 생성형 AI 조사 | 관련 데이터를 모아 가설을 제시 | 가설을 받아들이거나 버림 |
| AI 에이전트 | 조사 → 원인 분석 → 완화 계획까지 스스로 진행, 승인하면 조치도 실행 | 결과를 검토하고 조치를 승인 |

## 3단계: AWS의 세 서비스

| 항목 | **AWS DevOps Agent** | **CloudWatch Investigations** | **Amazon DevOps Guru** |
| --- | --- | --- | --- |
| 방식 | AI 에이전트 | 생성형 AI 조사(CloudWatch 기능) | ML 이상 탐지 |
| 상태 | 2025-12-02 미리 보기 → **2026-03-31 정식 출시** | 2024-12 미리 보기(Amazon Q Developer operational investigations) → **2025-06-24 정식 출시** | **2026-10-29 신규 가입 중단, 2027-09-30 지원 종료** |
| 서울 리전 | **지원**(2026-10-09 추가) | 정식 출시 리전 목록에 **없음**(확인 필요) | 지원(종료 예정) |
| 데이터 범위 | AWS + Datadog·New Relic 등 외부 관측 도구, GitHub·GitLab, ServiceNow·PagerDuty·Slack, Azure | CloudWatch 메트릭·로그, X-Ray, CloudTrail 변경, AWS Health | CloudWatch 메트릭·로그·이벤트, CloudTrail |
| 시작 방법 | 웹훅(PagerDuty·Grafana 등), 티켓(ServiceNow), 수동, 일정 | **CloudWatch 알람 액션**, 콘솔 "Investigate", 채팅 | 항상 자동 분석 |
| 조치 | 완화 계획 + (승인 시) **directed action** | SSM Automation runbook 제안 | 권장 사항 제시 |
| 요금 | **에이전트 사용 초당 $0.0083**(대기 시간 무료) | **무료**(일부 API 호출 비용만) | 리소스-시간당 |
| 자세히 | [02](./02-devops-agent.md), [03](./03-devops-agent-setup.md) | [04](./04-investigations-and-guru.md) | [04](./04-investigations-and-guru.md) |

출처: [DevOps Agent GA](https://aws.amazon.com/about-aws/whats-new/2026/03/aws-devops-agent-generally-available/), [DevOps Agent regions](https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-supported-regions.html), [CloudWatch investigations GA](https://aws.amazon.com/about-aws/whats-new/2025/06/ga-accelerate-troubleshooting-amazon-cloudwatch-investigations), [DevOps Guru end of support](https://docs.aws.amazon.com/devops-guru/latest/userguide/devops-guru-end-of-support.html)

### DevOps Guru가 끝나면 무엇으로 바꾸나

AWS가 안내하는 대체 방법이다. ([DevOps Guru transition](https://docs.aws.amazon.com/devops-guru/latest/userguide/devops-guru-transition.html))

| DevOps Guru 기능 | 대체 |
| --- | --- |
| 메트릭 이상 탐지·알림 | CloudWatch anomaly detection 알람, composite alarm |
| 조사·원인 연결 | **DevOps Agent** 또는 **CloudWatch Investigations** |
| DevOps Guru for RDS | CloudWatch Database Insights + `DBLoad` 이상 탐지 알람 |
| 로그 이상 | CloudWatch Logs anomaly detection |
| 예측(proactive) 인사이트 | **대체 없음**. 사용률 임계 알람, Service Quotas 사용량 알람으로 보완 |

DevOps Guru 페이지는 대체 서비스를 "Amazon DevOps Agent"라고 적는다. DevOps Agent 자체 문서는 "AWS DevOps Agent"라고 부르므로 같은 서비스로 보인다.

## 4단계: DevOps Agent의 핵심 개념 미리 보기

| 개념 | 뜻 |
| --- | --- |
| **Agent Space** | 에이전트가 볼 수 있는 계정·연동·사용자를 정한 **보안 경계 단위**. 데이터도 Space별로 분리 |
| **Topology** | 에이전트가 자동으로 찾아낸 애플리케이션 리소스 관계도 |
| **Investigation** | 알림이나 티켓으로 시작되는 자동 조사 |
| **Evaluation** | 과거 장애를 분석해 관측·인프라·파이프라인 개선을 제안(예방) |
| **Directed action** | 운영자가 명시적으로 요청하고 승인하면 에이전트가 리소스를 바꾸는 작업(기본 꺼짐) |
| **웹 앱** | 운영자가 조사 결과를 보고 에이전트와 대화하는 별도 화면 |

자세한 내용은 [02-devops-agent.md](./02-devops-agent.md)에서 다룬다.

## 5단계: 언제 무엇을 쓰나

| 상황 | 추천 |
| --- | --- |
| CloudWatch 중심, 비용 없이 AI 조사를 시작 | **CloudWatch Investigations**(지원 리전 확인 필요) |
| 외부 관측 도구(Datadog 등)·GitHub·ServiceNow·PagerDuty까지 엮어 자동 조사 | **AWS DevOps Agent** |
| 서울 리전에서 AI 조사 | **AWS DevOps Agent**(Investigations는 서울 미지원으로 보임) |
| DevOps Guru를 쓰는 중 | 2027-06-30까지 이전 완료 목표로 위 표대로 이전 |
| 메트릭 이상 탐지만 필요 | CloudWatch anomaly detection 알람(이 주제의 AI 도구가 꼭 필요하지 않음) |

**AI 조사 도구는 기본 관측성을 대신하지 않는다.** 메트릭·로그·트레이스·배포 기록이 잘 모여 있어야 AI가 원인을 찾을 수 있다. 데이터가 없으면 AI도 추측할 수밖에 없다.

## 6단계: 요금 개요

| 서비스 | 과금 | 출처 |
| --- | --- | --- |
| DevOps Agent | 에이전트가 **일하는 시간** 초당 $0.0083(시간당 약 $29.88). 조사·예방 평가·채팅 모두 같은 단가, 대기 중은 무료. 신규 고객 2개월 무료 체험, AWS Support 플랜에 따라 월 크레딧 | [Pricing](https://aws.amazon.com/devops-agent/pricing/) |
| CloudWatch Investigations | 기능 무료. 조사 중 호출하는 일부 CloudWatch·X-Ray API만 과금 | [Costs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/investigations-costs.html) |
| DevOps Guru | 분석하는 리소스-시간당 $0.0028(Lambda·S3) 또는 $0.0042(그 외) | [Pricing](https://aws.amazon.com/devops-guru/pricing/) |

DevOps Agent는 **조사 한 번이 몇 분**이면 수 달러 수준이다(요금 페이지 예: 8분짜리 조사 10회 = $39.84). 자세한 내용은 [05-cost-issues.md](./05-cost-issues.md)에서 다룬다.

## 용어 정리

| 용어 | 뜻 | 처음 나오는 곳 |
| --- | --- | --- |
| 사후 분석(postmortem) | 장애 후 원인과 재발 방지책을 정리하는 활동 | 1단계 |
| AIOps | 운영 데이터에 ML·AI를 적용하는 방식 | 2단계 |
| 이상 탐지 | 과거 패턴과 다른 값을 찾는 기법 | 2단계 |
| LLM / 가설 | 대형 언어 모델 / AI가 제시하는 원인 후보 | 2단계 |
| AI 에이전트 / 도구 | 도구를 스스로 골라 여러 단계를 진행하는 AI / 에이전트가 호출하는 API·기능 | 2단계 |
| Frontier agent | AWS가 DevOps Agent 같은 자율 에이전트 서비스를 부르는 이름 | 2단계 |
| AWS DevOps Agent / CloudWatch Investigations / DevOps Guru | 3단계 표 참고 | 3단계 |
| Agent Space / topology | 에이전트의 보안 경계 단위 / 리소스 관계도 | 4단계 |
| Investigation / evaluation | 자동 조사 / 예방 평가 | 4단계 |
| Directed action | 승인 후 에이전트가 실행하는 변경 작업 | 4단계 |

## 출처

- https://aws.amazon.com/devops-agent/
- https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent.html
- https://docs.aws.amazon.com/devopsagent/latest/userguide/about-aws-devops-agent-supported-regions.html
- https://aws.amazon.com/blogs/aws/aws-devops-agent-helps-you-accelerate-incident-response-and-improve-system-reliability-preview/
- https://aws.amazon.com/about-aws/whats-new/2026/03/aws-devops-agent-generally-available/
- https://aws.amazon.com/devops-agent/pricing/
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations.html
- https://aws.amazon.com/about-aws/whats-new/2025/06/ga-accelerate-troubleshooting-amazon-cloudwatch-investigations
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/investigations-costs.html
- https://docs.aws.amazon.com/devops-guru/latest/userguide/devops-guru-end-of-support.html
- https://docs.aws.amazon.com/devops-guru/latest/userguide/devops-guru-transition.html
- https://aws.amazon.com/devops-guru/pricing/

---
*조사 기준일: 2026-10-20. 이 영역은 기능·리전·요금이 거의 매주 바뀐다. 적용 전에 각 서비스의 What's new와 요금 페이지를 다시 확인할 것.*
