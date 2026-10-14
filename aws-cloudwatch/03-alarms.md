# Alarms

> 알람은 메트릭(또는 수식)을 주기적으로 평가해 **OK / ALARM / INSUFFICIENT_DATA** 중 하나의 상태를 정하고, **상태가 바뀔 때만** 액션(SNS 알림, Lambda, EC2 재시작, Auto Scaling 등)을 실행한다. 핵심 설정은 세 가지다. **"N개 기간 중 M개가 임계값을 넘으면"(M out of N)**, **데이터가 없을 때의 처리(TreatMissingData)**, 그리고 **알림 피로를 줄이는 composite alarm·suppression·mute rule**이다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)와 [02-metrics.md](./02-metrics.md)의 용어(period, statistic, percentile, metric math 등)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Threshold(임계값) | 넘으면 문제로 보는 기준 값(예: CPU 80%) |
| Breaching | 데이터 포인트가 임계값을 넘은 상태 |
| Evaluation period | 알람이 판단에 쓰는 최근 기간 수(N) |
| Datapoints to alarm | N개 중 몇 개가 넘어야 ALARM이 되는지(M) |
| Evaluation range | 데이터가 빠졌을 때 CloudWatch가 더 거슬러 올라가 가져오는 범위 |
| SNS | 알림을 이메일·SMS·Slack 연동 등으로 보내는 메시지 서비스 |
| Composite alarm | 여러 알람의 상태를 AND/OR/NOT으로 조합한 상위 알람 |
| Suppression / mute rule | 특정 조건이나 시간대에 알람 액션을 실행하지 않도록 막는 기능 |
| Anomaly detection | 과거 패턴을 학습해 "정상 범위(band)"를 만들고, 그 밖으로 나가면 알리는 기능 |
| Flapping | 알람이 OK와 ALARM을 짧은 간격으로 오가는 현상 |

## 상태

출처: [Alarm evaluation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-evaluation.html)

| 상태 | 뜻 |
| --- | --- |
| `OK` | 값이 임계값 안쪽 |
| `ALARM` | 값이 임계값 바깥 |
| `INSUFFICIENT_DATA` | 알람을 막 만들었거나, 메트릭이 없거나, 판단할 데이터가 부족 |

알람은 처음 만들면 `INSUFFICIENT_DATA`에서 시작한다. 최근에는 상태와 별도로 **evaluation state**도 볼 수 있다. 예를 들어 `PARTIAL_DATA`(한도 때문에 데이터 일부만 가져옴), `EVALUATION_ERROR`(설정 오류), `IN_WARM_UP`(워밍업 중)이 있다.

## 평가 방식: M out of N

출처: [Alarm evaluation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-evaluation.html), [PutMetricAlarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutMetricAlarm.html)

| 설정 | 뜻 | 예 |
| --- | --- | --- |
| `Period` | 데이터 포인트 하나를 만드는 시간(초). 10, 20, 30 또는 60의 배수 | 300(5분 평균) |
| `EvaluationPeriods`(N) | 최근 몇 개의 period를 볼지 | 5 |
| `DatapointsToAlarm`(M) | 그중 몇 개가 넘으면 ALARM인지. **연속일 필요 없음** | 3 |
| `Threshold` + `ComparisonOperator` | 기준 값과 비교 방식 | `> 80` |

```
period = 5분, N = 5, M = 3, 임계값 CPU > 80%

시간    00:00  00:05  00:10  00:15  00:20
CPU      85     70     90     60     88
넘음?    O      X      O      X      O     → 5개 중 3개 넘음 → ALARM
```

- **M = N = 1**이면 한 번 튀어도 바로 울린다. 잡음이 많은 메트릭은 flapping이 생긴다.
- **M < N**(예: 5개 중 3개)으로 하면 잠깐 튀는 값은 무시하고, 지속되는 문제만 잡는다.
- 평가는 period가 1분 이상이면 **매분**, 10·20·30초면 **10초마다** 이뤄진다.
- N × period는 최대 7일이고, period가 1시간 미만이면 최대 1일이다.

### 데이터가 빠졌을 때: TreatMissingData

출처: [Alarms and missing data](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarms-and-missing-data.html)

메트릭이 항상 데이터를 보내는 것은 아니다. 예를 들어 에러가 없으면 에러 메트릭 자체가 안 오는 경우가 많다.

CloudWatch는 먼저 N보다 넓은 범위(**evaluation range**)에서 데이터를 가져온다. 실제 데이터가 N개 이상 있으면 그것으로 판단하고, 이때는 아래 설정을 쓰지 않는다. N개가 안 되면 빈 자리를 아래 규칙으로 채운다.

