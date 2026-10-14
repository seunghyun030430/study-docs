# Metrics

> CloudWatch 메트릭은 **namespace + 이름 + dimension 조합**으로 식별되는 시계열이다. **dimension 값 조합 하나하나가 별도 메트릭**(그리고 custom metric이면 별도 과금)이라는 점이 설계와 비용의 핵심이다. 데이터는 오래될수록 1분 → 5분 → 1시간 단위로 뭉쳐지며 최대 15개월 보관되고, 메트릭 자체는 삭제할 수 없다. 여러 메트릭을 계산하려면 **metric math**, 많은 메트릭을 한 번에 훑으려면 **Metrics Insights**(SQL)를 쓴다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)의 용어(메트릭, 데이터 포인트, namespace, custom metric 등)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Dimension | 메트릭에 붙는 `이름=값` 꼬리표(예: `InstanceId=i-0123…`). 메트릭 식별의 일부 |
| Cardinality(카디널리티) | dimension 값의 가짓수. 값이 많을수록 메트릭 수가 폭증 |
| Period | 통계를 계산하는 시간 단위(예: 60초마다 평균) |
| Statistic | 기간 안의 데이터 포인트를 요약하는 방법(평균, 합계, 최대, 백분위수 등) |
| Percentile(백분위수) | `p99`면 "하위 99%가 이 값 이하". 지연 시간처럼 꼬리가 중요한 값에 씀 |
| Resolution(해상도) | 데이터를 저장하는 최소 시간 단위. 표준 1분, 고해상도 1초 |
| Rollup | 오래된 데이터를 더 큰 시간 단위로 합쳐 저장하는 것 |
| EMF(Embedded Metric Format) | 로그 한 줄(JSON) 안에 메트릭 정의를 넣으면 CloudWatch Logs가 메트릭을 뽑아 주는 형식 |
| Metric math | 여러 메트릭을 수식으로 계산해 새 시계열을 만드는 기능 |
| Metrics Insights | 메트릭을 SQL 비슷한 문법으로 조회하는 기능 |

## 메트릭의 구조

출처: [CloudWatch concepts](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html)

```
Namespace:  AWS/EC2
Metric:     CPUUtilization
Dimensions: InstanceId=i-0123456789abcdef0
Unit:       Percent
Datapoints: (2026-10-14T00:00Z, 12.5), (2026-10-14T00:01Z, 13.1), ...
```

| 요소 | 규칙 |
| --- | --- |
| Namespace | 메트릭을 담는 칸. AWS 서비스는 `AWS/<서비스>`. 직접 만드는 namespace는 `AWS/`로 시작하면 안 됨. 최대 255자 |
| Metric name | namespace 안의 이름 |
| Dimension | 메트릭당 최대 30개. **이름·값 조합이 다르면 다른 메트릭**. custom metric은 dimension을 가로질러 자동 합산하지 않음 |
| Unit | `Percent`, `Bytes`, `Seconds`, `Count` 등. 기본 `None`. 단위가 다르면 따로 집계 |
| Timestamp | 2주 전부터 2시간 뒤까지 허용. UTC 권장 |
| 리전 | 메트릭은 그 리전에만 존재 |

**메트릭은 삭제할 수 없다.** 새 데이터가 15개월 동안 없으면 자동으로 만료된다. 최근 2주간 데이터가 없는 메트릭은 콘솔과 `list-metrics`에 보이지 않는다.

### Dimension이 비용과 설계를 정한다

```
MyApp / Latency / Service=checkout, Env=prod      → 메트릭 1
MyApp / Latency / Service=checkout, Env=dev       → 메트릭 2
MyApp / Latency / Service=checkout                → 메트릭 3 (Env 없는 조합은 또 다른 메트릭)
```

`Env` 없이 "checkout 전체 지연 시간"을 보고 싶으면, 그 조합으로도 **따로 발행**해야 한다. CloudWatch가 custom metric을 알아서 합쳐 주지 않기 때문이다.

`UserId`, `RequestId`처럼 값이 무한히 많은 것을 dimension으로 쓰면 **값마다 custom metric이 생겨 비용이 폭증**한다. 이런 값은 로그에 남기고 Logs Insights로 분석한다(→ [06](./06-cost-issues.md)).

