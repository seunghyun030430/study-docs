# 비용·한도·리스크·선택 기준

> DevOps Agent는 **에이전트가 일한 초 단위**로 과금되고(초당 $0.0083, 시간당 약 $29.88), 조사 한 번이 보통 몇 분이라 **조사 횟수가 비용을 정한다.** 그래서 모든 알람을 조사로 연결하지 않고, 중요한 composite alarm·장애 티켓만 연결한다. CloudWatch Investigations는 무료지만 월 150회·동시 2개 한도가 있다. 리스크는 비용보다 **데이터 위치(국외 추론), PII, prompt injection, 자동 조치 남용**에서 나온다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[04](./04-investigations-and-guru.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 에이전트 시간(agent-second) | 에이전트가 실제로 작업한 시간. 대기 중은 제외 |
| Support 크레딧 | AWS Support 플랜 요금의 일부를 DevOps Agent 사용료로 돌려주는 월 크레딧 |
| 알림 폭주(alert storm) | 장애 하나로 알람이 수십 개 동시에 울리는 현상 |

## 요금

### AWS DevOps Agent

출처: [DevOps Agent pricing](https://aws.amazon.com/devops-agent/pricing/)

| 항목 | 내용 |
| --- | --- |
| 단가 | **초당 $0.0083**(시간당 약 $29.88). 조사, 예방 평가, 온디맨드 SRE 작업(채팅·custom agent)이 모두 같은 단가 |
| 대기 | 에이전트가 쉬는 동안은 무료 |
| 릴리스 관리 | 미리 보기 동안 무료 |
| 무료 체험 | 신규 고객은 정식 출시 후 첫 작업부터 **2개월**. 매월 Agent Space 10개, 조사 20시간, 평가 15시간, 채팅 20시간까지 |
| Support 크레딧 | 전달 Support 요금의 일정 비율을 매월 크레딧으로: Unified Operations 100%, Enterprise 75%, Business Support+ 30%. 그달에 쓰지 않으면 소멸 |
| 별도 요금 | 에이전트가 호출하는 다른 서비스(CloudWatch Logs Insights 쿼리 등) |

```
요금 페이지 예: 8분짜리 조사 10회 = 10 × 480초 × $0.0083 = $39.84
```

### 비용 시나리오(이 레포의 계산)

| 운영 방식 | 월 조사 수(평균 8분 가정) | 월 비용 |
| --- | --- | --- |
| 장애 티켓·중요 알람만 연결 | 30회 | 30 × 480 × $0.0083 ≈ **$120** |
| 모든 경고 알람을 연결 | 600회 | 600 × 480 × $0.0083 ≈ **$2,390** |
| 매일 예방 평가 1시간 | 30시간 | 30 × 3,600 × $0.0083 ≈ **$896** |

조사 시간은 환경마다 다르므로 실제로는 무료 체험 기간에 조사 한 번이 몇 분인지 재 보고 추정한다. Logs Insights 쿼리 비용([aws-cloudwatch/05](../aws-cloudwatch/05-logs-insights.md#한도와-비용))도 함께 늘어난다.

### CloudWatch Investigations와 DevOps Guru

| 서비스 | 요금 | 한도 |
| --- | --- | --- |
| CloudWatch Investigations | **무료**. 조사가 호출하는 일부 API(`ListMetrics`, `GetInsightRuleReport`, X-Ray `GetTraceSummaries` 등)만 과금, AWS는 "minimal"이라고 안내 | 계정당 그룹 1개, 동시 2개, AI 분석 월 150회 |
| DevOps Guru | 활성 리소스-시간당 $0.0028(Lambda·S3) / $0.0042(그 외), API 1만 건당 $0.40 | 2027-09-30 종료 |

출처: [Investigations costs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/investigations-costs.html), [DevOps Guru pricing](https://aws.amazon.com/devops-guru/pricing/)

## 한도

출처: [DevOps Agent quotas](https://docs.aws.amazon.com/devopsagent/latest/userguide/quotas.html)

| 항목(DevOps Agent) | 기본값 | 증설 |
| --- | --- | --- |
| 계정·리전당 Agent Space | 100 | 가능 |
| Space당 동시 조사 | **3** | 가능 |
| Space당 동시 예방 평가 | 1 | 불가 |
| Space당 동시 채팅 | 10 | 가능 |
| Space당 MCP 도구 | 500 | 불가 |
| Space당 custom agent | 200 | 불가 |
| custom agent 실행 제한 시간 | 1시간 | 불가 |

**동시 조사 3개**는 알림 폭주 때 금방 찬다. 알람 하나하나가 아니라 composite alarm이나 장애 티켓 단위로 조사를 시작하게 설계한다([aws-cloudwatch/03](../aws-cloudwatch/03-alarms.md#알림-줄이기)).

## 리스크와 대응

| 리스크 | 내용 | 대응 | 자세히 |
| --- | --- | --- | --- |
| **국외 추론** | 서울 등 일부 리전은 추론이 전 세계로 라우팅. 리전 제한 SCP로 막을 수 없음 | 전달되는 데이터 범위 검토, 규정 검토 | [03](./03-devops-agent-setup.md#데이터-처리와-보안) |
| **PII 노출** | 에이전트가 로그 원문을 읽음, PII 필터 없음 | 로그 마스킹(CloudWatch data protection), 앱에서 PII를 기록하지 않기 | [aws-cloudwatch/04](../aws-cloudwatch/04-logs.md#그-밖의-기능) |
| **Prompt injection** | 로그·태그·티켓에 숨긴 지시문 | 읽기 전용 role, 신뢰할 수 있는 MCP만 연결, 승인 내용 확인 | [03](./03-devops-agent-setup.md#prompt-injection) |
| **자동 조치 남용** | directed action·runbook을 습관적으로 승인 | 최소 권한 action role, 승인자 한정, CloudTrail 검토 | [02](./02-devops-agent.md#directed-action-에이전트가-리소스를-바꿀-때) |
| **잘못된 결론** | 데이터가 부족하면 그럴듯한 오답 | 결론을 가설로 다루고 근거 데이터를 확인, 관측성 보강 | [01](./01-overview.md#5단계-언제-무엇을-쓰나) |
| **비용 폭증** | 모든 알람 → 조사 | 중요한 알람만, 중복 제거 | 위 시나리오 |
| **서비스 종료** | DevOps Guru처럼 AI 운영 도구는 수명이 짧을 수 있음 | 특정 서비스 고유 기능에 운영 절차를 깊게 묶지 않기, IaC로 관리 | [04](./04-investigations-and-guru.md#amazon-devops-guru) |

## 선택 기준

| 상황 | 추천 |
| --- | --- |
| 지원 리전에서 CloudWatch 중심 운영, 비용 0으로 시작 | **CloudWatch Investigations** |
| 서울 리전 워크로드 | **AWS DevOps Agent**(Investigations 서울 지원 여부 확인 필요) |
| Datadog·New Relic·GitHub·ServiceNow·PagerDuty 등 외부 도구를 함께 봐야 함 | **AWS DevOps Agent** |
| 장애 후 개선 제안(예방)까지 자동화 | **AWS DevOps Agent**(evaluation) |
| 데이터의 국외 처리가 규정상 불가 | 도입 전 규정 검토. 추론 위치를 통제할 방법이 문서에 없음 |
| 메트릭 이상 탐지만 필요 | CloudWatch anomaly detection 알람으로 충분 |
| DevOps Guru 사용 중 | [04의 이전 계획](./04-investigations-and-guru.md#이전-계획)대로 2027-06-30까지 이전 |

위 표는 공식 문서의 기능·제약으로 이 레포에서 정리한 것이며, AWS의 공식 선택 가이드는 아니다.

## 안티패턴

| 안티패턴 | 문제 | 대안 |
| --- | --- | --- |
| 관측성이 부족한데 AI부터 도입 | AI가 근거 없이 추측 | 메트릭·로그·트레이스·배포 기록부터 정비 |
| 모든 알람을 조사로 연결 | 비용·동시 한도 소진 | composite alarm·티켓 단위 |
| 에이전트 role에 넓은 쓰기 권한 | prompt injection 시 피해 | 읽기 전용, action role 분리 |
| AI 결론을 바로 실행 | 오진에 따른 잘못된 조치 | 근거 확인 후 승인 |
| 로그에 PII를 그대로 남김 | AI 서비스로 개인정보 전달 | 마스킹 |
| DevOps Guru 신규 도입 | 2026-10-29 이후 가입 불가, 2027-09-30 종료 | CloudWatch·DevOps Agent |

## 출처

- https://aws.amazon.com/devops-agent/pricing/
- https://docs.aws.amazon.com/devopsagent/latest/userguide/quotas.html
- https://docs.aws.amazon.com/devopsagent/latest/userguide/aws-devops-agent-security.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/investigations-costs.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Investigations.html
- https://aws.amazon.com/devops-guru/pricing/
- https://docs.aws.amazon.com/devops-guru/latest/userguide/devops-guru-end-of-support.html

---
*조사 기준일: 2026-10-20. 비용 시나리오는 요금 페이지 단가로 이 레포에서 계산했으며, 조사 한 번의 실제 소요 시간은 환경마다 다르다.*
