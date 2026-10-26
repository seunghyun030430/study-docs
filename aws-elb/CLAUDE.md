# CLAUDE.md — aws-elb

공통 작성 원칙·커밋 규칙은 루트 [CLAUDE.md](../CLAUDE.md)를 따른다. 여기에는 이 주제에만 해당하는 규칙을 적는다.

## 범위

- 주제: AWS Elastic Load Balancing(ELB)을 실무 관점에서 정리. **ALB(Application Load Balancer) 중심**, NLB·GWLB·CLB는 비교와 선택 기준 위주
- 중점: 라우팅·타깃 그룹·헬스체크 / TLS·보안 / EKS·ECS 연동과 IaC / 운영·관측·비용
- 관련 주제: 인증서 키와 암호화는 [`aws-kms`](../aws-kms/CLAUDE.md), 메트릭·알람은 [`aws-cloudwatch`](../aws-cloudwatch/README.md), IAM role·IRSA는 [`aws-iam`](../aws-iam/README.md), Terraform 문법·구조는 [`terraform`](../terraform/README.md)을 링크하고 반복 설명하지 않는다.
- 예제 스택: 문서 안의 AWS CLI(`aws elbv2`) + Terraform HCL(hashicorp/aws) + Kubernetes YAML 코드블록. 별도 `examples/` 없음.

## 주제 전용 규칙

- **용어는 처음 나올 때 정의한다.** `01-overview.md`는 처음 보는 사람 기준으로 단계별로 쌓아 올리고 끝에 용어 정리를 둔다. `02` 이후 문서는 요약 바로 아래에 "이 문서에서 새로 나오는 용어" 표를 둔다.
- 1차 출처는 Elastic Load Balancing 사용자 가이드(ALB·NLB·GWLB·CLB), AWS CLI `elbv2` 레퍼런스, 요금 페이지, AWS Load Balancer Controller 공식 문서를 우선한다.
- **기본값과 한도(타임아웃, 헬스체크, 쿼터)와 요금은 자주 바뀐다.** 조사 기준일과 출처를 남기고, 금액은 us-east-1과 서울 리전을 구분한다.
- 로드 밸런서 종류마다 같은 이름의 설정이 기본값이 다르다(예: cross-zone). 어느 종류의 값인지 항상 밝힌다.
- HCL은 ` ```hcl `로 쓰고 `terraform validate`로 검사한다. YAML은 파싱 검사를 한다.
- 로드 밸런서·리스너·타깃 그룹 삭제처럼 트래픽을 끊는 작업에는 경고를 함께 적는다.
- 커밋 scope: `docs(aws-elb): ...`
