# Logs Insights

> Logs Insights는 log group을 **파이프(`|`)로 이어지는 명령**(`fields` → `filter` → `stats` → `sort` → `limit`)으로 조회하는 쿼리 기능이다. 요금은 **스캔한 데이터 GB당**($0.005, us-east-1)이므로, **시간 범위를 좁히고 필요한 log group만 고르는 것**이 속도와 비용을 함께 줄인다. 기본 언어(Logs Insights QL) 말고 OpenSearch PPL·SQL도 쓸 수 있다.

## 이 문서에서 새로 나오는 용어

[04-logs.md](./04-logs.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 쿼리(query) | 데이터를 고르고 집계하는 질의문 |
| 파이프(`\|`) | 앞 명령의 결과를 다음 명령으로 넘기는 기호 |
| Discovered field | Logs Insights가 로그에서 자동으로 뽑아 주는 필드(`@timestamp`, `@message` 등) |
| 집계(aggregation) | 여러 줄을 묶어 개수·평균 등을 계산하는 것 |
| `bin()` | 시간을 일정 간격(예: 5분)으로 묶는 함수 |
| 스캔량 | 쿼리가 읽은 원본(압축 전) 로그 크기. 요금 기준 |
| PPL / SQL | OpenSearch의 파이프 쿼리 언어 / 표준 SQL 문법 |

## 쿼리의 모양

```
fields @timestamp, @message
| filter @message like /ERROR/
| sort @timestamp desc
| limit 20
```

"ERROR가 들어간 로그를 최신순으로 20개"를 뜻한다. 명령은 위에서 아래로 차례대로 적용된다.

### 자동으로 생기는 필드

출처: [Discovered fields](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_AnalyzeLogData-discoverable-fields.html)

| 필드 | 뜻 |
| --- | --- |
| `@timestamp` | 로그 이벤트 시각 |
| `@message` | 원본 메시지 |
| `@logStream`, `@log` | log stream 이름, `계정ID:log group` |
| `@ingestionTime` | CloudWatch가 받은 시각 |
| Lambda 전용 | `@requestId`, `@duration`, `@billedDuration`, `@maxMemoryUsed`, `@memorySize`, `@type`(`REPORT` 등) |
| JSON 로그 | 필드를 자동으로 추출(최대 200개). 예: `level`, `user.id` |

필드 자동 추출은 Standard 클래스에서만 동작한다.

## 주요 명령

출처: [Query syntax](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html)

| 명령 | 하는 일 | 예 |
| --- | --- | --- |
| `fields` | 보여 줄 필드 선택, 계산 필드 생성 | `fields @timestamp, level, @message` |
| `filter` | 조건에 맞는 줄만 | `filter level = "ERROR" and @duration > 1000` |
| `parse` | 메시지에서 값을 뽑아 필드로 | `parse @message "user=*, latency=*" as user, latency` |
| `stats` | 집계 | `stats count(*) by bin(5m)` |
| `sort` | 정렬 | `sort @timestamp desc` |
| `limit` | 개수 제한 | `limit 50` |
| `dedup` | 중복 제거 | `dedup @logStream` |
| `display` | 최종 출력 필드 | `display @timestamp, user` |
| `pattern` | 비슷한 로그를 패턴별로 묶음 | `pattern @message` |
| `diff` | 이전 기간과 패턴 비교 | — |
| `unmask` | 마스킹된 값 보기(`logs:Unmask` 권한 필요) | — |
| `filterIndex` | field index를 써서 빠르게 거르기 | — |

최근에는 `join`, `lookup`, 서브쿼리, `where` 같은 명령도 추가되었다. IA 클래스에서는 `pattern`, `diff`, `unmask`, `filterIndex`를 쓸 수 없다.

### 자주 쓰는 함수

출처: [Stats functions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-Stats.html), [Operations and functions](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-operations-functions.html)

| 함수 | 뜻 |
| --- | --- |
| `count(*)`, `countDistinct(f)` | 개수, 서로 다른 값 개수(값이 매우 많으면 근삿값) |
| `values(f)`, `topk(k, f)` | 서로 다른 값 목록, 가장 자주 나온 값 k개 |
| `avg`, `sum`, `min`, `max` | 평균·합계·최소·최대 |
| `pct(f, 95)` | 95번째 백분위수 |
| `earliest(f)`, `latest(f)` | 가장 이른 / 늦은 값 |
| `bin(5m)` | 5분 단위로 묶기 |
| `ispresent(f)` | 필드가 있는지 |
| `like /정규식/` | 패턴 일치 |

- `bin()`에서 `s`·`m` 단위는 60까지만 쓸 수 있다. 5분은 `bin(300s)`가 아니라 `bin(5m)`로 쓴다.
- `stats`는 쿼리 하나에 10개까지 쓸 수 있다.

## 자주 쓰는 쿼리

공식 예시 페이지의 쿼리이거나, 그 문법을 조합해 만든 쿼리다. ([Sample queries](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-examples.html))

**최근 로그 25개** (공식 예시)

```
fields @timestamp, @message | sort @timestamp desc | limit 25
```

**시간대별 Exception 수** (공식 예시)

```
filter @message like /Exception/
| stats count(*) as exceptionCount by bin(1h)
| sort exceptionCount desc
```

**Lambda 실행 시간 추이** (공식 예시)

```
filter @type = "REPORT"
| stats avg(@duration), max(@duration), min(@duration) by bin(5m)
```

**Lambda p95 실행 시간과 메모리 여유** (조합 예시)

```
filter @type = "REPORT"
| stats pct(@duration, 95) as p95ms,
        max(@maxMemoryUsed / 1000 / 1000) as maxMemMB,
        max(@memorySize / 1000 / 1000) as allocMB
  by bin(1h)
```

**JSON 로그에서 에러가 가장 많은 API** (조합 예시)

```
filter level = "ERROR"
| stats count(*) as errors by path
| sort errors desc
| limit 10
```

**문자열 로그에서 값 뽑기** (공식 예시)

```
parse @message "user=*, method:*, latency := *" as @user, @method, @latency
| stats avg(@latency) by @method, @user
```

**VPC Flow Logs: 트래픽이 많은 IP 쌍** (공식 예시)

```
stats sum(packets) as packetsTransferred by srcAddr, dstAddr
| sort packetsTransferred desc
| limit 15
```

"조합 예시"는 공식 문법으로 이 레포에서 만든 것이며, 실제 데이터에서 실행해 보지 않았다.

## CLI로 실행하기

```bash
QUERY_ID=$(aws logs start-query \
  --log-group-names /aws/lambda/checkout \
  --start-time $(date -u -v-1H +%s) --end-time $(date -u +%s) \
  --query-string 'filter @type = "REPORT" | stats pct(@duration, 95) by bin(5m)' \
  --query queryId --output text)

aws logs get-query-results --query-id "$QUERY_ID"
```

- `start-query`의 시각은 **초 단위 epoch**다(S3 export는 밀리초라서 서로 다르다).
- `date -v-1H`는 macOS 문법이다. Linux에서는 `date -u -d '1 hour ago' +%s`를 쓴다.
- 쿼리는 비동기로 실행된다. `get-query-results`의 `status`가 `Complete`가 될 때까지 다시 조회한다.
- OpenSearch 언어로 실행하려면 `--query-language SQL` 또는 `PPL`을 쓴다. ([StartQuery](https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_StartQuery.html))

옵션 형식은 StartQuery API 예시를 바탕으로 했고, CLI 레퍼런스와 하나씩 대조하지는 않았다(확인 필요).

## 한도와 비용

출처: [Analyzing log data](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html), [Quotas](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cloudwatch_limits_cwl.html), [Pricing](https://aws.amazon.com/cloudwatch/pricing/)

| 항목 | 값 |
| --- | --- |
| 요금 | 스캔 GB당 $0.005(서울 $0.0076). 월 5 GB 무료. 언어와 무관하게 **압축 전 크기** 기준 |
| 동시 쿼리 | Logs Insights QL 100개(대시보드·예약 쿼리 포함), PPL·SQL 15개 |
| 타임아웃 | 60분 |
| 결과 보관 | 7일 |
| 한 번에 지정할 log group | 50개(field index·`SOURCE`로 최대 10,000개) |
| 결과 행 수 | `limit` 최대 100,000(QL만). `GetQueryResults` 한 번에 최대 10,000행 |
| 쿼리 문자열 | 10,000자 |
| API | `StartQuery`·`GetQueryResults` 각 초당 10회 |

### 비용을 줄이는 습관

1. **시간 범위를 좁힌다.** 스캔량은 고른 기간의 로그 전체다. 장애 시각 앞뒤 30분처럼 좁게 잡는다.
2. **log group을 최소로 고른다.** 여러 서비스를 한꺼번에 고르면 그만큼 스캔한다.
3. **자주 찾는 필드에는 field index를 만든다.** 스캔량이 줄어든다([04](./04-logs.md#그-밖의-기능)).
4. **`filter`를 앞에 두는 것은 결과를 빨리 줄이는 데는 도움이 되지만, 과금되는 스캔량을 줄이지는 않는다**(확인 필요). 스캔량은 시간 범위·log group·인덱스로 줄인다.
5. **대시보드 위젯의 Logs Insights 쿼리는 열 때마다 실행되어 과금된다.** 자주 보는 값은 metric filter로 메트릭을 만들어 그 메트릭을 그린다.

## 이슈 / 주의

- **로그 형식이 쿼리 편의를 정한다.** JSON 구조화 로그는 필드가 자동으로 잡히고, 자유 텍스트는 매번 `parse`해야 한다.
- **IA 클래스 log group에서는 일부 명령이 안 된다**(`pattern`, `diff` 등).
- **결과는 7일 뒤 사라진다.** 보고서용 결과는 따로 저장한다.
- **2018-11-05 이전에 들어온 로그는 조회할 수 없다.**

## 출처

- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-Stats.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-operations-functions.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_AnalyzeLogData-discoverable-fields.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/CWL_QuerySyntax-examples.html
- https://docs.aws.amazon.com/AmazonCloudWatchLogs/latest/APIReference/API_StartQuery.html
- https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cloudwatch_limits_cwl.html
- https://aws.amazon.com/cloudwatch/pricing/

---
*조사 기준일: 2026-10-14. 요금은 AWS Price List API(2026-10-07 게시) 값이다.*