## 해상도와 보존 기간

### 해상도

| 종류 | 최소 단위 | 대상 |
| --- | --- | --- |
| 표준 해상도 | 1분 | AWS 서비스 메트릭 기본값, custom metric 기본값 |
| **고해상도** | 1초 | custom metric을 `StorageResolution=1`로 보낸 경우. 알람은 10초·30초 period 가능(더 비쌈) |

period로 쓸 수 있는 값은 1, 5, 10, 30초 또는 60의 배수다. 1분 미만 period는 고해상도 메트릭에서만 의미가 있다. (`GetMetricStatistics` 레퍼런스는 20초도 허용 값으로 적고 있어 문서 간 차이가 있다.)

### 보존 기간과 rollup

| 데이터 포인트 단위 | 보존 기간 |
| --- | --- |
| 60초 미만(고해상도) | 3시간 |
| 60초 | 15일 |
| 300초(5분) | 63일 |
| 3600초(1시간) | 455일(15개월) |

데이터는 시간이 지나면 더 큰 단위로 합쳐진다. 1분 데이터는 15일 뒤 5분 단위로, 63일 뒤 1시간 단위로 바뀐다. 그래서 **두 달 전 장애를 1분 단위로 다시 볼 수는 없다.** 장기 보관이 필요한 세밀한 데이터는 로그나 별도 저장소에 남긴다.

### EC2 기본 모니터링 vs 상세 모니터링

출처: [Manage detailed monitoring](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/manage-detailed-monitoring.html)

| 항목 | 기본(basic) | 상세(detailed) |
| --- | --- | --- |
| 주기 | 5분(상태 검사 메트릭은 1분) | 1분 |
| 요금 | 무료 | 메트릭당 과금 |
| 켜는 법 | 기본값 | `aws ec2 monitor-instances --instance-ids i-...` 또는 실행 시 `--monitoring Enabled=true` |

Auto Scaling이 빠르게 반응해야 하는 인스턴스만 상세 모니터링을 켠다. 모든 인스턴스에 켜면 비용이 늘어난다.

## 통계(statistic)

출처: [Statistics definitions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Statistics-definitions.html)

| 통계 | 뜻 | 쓰임새 |
| --- | --- | --- |
| `SampleCount` | 데이터 포인트 개수 | 요청 수 |
| `Sum` | 합계 | 에러 건수, 바이트 수 |
| `Average` | 평균(Sum / SampleCount) | CPU 사용률 |
| `Minimum` / `Maximum` | 최솟값 / 최댓값 | 순간 최고치 |
| `pNN`(예: `p99`) | 백분위수 | **지연 시간**. 평균은 느린 요청을 가림 |
| `TM`(trimmed mean, 예: `tm90`) | 위아래 극단값을 잘라낸 평균 | 이상치에 흔들리지 않는 평균 |
| `IQM`, `WM`, `PR`, `TC`, `TS` | 사분위 평균, winsorized 평균, 백분위 순위, 절삭 개수·합계 | 고급 분석 |

- 백분위수 계열 통계는 원본 데이터 포인트가 있어야 계산되고, **음수 값이 하나라도 있으면 쓸 수 없다.**
- 미리 집계한 값(statistic set: Min·Max·Sum·SampleCount)으로 보내면 백분위수는 제한적으로만 계산된다. 지연 시간을 `p99`로 보려면 원본 값을 보내야 한다.

## Custom metric 보내기

### PutMetricData

출처: [PutMetricData](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutMetricData.html)

| 제한 | 값 |
| --- | --- |
| 요청 크기 | 1 MB(gzip 압축 가능) |
| 요청당 메트릭 | 최대 1,000개 |
| 메트릭당 값 배열(`Values`/`Counts`) | 최대 150개 |
| 값 범위 | ±2^360, NaN·Infinity 불가 |
| 새 메트릭이 목록에 보이기까지 | 최대 15분 |

```bash
aws cloudwatch put-metric-data \
  --namespace MyApp \
  --metric-name Latency \
  --unit Milliseconds \
  --value 123 \
  --dimensions Service=checkout,Env=prod \
  --storage-resolution 1
```