| 값 | 빈 데이터를 | 언제 |
| --- | --- | --- |
| `missing`(기본값) | 판단에 안 씀. 범위 전체가 비면 `INSUFFICIENT_DATA` | 꾸준히 오는 메트릭, **EC2 stop/terminate/reboot/recover 액션 알람** |
| `notBreaching` | 정상으로 봄 | **문제가 있을 때만 데이터가 오는 메트릭**(에러 수, throttling 수) |
| `breaching` | 문제로 봄 | "데이터가 안 오는 것 자체가 장애"인 경우(하트비트, 배포 롤백 판단) |
| `ignore` | 무시하고 현재 상태 유지 | 상태를 쉽게 바꾸고 싶지 않을 때 |

- 에러 메트릭에 기본값(`missing`)을 쓰면, 에러가 없는 평소에 `INSUFFICIENT_DATA`가 되어 대시보드가 회색으로 보인다. `notBreaching`이 맞다.
- DynamoDB 메트릭 알람의 동작은 문서마다 다르게 적혀 있다. 사용자 가이드는 "기본값이 `ignore`이고 바꿀 수 있다"고 하지만, API 레퍼런스는 "다른 값을 골라도 항상 무시한다"고 한다(확인 필요).

### 백분위수 알람

`p99` 같은 백분위수 알람은 샘플이 적으면 값이 들쭉날쭉하다. `EvaluateLowSampleCountPercentile=ignore`로 두면, 샘플이 너무 적은 동안에는 상태를 바꾸지 않는다. 여기서 "너무 적다"는 `p99`의 경우 데이터 포인트 1,000개 미만을 뜻한다. ([Percentiles with low samples](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/percentiles-with-low-samples.html))

### 그 밖의 평가 옵션

| 옵션 | 내용 | 출처 |
| --- | --- | --- |
| 고해상도 알람 | period 10·20·30초. 메트릭도 1초 해상도로 보내야 함(아니면 `INSUFFICIENT_DATA`가 잦음). 더 비쌈 | [PutMetricAlarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutMetricAlarm.html) |
| Metric math 알람 | 수식 결과에 알람. 메트릭 최대 10개, 결과는 시계열 하나, `SEARCH` 불가 | [Alarm on metric math](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create-alarm-on-metric-math-expression.html) |
| Evaluation window | 기본은 sliding window. **wall clock window**는 시·일·주 경계에 맞춰 평가(고해상도·Auto Scaling 알람에는 비권장) | [Evaluation window](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-evaluation-window.html) |
| Warm-up | 1–2,880분 동안 `INSUFFICIENT_DATA`로 두고 액션을 실행하지 않음(새 리소스 시작 직후 오탐 방지) | [Warm-up](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-warm-up.html) |
| Metrics Insights 알람 | SQL 쿼리 하나로 여러 시계열을 감시(예: `GROUP BY InstanceId`). 새 리소스 자동 포함. 최근 3시간만 평가, 리전당 200개 | [Multi-time-series alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/multi-time-series-alarm.html) |

## 액션

출처: [Alarm actions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-actions.html)

| 액션 | 예 | 비고 |
| --- | --- | --- |
| **SNS** | 이메일, Slack 연동, PagerDuty | 가장 흔함 |
| **Lambda** | 자동 조치 함수 실행 | 비동기 호출. 함수에 `lambda.alarms.cloudwatch.amazonaws.com` 호출 권한 필요 |
| **EC2** | stop, terminate, reboot, recover | ARN 예: `arn:aws:automate:ap-northeast-2:ec2:recover` |
| **Auto Scaling** | 스케일 아웃/인 | anomaly detection 알람에는 사용 불가 |
| Systems Manager | OpsItem 생성, Incident Manager 인시던트 | ALARM 상태에서만 |
| CloudWatch investigations | AI 기반 조사 시작 | ALARM 상태에서만 |

- **액션은 상태가 바뀔 때만 실행**된다. ALARM이 계속되어도 SNS 알림은 다시 오지 않는다. 예외는 Auto Scaling으로, 상태가 유지되는 동안 1분마다 반복된다.
- 상태를 `ALARM`, `OK`, `INSUFFICIENT_DATA` 각각에 대해 다른 액션을 걸 수 있다(`--alarm-actions`, `--ok-actions`, `--insufficient-data-actions`). 복구 알림을 받으려면 OK 액션도 건다.
- 상태 변화는 액션과 별개로 **EventBridge 이벤트**로도 나간다.

## 알림 줄이기

### Composite alarm

출처: [Combining alarms](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-combining.html), [PutCompositeAlarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutCompositeAlarm.html)

여러 알람(자식 알람)의 상태를 규칙으로 조합해, **정말 사람이 봐야 할 때만** 알린다.

