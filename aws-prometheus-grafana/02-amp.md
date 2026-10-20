# Amazon Managed Service for Prometheus(AMP)

> AMP는 Prometheus와 호환되는 **서버리스 메트릭 저장소**다. **워크스페이스** 단위로 remote write를 받아 3개 AZ에 복제해 저장하고(기본 150일, 최대 3년), PromQL로 조회하며, **규칙(recording·alerting)과 Alertmanager**까지 제공한다. 쓰고 읽는 요청은 모두 **IAM SigV4 서명**이 필요하다. 가장 먼저 부딪히는 한도는 **활성 시계열 수**(기본 5,000만)와 **조회 한도**다.

## 이 문서에서 새로 나오는 용어

[01-overview.md](./01-overview.md)의 용어(시계열, 라벨, remote write, PromQL, recording·alerting rule, Alertmanager, 워크스페이스, SigV4)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 활성 시계열(active series) | 최근 2시간 안에 샘플이 들어온 시계열 |
| Rule groups namespace | AMP에 올리는 규칙 파일 하나의 단위 |
| Alertmanager definition | AMP Alertmanager의 설정 파일(라우팅·수신자·템플릿) |
| Receiver | Alertmanager가 알림을 보낼 대상(SNS, PagerDuty) |
| Label set limit | 특정 라벨 조합(예: `team=A`)별로 활성 시계열 수를 제한하는 설정 |
| Out-of-order 샘플 | 이미 들어온 것보다 이른 시각의 샘플 |
| `awscurl` | SigV4 서명을 붙여 HTTP 요청을 보내는 curl 계열 도구 |

## 워크스페이스

