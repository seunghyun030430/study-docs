# Elastic Load Balancing 개요

> Elastic Load Balancing(ELB)은 들어오는 요청을 여러 서버로 나눠 보내는 AWS 관리형 서비스다. 종류는 넷이다. **웹·API(HTTP)면 ALB**를 쓴다. **TCP/UDP, 고정 IP, PrivateLink가 필요하면 NLB**를 쓴다. 방화벽 같은 보안 장비를 끼울 때는 **GWLB**를 쓴다. **CLB**는 이전 세대라 새로 쓰지 않는다. 요청은 **리스너 → 규칙 → 타깃 그룹 → 타깃** 순으로 흐르고, 헬스체크를 통과한 타깃에만 보낸다. 요금은 **시간당 요금 + 사용량 단위(LCU)**다.

## 1단계: 로드 밸런서가 왜 필요한가

서버 한 대로 서비스하면 두 가지 문제가 있다.

- 그 서버가 죽으면 서비스가 멈춘다.
- 사용자가 늘면 한 대로 감당할 수 없다.

그래서 서버를 여러 대 두고, 그 앞에 **요청을 나눠 주는 장치**를 둔다. 이것이 **로드 밸런서(load balancer)**다. 사용자는 로드 밸런서의 주소 하나만 알고, 뒤에 서버가 몇 대인지, 어떤 서버가 죽었는지 모른다.

```
사용자 ──▶ 로드 밸런서 ──┬──▶ 서버 A
                         ├──▶ 서버 B
                         └──▶ 서버 C (죽음 → 보내지 않음)
```

AWS의 로드 밸런서 서비스가 **Elastic Load Balancing(ELB)**이다. AWS가 운영하므로 장비를 설치하거나 관리하지 않고, 트래픽에 따라 자동으로 용량을 늘린다.

## 2단계: L4와 L7 — 무엇을 보고 나누나

네트워크 통신은 층(layer)으로 나눠 설명한다. 로드 밸런서에서 중요한 것은 두 층이다.

| 층 | 보는 정보 | 예시 판단 |
| --- | --- | --- |
| **L4**(전송 계층) | IP 주소, 포트, TCP/UDP | "443 포트로 온 연결은 이 서버 묶음으로" |
| **L7**(응용 계층) | HTTP 내용(도메인, 경로, 헤더, 쿠키) | "`/api`는 API 서버로, `/images`는 이미지 서버로" |

- L7 로드 밸런서는 HTTP를 이해하므로 **경로나 도메인에 따라** 다른 서버로 보낼 수 있다. 대신 HTTP가 아닌 통신(DB 연결, 게임 UDP)은 다루지 못한다.
- L4 로드 밸런서는 내용을 보지 않고 연결만 넘기므로 **빠르고 어떤 프로토콜이든** 처리한다.

## 3단계: 네 가지 종류

