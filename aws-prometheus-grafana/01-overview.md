# AMP·AMG 개요

> **Prometheus**는 메트릭을 **라벨이 붙은 시계열**로 모으고 **PromQL**로 조회하는 오픈 소스 모니터링 도구이고, **Grafana**는 여러 데이터를 대시보드로 보여 주는 오픈 소스 도구다. AWS는 이 둘을 관리형으로 제공한다. **AMP**(Amazon Managed Service for Prometheus)는 메트릭 **저장·조회·알림**을, **AMG**(Amazon Managed Grafana)는 **시각화·알림 화면**을 맡는다. 수집은 별도의 수집기(AWS 관리형 수집기, ADOT, Prometheus agent)가 **remote write**로 AMP에 보낸다. 컨테이너·쿠버네티스 메트릭을 Prometheus 생태계 그대로 쓰고 싶을 때 고르는 조합이다.

이 문서는 처음 보는 사람을 기준으로 용어를 한 단계씩 쌓아 간다. 처음 나오는 용어는 **굵게** 표시하고 바로 뜻을 적는다. 메트릭·알람의 기본 개념은 [aws-cloudwatch/01](../aws-cloudwatch/01-overview.md)을 참고한다. 모든 용어는 맨 아래 [용어 정리](#용어-정리)에 모아 두었다.

## 1단계: Prometheus의 데이터 모델

출처: [Data model](https://prometheus.io/docs/concepts/data_model/)

Prometheus에서 메트릭 하나는 **메트릭 이름 + 라벨(label)**으로 식별되는 **시계열(time series)**이다.

```
http_requests_total{service="checkout", method="POST", status="500"}   1027
└─ 메트릭 이름 ─┘ └──────────────── 라벨 ────────────────┘   └ 값 ┘
```

- **라벨**은 `이름="값"` 꼬리표다. CloudWatch의 dimension과 비슷하다.
- **라벨 값 조합 하나하나가 별도 시계열**이다. `status`가 200, 404, 500이면 시계열 3개가 생긴다.
- 라벨 값의 가짓수를 **카디널리티(cardinality)**라고 한다. 사용자 ID처럼 값이 무한히 많은 것을 라벨로 쓰면 시계열이 폭증한다. 이 점은 CloudWatch의 고카디널리티 dimension 문제([aws-cloudwatch/02](../aws-cloudwatch/02-metrics.md#dimension이-비용과-설계를-정한다))와 같다.

### 메트릭 타입

출처: [Metric types](https://prometheus.io/docs/concepts/metric_types/)

| 타입 | 뜻 | 예 |
| --- | --- | --- |
| **Counter** | 계속 늘어나기만 하는 값(재시작하면 0) | 누적 요청 수, 누적 에러 수 |
| **Gauge** | 오르내리는 현재 값 | 메모리 사용량, 큐 길이 |
| **Histogram** | 값을 구간(bucket)별로 세어 둔 것. 백분위수 계산에 씀 | 응답 시간 분포 |
| Summary | 앱 쪽에서 백분위수를 미리 계산한 것 | — |

Counter 값 자체(누적 1027)는 의미가 적다. 보통 **초당 증가율**로 바꿔서 본다(아래 PromQL).

## 2단계: 수집 방식 — scrape와 remote write

출처: [Overview](https://prometheus.io/docs/introduction/overview/), [Configuration](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)

```
애플리케이션 ── /metrics(HTTP)로 현재 값 노출
      ▲
      │ scrape (기본 1분마다 가져감, pull 방식)
Prometheus / 수집기 ── remote write ──▶ 저장소(AMP)
```

- **Scrape**: Prometheus가 애플리케이션의 `/metrics` 주소를 주기적으로 **가져오는(pull)** 방식이다. 앱이 보내는 게 아니라 Prometheus가 가지러 간다.
- **Exporter**: 스스로 `/metrics`를 내지 못하는 대상(노드, DB 등)의 값을 Prometheus 형식으로 바꿔 주는 프로그램이다(예: node-exporter).
- **Remote write**: 수집한 데이터를 **원격 저장소로 밀어 보내는** 표준 방식이다. AMP는 이 방식으로만 데이터를 받는다.
- **Agent 모드**: Prometheus에서 조회·알림·로컬 저장을 끄고 **수집 + remote write만** 하게 한 가벼운 모드다.

## 3단계: PromQL — 조회 언어

출처: [Functions](https://prometheus.io/docs/prometheus/latest/querying/functions/), [Operators](https://prometheus.io/docs/prometheus/latest/querying/operators/)

```promql
# 서비스별 초당 요청 수 (최근 5분 평균)
sum by (service) (rate(http_requests_total[5m]))

# 서비스별 5xx 비율
sum by (service) (rate(http_requests_total{status=~"5.."}[5m]))
  / sum by (service) (rate(http_requests_total[5m]))

# p99 응답 시간 (histogram)
histogram_quantile(0.99, sum by (le, service) (rate(http_request_duration_seconds_bucket[5m])))
```

| 구성 | 뜻 |
| --- | --- |
| `{status=~"5.."}` | 라벨로 거르기(`=~`는 정규식) |
| `[5m]` | 최근 5분 구간 |
| `rate()` | counter의 **초당 증가율**(재시작으로 0이 된 것도 보정) |
| `sum by (service)` | service 라벨별로 합치기 |
| `histogram_quantile(0.99, …)` | histogram으로 99번째 백분위수 계산 |

## 4단계: 규칙과 알림

출처: [Recording rules](https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/), [Alerting rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/), [Alertmanager](https://prometheus.io/docs/alerting/latest/overview/)

| 개념 | 뜻 |
| --- | --- |
| **Recording rule** | 무거운 PromQL을 미리 계산해 새 시계열로 저장(대시보드가 빨라짐) |
| **Alerting rule** | PromQL 조건이 일정 시간 참이면 알림을 만듦 |
| **Alertmanager** | 알림을 받아 **묶고(grouping), 중복을 없애고, 묵음 처리하고, 보낼 곳으로 라우팅** |

CloudWatch로 치면 alerting rule이 알람, Alertmanager가 composite alarm·suppression·SNS를 합친 역할이다([aws-cloudwatch/03](../aws-cloudwatch/03-alarms.md)).

## 5단계: Grafana — 보는 도구

**Grafana**는 데이터를 대시보드로 보여 주는 도구다. 데이터를 직접 저장하지 않고, **데이터 소스**(Prometheus, CloudWatch, DB 등)에 질의해 그래프를 그린다. 여러 데이터 소스를 한 대시보드에 섞을 수 있고, 알림 기능도 있다.

## 6단계: AWS가 대신 해 주는 것 — AMP와 AMG

직접 운영하면 Prometheus 서버의 저장 공간·고가용성·확장, Grafana 서버의 운영·로그인·업그레이드를 모두 챙겨야 한다. AWS 관리형 서비스는 이 부분을 대신한다.

```
EKS · ECS · EC2 워크로드
   │ scrape
   ▼
수집기 ─────────────────────────── remote write (SigV4 서명)
 · AWS 관리형 수집기(EKS, 에이전트 없음)                │
 · ADOT Collector(OpenTelemetry 배포판)                 ▼
 · Prometheus agent 모드                       ┌─ AMP 워크스페이스 ─┐
                                               │ 저장(기본 150일)   │
                                               │ PromQL 조회        │
                                               │ 규칙·Alertmanager ─┼─▶ SNS·PagerDuty
                                               └────────┬───────────┘
                                                        │ 데이터 소스
                                               ┌─ AMG 워크스페이스 ─┐
사용자 ── IAM Identity Center / SAML 로그인 ─▶ │ 대시보드·알림       │ ◀─ CloudWatch, X-Ray 등
                                               └────────────────────┘
```

| 서비스 | 맡는 일 | 대신 해 주는 것 | 자세히 |
| --- | --- | --- | --- |
| **AMP** | 메트릭 저장·조회·규칙·알림 | 저장소 확장, 3개 AZ 복제, 고가용성 | [02-amp.md](./02-amp.md) |
| 수집기 | scrape → remote write | (관리형 수집기를 쓰면) 수집기 운영까지 | [03-collection.md](./03-collection.md) |
| **AMG** | 대시보드·알림 화면 | Grafana 서버 운영, SSO 로그인, 버전 업그레이드 | [04-amg.md](./04-amg.md) |

- AMP·AMG의 단위는 둘 다 **워크스페이스**다. 리전 리소스이고, 각각 독립된 Prometheus 저장소·Grafana 서버라고 보면 된다.
- AMP에 데이터를 쓰고 읽는 요청은 IAM의 **SigV4 서명**이 필요하다. 그래서 수집기에는 IAM 권한(IRSA, Pod Identity 등)을 줘야 한다([aws-iam/05](../aws-iam/05-workload-identity.md)).
- AMG에는 **IAM user로 로그인할 수 없고**, IAM Identity Center나 SAML로 로그인한다([aws-organizations/04](../aws-organizations/04-identity-center.md)).

## 7단계: CloudWatch와 무엇이 다른가

| 항목 | CloudWatch | AMP + AMG |
| --- | --- | --- |
| 데이터 모델 | namespace·metric·dimension | 메트릭 이름·라벨(Prometheus) |
| 조회 언어 | metric math, Metrics Insights(SQL) | **PromQL**(오픈 소스 표준) |
| 수집 | AWS 서비스 자동 전송, agent, PutMetricData | scrape + remote write(Prometheus 생태계 그대로) |
| 강점 | AWS 서비스 메트릭이 기본으로 들어옴, 로그·알람과 통합 | 쿠버네티스·오픈 소스 exporter 생태계, 기존 Prometheus 규칙·대시보드 재사용 |
| 과금 단위 | 메트릭 수, API 호출, 알람 수 | **수집한 샘플 수**, 저장 GB, 조회 샘플 수 / 사용자 수(AMG) |

- **둘 중 하나만 고를 필요는 없다.** AWS 리소스 메트릭은 CloudWatch에, 쿠버네티스·앱 메트릭은 AMP에 두고, AMG에서 두 데이터 소스를 한 대시보드로 보는 구성이 흔하다.
- CloudWatch도 최근 OTLP·PromQL을 지원하기 시작했다([aws-cloudwatch/01](../aws-cloudwatch/01-overview.md)). 그래서 "Prometheus 호환"만으로는 AMP를 고를 이유가 약해질 수 있다. 기존 Prometheus 자산(규칙, 대시보드, exporter)이 있는지가 더 중요한 기준이다.

## 8단계: 요금은 어떻게 매겨지나

출처: [AMP pricing](https://aws.amazon.com/prometheus/pricing/), [AMG pricing](https://aws.amazon.com/grafana/pricing/)

| 서비스 | 과금 기준 | 단가(us-east-1, 서울 같음) |
| --- | --- | --- |
| AMP 수집 | **수집한 샘플 수** | 첫 20억 개는 1,000만 개당 $0.90, 이후 구간별 할인 |
| AMP 저장 | GB-월 | $0.03 |
| AMP 조회 | 처리한 샘플 수(QSP) | 10억 개당 $0.10 |
| AMP 관리형 수집기 | 수집기-시간 + 수집 샘플 | 시간당 $0.04 + 1,000만 개당 $0.03 |
| AMG | **로그인한 사용자 수** | 편집자 월 $9, 뷰어 월 $5(워크스페이스마다) |

**샘플 수 = 시계열 수 × 수집 횟수**다. 시계열 1만 개를 30초마다 수집하면 한 달(744시간)에 1만 × 89,280회 = 약 8.9억 샘플이고, 수집 비용은 약 89.28 × $0.90 ≈ **$80**다. 그래서 비용을 줄이는 방법은 세 가지다. **시계열 수(카디널리티)를 줄이고, 수집 주기를 늘리고, 필요 없는 메트릭을 버린다.** 자세한 내용은 [05-cost-issues.md](./05-cost-issues.md)에서 다룬다.

## 용어 정리

| 용어 | 뜻 | 처음 나오는 곳 |
| --- | --- | --- |
| Prometheus | 라벨 기반 메트릭 수집·저장·조회 오픈 소스 | 1단계 |
| 시계열 / 라벨 | 메트릭 이름+라벨 조합으로 식별되는 값의 흐름 / `이름="값"` 꼬리표 | 1단계 |
| 카디널리티 | 라벨 값 조합의 가짓수 = 시계열 수 | 1단계 |
| Counter / Gauge / Histogram | 누적 증가값 / 오르내리는 값 / 구간별 분포 | 1단계 |
| Scrape / exporter | 대상의 `/metrics`를 가져가는 수집 / 형식을 바꿔 노출하는 프로그램 | 2단계 |
| Remote write / agent 모드 | 원격 저장소로 밀어 보내는 방식 / 수집+전송만 하는 Prometheus 모드 | 2단계 |
| PromQL / `rate()` | Prometheus 조회 언어 / counter의 초당 증가율 | 3단계 |
| Recording rule / alerting rule | 미리 계산해 저장 / 조건이 참이면 알림 | 4단계 |
| Alertmanager | 알림을 묶고·묵음·라우팅하는 구성 요소 | 4단계 |
| Grafana / 데이터 소스 | 대시보드 도구 / Grafana가 질의하는 대상 | 5단계 |
| AMP / AMG | AWS 관리형 Prometheus / 관리형 Grafana | 6단계 |
| 워크스페이스 | AMP·AMG의 기본 단위(리전 리소스) | 6단계 |
| ADOT | AWS Distro for OpenTelemetry. 수집기 중 하나 | 6단계 |
| SigV4 | AWS API 요청 서명 방식 | 6단계 |
| 샘플 / QSP | 값 하나(시각+숫자) / 조회 때 처리한 샘플 수 | 8단계 |

## 출처

- https://prometheus.io/docs/concepts/data_model/
- https://prometheus.io/docs/concepts/metric_types/
- https://prometheus.io/docs/introduction/overview/
- https://prometheus.io/docs/prometheus/latest/configuration/configuration/
- https://prometheus.io/docs/prometheus/latest/querying/functions/
- https://prometheus.io/docs/prometheus/latest/querying/operators/
- https://prometheus.io/docs/prometheus/latest/configuration/recording_rules/
- https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/
- https://prometheus.io/docs/alerting/latest/overview/
- https://docs.aws.amazon.com/prometheus/latest/userguide/what-is-Amazon-Managed-Service-Prometheus.html
- https://docs.aws.amazon.com/grafana/latest/userguide/what-is-Amazon-Managed-Service-Grafana.html
- https://aws.amazon.com/prometheus/pricing/
- https://aws.amazon.com/grafana/pricing/

---
*조사 기준일: 2026-10-20. 다이어그램과 계산 예시는 이해를 돕기 위해 이 레포에서 만든 것이다(free tier 제외).*
