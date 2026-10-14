# Amazon CloudWatch 개요

> CloudWatch는 AWS 리소스와 애플리케이션의 **숫자(메트릭)와 기록(로그)을 모아서 보고, 이상하면 알리는(알람)** 모니터링 서비스다. 많은 AWS 서비스가 기본 메트릭을 무료로 자동 전송하고, 직접 보내는 custom metric·로그·추가 기능은 **사용량만큼 과금**된다. 그래서 CloudWatch를 잘 쓴다는 것은 "무엇을 보고 알릴지"와 "얼마나 모으고 얼마나 오래 둘지"를 함께 설계하는 것이다.

이 문서는 CloudWatch를 처음 보는 사람을 기준으로 용어를 한 단계씩 쌓아 간다. 처음 나오는 용어는 **굵게** 표시하고 바로 뜻을 적는다. IAM·계정 관련 용어는 [aws-iam/01-overview.md](../aws-iam/01-overview.md)를 참고한다. 모든 용어는 맨 아래 [용어 정리](#용어-정리)에 모아 두었다.

## 1단계: 모니터링이 왜 필요한가

서비스를 운영하면 이런 질문에 답해야 한다.

- 지금 서버가 바쁜가? 응답이 느려졌나? → **숫자의 변화**를 봐야 한다.
- 에러가 났다면 어떤 요청에서, 무슨 메시지로? → **사건의 기록**을 봐야 한다.
- 요청 하나가 여러 서비스를 거치며 어디서 느려졌나? → **요청의 경로**를 봐야 한다.

이 세 가지를 시스템 밖에서 보고 내부 상태를 알아내는 능력을 **관측성(observability)**이라고 한다. 흔히 세 기둥으로 나눈다.

| 기둥 | 무엇 | 예 | CloudWatch에서 |
| --- | --- | --- | --- |
| **메트릭(metric)** | 시간에 따른 숫자 | CPU 사용률 72%, 초당 요청 1,200건 | Metrics |
| **로그(log)** | 일어난 일의 텍스트 기록 | `ERROR payment failed order=123` | Logs |
| **트레이스(trace)** | 요청 하나가 거친 경로와 구간별 시간 | API → 주문 서비스 → DB, 각 120ms | X-Ray·Application Signals |

> "Amazon CloudWatch monitors your Amazon Web Services (AWS) resources and the applications you run on AWS in real time…" — [What is Amazon CloudWatch?](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html)

## 2단계: 메트릭 — 시간에 따른 숫자

**메트릭**은 시간 순서로 쌓이는 숫자 값들이다. 값 하나하나(시각 + 숫자)를 **데이터 포인트(datapoint)**라고 한다.

```
CPUUtilization (EC2 인스턴스 i-0123…)
10:00  12.5%
10:01  13.1%
10:02  71.8%   ← 갑자기 튐
```

메트릭은 세 가지 정보로 구분한다.

| 요소 | 뜻 | 예 |
| --- | --- | --- |
| **Namespace** | 메트릭을 담는 칸. 보통 서비스 이름 | `AWS/EC2`, `AWS/Lambda`, `MyApp` |
| **Metric name** | 무엇을 쟀는지 | `CPUUtilization` |
| **Dimension** | 어느 대상인지 구분하는 `이름=값` | `InstanceId=i-0123…` |

메트릭이 어디서 오는지에 따라 두 종류로 나뉜다.

- **기본 메트릭**: EC2, Lambda, RDS, ELB 등 많은 AWS 서비스가 CloudWatch로 **자동·무료로** 보낸다.
- **Custom metric**: 내 애플리케이션이 직접 보내는 메트릭(예: 주문 수, 결제 지연 시간). **메트릭마다 과금**된다.

메트릭 하나를 평균, 최대, 99번째 백분위수(`p99`) 등 여러 **통계(statistic)**로 볼 수 있다. 자세한 내용은 [02-metrics.md](./02-metrics.md)에서 다룬다.

## 3단계: 알람 — 이상하면 알리기

사람이 그래프를 계속 볼 수는 없다. **알람(alarm)**은 메트릭을 주기적으로 검사해 기준을 넘으면 알린다.

```
알람: "5분 평균 CPU가 80%를 넘는 상태가 3번 나오면"
  → 상태 OK → ALARM 으로 바뀜
  → 액션 실행: SNS로 Slack·이메일 알림, 자동 복구 등
```

- 알람에는 **상태**가 세 가지 있다. `OK`(정상), `ALARM`(기준 초과), `INSUFFICIENT_DATA`(판단할 데이터 부족)다.
- 상태가 바뀔 때 실행하는 일을 **액션(action)**이라고 한다. 알림 메시지를 보내는 서비스인 **SNS**(Simple Notification Service)로 알리는 경우가 가장 흔하다. 서버 재시작이나 자동 확장도 걸 수 있다.

자세한 내용은 [03-alarms.md](./03-alarms.md)에서 다룬다.

## 4단계: 로그 — 일어난 일의 기록

**CloudWatch Logs**는 로그를 모아 저장하고 검색하는 기능이다.

```
Log group:  /aws/lambda/checkout          ← 같은 앱의 로그를 담는 큰 묶음(설정 단위)
 ├─ Log stream: 2026/10/14/[$LATEST]abc   ← 같은 출처(인스턴스·컨테이너)에서 온 로그 줄들
 │   ├─ Log event: 10:00:01 START RequestId ...
 │   └─ Log event: 10:00:02 ERROR payment failed
 └─ Log stream: ...
```

| 단위 | 뜻 |
| --- | --- |
| **Log event** | 로그 한 줄(시각 + 메시지) |
| **Log stream** | 같은 출처에서 온 log event의 연속 |
| **Log group** | log stream들의 묶음. **보존 기간·암호화·권한을 여기서 설정** |

- **보존 기간(retention)**의 기본값은 **"영구 보관(Never Expire)"**이다. 그대로 두면 저장 비용이 계속 쌓인다.
- 로그로 할 수 있는 일:
  - 검색과 분석: **Logs Insights**(전용 쿼리 언어)
  - 실시간 보기: **Live Tail**
  - 패턴이 나오면 숫자로 세기: **metric filter**. 예를 들어 `ERROR`가 나올 때마다 1을 세면 그 값으로 알람을 걸 수 있다.
  - 다른 곳으로 흘려보내기: **subscription filter**

자세한 내용은 [04-logs.md](./04-logs.md)와 [05-logs-insights.md](./05-logs-insights.md)에서 다룬다.

## 5단계: 데이터는 어떻게 들어오나

| 경로 | 무엇 | 예 |
| --- | --- | --- |
| AWS 서비스 자동 전송 | 기본 메트릭, 서비스 로그 | EC2 CPU, Lambda 실행 로그 |
| **CloudWatch agent** | 서버·컨테이너에 설치하는 수집 프로그램. 메모리·디스크 같은 OS 지표와 로그 파일을 보냄 | EC2의 메모리 사용률(기본 메트릭에 없음) |
| API 직접 호출 | `PutMetricData`(메트릭), `PutLogEvents`(로그) | 앱 코드·SDK |
| **EMF**(Embedded Metric Format) | 정해진 JSON 형식으로 로그를 쓰면 CloudWatch가 메트릭을 뽑아냄 | Lambda에서 로그 한 줄로 메트릭까지 |
| **OpenTelemetry(OTLP)** | 업계 표준 수집 규격. CloudWatch가 OTLP 엔드포인트를 제공 | 메트릭·로그·트레이스를 표준 SDK로 전송 |

([OTLP endpoints](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-OTLPEndpoint.html))

EC2의 **메모리·디스크 사용률은 기본 메트릭에 없다.** 하이퍼바이저 밖에서는 보이지 않는 값이라 agent를 설치해야 한다(이 레포 범위에서는 개요만 다룬다).

## 6단계: CloudWatch 기능 지도

공식 문서의 기능 목록을 이 레포 기준으로 정리했다. ([What is Amazon CloudWatch?](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html))

| 영역 | 기능 | 한 줄 설명 | 이 레포 |
| --- | --- | --- | --- |
| 메트릭·알람 | Metrics, Alarms, **Dashboards** | 숫자 수집·알림·그래프 화면 | **02, 03** |
| 로그 | Logs, Logs Insights, Live Tail, 로그 이상 탐지 | 로그 수집·검색·실시간 보기 | **04, 05** |
| APM(애플리케이션 성능 모니터링) | Application Signals, X-Ray, Synthetics, RUM, SLO | 아래 표 | 개요만 |
| 인프라 인사이트 | Container Insights, Lambda Insights, Database Insights | ECS·EKS, Lambda, DB의 상세 지표를 미리 만든 화면으로 | 개요만 |
| 수집 | CloudWatch agent, OpenTelemetry | 5단계 | 개요만 |
| 여러 계정 | **Cross-account observability** | 모니터링 계정 하나에서 다른 계정의 메트릭·로그·트레이스를 봄 | — |
| 네트워크 | Internet Monitor, Network Flow Monitor, Network Synthetic Monitor | 인터넷·네트워크 경로 품질 | — |
| AI | Investigations, CloudWatch Omni | 장애 원인 조사 보조, 자연어 질의 | — |

### APM 기능 한눈에

| 기능 | 무엇 | 언제 |
| --- | --- | --- |
| **Application Signals** | EC2·ECS·Lambda 위 앱의 호출 수·지연·에러를 자동 수집해 서비스 지도와 SLO로 보여 줌(Java, Python, Node.js, .NET) | "어느 서비스가 느린가" |
| **X-Ray**(트레이스) | 요청 하나가 거친 서비스별 구간과 시간을 기록. CloudWatch와 통합되어 trace 데이터를 로그 그룹(`aws/spans`)으로도 저장 | "요청 하나가 어디서 느려졌나" |
| **Synthetics** | 정해진 주기(최소 1분)로 스크립트(canary)가 내 API·웹을 호출해 확인 | "사용자보다 먼저 장애 발견" |
| **RUM**(Real User Monitoring) | 실제 사용자 브라우저·모바일에서 성능 데이터 수집(30일 보관) | "실제 사용자 체감 속도" |
| **SLO**(Service Level Objective) | "99.9%는 300ms 안에" 같은 목표와 달성률 추적 | 신뢰성 목표 관리 |

출처: [Application Signals](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Application-Monitoring-Sections.html), [Synthetics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries.html), [RUM](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-RUM.html), [Transaction Search](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Transaction-Search.html)

### 이름이 바뀐 것들

- **CloudWatch Events는 Amazon EventBridge가 되었다.** 예전 CloudWatch Events 문서 주소는 지금 EventBridge 문서로 연결된다. "EC2가 종료되면 Lambda 실행" 같은 이벤트 규칙은 EventBridge에서 만든다. 알람 상태 변화도 EventBridge 이벤트로 나간다.
- 현재 공식 문서는 메트릭을 "OpenTelemetry Metrics(권장)"와 "CloudWatch Metrics(Classic)"로 나눠 소개한다. 이 레포는 지금 가장 널리 쓰이는 **Classic 메트릭**을 기준으로 설명한다.

## 7단계: 요금은 어떻게 매겨지나

CloudWatch는 **기능별 사용량 과금**이다. 큰 틀만 먼저 잡고, 자세한 내용은 [06-cost-issues.md](./06-cost-issues.md)에서 다룬다.

us-east-1 기준 값이다(2026-10-07 게시 가격표). 서울 리전 값은 확인한 것만 괄호에 적었다.

| 항목 | 단가 | Free tier(매월) |
| --- | --- | --- |
| Custom metric | 메트릭당 월 $0.30(1만 개까지, 이후 구간별 할인) | 10개 |
| API 요청 | 1,000건당 $0.01(`GetMetricData`는 조회한 메트릭 1,000개당 $0.01) | 100만 건(`GetMetricData` 등 일부 제외) |
| 알람 | 표준 알람 메트릭당 월 $0.10 | 10개 |
| 로그 수집(Standard) | GB당 $0.50(서울 $0.76) | 5 GB |
| 로그 저장 | GB-월당 $0.03(서울 $0.0314) | 5 GB |
| Logs Insights 조회 | 스캔한 GB당 $0.005(서울 $0.0076) | 5 GB |
| Dashboard | — | 3개(각 메트릭 50개까지) |

출처: [CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/), [Understanding CloudWatch billing](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html)

비용이 커지는 전형적인 원인은 세 가지다.

1. **로그 수집량**: GB당 요금이 저장보다 훨씬 비싸다.
2. **custom metric 개수**: dimension 값이 많으면 메트릭 수가 폭증한다.
3. **영구 보관 로그**: 보존 기간 기본값이 영구라 저장 비용이 계속 쌓인다.

## 용어 정리

| 용어 | 뜻 | 처음 나오는 곳 |
| --- | --- | --- |
| 관측성(observability) | 메트릭·로그·트레이스로 시스템 내부 상태를 알아내는 능력 | 1단계 |
| 메트릭 / 로그 / 트레이스 | 시간에 따른 숫자 / 사건의 텍스트 기록 / 요청 하나의 경로 | 1단계 |
| 데이터 포인트 | 메트릭의 값 하나(시각 + 숫자) | 2단계 |
| Namespace / metric name / dimension | 메트릭을 담는 칸 / 메트릭 이름 / 대상을 구분하는 `이름=값` | 2단계 |
| 기본 메트릭 / custom metric | AWS가 무료로 보내는 메트릭 / 직접 보내는 유료 메트릭 | 2단계 |
| 통계(statistic) | 평균·최대·백분위수 등 데이터를 요약하는 방법 | 2단계 |
| 알람 / 상태 / 액션 | 메트릭 감시 규칙 / OK·ALARM·INSUFFICIENT_DATA / 상태가 바뀔 때 할 일 | 3단계 |
| SNS | 알림 메시지를 보내는 AWS 서비스 | 3단계 |
| Log event / stream / group | 로그 한 줄 / 같은 출처의 줄들 / stream 묶음(설정 단위) | 4단계 |
| 보존 기간(retention) | 로그를 얼마나 보관할지. 기본값은 영구 | 4단계 |
| Logs Insights / Live Tail | 로그 쿼리 / 실시간 로그 보기 | 4단계 |
| Metric filter / subscription filter | 로그 패턴을 메트릭으로 / 로그를 다른 곳으로 전달 | 4단계 |
| CloudWatch agent | 서버에 설치하는 수집 프로그램 | 5단계 |
| EMF | 로그 안에 메트릭을 넣는 JSON 형식 | 5단계 |
| OpenTelemetry(OTLP) | 관측 데이터 수집 표준 | 5단계 |
| Dashboard | 그래프를 모아 보는 화면 | 6단계 |
| APM | 애플리케이션 성능 모니터링 | 6단계 |
| Application Signals / X-Ray / Synthetics / RUM / SLO | 6단계 APM 표 참고 | 6단계 |
| Cross-account observability | 한 계정에서 여러 계정 데이터를 보는 기능 | 6단계 |
| EventBridge | 이벤트 규칙 서비스(예전 CloudWatch Events) | 6단계 |
| Free tier | 매월 무료로 쓸 수 있는 양 | 7단계 |

## 출처

- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogsConcepts.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-OTLPEndpoint.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Application-Monitoring-Sections.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Synthetics_Canaries.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-RUM.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch-Transaction-Search.html
- https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-what-is.html
- https://aws.amazon.com/cloudwatch/pricing/
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html

---
*조사 기준일: 2026-10-14. 요금은 AWS Price List API(2026-10-07 게시)의 us-east-1·서울 값이다. 대시보드 단가는 1차 출처에서 확인하지 못했다(확인 필요). 다이어그램은 이해를 돕기 위한 예시다.*
