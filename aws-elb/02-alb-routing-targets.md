# ALB 라우팅·타깃 그룹·헬스체크

> ALB는 **리스너 규칙을 우선순위 숫자가 낮은 것부터** 평가하고, 처음 맞는 규칙의 액션을 실행한다. 맞는 규칙이 없으면 **default rule**이 실행된다. 요청을 받는 쪽은 **타깃 그룹**이고, 헬스체크를 통과한 타깃에만 보낸다. 다만 **모든 타깃이 unhealthy면 전부에게 보낸다(fail-open).** 배포 중 끊김은 대부분 **deregistration delay(기본 300초)**와 앱의 종료 처리가 맞지 않아서 생긴다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)의 용어(리스너, 규칙, 타깃 그룹, 헬스체크)를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| 조건(condition) | 규칙이 요청과 맞는지 판단하는 기준(호스트, 경로, 헤더 등) |
| 액션(action) | 규칙이 맞을 때 할 일(forward, redirect, fixed-response, 인증) |
| Weighted target group | 한 규칙에서 여러 타깃 그룹에 비율(가중치)로 나눠 보내는 것 |
| Transform | 요청을 타깃에 보내기 전에 Host 헤더나 URL을 바꾸는 기능 |
| Draining | 타깃을 빼는 중. 새 요청은 안 보내고 진행 중인 요청만 마무리 |
| Stickiness | 같은 클라이언트를 계속 같은 타깃으로 보내는 것(쿠키 사용) |
| Cross-zone load balancing | 다른 AZ에 있는 타깃에도 고르게 나눠 보내는 것 |

## 리스너와 규칙

출처: [Listeners](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html), [Listener rules](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-rules.html), [CreateRule](https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_CreateRule.html)

```
HTTPS :443 리스너
 ├─ 우선순위 10  host = api.example.com                → forward: tg-api
 ├─ 우선순위 20  path = /admin/*  AND source-ip = 사무실 → forward: tg-admin
 ├─ 우선순위 30  path = /old/*                         → redirect: /new/#{path}
 └─ default rule                                      → fixed-response 404
```

| 항목 | 값 |
| --- | --- |
| 리스너 프로토콜 | HTTP, HTTPS(포트 1–65535). HTTPS는 인증서 1개 이상 필요 |
| 우선순위 | 1–50000, **낮은 숫자부터** 평가. 같은 리스너에서 중복 불가 |
| default rule | 삭제·우선순위 변경 불가, 조건·transform 불가, 항상 마지막 |
| 규칙 수 | ALB당 100개(default 제외, 증가 요청 가능) |

### 조건

출처: [Rule condition types](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-condition-types.html)

| 조건 | 대소문자 | 와일드카드(`*` `?`) | 정규식 | 메모 |
| --- | --- | --- | --- | --- |
| `host-header` | 구분 안 함 | 가능 | 가능 | `*.example.com`은 `example.com`과 맞지 않음 |
| `path-pattern` | **구분함** | 가능 | 가능 | 경로만 봄(쿼리 문자열 제외), 경로는 바꾸지 않음 |
| `http-header` | 구분 안 함 | 값에만 | 가능 | |
| `http-request-method` | 구분함 | 불가 | — | GET과 HEAD는 같게 라우팅하라고 권장 |
| `query-string` | 구분 안 함 | 가능 | — | |
| `source-ip` | — | 불가 | — | CIDR. **`X-Forwarded-For` 값은 보지 않음** |

- 한 규칙에 `host-header`, `path-pattern`, `http-request-method`, `source-ip`는 **0~1개씩**, `http-header`와 `query-string`은 여러 개 둘 수 있다. 여러 조건은 AND다.
- 비교 문자열은 **조건당 3개, 규칙당 5개**까지다.
- 규칙당 와일드카드 개수는 조건 문서에는 5개, 쿼터 페이지에는 6개로 적혀 있어 **확인 필요**.

### 액션

출처: [Rule action types](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-action-types.html)

