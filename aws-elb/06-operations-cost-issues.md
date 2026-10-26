# 운영·관측·비용·이슈

> 오류가 나면 먼저 **누가 만든 5xx인가**를 본다. `HTTPCode_ELB_5XX_Count`는 ALB가 만든 것이고, `HTTPCode_Target_5XX_Count`는 앱이 만든 것이다. ALB 5xx 중 **502는 타깃이 연결을 끊거나 잘못 응답한 것**(앱 keep-alive가 ALB idle timeout보다 짧은 경우가 흔함), **503은 보낼 타깃이 없는 것**, **504는 타깃이 제때 응답하지 않은 것**이다. 원인은 **액세스 로그**(5분 간격, S3)의 처리 시간 필드로 좁힌다. 비용은 시간당 요금 + **LCU**이고, internet-facing이면 **공인 IPv4 요금**이 추가된다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[05](./05-eks-ecs-iac.md)의 용어와 CloudWatch 메트릭·알람([aws-cloudwatch](../aws-cloudwatch/README.md))을 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 액세스 로그 | ALB가 처리한 요청 하나하나의 기록(S3에 저장) |
| 연결 로그 | TLS 연결 단위 기록(클라이언트 IP, TLS 버전, mTLS 인증서 정보) |
| 헬스체크 로그 | 헬스체크 결과와 실패 이유 기록 |
| LCU | ALB 사용량 과금 단위(네 가지 차원 중 최댓값) |
| LCU reservation | 미리 용량(LCU)을 예약해 갑작스러운 트래픽에 대비하는 기능 |
| Quota | 계정·리전별 리소스 개수 한도 |

## 핵심 메트릭

출처: [ALB CloudWatch metrics](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html)

네임스페이스는 `AWS/ApplicationELB`다. 요청이 있을 때만 60초 간격으로 보고되고, **헬스체크 요청은 메트릭에 포함되지 않는다.**

| 메트릭 | 뜻 | 볼 통계 | 알람 예 |
| --- | --- | --- | --- |
| `HTTPCode_ELB_5XX_Count` | **ALB가 만든** 5xx | Sum | 0보다 크게 지속 |
| `HTTPCode_Target_5XX_Count` | **타깃(앱)이 돌려준** 5xx | Sum | `RequestCount` 대비 비율 |
| `TargetResponseTime` | ALB가 요청을 보낸 뒤 응답 헤더를 받기까지 | p99, Average | p99 임계값 |
| `UnHealthyHostCount` | unhealthy 타깃 수(타깃 그룹별) | **Minimum** | Minimum > 0 |
| `HealthyHostCount` | healthy 타깃 수 | Minimum | 기대값 미만 |
| `RequestCount` | 타깃이 선택된 요청 수 | Sum | 급감 |
| `RejectedConnectionCount` | 최대 연결 수에 걸려 거부된 연결 | Sum | > 0 |
| `TargetConnectionErrorCount` | 타깃 연결 실패 | Sum | > 0 |
| `ConsumedLCUs` | 소비한 LCU | Sum | 비용 추세 |

- AWS는 `UnHealthyHostCount`를 **Minimum 통계로 0이 아닌지** 보라고 권한다. 짧은 깜빡임에 알람이 울리지 않게 하기 위해서다.
- 타깃을 등록 해제하면 `HealthyHostCount`는 줄지만 `UnHealthyHostCount`는 늘지 않는다. **둘 다 봐야** 타깃이 사라진 것을 안다.
- 메트릭 dimension(`LoadBalancer`, `TargetGroup`)은 ARN의 끝부분(`app/my-alb/0123456789abcdef`)이다.

```bash
aws cloudwatch put-metric-alarm \
  --alarm-name prod-alb-elb-5xx \
  --namespace AWS/ApplicationELB --metric-name HTTPCode_ELB_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/prod-public/0123456789abcdef \
  --statistic Sum --period 60 --evaluation-periods 5 --datapoints-to-alarm 3 \
  --threshold 10 --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions arn:aws:sns:ap-northeast-2:111122223333:oncall
```

요청이 없을 때는 메트릭이 비므로 `--treat-missing-data notBreaching`을 쓴다([aws-cloudwatch/03](../aws-cloudwatch/03-alarms.md)).

## 로그

출처: [Access logs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-access-logs.html), [Enable access logs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/enable-access-logging.html), [Connection logs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-connection-logs.html), [Health check logs](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-health-check-logs.html)

