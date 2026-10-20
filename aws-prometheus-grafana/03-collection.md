# 수집: 관리형 수집기·ADOT·Prometheus agent

> AMP에 데이터를 넣는 방법은 모두 **remote write**이고, 차이는 **누가 scrape하느냐**다. EKS라면 **AWS 관리형 수집기**(에이전트를 설치하지 않고 AWS가 대신 scrape)가 가장 간단하다. ECS·EC2이거나 OpenTelemetry로 트레이스까지 모을 거라면 **ADOT Collector**, 기존 Prometheus 설정과 exporter를 그대로 쓰려면 **Prometheus agent 모드**를 쓴다. 어느 쪽이든 비용을 정하는 것은 **수집하는 시계열 수**이므로, 수집 단계에서 필요 없는 메트릭과 라벨을 버린다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)·[02](./02-amp.md)의 용어와 [aws-iam/05](../aws-iam/05-workload-identity.md)의 IRSA·Pod Identity를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 관리형 수집기(managed collector, scraper) | AWS가 운영하는 에이전트 없는 Prometheus scraper(EKS 전용) |
| ADOT(AWS Distro for OpenTelemetry) | AWS가 배포·지원하는 OpenTelemetry Collector |
| Receiver / exporter(OTel) | OTel Collector에서 데이터를 받는 부분 / 내보내는 부분 |
| Service discovery | 쿠버네티스 API 등에서 scrape할 대상을 자동으로 찾는 기능(`kubernetes_sd_configs`) |
| Relabeling | scrape 대상이나 메트릭의 라벨을 수집 단계에서 바꾸거나 버리는 설정 |
| Helm | 쿠버네티스 애플리케이션 패키지 관리자 |
| Access entry | EKS 클러스터에 IAM 주체의 접근 권한을 주는 방식 |

## 방식 비교

출처: [Ingest methods](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-ingest-methods.html). 아래 비교표는 각 방식의 공식 문서를 바탕으로 이 레포에서 정리한 것이다(AWS 공식 비교표는 찾지 못함).

| 항목 | **AWS 관리형 수집기** | **ADOT Collector** | **Prometheus agent 모드** |
| --- | --- | --- | --- |
| 실행 위치 | AWS(클러스터에 설치 없음) | 클러스터·태스크·인스턴스에 직접 배포 | 클러스터에 직접 배포(Helm 등) |
| 대상 | **EKS만** | EKS, ECS, EC2 | 어디든 |
| 설정 | Prometheus scrape 설정(`kubernetes_sd_configs`만) | OTel 설정(prometheus receiver + remote write exporter) | Prometheus 설정 그대로 |
| 운영 부담 | 가장 적음 | 수집기 운영 필요 | 수집기 운영 필요 |
| 트레이스·로그 | 메트릭만 | **메트릭·트레이스·로그 함께 가능** | 메트릭만 |
| 추가 비용 | 수집기 시간당 $0.04 + 샘플 1,000만 개당 $0.03 | 수집기가 쓰는 컴퓨팅 | 수집기가 쓰는 컴퓨팅 |

## AWS 관리형 수집기(EKS)

출처: [Managed collectors](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-collector-how-to.html), [create-scraper](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/amp/create-scraper.html)

```bash
# 1. 기본 설정을 받아 필요한 부분만 고친다
aws amp get-default-scraper-configuration --query configuration --output text | base64 -d > scraper.yaml

# 2. 수집기 생성
aws amp create-scraper --alias prod-eks \
  --scrape-configuration configurationBlob=$(base64 -i scraper.yaml | tr -d '\n') \
  --source eksConfiguration="{clusterArn=arn:aws:eks:ap-northeast-2:111122223333:cluster/prod,subnetIds=[subnet-aaa,subnet-bbb],securityGroupIds=[sg-ccc]}" \
  --destination ampConfiguration="{workspaceArn=arn:aws:aps:ap-northeast-2:111122223333:workspace/ws-0123}"
```

EKS 콘솔에서 클러스터를 만들 때나 Observability 탭에서도 만들 수 있다.

| 요구 사항·한도 | 내용 |
| --- | --- |
| EKS 엔드포인트 | **private 접근 포함** 필요 |
| VPC | DNS 활성화, 서로 다른 AZ의 서브넷 2개 이상(각각 IP 1개 이상 여유) |
| 클러스터 권한 | 인증 모드가 `API` 또는 `API_AND_CONFIG_MAP`이면 access entry 자동 생성. 아니면 aws-auth와 ClusterRole을 직접 설정 |
| 같은 리전 | 클러스터·수집기·워크스페이스가 모두 같은 리전 |
| 수집기 수 | 리전당 10개(증설 가능) |
| scrape 한도 | `/metrics` 응답 50 MB, 수집기당 엔드포인트 30,000개, **scrape 주기 최소 30초** |
| 설정 변경 | 대상 클러스터를 바꾸려면 수집기를 지우고 다시 만들어야 함 |
| 다른 계정 | EKS 계정의 원본 role과 워크스페이스 계정의 대상 role을 `--role-configuration`으로 지정 |

## ADOT Collector

출처: [Ingest with ADOT](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-ingest-with-adot.html), [ADOT on ECS](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-onboard-ingest-metrics-OpenTelemetry-ECS.html)