| 액션 | 하는 일 | 메모 |
| --- | --- | --- |
| `forward` | 타깃 그룹으로 전달 | 타깃 그룹 최대 5개에 가중치 0–999로 분배 |
| `redirect` | 301/302 응답 | `#{protocol}` `#{host}` `#{port}` `#{path}` `#{query}` 사용. HTTPS→HTTP 불가 |
| `fixed-response` | 정해진 상태 코드·본문 응답 | 점검 페이지, 404 기본 규칙에 씀 |
| `authenticate-oidc` / `authenticate-cognito` | 로그인 후 전달 | HTTPS 리스너만([03](./03-tls-security.md#로그인-인증-oidccognito)) |
| `jwt-validation` | 액세스 토큰(JWT) 서명·claim 검증 | HTTPS 리스너만, RS256만. 2025-11-21 추가 |

규칙마다 `forward`·`redirect`·`fixed-response` 중 **하나가 정확히 마지막에** 와야 한다. 인증 액션은 그 앞에 둔다.

### 가중치 라우팅: 카나리·블루/그린

```hcl
resource "aws_lb_listener_rule" "api_canary" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 10

  action {
    type = "forward"
    forward {
      target_group {
        arn    = aws_lb_target_group.api_blue.arn
        weight = 90
      }
      target_group {
        arn    = aws_lb_target_group.api_green.arn
        weight = 10
      }
      stickiness {
        enabled  = true
        duration = 300 # 같은 사용자는 5분 동안 같은 타깃 그룹으로
      }
    }
  }

  condition {
    host_header {
      values = ["api.example.com"]
    }
  }
}
```

- 가중치 라우팅에는 **자동 페일오버가 없다.** green이 전부 unhealthy여도 10%는 계속 green으로 간다.
- 타깃 그룹 단위 stickiness는 `AWSALBTG` 쿠키를 쓰고, 기간은 1초–7일이다.

### Transform: Host·URL 바꾸기(2025-10-15+)

출처: [Transforms](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-transforms.html)

- `host-header-rewrite`, `url-rewrite`를 규칙당 하나씩, 정규식 + 치환으로 쓴다. 예: `/api/v1/(.*)` → `/$1`.
- 라우팅 결정은 **원래 요청 기준**이고, 바뀐 값은 타깃에 보낼 때만 쓴다.
- 정규식이 안 맞으면 원래 요청을 그대로 보내고, 맞았는데 변환에 실패하면 500을 돌려준다.
- 이전에는 경로를 바꾸려면 앱이나 프록시(nginx 등)에서 처리해야 했다.

## 타깃 그룹

출처: [Target groups](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html)

| 타깃 타입 | 등록하는 것 | 쓰는 곳 | 메모 |
| --- | --- | --- | --- |
| `instance` | EC2 인스턴스 ID | EC2, Auto Scaling 그룹 | 인스턴스의 기본 네트워크 인터페이스로 보냄 |
| `ip` | IP 주소 | **ECS awsvpc, EKS Pod**, 온프레미스(VPN·Direct Connect) | VPC 서브넷 또는 사설 대역만. 공인 IP 불가 |
| `lambda` | Lambda 함수 1개 | 서버리스 | HTTP/2·gRPC 불가, 헬스체크 기본값이 다름 |

- **타깃 타입은 만든 뒤 바꿀 수 없다.** 바꾸려면 새 타깃 그룹을 만들어야 한다(→ Terraform 교체 주의는 [05](./05-eks-ecs-iac.md)).
- 타깃까지의 프로토콜 버전은 HTTP/1.1(기본), HTTP/2, gRPC다. HTTP/2·gRPC는 HTTPS 리스너와 `forward` 액션에서만 된다.
- ALB는 **타깃의 인증서를 검증하지 않는다.** 타깃이 자체 서명 인증서를 써도 HTTPS로 연결된다.

### 헬스체크

출처: [Health checks](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html)

| 설정 | 범위 | 기본값(instance·ip) |
| --- | --- | --- |
| 프로토콜·포트 | HTTP/HTTPS, GET | HTTP, 트래픽 포트 |
| 경로 | — | `/` (gRPC는 `/AWS.ALB/healthcheck`) |
| 간격(interval) | 5–300초 | 30초(lambda 35초) |
| 타임아웃 | 2–120초 | 5초(lambda 30초) |
| healthy 임계값 | 2–10회 | 5회 |
| unhealthy 임계값 | 2–10회 | 2회 |
| 성공 코드(matcher) | 200–499 | 200(gRPC는 12) |

- 기본값이면 unhealthy 판정에 약 **1분**(30초 × 2회), 다시 healthy가 되기까지 약 **2분 30초**(30초 × 5회)가 걸린다(설정값에서 계산한 값).
- 새로 등록한 타깃은 **한 번만 통과하면** healthy가 된다.
- 헬스체크 경로는 **DB 같은 외부 의존성을 확인하지 않는 가벼운 엔드포인트**가 일반적이다. DB가 잠깐 느려질 때 모든 타깃이 동시에 unhealthy가 되는 것을 피하기 위해서다(이 레포의 정리).

### 모두 unhealthy면: fail-open

> "If a target group contains only unhealthy registered targets, the load balancer routes requests to all those targets, regardless of their health status" — [Health checks](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html)

- 헬스체크 설정이 잘못돼 전부 unhealthy여도 트래픽은 계속 간다. 그래서 **"서비스는 되는데 UnHealthyHostCount가 전부"**인 상황이 생길 수 있다.
- 타깃 그룹 속성 `target_group_health.unhealthy_state_routing.minimum_healthy_targets.count`(기본 1)·`.percentage`(기본 off)로 기준을 바꿀 수 있다.

### 배포와 연결 종료

출처: [Target group attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html)

| 속성 | 범위 | 기본값 | 하는 일 |
| --- | --- | --- | --- |
| `deregistration_delay.timeout_seconds` | 0–3600초 | **300초** | 타깃을 뺄 때 진행 중 요청을 기다리는 시간 |
| `slow_start.duration_seconds` | 30–900초 | 0(꺼짐) | 새 타깃에 트래픽을 천천히 늘림 |

- 진행 중 요청이 없으면 바로 빠지지만, 상태는 타임아웃까지 `draining`으로 보인다.
- **API 응답이 몇 초 안에 끝나는 서비스**라면 300초는 배포를 불필요하게 늦춘다. 가장 긴 요청 시간보다 조금 길게(예: 30–60초) 줄이는 경우가 많다.
- 앱은 종료 신호(SIGTERM)를 받으면 **바로 죽지 말고** draining이 반영될 시간만큼 기다린 뒤 종료해야 502를 피한다(→ [05](./05-eks-ecs-iac.md), [06](./06-operations-cost-issues.md)).

### 분배 알고리즘

| `load_balancing.algorithm.type` | 동작 | 제약 |
| --- | --- | --- |
| `round_robin`(기본) | 차례대로 | — |
| `least_outstanding_requests` | 처리 중 요청이 가장 적은 타깃 | slow start와 함께 못 씀 |
| `weighted_random` | 가중 무작위. `anomaly_mitigation = on`이면 이상 타깃의 가중치를 자동으로 낮춤(Automatic Target Weights, 2023-11-26) | slow start·stickiness와 함께 못 씀, healthy 타깃 3개 이상 필요 |

요청마다 처리 시간이 크게 다르면 `least_outstanding_requests`가 유리하다.

### Stickiness

| 타입 | 쿠키 | 기간 |
| --- | --- | --- |
| `lb_cookie` | ALB가 만드는 `AWSALB` | 1초–7일, 기본 1일 |
| `app_cookie` | 앱이 정한 쿠키 이름을 기준으로 ALB가 `AWSALBAPP-*` 생성 | 1초–7일, 기본 1일 |

- 기본은 꺼져 있다. 세션을 서버 메모리에 두는 앱에만 쓰고, 가능하면 세션을 Redis 같은 외부 저장소로 옮겨 **stickiness 없이** 운영하는 편이 고르게 분배된다.
- target group의 cross-zone을 끄면 stickiness를 쓸 수 없다.

### Cross-zone load balancing

> "With Application Load Balancers, cross-zone load balancing is on by default and cannot be changed at the load balancer level" — [Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html)

- ALB는 **항상 켜져 있고**, 타깃 그룹 단위로만 끌 수 있다(2022-11+). NLB·GWLB는 기본이 꺼짐이라 차이에 주의한다([04](./04-nlb-gwlb.md)).
- 켜져 있으면 AZ마다 타깃 수가 달라도 타깃당 트래픽이 고르게 된다.

## 연결 관련 속성

출처: [Load balancer attributes](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html), [X-Forwarded headers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/x-forwarded-headers.html)

| 속성 | 기본값 | 메모 |
| --- | --- | --- |
| `idle_timeout.timeout_seconds` | 60초(1–4000) | 데이터가 오가지 않으면 연결을 닫는 시간. **앱의 keep-alive 타임아웃은 이보다 길게** |
| `client_keep_alive.seconds` | 3600초(60–604800) | 클라이언트 연결의 최대 유지 시간 |
| `routing.http.xff_header_processing.mode` | `append` | 클라이언트 IP를 `X-Forwarded-For`에 덧붙임 |
| `routing.http.preserve_host_header.enabled` | false | Host 헤더를 그대로 전달 |
| `routing.http.desync_mitigation_mode` | `defensive` | HTTP desync(요청 밀반입) 방어 수준 |
| `routing.http.drop_invalid_header_fields.enabled` | false | 이름이 잘못된 헤더 제거(보안상 켜는 것 권장, [03](./03-tls-security.md)) |
| `routing.http2.enabled` | true | 클라이언트–ALB HTTP/2 |
| `deletion_protection.enabled` | false | 삭제 보호 |

- 타깃이 보는 **소스 IP는 ALB 노드의 IP**다. 실제 클라이언트 IP는 `X-Forwarded-For`, 원래 프로토콜은 `X-Forwarded-Proto`로 읽는다.
- 앱의 keep-alive 타임아웃이 ALB idle timeout보다 짧으면, ALB가 이미 닫힌 연결로 요청을 보내 **간헐적 502**가 난다([06](./06-operations-cost-issues.md)). 이 설명은 AWS 문제 해결 문서의 502 원인 목록에 근거한 일반적 정리다.

## CLI로 상태 보기

```bash
# 규칙 목록(우선순위 순)
aws elbv2 describe-rules --listener-arn arn:aws:elasticloadbalancing:ap-northeast-2:111122223333:listener/app/my-alb/0123456789abcdef/0123456789abcdef \
  --query 'Rules[].{P:Priority,Cond:Conditions[].Field,Act:Actions[].Type}' --output table

# 타깃 상태와 unhealthy 이유
aws elbv2 describe-target-health --target-group-arn arn:aws:elasticloadbalancing:ap-northeast-2:111122223333:targetgroup/tg-api/0123456789abcdef \
  --query 'TargetHealthDescriptions[].{Id:Target.Id,State:TargetHealth.State,Reason:TargetHealth.Reason}'

# deregistration delay 줄이기
aws elbv2 modify-target-group-attributes --target-group-arn arn:aws:elasticloadbalancing:ap-northeast-2:111122223333:targetgroup/tg-api/0123456789abcdef \
  --attributes Key=deregistration_delay.timeout_seconds,Value=30
```

## 이슈 / 주의

- **규칙 우선순위 실수**: 넓은 조건(`/*`)이 좁은 조건보다 앞에 있으면 좁은 규칙이 영원히 실행되지 않는다.
- **`source-ip`로 사용자 IP 제한**: CloudFront나 다른 프록시 뒤에 있으면 `source-ip`는 프록시 IP다. XFF 값은 보지 않는다.
- **가중치 라우팅은 페일오버가 아니다.** 한쪽이 죽어도 비율대로 계속 보낸다.
- **fail-open**: 헬스체크가 전부 실패해도 트래픽이 간다. 헬스체크 설정 오류를 알아차리기 어렵다.
- **deregistration delay 300초 그대로 두기**: 배포가 느려지고, 반대로 앱이 SIGTERM에 바로 종료하면 502가 난다.
- **헬스체크에 DB 확인 넣기**: DB 장애가 전체 타깃 unhealthy로 번진다(fail-open 동작과 겹치면 더 혼란스럽다).

## 출처

- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-rules.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-condition-types.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-action-types.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-transforms.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/listener-verify-jwt.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/APIReference/API_CreateRule.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-target-groups.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-target-group-attributes.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/application-load-balancers.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/x-forwarded-headers.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-limits.html
- https://docs.aws.amazon.com/elasticloadbalancing/latest/application/doc-history.html

---
*조사 기준일: 2026-10-26. HCL 예시는 Terraform `validate`로 검사했다(결과는 README 참고). ARN은 예시 값이다.*