| 로그 | 단위 | 주기 | 쓰임 |
| --- | --- | --- | --- |
| 액세스 로그 | 요청 | 노드별 **5분마다** | 5xx 원인, 느린 요청, 클라이언트 분석 |
| 연결 로그 | TLS 연결 | 5분 | TLS 버전·암호 현황, mTLS 실패 |
| 헬스체크 로그(2025-11-21+) | 헬스체크 | 5분 | unhealthy 이유(TimedOut, ResponseCodeMismatch 등) |

- 기본은 꺼져 있다. **로그 전송은 무료**이고 S3 저장 비용만 낸다. 기록은 **best-effort**라 일부 누락될 수 있다.
- 버킷은 **같은 리전**이어야 하고(계정은 달라도 됨), 암호화는 **SSE-S3만** 지원한다. **SSE-KMS 버킷에는 쓸 수 없다.**
- 버킷 정책 principal은 `logdelivery.elasticloadbalancing.amazonaws.com`이다(2022-08 이전 리전은 리전별 ELB 계정 ID 방식도 동작).
- prefix에 `AWSLogs`를 넣으면 안 된다.
- 연결 로그와 액세스 로그는 `conn_trace_id` 필드로 연결할 수 있다.

### 액세스 로그로 원인 좁히기

| 필드 | 뜻 |
| --- | --- |
| `request_processing_time` | ALB가 요청을 받아 타깃에 보내기까지 |
| `target_processing_time` | 타깃에 보낸 뒤 응답 헤더를 받기까지(`-1`이면 타깃 응답 없음) |
| `response_processing_time` | ALB가 응답을 클라이언트에 보내기까지 |
| `elb_status_code` / `target_status_code` | ALB가 돌려준 코드 / 타깃이 돌려준 코드(`-`이면 타깃 응답 없음) |
| `error_reason` | 인증 액션·Lambda 타깃·WAF 관련 오류 이유 |
| `ssl_protocol`, `ssl_cipher` | TLS 정책 변경 전 영향 확인 |

S3의 액세스 로그는 Athena로 조회하는 경우가 많다(AWS가 테이블 정의 예시를 제공, 이번 조사에서 1차 출처로 재확인하지 않음).

## 5xx 진단표

출처: [Troubleshoot your ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-troubleshooting.html)

