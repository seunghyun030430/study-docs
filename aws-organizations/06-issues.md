# 이슈·한계·안티패턴

> Organizations 사고는 대부분 **"정책을 붙이거나 계정을 옮기는 순간 바로 적용된다"**는 점에서 나온다. 테스트 없이 조직 root에 SCP를 붙이기, FullAWSAccess 떼기, 예외 role 없는 Deny, management 계정에 워크로드 두기가 대표적이다. 한도 쪽에서는 **새 조직의 계정 수 기본 10개**와 **SCP 크기 10,240자**에 가장 먼저 부딪힌다.

## 이 문서에서 새로 나오는 용어

[01](./01-overview.md)~[05](./05-other-policies.md)의 용어를 안다고 가정한다.

| 용어 | 뜻 |
| --- | --- |
| Hard limit | 늘릴 수 없는 고정 한도 |
| Break-glass | 평소엔 쓰지 않고 비상시에만 꺼내 쓰는 접근 수단(비상용 role·계정) |
| Minify | JSON의 공백·줄바꿈을 없애 크기를 줄이는 것 |

## 주요 한도

출처: [Quotas for AWS Organizations](https://docs.aws.amazon.com/organizations/latest/userguide/orgs_reference_limits.html)

| 항목 | 값 | 비고 |
| --- | --- | --- |
| 조직당 계정 | **기본 10개**, 최대 50,000개까지 증설 | 새 조직은 10보다 낮을 수 있음. 초대 대기 계정도 포함 |
| OU | 2,000개 | — |
| OU 중첩 | 조직 root 아래 5단계 | — |
| 동시 계정 생성 | 5개 | — |
| 초대 | 24시간에 20개 또는 계정 한도 중 큰 값. 15일 뒤 만료 | — |
| 계정 해지 | 30일에 member 계정의 20% 또는 250개 중 큰 값(최대 1,000), 동시 3개 | hard limit |
| 조직에서 만든 계정 제거 | 생성 후 4일 경과 필요 | — |
| SCP 크기 / 직접 연결 | 10,240자 / 대상당 10개 | hard limit |
| RCP 크기 / 직접 연결 | 5,120자 / 대상당 5개(RCPFullAWSAccess 포함) | hard limit |
| 선언형 정책 크기 | 대부분 10,000자(AI opt-out 2,500자) | — |
| 엔티티당 태그 | 50개 | — |
| Delegation policy | 40,000자 | — |

## 흔한 장애와 함정

| 증상 | 원인 | 대응 |
| --- | --- | --- |
| 계정 전체에서 모든 API가 `AccessDenied` | FullAWSAccess를 대체 정책 없이 뗌(allow-list 전환 실수) | 대체 Allow를 먼저 붙이고 뗀다. 복구는 management 계정에서 정책 재연결 |
| 서울 리전만 허용했더니 IAM·Route 53·지원 센터가 안 됨 | 리전 제한 SCP에서 글로벌 서비스 예외(`NotAction`) 누락 | AWS 예시의 `NotAction` 목록 사용, us-east-1 허용 검토 |
| 관리자도 고칠 수 없게 됨 | 예외 없는 Deny | `aws:PrincipalArn`으로 관리·break-glass role 예외 |
| 계정을 OU로 옮기자 배포 파이프라인 실패 | 새 OU의 SCP가 즉시 적용 | 이동 전 접근 보고서·CloudTrail로 영향 조사([03](./03-scp-rcp.md#안전한-테스트와-롤아웃)) |
| 외부 파트너가 갑자기 버킷에 접근 못 함 | identity perimeter RCP 적용 | 적용 전 Access Analyzer 외부 접근 결과 확인, 파트너 계정 예외 |
| SCP를 저장하려니 크기 초과 | 10,240자 한도, CLI는 공백도 셈 | JSON minify, 정책을 목적별로 나누기(대상당 10개까지) |
| management 계정에서 SCP가 안 먹음 | 설계상 SCP·RCP 미적용 | management 계정에 워크로드를 두지 않음 |
| 새 계정을 더 못 만듦 | 계정 수 한도(기본 10), 해지 대기 계정도 90일간 포함 | 미리 증설 요청, 해지 계정은 Suspended OU로 |
| `AWSReservedSSO_…` role을 참조하던 정책이 깨짐 | permission set 재할당으로 접미사 변경 | `ArnLike` + 와일드카드([04](./04-identity-center.md#permission-set과-할당)) |
| Identity Center 사용자 동기화 멈춤 | SCIM 토큰 1년 만료 | 만료 전 갱신 일정 관리 |
| Control Tower가 drift 경고, 계정 등록 불가 | 관리 SCP나 OU를 Organizations에서 직접 수정 | Control Tower 콘솔·API로 변경, landing zone 복구([05](./05-other-policies.md#organizations를-직접-다룰-때-주의-drift)) |
| SCP를 껐다 켰더니 정책이 다 사라짐 | 재활성화 시 FullAWSAccess만 남음 | SCP 정책 유형을 끄지 않음. IaC로 정책·연결 관리 |

## 안티패턴

| 안티패턴 | 왜 위험한가 | 대안 |
| --- | --- | --- |
| management 계정에서 서비스 운영 | SCP·RCP 밖이고, 탈취되면 조직 전체가 위험 | 워크로드 금지, 위임 관리자 활용 |
| management 계정 접근을 그룹에 할당 | 그룹 멤버십을 바꿀 수 있는 사람이 management 접근을 얻음 | 개인 할당 + 최소 인원 + MFA |
| 조직도 그대로 OU 설계 | 팀 개편마다 계정 이동 = 정책 변경 | 정책(보안·환경) 기준 OU |
| 깊은 OU 계층 | 상속 추적이 어려워 의도치 않은 Deny·누락 | 2–3단계 위주로 얕게 |
| 테스트 없이 조직 root에 SCP 연결 | 전 계정 동시 장애 | 테스트 OU → 하위 OU → 상위 OU |
| 콘솔에서 정책을 손으로 관리 | 변경 이력·리뷰가 없음 | IaC(CloudFormation, Terraform)와 코드 리뷰, `validate-policy`를 CI에 |
| SCP 정책 관리를 넓게 위임 | 위임받은 계정이 조직 전체를 막을 수 있음 | 위임은 backup·tag 같은 선언형 정책 위주로 |
| member 계정 root를 그대로 방치 | 계정마다 최고 권한 자격증명이 흩어져 있음 | root 중앙 관리로 자격증명 제거([05](./05-other-policies.md#root-자격증명-중앙-관리)) |

## 출처

- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_reference_limits.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html
- https://docs.aws.amazon.com/organizations/latest/userguide/orgs_best-practices_mgmt-acct.html
- https://docs.aws.amazon.com/whitepapers/latest/organizing-your-aws-environment/design-principles-for-your-multi-account-strategy.html
- https://docs.aws.amazon.com/singlesignon/latest/userguide/delegated-admin.html
- https://docs.aws.amazon.com/controltower/latest/userguide/drift.html

---
*조사 기준일: 2026-10-13. "흔한 장애" 표는 공식 문서의 규칙에서 예상되는 결과를 정리한 것이며, 실제 장애 사례 통계에 근거한 것은 아니다.*