```yaml
extensions:
  sigv4auth:
    region: ap-northeast-2
    service: aps

receivers:
  prometheus:
    config:
      scrape_configs:
        - job_name: app
          scrape_interval: 30s
          static_configs:
            - targets: ["localhost:8080"]

exporters:
  prometheusremotewrite:
    endpoint: https://aps-workspaces.ap-northeast-2.amazonaws.com/workspaces/ws-0123/api/v1/remote_write
    auth:
      authenticator: sigv4auth

service:
  extensions: [sigv4auth]
  pipelines:
    metrics:
      receivers: [prometheus]
      exporters: [prometheusremotewrite]
```

- `sigv4auth` 확장이 IAM 자격증명으로 요청에 서명한다. 수집기에는 `aps:RemoteWrite` 권한이 있는 role이 필요하다(EKS는 IRSA·Pod Identity, ECS는 task role).
- 같은 Collector로 트레이스(X-Ray)·로그도 보낼 수 있어서, OpenTelemetry로 통일할 때 유리하다.
- 위 설정은 ADOT 문서의 구성 요소 이름으로 이 레포에서 조합한 최소 예시다. 실제 배포 전에 ADOT 버전별 문서와 대조한다(확인 필요).

## Prometheus agent 모드

출처: [Ingest with Prometheus](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-ingest-with-prometheus.html), [Remote write from EC2](https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-onboard-ingest-metrics-remote-write-EC2.html), [Agent mode](https://prometheus.io/docs/prometheus/latest/prometheus_agent/)

```yaml
global:
  scrape_interval: 30s
scrape_configs:
  - job_name: node
    static_configs:
      - targets: ["localhost:9100"]
remote_write:
  - url: https://aps-workspaces.ap-northeast-2.amazonaws.com/workspaces/ws-0123/api/v1/remote_write
    queue_config:
      max_samples_per_send: 1000
      max_shards: 200
      capacity: 2500
    sigv4:
      region: ap-northeast-2
```

- Prometheus 2.26 이상에서 `sigv4` 블록으로 서명한다.
- EKS에서는 Helm 차트 `prometheus-community/prometheus`의 `server.remoteWrite`에 같은 설정을 넣고, IRSA로 권한을 준다.
- **HA로 2대 이상 돌리면 중복 제거 설정**(AMP의 HA dedup용 외부 라벨 등)을 해야 한다. 안 하면 같은 샘플이 두 번 과금된다. 정확한 설정은 AMP 문서의 HA 항목을 따른다.

## 수집 단계에서 비용 줄이기

수집하지 않은 메트릭은 과금되지 않는다. **remote write 전에 버리는 것**이 가장 효과적이다.

```yaml
remote_write:
  - url: https://aps-workspaces.ap-northeast-2.amazonaws.com/workspaces/ws-0123/api/v1/remote_write
    sigv4:
      region: ap-northeast-2
    write_relabel_configs:
      # 쓰지 않는 메트릭 이름 통째로 버리기
      - source_labels: [__name__]
        regex: "go_gc_.*|process_.*"
        action: drop
      # 고카디널리티 라벨 제거
      - regex: "pod_template_hash|request_id"
        action: labeldrop
```

| 방법 | 효과 |
| --- | --- |
| 메트릭 drop(`write_relabel_configs`, `metric_relabel_configs`) | 쓰지 않는 메트릭 제거 |
| 라벨 drop(`labeldrop`) | 시계열 수(카디널리티) 감소 |
| scrape 주기 늘리기(15초 → 60초) | 샘플 수 4분의 1 |
| recording rule로 집계 후 원본 보존 기간 단축 | 조회 비용·속도 개선 |

위 relabel 설정은 Prometheus 설정 문법으로 이 레포에서 작성한 예시다. `labeldrop`으로 시계열을 구분하던 라벨을 지우면 서로 다른 시계열이 합쳐져 충돌할 수 있으니, 지울 라벨을 신중히 고른다.

## 이슈 / 주의

- **관리형 수집기는 EKS 전용**이고, `kubernetes_sd_configs`만 지원하며, scrape 주기 최소 30초다. 더 세밀한 수집이 필요하면 ADOT나 Prometheus agent를 쓴다.
- **private 서브넷의 수집기**는 VPC endpoint(`aps-workspaces`)와 **STS endpoint**가 둘 다 있어야 remote write가 된다([02](./02-amp.md#네트워크와-계정-간-접근)).
- **수집기 권한은 워크스페이스 단위로** 좁힌다. `aps:RemoteWrite`를 `*`에 주면 어떤 워크스페이스에도 쓸 수 있다.
- **수집기가 멈추면** 그동안의 데이터는 1시간 안에 다시 보내야 한다(더 오래된 샘플은 거부됨).

## 출처

- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-ingest-methods.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-collector-how-to.html
- https://awscli.amazonaws.com/v2/documentation/api/latest/reference/amp/create-scraper.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-ingest-with-adot.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-onboard-ingest-metrics-OpenTelemetry-ECS.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-ingest-with-prometheus.html
- https://docs.aws.amazon.com/prometheus/latest/userguide/AMP-onboard-ingest-metrics-remote-write-EC2.html
- https://prometheus.io/docs/prometheus/latest/prometheus_agent/
- https://prometheus.io/docs/prometheus/latest/configuration/configuration/

---
*조사 기준일: 2026-10-20. 설정 YAML은 공식 문서의 구성 요소로 이 레포에서 조합했고 실제 클러스터에 배포하지 않았다.*