출처: [ELB features](https://aws.amazon.com/elasticloadbalancing/features/), [What is Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html)

| 종류 | 층 | 한 줄 |
| --- | --- | --- |
| **ALB**(Application Load Balancer) | L7 | HTTP·HTTPS·gRPC. 경로·도메인 기반 라우팅, 인증, WAF |
| **NLB**(Network Load Balancer) | L4 | TCP·UDP·TLS·QUIC. 고정 IP, 매우 많은 연결, PrivateLink |
| **GWLB**(Gateway Load Balancer) | L3 + L4 | 방화벽·IDS 같은 보안 장비(appliance)에 트래픽을 통과시킴 |
| **CLB**(Classic Load Balancer) | L4/L7 | **이전 세대.** 새로 만들지 않고 ALB·NLB로 옮김 |

> "Classic Load Balancers are the previous generation of load balancers from Elastic Load Balancing. We recommend that you migrate to a current generation load balancer." — [ELB User Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html)

CLB의 지원 종료일은 공식 발표를 찾지 못했다(2026-10-26 기준).

## 4단계: ALB의 구성 요소

ALB를 이해하려면 네 가지 부품을 알면 된다.

```
클라이언트
   │ https://api.example.com/users
   ▼
[리스너] HTTPS:443  ─ 인증서로 암호화를 풂
   │
   ├─ [규칙] host = api.example.com      → 타깃 그룹 A
   ├─ [규칙] path = /admin/*             → 타깃 그룹 B
   └─ [기본 규칙]                         → 404 응답
              │
              ▼
        [타깃 그룹 A] ─ 헬스체크: GET /health
          ├─ 타깃: EC2 i-0123456789abcdef0 (healthy)
          └─ 타깃: 10.0.1.25 (unhealthy → 제외)
```

| 부품 | 뜻 |
| --- | --- |
| **리스너(listener)** | 로드 밸런서가 연결을 받는 프로토콜과 포트. 예: HTTPS 443 |
| **규칙(rule)** | "이런 요청이면 이렇게 하라". 조건(도메인, 경로 등)과 액션(어느 타깃 그룹으로 보낼지 등) |
| **타깃 그룹(target group)** | 같은 일을 하는 서버 묶음. 헬스체크 설정을 가짐 |
| **타깃(target)** | 실제로 요청을 처리하는 것. EC2 인스턴스, IP 주소(컨테이너·Pod), Lambda 함수 |

자세한 규칙과 타깃 그룹 설정은 [02](./02-alb-routing-targets.md)에서 다룬다.

## 5단계: 헬스체크 — 죽은 서버 빼기

로드 밸런서는 타깃마다 **헬스체크(health check)**를 보낸다. ALB는 기본으로 30초마다 `GET /`을 보내 200이 오는지 본다.

- 연속으로 실패하면 **unhealthy**로 보고 요청을 보내지 않는다.
- 다시 연속으로 성공하면 **healthy**로 돌려놓는다.

헬스체크 덕분에 서버가 죽거나 배포 중이어도 사용자는 오류를 덜 본다. 다만 **모든 타깃이 unhealthy면 ALB는 전부에게 보낸다(fail-open).** 아무 데도 안 보내는 것보다 낫다는 설계다([02](./02-alb-routing-targets.md#모두-unhealthy면-fail-open)).

## 6단계: 가용 영역과 고가용성

AWS 리전은 물리적으로 떨어진 여러 **가용 영역(AZ, Availability Zone)**으로 나뉜다. ALB는 **최소 2개 AZ의 서브넷**에 만들어야 하고, AZ마다 로드 밸런서 노드가 생긴다. 한 AZ가 장애 나도 다른 AZ 노드가 계속 받는다.

| 용어 | 뜻 |
| --- | --- |
| **Cross-zone load balancing** | AZ를 넘어 모든 타깃에 고르게 나누는 것. **ALB는 항상 켜짐**, NLB·GWLB는 기본 꺼짐 |
| **internet-facing** | 인터넷에서 접근하는 로드 밸런서(공인 IP) |
| **internal** | VPC 안에서만 접근하는 로드 밸런서(사설 IP) |

## 7단계: TLS 종료

HTTPS 통신은 암호화되어 있다. ALB의 HTTPS 리스너는 **인증서로 암호화를 풀고(TLS 종료)** 내용을 본 뒤 타깃에 보낸다. 그래서 서버마다 인증서를 설치할 필요가 없고, 인증서는 **ACM(AWS Certificate Manager)**에서 무료로 발급·자동 갱신한다. 보안 정책, mTLS, 로그인 연동은 [03](./03-tls-security.md)에서 다룬다.

## 8단계: 종류 고르기

출처: [ELB features](https://aws.amazon.com/elasticloadbalancing/features/), [How ELB works](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)

| 기능 | ALB | NLB | GWLB | CLB |
| --- | --- | --- | --- | --- |
| 프로토콜 | HTTP, HTTPS, gRPC | TCP, UDP, TLS, TCP_UDP, QUIC, TCP_QUIC | IP(GENEVE) | TCP, SSL/TLS, HTTP, HTTPS |
| 경로·도메인 라우팅 | ✔ | — | — | — |
| 고정 IP·Elastic IP | — | **✔** | — | — |
| PrivateLink 서비스로 노출 | — (NLB 뒤에 두면 가능) | **✔** | ✔(GWLB endpoint) | — |
| 보안 그룹 | ✔ | ✔(만들 때 붙여야 함) | — | ✔ |
| 클라이언트 IP 전달 | `X-Forwarded-For` 헤더 | 패킷 출발지 그대로(조건부) | 그대로 | 헤더 |
| TLS 종료 | ✔ | ✔(TLS 리스너, mTLS 없음) | — | ✔ |
| 타깃 | instance, ip, Lambda | instance, ip, ALB | instance, ip | instance |
| cross-zone 기본값 | 항상 켜짐 | 꺼짐 | 꺼짐 | API 꺼짐 / 콘솔 켜짐 |
| idle timeout | 60초(1–4000) | TCP 350초(60–6000), UDP 120초 고정 | TCP 350초, UDP 120초 | 60초 |

AWS features 비교표는 NLB의 QUIC 지원과 idle timeout 설정을 아직 반영하지 않았다. 위 표는 사용자 가이드 문서를 기준으로 했다.

**고르는 순서**

1. HTTP·HTTPS·gRPC 웹 서비스 → **ALB**
2. HTTP가 아니거나(DB, MQTT, 게임 UDP), 고정 IP를 방화벽에 등록해야 하거나, 다른 VPC·계정에 PrivateLink로 제공 → **NLB**([04](./04-nlb-gwlb.md))
3. HTTP인데 고정 IP나 PrivateLink도 필요 → **NLB 뒤에 ALB**([04](./04-nlb-gwlb.md#alb를-nlb-뒤에-두기))
4. 모든 트래픽을 보안 장비로 검사 → **GWLB**
5. CLB를 쓰고 있다면 → ALB·NLB로 이전

## 9단계: 요금 개요

출처: [ELB pricing](https://aws.amazon.com/elasticloadbalancing/pricing/), Price List API(2026-09-11 게시본)

| 종류 | 시간당 | 사용량 단위 |
| --- | --- | --- |
| ALB | $0.0225 | LCU-시간당 $0.008 |
| NLB | $0.0225 | NLCU-시간당 $0.006 |
| GWLB | $0.0125 | GLCU-시간당 $0.004 |
| CLB | $0.025 | GB당 $0.008 |

us-east-1과 서울(ap-northeast-2) 금액이 같다. **LCU**(Load Balancer Capacity Unit)는 새 연결 수, 동시 연결 수, 처리 바이트, 규칙 평가 수 중 **가장 큰 값** 기준으로 계산하는 사용량 단위다. 여기에 internet-facing이면 **공인 IPv4 요금**(IP당 시간당 $0.005)이 따로 붙는다. 계산 방법과 비용 줄이기는 [06](./06-operations-cost-issues.md)에서 다룬다.

## 10단계: 어디서 만들고 연결하나

| 환경 | 보통 쓰는 방법 |
| --- | --- |
| EC2·Auto Scaling | Auto Scaling 그룹을 타깃 그룹에 연결 |
| ECS | 서비스 정의에 로드 밸런서·타깃 그룹 지정(타깃 타입 `ip`) |
| EKS | **AWS Load Balancer Controller**가 Ingress로 ALB, Service로 NLB를 만듦 |
| IaC | Terraform `aws_lb` 계열 리소스, CloudFormation `AWS::ElasticLoadBalancingV2::*` |

자세한 내용은 [05](./05-eks-ecs-iac.md)에 있다.

## 용어 정리

| 용어 | 뜻 | 처음 나온 곳 |
| --- | --- | --- |
| 로드 밸런서 | 요청을 여러 서버로 나눠 보내는 장치 | 1단계 |
| ELB | AWS의 관리형 로드 밸런서 서비스 이름 | 1단계 |
| L4 / L7 | 연결(IP·포트) 수준 / HTTP 내용 수준 | 2단계 |
| ALB / NLB / GWLB / CLB | ELB의 네 가지 종류 | 3단계 |
| 리스너 | 연결을 받는 프로토콜·포트 | 4단계 |
| 규칙 | 조건 + 액션. 우선순위 순으로 평가 | 4단계 |
| 타깃 그룹 | 같은 일을 하는 타깃 묶음 + 헬스체크 | 4단계 |
| 타깃 | 요청을 처리하는 EC2, IP, Lambda | 4단계 |
| 헬스체크 | 타깃이 정상인지 주기적으로 확인하는 요청 | 5단계 |
| fail-open | 모든 타깃이 unhealthy면 전부에게 보내는 동작 | 5단계 |
| AZ | 리전 안의 물리적으로 분리된 데이터센터 묶음 | 6단계 |
| Cross-zone load balancing | AZ를 넘어 고르게 분배 | 6단계 |
| internet-facing / internal | 인터넷용 / VPC 내부용 | 6단계 |
| TLS 종료 | 로드 밸런서에서 암호화를 푸는 것 | 7단계 |
| ACM | 인증서 발급·갱신 서비스 | 7단계 |
| PrivateLink | 다른 VPC·계정에 서비스를 사설 엔드포인트로 제공하는 기능 | 8단계 |
| LCU | 로드 밸런서 사용량 과금 단위 | 9단계 |
| AWS Load Balancer Controller | EKS에서 Kubernetes 리소스로 ALB·NLB를 만드는 컨트롤러 | 10단계 |

## 출처

- https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html
- https://aws.amazon.com/elasticloadbalancing/features/
- https://aws.amazon.com/elasticloadbalancing/pricing/
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-listeners.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/introduction.html

---
*조사 기준일: 2026-10-26. 요금은 Price List API(2026-09-11 게시본) 기준이다.*