```
ALARM("api-5xx-high") AND ALARM("api-latency-p99-high")
ALARM("cpu-high") AND NOT ALARM("deployment-in-progress")
```

| 항목 | 내용 |
| --- | --- |
| 규칙 함수 | `ALARM()`, `OK()`, `INSUFFICIENT_DATA()` + `AND`, `OR`, `NOT`, 괄호 |
| 한도 | 규칙당 자식 알람 100개, 요소 500개 |
| 액션 | SNS, Lambda, OpsItem, Incident Manager만 가능(**EC2·Auto Scaling 불가**) |
| 범위 | 자식 알람은 같은 계정·리전(교차 계정 observability 모니터링 계정은 예외) |

흔한 패턴은 이렇다. 자식 알람에는 액션을 걸지 않고, composite alarm에만 SNS를 건다. 그러면 증상 하나하나가 아니라 "서비스 이상"이라는 결론 하나만 알림으로 온다.

### Actions suppressor

composite alarm에 `ActionsSuppressor` 알람을 지정하면, 그 알람이 ALARM인 동안 composite alarm의 액션이 실행되지 않는다. 예를 들어 "배포 중" 알람을 suppressor로 두면 배포 중의 일시적인 에러로는 호출되지 않는다. ([Alarm suppression](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-suppression.html))

### Mute rule

정해진 시간대(점검 시간 등)에 알람 액션을 끈다. `cron()` 또는 `at()`로 일정을 정하고, 지속 시간은 1분부터 15일까지 줄 수 있다. 감시는 계속되므로, 창이 끝났을 때 여전히 ALARM이면 그때 액션이 실행된다. EventBridge 이벤트는 muted 상태에서도 나간다. ([Alarm mute rules](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-mute-rules.html))

| 방법 | 언제 끝나나 | 쓰임새 |
| --- | --- | --- |
| `disable-alarm-actions` | `enable-alarm-actions`를 호출할 때까지 | 수동 작업. 다시 켜는 것을 잊기 쉬움 |
| Mute rule | 일정이 끝나면 자동 | 정기 점검, 배치 시간대 |
| Actions suppressor | suppressor 알람 상태에 따라 | 배포 중 자동 억제 |

## Anomaly detection 알람

출처: [Anomaly detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html), [Create anomaly detection alarm](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Anomaly_Detection_Alarm.html)

고정 임계값을 정하기 어려운 메트릭(요일·시간대마다 트래픽이 다른 경우 등)에 쓴다.

- 최대 2주치 데이터로 모델을 학습해 **예상 범위(band)**를 만든다. 새 모델의 band가 나타나기까지 최대 3시간이 걸린다.
- 비교 연산자는 `LessThanLowerOrGreaterThanUpperThreshold`(범위 밖), `GreaterThanUpperThreshold`(위로 벗어남), `LessThanLowerThreshold`(아래로 벗어남) 중에서 고른다.
- band 폭은 표준편차 배수로 정하며, 값이 클수록 band가 넓어진다(덜 민감).
- period는 1분 이상이어야 한다. 모델은 메트릭 하나·통계 하나 조합마다 따로 만들어진다.

```json
[
  { "Id": "m1", "MetricStat": { "Metric": { "Namespace": "AWS/ApplicationELB", "MetricName": "RequestCount", "Dimensions": [{ "Name": "LoadBalancer", "Value": "app/my-alb/50dc6c495c0c9188" }] }, "Period": 300, "Stat": "Sum" } },
  { "Id": "ad1", "Expression": "ANOMALY_DETECTION_BAND(m1, 2)" }
]
```

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name alb-requests-anomaly \
  --metrics file://anomaly.json \
  --threshold-metric-id ad1 \
  --comparison-operator LessThanLowerOrGreaterThanUpperThreshold \
  --evaluation-periods 3 --datapoints-to-alarm 3 \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:ops-alerts
```

위 JSON과 명령은 API 레퍼런스의 `Metrics`/`ThresholdMetricId` 구조를 따라 이 레포에서 작성했다(실행해 보지 않음).

## 사용법 / 예시

```bash
# 5분 평균 CPU가 5개 기간 중 3번 80%를 넘으면 알림, 복구되면 다시 알림
aws cloudwatch put-metric-alarm \
  --alarm-name cpu-high-i-0123 \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --statistic Average --period 300 \
  --evaluation-periods 5 --datapoints-to-alarm 3 \
  --threshold 80 --comparison-operator GreaterThanThreshold \
  --treat-missing-data missing \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:ops-alerts \
  --ok-actions arn:aws:sns:ap-northeast-2:111122223333:ops-alerts