출처: [What is AMP](https://docs.aws.amazon.com/prometheus/latest/userguide/what-is-Amazon-Managed-Service-Prometheus.html), [Create a workspace](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-create-workspace.html), [Workspace configuration](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-workspace-configuration.html)

> AMP는 "a serverless, Prometheus-compatible monitoring service for container metrics"다.

| 항목 | 내용 |
| --- | --- |
| 단위 | 워크스페이스 = Prometheus 메트릭 저장·조회 공간. 리전 리소스, ID는 `ws-…` |
| 고가용성 | 수집한 데이터를 같은 리전의 **3개 AZ에 복제** |
| 보존 기간 | **기본 150일**, 최대 1,095일(3년)까지 조정 |
| 리전 | 서울 포함. 관리 엔드포인트 `aps.<region>.amazonaws.com`, 데이터 엔드포인트 `aps-workspaces.<region>.amazonaws.com` |
| 암호화 | 기본은 AWS 소유 키(무료). **고객 관리형 KMS 키는 워크스페이스 생성 때만 선택 가능**, 나중에 바꿀 수 없음 |
| 추가 설정 | out-of-order 허용 창(기본 60초, 최대 600초), 규칙 조회 지연(rule query offset) |

```bash
WS=$(aws amp create-workspace --alias prod-metrics --query workspaceId --output text)
aws amp describe-workspace --workspace-id "$WS" --query 'workspace.status.statusCode'   # CREATING → ACTIVE

# 보존 기간 1년으로
aws amp update-workspace-configuration --workspace-id "$WS" --retention-period-in-days 365
```

| 엔드포인트 | 주소 |
| --- | --- |
| Remote write | `https://aps-workspaces.<region>.amazonaws.com/workspaces/<ws-id>/api/v1/remote_write` |
| Query | `https://aps-workspaces.<region>.amazonaws.com/workspaces/<ws-id>/api/v1/query` |

## 권한: 모든 요청에 SigV4

출처: [Prometheus-compatible APIs](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-compatible-APIs.html), [AmazonPrometheusRemoteWriteAccess](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AmazonPrometheusRemoteWriteAccess.html)

| 관리형 정책 | 포함 권한 | 누구에게 |
| --- | --- | --- |
| `AmazonPrometheusRemoteWriteAccess` | `aps:RemoteWrite` | 수집기(ADOT, Prometheus agent) |
| `AmazonPrometheusQueryAccess` | `aps:QueryMetrics`, `aps:GetSeries`, `aps:GetLabels`, `aps:GetMetricMetadata` | AMG 워크스페이스, 조회 도구 |
| `AmazonPrometheusFullAccess` | `aps:*` 등 | 관리자 |

```bash
# 조회 (awscurl이 SigV4 서명을 붙여 줌)
awscurl -X POST --region ap-northeast-2 --service aps \
  "https://aps-workspaces.ap-northeast-2.amazonaws.com/workspaces/$WS/api/v1/query" \
  -d 'query=sum by (job) (up)'
```

운영에서는 관리형 정책 대신 `Resource`를 특정 워크스페이스 ARN으로 좁힌 정책을 쓰는 편이 좋다([aws-iam/06](../aws-iam/06-least-privilege.md)).

### 네트워크와 계정 간 접근

출처: [Interface VPC endpoints](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-and-interface-VPC.html), [put-resource-policy](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/amp/put-resource-policy.html)

| 상황 | 방법 |
| --- | --- |
| 인터넷 없이 remote write | VPC endpoint `com.amazonaws.<region>.aps-workspaces`(데이터) + **STS interface endpoint**(그리고 STS 리전 엔드포인트 사용). 관리 API는 `com.amazonaws.<region>.aps` |
| 다른 계정이 쓰고 읽기 | 워크스페이스에 **resource-based policy**(`put-resource-policy`). 고객 관리형 KMS 키를 쓰면 KMS grant도 필요 |
| 다른 계정 EKS를 관리형 수집기로 | 원본·대상 role을 `--role-configuration`으로 지정([03](./03-collection.md)) |

STS 엔드포인트가 왜 필요한지는 [aws-iam/04 STS 엔드포인트](../aws-iam/04-roles-sts.md#sts-엔드포인트)를 참고한다. 수집기가 role을 assume해 임시 자격증명을 받기 때문이다.

## 규칙(Ruler)

출처: [Ruler](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-Ruler.html), [Upload rules](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-rules-upload.html)

Prometheus와 **같은 규칙 파일 형식**을 쓴다. 규칙 파일 하나가 **rule groups namespace** 하나가 된다.

```yaml
groups:
  - name: checkout
    rules:
      - record: service:http_requests:rate5m
        expr: sum by (service) (rate(http_requests_total[5m]))
      - alert: HighErrorRate
        expr: |
          sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
            / sum by (service) (rate(http_requests_total[5m])) > 0.05
        for: 10m
        labels:
          severity: page
        annotations:
          summary: "{{ $labels.service }} 5xx 비율 5% 초과"
```

```bash
base64 -i rules.yaml -o rules.b64     # macOS. Linux는 base64 -w0 rules.yaml > rules.b64
aws amp create-rule-groups-namespace --workspace-id "$WS" --name checkout --data file://rules.b64
aws amp put-rule-groups-namespace   --workspace-id "$WS" --name checkout --data file://rules.b64   # 수정
aws amp describe-rule-groups-namespace --workspace-id "$WS" --name checkout
```

- `--data`는 **base64로 인코딩한 파일**이다(AWS CLI v2 기준 `file://`, v1은 `fileb://`).
- 한 그룹 안의 규칙은 위에서 아래로 실행되지만, 그룹·파일 사이의 순서는 보장되지 않는다.
- 워크스페이스당 규칙 2,000개, 최소 평가 간격 30초다(둘 다 증설 가능).
- `for: 10m`은 조건이 10분 동안 계속 참일 때만 알림을 낸다는 뜻이다. CloudWatch의 "N개 중 M개"와 비슷한 잡음 줄이기다.

## Alertmanager

출처: [Alert manager](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-alert-manager.html), [Receivers](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-alertmanager-receiver.html), [Receiver config](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-alertmanager-receiver-config.html)

```yaml
alertmanager_config: |
  route:
    receiver: default
    group_by: [alertname, service]
    group_wait: 30s
    repeat_interval: 4h
  receivers:
    - name: default
      sns_configs:
        - topic_arn: arn:aws:sns:ap-northeast-2:111122223333:prom-alerts
          sigv4:
            region: ap-northeast-2
          attributes:
            key: severity
            value: page
```

```bash
base64 -i alertmanager.yaml -o am.b64
aws amp create-alert-manager-definition --workspace-id "$WS" --data file://am.b64
```

| 항목 | 내용 |
| --- | --- |
| 수신자 | **SNS, PagerDuty**(PagerDuty 키는 Secrets Manager에 보관) |
| SNS 조건 | Standard 토픽만. `sigv4.region` 필수, `api_url`·`http_config` 사용 불가 |
| 기능 | 그룹화, 중복 제거, 라우팅, 묵음(silence), 억제(inhibition), 템플릿 |
| 한도 | 활성 알림 1,000, 묵음 1,000, 억제 규칙·템플릿·라우팅 노드 각 100 |

AMP 문서의 개요 페이지는 아직 "SNS만 지원"이라고 적혀 있고, 수신자 페이지는 PagerDuty도 안내한다. 문서끼리 다르니 적용 전에 확인한다(확인 필요). SNS 토픽 정책에서 AMP 서비스(`aps.amazonaws.com`)의 게시를 허용해야 한다(정확한 정책은 수신자 설정 문서를 참고).

## 한도

출처: [AMP quotas](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP_quotas.html)

| 항목 | 기본값 | 증설 |
| --- | --- | --- |
| **활성 시계열** | 워크스페이스당 5,000만(최대 15억) | 가능 |
| 수집 속도 | 워크스페이스당 초당 1,666,666 샘플(활성 시계열 한도의 1/30로 자동 조정) | 가능 |
| 시계열당 라벨 | 150개, 라벨 크기 합계 7 KB | 라벨 수만 가능 |
| 조회 | 24시간 구간당 샘플 5,000만, 시계열 1,200만, 5 GB | — |
| **조회 기간** | 최대 95일 | — |
| `QueryMetrics` / `RemoteWrite` API | 초당 300 / 3,000 | 가능 |
| 규칙 | 워크스페이스당 2,000, 최소 평가 간격 30초 | 가능 |
| 워크스페이스 | 리전당 25 | 가능 |
| 오래된 샘플 | 1시간보다 오래된 샘플은 거부 | — |

활성 시계열 용량은 사용량에 따라 자동으로 늘어난다(최근 30분 기준, 200만 미만이면 쓰로틀링 없음). 하지만 **갑자기 시계열이 폭증하면** 용량이 따라잡기 전에 쓰로틀링될 수 있다. 쓰로틀링된 샘플은 CloudWatch 메트릭 `DiscardedSamples`(reason `rate_limited`)로 보인다.

### Label set limit: 한 팀의 폭증이 전체를 막지 않게

```bash
aws amp update-workspace-configuration --workspace-id "$WS" \
  --limits-per-label-set '[{"labelSet":{"team":"payments"},"limits":{"maxSeries":1000000}}]'
```

`team=payments` 라벨의 시계열만 100만 개로 제한하고, 다른 팀은 영향을 받지 않게 한다. 위 JSON 형식은 CLI 레퍼런스의 구조를 따라 이 레포에서 작성했다(확인 필요).

## 이슈 / 주의

- **고카디널리티 라벨**(사용자 ID, 요청 ID, 전체 URL 경로)은 활성 시계열 한도와 비용을 동시에 터뜨린다. 수집 단계에서 라벨을 버리거나(relabel) 값을 묶는다([03](./03-collection.md)).
- **고객 관리형 KMS 키는 생성 때만** 정할 수 있다. 관리형 수집기와의 조합은 문서끼리 설명이 다르다(한쪽은 불가, 다른 쪽은 우회 방법 안내, 확인 필요).
- **HA로 Prometheus를 2대 돌리면서 중복 제거를 설정하지 않으면** 같은 샘플이 두 번 과금된다. ([Ingest with Prometheus](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-ingest-with-prometheus.html))
- **1시간보다 오래된 샘플은 버려진다.** 수집기가 오래 멈췄다가 밀린 데이터를 보내면 일부가 사라진다.
- **조회 한도**: 긴 기간·많은 시계열을 한 번에 그리는 대시보드는 한도에 걸린다. recording rule로 미리 집계한다.

## 출처

- https://docs.aws.amazon.com/prometheus/latest/userguide/what-is-Amazon-Managed-Service-Prometheus.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-create-workspace.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-workspace-configuration.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-compatible-APIs.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-and-interface-VPC.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/encryption-at-rest-Amazon-Service-Prometheus.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-Ruler.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-rules-upload.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-alert-manager.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-alertmanager-receiver.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-alertmanager-receiver-config.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP_quotas.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-ingest-with-prometheus.html
- https://docs.aws.amazon.com/aws-managed-policy/latest/reference/AmazonPrometheusRemoteWriteAccess.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/amp/index.html

---
*조사 기준일: 2026-10-20. 규칙·Alertmanager YAML은 문서 형식을 따라 이 레포에서 작성했다. CLI 예시는 실제 계정에서 실행하지 않았다.*
