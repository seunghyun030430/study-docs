# AWS Systems Manager 정리

AWS Systems Manager(SSM)의 개념·구조·사용법·이슈를 실무 관점에서 정리한 문서 모음. 중점 기능은 **Session Manager / Parameter Store / Run Command·Automation**. 조사 기준일 **2026-10-12**.

## 한눈에 보기

| 기능 | 한 줄 요약 | 핵심 요금 (기준일 시점) | 꼭 알아둘 함정 |
| --- | --- | --- | --- |
| Session Manager | 인바운드 포트·bastion·SSH 키 없이 IAM으로 셸 접속·포트포워딩 | EC2 무료 / 하이브리드 세션당 $0.05 | 포트포워딩·SSH 세션은 **로그 미기록** |
| Parameter Store | 계층형 설정값·SecureString 저장 | Standard 무료 / Advanced 월 $0.05 + API 1만 건당 $0.05 | Get 계열 기본 **40 TPS**, 자동 로테이션 없음 |
| Run Command | 태그·리소스 그룹 단위로 노드에 명령 일괄 실행 | EC2 무료 / 하이브리드 호출당 $0.002 | `Success` = 마지막 명령 종료 코드 0일 뿐 |
| Automation | AWS API + 노드 명령을 엮은 다단계 런북 | step당 $0.002, **free tier 없음** | 기본은 실행자 권한, assume role 권장 |
| Quick Setup | 권장 구성(Host Management, Patch Policy, Resource Scheduler 등)을 계정·조직에 일괄 배포 | 무료(설정한 서비스 요금은 별도) | 만든 리소스를 직접 고치면 drift, management 계정엔 배포 안 됨, Resource Scheduler는 EC2 전용 |

공통 전제: **SSM Agent + IAM(instance profile 또는 DHMC) + 아웃바운드 443(`ssm`, `ssmmessages`)** 가 갖춰져야 managed node가 된다.

| 비교 | Parameter Store | Secrets Manager |
| --- | --- | --- |
| 자동 로테이션 | 없음 | 지원 |
| 최대 크기 | 4 KB / 8 KB | 64 KB |
| 저장 비용 | Standard 무료 | 시크릿당 월 $0.40 |
| 언제 | 설정값, 로테이션이 필요 없는 시크릿 | DB 자격증명 등 로테이션 대상 |

## 문서 인덱스

| 문서 | 다루는 것 |
| --- | --- |
| [01-overview.md](./01-overview.md) | 개요 — 정의, 4개 범주 기능 지도, managed node, 2024–2026 변화, 요금 |
| [02-architecture.md](./02-architecture.md) | 동작 원리 — SSM Agent, 엔드포인트, VPC endpoint, IAM(DHMC/instance profile), SSM Document, 하이브리드 노드 |
| [03-session-manager.md](./03-session-manager.md) | Session Manager — 접속, 포트포워딩, SSH, 세션 설정·로깅, IAM 제한 |
| [04-parameter-store.md](./04-parameter-store.md) | Parameter Store — 타입·티어, TPS, KMS, 버전·라벨, 공유, Secrets Manager 비교 |
| [05-run-command-automation.md](./05-run-command-automation.md) | Run Command·Automation — 대상·rate control, 출력·상태, 런북 작성·실행 |
| [06-quick-setup.md](./06-quick-setup.md) | Quick Setup — 동작 방식(StackSets·association), 온보딩 role, 설정 유형 11개(Host Management, DHMC, Patch Policy 등), Resource Scheduler, 위임 관리자, CLI·Terraform |
| [07-issues.md](./07-issues.md) | 이슈 — 노드 미등록 진단, 쓰로틀링·한도, 보안 안티패턴 |

## 출처 (1차)

- 사용자 가이드: https://docs.aws.amazon.com/systems-manager/latest/userguide/what-is-systems-manager.html
- 요금: https://aws.amazon.com/systems-manager/pricing/
- 서비스 한도: https://docs.aws.amazon.com/general/latest/gr/ssm.html
- AWS CLI `ssm` 레퍼런스: https://docs.aws.amazon.com/cli/latest/reference/ssm/
- 보안 모범 사례: https://docs.aws.amazon.com/systems-manager/latest/userguide/security-best-practices.html
- Release history: https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-release-history.html
- SSM Agent: https://github.com/aws/amazon-ssm-agent

---
*이 문서는 학습·정리 목적의 요약이다. 요금·한도·기능 제공 여부는 자주 바뀌므로 정확한 값은 항상 위 공식 출처를 확인할 것.*
