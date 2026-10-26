# Elastic Load Balancing 정리

AWS Elastic Load Balancing(ELB)을 실무 관점에서 정리한 문서 모음. **ALB 중심**이고, NLB·GWLB·CLB는 비교와 선택 기준 위주다. 중점은 **라우팅·타깃 그룹·헬스체크 / TLS·보안 / EKS·ECS 연동과 Terraform / 운영·관측·비용**. 조사 기준일 **2026-10-26**.

관련 주제: 메트릭·알람은 [aws-cloudwatch](../aws-cloudwatch/README.md), IRSA·Pod Identity는 [aws-iam](../aws-iam/README.md), Terraform 문법·구조는 [terraform](../terraform/README.md).

## 한눈에 보기

| 질문 | 짧은 답 | 문서 |
| --- | --- | --- |
| 어떤 걸 쓰나? | 웹·API는 **ALB**, TCP/UDP·고정 IP·PrivateLink는 **NLB**, 보안 장비 경유는 **GWLB**, CLB는 이전 대상 | [01](./01-overview.md) |
| 요청은 어떻게 흐르나? | 리스너 → 규칙(우선순위 낮은 숫자부터) → 타깃 그룹 → healthy 타깃 | [01](./01-overview.md), [02](./02-alb-routing-targets.md) |
| 타깃이 전부 unhealthy면? | **전부에게 보냄(fail-open)** | [02](./02-alb-routing-targets.md) |
| 배포 때 502가 나면? | deregistration delay(기본 300초), preStop, 앱 keep-alive > idle timeout(60초) | [02](./02-alb-routing-targets.md), [05](./05-eks-ecs-iac.md), [06](./06-operations-cost-issues.md) |
| 보안 정책은? | `ELBSecurityPolicy-TLS13-1-2-Res-PQ-2025-09`. **IaC에서 생략하면 `2016-08`** | [03](./03-tls-security.md) |
| 로그인을 붙이려면? | HTTPS 리스너에 OIDC/Cognito 인증 액션. 앱은 `x-amzn-oidc-data` 서명 검증 | [03](./03-tls-security.md) |
| 고정 IP가 필요하면? | NLB(Elastic IP). HTTP 기능도 필요하면 **NLB → ALB** | [04](./04-nlb-gwlb.md) |
| EKS에서는? | AWS Load Balancer Controller. **기본 internal·instance**라 명시, `target-type: ip` + readiness gate | [05](./05-eks-ecs-iac.md) |
| Terraform 함정은? | `ssl_policy`·헬스체크 기본값이 콘솔과 다름, 타깃 그룹 교체 → `name_prefix` + `create_before_destroy` | [05](./05-eks-ecs-iac.md) |
| 5xx 원인은? | ELB 5xx vs Target 5xx 구분 → 502/503/504 표 → 액세스 로그 처리 시간 | [06](./06-operations-cost-issues.md) |
| 비용은? | 시간당 + LCU(네 차원 중 최댓값) + 공인 IPv4. 보통 **처리 바이트**가 좌우 | [06](./06-operations-cost-issues.md) |

| 종류 | 층 | 시간당(us-east-1 = 서울) | 사용량 단위 | 핵심 |
| --- | --- | --- | --- | --- |
| ALB | L7 | $0.0225 | $0.008/LCU-시간 | 경로·호스트 라우팅, 인증, WAF, Lambda 타깃 |
| NLB | L4 | $0.0225 | $0.006/NLCU-시간 | 고정 IP, TCP·UDP·TLS·QUIC, PrivateLink |
| GWLB | L3+L4 | $0.0125 | $0.004/GLCU-시간 | GENEVE로 보안 장비 경유 |
| CLB | L4/L7 | $0.025 | $0.008/GB | 이전 세대 |

| 같은 이름, 다른 기본값 | ALB | NLB |
| --- | --- | --- |
| cross-zone | 항상 켜짐(타깃 그룹에서만 끔) | 꺼짐 |
| idle timeout | 60초 | TCP 350초, UDP 120초 |
| 헬스체크 | HTTP, healthy 5 / unhealthy 2 | TCP, healthy 5 / unhealthy 2 |
| 보안 그룹 | 언제든 | 생성 시에만 |

## 문서 인덱스

| 문서 | 다루는 것 |
| --- | --- |
| [01-overview.md](./01-overview.md) | 개요(단계별 입문) — 로드 밸런서, L4/L7, 네 종류, 리스너·규칙·타깃 그룹, 헬스체크, AZ, TLS 종료, 종류 비교·선택, 요금 개요, 용어 정리 |
| [02-alb-routing-targets.md](./02-alb-routing-targets.md) | 리스너 규칙의 조건·액션·우선순위, 가중치 라우팅, transform, 타깃 타입, 헬스체크·fail-open, deregistration delay·slow start, 분배 알고리즘, stickiness, cross-zone, 연결 속성 |
| [03-tls-security.md](./03-tls-security.md) | ACM 인증서·SNI, 보안 정책(PQ), mTLS, OIDC·Cognito 인증, JWT 검증, WAF, 보안 그룹·scheme·서브넷·IP 타입 |
| [04-nlb-gwlb.md](./04-nlb-gwlb.md) | NLB 용도·리스너·ALB와 다른 기본값, 클라이언트 IP 보존·proxy protocol v2, NLB→ALB, PrivateLink, GWLB 구조, CLB 이전 |
| [05-eks-ecs-iac.md](./05-eks-ecs-iac.md) | AWS Load Balancer Controller 설치·Ingress·Service·annotation, readiness gate·preStop, EKS Auto Mode 차이, ECS 연동·blue/green, Terraform 예시·기본값 함정·교체 인자 |
| [06-operations-cost-issues.md](./06-operations-cost-issues.md) | 핵심 메트릭·알람, 액세스·연결·헬스체크 로그, 5xx 진단표, LCU 계산 예·비용 절감, 쿼터, 운영 체크리스트, 안티패턴 |

## 예제 검증

- 02~05의 HCL은 Terraform 1.16.6 + AWS provider 6.x에서 `terraform validate`와 `fmt -check`를 통과했다(2026-10-26). 문서에 없는 참조 대상(타깃 그룹, 보안 그룹, ACM 인증서, WAF web ACL)은 검증할 때만 보조 정의를 붙였다.
- 05의 Kubernetes YAML은 파싱만 확인했고, 실제 클러스터에 적용하지 않았다.
- AWS CLI 예시는 실제 계정에서 실행하지 않았다. ARN·ID는 예시 값이다.

## 출처 (1차)

- ALB 사용자 가이드: https://docs.aws.amazon.com/elasticloadbalancing/latest/application/introduction.html
- NLB 사용자 가이드: https://docs.aws.amazon.com/elasticloadbalancing/latest/network/introduction.html
- GWLB 사용자 가이드: https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html
- ELB 공통 가이드: https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html
- 요금: https://aws.amazon.com/elasticloadbalancing/pricing/
- AWS Load Balancer Controller: https://kubernetes-sigs.github.io/aws-load-balancer-controller/
- Terraform `aws_lb`: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb

---
*이 문서는 학습·정리 목적의 요약이다. 기본값·한도·요금은 자주 바뀌므로 적용 전에 공식 문서를 다시 확인할 것. 로드 밸런서·리스너·타깃 그룹을 지우거나 scheme·타깃 타입을 바꾸면 트래픽이 끊기거나 DNS 이름이 바뀌므로 먼저 영향 범위를 확인할 것.*
