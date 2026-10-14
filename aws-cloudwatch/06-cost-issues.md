# 비용·한도·운영 이슈

> CloudWatch 요금이 갑자기 커지는 원인은 대부분 다섯 가지다. **로그 수집량**, **영구 보관 로그**, **custom metric 카디널리티**(EMF·metric filter의 dimension 포함), **외부 도구의 `GetMetricData` 폴링**, **상세 모니터링·고해상도 남용**이다. 먼저 Cost Explorer에서 **usage type별로** 어디서 돈이 나가는지 확인하고, 수집량 줄이기 → 보존 기간 → 메트릭 설계 순으로 손본다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[05](./05-logs-insights.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Cost Explorer | AWS 비용을 서비스·사용 유형별로 나눠 보는 도구 |
| Usage type | 청구서에서 비용 항목을 구분하는 이름(예: `DataProcessing-Bytes`) |
| Metric stream | 메트릭을 Firehose로 거의 실시간 내보내는 기능. 외부 도구가 API로 폴링하는 대신 씀 |
| Contributor Insights | 로그에서 "가장 많이 기여한 값"(상위 IP, 상위 사용자 등)을 계산하는 기능 |
| TPS | 초당 요청 수(transactions per second) |

## 요금 구조 한눈에

출처: [CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/), AWS Price List API(2026-10-07 게시). us-east-1 기준이며, 서울은 확인한 값만 괄호에 적었다.

### 메트릭·알람·API

| 항목 | 단가 |
| --- | --- |
| Custom metric(월) | 첫 10,000개 $0.30 → 다음 240,000개 $0.10 → 다음 750,000개 $0.05 → 100만 개 초과 $0.02 (서울 같음) |
| 상세 모니터링(EC2 1분) | custom metric과 같은 방식으로 과금 |
| API 요청 | 1,000건당 $0.01 |
| `GetMetricData` | 조회한 메트릭 1,000개당 $0.01. **free tier 대상 아님** |
| 알람 | 표준 $0.10, 고해상도 $0.30(알람 메트릭당 월), composite $0.50 |
| Contributor Insights | 규칙당 월 $0.50 + 일치 이벤트 100만 건당 $0.02 |
| Dashboard | 3개까지 무료(단가는 1차 출처에서 확인하지 못함, 확인 필요) |

### 로그

| 항목 | us-east-1 | 서울 |
| --- | --- | --- |
| 수집 — Standard | GB당 $0.50 | $0.76 |
| 수집 — Infrequent Access | GB당 $0.25 | $0.38 |
| 저장 | GB-월당 $0.03 | $0.0314 |
| 저장 — Intelligent Tiering IA / Archive Instant Access | $0.018 / $0.006 | $0.01884 / $0.00628 |
| Logs Insights 스캔 | GB당 $0.005 | $0.0076 |
| Live Tail | 분당 $0.01 | $0.01 |
| Data protection 스캔 | GB당 $0.12 | 확인 필요 |

### Vended logs(AWS 서비스가 보내는 로그, us-east-1)

| 월 수집량 구간 | CloudWatch Logs Standard | IA | S3·Firehose로 전달 |
| --- | --- | --- | --- |
| 첫 10 TB | $0.50 | $0.25 | $0.25 |
| 다음 20 TB | $0.25 | $0.15 | $0.15 |
| 다음 20 TB | $0.10 | $0.075 | $0.075 |
| 50 TB 초과 | $0.05 | $0.05 | $0.05 |

VPC Flow Logs처럼 양이 많은 로그는 **S3로 바로 보내면 수집 단가가 절반**이다. 대신 Logs Insights·metric filter를 쓸 수 없다.

### Free tier(매월)

메트릭 10개, 알람 10개, API 요청 100만 건(`GetMetricData` 등 제외), 로그 수집·저장·Logs Insights 스캔 각 5 GB, Live Tail 1,800분, 대시보드 3개(각 메트릭 50개), Contributor Insights 규칙 1개. ([Understanding billing](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html))

## 계산해 보기

아래는 조사 기준일의 us-east-1 단가로 계산한 예시다(free tier 제외).

**예 1: 하루 20 GB 로그를 영구 보관**

| 항목 | 계산 | 월 비용 |
| --- | --- | --- |
| 수집 | 20 GB × 30일 × $0.50 | $300 |
| 저장(1년 뒤 누적 약 7.3 TB) | 20 GB × 365일 = 7,300 GB × $0.03 | 1년 뒤 월 약 $219, 이후에도 계속 증가 |

같은 로그를 보존 30일로 두면 저장량은 약 600 GB, 월 $18 수준이다. 이 예에서는 **수집 비용이 가장 크다.** 로그 양 자체를 줄이는 것이 가장 효과적이다.

**예 2: EMF에 `UserId`를 dimension으로 넣음**

활성 사용자 5만 명 × 메트릭 2개 = 메트릭 100,000개다.

| 구간 | 계산 | 금액 |
| --- | --- | --- |
| 첫 10,000개 | 10,000 × $0.30 | $3,000 |
| 다음 90,000개 | 90,000 × $0.10 | $9,000 |
| 합계 | | **월 약 $12,000** |

dimension을 `Service`, `Env` 정도로 두면 메트릭 수십 개, 월 수 달러에 그친다.

## 비용이 커지는 원인과 대응

출처: [Analyzing, optimizing, and reducing CloudWatch costs](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html)

| 원인 | 확인할 usage type | 대응 |
| --- | --- | --- |
| 로그 수집량 | `DataProcessing-Bytes`, `VendedLog-Bytes` | 로그 레벨 낮추기, 디버그 로그 샘플링, 가끔 보는 로그는 IA 클래스, VPC Flow Logs 등은 S3로 |
| 로그 저장량 | `TimedStorage-ByteHrs` | 모든 log group에 보존 기간, Intelligent Tiering, 장기 보관은 S3 |
| Custom metric 수 | `MetricMonitorUsage` | 고카디널리티 dimension 제거(EMF, metric filter 포함), 안 쓰는 메트릭 발행 중단 |
| `GetMetricData` 호출 | `GMD-Metrics` | 외부 모니터링 도구의 수집 주기·대상 축소, **metric stream**으로 전환 검토 |
| `PutMetricData` 호출 수 | `Requests` | 값을 모아 한 번에 보내기(요청당 메트릭 1,000개, 값 배열 150개) |
| 상세 모니터링 | `MetricMonitorUsage` | Auto Scaling 등 꼭 필요한 인스턴스만 1분 모니터링 |
| 알람 | `AlarmMonitorUsage`, `HighResAlarmMonitorUsage` | 고해상도 알람은 꼭 필요한 곳만, 쓰지 않는 알람 정리 |
| Logs Insights 스캔 | `DataScanned-Bytes` | 시간 범위·log group 좁히기, field index, 대시보드 쿼리 위젯 줄이기 |

usage type에는 리전 접두사가 붙는다(예: us-east-1은 `USE1-`, 서울은 `APN2-`). Cost Explorer에서 서비스를 CloudWatch로 거르고 **Usage Type으로 그룹화**하면 위 표의 어느 항목인지 바로 보인다.

### 보존 기간 없는 log group 찾기

```bash
aws logs describe-log-groups \
  --query 'logGroups[?retentionInDays==`null`].[logGroupName,storedBytes]' \
  --output table
```

JMESPath에서 보존 기간이 없는 항목을 거르는 조건(`==\`null\``)은 환경에 따라 다르게 동작할 수 있다. 결과가 비어 있으면 `--query 'logGroups[].[logGroupName,retentionInDays,storedBytes]'`로 전체를 보고 확인한다.

## 주요 한도

출처: [CloudWatch quotas](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_limits.html), [CloudWatch Logs quotas](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cloudwatch_limits_cwl.html)

| 항목 | 기본 한도 | 증설 |
| --- | --- | --- |
| `PutMetricData` | 초당 500회 | 가능 |
| `GetMetricData` | 초당 500회 | 가능 |
| `GetMetricStatistics` / `ListMetrics` | 초당 400회 / 25회 | — |
| `PutMetricAlarm` | 초당 3회 | 가능 |
| 알람 수 | 제한 없음(Metrics Insights 알람은 리전당 200개) | — |
| Log group | 계정당 1,000,000개 | 가능 |
| `PutLogEvents` | 계정당 초당 5,000회 | 가능 |
| `CreateLogGroup` | 초당 10회 | — |
| `FilterLogEvents` | us-east-1 초당 25회, 대부분 리전 10회 | — |
| Metric filter / subscription filter | log group당 100개 / 5개 | 불가 |
| 동시 S3 export | 계정당 1개 | 불가 |
| Logs Insights 동시 쿼리 | 100개(PPL·SQL 15개) | — |
| Live Tail 동시 세션 | 15개 | 가능 |

**`ThrottlingException`이 나면** 먼저 어느 API인지 보고, 호출을 모으거나(배치) 지수 백오프로 재시도한다. 기본 한도가 낮은 API(`PutMetricAlarm` 초당 3회, `CreateLogGroup` 초당 10회)는 IaC로 리소스를 한꺼번에 만들 때 자주 걸린다.

## 운영 안티패턴 종합

| 안티패턴 | 문제 | 대안 | 자세히 |
| --- | --- | --- | --- |
| 로그 보존 기간 미설정 | 저장 비용 무한 증가 | 모든 log group에 보존 기간 | [04](./04-logs.md#보존-기간) |
| 고유 ID를 dimension으로 | custom metric 폭증 | 로그에 남기고 Logs Insights로 | [02](./02-metrics.md#dimension이-비용과-설계를-정한다) |
| 에러 메트릭 알람에 기본 missing data 처리 | 평소 `INSUFFICIENT_DATA` | `notBreaching` 또는 metric filter `defaultValue=0` | [03](./03-alarms.md#데이터가-빠졌을-때-treatmissingdata) |
| 증상마다 사람 호출 | 알림 피로 | composite alarm, suppressor, mute rule | [03](./03-alarms.md#알림-줄이기) |
| 평균 지연 시간만 감시 | 꼬리 지연을 놓침 | `p99`·`p95` | [02](./02-metrics.md#통계statistic) |
| 대시보드에 Logs Insights 위젯 남발 | 열 때마다 스캔 과금 | metric filter로 메트릭화 | [05](./05-logs-insights.md#비용을-줄이는-습관) |
| 필터가 필요한 로그를 IA 클래스로 | 기능 제한, 클래스 변경 불가 | 용도별 log group 분리 | [04](./04-logs.md#로그-클래스) |
| EC2 메모리·디스크 알람을 기본 메트릭으로 찾음 | 기본 메트릭에 없음 | CloudWatch agent 설치 | [01](./01-overview.md#5단계-데이터는-어떻게-들어오나) |
| 알림 경로를 테스트하지 않음 | SNS 구독 미확인 등으로 장애를 모름 | `set-alarm-state`로 정기 테스트 | [03](./03-alarms.md#사용법--예시) |
| 비용 원인을 추측으로 대응 | 엉뚱한 곳을 줄임 | Cost Explorer usage type부터 확인 | 이 문서 |

## 출처

- https://aws.amazon.com/cloudwatch/pricing/
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_limits.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cloudwatch_limits_cwl.html
- https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonCloudWatch/current/us-east-1/index.json
- https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AmazonCloudWatch/current/ap-northeast-2/index.json

---
*조사 기준일: 2026-10-14. 요금은 AWS Price List API(2026-10-07 게시) 값이고, 계산 예시는 그 단가로 이 레포에서 계산한 것이다. 실제 청구액은 free tier, 리전, 할인 약정에 따라 다르다.*
