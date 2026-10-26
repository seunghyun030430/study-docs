# EKS·ECS 연동과 Terraform

> EKS에서는 **AWS Load Balancer Controller**(LBC)가 Ingress로 ALB를, `type: LoadBalancer` Service로 NLB를 만든다. 타깃 타입은 **`ip`**(Pod 직접)를 쓰고, 무중단 배포를 위해 **readiness gate + preStop sleep + 짧은 deregistration delay**를 함께 맞춘다. LBC의 기본 scheme은 **internal**이고 기본 target-type은 **instance**라 명시하는 편이 안전하다. ECS는 awsvpc면 타깃 타입이 반드시 `ip`이고, 2025-07부터 **ECS 자체 blue/green**이 ALB 리스너 규칙을 바꿔 준다. Terraform에서는 **`ssl_policy`와 헬스체크 기본값이 콘솔·API와 다르다**는 점, 타깃 그룹의 많은 인자가 **교체(replace)**를 일으킨다는 점을 주의한다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[04](./04-nlb-gwlb.md)의 용어와 Kubernetes 기본 개념(Pod, Service, Ingress, Helm)을 안다고 가정한다. IRSA·Pod Identity는 [aws-iam/05](../aws-iam/05-workload-identity.md#eks-pod-identity-vs-irsa)를 참고한다.

| 용어 | 뜻 |
| --- | --- |
| AWS Load Balancer Controller(LBC) | Kubernetes 리소스를 보고 ALB·NLB·타깃 그룹을 만들고 Pod를 등록하는 컨트롤러 |
| IngressClass | 어떤 컨트롤러가 Ingress를 처리할지 정하는 리소스 |
| IngressGroup | 여러 Ingress가 ALB 하나를 같이 쓰게 묶는 LBC 기능(`group.name`) |
| TargetGroupBinding | 이미 있는 타깃 그룹에 Pod를 등록하게 하는 LBC 사용자 정의 리소스 |
| Pod readiness gate | 타깃 그룹에서 healthy가 될 때까지 Pod를 Ready로 보지 않게 하는 조건 |
| preStop hook | 컨테이너 종료 직전에 실행하는 명령(종료를 늦추는 데 씀) |
| EKS Auto Mode | 로드 밸런싱 등 일부 컴포넌트를 AWS가 관리하는 EKS 모드 |
| Blue/green 배포 | 새 버전(green)을 따로 띄운 뒤 트래픽을 한 번에 또는 단계적으로 옮기는 배포 |

## EKS: AWS Load Balancer Controller

출처: [EKS user guide: LBC](https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html), [LBC installation](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/deploy/installation.md)

최신 릴리스는 **v3.6.2(2026-10-25)**다.

| Kubernetes 리소스 | 만들어지는 것 |
| --- | --- |
| `Ingress`(IngressClass `alb`) | **ALB** + 리스너·규칙·타깃 그룹 |
| `Service` `type: LoadBalancer` | **NLB**(v2.5부터 웹훅이 `spec.loadBalancerClass: service.k8s.aws/nlb`를 자동 설정) |
| `Gateway` + `HTTPRoute`/`GRPCRoute` | ALB(Gateway API, **v3.0.0부터 GA**) |
| `Gateway` + `TCPRoute`/`UDPRoute`/`TLSRoute` | NLB |
| `TargetGroupBinding` | 기존 타깃 그룹에 Pod 등록만(ALB는 밖에서 관리) |

### 설치

```bash
# IAM 정책 (버전에 맞춘 정책 파일 사용)
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v3.6.2/docs/install/iam_policy.json
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam_policy.json

# 서비스 계정에 IAM role 연결(IRSA 또는 Pod Identity)을 만든 뒤
helm repo add eks https://aws.github.io/eks-charts
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system \
  --set clusterName=my-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller
```

- `helm install`은 CRD를 적용하지만 **`helm upgrade`는 CRD를 갱신하지 않는다.** 업그레이드 때 CRD를 따로 적용한다.
- 자격증명은 IRSA, Pod Identity, 노드 role 중 하나다. 설치 문서 본문은 IRSA를, 같은 문서의 소제목은 Pod Identity를 "Recommended"로 적고 있어 문서 안에서 엇갈린다(2026-10-26 기준). 노드 role에 붙이면 그 노드의 모든 Pod가 권한을 갖게 되므로 피한다.

### 서브넷 찾기

LBC는 서브넷 태그로 ALB·NLB를 어디에 만들지 정한다.

| 태그 | 값 | 용도 |
| --- | --- | --- |
| `kubernetes.io/role/elb` | `1` | internet-facing용 퍼블릭 서브넷 |
| `kubernetes.io/role/internal-elb` | `1` | internal용 프라이빗 서브넷 |

v2.12.1부터는 태그가 없으면 라우팅 테이블(IGW 경로 유무)로 퍼블릭·프라이빗을 판단하지만, 태그를 명시하는 편이 예측 가능하다.

### Ingress 예시

출처: [Ingress annotations](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/guide/ingress/annotations.md)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api
  namespace: prod
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing        # 기본값 internal
    alb.ingress.kubernetes.io/target-type: ip                # 기본값 instance
    alb.ingress.kubernetes.io/group.name: prod-public        # 여러 Ingress가 ALB 하나 공유
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:ap-northeast-2:111122223333:certificate/12345678-1234-1234-1234-123456789012
    alb.ingress.kubernetes.io/ssl-policy: ELBSecurityPolicy-TLS13-1-2-Res-PQ-2025-09
    alb.ingress.kubernetes.io/healthcheck-path: /health
    alb.ingress.kubernetes.io/target-group-attributes: deregistration_delay.timeout_seconds=30
    alb.ingress.kubernetes.io/load-balancer-attributes: deletion_protection.enabled=true
spec:
  ingressClassName: alb
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api
                port:
                  number: 80
```

| annotation | 주의 |
| --- | --- |
| `scheme` | **기본 internal.** 바꾸면 ALB가 **새로 만들어진다**(DNS 이름이 바뀜) |
| `target-type` | 기본 `instance`(NodePort 경유). `ip`는 Pod로 직접 보내고, stickiness·readiness gate에 필요 |
| `group.name` | 같은 그룹의 Ingress가 ALB를 공유. **그룹 안 누구든 공유 설정(WAF 등)을 바꿀 수 있음** |
| `ssl-redirect`, `listen-ports` | 그룹 전체에 영향 |
| `certificate-arn` | 첫 번째가 기본 인증서, 나머지는 SNI |
| `wafv2-acl-arn` | 리전 WAF만 |
| `load-balancer-attributes: deletion_protection.enabled=true` | 켜면 컨트롤러도 ALB를 지우지 못함 |

**Ingress를 지우면 ALB도 지워진다.** EKS Auto Mode 문서는 이를 명시하고, 자체 설치 LBC도 삭제 보호를 켜지 않으면 같은 동작을 한다(LBC 문서에서 명시 문장은 찾지 못함, 삭제 보호 설명에서 유추). ALB DNS 이름을 Route 53 등에서 직접 참조한다면 삭제 보호를 켠다.

### NLB Service 예시

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mqtt
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing   # 기본 internal (v2.2.0+)
spec:
  type: LoadBalancer
  selector:
    app: mqtt
  ports:
    - port: 8883
      targetPort: 8883
      protocol: TCP
```

ip 타깃 + TCP는 클라이언트 IP 보존이 기본 꺼짐이다([04](./04-nlb-gwlb.md#클라이언트-ip-보존)).

### 무중단 배포: readiness gate와 preStop

출처: [Pod readiness gate](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/deploy/pod_readiness_gate.md), [EKS best practices: load balancing](https://docs.aws.amazon.com/eks/latest/best-practices/load-balancing.html)

**문제**: Pod는 Kubernetes에서 Ready가 되는 시점과 ALB 타깃 그룹에서 healthy가 되는 시점이 다르다. 종료할 때도 SIGTERM을 받는 시점과 ALB에서 빠지는 시점이 다르다.
- 롤링 업데이트가 너무 빨리 진행되면 **새 Pod가 아직 healthy가 아닌데 옛 Pod가 모두 사라질** 수 있다.
- 종료 중인 Pod로 요청이 가면 502가 난다.

```bash
# 1) 시작: 타깃 그룹에서 healthy가 될 때까지 Pod를 Ready로 보지 않게
kubectl label namespace prod elbv2.k8s.aws/pod-readiness-gate-inject=enabled
```

```yaml
# 2) 종료: SIGTERM 전에 잠시 대기해 ALB에서 빠질 시간을 줌
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: api
      lifecycle:
        preStop:
          exec:
            command: ["sleep", "10"]
```

| 설정 | 하는 일 | 주의 |
| --- | --- | --- |
| readiness gate | 새 Pod가 타깃 그룹에서 healthy가 될 때까지 롤링을 멈춤 | `target-type: ip`에서만. **라벨을 붙인 뒤 만들어진 Pod**에만 적용 |
| preStop `sleep` | LBC가 타깃을 빼는(비동기) 동안 앱이 계속 요청을 처리 | 컨테이너 이미지에 `sleep`이 있어야 함 |
| deregistration delay | 빠진 타깃의 진행 중 요청을 기다림 | 기본 300초. 짧은 API는 30초 정도로 |
| terminationGracePeriodSeconds | preStop + 앱 종료 시간보다 길게 | 기본 30초 |

LBC는 Pod IP를 **비동기로** 등록 해제한다. Kubernetes API나 ELB API가 느리면 시간이 더 걸리므로 preStop 대기가 필요하다(EKS best practices 문서).

### EKS Auto Mode와의 차이

출처: [Auto Mode ALB](https://docs.aws.amazon.com/eks/latest/userguide/auto-configure-alb.html), [Auto Mode NLB](https://docs.aws.amazon.com/eks/latest/userguide/auto-configure-nlb.html)

| 항목 | 자체 설치 LBC | EKS Auto Mode |
| --- | --- | --- |
| 설치 | Helm으로 직접 | 내장 |
| IngressClass controller | `ingress.k8s.aws/alb` | `eks.amazonaws.com/alb` |
| 기본 target-type | `instance` | **`ip`** |
| `group.name` annotation | 지원 | **미지원**(IngressClass로만 그룹 지정) |
| IngressClass annotation | 사용 | 사용 불가(IngressClassParams로 설정) |
| NLB | `aws-load-balancer-type` annotation | `spec.loadBalancerClass: eks.amazonaws.com/nlb`(클러스터 기본) |

같은 YAML을 두 환경에 그대로 쓰면 동작이 달라질 수 있다.

## ECS

출처: [ECS with ALB](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/alb.html), [Connection draining](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/load-balancer-connection-draining.html), [CreateService](https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_CreateService.html)

| 항목 | 내용 |
| --- | --- |
| 타깃 타입 | `awsvpc` 네트워크 모드(Fargate 포함)면 **반드시 `ip`** |
| 타깃 그룹 | **서비스마다 따로** |
| `healthCheckGracePeriodSeconds` | 태스크 시작 후 헬스체크 실패를 무시하는 시간. **기본 0** → 시작이 느린 앱은 계속 재시작될 수 있음 |
| 헬스체크 실패 | 태스크를 멈추고 새로 띄움 |
| deregistration delay | 기본 300초. AWS는 응답이 1초 안에 끝나는 서비스에 5초를 예로 듦(오래 걸리는 요청이 있으면 안 됨) |
| 종료 신호 | SIGTERM 후 기본 30초 뒤 SIGKILL(`ECS_CONTAINER_STOP_TIMEOUT`) |
| 서비스 간 통신 | 내부 호출은 **Service Connect**(추가 요금 없음), 외부 트래픽은 ALB·NLB |

### Blue/green 배포

- **ECS 자체 blue/green(2025-07-17+)**: 서비스에 타깃 그룹 2개(primary, alternate)와 리스너 규칙을 지정하고 `deploymentConfiguration.strategy: BLUE_GREEN`으로 설정한다. ECS가 **리스너 규칙을 alternate로 바꿔** 트래픽을 옮기고, bake time 동안 문제가 있으면 되돌린다. 테스트 리스너와 Lambda 수명 주기 훅도 쓸 수 있다. 현재 문서는 linear·canary 전략도 다룬다(추가 날짜 확인 필요).
- **CodeDeploy**: 기존 방식. `CODE_DEPLOY` 배포 컨트롤러 + 타깃 그룹 쌍.
- 서비스를 만든 뒤 로드 밸런서 설정은 **콘솔에서 바꿀 수 없다**(CLI·SDK·CloudFormation으로 rolling 배포 컨트롤러일 때만 가능).

## Terraform

출처: Terraform Registry [`aws_lb`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb), [`aws_lb_listener`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener), [`aws_lb_target_group`](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group)

AWS provider 최신은 6.70.0(2026-10-22)이다. `aws_alb*`는 `aws_lb*`의 별칭이고 기능이 같다.

```hcl
resource "aws_lb" "main" {
  name                       = "prod-public"
  load_balancer_type         = "application" # 기본값
  internal                   = false
  security_groups            = [aws_security_group.alb.id]
  subnets                    = ["subnet-0123456789abcdef0", "subnet-0fedcba9876543210"]
  drop_invalid_header_fields = true
  enable_deletion_protection = true
  idle_timeout               = 60

  access_logs {
    bucket  = "acme-alb-logs-111122223333"
    prefix  = "prod-public"
    enabled = true # bucket만 적으면 기본 false
  }
}

resource "aws_lb_target_group" "api" {
  name_prefix          = "api-" # 6자 이하. 교체 시 이름 충돌 방지
  port                 = 8080
  protocol             = "HTTP"
  target_type          = "ip"
  vpc_id               = "vpc-0123456789abcdef0"
  deregistration_delay = 30

  health_check {
    path                = "/health"
    matcher             = "200"
    interval            = 15
    timeout             = 5
    healthy_threshold   = 2
    unhealthy_threshold = 2
  }

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_lb_listener_rule" "api" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 10

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }

  condition {
    host_header {
      values = ["api.example.com"]
    }
  }
}
```

HTTPS·HTTP 리스너 예시는 [03](./03-tls-security.md#보안-정책)에 있다.

### Terraform에서 특히 주의할 기본값

| 항목 | 콘솔·API 기본 | Terraform 기본 |
| --- | --- | --- |
| `ssl_policy` | 콘솔: `ELBSecurityPolicy-TLS13-1-2-Res-PQ-2025-09` | **`ELBSecurityPolicy-2016-08`** |
| `health_check.healthy_threshold` | 5 | **3** |
| `health_check.unhealthy_threshold` | 2 | **3** |
| `health_check.timeout` | 5초(ALB HTTP) | **6초**(HTTP) |
| `access_logs.enabled` | — | bucket을 지정해도 **false** |
| `target_type` | — | `instance` |

Terraform 기본값은 Registry 문서 기준이다. 헬스체크 값은 **항상 명시**하는 편이 콘솔·문서와 헷갈리지 않는다.

### 교체(replace)를 일으키는 인자

`aws_lb_target_group`의 `name`, `name_prefix`, `port`, `protocol`, `protocol_version`, `target_type`, `vpc_id`, `ip_address_type`은 바꾸면 **타깃 그룹이 새로 만들어진다**(Registry 문서의 "Forces new resource").

- 고정 `name`을 쓰면 새 타깃 그룹이 같은 이름으로 만들어질 수 없어 실패하고, 리스너가 참조 중이라 먼저 지울 수도 없다.
- 그래서 **`name_prefix` + `create_before_destroy`**를 함께 쓰는 패턴이 흔하다. 공식 Registry 문서에 이 조합 권장은 없고, 위 사실에서 나온 관행이다([terraform/02](../terraform/02-hcl.md#lifecycle)).
- NLB의 `aws_lb`는 보안 그룹을 처음 추가하거나 모두 빼거나, 서브넷을 빼면 **로드 밸런서가 재생성**된다. 고정 IP가 바뀌므로 특히 주의한다.

### 누가 무엇을 관리하나

| 구성 | Terraform | Kubernetes(LBC) |
| --- | --- | --- |
| LBC가 전부 | VPC, 서브넷 태그, LBC IAM role | ALB, 리스너, 규칙, 타깃 그룹 |
| **TargetGroupBinding** | ALB, 리스너, 규칙, 타깃 그룹, WAF | Pod 등록만 |

- LBC가 전부 만들면 앱 팀이 YAML로 빠르게 바꿀 수 있지만, ALB가 Terraform state 밖에 있어 Route 53·WAF 연결을 Terraform에서 참조하기 불편하다.
- TargetGroupBinding은 ALB를 Terraform으로 고정하고 Kubernetes는 Pod 등록만 맡는다. ALB가 Ingress 삭제로 사라질 걱정이 없다. 다만 컨트롤러가 만든 TargetGroupBinding을 지우면 모든 타깃이 바로 등록 해제된다.

## 이슈 / 주의

- **LBC 기본값에 기대기**: internal·instance가 기본이다. 의도와 다르면 바꾸는 순간 ALB가 재생성된다.
- **IngressGroup 권한**: 그룹 안 다른 팀의 Ingress가 공유 ALB의 WAF·리다이렉트 설정을 바꿀 수 있다.
- **Ingress 삭제 = ALB 삭제**: DNS 이름이 바뀐다. 중요한 ALB는 삭제 보호 또는 TargetGroupBinding 구성을 쓴다.
- **readiness gate 라벨 누락**: 롤링 배포 중 healthy 타깃이 0이 되는 순간이 생길 수 있다.
- **preStop 없이 종료**: 종료 중인 Pod로 요청이 가서 502가 난다.
- **ECS `healthCheckGracePeriodSeconds = 0`**: 시작이 느린 앱이 헬스체크 실패로 계속 재시작된다.
- **Terraform `ssl_policy` 생략**: TLS 1.0 허용 정책이 들어간다.
- **타깃 그룹 고정 이름**: 포트·프로토콜 변경 때 apply가 실패한다.

## 출처

- https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html
- https://github.com/kubernetes-sigs/aws-load-balancer-controller/releases
- https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/deploy/installation.md
- https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/deploy/subnet_discovery.md
- https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/guide/ingress/annotations.md
- https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/guide/service/annotations.md
- https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/guide/gateway/gateway.md
- https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/guide/targetgroupbinding/targetgroupbinding.md
- https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/main/docs/deploy/pod_readiness_gate.md
- https://docs.aws.amazon.com/eks/latest/best-practices/load-balancing.html
- https://docs.aws.amazon.com/eks/latest/userguide/auto-configure-alb.html
- https://docs.aws.amazon.com/eks/latest/userguide/auto-configure-nlb.html
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/alb.html
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/load-balancer-connection-draining.html
- https://docs.aws.amazon.com/AmazonECS/latest/APIReference/API_CreateService.html
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/alb-resources-for-blue-green.html
- https://aws.amazon.com/about-aws/whats-new/2025/07/amazon-ecs-built-in-blue-green-deployments/
- https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-connect.html
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_listener
- https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/lb_target_group
- https://github.com/hashicorp/terraform-provider-aws/releases

---
*조사 기준일: 2026-10-26. HCL은 Terraform `validate`, YAML은 파싱 검사를 했다(결과는 README 참고). 실제 클러스터에 적용해 보지는 않았다.*