# 5xx 에러: 에러가 없으면 데이터도 없으므로 notBreaching
aws cloudwatch put-metric-alarm \
  --alarm-name alb-5xx \
  --namespace AWS/ApplicationELB --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/my-alb/50dc6c495c0c9188 \
  --statistic Sum --period 60 \
  --evaluation-periods 5 --datapoints-to-alarm 3 \
  --threshold 10 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching

# composite alarm: 둘 다 문제일 때만, 배포 중에는 억제
aws cloudwatch put-composite-alarm \
  --alarm-name checkout-unhealthy \
  --alarm-rule 'ALARM("alb-5xx") AND ALARM("cpu-high-i-0123")' \
  --actions-suppressor deployment-in-progress \
  --actions-suppressor-wait-period 120 \
  --actions-suppressor-extension-period 180 \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:ops-alerts

# 현재 울리고 있는 알람
aws cloudwatch describe-alarms --state-value ALARM --query 'MetricAlarms[].AlarmName'

# 알림 경로 테스트: 상태를 강제로 바꿔 액션 실행
aws cloudwatch set-alarm-state \
  --alarm-name cpu-high-i-0123 --state-value ALARM --state-reason "SNS 경로 테스트"
```

- `set-alarm-state`로 바꾼 metric alarm은 다음 평가에서 보통 몇 초 안에 실제 상태로 돌아온다. composite alarm은 돌아온다는 보장이 없다. ([set-alarm-state](https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/set-alarm-state.html))
- 옵션 이름은 CLI 레퍼런스와 대조했지만, 명령 자체는 이 레포에서 조합한 예시다.

## 요금과 한도

출처: [CloudWatch pricing](https://aws.amazon.com/cloudwatch/pricing/)(us-east-1, 조사 기준일 값), [Quotas](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_limits.html)

| 항목 | 요금(us-east-1) |
| --- | --- |
| 표준 해상도 알람 | 알람 메트릭당 월 $0.10 |
| 고해상도 알람(period 60초 미만) | 알람 메트릭당 월 $0.30 |
| Composite alarm | 알람당 월 $0.50 |
| Anomaly detection 알람 | 수식의 메트릭 + 2개(band 위·아래)로 계산. 예: 메트릭 1개면 3 × $0.10 = 월 $0.30 |
| Metrics Insights 알람 | 분석한 메트릭당 $0.10 |
| Free tier | 알람 메트릭 10개(표준 알람만) |

- metric math 알람은 수식에 들어간 메트릭 수(최대 10개)만큼 과금된다.
- **알람 개수 자체에는 한도가 없다.** `PutMetricAlarm`·`PutCompositeAlarm`은 초당 3회, `DescribeAlarms`는 초당 9회다(증설 가능).
- 서울 리전 요금은 요금 페이지에서 확인한다.

## 이슈 / 안티패턴

| 안티패턴 | 문제 | 대안 |
| --- | --- | --- |
| 모든 알람을 M=N=1로 | 잠깐 튀는 값에도 울림(flapping, 알림 피로) | M out of N(예: 5개 중 3개) |
| 에러 메트릭에 `missing` | 평소에 `INSUFFICIENT_DATA` | `notBreaching` |
| 하트비트 알람에 `notBreaching` | 앱이 죽어 데이터가 끊기면 정상으로 보임 | `breaching` |
| 증상 알람마다 사람을 호출 | 하나의 장애에 알림 수십 개 | 자식 알람은 액션 없이, composite alarm만 호출 |
| ALARM 액션만 설정 | 복구됐는지 모름 | OK 액션도 설정 |
| 평균 지연 시간 알람 | 일부 사용자의 느린 요청이 가려짐 | `p99`·`p95` 알람 + 샘플 수 옵션 |
| `disable-alarm-actions` 후 잊음 | 장애가 나도 아무도 모름 | mute rule이나 suppressor로 자동 해제 |
| 알람 테스트를 안 함 | SNS 구독 미확인 등으로 알림 경로가 끊겨 있음 | `set-alarm-state`로 정기 테스트 |

## 출처

- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Alarms.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-evaluation.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarms-and-missing-data.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/percentiles-with-low-samples.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-evaluation-window.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-warm-up.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create-alarm-on-metric-math-expression.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/multi-time-series-alarm.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-actions.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-combining.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-suppression.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/alarm-mute-rules.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Anomaly_Detection.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Create_Anomaly_Detection_Alarm.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutMetricAlarm.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutCompositeAlarm.html
- https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/set-alarm-state.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_limits.html
- https://aws.amazon.com/cloudwatch/pricing/

---
*조사 기준일: 2026-10-14. 요금은 요금 페이지의 데이터 피드(2026-10-07 게시)에서 확인한 us-east-1 값이다. PromQL 알람과 로그 알람(`PutLogAlarm`)도 새로 생겼지만 이 문서 범위에서는 다루지 않았다.*
