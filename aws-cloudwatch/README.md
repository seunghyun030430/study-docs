# Amazon CloudWatch 정리

Amazon CloudWatch를 실무 관점에서 정리한 문서 모음. 중점은 **Metrics·Alarms / Logs·Logs Insights / 비용·운영 이슈**이고, 트레이싱·APM(Application Signals, X-Ray, Synthetics, RUM)은 개요 수준으로만 다룬다. 조사 기준일 **2026-10-14**.

관련 주제: IAM 용어는 [aws-iam](../aws-iam/README.md), 여러 계정 구성은 [aws-organizations](../aws-organizations/README.md).

## 한눈에 보기

| 질문 | 짧은 답 | 문서 |
| --- | --- | --- |
| CloudWatch가 하는 일? | 메트릭·로그를 모아 보고(Dashboards·Logs Insights), 이상하면 알림(Alarms) | [01](./01-overview.md) |
| 메트릭은 무엇으로 구분하나? | namespace + 이름 + **dimension 조합**. 조합 하나가 메트릭 하나(= custom metric 과금 단위) | [02](./02-metrics.md) |
| 데이터는 얼마나 남나? | 1분 데이터 15일 → 5분 63일 → 1시간 15개월. 메트릭은 삭제 불가 | [02](./02-metrics.md) |
| 알람이 너무 자주 울리면? | M out of N, 올바른 TreatMissingData, composite alarm·suppressor·mute rule | [03](./03-alarms.md) |
| 에러 메트릭 알람의 missing data 처리? | `notBreaching`(에러가 없으면 데이터도 없음) | [03](./03-alarms.md) |
| 로그를 만들면 가장 먼저 할 일? | **보존 기간 설정**(기본값이 영구 보관) | [04](./04-logs.md) |
| Standard vs IA 로그 클래스? | IA는 수집 단가 절반이지만 metric filter·subscription·Live Tail 불가, 클래스 변경 불가 | [04](./04-logs.md) |
| 로그 분석은? | Logs Insights: `filter \| stats \| sort`, 스캔 GB당 과금 → 시간 범위를 좁게 | [05](./05-logs-insights.md) |
| 요금이 튀었다면? | Cost Explorer usage type 확인 → 로그 수집량·보존 기간·카디널리티·`GetMetricData` 순으로 점검 | [06](./06-cost-issues.md) |

| 비용 항목(us-east-1) | 단가 | 줄이는 방법 |
| --- | --- | --- |
| 로그 수집 | GB당 $0.50(IA $0.25) | 로그 레벨, 샘플링, IA, vended logs는 S3로 |
| 로그 저장 | GB-월당 $0.03 | 보존 기간, Intelligent Tiering |
| Custom metric | 메트릭당 월 $0.30부터 | 고카디널리티 dimension 제거 |
| `GetMetricData` | 메트릭 1,000개당 $0.01 | 외부 도구 폴링 축소, metric stream |
| Logs Insights | 스캔 GB당 $0.005 | 시간 범위·log group 축소, field index |

## 문서 인덱스

| 문서 | 다루는 것 |
| --- | --- |
| [01-overview.md](./01-overview.md) | 개요(단계별 입문) — 관측성 세 기둥, 메트릭·알람·로그 기초, 데이터 수집 경로, 기능 지도와 APM 개요, 요금 개요, 용어 정리 |
| [02-metrics.md](./02-metrics.md) | 메트릭 구조와 dimension, 해상도·보존·rollup, 상세 모니터링, 통계·백분위수, PutMetricData·EMF, GetMetricData, metric math, Metrics Insights |
| [03-alarms.md](./03-alarms.md) | 상태, M out of N 평가, TreatMissingData, 백분위수·고해상도·Metrics Insights 알람, 액션, composite alarm·suppressor·mute rule, anomaly detection, 요금 |
| [04-logs.md](./04-logs.md) | log group 설정, 보존 기간, 로그 클래스, Intelligent Tiering, KMS, PutLogEvents, metric filter, subscription filter, S3 export, Live Tail, data protection 등 |
| [05-logs-insights.md](./05-logs-insights.md) | 쿼리 구조, 자동 필드, 명령·함수, 자주 쓰는 쿼리, CLI 실행, 한도와 비용 줄이는 습관 |
| [06-cost-issues.md](./06-cost-issues.md) | 요금 구조 전체, 계산 예시, 비용 원인별 usage type과 대응, 주요 한도, 운영 안티패턴 종합 |

## 출처 (1차)

- CloudWatch User Guide: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/WhatIsCloudWatch.html
- CloudWatch Logs User Guide: https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/WhatIsCloudWatchLogs.html
- CloudWatch API Reference: https://docs.aws.amazon.com/AmazonCloudWatch/latest/APIReference/Welcome.html
- 요금: https://aws.amazon.com/cloudwatch/pricing/
- 비용 분석·절감: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_billing.html
- 한도: https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/cloudwatch_limits.html, https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/cloudwatch_limits_cwl.html

---
*이 문서는 학습·정리 목적의 요약이다. 요금은 리전과 시점에 따라 다르므로 적용 전에 요금 페이지를 다시 확인할 것.*
