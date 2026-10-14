# Logs

> CloudWatch Logs는 **log group 단위로 보존 기간·로그 클래스·암호화·필터를 설정**한다. 가장 먼저 할 일은 **보존 기간 설정**이다(기본값이 영구 보관). 로그를 숫자로 바꾸려면 **metric filter**, 다른 시스템으로 흘리려면 **subscription filter**(log group당 5개), 가끔 보는 로그는 수집 단가가 절반인 **Infrequent Access 클래스**를 쓴다. 단, 클래스는 생성 후 바꿀 수 없고 IA에서는 필터·Live Tail을 쓸 수 없다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md) 4단계(log event·stream·group, 보존 기간)와 [02-metrics.md](./02-metrics.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 수집(ingestion) | 로그를 CloudWatch Logs로 보내 저장하는 것. GB당 과금 |
| 로그 클래스(log class) | log group의 요금·기능 등급. Standard, Infrequent Access 등 |
| Vended logs | AWS 서비스가 대신 만들어 보내는 로그(VPC Flow Logs, Route 53 쿼리 로그 등). 구간별 할인 요금 |
| Filter pattern | 로그 줄에서 원하는 내용을 고르는 패턴 문법 |
| Kinesis Data Streams / Data Firehose | 데이터를 실시간으로 흘려보내는 스트리밍 서비스 / S3 등에 적재해 주는 전달 서비스 |
| KMS | 암호화 키를 관리하는 서비스([aws-iam/02](../aws-iam/02-policies.md) 참고) |
| PII | 이메일·전화번호처럼 개인을 식별할 수 있는 정보 |
| Field index | 특정 필드로 빠르게 찾도록 만드는 색인 |

## 기본 구조와 설정

출처: [Concepts](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogsConcepts.html)

| 설정(log group 단위) | 기본값 | 메모 |
| --- | --- | --- |
| 보존 기간 | **영구(Never Expire)** | 1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1096, 1827, 2192, 2557, 2922, 3288, 3653일 중 선택 |
| 로그 클래스 | Standard | **생성 후 변경 불가** |
| 암호화 | AWS 관리 암호화(AES-GCM 256) | KMS 고객 관리 키 선택 가능 |
| 삭제 보호 | 꺼짐 | 실수로 log group을 지우는 것을 막음 |

log group 하나에 log stream 수 제한은 없다.

### 보존 기간

```bash
aws logs put-retention-policy --log-group-name /aws/lambda/checkout --retention-in-days 30
aws logs delete-retention-policy --log-group-name /aws/lambda/checkout   # 다시 영구 보관
```

- 보존 기간이 지난 로그는 약 72시간 안에 삭제되지만, 삭제 대상으로 표시되는 즉시 저장 요금이 멈춘다. ([PutRetentionPolicy](https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_PutRetentionPolicy.html))
- Lambda 등 AWS 서비스가 자동으로 만드는 log group도 영구 보관으로 생긴다. IaC로 log group을 먼저 만들어 보존 기간을 정해 두는 것이 안전하다.

### 로그 클래스

출처: [Log classes](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatch_Logs_Log_Classes.html), 요금은 us-east-1 기준(서울)

| 항목 | Standard | Infrequent Access(IA) |
| --- | --- | --- |
| 수집 요금 | GB당 $0.50($0.76) | GB당 $0.25($0.38) |
| 저장 요금 | GB-월당 $0.03 | 같음 |
| Logs Insights | 전체 명령 | 대부분 지원(`pattern`, `diff` 등 일부 제외) |
| S3 export, KMS, 교차 계정, 마스킹 | 지원 | 지원 |
| **Metric filter, subscription filter** | 지원 | **미지원** |
| **Live Tail**, 이상 탐지, field index | 지원 | **미지원** |
| EMF, Container·Lambda Insights 수집 | 지원 | **미지원** |
| `GetLogEvents`/`FilterLogEvents` | 지원 | 미지원 |

공식 문서도 두 클래스의 차이를 "수집 비용뿐"이라고 설명하지만, 실제로는 위처럼 **기능 차이가 크다**. IA는 "알람이나 실시간 처리 없이 가끔 Logs Insights로 조회하는 로그"(감사 로그, 디버그 로그 등)에 맞다.

Lambda 로그를 S3나 Firehose로 보낼 때 쓰는 세 번째 클래스 `Delivery`도 있다. 보존 기간이 2일로 고정되고 Logs Insights는 쓸 수 없다.

### 저장 Intelligent Tiering

2026-07에 나온 기능으로, 계정·리전 단위로 켠다. 30일 동안 접근이 없는 로그는 IA 저장 계층(GB-월 $0.018)으로, 90일이 지나면 Archive Instant Access 계층($0.006)으로 자동으로 옮긴다. 조회하면 30일 동안 다시 Standard로 돌아온다. 켜는 비용과 계층 이동 비용은 없다. 이 저장 계층은 위의 로그 **클래스**(수집 등급)와는 다른 개념이다. ([Intelligent Tiering](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cwl_intelligent_tier.html))

```bash
aws logs put-storage-tier-policy --storage-tier INTELLIGENT_TIERING
```

이 명령은 연구 노트에서 확인한 형식이며, CLI 레퍼런스와 직접 대조하지 않았다(확인 필요).

### KMS 암호화

출처: [Encrypt log data with KMS](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/encrypt-log-data-kms.html)

```bash
aws logs create-log-group --log-group-name /app/payments \
  --kms-key-id arn:aws:kms:ap-northeast-2:111122223333:key/1234abcd-12ab-34cd-56ef-1234567890ab
aws logs associate-kms-key --log-group-name /app/payments --kms-key-id <key-arn>   # 기존 그룹
```

- 대칭 키만 쓸 수 있다. 키 정책에서 `logs.<region>.amazonaws.com`을 허용해야 한다.
- 키를 연결하거나 해제해도 반영까지 최대 5분이 걸리고, **새로 들어오는 로그에만** 적용된다.
- **키를 비활성화하거나 삭제하면 그 키로 암호화한 로그를 읽을 수 없다.**

### 로그 보내기: PutLogEvents

출처: [PutLogEvents](https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_PutLogEvents.html), [Quotas](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cloudwatch_limits_cwl.html)

| 제한 | 값 |
| --- | --- |
| 배치 크기 | 1,048,576바이트(메시지 UTF-8 바이트 + 이벤트당 26바이트) |
| 배치당 이벤트 | 10,000개 |
| 이벤트 하나 | 최대 1 MB |
| 시간 범위 | 배치 안의 이벤트가 24시간 안이어야 함. 14일보다 오래되거나 2시간 넘게 미래인 이벤트는 거부 |
| 처리량 | 계정당 5,000 TPS(증설 가능). 예전의 스트림당 5 TPS 제한은 없어짐 |
| Sequence token | 이제 무시됨. 같은 스트림에 병렬로 써도 됨 |

## Metric filter: 로그를 숫자로

출처: [Filter pattern syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/FilterAndPatternSyntax.html), [Metric filters](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html)

로그에서 패턴이 나올 때마다 메트릭 값을 발행한다. 그 메트릭에 알람을 걸 수 있다.

| 패턴 종류 | 예 | 뜻 |
| --- | --- | --- |
| 단어 | `ERROR` | `ERROR`가 들어간 줄 |
| 여러 단어 | `?ERROR ?WARN` | 둘 중 하나라도 |
| 제외 | `ERROR -HealthCheck` | `ERROR`는 있지만 `HealthCheck`는 없는 줄 |
| JSON | `{ $.level = "ERROR" }`, `{ $.latency > 500 }` | JSON 로그의 필드 조건. `&&`, `||` 가능 |
| 공백 구분 | `[ip, user, ts, request, status_code = 5*, bytes]` | 공백으로 나뉜 필드에 이름을 붙여 조건 |
| 정규식 | `%timeout|deadline%` | `%…%` 안에 정규식(log group당 5개 제한) |

```bash
aws logs put-metric-filter \
  --log-group-name /app/payments \
  --filter-name PaymentErrors \
  --filter-pattern '{ $.level = "ERROR" }' \
  --metric-transformations \
    metricName=PaymentErrorCount,metricNamespace=MyApp,metricValue=1,defaultValue=0
```

| 설정 | 뜻 |
| --- | --- |
| `metricValue` | 일치할 때 발행할 값. 고정값(1)이나 필드 값(`$.latency`, `$size`) |
| `defaultValue` | 로그는 들어왔지만 일치하는 줄이 없을 때 발행할 값(예: 0). **dimension과 함께 쓸 수 없음** |
| `dimensions` | filter당 최대 3개. **값마다 custom metric으로 과금** |

- metric filter는 **만든 이후의 로그에만** 적용된다. 과거 로그는 소급되지 않는다.
- log group당 100개까지 만들 수 있다(증설 불가). Standard 클래스에서만 쓸 수 있다.
- dimension 값이 짧은 시간 안에 1,000가지를 넘으면 CloudWatch Logs가 filter를 비활성화한다. ([MetricTransformation](https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_MetricTransformation.html))
- `defaultValue=0`을 주면 에러가 없을 때도 0이 기록되어, 알람의 missing data 문제가 줄어든다([03](./03-alarms.md#데이터가-빠졌을-때-treatmissingdata)).

## Subscription filter: 로그를 다른 곳으로

출처: [Subscriptions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Subscriptions.html)

| 목적지 | 쓰임새 |
| --- | --- |
| Kinesis Data Streams | 실시간 스트림 처리 |
| Data Firehose | S3·OpenSearch·외부 SaaS로 적재 |
| Lambda | 즉시 가공·알림 |
| OpenSearch Service | 검색·대시보드(별도 설정 방식) |

- log group당 5개, 그리고 계정 수준 subscription filter를 리전당 1개 둘 수 있다. Standard 클래스에서만 쓸 수 있다.
- 데이터는 base64 + gzip으로 전달되며, "최소 한 번" 전달이라 보통 3분 안에 도착한다.
- 재시도 가능한 오류는 24시간까지 재시도한다. 재시도할 수 없는 오류가 나면 filter가 약 10분간 비활성화되고, **그동안의 로그는 건너뛴다.**

## S3로 내보내기

출처: [Export to S3](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/S3Export.html), [S3 export tasks](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/S3ExportTasks.html)

```bash
aws logs create-export-task \
  --task-name payments-2026-10-13 \
  --log-group-name /app/payments \
  --from 1760313600000 --to 1760400000000 \
  --destination my-log-archive-bucket \
  --destination-prefix payments
```

- `--from`/`--to`는 **밀리초 단위 epoch**다.
- **계정당 동시에 하나의 export 작업만** 실행할 수 있다(증설 불가). 작업은 24시간 뒤 타임아웃된다.
- 로그가 export 가능해지기까지 최대 12시간이 걸린다. 버킷은 같은 리전이어야 하고, 버킷 정책에서 `logs.<region>.amazonaws.com`을 허용해야 한다.
- **AWS는 주기적 export를 지속적인 보관 수단으로 쓰지 말라**고 하고, subscription filter(Firehose)를 권장한다.

## Live Tail

새로 들어오는 로그를 실시간으로 보여 준다. 배포 직후 확인이나 장애 대응에 쓴다. ([Live Tail](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogs_LiveTail.html))

| 항목 | 내용 |
| --- | --- |
| 요금 | 분당 $0.01(us-east-1·서울), 월 1,800분 무료 |
| 제한 | 세션 최대 3시간, log group 10개, 계정당 동시 세션 15개 |
| 샘플링 | 일치 이벤트가 초당 500개를 넘으면 일부만 표시 |
| 클래스 | Standard만 |

```bash
aws logs start-live-tail --log-group-identifiers arn:aws:logs:ap-northeast-2:111122223333:log-group:/app/payments
aws logs tail /app/payments --follow --since 10m --filter-pattern ERROR   # 간단한 tail
```

`aws logs tail`은 기본으로 최근 10분을 보여 주고, `--since`에는 단위 하나만 쓸 수 있다(`5h30m`은 안 됨). ([logs tail](https://docs.aws.amazon.com/cli/latest/reference/logs/tail.html)) 이 명령이 Live Tail API를 쓰는지는 문서에 나와 있지 않다(확인 필요).

## 그 밖의 기능

| 기능 | 무엇 | 비용·주의 | 출처 |
| --- | --- | --- | --- |
| **Data protection** | 이메일·카드번호 같은 PII를 수집 시점에 마스킹. 원문 보기는 `logs:Unmask` 권한 필요 | 스캔 GB당 $0.12. 적용 **이전 로그는 마스킹 안 됨** | [Mask sensitive data](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/mask-sensitive-log-data.html) |
| **Field index** | 자주 찾는 필드에 색인을 만들어 쿼리 속도를 높이고 스캔량을 줄임. 최대 10,000개 log group 조회 가능 | 정책 생성 이후 로그만, 색인 30일 유지 | [Field indexing](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogs-Field-Indexing.html) |
| **Anomaly detection** | 최근 2주 패턴을 학습해 평소와 다른 로그 패턴을 알림 | 탐지기 생성 무료, 계정당 500개 | [Log anomaly detection](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/LogsAnomalyDetection.html) |
| **Transformation** | 수집 시점에 로그를 파싱·변환(JSON화 등) | Standard 수집 요금에 포함. 저장은 변환된 크기로 과금, 512 KB 넘는 이벤트는 변환 실패 | [Transformation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatch-Logs-Transformation.html) |
| **Centralization** | Organizations의 여러 계정·리전 로그를 한 곳으로 복사 | 새 데이터만, 요금은 확인 필요 | [Centralization](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogs_Centralization.html) |

## 이슈 / 안티패턴

| 안티패턴 | 문제 | 대안 |
| --- | --- | --- |
| 보존 기간을 기본값(영구)으로 둠 | 저장 비용이 계속 쌓임 | 모든 log group에 보존 기간 설정(IaC로 생성) |
| 디버그 로그를 운영에서 계속 수집 | 수집 GB당 비용이 가장 큼 | 로그 레벨 조정, 샘플링 |
| 필터가 필요한 로그를 IA로 생성 | metric filter·subscription을 못 씀, 클래스 변경 불가 | 용도별로 log group 분리 |
| metric filter dimension에 고유 값 사용 | custom metric 폭증, filter 비활성화 | dimension 없이 세거나 Logs Insights로 분석 |
| 주기적 export로 장기 보관 | 동시 1개 제한, 지연 최대 12시간 | Firehose subscription으로 S3에 적재 |
| KMS 키를 무심코 삭제 예약 | 그 키로 암호화한 로그를 영구히 못 읽음 | 키 삭제 전 연결된 log group 확인 |
| 로그에 PII를 그대로 기록 | 유출·규정 위반 | 앱에서 마스킹 + data protection 정책 |

## 출처

- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogsConcepts.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Working-with-log-groups-and-streams.html
- https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_PutRetentionPolicy.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatch_Logs_Log_Classes.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cwl_intelligent_tier.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/encrypt-log-data-kms.html
- https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_PutLogEvents.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/FilterAndPatternSyntax.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/MonitoringLogData.html
- https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_MetricTransformation.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/Subscriptions.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/S3Export.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/S3ExportTasks.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogs_LiveTail.html
- https://docs.aws.amazon.com/cli/latest/reference/logs/tail.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/mask-sensitive-log-data.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogs-Field-Indexing.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/LogsAnomalyDetection.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatch-Logs-Transformation.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CloudWatchLogs_Centralization.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cloudwatch_limits_cwl.html

---
*조사 기준일: 2026-10-14. 요금은 AWS Price List API(2026-10-07 게시) 값이다. CLI 예시는 실제 계정에서 실행하지 않았다.*