| 코드 | ALB 문서의 뜻 | 흔한 원인 | 확인할 것 |
| --- | --- | --- | --- |
| **502** | 타깃이 연결을 끊었거나(RST/FIN) 잘못된 응답 | **앱 keep-alive 타임아웃 < ALB idle timeout**, 응답 헤더 32K 초과, 배포 중 종료된 타깃, Lambda 오류·응답 1MB 초과, 타깃과의 TLS 오류 | 앱 keep-alive를 idle timeout보다 길게, preStop·deregistration delay([05](./05-eks-ecs-iac.md#무중단-배포-readiness-gate와-prestop)) |
| **503** | 타깃 그룹에 등록된 타깃이 없음 | 배포 중 타깃 0개, 규칙이 빈 타깃 그룹을 가리킴, ASG가 0으로 줄어듦 | `describe-target-health`, `HealthyHostCount` |
| **504** | 타깃이 제때 응답하지 않음 | 처리 시간 > idle timeout, 타깃 연결 10초 내 실패, **NACL이 ephemeral 포트를 막음** | `target_processing_time`, 보안 그룹·NACL, 느린 쿼리 |
| 460 | 클라이언트가 먼저 연결을 끊음 | 클라이언트 타임아웃이 서버 처리보다 짧음 | 클라이언트 타임아웃, 느린 요청 |
| 463 | `X-Forwarded-For`에 IP가 30개 초과 | 프록시가 여러 겹 | 앞단 프록시 설정 |
| 464 | 요청 프로토콜이 타깃 그룹 프로토콜 버전과 안 맞음 | HTTP/1.1 요청이 gRPC 타깃 그룹으로 | 규칙 조건, 타깃 그룹 버전 |
| 561 | IdP가 인증 오류를 돌려줌 | OIDC 설정 오류 | 액세스 로그 `error_reason` |

`HTTPCode_Target_5XX_Count`가 늘었다면 ALB 문제가 아니라 **앱 로그**를 본다.

## 비용

출처: [ELB pricing](https://aws.amazon.com/elasticloadbalancing/pricing/), Price List API(2026-09-11 게시본), [Public IPv4 charge](https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/)

### ALB 요금(us-east-1 = 서울)

| 항목 | 금액 |
| --- | --- |
| ALB 시간 | $0.0225/시간 |
| LCU | $0.008/LCU-시간 |
| mTLS trust store | $0.005/시간 |

LCU는 아래 네 차원을 각각 LCU로 환산해 **가장 큰 값**으로 매긴다.

| 차원 | 1 LCU |
| --- | --- |
| 새 연결 | 초당 25개 |
| 활성 연결 | 분당 3,000개(mTLS면 1,500) |
| 처리 바이트 | 시간당 1 GB(EC2·컨테이너·IP 타깃), Lambda 타깃은 0.4 GB |
| 규칙 평가 | 초당 1,000회(처음 10개 규칙은 무료) |

### 계산 예

요청 하루 내내 평균 초당 100건, 연결당 평균 5건 요청, 응답 평균 20 KB, 규칙 20개, internet-facing 2개 AZ라고 가정한다.

| 차원 | 계산 | LCU |
| --- | --- | --- |
| 새 연결 | 100 ÷ 5 = 초당 20개 → 20 ÷ 25 | 0.8 |
| 활성 연결 | 연결 유지 10초 가정 → 분당 약 200개 → 200 ÷ 3,000 | 0.07 |
| 처리 바이트 | 100 × 20 KB × 3,600 ≈ 시간당 7.2 GB → 7.2 ÷ 1 | **7.2** |
| 규칙 평가 | 초당 100 × (20 − 10) = 1,000 → 1,000 ÷ 1,000 | 1.0 |

- 시간당 최대 7.2 LCU × $0.008 = $0.0576, ALB 시간 $0.0225 → 시간당 약 $0.080, **월(730시간) 약 $58.4**
- 공인 IPv4 2개 × $0.005 × 730 = **월 $7.3**
- 합계 월 약 **$66**(데이터 전송 요금 별도). 위 숫자는 가정에서 계산한 예시이고, AWS Pricing Calculator로 확인한다.

**이 예에서는 처리 바이트가 비용을 정한다.** 응답 압축(gzip 등), 정적 파일을 CloudFront·S3로 분리하는 것이 효과가 크다.

### 비용 줄이기

| 방법 | 효과 |
| --- | --- |
| **ALB 공유**(호스트·경로 규칙, EKS IngressGroup) | ALB 시간 요금과 공인 IPv4를 서비스마다 내지 않음 |
| 정적 콘텐츠를 CloudFront·S3로 | 처리 바이트 감소 |
| 규칙 수 줄이기·순서 정리 | 10개 초과분 규칙 평가 감소 |
| `dualstack-without-public-ipv4` | ALB의 공인 IPv4 요금 제거(클라이언트가 IPv6여야 함) |
| 쓰지 않는 ALB 정리 | 트래픽이 없어도 시간 요금과 공인 IP 요금은 나감 |
| NLB cross-zone은 필요할 때만 | AZ 간 데이터 전송 요금 |

- **Free Tier**: CLB·ALB 합쳐 750시간, ALB 15 LCU(요금 페이지 기준).
- **LCU reservation**: 대형 이벤트 전에 최소 100 LCU부터 용량을 예약한다. 예약 단가는 사용 단가와 같고, 예약분 + 초과분을 낸다. 리전 단위 예약 한도 기본값이 0이라 증가 요청이 먼저 필요하다. 서울에서 NLB·GWLB 예약이 가능한지는 확인 필요.
- AZ 수만큼 공인 IP가 쓰인다는 계산은 문서 서술에서 유추한 것이고, ALB가 확장하면 IP가 늘 수 있다(확인 필요).

## 쿼터

출처: [ALB quotas](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-limits.html), [NLB quotas](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-limits.html)

| 항목 | 기본값 | 조정 |
| --- | --- | --- |
| 리전당 ALB | 50 | 가능 |
| 리전당 타깃 그룹(ALB·NLB 공용) | 3,000 | 가능 |
| ALB당 리스너 | 50 | 가능 |
| ALB당 규칙(default 제외) | 100 | 가능 |
| ALB당 인증서(default 제외) | 25 | 가능 |
| ALB당 타깃 그룹 | 100 | 불가 |
| ALB당 타깃 | 1,000 | 확인 필요 |
| 액션당 타깃 그룹 | 5 | 불가 |
| 규칙당 조건 값 | 5 | 불가 |
| 리전당 NLB | 50 | 가능 |
| NLB당 타깃 | 3,000(AZ당 500) | — |

**규칙 100개**는 IngressGroup으로 ALB 하나에 서비스를 많이 몰아넣을 때 먼저 걸린다.

## 운영 체크리스트

| 항목 | 권장 |
| --- | --- |
| 삭제 보호 | 운영 ALB는 `deletion_protection.enabled = true` |
| 보안 정책 | `ELBSecurityPolicy-TLS13-1-2-Res-PQ-2025-09` 명시([03](./03-tls-security.md)) |
| 잘못된 헤더 | `drop_invalid_header_fields = true` |
| 로그 | 액세스 로그(필요하면 연결·헬스체크 로그)를 S3에, 수명 주기 규칙으로 보존 기간 설정 |
| 알람 | ELB 5xx, Target 5xx 비율, `TargetResponseTime` p99, `UnHealthyHostCount` Minimum |
| idle timeout | 앱 keep-alive > ALB idle timeout(기본 60초) |
| deregistration delay | 가장 긴 요청보다 조금 길게. 300초 그대로 두지 않기 |
| 헬스체크 | 가벼운 전용 경로, Terraform에서는 값 명시 |
| WAF | 관리형 규칙을 Count로 시작해 Block으로 |
| CLB | ALB·NLB로 이전 계획 |

## 안티패턴

| 안티패턴 | 결과 | 대안 | 근거 |
| --- | --- | --- | --- |
| 서비스마다 ALB 하나 | 시간 요금·공인 IP 비용이 배로 | 규칙으로 ALB 공유 | 이 문서 |
| 앱 keep-alive를 ALB보다 짧게 | 간헐적 502 | 앱 타임아웃 > idle timeout | [02](./02-alb-routing-targets.md#연결-관련-속성) |
| SIGTERM 받자마자 종료 | 배포 때마다 502 | preStop, deregistration delay | [05](./05-eks-ecs-iac.md#무중단-배포-readiness-gate와-prestop) |
| 헬스체크에서 DB까지 확인 | DB 장애가 전체 unhealthy로 번짐 | 가벼운 헬스체크 | [02](./02-alb-routing-targets.md#헬스체크) |
| `ssl_policy` 생략 | TLS 1.0·1.1 허용 | 최신 정책 명시 | [03](./03-tls-security.md#보안-정책) |
| 타깃 보안 그룹을 VPC 전체에 열기 | ALB 우회, 헤더 위조 | ALB 보안 그룹만 허용 | [03](./03-tls-security.md#네트워크-보안) |
| 액세스 로그 없이 운영 | 5xx 원인을 찾을 수 없음 | 로그 켜기 | 이 문서 |
| 가중치 라우팅을 페일오버로 착각 | 죽은 쪽으로 계속 보냄 | 헬스체크·배포 자동화로 롤백 | [02](./02-alb-routing-targets.md#가중치-라우팅-카나리블루그린) |
| NLB를 보안 그룹 없이 생성 | 나중에 붙이려면 재생성 | 처음부터 보안 그룹 지정 | [04](./04-nlb-gwlb.md#alb와-다른-기본값) |
| Ingress를 지워 ALB가 사라짐 | DNS 이름 변경, 장애 | 삭제 보호, TargetGroupBinding | [05](./05-eks-ecs-iac.md#ingress-예시) |

## 출처

- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-cloudwatch-metrics.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-access-logs.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/enable-access-logging.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-connection-logs.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-health-check-logs.html
- https://aws.amazon.com/about-aws/whats-new/2025/11/application-load-balancer-health-check-logs/
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-troubleshooting.html
- https://aws.amazon.com/elasticloadbalancing/pricing/
- https://pricing.us-east-1.amazonaws.com/offers/v1.0/aws/AWSELB/current/index.json
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/capacity-unit-reservation.html
- https://aws.amazon.com/blogs/aws/new-aws-public-ipv4-address-charge-public-ip-insights/
- https://aws.amazon.com/vpc/pricing/
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-limits.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-limits.html

---
*조사 기준일: 2026-10-26. 요금은 Price List API(2026-09-11 게시본) 기준이고, 계산 예는 가정에서 계산한 것이다.*
