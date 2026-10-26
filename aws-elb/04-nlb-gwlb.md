# NLB·GWLB와 CLB 이전

> **NLB**는 L4 로드 밸런서로, **AZ마다 고정 IP(Elastic IP 가능)**, TCP·UDP·TLS·QUIC, **PrivateLink 서비스 노출**이 강점이다. ALB와 기본값이 많이 달라서 주의해야 한다. **cross-zone은 꺼져 있고**, 보안 그룹은 **만들 때 붙이지 않으면 나중에 못 붙인다.** 클라이언트 IP 보존은 타깃 타입·프로토콜에 따라 다르다. HTTP 기능과 고정 IP가 둘 다 필요하면 **NLB → ALB** 구성을 쓴다. **GWLB**는 방화벽 같은 보안 장비에 트래픽을 통과시키는 용도다. **CLB**는 이전 세대이므로 ALB·NLB로 옮긴다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[03](./03-tls-security.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Elastic IP | 계정에 고정으로 할당하는 공인 IPv4 주소 |
| Client IP preservation | 타깃이 보는 패킷 출발지 주소를 원래 클라이언트 IP로 유지하는 것 |
| Proxy protocol v2 | 연결 앞에 원래 클라이언트 주소를 붙여 보내는 표준 헤더 형식(L4에서 클라이언트 IP 전달용) |
| PrivateLink / endpoint service | 내 서비스를 다른 VPC·계정에 인터페이스 엔드포인트로 제공하는 기능 |
| QUIC | UDP 위에서 동작하는 전송 프로토콜(HTTP/3의 기반) |
| GENEVE | 원래 패킷을 감싸서 보내는 터널링 프로토콜. GWLB가 6081 포트로 사용 |
| Appliance | 방화벽, IDS/IPS 같은 보안 장비(가상 머신) |

## NLB

출처: [Network Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html), [NLB listeners](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-listeners.html)

### 언제 쓰나

| 요구 | 이유 |
| --- | --- |
| HTTP가 아닌 프로토콜(DB 프록시, MQTT, SMTP, 게임 UDP) | ALB는 HTTP만 |
| **고정 IP** — 고객사 방화벽에 IP 등록 | AZ마다 IP가 고정되고 Elastic IP를 붙일 수 있음. ALB IP는 바뀜 |
| **PrivateLink**로 다른 VPC·계정에 서비스 제공 | endpoint service는 NLB(또는 GWLB)만 가능 |
| 매우 많은 동시 연결, 낮은 지연 | 내용을 보지 않고 연결을 넘김 |
| 클라이언트 IP를 패킷 출발지로 받아야 함 | ALB는 헤더로만 전달 |

### 리스너

| 프로토콜 | 메모 |
| --- | --- |
| TCP / UDP / TCP_UDP | 기본 |
| TLS | NLB에서 TLS 종료. ACM 인증서 사용. ALPN 기본값 `None`. **mTLS 미지원** |
| QUIC / TCP_QUIC | 2025-11+. **passthrough만**(NLB가 풀지 않음), QUIC v1만. **보안 그룹이 없는 NLB만** |

### ALB와 다른 기본값

| 항목 | ALB | NLB |
| --- | --- | --- |
| cross-zone | 항상 켜짐 | **꺼짐**. 켜면 AZ 간 데이터 전송 요금 발생 |
| 보안 그룹 | 언제든 변경 | **만들 때 없으면 나중에 추가 불가**(2023-08-10부터 지원) |
| idle timeout | 60초 | TCP 350초(60–6000, 2024-09+), TLS 350초 고정, UDP 120초 고정 |
| 헬스체크 프로토콜 | HTTP | TCP(HTTP/HTTPS 선택 가능), matcher 200–399 |
| 헬스체크 타임아웃 | 5초 | HTTP 6초, TCP·HTTPS 10초 |
| fail-open | 타깃 그룹 단위 | **활성화된 모든 AZ의 타깃이 동시에 실패하면** |
| 클라이언트 IP | `X-Forwarded-For` | 패킷 출발지(조건부, 아래) |

- NLB 보안 그룹을 쓰면 헬스체크는 **아웃바운드 규칙만** 적용받는다(인바운드 규칙과 무관).
- idle timeout을 350초보다 길게 잡으면 타깃 쪽 연결 추적 타임아웃도 같이 늘려야 한다.
- NLB는 `scheme`을 생략하면 **internal**이 아니라 API 기본값(internet-facing)이 적용된다. EKS의 Load Balancer Controller는 반대로 internal이 기본이라 헷갈리기 쉽다([05](./05-eks-ecs-iac.md)).

### 클라이언트 IP 보존

출처: [NLB target group attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/edit-target-group-attributes.html)

`preserve_client_ip.enabled` 기본값:

| 타깃 타입 | 프로토콜 | 기본값 |
| --- | --- | --- |
| instance | 전부 | **켜짐** |
| ip | UDP, TCP_UDP, QUIC, TCP_QUIC | **켜짐** |
| ip | **TCP, TLS** | **꺼짐** → 타깃은 NLB 사설 IP를 봄 |

- EKS·ECS처럼 `ip` 타깃 + TCP면 기본으로 클라이언트 IP가 보이지 않는다. 켜거나, **proxy protocol v2**(`proxy_protocol_v2.enabled`)를 쓴다.
- proxy protocol v2를 켜면 **헬스체크 연결에도 헤더가 붙는다.** 타깃(nginx 등)이 헤더를 파싱하도록 설정하지 않으면 헬스체크까지 실패한다.
- 보존이 켜져 있으면 타깃 보안 그룹은 **클라이언트 IP 대역**을 허용해야 한다(NLB IP가 아니라).
- PrivateLink를 거친 트래픽은 출발지가 항상 NLB 사설 IP다. 이때는 proxy protocol v2의 TLV에 VPC endpoint ID가 담긴다.
- 같은 타깃이 NLB를 거쳐 자기 자신에게 연결하는 경우(hairpin)는 보존이 켜져 있으면 실패한다.

### 연결 종료와 deregistration

- deregistration delay 기본 300초(QUIC는 고정).
- `deregistration_delay.connection_termination.enabled`를 켜면 delay가 끝날 때 남은 연결을 닫는다. 끄면 오래 사는 TCP 연결은 delay가 지나도 계속 열려 있을 수 있다.
- unhealthy 타깃의 연결은 기본으로 종료된다(`target_health_state.unhealthy.connection_termination.enabled`).

### AZ 단위 장애 대응

- NLB는 AZ별 DNS 이름(`{az}.{이름}-{id}.elb.{region}.amazonaws.com`)을 제공한다.
- `dns_record.client_routing_policy`로 클라이언트를 같은 AZ로 보내는 비율(100%/85%/0%)을 정한다.
- 특정 AZ가 이상하면 **ARC zonal shift**로 그 AZ를 잠시 빼는 기능이 있다(`zonal_shift.config.enabled`, 기본 false). 한 번에 하나의 AZ만 지정할 수 있다.

## ALB를 NLB 뒤에 두기

출처: [ALB as NLB target](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/application-load-balancer-target.html)

```
고객사 방화벽 (IP 허용 목록)
     │
     ▼
NLB (AZ별 Elastic IP)  ──또는──  PrivateLink endpoint (다른 VPC·계정)
     │ TCP:443, 타깃 타입 alb
     ▼
ALB (HTTPS 리스너, 경로 라우팅, WAF, 인증)
     ▼
타깃 그룹들
```

| 제약 | 내용 |
| --- | --- |
| 타깃 타입·프로토콜 | `alb`, **TCP만**, 포트는 ALB 리스너 포트와 같아야 함 |
| 위치 | 같은 VPC, 같은 계정 |
| 개수 | ALB 1개는 최대 2개 NLB의 타깃 |
| 통신 | NLB↔ALB는 항상 IPv4 |
| 헬스체크 | HTTP 또는 HTTPS |
| 한도 | ALB 하나가 NLB 타깃 한도를 50개(cross-zone 꺼짐) 또는 100개(켜짐)만큼 차지 |

- TLS는 **ALB에서 종료**한다(NLB는 TCP로 넘김). 그래서 ALB의 인증서·WAF·규칙을 그대로 쓸 수 있다.
- 비용은 NLB + ALB 두 개다.

```hcl
resource "aws_lb_target_group" "alb_behind_nlb" {
  name        = "tg-alb-443"
  target_type = "alb"
  protocol    = "TCP"
  port        = 443
  vpc_id      = "vpc-0123456789abcdef0"

  health_check {
    protocol = "HTTPS"
    path     = "/health"
  }
}

resource "aws_lb_target_group_attachment" "alb" {
  target_group_arn = aws_lb_target_group.alb_behind_nlb.arn
  target_id        = aws_lb.main.arn # ALB ARN
  port             = 443
}
```

## PrivateLink로 서비스 제공

```bash
# NLB를 endpoint service로 등록 (소비자 연결은 승인 필요)
aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns arn:aws:elasticloadbalancing:ap-northeast-2:111122223333:loadbalancer/net/my-nlb/0123456789abcdef \
  --acceptance-required

# 연결을 허용할 소비자 계정
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id vpce-svc-0123456789abcdef0 \
  --add-allowed-principals arn:aws:iam::444455556666:root
```

- SaaS 제공자가 고객 VPC에 서비스를 노출하거나, 조직 안에서 공용 서비스를 다른 계정에 제공할 때 쓴다. VPC 피어링과 달리 **CIDR이 겹쳐도** 된다(PrivateLink 문서 기준).
- 보안 그룹이 있는 NLB는 PrivateLink 트래픽에도 인바운드 규칙을 적용할지 선택할 수 있다.

## GWLB

출처: [Gateway Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html), [GWLB endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-gateway-load-balancer.html)

GWLB는 **트래픽을 받아서 보안 장비로 통과시킨 뒤 원래 목적지로 돌려보내는** 로드 밸런서다. 방화벽, IDS/IPS, 패킷 검사(DPI) 장비를 여러 대 두고 나눌 때 쓴다.

```
인터넷 → IGW → (라우팅) → GWLB endpoint → GWLB → 방화벽 장비들 (GENEVE 6081)
                                                    │ 검사 통과
                              앱 서브넷 ◀── GWLB endpoint ◀─┘
```

| 항목 | 내용 |
| --- | --- |
| 프로토콜 | 모든 IP 트래픽, 장비와는 **GENEVE(UDP 6081)** |
| 연결 방식 | 트래픽이 지나가는 VPC에 **GWLB endpoint**(PrivateLink)를 두고 **라우팅 테이블**로 보냄 |
| flow stickiness | 5-tuple(기본), 3-tuple, 2-tuple |
| MTU | 8500. GENEVE 헤더 68바이트 때문에 장비 MTU는 8568 이상. 단편화·PMTUD 미지원 |
| 보안 그룹 | 지원 안 함 |
| 요금 | GWLB 시간당 $0.0125 + GLCU, endpoint는 PrivateLink 요금 별도 |

- 인터넷 인입 검사 예: IGW 라우팅 테이블(ingress routing)에서 앱 서브넷 CIDR → GWLB endpoint, 앱 서브넷의 0.0.0.0/0 → GWLB endpoint, endpoint 서브넷의 0.0.0.0/0 → IGW.
- GWLB endpoint와 앱 서버는 **다른 서브넷**에 둬야 한다.
- 가는 길과 오는 길이 다른 장비를 지나는 **비대칭 흐름**은 권장하지 않는다.
- 보통 AWS Marketplace의 방화벽 제품과 함께 쓴다. AWS Network Firewall은 내부적으로 비슷한 구조지만 관리형 서비스라 GWLB를 직접 다루지 않는다(Network Firewall 문서로 별도 확인 필요).

## CLB에서 옮기기

출처: [Migrate your Classic Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/migrate-classic-load-balancer.html)

| 방법 | 내용 |
| --- | --- |
| 콘솔 마이그레이션 마법사 | 설정을 읽어 **새** ALB·NLB를 만든다. 기존 CLB는 바꾸지 않음 |
| 수동 | 새 LB를 만들고 Route 53 **가중치 레코드로 조금씩** 트래픽을 옮김 |

옮길 때 바뀌는 것:

| 항목 | CLB | ALB·NLB |
| --- | --- | --- |
| CLI | `aws elb` | `aws elbv2` |
| API 버전 | 2012-06-01 | 2015-12-01 |
| CloudWatch 네임스페이스 | `AWS/ELB` | `AWS/ApplicationELB`, `AWS/NetworkELB` |
| Terraform | `aws_elb` | `aws_lb` |

- 알람·대시보드·스크립트를 함께 바꿔야 한다.
- 조직에서 CLB 생성을 막는 IAM 정책 예시가 AWS 문서에 있다(`elasticloadbalancing:ListenerProtocol` 조건 키 사용).
- EC2-Classic 네트워크(2022-08-15 종료)와 CLB는 별개다. CLB 자체는 VPC에서 여전히 동작한다.

## 이슈 / 주의

- **NLB를 보안 그룹 없이 만들기**: 나중에 붙일 수 없어 재생성해야 한다. 반대로 QUIC를 쓰려면 보안 그룹이 없어야 한다.
- **NLB cross-zone 꺼짐**: AZ마다 타깃 수가 다르면 타깃당 부하가 크게 차이 난다. 켜면 AZ 간 전송 요금이 생긴다.
- **ip 타깃 + TCP에서 클라이언트 IP가 안 보임**: 기본값이 꺼짐이다. 켜거나 proxy protocol v2를 쓴다.
- **proxy protocol v2만 켜고 타깃 설정을 안 바꿈**: 헬스체크와 실제 요청이 모두 깨진다.
- **NLB 헬스체크가 TCP**: 포트만 열려 있으면 앱이 죽어도 healthy다. 가능하면 HTTP 헬스체크를 쓴다.
- **오래 사는 연결과 deregistration**: connection termination을 켜지 않으면 배포 후에도 옛 타깃에 연결이 남는다.
- **CLB 유지**: 새 기능(인증, WAF 연동 개선, HTTP/2 타깃 등)을 못 쓴다.

## 출처

- https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancers.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-listeners.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-security-groups.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/network/edit-target-group-attributes.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/network/target-group-health-checks.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/network/application-load-balancer-target.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-limits.html
- https://aws.amazon.com/about-aws/whats-new/2023/08/network-load-balancer-supports-security-groups/
- https://aws.amazon.com/about-aws/whats-new/2024/09/aws-network-load-balancer-tcp-idle-timeout/
- https://aws.amazon.com/about-aws/whats-new/2025/11/aws-network-load-balancer-quic-passthrough-mode/
- https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/introduction.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/gateway-load-balancers.html
- https://docs.aws.amazon.com/vpc/latest/privatelink/vpce-gateway-load-balancer.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/migrate-classic-load-balancer.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/classic/introduction.html

---
*조사 기준일: 2026-10-26. HCL 예시는 Terraform `validate`로 검사했다(결과는 README 참고). PrivateLink CIDR 중복 허용과 Network Firewall 내부 구조는 이번 조사에서 1차 출처로 재확인하지 않았다.*