`put-metric-data`의 `--dimensions`는 `Name=Value,...` 형식이고, 조회 명령(`get-metric-statistics`)은 `Name=...,Value=...` 형식이라 서로 다르다. ([put-metric-data](https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-data.html))

### EMF — 로그로 메트릭 보내기

출처: [EMF specification](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format_Specification.html)

애플리케이션이 아래 형식의 JSON 로그를 CloudWatch Logs에 쓰면, CloudWatch가 그 안의 값을 메트릭으로 뽑아 준다. 메트릭 API를 따로 호출할 필요가 없고, 원본 로그(`requestId` 등 상세 정보)도 그대로 남는다. Lambda처럼 로그가 자동으로 CloudWatch Logs로 가는 환경에서 특히 편하다.

```json
{
  "_aws": {
    "Timestamp": 1574109732004,
    "CloudWatchMetrics": [
      {
        "Namespace": "lambda-function-metrics",
        "Dimensions": [["functionVersion"]],
        "Metrics": [{ "Name": "time", "Unit": "Milliseconds", "StorageResolution": 60 }]
      }
    ]
  },
  "functionVersion": "$LATEST",
  "time": 100,
  "requestId": "989ffbf8-9ace-4817-a57c-e4dd734019ee"
}
```

| 부분 | 뜻 |
| --- | --- |
| `_aws.CloudWatchMetrics[].Namespace` | 메트릭 namespace |
| `Dimensions` | dimension으로 쓸 **필드 이름** 목록. 여기서는 `functionVersion` |
| `Metrics[].Name` | 메트릭 값으로 쓸 필드 이름. 여기서는 `time` |
| `requestId` | dimension이 아니므로 **메트릭이 되지 않고 로그에만** 남음 |

- 이벤트 하나는 최대 1 MB이고, 메트릭 정의는 최대 100개, dimension set당 key는 최대 30개다.
- 전달은 "최소 한 번(at least once)"이라 가끔 값이 중복될 수 있다.
- AWS 문서도 `requestId` 같은 고유 값을 dimension에 넣으면 값마다 custom metric이 생긴다고 경고한다.
- EMF는 **로그 수집 요금 + custom metric 요금**이 둘 다 든다.

## 메트릭 조회와 계산

### GetMetricData vs GetMetricStatistics

| 항목 | `GetMetricData` | `GetMetricStatistics` |
| --- | --- | --- |
| 한 번에 조회 | 최대 500개 쿼리 | 메트릭 1개 |
| 반환 데이터 포인트 | 기본 최대 100,800개 | 최대 1,440개 |
| metric math·Metrics Insights | 지원 | 미지원 |

출처: [GetMetricData](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_GetMetricData.html), [GetMetricStatistics](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_GetMetricStatistics.html)

기능 차이 때문에 새 코드에서는 `GetMetricData`를 쓰는 편이 낫다(AWS가 명시적으로 권장한 문구는 찾지 못함). 단, `GetMetricData`는 조회한 메트릭 수만큼 과금되므로 외부 모니터링 도구가 자주 호출하면 비용이 커진다(→ [06](./06-cost-issues.md)).

```bash
aws cloudwatch list-metrics --namespace AWS/EC2 --metric-name CPUUtilization

aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0123456789abcdef0 \
  --start-time 2026-10-13T00:00:00Z --end-time 2026-10-14T00:00:00Z \
  --period 3600 --statistics Maximum
```

### Metric math

여러 메트릭을 계산해 새 시계열을 만든다. ([Metric math](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/using-metric-math.html))

```json
[
  { "Id": "errors",   "MetricStat": { "Metric": { "Namespace": "AWS/ApplicationELB", "MetricName": "HTTPCode_Target_5XX_Count", "Dimensions": [{ "Name": "LoadBalancer", "Value": "app/my-alb/50dc6c495c0c9188" }] }, "Period": 60, "Stat": "Sum" }, "ReturnData": false },
  { "Id": "requests", "MetricStat": { "Metric": { "Namespace": "AWS/ApplicationELB", "MetricName": "RequestCount",              "Dimensions": [{ "Name": "LoadBalancer", "Value": "app/my-alb/50dc6c495c0c9188" }] }, "Period": 60, "Stat": "Sum" }, "ReturnData": false },
  { "Id": "error_rate", "Expression": "100 * FILL(errors, 0) / requests", "Label": "5xx error rate (%)" }
]
```

