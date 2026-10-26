# ALB TLS·보안

> HTTPS 리스너에는 **ACM 인증서**를 붙이고, 보안 정책은 **콘솔 기본값인 `ELBSecurityPolicy-TLS13-1-2-Res-PQ-2025-09`**를 쓴다. CLI·CloudFormation·CDK·Terraform으로 만들 때 정책을 지정하지 않으면 **오래된 `ELBSecurityPolicy-2016-08`**이 적용된다. 앱 앞단의 보호는 **보안 그룹(타깃은 ALB 보안 그룹만 허용) + WAF + `drop_invalid_header_fields`**가 기본이다. 로그인은 ALB의 OIDC/Cognito 인증 액션으로, 기기 인증은 **mTLS**로 앱 코드 없이 처리할 수 있다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)·[02](./02-alb-routing-targets.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| TLS 종료(termination) | 암호화된 연결을 ALB에서 풀고, 그 뒤로는 ALB가 새 연결을 맺는 것 |
| ACM | AWS Certificate Manager. 인증서를 발급·갱신하는 서비스 |
| SNI | 클라이언트가 TLS 시작 때 접속할 도메인 이름을 알려 주는 확장. 한 리스너에 인증서 여러 개를 쓰게 해 줌 |
| 보안 정책(security policy) | 허용할 TLS 버전과 암호 스위트 묶음 |
| PQ(post-quantum) | 양자 컴퓨터에도 안전하도록 설계한 키 교환(ML-KEM)을 섞은 방식 |
| mTLS | 서버뿐 아니라 **클라이언트도 인증서를 제시**하는 TLS |
| Trust store | mTLS에서 클라이언트 인증서를 검증할 CA 인증서 묶음 |
| OIDC | OAuth 2.0 위의 로그인 표준. IdP(Google, Okta 등)가 사용자를 인증 |
| AWS WAF | 웹 요청을 규칙(SQL 인젝션, IP 평판 등)으로 차단하는 방화벽 서비스 |

## 인증서

출처: [HTTPS listener certificates](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/https-listener-certificates.html)

| 항목 | 내용 |
| --- | --- |
| 발급처 | ACM(권장, 자동 갱신) 또는 IAM에 올린 인증서 |
| 키 | ACM은 RSA 2048/3072/4096, ECDSA 전부 |
| 기본 인증서 | HTTPS 리스너당 **정확히 1개**. SNI가 없거나 맞는 인증서가 없을 때 사용 |
| 추가 인증서(SNI) | ALB당 25개(기본 인증서 제외) |
| 선택 순서 | ECDSA > RSA → 만료 안 됨 → 더 높은 SHA → 더 큰 키 → 유효 기간 |

- 맞는 이름의 인증서는 있는데 클라이언트와 호환되는 것이 없으면, **기본 인증서로 돌아가지 않고 핸드셰이크가 실패**한다.
- ACM 인증서는 **ALB와 같은 리전**에 있어야 한다. CloudFront용 인증서가 us-east-1에 있어야 하는 것과 혼동하기 쉽다(리전 규칙은 ACM 문서 기준, 확인 필요).

```bash
# 리스너에 SNI 인증서 추가
aws elbv2 add-listener-certificates \
  --listener-arn arn:aws:elasticloadbalancing:ap-northeast-2:111122223333:listener/app/my-alb/0123456789abcdef/0123456789abcdef \
  --certificates CertificateArn=arn:aws:acm:ap-northeast-2:111122223333:certificate/12345678-1234-1234-1234-123456789012
```

## 보안 정책

출처: [Security policies](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/describe-ssl-policies.html)

> "Console – The default security policy is ELBSecurityPolicy-TLS13-1-2-Res-PQ-2025-09." / "Other methods (for example, the AWS CLI, AWS CloudFormation, and the AWS CDK) – The default security policy is ELBSecurityPolicy-2016-08."

| 정책 | 쓰임 |
| --- | --- |
| `ELBSecurityPolicy-TLS13-1-2-Res-PQ-2025-09` | **AWS 권장**, 콘솔 기본. TLS 1.2·1.3, 제한된 암호 스위트, PQ 하이브리드 키 교환 |
| `ELBSecurityPolicy-TLS13-1-2-Res-FIPS-PQ-2025-09` | FIPS 요구가 있을 때 |
| `ELBSecurityPolicy-2016-08` | **IaC·CLI에서 지정하지 않을 때의 기본**. TLS 1.0·1.1까지 허용 |

- ALB는 **사용자 정의 보안 정책을 지원하지 않는다.** 제공되는 정책 중 고른다.
- 오래된 클라이언트(구형 단말, 레거시 시스템)가 있다면 바꾸기 전에 액세스 로그의 `ssl_protocol`·`ssl_cipher` 필드로 실제 사용 현황을 확인한다([06](./06-operations-cost-issues.md)).
- ALB와 타깃 사이(HTTPS 타깃 그룹)의 정책은 리스너 정책에 따라 정해진다.

```hcl
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-Res-PQ-2025-09" # 생략하면 2016-08
  certificate_arn   = aws_acm_certificate.main.arn

  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "not found"
      status_code  = "404"
    }
  }
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "redirect"
    redirect {
      protocol    = "HTTPS"
      port        = "443"
      status_code = "HTTP_301"
    }
  }
}
```

## mTLS(2023-11-26+)

출처: [Mutual authentication](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/mutual-authentication.html), [MutualAuthenticationAttributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_MutualAuthenticationAttributes.html)

| 모드 | ALB가 하는 일 | 앱이 할 일 |
| --- | --- | --- |
| `off`(기본) | 클라이언트 인증서 요청 안 함 | — |
| `passthrough` | 클라이언트 인증서 체인을 **검증 없이** `X-Amzn-Mtls-Clientcert` 헤더로 전달 | 앱이 직접 검증 |
| `verify` | trust store(S3에 둔 CA 묶음, 선택적으로 CRL)로 **검증**하고, 통과한 연결만 받음 | 헤더(`X-Amzn-Mtls-Clientcert-Subject` 등)로 누구인지 확인 |

- IoT 기기, 파트너 시스템 연동처럼 **클라이언트가 정해진 경우**에 맞다. 일반 브라우저 사용자에게는 쓰지 않는다.
- 제한: verify 모드 리스너는 ALB당 2개, trust store당 CA 25개, 체인 깊이 4, mTLS에서는 TLS 세션 재개 미지원.

## 로그인 인증: OIDC·Cognito

출처: [Authenticate users](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html)

ALB 규칙에 `authenticate-oidc`(Google, Okta, Entra ID 등) 또는 `authenticate-cognito` 액션을 `forward` 앞에 두면, 로그인하지 않은 사용자는 IdP 로그인 화면으로 보내고 로그인한 사용자만 앱에 전달한다.

| 항목 | 내용 |
| --- | --- |
| 리스너 | **HTTPS만** |
| 세션 | 쿠키(`AWSELBAuthSessionCookie`), 기본 7일 |
| 미인증 요청 | `authenticate`(기본, 로그인으로 보냄) / `allow` / `deny`(401) |
| 앱에 전달 | `x-amzn-oidc-data`(사용자 claim, **ES256 서명**), `x-amzn-oidc-accesstoken`, `x-amzn-oidc-identity` |
| 네트워크 | ALB가 IdP의 token·user-info 엔드포인트에 **IPv4로** 나갈 수 있어야 함. 내부 ALB는 NAT 게이트웨이 필요 |

- **앱은 `x-amzn-oidc-data`의 서명을 검증해야 한다.** ALB를 거치지 않는 경로(타깃 직접 접근)가 있으면 헤더를 위조할 수 있기 때문이다. 서명된 것은 이 헤더 하나뿐이다.
- 내부 관리 도구(사내 대시보드 등)에 로그인을 붙이는 가장 간단한 방법이다. 권한(누가 무엇을 할 수 있는지)은 여전히 앱이 판단한다.
- API 클라이언트가 이미 토큰을 갖고 있다면 로그인 리다이렉트가 아니라 `jwt-validation` 액션(2025-11-21+, RS256만)을 쓴다([02](./02-alb-routing-targets.md#액션)).

## AWS WAF

출처: [Integrations](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-integrations.html)

```hcl
resource "aws_wafv2_web_acl_association" "alb" {
  resource_arn = aws_lb.main.arn
  web_acl_arn  = aws_wafv2_web_acl.main.arn
}
```

- 시작점으로 AWS 관리형 규칙 `AmazonIpReputationList`, `CommonRuleSet`, `KnownBadInputsRuleSet`을 쓴다(콘솔의 사전 정의 web ACL과 같은 구성).
- **WAF가 응답하지 않으면 기본으로 500을 돌려주고 요청을 막는다(fail-closed).** `waf.fail_open.enabled = true`로 바꿀 수 있지만 보안과 가용성 중 무엇을 택할지의 결정이다.
- 새 규칙은 **Count 모드로 먼저 켜고** 로그를 확인한 뒤 Block으로 바꾼다(오탐으로 정상 요청을 막는 사고 방지, 이 레포의 정리).

## 네트워크 보안

출처: [Security groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-update-security-groups.html), [CreateLoadBalancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_CreateLoadBalancer.html)

```
인터넷 ──443──▶ [sg-alb: 0.0.0.0/0 → 443 허용]
                     │ 8080 (트래픽·헬스체크)
                     ▼
              [sg-app: sg-alb → 8080만 허용]   ← IP 대역이 아니라 보안 그룹 참조
```

| 항목 | 권장 |
| --- | --- |
| ALB 보안 그룹 | 인터넷용: 0.0.0.0/0(필요하면 ::/0)에서 리스너 포트. 내부용: VPC CIDR 등 필요한 대역만 |
| 타깃 보안 그룹 | **ALB 보안 그룹을 소스로** 트래픽 포트·헬스체크 포트만 허용 |
| scheme | `internet-facing`(기본) / `internal`. 내부 ALB의 DNS 이름은 **공개 DNS에서 사설 IP로** 풀림 |
| 서브넷 | **서로 다른 AZ 2개 이상**, 서브넷마다 최소 `/27`·여유 IP 8개 |
| IP 주소 타입 | `ipv4` / `dualstack` / `dualstack-without-public-ipv4`(공인 IPv4 없음 → 공인 IPv4 요금 절감) |
| `drop_invalid_header_fields` | 기본 false. 켜는 것 권장(이름이 `[-A-Za-z0-9]+`가 아닌 헤더 제거) |
| desync 방어 | 기본 `defensive` 유지. `monitor`로 낮추지 않기 |

- CloudFront 뒤에 ALB를 둔다면, ALB 보안 그룹을 **CloudFront 관리형 prefix list**로 제한하고 CloudFront가 붙이는 비밀 헤더를 리스너 규칙으로 확인하는 패턴이 흔하다(CloudFront 문서 기준, 이번 조사에서 1차 출처로 재확인하지 않음).
- `dualstack-without-public-ipv4`에서 OIDC 인증을 쓰면 IdP로 나갈 IPv4 경로(NAT)가 필요하다.

## 이슈 / 주의

- **IaC에서 `ssl_policy` 생략**: TLS 1.0·1.1이 열린 `2016-08`이 적용된다. 보안 점검에서 가장 흔히 걸리는 항목이다.
- **타깃 보안 그룹을 VPC 전체에 열기**: ALB를 우회해 타깃에 직접 접근할 수 있고, OIDC 헤더 위조가 가능해진다.
- **OIDC 헤더를 검증하지 않고 신뢰**: 서명을 확인하지 않으면 우회 경로가 생겼을 때 막을 수 없다.
- **WAF 규칙을 바로 Block**: 오탐으로 정상 트래픽을 막는다.
- **인터넷용 ALB에 내부 관리 경로 노출**: `/admin`은 `source-ip` 조건, 인증 액션, 또는 별도 내부 ALB로 분리한다.
- **내부 ALB에서 OIDC 인증**: NAT가 없으면 IdP와 통신하지 못해 로그인이 실패한다.

## 출처

- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/https-listener-certificates.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/describe-ssl-policies.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/mutual-authentication.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/configuring-mtls-with-elb.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_MutualAuthenticationAttributes.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-authenticate-users.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-verify-jwt.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-integrations.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-update-security-groups.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_CreateLoadBalancer.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-limits.html

---
*조사 기준일: 2026-10-26. HCL 예시는 Terraform `validate`로 검사했다(결과는 README 참고). PQ 정책이 콘솔 기본값이 된 날짜는 찾지 못했다.*
