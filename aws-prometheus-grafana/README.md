# AMP·AMG 정리

Amazon Managed Service for Prometheus(AMP)와 Amazon Managed Grafana(AMG)로 만드는 관측 파이프라인을 정리한 문서 모음. 중점은 **Prometheus 기초 / AMP(저장·조회·규칙·알림) / 수집 방식 / AMG(로그인·데이터 소스·알림) / 비용·장애·선택 기준**. 조사 기준일 **2026-10-20**.

관련 주제: [aws-cloudwatch](../aws-cloudwatch/README.md)(대안·함께 쓰는 서비스), [aws-iam](../aws-iam/README.md)(SigV4·IRSA·Pod Identity), [aws-organizations](../aws-organizations/README.md)(IAM Identity Center), [istio](../istio/README.md)(Prometheus로 수집하는 대표 대상).

## 한눈에 보기

```
워크로드 ──scrape──▶ 수집기 ──remote write(SigV4)──▶ AMP ──PromQL──▶ AMG ◀── SSO 로그인
                    (관리형 수집기 / ADOT / Prometheus agent)   └─ 규칙·Alertmanager ─▶ SNS·PagerDuty
```

| 질문 | 짧은 답 | 문서 |
| --- | --- | --- |
| Prometheus 메트릭은 어떻게 생겼나? | 메트릭 이름 + 라벨 = 시계열. **라벨 값 조합마다 시계열 하나** | [01](./01-overview.md) |
| AMP가 하는 일은? | remote write로 받아 3 AZ에 저장(기본 150일), PromQL 조회, 규칙·Alertmanager | [02](./02-amp.md) |
| 어떻게 수집하나? | EKS면 **관리형 수집기**, ECS·EC2·OTel이면 **ADOT**, 기존 설정 재사용이면 **Prometheus agent** | [03](./03-collection.md) |
| AMG 로그인은? | **IAM Identity Center 또는 SAML만**(IAM user 불가) | [04](./04-amg.md) |
| CLI로 AMG를 만들 때 주의? | `--permission-type CUSTOMER_MANAGED`만 가능 | [04](./04-amg.md) |
| 비용은 무엇으로 정해지나? | AMP: **샘플 수 = 시계열 수 × 수집 횟수**. AMG: **로그인한 사용자 수** | [05](./05-cost-issues.md) |
| CloudWatch 대신 언제 쓰나? | EKS 중심이고 기존 Prometheus 규칙·대시보드가 있을 때. 둘을 섞어 쓰는 경우가 많음 | [01](./01-overview.md), [05](./05-cost-issues.md) |

| 항목 | AMP | AMG |
| --- | --- | --- |
| 단위 | 워크스페이스(`ws-…`) | 워크스페이스(`g-…`) |
| 핵심 한도 | 활성 시계열 5,000만, 조회 기간 95일, 규칙 2,000 | 리전당 워크스페이스 5, 알림 규칙 100 |
| 요금(us-east-1) | 수집 1,000만 샘플당 $0.90부터, 저장 GB-월 $0.03, 조회 10억 샘플당 $0.10 | 편집자 월 $9, 뷰어 월 $5 |

## 문서 인덱스

| 문서 | 다루는 것 |
| --- | --- |
| [01-overview.md](./01-overview.md) | 개요(단계별 입문) — Prometheus 데이터 모델·메트릭 타입, scrape·remote write, PromQL, 규칙·Alertmanager, Grafana, AMP·AMG 역할, CloudWatch 비교, 요금 개요, 용어 정리 |
| [02-amp.md](./02-amp.md) | 워크스페이스·엔드포인트·보존 기간·암호화, SigV4 권한, VPC endpoint·계정 간 접근, 규칙 업로드, Alertmanager, 한도, label set limit |
| [03-collection.md](./03-collection.md) | 관리형 수집기·ADOT·Prometheus agent 비교와 설정 예시, 수집 단계에서 비용 줄이기(relabel) |
| [04-amg.md](./04-amg.md) | 워크스페이스·버전, 로그인(Identity Center·SAML)·역할, service account, 데이터 소스 권한 방식, 네트워크, 알림, 요금·한도 |
| [05-cost-issues.md](./05-cost-issues.md) | AMP·AMG 요금, 계산 예시, 흔한 장애 표, CloudWatch·직접 운영과의 선택 기준, 안티패턴 |

## 출처 (1차)

- AMP User Guide: https://docs.aws.amazon.com/prometheus/latest/userguide/what-is-Amazon-Managed-Service-Prometheus.html
- AMG User Guide: https://docs.aws.amazon.com/grafana/latest/userguide/what-is-Amazon-Managed-Service-Grafana.html
- AMP 요금: https://aws.amazon.com/prometheus/pricing/
- AMG 요금: https://aws.amazon.com/grafana/pricing/
- Prometheus 문서: https://prometheus.io/docs/
- AWS CLI `amp`: https://awscli.amazonaws.com/v2/documentation/api/latest/reference/amp/index.html
- AWS CLI `grafana`: https://docs.aws.amazon.com/cli/latest/reference/grafana/

---
*이 문서는 학습·정리 목적의 요약이다. 문서 속 YAML·JSON은 문법 검사만 했고, Prometheus 규칙은 promtool로 검사하지 않았으며, 실제 계정·클러스터에 배포하지 않았다.*