```bash
aws cloudwatch get-metric-data \
  --metric-data-queries file://error-rate.json \
  --start-time 2026-10-14T00:00:00Z --end-time 2026-10-14T01:00:00Z
```

| 함수(예) | 용도 |
| --- | --- |
| `SUM`, `AVG`, `MAX` | 여러 시계열 합치기 |
| `RATE`, `DIFF` | 변화율, 차이 |
| `IF(조건, 참, 거짓)` | 조건 계산 |
| `FILL(m, 0 / REPEAT / LINEAR)` | 빈 구간 채우기(에러가 없을 때 데이터가 없는 경우 등) |
| `SEARCH(...)` | 조건에 맞는 메트릭을 찾아 한꺼번에 그리기. **알람에는 쓸 수 없음** |
| `ANOMALY_DETECTION_BAND` | 이상 탐지 범위(→ [03](./03-alarms.md)) |
| `SERVICE_QUOTA` | 사용량을 한도와 비교 |

- 함수 이름은 대문자, 수식 ID(`Id`)는 소문자로 시작해야 한다.
- 그래프 하나에 메트릭과 수식은 최대 500개다.
- 위 JSON은 이 레포에서 작성한 예시이며, 형식은 `GetMetricData` API의 `MetricDataQuery` 구조를 따랐다.

### Metrics Insights

수천 개 메트릭을 SQL 비슷한 문법으로 한 번에 집계한다. ([Query language](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-metrics-insights-querylanguage.html), [Limits](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-metrics-insights-limits.html))

```
SELECT AVG(CPUUtilization)
FROM SCHEMA("AWS/EC2", InstanceId)
GROUP BY InstanceId
ORDER BY AVG() DESC
LIMIT 10
```

"CPU를 가장 많이 쓰는 인스턴스 10개"를 뜻한다.

| 항목 | 내용 |
| --- | --- |
| 집계 함수 | `AVG`, `COUNT`, `MAX`, `MIN`, `SUM` |
| 필터 | `WHERE`에서 dimension, 태그(`tag.<key>`), `AWS.AccountId` 사용 가능 |
| 조회 범위 | 그래프는 최근 2주 데이터. 알람 평가는 최근 3시간 |
| 한도 | 쿼리당 메트릭 10,000개 처리, 결과 시계열 500개, 1분 단위로 집계(고해상도 미지원) |

## 이슈 / 주의

- **고카디널리티 dimension**(사용자 ID, 요청 ID, 타임스탬프 등)은 custom metric 비용 폭증의 1순위 원인이다.
- **메트릭은 지울 수 없다.** 잘못 만든 메트릭은 데이터를 그만 보내고 15개월 뒤 만료되기를 기다려야 한다(과금은 데이터를 보낸 기간만큼).
- **평균만 보지 않는다.** 지연 시간은 `p99`·`p95`로 본다. 평균이 정상이어도 일부 사용자는 매우 느릴 수 있다.
- **rollup 때문에 오래된 데이터는 거칠다.** 사후 분석에 1분 데이터가 필요하면 15일 안에 확인하거나 별도로 저장한다.
- **24시간보다 오래된 타임스탬프로 보낸 데이터는 조회까지 48시간 이상 걸릴 수 있다.** ([PutMetricData](https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutMetricData.html))

## 출처

- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_concepts.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Statistics-definitions.html
- https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/manage-detailed-monitoring.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_PutMetricData.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/CloudWatch_Embedded_Metric_Format_Specification.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_GetMetricData.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/API_GetMetricStatistics.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/using-metric-math.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-metrics-insights-querylanguage.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch-metrics-insights-limits.html
- https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/put-metric-data.html
- https://docs.aws.amazon.com/cli/latest/reference/cloudwatch/get-metric-statistics.html

---
*조사 기준일: 2026-10-14. metric math JSON과 Metrics Insights의 `GROUP BY ... ORDER BY ... LIMIT` 조합 예시는 이 레포에서 작성했고 실제 계정에서 실행하지는 않았다.*
