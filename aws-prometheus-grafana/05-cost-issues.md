# 비용·장애·선택 기준

> AMP 비용은 **수집 샘플 수 = 시계열 수 × 수집 횟수**가 대부분이고, AMG 비용은 **로그인한 사용자 수**다. 그래서 AMP는 **카디널리티와 scrape 주기**를, AMG는 **워크스페이스 수와 편집자 수**를 관리한다. 흔한 장애는 시계열 폭증으로 인한 쓰로틀링, SigV4·STS 권한 문제로 인한 remote write 실패, 조회 한도 초과다. CloudWatch·직접 운영과 비교해 **기존 Prometheus 자산이 있고 쿠버네티스 중심일 때** AMP·AMG가 가장 잘 맞는다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[04](./04-amg.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Free tier | 매월 무료로 쓸 수 있는 양 |
| Native histogram | bucket을 미리 정하지 않는 Prometheus의 새 histogram 형식 |
| 쓰로틀링 | 한도를 넘는 요청을 거부하거나 늦추는 것 |
| 자체 운영(self-managed) | Prometheus·Grafana 서버를 직접 설치·운영하는 것 |

## 요금

### AMP

출처: [AMP pricing](https://aws.amazon.com/prometheus/pricing/)(us-east-1·서울 같은 단가, 2026-10-20 기준)

| 항목 | 단가 |
| --- | --- |
| 수집 — 첫 20억 샘플 | 1,000만 샘플당 $0.90 |
| 수집 — 다음 2,500억 샘플 | 1,000만 샘플당 $0.35 |
| 수집 — 2,520억 샘플 초과 | 1,000만 샘플당 $0.16 |
| 저장 | GB-월 $0.03 |
| 조회(QSP) | 처리한 샘플 10억 개당 $0.10 |
| 관리형 수집기 | 수집기-시간당 $0.04 + 수집 샘플 1,000만 개당 $0.03 |
| Free tier(매월) | 수집 4,000만 샘플, QSP 2,000억, 저장 10 GB |

- native histogram은 채워진 bucket 하나를 샘플 0.25개로 센다.
- 들어오는 데이터 전송(Data Transfer IN)은 무료다.

### AMG

출처: [AMG pricing](https://aws.amazon.com/grafana/pricing/)

| 항목 | 단가 |
| --- | --- |
| Editor·Admin | 워크스페이스당 활성 사용자 월 $9(워크스페이스마다 최소 1명분) |
| Viewer | 워크스페이스당 활성 사용자 월 $5 |
| Enterprise 플러그인 | 활성 사용자 월 $45 추가 |

## 계산해 보기

아래 예시는 위 단가로 이 레포에서 계산했다(free tier 제외, 한 달 744시간).

**예: EKS 노드 10대, 노드당 시계열 1,000개, 30초마다 수집**

```
시계열 = 10 × 1,000 = 10,000
수집 횟수 = 744시간 × 3,600초 ÷ 30초 = 89,280회
샘플 = 10,000 × 89,280 = 892,800,000 (약 8.9억)
```

| 항목 | 계산 | 월 비용 |
| --- | --- | --- |
| 수집 | 89.28 × $0.90 | 약 $80.4 |
| 관리형 수집기(쓰는 경우) | 744 × $0.04 + 89.28 × $0.03 | 약 $32.4 |
| AMG(편집자 3명, 뷰어 10명) | 3 × $9 + 10 × $5 | $77 |

요금 페이지의 같은 조건 예시(수집·저장·조회 합계 $81.75, 관리형 수집기 $32.44)와 같은 범위다. 페이지는 저장(약 $0.10)과 조회(약 $0.72)를 따로 계산해 더했다.

**같은 환경에서 바꿨을 때**

| 변경 | 수집 비용 |
| --- | --- |
| scrape 주기 30초 → 60초 | 절반(약 $40) |
| 노드당 시계열 1,000 → 3,000(라벨 하나 추가로 폭증) | 3배(약 $241) |
| 쓰지 않는 메트릭 40% drop | 약 $48 |

**AMP 비용은 저장보다 수집이 압도적으로 크다.** 보존 기간을 줄이는 것보다 수집하는 시계열 수를 줄이는 것이 효과가 크다.

## 흔한 장애와 함정

| 증상 | 원인 | 대응 | 자세히 |
| --- | --- | --- | --- |
| 일부 메트릭이 사라짐, `DiscardedSamples` 증가 | 활성 시계열 급증·수집 속도 한도로 쓰로틀링 | 카디널리티 원인 라벨 찾기, label set limit, 한도 증설 | [02](./02-amp.md#한도) |
| remote write `403` | `aps:RemoteWrite` 권한 없음, 다른 워크스페이스 ARN | 수집기 role 정책 확인 | [02](./02-amp.md#권한-모든-요청에-sigv4) |
| private 서브넷에서 remote write 실패 | `aps-workspaces` VPC endpoint나 **STS endpoint** 없음 | 둘 다 만들고 STS 리전 엔드포인트 사용 | [03](./03-collection.md#이슈--주의) |
| 수집기 재시작 후 그래프에 구멍 | 1시간보다 오래된 샘플은 거부 | 수집기 가용성 확보, 알림으로 수집 중단 감지 | [02](./02-amp.md#이슈--주의) |
| 비용이 두 배 | HA Prometheus 2대가 중복 제거 없이 같은 샘플 전송 | HA dedup 설정 | [03](./03-collection.md#prometheus-agent-모드) |
| 대시보드가 느리거나 오류 | 조회 한도(24시간당 샘플 5,000만 등), 95일 초과 기간 | recording rule로 미리 집계, 기간 축소 | [02](./02-amp.md#한도) |
| AMG에 로그인이 안 됨 | IAM user로 로그인 시도, Identity Center에 사용자 미할당 | SSO 사용자·그룹에 역할 할당 | [04](./04-amg.md#로그인과-사용자) |
| AMG에서 AMP 데이터가 안 보임 | 워크스페이스 role에 `AmazonPrometheusQueryAccess` 없음, 데이터 소스 미추가 | role 정책 확인, Grafana에서 데이터 소스 추가 | [04](./04-amg.md#데이터-소스와-권한) |
| AMG 알림이 두 번 옴 | Grafana 버전 9의 알림 중복 문제 | 10.4 이상으로 업그레이드 | [04](./04-amg.md#알림) |
| 자동화가 갑자기 실패 | service account 토큰 30일 만료 | 토큰 자동 갱신 | [04](./04-amg.md#자동화용-service-account) |

**"모니터링을 모니터링"**하는 알림을 하나 둔다. 예를 들어 `absent(up{job="app"})`처럼 수집이 끊긴 것을 감지하는 규칙을 AMP에 걸어 둔다.

## 선택 기준: CloudWatch vs AMP+AMG vs 직접 운영

| 상황 | 추천 |
| --- | --- |
| AWS 서비스 메트릭·로그·알람이 중심, 쿠버네티스 비중이 작음 | **CloudWatch**([aws-cloudwatch](../aws-cloudwatch/README.md)) |
| EKS 중심, 기존 Prometheus 규칙·exporter·Grafana 대시보드가 있음 | **AMP + AMG** |
| 여러 클라우드·온프레미스를 한 Grafana로, 사용자 수가 매우 많음 | AMP + 직접 운영 Grafana 또는 Grafana Cloud 검토 |
| 비용이 최우선, 운영 인력이 있음 | 직접 운영(Prometheus + Thanos/Mimir + Grafana). 대신 저장소 확장·HA·업그레이드를 직접 책임 |

- 많은 조직이 **혼합**한다. AWS 리소스는 CloudWatch, 쿠버네티스·앱은 AMP에 두고, AMG에서 두 데이터 소스를 함께 본다.
- AMG는 사용자당 과금이라 **보는 사람이 많을수록 비싸진다.** 반대로 AMP 비용은 사용자 수와 무관하다.
- 위 표는 이 레포의 정리이며, AWS 공식 선택 가이드는 아니다.

## 안티패턴

| 안티패턴 | 문제 | 대안 |
| --- | --- | --- |
| 라벨에 사용자 ID·요청 ID·전체 URL | 시계열 폭증, 비용·쓰로틀링 | 라벨 drop, 경로는 패턴으로 묶기 |
| 모든 메트릭을 15초 주기로 | 샘플 수 증가 | 중요한 것만 짧게, 나머지는 60초 |
| exporter 기본 메트릭을 전부 수집 | 쓰지 않는 메트릭에 과금 | `metric_relabel_configs`로 필요한 것만 |
| HA Prometheus를 중복 제거 없이 | 비용 2배 | HA dedup 설정 |
| 대시보드에서 긴 기간 원본 조회 | 조회 한도·QSP 비용 | recording rule로 미리 집계 |
| 사람마다 AMG 편집자 권한 | 편집자 요금($9) | 보기만 하면 뷰어($5) |
| AMG를 인터넷 전체에 열어 둠 | 접근 통제 부족 | network access control |
| 알림 규칙을 Grafana UI에서 손으로 | 변경 이력 없음, 100개 한도 | AMP 규칙 파일·Terraform으로 코드화 |

## 출처

- https://aws.amazon.com/prometheus/pricing/
- https://aws.amazon.com/grafana/pricing/
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP_quotas.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-ingest-with-prometheus.html
- https://docs.aws.amazon.com/grafana/latest/userguide/AMG_quotas.html
- https://prometheus.io/docs/practices/naming/

---
*조사 기준일: 2026-10-20. 계산 예시는 요금 페이지 단가로 이 레포에서 계산했다. 선택 기준표는 이 레포의 정리다.*
